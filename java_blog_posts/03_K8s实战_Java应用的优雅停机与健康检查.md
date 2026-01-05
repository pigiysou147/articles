# K8s 实战：Java 应用的优雅停机与健康检查

## 背景
刚把 Java 应用迁移到 Kubernetes 时，我们经常遇到发布时服务出现短暂 502 错误，或者 Pod 一直重启的问题。经过深入排查，发现是**优雅停机**和**健康检查**配置不当导致的。

## 1. 为什么发布时会报错？
当 K8s 销毁一个 Pod 时，流程如下：
1.  Endpoint Controller 将 Pod IP 从 Service 的 Endpoints 列表中剔除（停止流量转发）。
2.  同时，Kubelet 向容器发送 `SIGTERM` 信号。
3.  应用收到信号开始清理资源（关闭连接池、停止接收新请求）。

**问题在于**：步骤 1 和 2 是异步并行的。如果应用先关闭了，但流量还在转发过来，用户就会收到 502。

## 2. 实现优雅停机 (Graceful Shutdown)

### 2.1 Spring Boot 配置
Spring Boot 2.3+ 内置了优雅停机支持，只需在 `application.yml` 中配置：

```yaml
server:
  shutdown: graceful # 开启优雅停机
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s # 设置缓冲时间
```
开启后，Spring Boot 在收到 `SIGTERM` 时，会拒绝新请求，但会等待现有请求处理完成。

### 2.2 K8s preStop Hook
为了绝对安全，我们通常在 K8s 层加一个 `preStop` 钩子，强制睡几秒，确保流量切换动作已完成。

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]
```
这样，Pod 在收到销毁指令后，先睡10秒（此时 K8s 正在切断流量），然后再发送 `SIGTERM` 给 Java 进程。

## 3. 健康检查的坑 (Liveness vs Readiness)
K8s 提供了两种探针，很多人容易混淆：

- **LivenessProbe (存活探针)**：如果失败，K8s 会**重启** Pod。
- **ReadinessProbe (就绪探针)**：如果失败，K8s 会**停止流量转发**到该 Pod。

### 3.1 最佳实践
不要把数据库连接检查放在 LivenessProbe 里！
如果数据库抖动，所有 Pod 的 Liveness 检测失败，会导致所有 Pod 同时重启，引发雪崩。

**推荐配置：**
*   **StartupProbe**: (K8s 1.16+) 专门用于启动慢的应用，避免启动时被 Liveness 杀掉。
*   **LivenessProbe**: 仅检查应用死锁或死循环。直接返回 200 OK 即可，或者简单检查内部状态。
*   **ReadinessProbe**: 检查外部依赖（DB、Redis），确保依赖正常才接收流量。

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 20
  periodSeconds: 10
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 40
  periodSeconds: 20
```

## 总结
Kubernetes 部署 Java 应用不仅仅是写个 Dockerfile 那么简单。
1. 配置 `server.shutdown: graceful`。
2. 配置 `preStop` 钩子 `sleep` 几秒。
3. 严格区分 Liveness 和 Readiness 探针的用途。
这样才能保证发布的丝般顺滑。
