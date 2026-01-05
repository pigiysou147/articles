# AI 进阶：Spring AI 的抽象哲学与 Function Calling 原理

## 背景
在对接大模型时，我们很容易陷入具体的 API 细节中：OpenAI 的 JSON 结构、通义千问的 HTTP 头、文心一言的签名算法。
**Spring AI** 的核心价值在于 **Portable API**。它不仅屏蔽了异构模型的差异，还通过高层抽象实现了 **Function Calling (工具调用)** 的标准化。

## 1. 核心抽象：ChatModel 与 Prompts
Spring AI 的设计灵感来源于 JDBC。
*   `ChatClient` $\approx$ `JdbcTemplate`
*   `Prompt` $\approx$ `SQL Statement`
*   `ChatResponse` $\approx$ `ResultSet`

这种设计允许开发者在不修改业务代码的情况下，通过更改配置（YAML）从 OpenAI 切换到 Qwen 或 Ollama。这对于信创项目至关重要。

## 2. Function Calling：让 AI 拥有“手”
大模型本身是封闭的。Function Calling 允许 AI 在需要时“回调”我们的 Java 代码（比如查询天气、查数据库）。

### 2.1 原理剖析
1.  **Schema 生成**: Spring AI 会扫描注册的 Java Bean（实现了 `java.util.function.Function`），利用反射分析其入参类型，生成 **JSON Schema**。
2.  **Prompt 注入**: 在发送给 LLM 的请求中，Spring AI 会自动附带 `tools` 字段，包含上述 Schema。
    ```json
    "tools": [{
      "type": "function",
      "function": {
        "name": "getCurrentWeather",
        "parameters": { "type": "object", "properties": { "location": ... } }
      }
    }]
    ```
3.  **LLM 决策**: LLM 分析用户问题（"北京天气如何？"），发现需要调用工具，于是返回一个特殊的 `tool_calls` 响应，而不是普通文本。
4.  **反射执行**: Spring AI 拦截到这个响应，提取函数名和参数，反射调用 Java 方法。
5.  **递归调用**: 拿到 Java 方法的返回值（"25度"），Spring AI 再次将其作为 Context 发送给 LLM，LLM 最终生成自然语言回答。

### 2.2 实战代码
```java
@Configuration
public class ToolsConfig {
    @Bean
    @Description("查询某地的天气情况") // 这个描述会被 AI 看到，非常重要！
    public Function<WeatherRequest, WeatherResponse> weatherFunction() {
        return request -> weatherService.query(request.location());
    }
}
```
我们只需要写标准的 Java Function，Spring AI 负责所有的协议转换和反射调用。

## 3. 国产模型的适配挑战
虽然 Spring AI 抽象得很好，但国产模型的 Function Calling 实现参差不齐。
*   **通义千问 (Qwen)**: 对 OpenAI 格式支持较好，JSON 输出稳定。
*   **文心一言**: 早期的 Function Calling 协议比较独特，但在 4.0 版本后趋于标准。

在使用 Spring AI Alibaba 时，底层依然依赖于 JSON Schema 的准确性。建议在定义 Java Bean 时，使用 `@JsonPropertyDescription` 详细描述每个字段的用途，这能显著提高 AI 调用工具的成功率。

## 总结
Spring AI 不仅仅是一个 HTTP 客户端包装器。它通过精妙的抽象，将 **Prompt Engineering** 和 **Function Calling** 变成了标准的 Java 开发范式。
理解其背后的反射机制和 JSON Schema 生成逻辑，能让我们更自如地驾驭 AI Agent 的开发。
