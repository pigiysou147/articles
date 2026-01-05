# K8s 排错：从 glibc 内存分配器聊到容器 OOM

## 背景
OOMKilled 是 K8s 运维中最头疼的问题之一。有时我们发现 JVM Heap 只用了 50%，但容器内存监控却显示 99%，最后惨遭 Kill。
这中间的“暗物质”到底是什么？本文将深入 Linux 系统编程领域，探讨 **glibc malloc**、**Native Memory Tracking (NMT)** 以及内存碎片问题。

## 1. 内存的“罗生门”：RSS vs Heap

### 1.1 什么是 RSS？
K8s 监控的内存指标通常是 **RSS (Resident Set Size)**，即进程实际占用的物理内存页（Pages）。
而 JVM 的 `-Xmx` 仅仅限制了 Java Heap 的大小。
$$ \text{RSS} \approx \text{Heap} + \text{Metaspace} + \text{CodeCache} + \text{DirectMemory} + \text{ThreadStack} + \text{NativeLib Overhead} $$

### 1.2 谁在消耗 Native 内存？
*   **Metaspace**: 存储类的元数据。Spring 应用因为大量动态代理，Metaspace 往往不小。
*   **CodeCache**: JIT (C1/C2) 编译后的机器码。
*   **DirectMemory**: Netty/NIO 使用的堆外内存。
*   **Symbol Table**: 字符串常量池（String.intern）。
*   **glibc Overhead**: **这是最容易被忽视的**。

## 2. 隐藏的杀手：glibc malloc 碎片

### 2.1 sbrk 与 mmap
在 Linux 中，C 标准库的 `malloc` 申请内存底层主要通过两个 syscall：
1.  **sbrk**: 扩展 Data 段（堆）。适用于小内存分配。
2.  **mmap**: 申请匿名内存映射。适用于大内存分配（默认 > 128KB）。

### 2.2 Per-thread Arenas 机制
JVM（基于 HotSpot）底层依赖 `glibc` 来申请 Native 内存（比如解压缩、NIO 缓冲区、JNI 调用）。
为了解决多线程竞争锁的问题，`glibc` 引入了 **Arena** 机制。每个线程会尝试绑定一个 Arena，在自己的 Arena 中分配内存，无需全局锁。

**内存碎片陷阱**：
默认配置下，`MALLOC_ARENA_MAX` = 8 * CPU Cores。
在一个 8 核的容器中，最多可能产生 64 个 Arena。
当线程频繁申请和释放小内存时（比如大量解压 JSON、GZIP），这些 Arena 中会产生大量的**内存碎片**。
虽然程序调用了 `free()`，但在 glibc 层面，这些内存页并没有归还给操作系统（Kernel），而是留在了 Arena 的 Free List 中，以备下次使用。
**结果**：从 OS 视角看，进程的 RSS 居高不下，最终触发 K8s 的 OOM Limit。

### 2.3 解决方案
1.  **设置环境变量**：`MALLOC_ARENA_MAX=2`。
    这会限制 Arena 的数量，增加锁竞争，稍微牺牲一点多线程分配性能，但能显著减少内存碎片，让 RSS 曲线更加平稳。在微服务场景下，这是性价比极高的优化。
2.  **更换分配器**：使用 **jemalloc** 或 **tcmalloc**。
    jemalloc 对碎片处理更优秀。许多高性能中间件（如 Redis, Netty）都推荐使用 jemalloc。

## 3. 使用 NMT 抽丝剥茧

不要瞎猜，用数据说话。JVM 自带的 **Native Memory Tracking (NMT)** 是排查利器。

### 3.1 开启 NMT
启动参数：`-XX:NativeMemoryTracking=summary`
*注意*：这会有 5%-10% 的性能损耗，建议仅在预发环境或灰度实例开启。

### 3.2 分析输出
进入容器执行：
```bash
jcmd 1 VM.native_memory summary
```

**关键输出解读**：
```
Total: reserved=4GB, committed=2.5GB
-                 Java Heap (reserved=2GB, committed=1.5GB)
-                     Class (reserved=1GB, committed=100MB) # Metaspace
-                    Thread (reserved=200MB, committed=50MB) # Stack
-                  Internal (reserved=100MB, committed=100MB) # Unsafe.allocateMemory
```

**差值分析**：
如果 `RSS (监控值) - NMT Committed (统计值)` 的差值非常大（比如超过 500MB），那么基本可以确定是 **glibc 碎片** 或者 **非 JVM 管理的 Native 内存泄漏**（如 JNI 库泄漏）。

### 3.3 使用 pmap 查看内存映射
如果 NMT 看不出问题，可以使用 Linux 原生工具 `pmap`：
```bash
pmap -x 1 | sort -rn -k3 | head -20
```
查看是否有大量的 **64MB** 块（glibc Arena 的典型特征）或者异常的匿名内存段（anon）。

## 4. 堆外内存泄漏：Netty 的引用计数

使用了 Netty 或 Spring WebFlux 的应用，很容易遇到 **Direct Memory Leak**。
Netty 使用 `ByteBuf` 池化技术（PooledByteBufAllocator）来减少内存分配开销。这些 buffer 是基于**引用计数 (Reference Counting)** 管理的。

**泄漏场景**：
开发者在 ChannelHandler 中处理完请求后，忘记调用 `ReferenceCountUtil.release(msg)`，或者在异常路径中跳过了 release。这块直接内存就永远不会被回收。

**调试技巧**：
设置 `-Dio.netty.leakDetection.level=PARANOID`。
Netty 会采样并追踪 ByteBuf 的分配堆栈。一旦发现 GC 回收了一个引用计数不为 0 的 ByteBuf，它会打印出 error 日志，并附带该 buffer 的**分配堆栈信息**，帮你精准定位代码行号。

## 总结
容器内的 OOM 问题，往往是 Java 运行时与 Linux 内核交互的灰色地带。
*   理解 **glibc Arena** 的分配策略。
*   熟练使用 **NMT** 和 **pmap**。
*   掌握 **Netty 引用计数** 的调试方法。
只有深入到底层原理，才能在监控报警的那一刻，透过现象看本质，而不是无助地重启服务。
