# ES 实战：基于 ELK 的实时日志分析平台搭建

## 背景
传统的排查方式是 SSH 登服务器，用 `grep` 查日志。微服务架构下，一个请求经过几十个服务，几十个实例，根本没法查。
企业通常需要一套**集中式日志系统**。ELK (Elasticsearch, Logstash, Kibana) 是业界的标准答案。

## 1. 架构演进

### v1.0：直接推送到 ES
Logback -> Logstash -> Elasticsearch -> Kibana
*   *缺点*：流量高峰期 Logstash 或 ES 扛不住，会丢日志。

### v2.0：引入 Kafka 缓冲 (当前主流)
Logback -> Filebeat -> **Kafka** -> Logstash -> Elasticsearch -> Kibana
*   *Filebeat*: 极其轻量，部署在应用服务器收集日志文件。
*   *Kafka*: 削峰填谷，保证日志不丢失。
*   *Logstash*: 负责从 Kafka 消费，进行复杂的过滤、Grok 解析，再发给 ES。

## 2. 关键配置实战

### 2.1 Java 应用日志格式化
为了方便 ES 索引，建议 Java 应用直接输出 JSON 格式日志。
使用 `net.logstash.logback:logstash-logback-encoder` 依赖。

```xml
<!-- logback-spring.xml -->
<appender name="JSON_CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <customFields>{"app_name": "my-service"}</customFields>
    </encoder>
</appender>
```
这样输出的每一行都是标准的 JSON，Logstash 甚至不需要写 Grok 规则。

### 2.2 TraceID 链路追踪
为了把微服务之间的日志串起来，必须通过 MDC 注入 TraceID。
配合 Spring Cloud Sleuth 或 Micrometer Tracing，自动在日志中加入 `traceId` 和 `spanId`。
Kibana 中搜索 `traceId="abc"`，即可看到全链路日志。

## 3. 遇到的坑

### 3.1 字段类型冲突
服务 A 的 `status` 字段是数字，服务 B 的 `status` 是字符串。
当它们写入同一个 Index 时，ES 会报错。
*   *解法*：为不同服务建立不同的 Index（如 `log-service-a-2023.10.01`），或者严格规范字段命名。

### 3.2 磁盘爆炸
日志数据量惊人，必须设置过期策略。
使用 ILM (Index Lifecycle Management)：
1.  Hot 阶段：索引可写。
2.  Delete 阶段：超过 7 天自动删除索引。

## 总结
ELK 不仅仅是装几个软件。核心价值在于：
1. **标准化**：全公司统一日志格式 (JSON) 和 TraceID。
2. **可视化**：用 Kibana 制作 Dashboard，甚至可以做业务监控（如每分钟下单量）。
3. **稳定性**：Kafka 缓冲和 ILM 策略至关重要。
