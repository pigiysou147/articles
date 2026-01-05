# K8s 排错：容器中 JVM 内存 OOM 问题分析

## 现象
当 Java 服务运行在 Kubernetes Pod 中时，运维监控可能会收到 Pod 重启的报警（OOMKilled）。
查看 Grafana 监控，发现堆内存（Heap）并没有满，但 Pod 内存占用率却一直飙升直到被 Kill。

## 1. 容器视角的 OOM vs JVM 视角的 OOM
这是两个概念：
- **JVM OOM (`java.lang.OutOfMemoryError`)**: 堆内存满了，JVM 抛出异常，进程通常还在，应用日志里有错误栈。
- **K8s OOMKilled**: 容器使用的总内存超过了 `resources.limits.memory`，Linux 内核直接 Kill 掉进程（SIGKILL），**没有 Java 日志**。

既然监控显示 Heap 没满，那一定是 **Non-Heap (堆外内存)** 或者是 **Overhead** 导致的总内存超标。

## 2. 罪魁祸首排查

### 2.1 常见的堆外内存消耗者
1.  **Metaspace (元空间)**: 存放类信息。如果动态加载类太多（如滥用反射、动态代理），可能撑爆。
2.  **Direct Memory (直接内存)**: NIO (Netty) 大量使用。
3.  **Thread Stack (线程栈)**: 每个线程占用 1MB (默认)。线程数过多极为致命。
4.  **Code Cache**: JIT 编译后的代码。

### 2.2 案例分析：JVM 参数配置失误
常见的一个误区是 Dockerfile 启动参数配置失误，例如：
```bash
java -Xmx2G -jar app.jar
```
而 K8s 的 limit 设置为：
```yaml
resources:
  limits:
    memory: "2Gi"
```
**问题所在**：`-Xmx` 仅仅限制了 Heap 大小。
Total Memory = Heap + Metaspace + DirectMemory + ThreadStack * Threads + ...
如果 Heap 占了 2G，留给其他的空间几乎为 0，只要稍微有点线程或 Netty 操作，立马 OOMKilled。

## 3. 解决方案

### 3.1 预留缓冲空间
经验法则：将 Heap 大小设置为容器 Limit 的 **75% - 80%**。
如果 Limit 是 2Gi (2048MB)，建议 Xmx 设置为 1536MB 左右。

### 3.2 自动感知容器限制
从 Java 10 开始（Java 8u191+ 移植），JVM 支持容器感知。
推荐参数：
```bash
java -XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -jar app.jar
```
这样 JVM 会自动检测容器的 Limit，并将 Heap 最大值设为 Limit 的 75%，无需硬编码 `-Xmx`。

### 3.3 限制直接内存和线程数
如果使用了 Netty，建议显式限制直接内存：
`-XX:MaxDirectMemorySize=256m`

## 4. 辅助排查工具
如果调整参数后依然 OOM，需要深入分析。
1.  **Native Memory Tracking (NMT)**:
    启动参数加 `-XX:NativeMemoryTracking=summary`。
    进入容器执行 `jcmd <pid> VM.native_memory summary` 查看各部分内存占用。
2.  **Dump 分析**:
    虽然 OOMKilled 没有 Dump，但可以配置 `-XX:+HeapDumpOnOutOfMemoryError` 应对 JVM OOM。
    对于堆外泄漏，可能需要 `gperftools` 或 `jemalloc` 等系统级工具。

## 总结
在 K8s 中运行 Java：
1. 不要让 Xmx 等于 Pod Limit。
2. 使用 `-XX:MaxRAMPercentage=75.0` 替代硬编码。
3. 关注非堆内存的占用。
