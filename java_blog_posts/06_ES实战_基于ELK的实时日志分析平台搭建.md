# ES 实战：Logstash 队列机制与 Zero Copy 性能调优

## 背景
在构建 ELK (Elasticsearch, Logstash, Kibana) 日志平台时，我们常遇到的瓶颈不在 ES，而在 Logstash。
Logstash 作为 ETL 管道，如果配置不当，会成为吞吐量的短板。本文将深入 Logstash 的 **Pipeline 机制**、**Persistent Queues** 以及 Kafka 的 **Zero Copy** 原理。

## 1. Logstash 的内部模型：Pipeline
Logstash 的核心是一个 Processing Pipeline，包含三个阶段：Inputs -> Filters -> Outputs。
这些阶段之间通过 **Queue** 连接。

### 1.1 In-Memory Queue (默认)
默认情况下，Logstash 使用内存队列。
*   **优点**：速度极快。
*   **致命弱点**：如果 Logstash 进程崩溃或服务器断电，队列中未处理的日志会**永久丢失**。

### 1.2 Persistent Queue (PQ)
为了数据安全，我们可以开启磁盘持久化队列：
```yaml
queue.type: persisted
queue.max_bytes: 4gb
```
**技术细节**：PQ 使用了 **Page-mapped files** 技术。它直接将队列数据映射到磁盘文件，但操作系统会利用 Page Cache 进行加速。这意味着在正常负载下，PQ 的性能损耗非常小（接近内存读写），但提供了 Crash Safety。

## 2. Kafka 作为缓冲层的底层原理
为什么要在 Logstash 前面加 Kafka？除了削峰填谷，更深层的原因是 Kafka 的高吞吐设计。

### 2.1 Zero Copy (零拷贝)
当 Logstash 从 Kafka 消费数据时，Kafka 利用了 Linux 的 `sendfile` 系统调用。
数据路径：`Disk -> Kernel Buffer -> NIC Buffer (网卡)`。
**关键点**：数据**不需要**拷贝到 User Space（用户态），也**不需要** CPU 参与上下文切换。这使得 Kafka 能够打满网卡带宽，单机吞吐量轻松达到几十万 EPS。

### 2.2 Consumer Group rebalance
在部署多个 Logstash 实例消费同一个 Kafka Topic 时，必须注意 **Consumer Rebalance** 问题。
如果某个 Logstash 实例因为 GC 停顿（Stop-The-World）导致心跳超时，Kafka Coordinator 会认为它挂了，触发重平衡。这会导致所有 Logstash 实例暂停消费，严重影响实时性。
**调优**：
适当调大 `session.timeout.ms` 和 `max.poll.interval.ms`，给 JVM GC 留出喘息时间。

## 3. Java 日志的结构化陷阱
我们常说要打 JSON 日志，但怎么打有讲究。

### 3.1 Logback Encoder 的 Buffer
使用 `logstash-logback-encoder` 时，注意它是**异步**的。
如果日志产生速度超过了 TCP 发送给 Logstash 的速度，Logback 的 RingBuffer 会满。
**丢弃策略**：默认情况下，如果 Buffer 满了，Logback 会丢弃 `DEBUG` 和 `INFO` 级别的日志，保留 `WARN` 和 `ERROR`。
这是为了保护业务线程不被日志阻塞。了解这一点，在排查“日志为何凭空消失”时至关重要。

## 4. 链路追踪 TraceID 的传递
在微服务调用链中，MDC (Mapped Diagnostic Context) 是传递 TraceID 的核心。
但 MDC 是 ThreadLocal 的。当我们在代码中使用了 `@Async` 或线程池时，MDC 上下文会丢失。

**解决方案**：
需要重写 `TaskDecorator`，在主线程提交任务时，手动 snapshot MDC 的内容，并在子线程执行前 restore 回去。

```java
public class MdcTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        Map<String, String> contextMap = MDC.getCopyOfContextMap();
        return () -> {
            try {
                if (contextMap != null) MDC.setContextMap(contextMap);
                runnable.run();
            } finally {
                MDC.clear();
            }
        };
    }
}
```

## 总结
搭建日志平台不只是安装软件。你需要理解：
1.  **操作系统层**：Page Cache 和 Zero Copy。
2.  **中间件层**：Logstash 的 Queue 模型和 Kafka 的 Rebalance 机制。
3.  **应用层**：ThreadLocal 上下文传递和 Logback 的缓冲策略。
打通这三层，才能构建出高吞吐、高可靠的观测系统。
