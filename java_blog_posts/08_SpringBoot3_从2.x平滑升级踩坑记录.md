# Spring Boot 3：AOT 编译与 GraalVM 的技术内幕

## 背景
Spring Boot 3.0 的发布标志着 Java 生态进入了 Cloud Native 的新纪元。最引人注目的特性莫过于对 **GraalVM Native Image** 的正式支持。
从 JVM 的 JIT (Just-In-Time) 到 Native 的 AOT (Ahead-Of-Time)，这不仅仅是启动速度的提升，更是 Java 运行机制的根本性变革。

## 1. 为什么 AOT 启动这么快？
传统 JVM 启动时，需要：
1.  加载 JVM 运行时（庞大）。
2.  加载 Class 文件，验证字节码。
3.  解释执行字节码。
4.  C1/C2 编译器（JIT）根据热点探测，将字节码编译为机器码。

**AOT (GraalVM)** 则是在构建阶段（Build Time）：
1.  **静态分析**: 从 Main 方法开始，扫描所有可达的代码路径。
2.  **堆快照 (Heap Snapshot)**: 将静态变量、常量池直接写入可执行文件的 Data 段。
3.  **预编译**: 直接生成特定 CPU 架构的机器码。

结果：应用启动时，几乎没有“初始化”过程，直接映射内存并执行机器码。启动时间从秒级缩短到 **毫秒级**。

## 2. 踩坑记录：动态性的代价
Java 的强大在于动态性（反射、动态代理、序列化），但这正是 AOT 的噩梦。静态分析无法推断运行时的动态调用。

### 2.1 反射与 Hints
如果使用了 `Class.forName("com.mysql.cj.jdbc.Driver")`，GraalVM 在编译时不知道这个类会被用到，会把它由“摇树优化 (Tree Shaking)”剔除。
**解决方案**：Spring Boot 3 引入了 **Runtime Hints API**。
```java
public class MyHints implements RuntimeHintsRegistrar {
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        hints.reflection().registerType(MyEntity.class, MemberCategory.DECLARED_FIELDS);
    }
}
```
Spring 框架内部已经为大多数标准库注册了 Hints，但对于第三方库或自己的反射代码，必须手动注册，否则运行时报 `ClassNotFoundException`。

### 2.2 CGLIB vs JDK Proxy
Spring AOP 默认使用 CGLIB（字节码生成）。但在 Native Image 中，动态生成字节码是不被支持的（或者非常受限）。
Spring Boot 3 倾向于在 AOT 处理阶段生成代理类。这意味着如果你的 Bean 没有实现接口，Spring 需要在编译期就生成子类。这要求开发者对 Bean 的定义更加规范。

## 3. 可观测性：Micrometer Tracing
Spring Boot 3 移除了 Spring Cloud Sleuth，全面拥抱 **Micrometer Tracing**。
这不仅仅是改名，而是基于 **Observation API** 的重构。

```java
Observation.createNotStarted("my.operation", registry)
    .lowCardinalityKeyValue("user.type", "vip")
    .observe(() -> {
        // 业务逻辑
    });
```
**技术细节**：
Observation API 将 **Metrics** (Timer, Counter) 和 **Tracing** (Span) 统一了。
当你开启一个 Observation，它会自动开启一个 Trace Span，并在结束时记录 Metric。这种统一模型大大降低了在代码中埋点的复杂度，同时保证了监控和链路追踪数据的一致性。

## 总结
升级 Spring Boot 3 不仅仅是为了跟风。
理解 AOT 的**封闭世界假设 (Closed World Assumption)**，理解 Observation API 的**统一观测模型**，有助于我们编写出更规范、更云原生友好的 Java 代码。
Java 正在变“轻”，而我们需要变“强”。
