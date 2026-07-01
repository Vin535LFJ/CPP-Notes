# 04 — 工程实践：从 Bug 排查到构建优化的全链路

> **定位**：内存泄漏排查、GDB 高级调试、CMake 构建优化、Linux 系统调用与性能工具链。
> **适用大厂**：所有一线大厂（工程能力是所有面试的底层要求）

---

## 目录

1. [内存泄漏排查工具全比较](#1-内存泄漏排查工具全比较)
2. [GDB 高级调试技巧](#2-gdb-高级调试技巧)
3. [CMake 现代构建最佳实践](#3-cmake-现代构建最佳实践)
4. [Linux 系统调用与内核交互](#4-linux-系统调用与内核交互)
5. [性能剖析工具链](#5-性能剖析工具链)
6. [C++ 编译优化实战](#6-c-编译优化实战)
7. [网络编程调试与抓包](#7-网络编程调试与抓包)
8. [代码规范与静态分析](#8-代码规范与静态分析)
9. [常见崩溃场景与 Core Dump 分析](#9-常见崩溃场景与-core-dump-分析)
10. [面试官的\"项目经历\"追问模板](#10-面试官的项目经历追问模板)

---

## 1. 内存泄漏排查工具全比较

### 1.1 工具矩阵

| 工具 | 平台 | 原理 | 侵入性 | 运行时开销 | 适合阶段 |
|------|------|------|--------|------------|----------|
| **Valgrind Memcheck** | Linux | 动态二进制插桩 | ❌ 无需重编译 | 10-30x 慢 | 开发/测试 |
| **AddressSanitizer (ASan)** | GCC/Clang/MSVC | 编译期插桩 | ✅ 需重编译 | 2x 慢 | 开发/测试 |
| **LeakSanitizer (LSan)** | GCC/Clang | ASan 的轻量子集 (仅泄漏) | ✅ 需重编译 | ~1.2x | CI/测试 |
| **Dr.Memory** | Windows/Linux | 动态二进制插桩 | ❌ 无需重编译 | 5-10x 慢 | 开发/测试 |
| **Heaptrack** | Linux | LD_PRELOAD 劫持 malloc | ❌ 无需重编译 | ~2x | 开发/测试/性能分析 |
| **tcmalloc Heap Profiler** | Linux | 替换 malloc 实现 | ❌ 需 LD_PRELOAD | ~1.1x | 线上轻量采样 |
| **mtrace** | Linux (glibc) | glibc 内置 | ✅ 需重新 link | 低 | 快速诊断 |

### 1.2 AddressSanitizer：现代首选

```bash
# 编译
g++ -fsanitize=address -g -O1 main.cpp -o main

# 运行 —— 自动检测：
# - heap-use-after-free
# - heap-buffer-overflow
# - stack-buffer-overflow
# - global-buffer-overflow
# - memory leaks (退出时报告)

# 运行后如果报错，会精确输出：
# ==12345==ERROR: AddressSanitizer: heap-use-after-free on address 0x...
#     READ of size 4 at 0x... thread T0
#     #0 0x... in useAfterFree() main.cpp:15
#     #1 0x... in main main.cpp:25
#   freed by thread T0 here:
#     #0 0x... in operator delete(void*)
#     #1 0x... in deleteAndUse() main.cpp:10
#     #2 0x... in main main.cpp:24

# 高级选项
export ASAN_OPTIONS=detect_leaks=1:halt_on_error=1:abort_on_error=1
export ASAN_SYMBOLIZER_PATH=/usr/bin/llvm-symbolizer  # 更好的符号化
```

### 1.3 Valgrind：经典无需重编译

```bash
# 基础内存检测
valgrind --leak-check=full --show-leak-kinds=all ./my_program

# 详细报告
valgrind --leak-check=full --track-origins=yes \
         --verbose --log-file=valgrind-out.txt ./my_program

# --leak-check=full: 详细报告每个泄漏
# --track-origins=yes: 追踪未初始化值的来源
# --show-leak-kinds=all: 显示所有类型泄漏 (definite/possible/indirect)

# 配合 GDB 使用
valgrind --vgdb=yes --vgdb-error=0 ./my_program
# 另一个终端: gdb ./my_program  →  target remote | vgdb
```

### 1.4 Heaptrack：内存分配火焰图

```bash
# 运行 profiling
heaptrack ./my_program

# 分析结果（GUI）
heaptrack_gui heaptrack.my_program.XXXX.gz

# 输出：
# - 总内存分配量
# - Top 分配热点（类似火焰图）
# - 临时分配 vs 长期持有
# - 每个调用栈的分配数量
```

### 1.5 shared_ptr 循环引用排查

```cpp
// 经典循环引用导致内存泄漏
class A {
public:
    std::shared_ptr<B> ptr_b;
    ~A() { std::cout << "~A\n"; }
};

class B {
public:
    std::shared_ptr<A> ptr_a;  // ❌ 应改为 weak_ptr<A>
    // std::weak_ptr<A> ptr_a;  // ✅ 解决循环引用
    ~B() { std::cout << "~B\n"; }
};

// 排查方法 —— 在析构函数中打印日志
// 如果程序退出时 ~A 和 ~B 都没有输出 → 怀疑循环引用
```

### 1.6 面试官连环追问 🎯

> **Q1**：Valgrind 和 ASan 的原理有何本质区别？

> **回答**：Valgrind 是**动态二进制翻译**——将原始机器码翻译为中间表示 (VEX IR)，在 IR 上插桩检测代码，然后重新编译执行。这不需要修改源码或重编译，但代价是将程序运行在\"虚拟机\"中，速度慢 10-30 倍。ASan 是**编译期插桩**——编译器在每次内存访问前插入检测代码（shadow memory 检查），将程序本身变异为带检测的版本，运行时开销只有约 2 倍。两者互补：ASan 用于日常开发和 CI，Valgrind 用于无法重编译的第三方库检测。

> **Q2**：线上的程序能跑 ASan 吗？

> **回答**：**不建议**。ASan 有 2 倍性能开销和约 2-3 倍的内存开销（shadow memory），且检测到错误默认会 abort。但可以使用**LSan 的\"standalone 模式\"**（`-fsanitize=leak`）在线上做低开销的泄漏检测，或使用 **tcmalloc 的 Heap Profiler** 做采样（开销约 1%）。

> **Q3**：`shared_ptr` 的循环引用为什么 `unique_ptr` 不存在？

> **回答**：因为 `unique_ptr` 是**独占所有权**——在任何时刻只有一个 `unique_ptr` 拥有资源。它根本不允许两个对象互相持有对方的 `unique_ptr`（拷贝被 delete），所以不可能形成引用环。即使通过移动实现\"互相引用\"，由于独占性，所有权图必然是有向无环图 (DAG)，析构自然终止。

---

## 2. GDB 高级调试技巧

### 2.1 调试符号与编译

```bash
# 必须编译时带调试符号
g++ -g -O0 main.cpp -o main   # -g: 调试符号, -O0: 关闭优化（避免变量被优化掉）

# 对于优化后的 Release 版本
g++ -g -O2 main.cpp -o main   # 仍然可以加 -g，但变量可能被优化

# 检查是否有调试符号
file main              # 输出 "not stripped" 表示有符号
readelf -S main | grep debug   # 查看 .debug_* 段
```

### 2.2 核心 GDB 命令速查

```gdb
# === 启动 ===
gdb ./program                          # 直接调试
gdb ./program core.12345               # 分析 core dump
gdb -p $(pidof program)                # attach 到运行中进程

# === 断点 ===
b main                                 # 函数断点
b file.cpp:42                          # 行号断点
b *0x400123                            # 地址断点
b func if x > 10                       # 条件断点
tbreak func                            # 临时断点（一次生效）
rbreak regex                           # 正则匹配所有匹配函数
info breakpoints                       # 列出所有断点
delete 1                               # 删除断点 1
disable/enable 1                       # 禁用/启用

# === 运行控制 ===
r                                      # 运行 (run)
r arg1 arg2                            # 带参数运行
c                                      # 继续 (continue)
s                                      # 单步进入 (step into)
n                                      # 单步跳过 (step over)
finish                                 # 执行到当前函数返回
until 42                               # 运行到第 42 行

# === 查看数据 ===
p var                                  # 打印变量
p/x var                                # 十六进制打印
p/t var                                # 二进制打印
p *ptr@10                              # 打印数组 10 个元素
watch var                              # 监视变量变化
info locals                            # 查看所有局部变量
info args                              # 查看函数参数
ptype ClassName                        # 打印类型定义（包含成员）

# === 调用栈 ===
bt                                     # 当前线程调用栈 (backtrace)
bt full                                # 调用栈 + 局部变量
thread apply all bt                    # 🔥 所有线程的调用栈
frame 3                                # 切换到栈帧 3
up/down                                # 上/下切换栈帧

# === 线程 ===
info threads                           # 列出所有线程
thread 5                               # 切换到线程 5

# === 内存 ===
x/16xw $rsp                            # 查看栈顶 16 个 word（十六进制）
x/s ptr                                # 以字符串形式查看
x/i $pc                                # 查看当前指令

# === 反汇编 ===
disas func                             # 反汇编函数
disas /m func                          # 混合源码和汇编

# === 多进程 ===
set follow-fork-mode child             # 跟踪子进程
set detach-on-fork off                 # 父进程不退出

# === 布局 ===
layout src                             # 源码窗口
layout asm                             # 汇编窗口
layout split                           # 源码 + 汇编
tui disable                            # 退出 TUI 模式
Ctrl-x a                               # 切换 TUI
```

### 2.3 反向调试（Reverse Debugging）

```gdb
# GDB 7.0+ 支持，但开销大
target record-full                     # 开始记录
reverse-continue                       # 反向继续
reverse-step                           # 反向单步
reverse-next                           # 反向跳过
reverse-finish                         # 反向执行到调用点

# 使用场景：发现 crash 后反向追踪到崩溃前的状态
```

### 2.4 Python 脚本扩展

```python
# gdb_pretty.py — 自定义 GDB 美化打印
class StdVectorPrinter:
    """为 std::vector 提供美观的打印"""
    def __init__(self, val):
        self.val = val
    def to_string(self):
        start = self.val['_M_impl']['_M_start']
        finish = self.val['_M_impl']['_M_finish']
        size = finish - start
        return f"std::vector of length {size}"
    def children(self):
        start = self.val['_M_impl']['_M_start']
        finish = self.val['_M_impl']['_M_finish']
        for i, item in enumerate(range(int(start), int(finish))):
            yield f"[{i}]", item.dereference()

# 加载：在 ~/.gdbinit 中添加 source /path/to/gdb_pretty.py
```

### 2.5 核心 Dump 分析

```bash
# 1. 允许生成 core dump
ulimit -c unlimited
echo "core.%e.%p" > /proc/sys/kernel/core_pattern

# 2. 触发 core dump
kill -ABRT $(pidof program)   # 进程不退出但生成 core
# 或程序 crash 自动生成 core

# 3. 分析
gdb ./program core.12345
(gdb) bt full               # 完整调用栈
(gdb) info registers        # 寄存器状态
(gdb) x/16xw $rsp           # 栈内存
(gdb) thread apply all bt   # 所有线程
```

### 2.6 面试官连环追问 🎯

> **Q1**：Release 版本崩溃了，只有地址没有符号，怎么定位问题？

> **回答**：**①** 保留带 `-g` 编译的版本，用 `strip` 发布——保留一份 `unstripped` 版本用于事后调试。**②** 用 `addr2line -e program 0xaddress` 或 `gdb -batch -ex "info line *0xaddress" program` 将地址转换为行号。**③** 在 CMake 中用 `add_custom_command` 自动保存 debug 版本的二进制到 CI artifact。

> **Q2**：`thread apply all bt` 是所有线程的全量调用栈，如何快速定位哪些线程在等锁？

> **回答**：grep `pthread_mutex_lock`、`futex_wait`、`__lll_lock_wait`、`std::mutex::lock`。等待锁的线程会停在这些函数中。如果多个线程都在各自的 mutex lock 处阻塞，且形成了依赖循环 → 死锁。此外 `info threads` 可以看到每个线程的当前函数，一眼能看出是否有大量线程在同步原语上等待。

> **Q3**：GDB 附加到进程会有什么影响？生产环境能 attach 吗？

> **回答**：GDB attach 会通过 `ptrace` 系统调用接管进程——**进程被暂停**（发送 SIGSTOP）。在单线程场景下只是短暂暂停，多线程影响更大（所有线程暂停）。attach 后进程的性能不受影响（调试断点除外）。生产环境**可以** attach（用 `gdb -p`）但不建议设断点——建议只看 `bt` 和 `info threads` 后立即 `detach`。频繁 attach/detach 可能触发内核调度抖动。

---

## 3. CMake 现代构建最佳实践

### 3.1 现代 CMake 项目骨架

```cmake
cmake_minimum_required(VERSION 3.20)
project(MyProject VERSION 1.0.0 LANGUAGES CXX)

# C++ 标准设定
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)  # 禁用 GCC 扩展以保证可移植性

# 导出 compile_commands.json（供 clangd / clang-tidy / IDE 使用）
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# 编译选项
add_compile_options(-Wall -Wextra -Wpedantic -Wshadow)
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    add_compile_options(-g -O0 -fsanitize=address,undefined)
elseif(CMAKE_BUILD_TYPE STREQUAL "Release")
    add_compile_options(-O3 -DNDEBUG)
elseif(CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
    add_compile_options(-O2 -g -DNDEBUG)
endif()

# 库（用目标而非变量）
add_library(my_lib STATIC
    src/foo.cpp
    src/bar.cpp
)
target_include_directories(my_lib PUBLIC include)      # PUBLIC: 消费者也可见
target_compile_features(my_lib PUBLIC cxx_std_20)       # 传播 C++20 要求

# 可执行文件
add_executable(my_app src/main.cpp)
target_link_libraries(my_app PRIVATE my_lib)             # PRIVATE: 不传播

# 第三方依赖 (FetchContent — CMake 3.14+)
include(FetchContent)
FetchContent_Declare(
    fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG 10.2.1
)
FetchContent_MakeAvailable(fmt)
target_link_libraries(my_app PRIVATE fmt::fmt)

# 测试
enable_testing()
add_executable(unit_tests test/test_main.cpp)
target_link_libraries(unit_tests PRIVATE my_lib)
add_test(NAME UnitTests COMMAND unit_tests)
```

### 3.2 依赖管理对比

| 方案 | 优点 | 缺点 | 适用 |
|------|------|------|------|
| **FetchContent** | CMake 原生、版本锁定 | 每次 configure 下载 | 中小项目 |
| **Conan / vcpkg** | 专业的包管理器 | 额外工具依赖 | 大中型项目 |
| **git submodule** | 简单 | 需手动管理版本 | 少量依赖 |
| **find_package** | 利用系统库 | 版本不可控 | 系统库 |

### 3.3 IMPORTED 目标 vs INTERFACE 目标

```cmake
# INTERFACE 库：header-only 库
add_library(my_headerlib INTERFACE)
target_include_directories(my_headerlib INTERFACE include)

# IMPORTED 库：预编译外部库
add_library(external_lib STATIC IMPORTED)
set_target_properties(external_lib PROPERTIES
    IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/lib/libexternal.a
    INTERFACE_INCLUDE_DIRECTORIES ${CMAKE_SOURCE_DIR}/3rdparty/include
)
```

### 3.4 生成器表达式（Generator Expressions）

```cmake
# 条件编译选项（仅在特定配置生效）
target_compile_options(my_lib PRIVATE
    $<$<CONFIG:Debug>:-fsanitize=address>
    $<$<CXX_COMPILER_ID:GNU>:-fdiagnostics-color=always>
)

# 多配置生成器的路径处理
add_custom_command(TARGET my_app POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
    $<TARGET_FILE:my_lib>
    $<TARGET_FILE_DIR:my_app>
)
```

### 3.5 面试官连环追问 🎯

> **Q1**：`target_link_libraries` 的 PUBLIC / PRIVATE / INTERFACE 有何区别？

> **回答**：这是 CMake 基于目标的依赖管理的核心：**PRIVATE**——依赖仅用于自身构建，不传播给消费者（如内部使用的实现库）。**INTERFACE**——依赖不用于自身构建，但传播给消费者（如 header-only 库的头文件路径）。**PUBLIC**——两者兼有（依赖既用于自身构建，又传播给消费者）。正确的选择能让依赖传播最小化，减少不必要的重编译。

> **Q2**：`FetchContent` 和 `ExternalProject` 有什么区别？

> **回答**：`FetchContent` 在 **configure** 阶段下载并引入依赖，依赖直接成为当前 CMake 构建的一部分（同一构建系统）。`ExternalProject` 在 **build** 阶段下载并在独立的构建系统中构建依赖，通过 `add_dependencies` 建立顺序关系。FetchContent 更简单直接，适合源码级集成；ExternalProject 适合完全隔离的构建（如交叉编译不同平台的库）。

---

## 4. Linux 系统调用与内核交互

### 4.1 用户态到内核态的路径

```
  ┌──────────────┐
  │   用户态      │  应用程序执行
  │   app code   │
  └──────┬───────┘
         │ syscall (int 0x80 / sysenter / syscall 指令)
         ▼
  ┌──────────────┐
  │   内核态      │  内核处理系统调用
  │   kernel     │
  │  - 权限检查  │
  │  - 参数拷贝  │  (copy_from_user)
  │  - 执行操作  │  (VFS → 驱动 → 硬件)
  │  - 结果返回  │  (copy_to_user)
  └──────────────┘
```

### 4.2 关键系统调用速查

| 系统调用 | glibc 封装 | 功能 | 高频场景 |
|----------|-----------|------|----------|
| `read` | `read(fd, buf, n)` | 读取文件/socket | I/O 操作 |
| `write` | `write(fd, buf, n)` | 写入文件/socket | I/O 操作 |
| `open / openat` | `open(path, flags)` | 打开文件 | 文件操作 |
| `close` | `close(fd)` | 关闭文件描述符 | 资源释放 |
| `mmap / munmap` | `mmap(NULL, sz, PROT, MAP, fd, 0)` | 内存映射 | 高性能 I/O、共享内存 |
| `brk / sbrk` | 内部由 malloc 调用 | 扩展堆空间 | 内存分配 |
| `epoll_create / epoll_ctl / epoll_wait` | — | I/O 多路复用 | 网络服务 |
| `fork / clone` | `fork()` / `std::thread` 底层 | 创建进程/线程 | 并发 |
| `futex` | `std::mutex` 底层 | 快速用户态互斥 | 锁机制 |
| `sendfile` | `sendfile(out_fd, in_fd, ...)` | 零拷贝文件传输 | Web 服务器 |

### 4.3 mmap：零拷贝的秘密

```cpp
// mmap vs read 的区别：
// read: 磁盘 → 内核页缓存 → 用户态缓冲区（两次拷贝）
// mmap: 将文件映射到虚拟地址空间
//       缺页中断 → 磁盘 → 内核页缓存（一次拷贝）
//       用户态直接通过指针访问页缓存中的内容

#include <sys/mman.h>
#include <fcntl.h>

int fd = open("large_file.dat", O_RDONLY);
struct stat sb;
fstat(fd, &sb);

void* ptr = mmap(nullptr, sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
// 现在可以像内存一样访问文件
const char* data = static_cast<const char*>(ptr);
use(data, sb.st_size);

munmap(ptr, sb.st_size);
close(fd);
```

### 4.4 fork / clone：进程 vs 线程的底层

```cpp
// fork: 创建子进程，父子进程独立地址空间 (COW)
pid_t pid = fork();
if (pid == 0) {
    // 子进程 — 只有当前线程被复制！
    // 父进程的其他线程在子进程中不存在
    // → 若父进程有线程持有锁，子进程中这个锁永远无法释放
    // → 多线程程序 fork 后子进程只能调用 async-signal-safe 函数 + exec()
}

// clone: std::thread 的 Linux 底层
// clone(CLONE_VM | CLONE_FS | CLONE_FILES | ...)
// CLONE_VM: 共享虚拟地址空间 → 线程
// 不加 CLONE_VM: 独立地址空间 → 进程
```

### 4.5 面试官连环追问 🎯

> **Q1**：为什么多线程程序 `fork()` 后在子进程中不能安全使用 `malloc`？

> **回答**：`fork()` 只复制调用线程到子进程。如果父进程中其他线程正在持有 `malloc` 内部锁（或任何锁），这个锁在子进程中永远无法释放——因为持有它的线程不存在。子进程下一次调用 `malloc` 就会死锁。标准规定 `fork()` 后子进程只能调用 **async-signal-safe** 函数（`exec`、`_exit`、`write` 等），`malloc` 和 `printf` 都不在此列。解决方案：`fork()` → 立即 `exec()` 替换进程映像，或使用 `pthread_atfork()` 注册 fork 前后的 handler。

> **Q2**：`sendfile` 的零拷贝是如何实现的？

> **回答**：`sendfile(out_fd, in_fd, &offset, count)` 在**内核态**直接将数据从文件页缓存复制到 socket 缓冲区，完全绕过用户态。传统的 read+write 需要 4 次拷贝（磁盘→页缓存→用户态→socket 缓冲区→网卡），而 sendfile 只 2 次（磁盘→页缓存→socket 缓冲区）甚至更低（DMA 直接传输）。在高性能 web 服务器（Nginx）、静态文件服务器中至关重要。

---

## 5. 性能剖析工具链

### 5.1 工具全景

| 工具 | 层级 | 主要能力 | 输出 |
|------|------|----------|------|
| **perf** | 内核/硬件 | CPU 采样、cache miss、分支预测 | `perf report` 火焰图 |
| **FlameGraph** | — | 将 perf 输出转为火焰图 | SVG |
| **gperftools (tcmalloc)** | 堆内存 | malloc profiling + CPU profiling | 文本/graphviz |
| **Valgrind Callgrind** | 用户态模拟 | 指令级分析 + 缓存模拟 | KCachegrind 可视化 |
| **Intel VTune** | 硬件 | 微架构分析 (μop, cache, CPI) | GUI |
| **uftrace** | 函数级 | 函数调用统计 + 参数/返回值 | CLI |
| **strace** | 系统调用 | 追踪所有系统调用 | 文本 |

### 5.2 perf：Linux 性能分析的瑞士军刀

```bash
# CPU 采样
perf record -g ./my_program          # 记录采样（-g: 保留调用栈）
perf report                          # 交互式查看报告

# 实时 top-down
perf top -g                          # 类似 htop 但看到的是函数

# 统计性能计数器
perf stat -e cycles,instructions,cache-misses,branch-misses ./my_program
# 输出:
#   1,234,567,890  cycles
#     987,654,321  instructions   # IPC = 0.80
#      12,345,678  cache-misses   # cache miss rate
#       1,234,567  branch-misses

# 生成火焰图
perf record -g ./my_program
perf script > out.perf
# 使用 Brendan Gregg 的 FlameGraph 工具
stackcollapse-perf.pl out.perf > out.folded
flamegraph.pl out.folded > flamegraph.svg
```

### 5.3 常见性能问题与对应工具

| 问题 | 症状 | 工具 | 解决方向 |
|------|------|------|----------|
| CPU 高 | top 显示 100% 单核 | `perf top` | 热点函数优化/算法复杂度 |
| 内存泄漏 | RSS 持续增长 | ASan/Valgrind/Heaptrack | 修复泄漏 |
| Cache Miss 高 | IPC < 1.0 | `perf stat` + VTune | 数据结构布局优化 (AoS→SoA) |
| 锁竞争 | 多核 CPU 但吞吐低 | `perf lock` | 无锁/细粒度锁 |
| 系统调用频繁 | strace 输出密集 | `strace -c` | 缓冲批量操作 |
| IO 等待 | iowait 高 | `iostat` + `iotop` | 异步 IO/缓存 |
| 上下文切换高 | cs > 10k/s | `vmstat` | 减少线程/协程 |

```bash
# 快速诊断组合
perf stat -e cycles,instructions,cache-misses,cache-references,\
  context-switches,cpu-migrations,page-faults,branches,branch-misses \
  -r 5 ./my_program
```

### 5.4 面试官连环追问 🎯

> **Q1**：`perf` 的采样原理是什么？为什么它不需要插桩就能工作？

> **回答**：`perf` 使用 **PMU（Performance Monitoring Unit）**——现代 CPU 内置的硬件计数器。它基于事件采样（如每 N 个时钟周期或每 M 次 cache miss 触发一次中断），在中断处理中记录当前的指令指针 (IP) 和调用栈。因为是硬件采样而非插桩，对被分析程序的运行时开销极小（通常 < 3%）。这也意味着它的精度受采样频率影响（采样频率越高越精确但开销越大）。

> **Q2**：如何判断一个程序的瓶颈是 CPU bound 还是 IO bound？

> **回答**：**①** `top`/`htop`：CPU 使用率 → CPU 高 = CPU bound，CPU 低但响应慢 = IO bound。**②** `perf stat`：IPC (instructions per cycle) 低 + cache-misses 高 → CPU bound（被内存延迟限制）。**③** `vmstat` / `iostat`：IO wait 百分比高 → IO bound。**④** `strace -c`：系统调用耗时最多的函数是 `read`/`write` → IO bound。

---

## 6. C++ 编译优化实战

### 6.1 加速编译

```bash
# ccache — 缓存编译结果
export CC="ccache gcc"
export CXX="ccache g++"
ccache -s  # 查看缓存命中率

# 预编译头 (Precompiled Headers)
# CMake:
target_precompile_headers(my_lib PRIVATE
    <vector>
    <string>
    <memory>
    "common/pch.h"
)

# 并行编译
make -j$(nproc)     # Make
cmake --build . -j  # CMake (自动检测核心数)

# 使用 mold/lld 替代 ld 加快链接
# CMake: -DCMAKE_EXE_LINKER_FLAGS="-fuse-ld=mold"

# 减少 include 依赖 — 使用前置声明 (forward declaration)
// ❌ foo.h
#include "bar.h"  // 包含完整定义

// ✅ foo.h
class Bar;  // 前置声明 — 编译依赖减少了 90%
```

### 6.2 二进制体积优化

```bash
# GCC
g++ -Os -s -ffunction-sections -fdata-sections \
    -Wl,--gc-sections -Wl,--strip-all

# -Os: 优化体积（而不是速度）
# -s: 剥离符号表
# -ffunction-sections: 每个函数独立段
# -fdata-sections: 每个数据变量独立段
# -Wl,--gc-sections: 链接时删除未引用的段
# 组合效果：可减少 30-50% 体积
```

### 6.3 LTO（链接时优化）

```bash
# GCC/Clang
g++ -flto -O2 *.cpp

# 原理：常规编译以 .cpp 为单位优化，LTO 在链接时
#       对整个程序做跨编译单元的优化（内联、去虚拟化等）
# 代价：链接时间大幅增加（数倍），但可获得 5-15% 性能提升

# CMake
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)  # C++17+ LTO 支持更好
```

### 6.4 面试官连环追问 🎯

> **Q1**：LTO 为什么能带来额外的优化机会？

> **回答**：常规编译的优化范围限于单个 `.cpp` 文件。LTO 打破了这一限制：**① 跨文件内联**——`inline` 函数定义在 A.cpp，调用在 B.cpp，无 LTO 时无法内联。**② 跨文件去虚拟化**——如果链接时能确定所有派生类，可以将虚调用转为直接调用再内联。**③ 死代码消除范围扩大**——被整个程序都未调用的函数可被完全移除。

> **Q2**：什么时候不应该开启 LTO？

> **回答**：**① 调试构建**（编译-调试循环时链接时间太长）；**② 增量开发**（改一行代码等几分钟链接不可接受）；**③ 需要精确符号定义的场景**（LTO 可能合并/重排符号）；**④ 分布式编译**（distcc/icecream 需要分发 `.o`，LTO 的中间表示 (IR/bytecode) 在分发时可能不兼容不同编译器版本）。

---

## 7. 网络编程调试与抓包

### 7.1 tcpdump / Wireshark

```bash
# tcpdump 基础
tcpdump -i eth0 port 8080 -w capture.pcap   # 抓包保存
tcpdump -r capture.pcap -A                  # 以 ASCII 查看
tcpdump -i any 'tcp and port 80'            # 过滤条件

# 常用 BPF 过滤
tcpdump 'host 192.168.1.1'
tcpdump 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0'  # SYN/FIN 包
tcpdump 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'  # 有数据的包
```

### 7.2 strace：追踪系统调用

```bash
# 追踪网络系统调用
strace -e trace=network -f ./my_server
# 输出：socket/bind/listen/accept/connect/send/recv 等

# 统计
strace -c ./my_program  # 运行后输出每个系统调用的次数和耗时

# 已运行的进程
strace -p $(pidof nginx) -f -e trace=epoll_wait,read,write

# 显示时间戳和耗时
strace -tt -T -e trace=network ./my_server
# 输出示例：
# 10:30:01.123456 read(5, "...", 4096) = 1024 <0.000045>
```

### 7.3 ss / netstat / lsof

```bash
# ss（现代替代 netstat）
ss -tlnp                     # TCP 监听端口
ss -tan                      # 所有 TCP 连接
ss -s                        # 统计摘要

# TIME_WAIT 状态连接数
ss -tan state time-wait | wc -l

# lsof
lsof -i :8080                # 占用 8080 端口的进程
lsof -p $(pidof my_server)   # 进程打开的所有文件/socket
```

---

## 8. 代码规范与静态分析

### 8.1 工具链

| 工具 | 类型 | 检查内容 |
|------|------|----------|
| **clang-format** | 格式化 | 代码风格 |
| **clang-tidy** | 静态分析 | 现代 C++ 最佳实践、bug 模式 |
| **Cppcheck** | 静态分析 | 未定义行为、内存泄漏、越界 |
| **include-what-you-use (IWYU)** | 依赖分析 | 多余/缺失的 #include |
| **cpplint** | 风格检查 | Google C++ Style Guide |

```bash
# clang-tidy
clang-tidy main.cpp --checks='*,-llvm-header-guard' -- -std=c++20

# Cppcheck
cppcheck --enable=all --inconclusive --std=c++20 src/

# IWYU
include-what-you-use -std=c++20 main.cpp
```

### 8.2 CI 集成示例

```yaml
# .github/workflows/static-analysis.yml
name: Static Analysis
on: [push, pull_request]
jobs:
  clang-tidy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
      - run: run-clang-tidy -p build
```

---

## 9. 常见崩溃场景与 Core Dump 分析

### 9.1 崩溃类型速查

| 信号 | 原因 | 典型代码 |
|------|------|----------|
| `SIGSEGV` (11) | 段错误：非法内存访问 | `*(int*)nullptr` / use-after-free |
| `SIGABRT` (6) | abort() 调用 / `std::terminate` | 析构函数抛异常 |
| `SIGBUS` (7) | 总线错误：未对齐访问 | 在 ARM 上非对齐访问 |
| `SIGFPE` (8) | 算术异常 | 整数除零 |
| `SIGILL` (4) | 非法指令 | 损坏的代码 / 不兼容的指令集 |

### 9.2 Core Dump 分析实战

```bash
# 1. 开启 core dump
ulimit -c unlimited
echo "/tmp/core.%e.%p.%t" | sudo tee /proc/sys/kernel/core_pattern

# 2. 复现崩溃
./my_program   # 崩溃 → 生成 /tmp/core.my_program.12345.1234567890

# 3. 分析
gdb ./my_program /tmp/core.my_program.12345.1234567890
(gdb) bt full                    # 完整调用栈
(gdb) frame 0                    # 切换到崩溃点
(gdb) info locals                # 查看变量
(gdb) p ptr                      # 查看指针
(gdb) x/16xw ptr                 # 查看指针指向的内存
```

### 9.3 面试官连环追问 🎯

> **Q1**：程序在开发环境正常，生产环境崩溃，且没有 core dump，怎么办？

> **回答**：**① 先确认 ulimit**——`ulimit -c` 可能为 0（禁用 core dump）。**② systemd 管理的服务**：core dump 被 `systemd-coredump` 接管，用 `coredumpctl list` 查看。**③ Docker 容器**：确保宿主机 `/proc/sys/kernel/core_pattern` 配置正确且容器有写权限。**④ 无论如何**都应该在 Release+DebugInfo 模式下保留 unstripped 二进制和对应的源码版本号，以便离线分析。

> **Q2**：如何判断是 stack overflow 还是 heap corruption 导致的 SIGSEGV？

> **回答**：**stack overflow** 的 SIGSEGV 发生在栈地址范围，`bt` 时会发现栈帧数量极大（递归失控）或单个栈帧分配了超大数组。`info proc mappings` 可看到栈区域。**heap corruption** 的 SIGSEGV 发生在堆地址范围，通常在 `malloc`/`free`/`new`/`delete` 内部调用中崩溃，`bt` 会显示 `operator new` 或 `malloc` 内部的异常。ASan 对两者都有精确检测。

---

## 10. 面试官的"项目经历"追问模板

### 10.1 STRT 法则回答项目问题

```
S — Situation    (背景：在什么背景下)
T — Task         (任务：需要解决什么问题)
R — Result       (结果：取得了什么效果，量化！)
T — Thinking     (思考：有哪些深度技术决策)
```

### 10.2 面试官典型追问链

> **Q**：\"你这个项目中最大的技术挑战是什么？\"

> **你应该准备的回答链**：
> 1. **具体是什么问题** → 性能瓶颈 / 内存泄漏 / 并发死锁
> 2. **你如何诊断的** → 用了什么工具（perf、GDB、ASan）
> 3. **有哪些候选方案** → A 方案 vs B 方案，为什么选了 A
> 4. **实现了什么效果** → 量化的指标（延迟从 100ms 降到 10ms）
> 5. **如果重新做会改进什么** → 体现反思能力

### 10.3 高频项目追问

| 追问 | 考察点 |
|------|--------|
| \"为什么选择这个方案而不是 XXX?\" | 技术决策能力 |
| \"如果流量增加 10 倍，架构哪里会出问题?\" | 扩展性思维 |
| \"你遇到过什么坑，花了多久排查?\" | 工程实战经验 |
| \"如果让你重新设计，你会改什么?\" | 反思与成长 |
| \"性能方面有没有进一步优化的空间?\" | 性能敏感度 |

### 10.4 面试官连环追问 🎯

> **Q**：\"项目中你说用了多线程，线程池参数是怎么确定的？\"

> **回答模板**：\"我们的场景是 IO 密集型（网络请求代理），理论公式是核心数 × 2，但我们实际压测后发现 8 核机器上 16 线程达到最优吞吐，超过 32 线程后因上下文切换开销反而吞吐下降。我们是通过 `perf stat` 看 `context-switches` 和 `cpu-migrations` 两个指标配合阶梯压测确定的最优值。最终选择 16 核心 + 有界队列 (2048) + CallerRuns 拒绝策略。\"

---

> **结束语**：以上四篇面经构成了 C++ 面试的完整知识体系。
> 从语言核心 [`01_Language_Core.md`](01_Language_Core.md) 到底层源码 [`02_STL_Deep_Dive.md`](02_STL_Deep_Dive.md)，
> 从并发系统 [`03_Concurrent_System.md`](03_Concurrent_System.md) 到工程实践 [`04_Engineering_Practice.md`](04_Engineering_Practice.md)。
> 每个模块都有**代码、图表、面试官连环追问**三个维度，建议按顺序学习，并在面试前逐项过一遍\"连环追问\"。
>
> **备考路径建议**：
> 1. 前两周：通读四篇笔记 + 理解每个代码示例
> 2. 第三周：手撕高频题（String类、LRU、线程池、单例）
> 3. 第四周：针对目标公司侧重点突击（腾讯→底层+网络，字节→算法+并发，阿里→STL深度）