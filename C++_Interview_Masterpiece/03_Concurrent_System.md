# 03 — 并发与系统编程：从锁到无锁的终极指南 (v2.0)

> **定位**：线程池、锁机制、原子操作、条件变量、多线程死锁排查 + Linux 系统调用与高性能网络 IO。
> **v2.0 新增**：CPU Cache / False Sharing 专题、CAS x86 vs ARM 硬件差异、memory order → 汇编对照表、Hazard Pointer 伪代码、thread_local FS/GS 寄存器实现
> **适用大厂**：字节跳动（并发最爱问）、腾讯（epoll 必问）、美团（分布式锁实战）

---

## 目录

1. [线程基础：从 std::thread 到 thread_local](#1-线程基础从-stdthread-到-thread_local)
2. [锁机制全景：互斥锁、自旋锁、读写锁、条件变量](#2-锁机制全景互斥锁自旋锁读写锁条件变量)
3. [原子操作与内存模型](#3-原子操作与内存模型)
4. [CPU Cache 与 False Sharing：并发性能的隐形杀手](#4-cpu-cache-与-false-sharing并发性能的隐形杀手)
5. [无锁编程：CAS 硬件层对比、ABA 问题与内存回收](#5-无锁编程cas-硬件层对比aba-问题与内存回收)
6. [线程池：设计、参数、优雅关闭](#6-线程池设计参数优雅关闭)
7. [多线程死锁排查](#7-多线程死锁排查)
8. [生产者-消费者模型](#8-生产者-消费者模型)
9. [单例模式的多线程安全](#9-单例模式的多线程安全)
10. [Linux I/O 多路复用：select / poll / epoll](#10-linux-io-多路复用select--poll--epoll)
11. [Reactor 模式与高性能网络框架](#11-reactor-模式与高性能网络框架)
12. [🔪 终极追杀令：并发编程的最后审判](#12--终极追杀令并发编程的最后审判)

---

## 1. 线程基础：从 std::thread 到 thread_local

### 1.1 核心 API 速查

```cpp
#include <thread>
#include <mutex>
#include <condition_variable>
#include <atomic>
#include <future>

// 创建线程
std::thread t(func, args...);
t.join();      // 等待线程结束
t.detach();    // 分离线程（后台运行，不可再 join）
t.joinable();  // 检查是否可以 join

// this_thread 命名空间
std::this_thread::sleep_for(std::chrono::milliseconds(100));
std::this_thread::yield();  // 让出 CPU 时间片
std::this_thread::get_id(); // 获取当前线程 ID

// thread_local：每个线程拥有独立的变量副本
thread_local int thread_specific_data = 0;
```

### 1.2 thread_local 底层实现：FS/GS 段寄存器

```cpp
// thread_local int x = 0;  ← 这行代码背后的硬件机制

// 在 x86-64 Linux 上：
//   ① 编译器将 thread_local 变量放入 .tdata/.tbss 段（TLS 模板）
//   ② 每个线程创建时（pthread_create → clone），内核分配独立的 TLS 块
//   ③ FS 寄存器（64位）指向当前线程的 TLS 块基址
//   ④ 访问 thread_local 变量 = FS + offset（偏移量由链接器确定）
//
//   汇编示例：
//      thread_local int x;
//      x = 42;
//      → mov eax, 42
//      → mov [fs:0xFFFFFFF8], eax   ; FS 寄存器 + 链接时常量偏移

// 动态 TLS（__tls_get_addr）：
//   对于共享库中的 thread_local 变量（偏移在链接时未知），
//   编译器生成对 __tls_get_addr 的调用：
//      lea rdi, [rip + tls_index]   ; TLS 索引结构
//      call __tls_get_addr          ; 返回当前线程该变量的地址
//      mov [rax], 42
//
//   __tls_get_addr 内部：
//     ① 读 FS 寄存器获取当前线程的 DTV (Dynamic Thread Vector) 指针
//     ② 检查 DTV[module_id] 是否已分配 → 若否，malloc + 初始化
//     ③ 返回 DTV[module_id] + offset

// 性能：静态 TLS 访问 ≈ 1-2 条指令（FS + offset）
//        动态 TLS 访问 ≈ 函数调用 + 查表（慢 10-50x）
//        因此性能敏感的 thread_local 应放在主可执行文件中（非共享库）
```

### 1.3 线程数量设置原则

| 任务类型 | 推荐线程数 | 原因 |
|----------|-----------|------|
| **CPU 密集型** | `hardware_concurrency()` | 超过核心数会导致上下文切换开销 > 并行收益 |
| **IO 密集型** | `hardware_concurrency() * 2`（或更多） | IO 等待期间 CPU 空闲，多线程可以提高利用率 |
| **混合型** | 需实际压测 | 没有万能的公式，需要 A/B 测试 |

```cpp
unsigned int n = std::thread::hardware_concurrency();
// 建议上限：实际核心数的 2-4 倍，再高上下文切换成为瓶颈
```

### 1.4 async / future / promise / packaged_task

```cpp
// std::async：最简单的异步
auto future = std::async(std::launch::async, []() {
    return heavy_computation();
});
int result = future.get();  // 阻塞等待结果

// std::packaged_task + std::future
std::packaged_task<int()> task([]() { return 42; });
auto fut = task.get_future();
std::thread t(std::move(task));  // task 不可拷贝
t.detach();
int val = fut.get();

// std::promise：在线程间传递值
std::promise<int> prom;
auto fut = prom.get_future();
std::thread([&prom]() {
    prom.set_value(42);  // 满足 future
}).detach();
int val = fut.get();
```

### 1.5 面试官连环追问 🎯

> **Q1**：`detach` 后线程对象析构了，线程还在运行吗？
>
> **回答**：**是的**。`detach` 将线程的执行与 `std::thread` 对象解绑——线程变为"守护线程"（daemon thread），独立于 `std::thread` 对象的生命周期。线程会继续运行直到其函数执行完毕。如果 `main` 函数返回，所有 detach 线程会立即被操作系统终止（不保证资源清理）。必须确保 detach 线程不访问已释放的局部变量。

> **Q2**：`std::launch::async` vs `std::launch::deferred` 的区别？
>
> **回答**：`async`：在新线程中**立即**执行，`future.get()` 等待结果。`deferred`：**延迟**执行——函数直到 `future.get()` 被调用时才在当前线程执行（惰性求值）。默认 (`std::launch::async | std::launch::deferred`) 由实现决定——不可预测，强烈建议显式指定。

> **Q3**：`thread_local` 变量的生命周期是什么？
>
> **回答**：`thread_local` 变量在**其所在线程启动时初始化**（线程专用的静态存储期），在**线程结束时析构**。同一程序的不同线程拥有该变量的独立副本，互不影响。它类似于"每个线程的 static 变量"，典型用途：线程局部缓存、per-thread logger、rand 种子。在 x86-64 上通过 FS 段寄存器实现访问。

---

## 2. 锁机制全景：互斥锁、自旋锁、读写锁、条件变量

### 2.1 互斥锁 (mutex) 与 RAII 守卫

```cpp
std::mutex mtx;

// ❌ 危险：手动 lock/unlock
mtx.lock();
// ... 如果这里抛异常 → 锁永远不会释放！
mtx.unlock();

// ✅ RAII 自动管理
{
    std::lock_guard<std::mutex> guard(mtx);  // 构造 lock, 析构 unlock
    // 临界区
}  // 自动释放（即使抛异常也安全）

// ✅ unique_lock：更灵活（可延迟加锁、转移所有权）
std::unique_lock<std::mutex> lock(mtx, std::defer_lock);  // 暂不加锁
// ... 准备工作 ...
lock.lock();    // 手动加锁
lock.unlock();  // 手动解锁（lock_guard 做不到）
// unique_lock 可以与 condition_variable 配合
```

### 2.2 互斥锁类型全家桶

| 类型 | 特点 | 场景 |
|------|------|------|
| `std::mutex` | 基本互斥锁 | 通用 |
| `std::recursive_mutex` | 同一线程可重复加锁（需要匹配解锁次数） | 递归函数中需要锁 |
| `std::timed_mutex` | 可尝试限时加锁 `try_lock_for` | 超时等待 |
| `std::recursive_timed_mutex` | recursive + timed | — |
| `std::shared_mutex` (C++17) | 读写锁：多个读线程共享，独占写 | 读多写少 |
| `std::scoped_lock` (C++17) | 同时锁定多个 mutex（避免死锁） | 需要锁多个资源 |

```cpp
// 读写锁示例 (C++17)
std::shared_mutex rw_mtx;

// 读操作：共享锁
void read() {
    std::shared_lock<std::shared_mutex> lock(rw_mtx);  // 多个线程可同时持有
    // 读取共享数据
}

// 写操作：独占锁
void write() {
    std::unique_lock<std::shared_mutex> lock(rw_mtx);  // 独占，阻塞所有读写
    // 修改共享数据
}
```

### 2.3 自旋锁：短临界区的选择

```cpp
// 简单自旋锁实现（不推荐生产使用，用 std::atomic_flag 更好）
class SpinLock {
    std::atomic_flag flag = ATOMIC_FLAG_INIT;
public:
    void lock() {
        while (flag.test_and_set(std::memory_order_acquire)) {
            // 自旋（busy-waiting）
            // x86: _mm_pause() 可以降低功耗 + 减少内存序冲突
            #if defined(__x86_64__) || defined(_M_X64)
            __builtin_ia32_pause();
            #endif
        }
    }
    void unlock() {
        flag.clear(std::memory_order_release);
    }
};
```

**互斥锁 vs 自旋锁**：

| 维度 | 互斥锁 | 自旋锁 |
|------|--------|--------|
| 等待方式 | 阻塞（线程挂起，让出 CPU） | 忙等（CPU 空转循环） |
| 内核态切换 | 是（context switch） | 否（用户态） |
| 适用场景 | 临界区较长 | 临界区极短（< 上下文切换时间） |
| 线程数建议 | 任意 | ≤ CPU 核心数 |
| 典型场景 | 文件 I/O、网络 I/O | 链表节点插入、简单计数器更新 |

### 2.4 条件变量：生产者-消费者的心跳

```cpp
std::mutex mtx;
std::condition_variable cv;
std::queue<int> data;
bool done = false;

// 消费者
void consumer() {
    while (true) {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [] { return !data.empty() || done; });
        // wait 等价于：while (!pred()) { cv.wait(lock); }
        // 即「自动释放锁→被 notify 唤醒→重新获取锁→检查条件」
        // 防止虚假唤醒 (spurious wakeup)

        if (done && data.empty()) break;
        int val = data.front();
        data.pop();
        lock.unlock();  // 可选：处理数据期间释放锁
        process(val);
    }
}

// 生产者
void producer() {
    for (int i = 0; i < 100; ++i) {
        {
            std::lock_guard<std::mutex> lock(mtx);
            data.push(i);
        }
        cv.notify_one();  // 唤醒一个等待线程
    }
    {
        std::lock_guard<std::mutex> lock(mtx);
        done = true;
    }
    cv.notify_all();  // 通知所有消费者结束
}
```

### 2.5 面试官连环追问 🎯

> **Q1**：条件变量的 `wait` 为什么要传入一个谓词？什么是"虚假唤醒"？
>
> **回答**：**虚假唤醒**（spurious wakeup）是指线程在没有收到 `notify` 的情况下被操作系统从 `wait` 中唤醒。这是 POSIX 标准允许的行为，源于某些操作系统调度优化的副作用。因此 `wait` 必须放在 `while` 循环中（或传入谓词），在唤醒后重新检查条件是否真正满足。C++ 传入谓词的 `wait(lock, pred)` 等价于 `while (!pred()) wait(lock);`。

> **Q2**：`notify_one` vs `notify_all` 如何选择？
>
> **回答**：**当所有等待线程都在竞争同一个资源时**，用 `notify_one`——只唤醒一个线程获取资源即可，唤醒多个会导致"惊群效应"（thundering herd），大量线程竞争失败又回去休眠，浪费 CPU。**当等待线程等待不同条件时**，用 `notify_all`——因为不确定哪个线程的条件满足了，需要全部唤醒各自检查。生产者-消费者模式中，多个消费者等待同一队列时用 `notify_one`（一个元素只能被一个消费者取走）。

> **Q3**：`std::scoped_lock` 如何防止死锁？和 `std::lock` 有什么区别？
>
> **回答**：两者都使用相同的死锁避免算法（`std::lock` 使用 try-and-back-off 策略同时锁定多个 mutex），`std::scoped_lock`（C++17）是 `std::lock_guard` 的升级版——既支持多 mutex 同时锁定，又是 RAII 自动释放，是推荐的现代写法。`std::lock` 只是锁定操作本身，不含 RAII 语义，必须配合 `lock_guard` 的 `std::adopt_lock` 使用。

---

## 3. 原子操作与内存模型

### 3.1 为什么需要 atomic？

```cpp
// ❌ 即使是简单的 ++ 也不是原子的
int counter = 0;
// counter++ 汇编层面分为三步：load → inc → store
// 两个线程同时执行 → 可能互相覆盖 → 结果不是 +2 而是 +1

// ✅ atomic 保证原子性
std::atomic<int> counter{0};
counter.fetch_add(1);  // 原子操作，结果必定正确
```

### 3.2 atomic 的常用操作

| 操作 | 说明 |
|------|------|
| `load()` | 原子读取 |
| `store(val)` | 原子写入 |
| `exchange(val)` | 原子交换（返回旧值） |
| `compare_exchange_weak(expected, desired)` | CAS：若当前值==expected，设为desired |
| `compare_exchange_strong(expected, desired)` | 同上但不会出现伪失败 |
| `fetch_add(n)` | 原子+ n，返回旧值 |
| `fetch_sub(n)` | 原子- n，返回旧值 |
| `is_lock_free()` | 是否无锁实现 |

```cpp
// 简易 CAS 自旋锁
std::atomic<bool> locked{false};

void lock_cas() {
    bool expected = false;
    while (!locked.compare_exchange_weak(expected, true,
            std::memory_order_acquire)) {
        expected = false;  // weak 可能伪失败，需重置 expected
    }
}

void unlock_cas() {
    locked.store(false, std::memory_order_release);
}
```

### 3.3 C++ Memory Order：全部 6 种 + 对应的 x86 汇编

```
   最强保证（代价最高）
   ↓
┌──────────────────────────────────────────────────────────┐
│ memory_order_seq_cst │ 全局顺序一致性      │ x86: mfence  │
├──────────────────────────────────────────────────────────┤
│ memory_order_acq_rel │ acquire + release   │ x86: 单一    │
│                      │                     │ mov + 编译屏障│
├──────────────────────┼─────────────────────┼──────────────┤
│ memory_order_acquire │ 读侧栅栏             │ x86: 无指令 │
│                      │ (之后的读写不能重排  │ (x86 硬件    │
│                      │  到此读之前)         │  天然保证)  │
├──────────────────────┼─────────────────────┼──────────────┤
│ memory_order_release │ 写侧栅栏             │ x86: 无指令 │
│                      │ (之前的读写不能重排  │ (x86 硬件    │
│                      │  到此写之后)         │  天然保证)  │
├──────────────────────┼─────────────────────┼──────────────┤
│ memory_order_consume │ 数据依赖排序         │ 已废弃      │
├──────────────────────┼─────────────────────┼──────────────┤
│ memory_order_relaxed │ 仅保证原子性         │ x86: 无指令 │
│                      │ 不保证顺序          │ 无额外开销  │
└──────────────────────────────────────────────────────────┘
   最弱保证（代价最低）
   ↓
```

**x86 的内存模型特性**：x86-64 是 **TSO (Total Store Order)** 模型——硬件天然保证"Load 不重排到 Load 之前"和"Store 不重排到 Store 之前"，因此 `acquire` 和 `release` 在 x86 上是**零指令开销**的。只有 `seq_cst` 需要 `mfence`（全屏障）或 `xchg`（隐式锁总线）指令。

**ARM/PowerPC 的内存模型**：弱内存模型——几乎任何操作都可能被 CPU 重排。`acquire` 需要 `dmb ishld`（ARM）或 `lwsync`（PowerPC），`release` 需要 `dmb ish` 或 `sync`。`seq_cst` 需要完整内存屏障。这就是为什么 `memory_order_relaxed` 在 ARM 上能带来显著的性能提升。

```cpp
// 场景 1：计数器 → relaxed 足够
std::atomic<size_t> hit_count{0};
void onRequest() {
    hit_count.fetch_add(1, std::memory_order_relaxed);  // x86: lock inc [mem]
    // ARM: ldrex; add; strex; (无 dmb)
}

// 场景 2：flag 通知 → release/acquire
std::atomic<bool> data_ready{false};
int shared_data = 0;  // 非原子变量

// 写线程
void writer() {
    shared_data = 42;                            // ① 写入数据
    data_ready.store(true, memory_order_release);// ② 发布 flag
    // x86 asm: mov BYTE PTR [data_ready], 1  ← 无额外屏障! (mov 自带 StoreStore 序)
    // ARM asm: stlr  ← Store-Release 指令
    // release 保证 ① happens-before ②（不会重排到后面）
}

// 读线程
void reader() {
    while (!data_ready.load(memory_order_acquire));  // ③ 获取 flag
    // x86 asm: mov al, [data_ready]; test al,al; jz loop  ← 无额外屏障!
    // ARM asm: ldar  ← Load-Acquire 指令
    // acquire 保证 ③ happens-before ④（不会重排到前面）
    assert(shared_data == 42);                       // ④ 必定能看到 ① 的结果
}

// 场景 3：双检锁单例 → acq_rel
class DoubleCheckedSingleton {
    static std::atomic<DoubleCheckedSingleton*> instance;
    static std::mutex mtx;
public:
    static DoubleCheckedSingleton* getInstance() {
        DoubleCheckedSingleton* tmp = instance.load(std::memory_order_acquire);
        if (tmp == nullptr) {
            std::lock_guard<std::mutex> lock(mtx);
            tmp = instance.load(std::memory_order_relaxed);
            if (tmp == nullptr) {
                tmp = new DoubleCheckedSingleton();
                instance.store(tmp, std::memory_order_release);
            }
        }
        return tmp;
    }
};
```

### 3.4 面试官连环追问 🎯

> **Q1**：`compare_exchange_weak` 和 `compare_exchange_strong` 的区别？
>
> **回答**：两者都是 CAS（Compare-And-Swap）。`weak` 可能出现**伪失败**（spurious failure）——即使当前值等于 expected 也可能返回 false（在某些平台如 ARM 的 LL/SC 指令上）。`strong` 保证只在值不相等时才失败。`weak` 用在循环中更高效（因为循环本身已处理失败），`strong` 用在非循环场景中更简单。

> **Q2**：什么时候用 `memory_order_relaxed`？
>
> **回答**：当**只关心原子性，不关心与其他操作的顺序**时。典型场景：全局计数器（如请求计数）、引用计数的原子增减（`shared_ptr` 内部）、统计信息收集。relaxed 在 x86 上几乎无额外开销（x86 本身就提供较强的硬件内存序），在 ARM/PowerPC 等弱内存平台上可以避免昂贵的内存屏障指令。

> **Q3**：C++11 的 `memory_order_consume` 还存在吗？为什么？
>
> **回答**：`memory_order_consume` 的概念是"仅排序对同一原子变量有数据依赖的操作"，比 `acquire` 更弱且更高效。但所有主流编译器都将它当作 `acquire` 来实现（因为正确实现 consume 的编译器优化难度极大）。C++17 标准委员会将其声明为不推荐使用，可能在 C++26 中移除。

---

## 4. CPU Cache 与 False Sharing：并发性能的隐形杀手

### 4.1 Cache Line 基础 — 一切的根本

```
  ┌─────────────────────────────────────────────────┐
  │            现代 CPU Cache 层级                   │
  │                                                 │
  │  Core 0                    Core 1               │
  │  ┌─────────┐               ┌─────────┐          │
  │  │ L1 DCache│               │ L1 DCache│          │
  │  │ 32KB 8路 │               │ 32KB 8路 │          │
  │  │ Lat:4cyc │               │ Lat:4cyc │          │
  │  └────┬─────┘               └────┬─────┘          │
  │       │                          │               │
  │  ┌────┴─────┐               ┌────┴─────┐          │
  │  │ L2 Cache │               │ L2 Cache │          │
  │  │ 256KB    │               │ 256KB    │          │
  │  │ Lat:12cyc│               │ Lat:12cyc│          │
  │  └────┬─────┘               └────┬─────┘          │
  │       └──────────┬──────────────┘                │
  │           ┌──────┴──────┐                        │
  │           │ L3 (Shared) │                        │
  │           │ 8-32MB      │                        │
  │           │ Lat:~40cyc  │                        │
  │           └─────────────┘                        │
  └─────────────────────────────────────────────────┘

  Cache Line = 64 bytes (x86/ARM) 或 128 bytes (Apple M-series)
  每次内存访问都抓取整个 cache line — 这是 false sharing 的根源
```

### 4.2 False Sharing — 并发最隐蔽的性能陷阱

```cpp
// ❌ False Sharing 演示
struct UnpaddedCounters {
    std::atomic<int> counter_a;  // offset 0-3
    std::atomic<int> counter_b;  // offset 4-7
    // counter_a 和 counter_b 在同一 64 字节 cache line 中！
};

UnpaddedCounters counters;

// Thread A (Core 0): 循环 10^7 次 counter_a++
// Thread B (Core 1): 循环 10^7 次 counter_b++
// 预期：二者"互不相关"，应该接近完美并行
// 实际：性能下降 10-100x！

// 原因（MESI 协议视角）：
//   T0: Core 0 读 counter_a → cache line 在 Core 0 L1 为 Shared 状态
//   T1: Core 0 写 counter_a → 必须先 Invalidate Core 1 的 cache line
//        → Core 0 cache line 变为 Modified 状态
//   T2: Core 1 读 counter_b → Cache Miss! → 从 Core 0 偷取数据（cache-to-cache transfer）
//        → Core 0 cache line 回到 Shared，Core 1 cache line 为 Shared
//   T3: Core 1 写 counter_b → 必须先 Invalidate Core 0 的 cache line
//        → Core 1 cache line 变为 Modified 状态
//   T4: 回到 T0，无限循环 —— 这是 "乒乓效应"
//
//   每次操作 ~100-300 CPU cycles 的 cache line bounce
//   而非 1-2 cycles 的本地 L1 访问
//   性能差异：~10-100x！
```

### 4.3 解决方案：Cache Line Padding

```cpp
// ✅ 方案 1：alignas 强制对齐 + padding 填充
struct alignas(64) PaddedCounter {
    std::atomic<int> value;
    // 自动 padding 到 64 字节 — 独占整个 cache line
};

struct FixedCounters {
    alignas(64) std::atomic<int> counter_a;   // 独占 cache line #1
    alignas(64) std::atomic<int> counter_b;   // 独占 cache line #2
    // sizeof = 128，性能接近完美并行
};

// ✅ 方案 2：手动 padding（跨编译器兼容）
struct ManualPaddedCounter {
    std::atomic<int> value;
    // 填充到 64 字节
    char __padding[64 - sizeof(std::atomic<int>)];
    static_assert(sizeof(ManualPaddedCounter) == 64);
};

// ✅ 方案 3：C++17 标准常量（推荐但需注意实际值）
struct StandardPaddedCounter {
    alignas(std::hardware_destructive_interference_size) std::atomic<int> value;
    // 注：此常量在 libstdc++ 中定义为 64，在 LLVM libc++ 中也是 64
    //     Apple M 系列 L1 cache line = 128B，但这个常量仍返回 64
    //     这是 ABI 兼容性考量，实际生产代码可能需要手动调整
};
```

### 4.4 perf 检测 False Sharing

```bash
# 编译两个版本对比
g++ -O2 -std=c++20 -pthread unpadded.cpp -o unpadded
g++ -O2 -std=c++20 -pthread padded.cpp -o padded

# perf stat 对比
perf stat -e cache-misses,cache-references,L1-dcache-load-misses \
    -e LLC-load-misses,cycles,instructions \
    -r 5 ./unpadded

perf stat -e cache-misses,cache-references,L1-dcache-load-misses \
    -e LLC-load-misses,cycles,instructions \
    -r 5 ./padded

# 关键指标：
#   unpadded: cache-misses 极高 + IPC (instructions/cycle) 极低 (< 0.5)
#   padded:   cache-misses 极低 + IPC 接近理论峰值 (2-4)
# 这个差异直接证明 false sharing 的存在
```

### 4.5 MESI 协议与 `LOCK` 前缀的交互

```
  ┌───────────────────────────────────────────────┐
  │        LOCK CMPXCHG 对 Cache Line 的影响        │
  │                                               │
  │  ① LOCK 前缀锁定 cache line（不是整个总线！）    │
  │     → 现代 CPU 使用 cache locking               │
  │  ② 其他核心对该 cache line 的读写被阻塞         │
  │  ③ 该核心的 store buffer 必须先排空             │
  │     → 隐含了完整的内存屏障                      │
  │  ④ LOCK 操作完成后，cache line 被 Invalidate    │
  │     → 其他核心的下次访问触发 cache miss         │
  │                                               │
  │  结论：LOCK 前缀 = 原子性保证 + StoreLoad 屏障  │
  │        这也是为什么 seq_cst CAS/fetch_add       │
  │        在 x86 上天然很强                        │
  └───────────────────────────────────────────────┘
```

### 4.6 面试官连环追问 🎯

> **Q1**：为什么锁实现中的 `atomic_flag` 自旋锁也应该对齐到 cache line？
>
> **回答**：如果锁变量与其他热点数据共享 cache line，每次锁的 `test_and_set`（LOCK 前缀操作）会 invalidate 其他核心的 cache line，导致持有锁的核心和等待锁的核心之间产生 cache line bounce。这不仅影响自旋等待的性能，还可能拖慢相邻数据的访问——这就是"锁数据污染"。最佳实践：每个锁变量独占一个 cache line（`alignas(64)`）。

> **Q2**：`std::hardware_destructive_interference_size` 和 `std::hardware_constructive_interference_size` 有什么区别？
>
> **回答**：前者（destructive）是"两个变量应该分开至少这么多字节以避免 false sharing"的最小间距——用于 padding 隔离。后者（constructive）是"两个变量应该放在这么多字节内以便在同一个 cache line 中"的最大间距——用于提升 true sharing 的 locality。两者在 x86-64 libstdc++ 中都定义为 64。

> **Q3**：在 NUMA 架构下，false sharing 的影响会更严重吗？
>
> **回答**：是的。NUMA 架构下，不同 socket 的 CPU 核心共享 L3 cache 的代价不同——跨 socket 的 cache line bounce 需要经过 UPI/QPI 互联总线，延迟远高于同 socket 内的 cache-to-cache transfer（80-150ns vs 20-40ns）。如果在 NUMA 系统上将两个频繁竞争的线程绑定到不同 socket 的核心，false sharing 的开销会进一步放大 2-4 倍。

---

## 5. 无锁编程：CAS 硬件层对比、ABA 问题与内存回收

### 5.1 无锁栈 (Treiber Stack) — 最简无锁原型

```cpp
template<typename T>
class LockFreeStack {
    struct Node {
        T data;
        Node* next;
        Node(const T& d) : data(d), next(nullptr) {}
    };
    std::atomic<Node*> head_{nullptr};

public:
    void push(const T& val) {
        Node* new_node = new Node(val);
        new_node->next = head_.load(std::memory_order_relaxed);
        while (!head_.compare_exchange_weak(
                new_node->next, new_node,
                std::memory_order_release,
                std::memory_order_relaxed)) {
            // CAS 失败 → new_node->next 已被更新为最新的 head
            // 循环重试
        }
    }

    bool pop(T& result) {
        Node* old_head = head_.load(std::memory_order_relaxed);
        while (old_head && !head_.compare_exchange_weak(
                old_head, old_head->next,
                std::memory_order_acquire,
                std::memory_order_relaxed)) {
            // CAS 失败 → 重新读取
        }
        if (!old_head) return false;
        result = old_head->data;
        // 💀 ABA 问题：这里能安全 delete 吗？
        delete old_head;
        return true;
    }
};
```

### 5.2 CAS 硬件实现：x86 vs ARM 的本质差异

```
  ╔══════════════════════════════════════════════════════╗
  ║            CAS 硬件指令对比                           ║
  ╠═══════════╦══════════════════╦═══════════════════════╣
  ║           ║ x86-64           ║ ARM (ARMv7+)          ║
  ╠═══════════╬══════════════════╬═══════════════════════╣
  ║ 指令      ║ LOCK CMPXCHG     ║ LDREX / STREX (LL/SC) ║
  ║ 原理      ║ 原子比较+交换    ║ 加载-监控 / 条件存储   ║
  ║ 粒度      ║ 锁 cache line    ║ 标记内存地址 (exclusive║
  ║           ║ (cache locking)  ║  monitor)            ║
  ║ 模式      ║ 单指令           ║ 双指令（可被中断打断） ║
  ║ ABA 影响  ║ 不关心（只看值） ║ 天然抗拒 ABA          ║
  ║           ║                  ║ (地址被 touch 即失败) ║
  ║ weak CAS  ║ 永不伪失败       ║ 频繁伪失败！          ║
  ║ 伪失败    ║ (强内存模型)     ║ (任何异常/中断都失败) ║
  ╚═══════════╩══════════════════╩═══════════════════════╝
```

```cpp
// x86-64 CAS 底层汇编：
// compare_exchange_weak(expected, desired)
//   → mov eax, expected        ; 将期望值放入 eax
//   → lock cmpxchg [mem], reg  ; 原子：if [mem]==eax then [mem]=reg else eax=[mem]
//   → sete al                   ; 根据 ZF 标志位设置返回值
//
//   lock 前缀所做的工作：
//     ① 锁定 cache line（通过 MESI 协议中的 RFO 消息）
//     ② 排空 store buffer（充当完整内存屏障）
//     ③ 阻止其他核心同时访问该 cache line
//     ④ 操作完成后释放锁 → 其他核心的读取会 miss

// ARM LL/SC (Load-Link / Store-Conditional) 底层汇编：
// compare_exchange_weak(expected, desired)
//   retry:
//     ldrex r0, [mem]           ; Load-Exclusive: 加载值并标记地址
//     cmp   r0, expected        ; 比较
//     bne   fail                ; 不相等 → 失败
//     strex r1, desired, [mem]  ; Store-Conditional: 仅在标记未被打破时存储
//     cmp   r1, #0              ; r1=0 表示成功，r1=1 表示被打破（任何中断/异常）
//     bne   retry               ; 伪失败 → 重试
//   fail:
//
//   STREX 失败条件（伪失败的根源）：
//     · 另一个核心 STREX 到同一地址（这是真正的竞争失败）
//     · 任何异常/中断/上下文切换（这是"伪失败"——即使值匹配！）
//     · 同一个核心的任何其他 STREX 指令
//     · 某些实现中，load 了同一 cache line 的其他数据
//
//   这解释了为什么 weak CAS 在 ARM 上必须放在循环中！
```

### 5.3 ABA 问题：无锁编程的最大陷阱

```
  Thread 1 读取 head = A，记录 A->next = B
  Thread 2 pop A, pop B, push A'
  Thread 1 CAS(head, A, B) → 成功！但 head 已经不是原来的 A 了
  → 实际上跳过了链表中间的节点 → 数据丢失/链表断裂

  ABA Problem 图解：
  时刻 T1: head → A → B → C
  时刻 T2: Thread1 读到 A 和 A->next = B
  时刻 T3: Thread2 pop A, pop B, push A' → head → A' → C
  时刻 T4: Thread1 CAS 比较 head==A? YES! (因为 A 的地址被重用了)
          → head → B (但 B 已经被释放了！)
  结果：head 指向已释放的内存 → use-after-free
```

**解决方案**：

| 方案 | 原理 | 适用性 |
|------|------|--------|
| **Tagged Pointer** | 指针高位加版本号，64 位系统可用 | ✅ 最常用 (x86-64) |
| **Hazard Pointer** | 线程声明正在使用哪些指针，延迟释放 | ✅ C++26 提案 |
| **RCU (Read-Copy-Update)** | 读不加锁，写时复制 | ✅ 内核常用 |
| **Epoch-Based Reclamation** | 按 epoch 批次回收内存 | ✅ 高吞吐 |

### 5.4 Hazard Pointer 伪代码实现

```cpp
// ===== Hazard Pointer 核心思想 =====
// 每个线程维护一组"危险指针"（hazard pointer）
// 访问节点前，将其地址写入自己的 HP 列表（"我在用这个节点"）
// 释放节点前，检查所有线程的 HP 列表（"还有人在用吗？"）
// 如果有人在用 → 延迟释放；如果没人 → 安全释放

// ===== 全局共享结构 =====
constexpr size_t MAX_HP_PER_THREAD = 2;  // 每个线程最多保护 2 个指针
constexpr size_t MAX_THREADS = 100;
constexpr size_t RETIRE_THRESHOLD = 100; // 攒够 100 个退役节点一批处理

// 全局 Hazard Pointer 数组（所有线程可见）
std::atomic<Node*> hazard_pointers[MAX_THREADS][MAX_HP_PER_THREAD];

// 每个线程私有的退役列表
thread_local std::vector<Node*> retire_list;

// ===== 线程注册自己的 HP（使用前调用）=====
void protect(size_t hp_index, std::atomic<Node*>& ptr) {
    Node* node;
    do {
        node = ptr.load(std::memory_order_acquire);
        // 将 node 写入我的 HP 槽
        hazard_pointers[my_thread_id][hp_index].store(
            node, std::memory_order_release);
    } while (node != ptr.load(std::memory_order_acquire));
    // 必须反复验证！如果在 store HP 和重新读取之间 ptr 变了，
    // 说明 node 可能已被其他线程释放 → 重试
}

// ===== 退役一个节点（删除前调用）=====
void retire(Node* node) {
    retire_list.push_back(node);
    if (retire_list.size() >= RETIRE_THRESHOLD) {
        scan_and_reclaim();  // 批次扫描
    }
}

// ===== 扫描所有线程的 HP 并回收安全节点 =====
void scan_and_reclaim() {
    // 第一步：收集所有线程的 Hazard Pointers
    std::unordered_set<Node*> in_use;
    for (size_t t = 0; t < MAX_THREADS; ++t) {
        for (size_t h = 0; h < MAX_HP_PER_THREAD; ++h) {
            Node* hp = hazard_pointers[t][h].load(std::memory_order_acquire);
            if (hp) in_use.insert(hp);
        }
    }

    // 第二步：释放不在 in_use 集合中的退役节点
    auto it = retire_list.begin();
    while (it != retire_list.end()) {
        if (!in_use.count(*it)) {
            delete *it;  // 安全释放！
            it = retire_list.erase(it);
        } else {
            ++it;  // 还在被使用，保留到下轮
        }
    }
}

// ===== 使用示例：pop 操作的完整流程 =====
bool pop_protected(T& result) {
    // 步骤 1：用 HP 槽 0 保护 head
    protect(0, head_);

    Node* old_head = hazard_pointers[my_thread_id][0].load();
    if (!old_head) return false;

    // 步骤 2：CAS 弹出 head
    if (head_.compare_exchange_strong(old_head, old_head->next,
            std::memory_order_acquire)) {
        result = old_head->data;

        // 步骤 3：清除 HP（不再保护）
        hazard_pointers[my_thread_id][0].store(nullptr);

        // 步骤 4：退役节点（延迟释放）
        retire(old_head);
        return true;
    }

    // CAS 失败 → 清除 HP 重试
    hazard_pointers[my_thread_id][0].store(nullptr);
    return false;  // 或循环重试
}
```

### 5.5 无锁 vs 有锁：何时该用？

| 条件 | 无锁 | 有锁 |
|------|------|------|
| 临界区极短 | ✅ | 上下文切换开销 > 自旋 |
| 高竞争 | ❌ 大量 CAS 重试 | ✅ 阻塞反而更高效 |
| 实时性要求 | ✅ 无阻塞等待 | ❌ mutex 可能被抢占 |
| 实现复杂度 | ❌ ABA/回收极复杂 | ✅ 简单 |
| 调试 | ❌ 极难重现 bug | ✅ 相对容易 |

**面试金句**："无锁不等于更快。在高竞争场景下，锁机制通过阻塞避免浪费 CPU，而无锁会陷入 CAS 重试风暴。选择取决于竞争程度、临界区长度和延迟需求。"

### 5.6 面试官连环追问 🎯

> **Q1**：为什么 Hazard Pointer 能解决 ABA 问题，而不是防止 ABA 发生？
>
> **回答**：Hazard Pointer 不防止 ABA 发生——它解决的是 ABA 导致的 **use-after-free** 问题。原理：线程释放节点前检查所有线程的 HP 列表——只要某个线程声明了"我在使用地址 A 的节点"，地址 A 就不能被释放（即使 ABA 导致 CAS 使用了一个不再正确的 A，也不会访问到已释放内存）。内存回收被安全地延迟到无人引用之时。

> **Q2**：x86 上 `std::atomic` 默认是 lock-free 的吗？
>
> **回答**：取决于类型大小。x86-64 上：≤ 8 字节的类型（`int`、`long`、`T*`）是 lock-free 的；16 字节的类型（如 `__int128` 或带 tag 的指针对）需要 CMPXCHG16B 指令（所有现代 x86-64 CPU 都支持）。`std::atomic<std::string>` 不是 lock-free 的（内部用互斥锁实现）。可以用 `is_lock_free()` 或 `is_always_lock_free` (C++17) 检查。

> **Q3**：为什么 ARM 的 LL/SC 机制天然抗拒 ABA 问题？
>
> **回答**：LL/SC 机制下，STREX 成功的前提是**地址自从上次 LDREX 以来没有被任何其他写入触碰过**——即使新写入的值与旧值完全相同。这意味着如果 ABA 发生了（Thread 2 pop A, push A'），地址 A 被写入过（push A'），Thread 1 的 STREX 会失败。相比之下，x86 的 LOCK CMPXCHG 只看值，不在乎地址是否被"触碰"过。因此 ARM 在硬件层面就能检测 ABA 并拒绝它——这恰恰是 `compare_exchange_weak` 存在的原因：它映射到 LL/SC 的自然行为。

---

## 6. 线程池：设计、参数、优雅关闭

### 6.1 完整线程池实现

```cpp
class ThreadPool {
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex mtx;
    std::condition_variable cv;
    std::atomic<bool> stop_flag{false};

public:
    explicit ThreadPool(size_t num_threads = std::thread::hardware_concurrency()) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(mtx);
                        cv.wait(lock, [this] {
                            return stop_flag.load() || !tasks.empty();
                        });
                        if (stop_flag.load() && tasks.empty()) return;
                        task = std::move(tasks.front());
                        tasks.pop();
                    }
                    task();  // 不在锁内执行任务！否则并发性为 0
                }
            });
        }
    }

    // 提交任务，返回 future 以获取结果（C++17）
    template<typename F, typename... Args>
    auto submit(F&& f, Args&&... args)
        -> std::future<std::invoke_result_t<F, Args...>>
    {
        using return_type = std::invoke_result_t<F, Args...>;
        auto task = std::make_shared<std::packaged_task<return_type()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );
        std::future<return_type> res = task->get_future();
        {
            std::lock_guard<std::mutex> lock(mtx);
            if (stop_flag) throw std::runtime_error("ThreadPool stopped");
            tasks.emplace([task]() { (*task)(); });
        }
        cv.notify_one();
        return res;
    }

    // 优雅关闭
    ~ThreadPool() {
        stop_flag.store(true);
        cv.notify_all();
        for (auto& w : workers) {
            if (w.joinable()) w.join();
        }
    }
};
```

### 6.2 线程池关键设计决策

| 参数 | 考量 |
|------|------|
| **核心线程数** | CPU 密集型 = `hardware_concurrency()`；IO 密集型 = 更多 |
| **最大线程数** | 可选，超出后任务入队（而非创建新线程） |
| **任务队列** | 有界队列（背压）+ 拒绝策略；无界队列（可能 OOM） |
| **拒绝策略** | 直接抛异常 / 调用者运行 / 丢弃最旧 / 丢弃最新 |
| **优雅关闭** | `stop_flag` → `notify_all` → `join`；确保所有已提交的任务执行完毕 |

### 6.3 面试官连环追问 🎯

> **Q1**：为什么线程池要在锁外执行 `task()`？
>
> **回答**：如果 task 在持有 mutex 的情况下执行，整个线程池的并发性退化为 1——每个 worker 从队列取任务都互斥，执行期间也互斥，其他所有 worker 都在等这个锁。退一步说，如果 task 中也有同步操作，极易造成死锁。任务执行应该在最小锁范围内完成——只保护队列操作的原子性。

> **Q2**：`stop_flag` 为什么用 `atomic<bool>` 而不是普通 `bool` + mutex？
>
> **回答**：`stop_flag` 在析构函数中被写入，在 worker 线程的 `wait` 谓词中被读取——这是一个典型的多线程读写场景。用 `atomic` 避免了额外的 mutex 开销（在这个场景下 mutex 的锁竞争与条件变量内部的锁重叠），且语义更清晰。对于简单 flag，atomic 是最佳的。

> **Q3**：`submit` 返回 `std::future`，但如果调用者丢弃了 future 呢？
>
> **回答**：`std::future` 的析构函数会阻塞等待 `std::packaged_task` 完成（若 future 是异步任务的最后一个引用）。这意味着如果调用者丢弃 future，`submit` 实际上变成了同步操作——调用者线程被阻塞直到任务完成。这是 `std::future` 的设计约束。如果想真正的 fire-and-forget，需要自己实现无 future 版本的 `execute` 方法。

---

## 7. 多线程死锁排查

### 7.1 死锁必需四条件

```
  ┌────────────────────────────────────────────────┐
  │             死锁的四个必要条件                    │
  │ ① 互斥：资源一次只能被一个线程使用                │
  │ ② 持有并等待：持有自身资源同时等待其他资源        │
  │ ③ 不可抢占：他人持有的资源不能被强制释放          │
  │ ④ 循环等待：形成资源等待环                       │
  │                                                 │
  │  破坏任意一个 → 死锁不可能发生                     │
  └────────────────────────────────────────────────┘
```

### 7.2 死锁预防策略

```cpp
// 策略 1：统一加锁顺序（破坏"循环等待"）
std::mutex mtx_a, mtx_b;

// ✅ 始终按 A → B 顺序加锁
void safe_transfer() {
    std::scoped_lock lock(mtx_a, mtx_b);  // C++17：原子地锁多个
    // 或手动：std::lock(mtx_a, mtx_b) + adopt_lock
}

// 策略 2：try_lock + 回退（破坏"持有并等待"）
void try_and_backoff() {
    while (true) {
        mtx_a.lock();
        if (mtx_b.try_lock()) return;  // 成功
        mtx_a.unlock();
        std::this_thread::sleep_for(std::chrono::milliseconds(1));  // 退避
    }
}

// 策略 3：超时机制（破坏"不可抢占"）
std::timed_mutex tmtx_a, tmtx_b;
void timed_lock() {
    auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(1);
    if (!tmtx_a.try_lock_until(deadline)) { /* 超时处理 */ return; }
    if (!tmtx_b.try_lock_until(deadline)) { tmtx_a.unlock(); return; }
    // 安全临界区
    tmtx_a.unlock(); tmtx_b.unlock();
}
```

### 7.3 死锁排查工具

| 工具 | 平台 | 能力 |
|------|------|------|
| **GDB `thread apply all bt`** | Linux | 查看所有线程调用栈，找出阻塞位置 |
| **Valgrind Helgrind** | Linux | 检测数据竞争、锁顺序违规 |
| **TSan (ThreadSanitizer)** | Linux/macOS | `-fsanitize=thread`，运行时检测 |
| **strace** | Linux | 追踪系统调用，定位 `futex` 等待 |
| **pstack** | Linux | 打印进程所有线程堆栈 |
| **WinDbg `!locks`** | Windows | 查看临界区/锁持有情况 |
| **`/proc/<pid>/task/*/wchan`** | Linux | 查看每个线程在内核态等待什么 |

```bash
# 快速死锁定位脚本
gdb -p $(pidof your_app) -batch \
    -ex "thread apply all bt" 2>/dev/null | \
    grep -A3 "pthread_mutex\|futex\|__lll_lock"
```

### 7.4 面试官连环追问 🎯

> **Q1**：程序在生产环境卡死了，但无法 attach GDB，如何快速确认是否为死锁？
>
> **回答**：查看进程状态：`ps -eo pid,stat,wchan:32,comm | grep <pid>`。若所有线程状态为 `D`（不可中断睡眠）且 `wchan`（等待通道）都是 `futex` 相关，大概率是死锁。再检查 CPU 使用率——死锁通常伴随 CPU 0%，活锁则 CPU 100%。最后 `/proc/<pid>/task/*/stack` 可查看内核栈（不需 ptrace）。

> **Q2**：死锁和活锁的区别？
>
> **回答**：**死锁**：线程相互等待对方释放资源，全部阻塞不执行任何操作。**活锁**：线程在不断"让步"或"重试"，都在活动但无法推进——典型场景是两个人在走廊中互相让路却总撞到同侧。活锁的 CPU 使用率很高但吞吐量为 0。死锁用 `std::lock` 的多锁同时获取解决，活锁用随机退避解决。

> **Q3**：`std::scoped_lock` 和 `std::lock` 的实现是如何避免死锁的？
>
> **回答**：使用**try-and-back-off**算法（非标准规定，但主流实现如 libstdc++ 和 libc++ 均如此）：对所有锁依次 `try_lock`，若某个失败，则释放之前所有已获取的锁，重新开始。通常会加入轻微随机退避避免活锁。某些实现还使用更精细的死锁避免算法如层级锁（每个锁有优先级，只能按优先级顺序加锁）。

---

## 8. 生产者-消费者模型

```cpp
// 完整的线程安全有界队列
template<typename T>
class BoundedBlockingQueue {
    std::queue<T> queue;
    std::mutex mtx;
    std::condition_variable not_full;
    std::condition_variable not_empty;
    size_t max_size;

public:
    explicit BoundedBlockingQueue(size_t max) : max_size(max) {}

    void push(T item) {
        std::unique_lock<std::mutex> lock(mtx);
        not_full.wait(lock, [this] { return queue.size() < max_size; });
        queue.push(std::move(item));
        lock.unlock();
        not_empty.notify_one();
    }

    T pop() {
        std::unique_lock<std::mutex> lock(mtx);
        not_empty.wait(lock, [this] { return !queue.empty(); });
        T item = std::move(queue.front());
        queue.pop();
        lock.unlock();
        not_full.notify_one();
        return item;
    }
};
```

### 面试官连环追问 🎯

> **Q1**：为什么 `notify_one` 要在 `unlock` 之后调用？
>
> **回答**：如果先 notify 再 unlock，被唤醒的线程可能立即尝试获取锁但发现锁还被持有 → 立即阻塞（"hurry up and wait"效应），浪费一次上下文切换。先 unlock 后 notify 可以避免这个问题。虽然标准允许两种顺序，但先 unlock 后 notify 通常性能更好。

---

## 9. 单例模式的多线程安全

```cpp
// Meyers' Singleton — C++11 起线程安全
class Singleton {
public:
    static Singleton& getInstance() {
        static Singleton instance;  // C++11: magic static — 线程安全初始化
        return instance;
    }
private:
    Singleton() = default;
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
};

// 双检锁 (DCLP) — C++11 起正确
class DCLPSingleton {
    static std::atomic<DCLPSingleton*> instance;
    static std::mutex mtx;
public:
    static DCLPSingleton* getInstance() {
        DCLPSingleton* tmp = instance.load(std::memory_order_acquire);
        if (tmp == nullptr) {
            std::lock_guard<std::mutex> lock(mtx);
            tmp = instance.load(std::memory_order_relaxed);
            if (tmp == nullptr) {
                tmp = new DCLPSingleton();
                instance.store(tmp, std::memory_order_release);
            }
        }
        return tmp;
    }
};
```

### 面试官连环追问 🎯

> **Q1**：C++11 为什么能保证 `static` 局部变量初始化的线程安全？
>
> **回答**：C++11 标准规定：如果多个线程同时进入同一个 `static` 局部变量的初始化，只有一个线程执行初始化，其他线程阻塞等待初始化完成。编译器通过隐藏的 atomic flag + mutex 来实现（类似双检锁但由编译器生成）。注意仅保证"初始化"是线程安全的，后续的并发访问仍需自行同步。

> **Q2**：C++11 之前的双检锁为什么有问题？
>
> **回答**：`instance = new Singleton()` 在底层分为三步：分配内存 → 构造对象 → 将地址赋给 instance。编译器/CPU 可能重排 ② 和 ③——即 instance 指向一块"已分配但未构造"的内存。若此时另一个线程检查 `if (instance != nullptr)` 并通过，它会使用一个**构造未完成**的对象，造成 UB。C++11 的 memory order + atomic 正确解决了这个问题。

---

## 10. Linux I/O 多路复用：select / poll / epoll

### 10.1 三代 I/O 多路复用对比

| 维度 | select | poll | epoll |
|------|--------|------|-------|
| **fd 数量限制** | 1024 (FD_SETSIZE) | 无上限 | 无上限 |
| **API** | `FD_SET/FD_ZERO` 每次重新设置 | 维护 `pollfd` 数组 | `epoll_ctl` 增量增删 |
| **内核-用户态拷贝** | 每次调用全量拷贝 fd_set | 每次调用全量拷贝 pollfd | 仅 `epoll_ctl` 拷贝一次 |
| **就绪事件获取** | O(n) 全量遍历 | O(n) 全量遍历 | O(1) 只取就绪链表 |
| **触发模式** | 水平触发 | 水平触发 | **LT(水平) + ET(边缘)** |
| **适用场景** | 少量 fd、跨平台 | 适量 fd | 大量 fd、高性能 |

### 10.2 epoll 底层原理

```
   ┌──────────────────────────────────────┐
   │ epoll_create(1)                      │
   │   → 创建 eventpoll 对象              │
   │   → 包含：红黑树 (rbr) + 就绪链表 (rdllist)  │
   └──────────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │ epoll_ctl(epfd, ADD, fd, &event)     │
   │   → 将 fd 节点插入红黑树              │
   │   → 注册内核回调：数据到达时自动将 fd │
   │     加入就绪链表                      │
   └──────────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │ 数据到达流程                           │
   │ ① 网卡中断 → 内核协议栈处理数据        │
   │ ② 内核调用 ep_poll_callback           │
   │ ③ 将 fd 节点加入 eventpoll 就绪链表    │
   │ ④ 唤醒阻塞在 epoll_wait 的进程         │
   └──────────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │ epoll_wait(epfd, events, max, timeout)│
   │   → 检查就绪链表是否为空              │
   │   → 非空：将就绪事件拷贝到用户态       │
   │   → 空：进程睡眠，等待被回调唤醒       │
   └──────────────────────────────────────┘
```

### 10.3 LT vs ET：关键区别

```cpp
// LT 模式 (Level Triggered) — 默认
// "只要 fd 就绪，每次 epoll_wait 都通知你"
// 可以不一次读完，下次 epoll_wait 还会通知

// ET 模式 (Edge Triggered)
// "只在状态变化时通知你一次"
// 必须一次性读完/写完（while 循环读到 EAGAIN）
// fd 必须设为非阻塞（否则 read 可能阻塞导致错过后续事件）

// ET 模式设置
event.events = EPOLLIN | EPOLLET;  // 边缘触发

// ET 模式正确读法
void handle_read_et(int fd) {
    while (true) {
        ssize_t n = read(fd, buf, sizeof(buf));
        if (n > 0) {
            process(buf, n);
        } else if (n == -1 && errno == EAGAIN) {
            break;  // 读完了
        } else {
            // 错误或对端关闭
            if (n == 0) close(fd);  // EOF
            else perror("read error");
            break;
        }
    }
}
```

### 10.4 面试官连环追问 🎯

> **Q1**：epoll 为什么用红黑树存储 fd？红黑树在这里的核心优势是什么？
>
> **回答**：epoll 需要支持动态的 **增删改查** fd 集合。红黑树在查找、插入、删除上都是 O(log n)，且保持有序——这样当需要删除 fd 时（如连接断开），可以快速定位。平衡二叉树的确定性时间复杂度比哈希表更适合内核场景（无 rehash 抖动）。

> **Q2**：为什么 ET 模式比 LT 模式"理论上"性能更好？
>
> **回答**：ET 模式下 `epoll_wait` 返回次数少——每个就绪事件只通知一次，减少了系统调用开销。LT 模式可能在数据没读完的情况下反复通知，用户态和内核态之间拷贝事件列表的次数更多。但 ET 实现复杂度更高——必须非阻塞 IO + while 循环读到 EAGAIN。对于大多数应用，LT 的性能足够，避免过早优化增加 bug。

---

## 11. Reactor 模式与高性能网络框架

### 11.1 Reactor 核心思想

```
          ┌──────────┐
          │ 主线程    │    ← 事件循环主线程
          │ Reactor  │
          │ epoll_wait│
          └────┬─────┘
               │ 事件分发
        ┌──────┼──────┐
        ▼      ▼      ▼
   ┌────────┐┌────────┐┌────────┐
   │Handler ││Handler ││Handler │  ← 工作线程池处理 I/O
   │ 读事件 ││ 写事件 ││ 错误   │
   └────────┘└────────┘└────────┘
```

### 11.2 多 Reactor 模型（muduo 架构）

```
  ┌─────────────────────────────────────┐
  │ Main Reactor (主线程)               │
  │ epoll (listen_fd)                   │
  │ accept 新连接                       │
  │ Round-Robin 分发给 Sub Reactors    │
  └──────────┬──────────────────────────┘
             │
     ┌───────┼────────┬────────┐
     ▼       ▼        ▼        ▼
  ┌────────┐┌────────┐┌────────┐┌────────┐
  │Sub     ││Sub     ││Sub     ││Sub     │
  │Reactor ││Reactor ││Reactor ││Reactor │  ← 每核一线程
  │epoll   ││epoll   ││epoll   ││epoll   │
  │(conns) ││(conns) ││(conns) ││(conns) │
  └────────┘└────────┘└────────┘└────────┘
```

### 11.3 Reactor vs Proactor

| 维度 | Reactor | Proactor |
|------|---------|----------|
| **I/O 模型** | 同步 I/O (epoll + read/write) | 异步 I/O (OS 完成读写) |
| **通知内容** | "数据已就绪，你可以读了" | "数据已经读完，这是数据" |
| **读写操作** | 用户线程执行 read/write | OS 异步完成 |
| **典型实现** | Linux epoll → muduo/Netty/Redis | Windows IOCP → Boost.ASIO |
| **Linux 支持** | ✅ 成熟稳定 | ⚠️ Linux AIO 不完全支持 socket |

### 11.4 面试官连环追问 🎯

> **Q1**：为什么 Reator 模式中 accept（接受新连接）要放在主 Reactor 中单独处理？
>
> **回答**：accept 是全局唯一的操作（只有一个 listen socket），多个线程同时 accept 会引起**惊群效应**（thundering herd）——一个新连接到来，所有等待在 listen_fd 上的线程被唤醒，但只有一个能 accept 成功，其余线程白忙一场。Linux 4.5+ 通过 `EPOLLEXCLUSIVE` 标志部分解决了此问题，但主 Reactor 模式从根本上避免了它。

> **Q2**：muduo 中，如何保证同一个连接的所有事件始终在同一个 Sub Reactor 中处理？
>
> **回答**：连接创建时（accept 后），通过**Round-Robin**将新 fd 分给某个 Sub Reactor 的事件循环。该 fd 之后的所有事件（读、写、关闭、错误）都注册在这个 Sub Reactor 的 epoll 实例中。这保证了**同一连接的操作天然是串行的**，无需在应用层加锁——这对简化状态机编程至关重要。

---

## 12. 🔪 终极追杀令：并发编程的最后审判

> 以下 4 道题是区分"懂并发的工程师"和"真正理解并发硬件模型的工程师"的分水岭。

### 🔪 第一刀：自旋锁在 false sharing 下的性能灾难

```cpp
// 下面两个自旋锁的性能差异是多少？为什么？
struct UnpaddedSpinLock {
    std::atomic_flag flag = ATOMIC_FLAG_INIT;
    void lock() { while (flag.test_and_set(std::memory_order_acquire)); }
    void unlock() { flag.clear(std::memory_order_release); }
};

struct alignas(64) PaddedSpinLock {
    std::atomic_flag flag = ATOMIC_FLAG_INIT;
    void lock() { while (flag.test_and_set(std::memory_order_acquire)); }
    void unlock() { flag.clear(std::memory_order_release); }
};

// 假设 4 个线程，每个线程各自操作自己的锁（4 个不同的锁对象）
// 它们被紧密排列在同一个数组中
```

> **满分答案**：
> 性能差异可达 **10-50x**。原因：
> 1. UnpaddedSpinLock 的 `sizeof` ≈ 1-2 字节，4 个锁实例挤在同一个 64 字节 cache line 中
> 2. Thread 0 执行 `test_and_set`（LOCK 前缀指令）→ 锁定其所在的 cache line → Invalidate 其他 3 个核心的该 cache line
> 3. Thread 1 执行 `test_and_set` → Cache Miss → 从 Core 0 偷取 cache line → 再次 LOCK → 再次 Invalidate 所有核心
> 4. 即使 4 个线程操作的是**完全不同的锁对象**，它们在硬件层面互相阻塞
> 5. PaddedSpinLock 中每个锁独占一个 cache line → 无 false sharing → 接近完美并行
>
> **验证**：`perf stat -e L1-dcache-load-misses,LLC-load-misses` 对比两个版本

### 🔪 第二刀：`memory_order_seq_cst` 在 x86 和 ARM 上的指令开销差异

```cpp
std::atomic<int> x{0}, y{0};

// Thread A                // Thread B
x.store(1, seq_cst);       y.store(1, seq_cst);
int r1 = y.load(seq_cst);  int r2 = x.load(seq_cst);

// 问题：r1 和 r2 能同时为 0 吗？
// 问题：这段代码在 x86 和 ARM 上分别生成什么关键指令？
```

> **满分答案**：
> **结果**：在 seq_cst 下，`r1 == 0 && r2 == 0` **不可能发生**——seq_cst 保证全局单一全序。
>
> **x86 汇编**：
> ```asm
> ; Thread A 的 store:         ; Thread B 的 store:
> mov DWORD PTR [x], 1         mov DWORD PTR [y], 1
> mfence                       mfence
> ; Thread A 的 load:          ; Thread B 的 load:
> mov eax, DWORD PTR [y]       mov eax, DWORD PTR [x]
> ```
> `mfence` 是关键——它是 x86 最重的屏障指令，阻止 Store-Load 重排序。
>
> **ARM 汇编**：
> ```asm
> ; Thread A 的 store:         ; Thread B 的 store:
> mov w0, #1                   mov w0, #1
> stlr w0, [x]       ; ← 注意: 必须是 stlr (store-release 不够!)
>                         ;     seq_cst store = stlr + 额外屏障
> dmb ish            ; 完整内存屏障
> ; Thread A 的 load:
> ldar w1, [y]       ; seq_cst load = ldar + 额外屏障
> dmb ish            ; 完整内存屏障
> ```
> ARM 上 seq_cst 的 load 和 store **双方都需要 dmb**，总共 4 条 dmb 指令。而 acquire/release 在 ARM 上用 `ldar`/`stlr` 零额外屏障。这就是为什么在 ARM 上 seq_cst 开销显著高于 acquire/release。
>
> **关键洞见**：如果场景允许，将 seq_cst 降级为 acq_rel + 适当的应用级屏障，在 ARM 上可带来 10-30% 的吞吐提升。

### 🔪 第三刀：有界无锁队列中，CAS 的 `memory_order` 可以降到多弱？

```cpp
// 无锁 SPSC 队列的 enqueue 操作
template<typename T>
class LockFreeSPSCQueue {
    struct Slot { std::atomic<size_t> sequence; T data; };
    std::vector<Slot> buffer;
    alignas(64) std::atomic<size_t> write_pos{0};  // 64 字节对齐！
    alignas(64) std::atomic<size_t> read_pos{0};

public:
    bool enqueue(const T& item) {
        size_t pos = write_pos.load(std::memory_order_relaxed);
        Slot& slot = buffer[pos % capacity];
        size_t seq = slot.sequence.load(std::memory_order_acquire);
        //                                    ^^^^^^^ 为什么这里必须是 acquire？

        // 🔪 杀手问题：
        // 1. 为什么 seq 的 load 必须用 acquire 而不能用 relaxed？
        // 2. 如果用了 relaxed，会出现什么问题？
        // 3. write_pos 的 load 为什么可以用 relaxed？
    }
};
```

> **满分答案**：
> **1.** `seq` 的 `load(acquire)` 与 dequeue 侧的 `slot.sequence.store(release)` 形成同步对。Dequeue 完毕后写入 `store(seq + capacity, release)`，enqueue 侧通过 `load(acquire)` 捕获这个写入，同时**保证 slot 中 data 的写入（dequeue 在 store 之前写入的槽标记/清空操作）对 enqueue 侧可见**。
>
> **2.** 如果 seq 使用 relaxed：enqueue 可能读到 seq 的旧值（虽然 seq 值正确），但由于没有 acquire 屏障，CPU/编译器可能将 **对 slot.data 的写入（enqueue 侧的 item 赋值）重排到 seq 检查之前**，导致 dequeue 读到损坏的数据——即 slot 标记为已填充但 data 还是旧的。
>
> **3.** write_pos 是单写者（SPSC）的——只有 enqueue 线程写入它，其他线程只读。write_pos 的值不需要与任何其他内存操作建立顺序关系。`relaxed` 只保证原子性，在此足够。

### 🔪 第四刀：`shared_ptr` 的原子操作到底在操作什么？

```cpp
// 这段代码有问题吗？请分析到指令层。
std::shared_ptr<Widget> global_widget;

// Thread A: 更新全局配置
void update_widget() {
    auto new_w = std::make_shared<Widget>();
    global_widget = new_w;  // ← 这个赋值是不是原子的？
}

// Thread B: 使用全局配置
void use_widget() {
    auto local_w = global_widget;  // ← 这个拷贝是不是原子的？
    if (local_w) local_w->do_work();
}
```

> **满分答案**：
> **都有问题！** `shared_ptr` 的拷贝赋值和拷贝构造都不是线程安全的。
>
> 每个操作在底层都涉及多个步骤：
> ```asm
> ; global_widget = new_w;  （拷贝赋值）分步：
> ; 1. 读取 new_w 的两个内部指针 (ptr, ctrl)
> ; 2. 原子递增 ctrl 的强引用计数
> ; 3. 原子递减旧 global_widget 的强引用计数（可能触发析构）
> ; 4. 将 (ptr, ctrl) 写入 global_widget  ← 两个 8 字节写入，非原子！
>
> ; auto local_w = global_widget;  （拷贝构造）分步：
> ; 1. 从 global_widget 读取 (ptr, ctrl)  ← 两个 8 字节读取，非原子！
> ; 2. 原子递增 ctrl 的强引用计数
> ```
>
> 如果 Thread A 的步骤 4 和 Thread B 的步骤 1 交错：
> Thread B 可能读到 (old_ptr, new_ctrl) 或 (new_ptr, old_ctrl) 的组合——即**撕裂读**。这会导致 Thread B 对错误的控制块做引用计数操作，造成 use-after-free 或 double-free。
>
> **正确做法**：
> ```cpp
> // 方案 1：加 mutex 保护
> std::mutex widget_mtx;
> // Thread A: lock → global_widget = new_w → unlock
> // Thread B: lock → auto local = global_widget → unlock
>
> // 方案 2：C++20 atomic<shared_ptr>
> std::atomic<std::shared_ptr<Widget>> global_widget;
> // Thread A: global_widget.store(new_w);
> // Thread B: auto local = global_widget.load();
>
> // 方案 3：C++11 atomic_load/store 自由函数
> // 内部通常使用 spinlock 保护
> ```

---

> **下一篇**：[`04_Engineering_Practice.md`](04_Engineering_Practice.md) — 内存泄漏排查、GDB 高级调试、CMake 构建优化、Linux 系统调用