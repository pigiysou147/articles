# K8s 实战：从 syscall 层面理解 Java 应用的优雅停机

## 背景
在 K8s 环境下，Pod 的生命周期管理远比传统虚拟机复杂。我们常遇到的 502 Bad Gateway 或连接重置问题，往往源于对 Linux 信号处理机制和 TCP 状态流转理解不深。
本文将跳过配置层面，深入 **System Call (系统调用)** 和 **TCP 协议栈**，剖析优雅停机的本质。

## 1. 信号传播与 PID 1 的僵尸进程

### 1.1 容器内的进程树
当 K8s 删除 Pod 时，Container Runtime (如 containerd) 会向容器内的 PID 1 进程发送 `SIGTERM`。
**技术陷阱**：
如果你的 Dockerfile 启动命令是 `CMD ["java", "-jar", "app.jar"]`，或者是 `CMD ./start.sh`，那么 `sh` 可能会成为 PID 1，而 Java 进程成为 PID 1 的子进程。

**内核机制**：
在 Linux 内核中，PID 1 (init 进程) 拥有特权。内核会**屏蔽**掉发送给 PID 1 的默认信号处理（Default Signal Handlers），除非 PID 1 显式注册了信号处理函数。
Shell 脚本通常不会转发信号。因此，当 K8s 发送 `SIGTERM` 给 `sh` 时，`sh` 无动于衷，Java 子进程也收不到信号。最终，K8s 等待 `terminationGracePeriodSeconds` (默认30s) 超时，发送 `SIGKILL` 强杀。

**后果**：Java 进程直接被 kill -9，没有机会关闭 DB 连接池，也没有机会处理完 inflight 请求。

**最佳实践**：
使用 `exec` 模式启动：
```dockerfile
ENTRYPOINT ["/bin/sh", "-c", "exec java -jar app.jar"]
```
`exec` 系统调用会用新进程（Java）替换当前进程（Shell），保留 PID。这样 Java 就变成了 PID 1，可以直接接收并处理 `SIGTERM`。

## 2. 深入 Spring Boot 的 Shutdown Hook

当 Java 进程正确收到 `SIGTERM` 后，JVM 会触发注册的 Shutdown Hook。
在 Spring Boot 中，这个过程由 `SpringApplicationShutdownHook` 驱动。让我们看看源码层面的执行顺序：

1.  **发布 ContextClosedEvent**：通知监听器，容器即将关闭。
2.  **停止 Web 容器 (Tomcat/Netty)**：
    *   **关键点**：Tomcat 会调用 `Connector.pause()`，停止 `Acceptor` 线程，不再 `accept` 新的 TCP Socket。
    *   **优雅窗口期**：对于 Keep-Alive 的长连接，Tomcat 的线程池会继续处理当前的请求，直到处理完毕或超时。
3.  **销毁 Bean**：按照依赖关系的逆序（Reverse Topology）销毁 Bean。先销毁 Controller，最后销毁 DataSource。

## 3. TCP 协议栈的“四次挥手”与 K8s 的竞态条件

为什么配置了优雅停机，还是会有 502？
这涉及 K8s 的网络模型。Pod 删除是**异步并行**的操作：

### 3.1 竞态条件 (Race Condition)
1.  **控制面 (APIServer)**：标记 Pod 为 Terminating。
2.  **网络面 (Kube-proxy/Ingress)**：Endpoint Controller 监听到变化，将 Pod IP 从 Service Endpoints 中移除。Kube-proxy 更新 iptables/IPVS 规则。
3.  **计算面 (Kubelet)**：Kubelet 收到事件，立即给容器发送 `SIGTERM`。

**问题核心**：步骤 2 和 3 是并行的，且步骤 2 往往比较慢（涉及集群状态传播）。
如果 Kubelet 先杀掉了应用，但 Ingress 的 Nginx 路由规则还没刷新，流量依然会被转发过来。此时 Pod 端口已关闭，TCP 握手直接收到 **RST (Reset)** 包，或者应用拒绝连接，导致 502/504。

### 3.2 PreStop Hook 的本质
我们加 `sleep 10` 的 PreStop，实际上是给 K8s 的**分布式状态传播**预留时间。
```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]
```
这段时间内，Pod 依然是 Running 状态，依然可以处理请求。我们希望在 `sleep` 结束时，集群内所有的 LoadBalancer 和 iptables 规则都已经剔除了该 Pod IP。

## 4. TCP 状态机：CLOSE_WAIT 与 TIME_WAIT

在停机排查中，`netstat` 是你的好朋友。我们需要关注两种状态。

### 4.1 CLOSE_WAIT (被动关闭方)
如果 Pod 日志没报错，但监控显示连接数居高不下，且大量 `CLOSE_WAIT`。
**原因**：客户端（如 Gateway）发送了 `FIN` 包（主动关闭），服务端（Pod）的 TCP 栈回复了 `ACK`，进入 `CLOSE_WAIT` 状态。
此时，服务端应用层**必须**显式调用 `socket.close()`，发送第二个 `FIN` 包，才能完成四次挥手。
如果代码中有 Bug，或者 Shutdown Hook 执行卡死（比如在等一个死锁的线程），`close()` 永远不会被调用，连接就卡在 `CLOSE_WAIT`。

### 4.2 TIME_WAIT (主动关闭方)
如果 Pod 是作为客户端（比如调用 DB 或其他微服务），在关闭连接池时，Pod 会主动发送 `FIN`。
完成挥手后，连接会进入 `TIME_WAIT` 状态，保持 `2 * MSL` (通常 60s)。
**影响**：在高并发短连接场景下，如果优雅停机策略导致短时间内大量主动关闭，Pod 可能会消耗光临时端口（Ephemeral Ports）。虽然在 Pod 销毁场景下问题不大（因为 IP 也回收了），但在滚动发布期间，需要注意 Node 级别的 conntrack 表是否满了。

## 总结
优雅停机不仅仅是改个 YAML 配置，它是对 **Linux 进程管理**（PID 1、信号屏蔽）、**TCP 状态机**（RST、FIN、CLOSE_WAIT）以及 **K8s 分布式协调机制**的综合考量。
作为工程师，当我们看到 502 时，脑海中应该浮现出 SYN, ACK, FIN, RST 的报文交互图，而不是盲目地调大 sleep 时间。
