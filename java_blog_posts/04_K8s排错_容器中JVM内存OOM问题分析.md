# K8s 排错：从 glibc 内存分配器聊到容器 OOM

## 背景
OOMKilled 是 K8s 运维中最头疼的问题之一。有时我们发现 JVM Heap 只用了 50%，但容器内存监控却显示 99%，最后惨遭 Kill。
这中间的“暗物质”到底是什么？本文将深入 Linux 系统编程领域，探讨 **glibc malloc**、**Native Memory Tracking (NMT)** 以及内存碎片问题。

## 1. 内存的“罗生门”：RSS vs Heap
K8s 监控的内存指标通常是 **RSS (Resident Set Size)**，即进程实际占用的物理内存。
$$ RSS = Heap + Metaspace + CodeCache + DirectMemory + ThreadStack + NativeLib Overhead $$

JVM 参数 `-Xmx` 仅仅限制了 Heap。很多时候，**Native Overhead** 才是元凶。

## 2. 隐藏的杀手：glibc malloc 碎片
在 Linux 环境下，JVM（基于 HotSpot）底层依赖 `glibc` 的 `malloc` 来申请 Native 内存（比如解压缩、NIO 缓冲区、JNI 调用）。
`glibc` 为了多线程性能，使用了 **Per-thread Arenas** 机制。每个线程都有自己的内存分配池（Arena），以减少锁竞争。

**技术细节**：
默认配置下，`MALLOC_ARENA_MAX` = 8 * CPU Cores。
在一个 8 核的容器中，最多可能产生 64 个 Arena。当线程频繁申请和释放小内存时，这些 Arena 中会产生大量的**内存碎片**。虽然 `free()` 被调用了，但由于碎片化，glibc 无法将内存页归还给操作系统（kernel），导致 RSS 居高不下。

**排查实战**：
如果发现 RSS 远大于 NMT 统计的 Committed 内存，极有可能是碎片问题。
**解决方案**：
在环境变量中设置 `MALLOC_ARENA_MAX=2`。这会牺牲微小的多线程分配性能，但能显著减少内存碎片，让 RSS 曲线更加平稳。

## 3. 使用 NMT 抽丝剥茧
不要瞎猜，用数据说话。JVM 自带的 **Native Memory Tracking (NMT)** 是排查利器。

启动参数：`-XX:NativeMemoryTracking=summary` (注意：有 5%-10% 的性能损耗，生产环境慎用或仅在灰度开启)

进入容器执行：
```bash
jcmd 1 VM.native_memory summary
```

**关键输出解读**：
*   **Internal**: 这一项通常很大，包含了 Unsafe.allocateMemory 的直接内存（如果没走 NIO DirectByteBuffer）以及 glibc 的管理开销。
*   **Symbol**: 字符串表。如果使用了大量的 `String.intern()`，这里会爆炸。
*   **Thread**: `Thread Stack Size * Thread Count`。注意，Java 11 以后，Stack 内存是 Lazy Allocation 的，但虚拟内存（Reserved）会直接占满 `1MB * Threads`。

## 4. 堆外内存泄漏：Netty 的引用计数
使用了 Netty 或 Spring WebFlux 的应用，很容易遇到 **Direct Memory Leak**。
Netty 使用 `ByteBuf` 池化技术（PooledByteBufAllocator）来减少内存分配开销。这些 buffer 是基于**引用计数 (Reference Counting)** 管理的。
如果开发者在处理完请求后，忘记调用 `ReferenceCountUtil.release(msg)`，这块直接内存就永远不会回收。

**调试技巧**：
设置 `-Dio.netty.leakDetection.level=PARANOID`。
Netty 会采样并追踪 ByteBuf 的分配堆栈。一旦发现泄漏，日志中会打印出该 buffer 是在哪里创建的，帮你精准定位代码行号。

## 总结
容器内的 OOM 问题，往往是 Java 运行时与 Linux 内核交互的灰色地带。
从 `glibc` 的 Arena 分配策略，到 Netty 的引用计数机制，只有深入到底层原理，才能在监控报警的那一刻，透过现象看本质。
