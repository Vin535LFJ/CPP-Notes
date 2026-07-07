# 05 — 音视频场景下的 C++ 内存、对象模型与现代特性

> **定位**：把 C++ 内存/对象模型、RAII、智能指针、移动语义、内存排查，与 FFmpeg / MediaCodec / OpenGL 渲染链路结合起来，面向音视频/客户端岗位的一面基础与二面专项。

---

## 目录

1. [RAII 与智能指针：从所有权到 AVFrame 生命周期](#1-raii-与智能指针从所有权到-avframe-生命周期)
2. [C++ 对象内存模型与对齐：从类布局到音视频缓冲区](#2-c-对象内存模型与对齐从类布局到音视频缓冲区)
3. [程序内存分区与动态分配底层](#3-程序内存分区与动态分配底层)
4. [右值引用、移动语义与零拷贝传递](#4-右值引用移动语义与零拷贝传递)
5. [内存问题排查：泄漏、野指针、堆踩踏](#5-内存问题排查泄漏野指针堆踩踏)
6. [音视频编解码与纹理化渲染核心链路](#6-音视频编解码与纹理化渲染核心链路)
7. [FFmpeg / MediaCodec / OpenGL / Android 图形栈专项](#7-ffmpeg--mediacodec--opengl--android-图形栈专项)
8. [面试优先级总结](#8-面试优先级总结)

---

## 1. RAII 与智能指针：从所有权到 AVFrame 生命周期

### 1.1 RAII 的核心规则

RAII（Resource Acquisition Is Initialization）要求：

- **构造函数获取资源**：内存、文件句柄、互斥锁、纹理 ID、解码器句柄。
- **析构函数释放资源**：对象离开作用域时自动清理，即使发生异常也能释放。
- **资源所有权必须唯一且可追踪**：谁拥有资源，谁负责释放；非拥有者只能观察，不释放。

音视频代码中最常见的资源包括：

| 资源 | 获取 | 释放 | 推荐封装 |
|---|---|---|---|
| `AVFrame*` | `av_frame_alloc()` | `av_frame_free(&frame)` | `std::unique_ptr<AVFrame, Deleter>` |
| `AVPacket*` | `av_packet_alloc()` | `av_packet_free(&pkt)` | `std::unique_ptr<AVPacket, Deleter>` |
| OpenGL texture | `glGenTextures` | `glDeleteTextures` | move-only RAII class |
| mutex lock | `lock()` | `unlock()` | `std::lock_guard` / `std::unique_lock` |
| decoder context | create/open API | close/release API | move-only wrapper |

### 1.2 `unique_ptr`：独占所有权

`std::unique_ptr<T>` 只允许一个对象拥有资源，不能拷贝，只能移动。它适合表达“这个模块唯一负责释放资源”。

```cpp
struct AVFrameDeleter {
    void operator()(AVFrame* frame) const noexcept {
        av_frame_free(&frame); // FFmpeg 要求传入 AVFrame**
    }
};

using AVFramePtr = std::unique_ptr<AVFrame, AVFrameDeleter>;

AVFramePtr make_frame() {
    AVFrame* raw = av_frame_alloc();
    if (!raw) throw std::bad_alloc{};
    return AVFramePtr(raw);
}
```

代码讲解：

- `AVFrameDeleter::operator()` 是自定义删除器，解决 FFmpeg 释放函数不是 `delete frame` 的问题。
- `av_frame_free(&frame)` 需要 `AVFrame**`，因为 FFmpeg 会释放对象并把传入指针置空；这里置空的是删除器参数的本地副本，但释放动作已经完成。
- `make_frame()` 在创建失败时抛 `std::bad_alloc`，让调用方不用检查裸指针；成功后立即交给 `unique_ptr`，从这一行开始资源具备异常安全。
- `using AVFramePtr = ...` 把复杂模板类型集中命名，避免业务代码到处重复 deleter 类型。

关键点：

- `unique_ptr` 对象本身通常只保存一个裸指针；如果 deleter 是无状态类型，常见实现可利用空基类优化避免额外空间。
- `unique_ptr` 的析构自动调用 deleter，避免早退路径泄漏。
- 用 `std::move(frame)` 把解码帧所有权转移给队列/渲染模块后，原指针进入空状态或有效但未指定状态，不应继续解引用。
- 如果某个 API 只是“借用”帧，不应该接收 `AVFramePtr`，而应该接收 `AVFrame*`、`AVFrame&` 或 `std::span` / view 类型，避免误表达所有权转移。

### 1.3 `shared_ptr`：共享所有权与引用计数

`std::shared_ptr<T>` 由两部分组成：

- 指向对象的指针。
- 控制块：强引用计数、弱引用计数、删除器、分配器等。

强引用计数归零时销毁对象；弱引用计数归零时释放控制块。引用计数增减是线程安全的，但**被管理对象本身的读写不是自动线程安全的**。

适合使用 `shared_ptr` 的音视频场景：

- 多个模块同时需要持有同一帧元数据，例如解码队列、同步模块、渲染调度器。
- 帧被异步回调引用，生命周期必须跨线程延长。
- 不清楚唯一所有者，且生命周期需要由最后使用者决定。

不适合使用 `shared_ptr` 的场景：

- 每帧高频传递且所有权单一，引用计数原子操作可能引入开销。
- 可以用对象池、环形队列或 `unique_ptr` 明确表达所有权。

### 1.4 `weak_ptr`：打破循环引用

循环引用示例：

```cpp
struct Decoder;
struct Renderer;

struct Decoder {
    std::shared_ptr<Renderer> renderer;
};

struct Renderer {
    std::shared_ptr<Decoder> decoder; // ❌ 循环引用，二者引用计数永不归零
};
```

修复方式：一边改成 `weak_ptr`。

```cpp
struct Renderer {
    std::weak_ptr<Decoder> decoder; // ✅ 非拥有观察

    void render() {
        if (auto d = decoder.lock()) {
            // 安全使用 Decoder
        }
    }
};
```

代码讲解：

- `weak_ptr` 不增加强引用计数，因此不会阻止 `Decoder` 析构。
- `lock()` 会尝试把弱引用提升为 `shared_ptr`：如果对象还活着，返回非空 `shared_ptr`，并在局部作用域内延长生命周期；如果对象已销毁，返回空指针。
- 不要用 `expired()` 后再单独访问对象，因为检查和访问之间可能被其他线程释放；`lock()` 才是原子地获取可用强引用的安全写法。

### 1.5 `AVFrame` 生命周期管理原则

FFmpeg 中 `AVFrame` 结构体与其底层 buffer 需要分开理解：

- `av_frame_alloc()` 分配 `AVFrame` 结构体。
- 解码器或 `av_frame_get_buffer()` 可能为数据平面分配引用计数 buffer。
- `av_frame_unref()` 释放/减少当前帧引用的底层 buffer，但保留 `AVFrame` 结构体以便复用。
- `av_frame_free()` 释放结构体本身，并 unref 其引用的 buffer。

工程建议：

```cpp
class FramePoolItem {
public:
    FramePoolItem() : frame_(make_frame()) {}

    AVFrame* get() noexcept { return frame_.get(); }

    void reset_for_reuse() noexcept {
        av_frame_unref(frame_.get());
    }

private:
    AVFramePtr frame_;
};
```

代码讲解：

- `FramePoolItem` 持有 `AVFramePtr`，表示结构体对象本身由池元素独占拥有。
- `get()` 返回裸指针只是为了调用 FFmpeg C API；它不转移所有权，调用方不能保存到生命周期之外。
- `reset_for_reuse()` 使用 `av_frame_unref()` 清掉当前帧引用的底层 buffer，但保留 `AVFrame` 外壳，适合对象池复用，减少每帧 `alloc/free` 抖动。
- 如果帧已经交给渲染线程异步使用，不能立刻 `reset_for_reuse()`；必须等渲染完成或引用计数/同步栅栏确认不再使用。

---

## 2. C++ 对象内存模型与对齐：从类布局到音视频缓冲区

### 2.1 类的基本内存布局

非静态数据成员通常按声明顺序排列，中间可能插入 padding 以满足对齐要求。静态成员不属于每个对象实例。成员函数不存储在对象里。

```cpp
struct S {
    char c;   // offset 0
    int  i;   // 通常 offset 4，中间 3 字节 padding
    char d;   // offset 8
};            // sizeof(S) 通常为 12，尾部 padding 到 alignof(S)
```

代码讲解：

- `char c` 只需要 1 字节对齐，但后面的 `int i` 通常需要 4 字节对齐，所以编译器会在 `c` 后插入 padding。
- 结构体总大小还要满足整体对齐，方便数组中每个元素都满足成员对齐要求。
- 调整成员顺序有时可以减少 padding，但不能为了省几个字节破坏数据语义；协议/文件格式/跨进程共享结构更不能随意改顺序。

教学注意：具体 layout 是 ABI/实现相关；可用 `sizeof`、`alignof`、`offsetof` 观察，但不能把某个平台结果当成标准保证。

### 2.2 虚函数表与多态原理

有虚函数的类通常会在对象中存放 vptr，指向类对应的 vtable；vtable 中保存虚函数入口和 RTTI/offset 信息。虚调用大致流程：

1. 通过对象 vptr 找到 vtable。
2. 根据虚函数槽位读取函数地址。
3. 必要时调整 `this` 指针。
4. 间接调用目标函数。

标准只规定多态语义，不规定 vptr/vtable 的具体布局。Itanium ABI、MSVC ABI 在虚继承、多继承、RTTI 存储上细节不同。

### 2.3 虚继承

虚继承解决菱形继承中公共基类子对象重复的问题：

```cpp
struct Base { int id; };
struct A : virtual Base {};
struct B : virtual Base {};
struct C : A, B {}; // C 中只有一个 Base 虚基类子对象
```

代价：

- 对象布局更复杂。
- 访问虚基类需要额外 offset 查找。
- 构造顺序由最派生类负责初始化虚基类。

### 2.4 内存对齐与音视频缓冲区性能

音视频场景中，对齐不仅影响 C++ 对象访问，也影响 SIMD、DMA、硬解码器和 GPU 上传性能。

常见规则：

- `alignof(T)` 表示类型要求的最小对齐。
- 对象地址必须满足其类型对齐，否则可能 UB 或性能下降。
- 视频帧每行通常有 stride/linesize，不一定等于 `width * bytes_per_pixel`。
- 硬解输出的 buffer 可能要求 16/32/64/128 字节对齐，不能假设像素平面紧密连续。

处理 YUV 数据时要始终使用 `linesize`：

```cpp
for (int y = 0; y < height; ++y) {
    const uint8_t* row = frame->data[0] + y * frame->linesize[0];
    // 只读取有效 width 范围，不越过 stride padding
}
```

代码讲解：

- `frame->data[0]` 是 Y 平面的首地址，`linesize[0]` 是相邻两行起始地址之间的字节跨度。
- `linesize` 可能大于可见宽度对应的字节数，因为解码器为了 SIMD/GPU/DMA 对齐会在行尾补 padding。
- 循环中每行都用 `y * linesize[0]` 定位，避免把上一行 padding 当成下一行有效像素。
- 写 UV 平面时要分别使用 `data[1]/linesize[1]`、`data[2]/linesize[2]`，并注意 YUV420 的 UV 高宽通常是 Y 的一半。

---

## 3. 程序内存分区与动态分配底层

| 区域 | 典型内容 | 生命周期 | 风险 |
|---|---|---|---|
| 栈 | 局部变量、返回地址、函数调用帧 | 进入作用域创建，离开销毁 | 返回局部变量地址、栈溢出 |
| 堆 | `new`/`malloc` 动态分配 | 手动释放或 RAII 自动释放 | 泄漏、UAF、double free、堆踩踏 |
| 全局/静态区 | 全局变量、静态局部变量 | 程序启动到退出 | 初始化顺序问题 |
| 常量/只读区 | 字符串字面量、只读常量 | 程序生命周期 | 修改字符串字面量 UB |
| 代码区 | 指令 | 程序生命周期 | 通常只读/可执行 |

动态分配底层通常经过：

1. C++ `operator new` / C `malloc`。
2. 运行时分配器维护小块缓存、arena、空闲链表或 size class。
3. 必要时向 OS 申请页：`brk/sbrk`、`mmap`、VirtualAlloc 等。
4. 释放时可能归还给分配器缓存，不一定立刻归还 OS。

工程建议：

- 高频帧对象用对象池/环形队列减少分配抖动。
- 大 buffer 避免频繁申请释放，使用池化和复用。
- 跨线程传递大帧数据时传递所有权/句柄，不传递大块拷贝。

---

## 4. 右值引用、移动语义与零拷贝传递

### 4.1 `std::move` 本质

`std::move(x)` 不移动任何字节；它只是把表达式转换成 xvalue，允许调用移动构造/移动赋值。

```cpp
template<class T>
constexpr std::remove_reference_t<T>&& move(T&& t) noexcept {
    return static_cast<std::remove_reference_t<T>&&>(t);
}
```

代码讲解：

- `T&&` 在模板中先接收任意值类别，再通过 `remove_reference_t<T>&&` 统一转换成右值引用表达式。
- `std::move` 本身不释放、不复制、不转移 buffer；如果类型没有移动构造，后续仍可能调用拷贝构造。
- 因此“用了 `std::move` 就零拷贝”是错误说法，真正决定是否零拷贝的是被移动类型的移动构造/赋值实现。

真正的移动发生在类型的移动构造/移动赋值里。

### 4.2 帧对象移动设计

```cpp
class VideoFrame {
public:
    VideoFrame() = default;
    explicit VideoFrame(AVFramePtr f) noexcept : frame_(std::move(f)) {}

    VideoFrame(VideoFrame&&) noexcept = default;
    VideoFrame& operator=(VideoFrame&&) noexcept = default;

    VideoFrame(const VideoFrame&) = delete;
    VideoFrame& operator=(const VideoFrame&) = delete;

    AVFrame* get() noexcept { return frame_.get(); }

private:
    AVFramePtr frame_;
};
```

代码讲解：

- 构造函数接收 `AVFramePtr` 并 `std::move` 到成员，表示 `VideoFrame` 接管帧所有权。
- 拷贝构造/拷贝赋值被 `delete`，可以在编译期阻止无意的帧深拷贝或双重释放。
- 默认移动构造/移动赋值会移动内部 `unique_ptr`，本质是转移指针和 deleter 状态，不复制像素数据。
- `get()` 暴露观察指针供 C API 使用，但类仍然保留释放责任。

设计理由：

- 解码帧通常不应无意深拷贝。
- 移动只转移句柄/引用，避免大 buffer 拷贝。
- `noexcept` 让 `std::vector<VideoFrame>` 扩容时优先移动，保持异常安全和性能。

### 4.3 零拷贝优化原则

- 软件解码 YUV → OpenGL：尽量复用 PBO/纹理，避免每帧重新分配。
- 硬解 Surface 模式：解码器输出到 GraphicBuffer，经 SurfaceTexture 绑定为 OES 外部纹理，CPU 不接触像素数据。
- 跨模块传递：传递 `unique_ptr`、`shared_ptr`、buffer handle、index，而不是复制整帧。
- 注意同步：零拷贝不等于零生命周期管理，引用帧必须保留到 GPU/显示消费完成。

---

## 5. 内存问题排查：泄漏、野指针、堆踩踏

### 5.1 常见诱因

| 问题 | 诱因 | 音视频典型场景 |
|---|---|---|
| 内存泄漏 | 早退未释放、循环引用、队列积压 | 解码失败路径未 `av_frame_free`，渲染队列无限增长 |
| 野指针/UAF | 释放后继续使用 | 渲染线程仍持有已回收 `AVFrame*` |
| double free | 所有权不清 | `unique_ptr` 和手动 `av_frame_free` 同时释放 |
| 堆踩踏 | 越界写 | 忽略 `linesize`，按 `width` 错误写完整平面 |
| 数据竞争 | 多线程无同步 | 解码线程和渲染线程同时改帧状态 |

### 5.2 定位思路

1. **先画所有权图**：解码器、队列、同步模块、渲染器谁拥有帧？谁只是观察？
2. **加生命周期日志**：构造、入队、出队、渲染完成、释放。
3. **使用工具**：ASan 查越界/UAF/double free；LSan 查泄漏；TSan 查数据竞争；Valgrind/heaptrack 查堆行为；Android 可结合 Perfetto、simpleperf、meminfo。
4. **检查队列背压**：播放卡顿时，队列可能持有大量未消费帧导致内存飙升。
5. **检查错误路径**：解码失败、seek、pause/resume、surface 重建、GL context 丢失。

---

## 6. 音视频编解码与纹理化渲染核心链路

### 6.1 I/P/B 帧与 DPB

- **I 帧**：帧内编码，可独立解码，常作为随机访问点。
- **P 帧**：参考过去的 I/P 帧。
- **B 帧**：可参考过去和未来帧，因此解码顺序与显示顺序可能不同。
- **DPB（Decoded Picture Buffer）**：保存已解码但仍可能被后续帧参考或等待显示的图片。

生命周期规则：

- B 帧如果不再作为参考，渲染/显示完成后即可回收。
- P/I 参考帧在参考期内必须保留，不能因为已经显示就提前释放。
- seek/flush 时必须清理解码器内部状态与应用层队列，避免旧参考帧污染新时间线。

### 6.2 MediaCodec Surface 零拷贝渲染链路

典型链路：

```text
MediaExtractor/网络包
        ↓
MediaCodec 解码
        ↓
Surface / BufferQueue
        ↓
SurfaceTexture 更新 GraphicBuffer
        ↓
GL_TEXTURE_EXTERNAL_OES 外部纹理
        ↓
OpenGL shader / FBO / 合成
        ↓
屏幕 / SurfaceFlinger
```

链路讲解：

- MediaCodec 的解码输出不是普通 C++ 堆内存，而是进入 Android 图形缓冲体系中的 `GraphicBuffer`。
- `Surface` / `BufferQueue` 负责在生产者（解码器）和消费者（SurfaceTexture / SurfaceFlinger）之间传递 buffer 所有权和状态。
- `SurfaceTexture` 在 GL 线程调用 `updateTexImage()` 后，把最新 buffer 绑定到 `GL_TEXTURE_EXTERNAL_OES`。
- shader 采样 OES 外部纹理时，GPU 直接读取底层图形缓冲，避免 CPU 把 YUV/RGB 数据搬到普通内存再上传纹理。

Surface 模式优势：

- 解码输出进入系统 GraphicBuffer，CPU 不拷贝像素。
- SurfaceTexture 将 buffer 暴露为 OES 外部纹理。
- GPU 直接采样纹理完成渲染和滤镜。

ByteBuffer 模式适合：

- 需要 CPU 访问解码后像素。
- 需要自定义软件处理。
- 分辨率较低或兼容性优先。

### 6.3 OpenGL 纹理池复用

纹理池目标：避免每帧 `glGenTextures` / `glDeleteTextures`。

原则：

- 按宽高、格式、用途分桶复用。
- Surface 重建或 GL context 丢失时必须整体释放并重建。
- FBO、texture、PBO 生命周期要绑定到 GL context 所在线程或共享上下文。
- GPU 未消费完成的纹理不能提前回收，可用 fence/sync 或渲染队列状态管理。

### 6.4 解码与渲染线程模型

推荐结构：

```text
解复用线程 → 解码线程 → 帧队列/纹理队列 → 渲染线程 → 显示
                    ↑              ↓
                 flush/seek       时钟同步
```

注意点：

- 解码线程不要阻塞 GL 渲染线程。
- 渲染线程只执行 GL 调用，避免跨线程误用 GL context。
- 异步解码回调只做轻量入队，不做耗时渲染。
- seek/stop/release 必须定义状态机，防止回调访问已释放对象。

### 6.5 资源释放顺序

通常应按依赖反向释放：

1. 停止输入，阻止新包/新帧进入。
2. 通知解码线程退出并 join。
3. flush/释放解码器输出 buffer。
4. 清空帧队列/纹理队列。
5. 在 GL 线程释放纹理、FBO、PBO。
6. 释放 SurfaceTexture / Surface / ANativeWindow。
7. 销毁 GL context。

禁忌：

- 参考帧仍在 DPB 或渲染队列中时提前释放。
- GL context 已销毁后再调用 `glDeleteTextures`。
- Java/Kotlin 层 Surface 生命周期结束后 native 层继续提交 buffer。

---

## 7. FFmpeg / MediaCodec / OpenGL / Android 图形栈专项

### 7.1 FFmpeg 软解与硬解封装

软解常见流程：

1. `av_read_frame` 得到 `AVPacket`。
2. `avcodec_send_packet` 输入解码器。
3. `avcodec_receive_frame` 取出 `AVFrame`。
4. 使用 `AVFrame::data[]` 与 `linesize[]` 访问平面。
5. 渲染完成后 `av_frame_unref` 或交还对象池。

YUV 上传 OpenGL 两种方案：

- **直接上传**：`glTexSubImage2D` 上传 Y/U/V 或 NV12 两个平面，简单但 CPU/GPU 同步压力较大。
- **PBO 加速**：CPU 写入 PBO，GPU 异步从 PBO 更新纹理，适合高分辨率连续帧。

FFmpeg 硬解帧：

- `AVFrame` 可能只持有硬件 frame 引用，像素内存在 GPU/驱动侧。
- 需要 `av_hwframe_transfer_data` 才能拷贝到 CPU 可读内存。
- 不应假设 `data[]` 总是普通 CPU 指针。

### 7.2 MediaCodec Buffer 状态机

输入 Buffer：

```text
dequeueInputBuffer → 填充压缩数据 → queueInputBuffer
```

输出 Buffer：

```text
dequeueOutputBuffer → 读取/渲染 → releaseOutputBuffer(render=true/false)
```

代码/时序讲解：

- `dequeueInputBuffer` 拿到的是“可写输入槽位”，写入压缩数据后必须 `queueInputBuffer` 交还给 codec。
- `dequeueOutputBuffer` 拿到的是“可消费输出槽位”，如果是 ByteBuffer 模式，应用读取像素；如果是 Surface 模式，应用通常不直接读像素。
- `releaseOutputBuffer(index, true)` 的 `true` 表示把该输出帧提交给 Surface；如果传 `false`，表示丢弃/不渲染。
- 每个 index 都必须按状态机归还，否则 codec 可能因为 buffer 被占满而停止输出。

Surface 模式下：

- `releaseOutputBuffer(index, true)` 表示把解码结果释放给 Surface 渲染链路。
- 输出像素由系统底层 GraphicBuffer 管理，调用方只管理 buffer index 和释放时机。

### 7.3 FFmpeg 软解 vs MediaCodec 硬解

| 维度 | FFmpeg 软解 | MediaCodec 硬解 |
|---|---|---|
| 性能/功耗 | CPU 压力大，功耗高 | 硬件加速，功耗低 |
| 兼容性 | 格式支持强，可控 | 依赖设备厂商实现 |
| 定制能力 | 方便插入软件处理 | Surface 零拷贝链路不适合 CPU 像素处理 |
| 延迟 | 可细调 | 可能受系统队列影响 |
| 渲染 | CPU YUV 上传 GPU | OES 外部纹理零拷贝 |

常见硬解坑：

- 分辨率变化需要重新配置 codec / Surface。
- 颜色格式和色彩范围设备差异。
- EOS、flush、seek 状态机错误导致卡死。
- Surface 销毁后输出 buffer 释放时序错误。

### 7.4 `GL_TEXTURE_EXTERNAL_OES`

外部纹理特点：

- 数据来源通常是 SurfaceTexture / camera / video decoder。
- 不能像普通 `GL_TEXTURE_2D` 一样任意 `glTexImage2D` 上传。
- shader 需要使用 `samplerExternalOES`。
- SurfaceTexture 提供的纹理矩阵用于处理裁剪、旋转、坐标变换。

### 7.5 YUV / NV12 转 RGB

YUV 转 RGB 要注意：

- BT.601 / BT.709 / BT.2020 矩阵不同。
- full range 与 limited range 不同。
- NV12 是 Y 平面 + 交错 UV；NV21 是 Y 平面 + 交错 VU。
- 色彩错误常表现为偏绿、偏紫、发灰、对比度异常。

### 7.6 FBO 离屏渲染

FBO 用途：

- 视频滤镜链。
- 多帧合成。
- 画面裁剪/缩放/旋转。
- 截帧或编码前预处理。

注意：FBO 绑定的 texture/renderbuffer 尺寸、格式必须匹配需求；频繁创建销毁 FBO 会造成性能抖动，应池化复用。

### 7.7 PTS / DTS 与音画同步

- **DTS**：解码时间戳，决定送入解码器顺序。
- **PTS**：显示时间戳，决定渲染/播放时间。
- B 帧存在时，DTS 与 PTS 可能不同。

主流同步策略：

1. 以音频为主时钟：最常见，音频连续性对用户更敏感。
2. 以视频为主时钟：特定无音频或视频主导场景。
3. 外部时钟：直播、会议、低延迟同步等。

追帧/丢帧：

- 视频落后音频过多：丢弃非关键过期帧，优先追到当前时钟。
- 视频超前：等待或重复上一帧。
- 网络抖动：使用 jitter buffer 平滑输入，但会增加延迟。

### 7.8 Android 图形栈

- **BufferQueue**：生产者-消费者队列。MediaCodec、Camera、OpenGL 可作为生产者；SurfaceTexture、SurfaceFlinger 可作为消费者。
- **GraphicBuffer**：可跨进程共享的图形缓冲，通常通过 gralloc/ION/dma-buf 等机制实现零拷贝共享。
- **SurfaceFlinger**：系统合成服务，按 Vsync 节奏合成各窗口图层。
- **Vsync**：显示刷新节拍；错过 Vsync 会产生掉帧/卡顿，错误同步可能造成撕裂。
- **ANativeWindow / Surface / SurfaceTexture**：`Surface` 提供生产端目标，`SurfaceTexture` 消费 buffer 并转成 GL 纹理，`ANativeWindow` 是 native 层操作窗口 buffer 的接口。

---

## 8. 面试优先级总结

1. **一面基础必补**：C++ RAII、智能指针、对象模型、内存分区、移动语义、常见内存错误与算法高频题。
2. **二面专项重点**：FFmpeg / MediaCodec 内存边界、DPB/参考帧生命周期、SurfaceTexture / OES 纹理零拷贝、音画同步。
3. **三面拔高拓展**：Android 图形栈、BufferQueue/GraphicBuffer、硬解兼容性、低延迟播放、纹理池和对象池工程化。
