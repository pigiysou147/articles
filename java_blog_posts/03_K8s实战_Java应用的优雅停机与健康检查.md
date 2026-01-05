# K8s 实战：从 syscall 层面理解 Java 应用的优雅停机

## 背景
在 K8s 环境下，Pod 的生命周期管理远比传统虚拟机复杂。我们常遇到的 502 Bad Gateway 或连接重置问题，往往源于对 Linux 信号处理机制和 TCP 状态流转理解不深。
本文将跳过配置层面，深入 **System Call (系统调用)** 和 **TCP 协议栈**，剖析优雅停机的本质。

## 1. 信号传播与 PID 1 的僵尸进程
当 K8s 删除 Pod 时，Container Runtime (如 containerd) 会向容器内的 PID 1 进程发送 `SIGTERM`。
**技术陷阱**：
如果你的 Dockerfile 启动命令是 `CMD ["java", "-jar", "app.jar"]`，或者是 `CMD ./start.sh`，那么 `sh` 可能会成为 PID 1，而 Java 进程成为 PID 1 的子进程。
Linux 中，信号不会自动传递给子进程。如果 `sh` 没有处理信号转发，Java 进程就永远收不到 `SIGTERM`，直到超时后被 `SIGKILL` 强杀。
**最佳实践**：使用 `exec` 模式启动：`ENTRYPOINT ["/bin/sh", "-c", "exec java -jar app.jar"]`，这会让 Java 进程直接替换 Shell 成为 PID 1。

## 2. 深入 Spring Boot 的 Shutdown Hook
当 Java 进程收到 `SIGTERM` 后，JVM 会触发注册的 Shutdown Hook。
在 Spring Boot 中，这个过程由 `SpringApplicationShutdownHook` 驱动。让我们看看源码层面的执行顺序：

1.  **发布 ContextClosedEvent**：通知监听器，容器即将关闭。
2.  **停止 Web 容器 (Tomcat/Netty)**：
    *   **关键点**：Tomcat 会停止 `Acceptor` 线程，不再接受新的 TCP 连接。
    *   但是！对于 Keep-Alive 的长连接，Tomcat 会继续处理当前的请求（如果在 graceful 窗口期内）。
3.  **销毁 Bean**：按照依赖关系的逆序（Reverse Topology）销毁 Bean。先销毁 Controller，最后销毁 DataSource。

## 3. TCP 协议栈的“四次挥手”与 K8s 的竞态条件
为什么配置了优雅停机，还是会有 502？
这涉及 K8s 的网络模型。Pod 删除是**异步并行**的操作：
1.  **控制面**：Endpoint Controller 从 Service Endpoints 中移除 Pod IP。
2.  **数据面**：Kube-proxy 更新 iptables/IPVS 规则。
3.  **节点侧**：Kubelet 杀容器。

如果步骤 3 比步骤 2 快，Pod 已经死了，但 iptables 规则还在。流量转发过来，发现端口不通，或者 TCP 握手直接被 RST。

**PreStop Hook 的本质**：
我们加 `sleep 10` 的 PreStop，实际上是给 K8s 的**分布式状态传播**预留时间。我们希望在应用真正开始 Shutdown 之前，集群内的所有 iptables 规则都已经更新完毕，不再有新流量流向该 Pod。

## 4. 连接池的“半关闭”状态 (CLOSE_WAIT)
在停机过程中，另一个常见问题是 DB 连接池报错。
如果应用层先关闭了 DataSource，而此时还有残留的 HTTP 请求在处理中（Race Condition），这些请求拿不到 DB 连接就会抛错。

**HikariCP 的处理**：
HikariCP 注册了自己的 Shutdown Hook。Spring Boot 2.3+ 做了优化，确保 web 容器关闭（不再进流量）之后，才去关闭 DataSource。
但在排查问题时，我们可以通过 `netstat -ant | grep CLOSE_WAIT` 观察。如果发现大量 `CLOSE_WAIT`，说明对方（客户端）已经发送了 FIN，但应用侧（服务端）卡住了，没有执行 `close()`，这通常意味着应用逻辑还在处理耗时任务，优雅停机超时时间设置得太短。

## 总结
优雅停机不仅仅是改个 YAML 配置，它是对 **Linux 进程管理**、**TCP 状态机**以及 **K8s 分布式协调机制**的综合考量。
作为工程师，当我们看到 502 时，脑海中应该浮现出 SYN, ACK, FIN, RST 的报文交互图，而不是盲目地调大 sleep 时间。
