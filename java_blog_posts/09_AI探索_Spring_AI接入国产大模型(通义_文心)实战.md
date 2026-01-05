# AI 进阶：Spring AI 的抽象哲学与 Function Calling 原理

## 背景
在对接大模型时，我们很容易陷入具体的 API 细节中：OpenAI 的 JSON 结构、通义千问的 HTTP 头、文心一言的签名算法。
**Spring AI** 的核心价值在于 **Portable API**。它不仅屏蔽了异构模型的差异，还通过高层抽象实现了 **Function Calling (工具调用)** 的标准化。
本文将深入探讨 Spring AI 的 **ChatModel 抽象**、**Prompt 模板引擎** 以及 **Function Calling 的反射机制**。

## 1. 核心抽象：ChatModel 与 Prompts

Spring AI 的设计灵感来源于 JDBC。
*   `ChatClient` $\approx$ `JdbcTemplate`
*   `Prompt` $\approx$ `SQL Statement`
*   `ChatResponse` $\approx$ `ResultSet`

这种设计允许开发者在不修改业务代码的情况下，通过更改配置（YAML）从 OpenAI 切换到 Qwen 或 Ollama。这对于信创项目至关重要。

### 1.1 Prompt Template 引擎
Prompt 不仅仅是字符串拼接。Spring AI 提供了类似 `StringTemplate` 的引擎。
```java
PromptTemplate template = new PromptTemplate("请扮演{role}，回答{topic}。");
Prompt prompt = template.create(Map.of("role", "律师", "topic", "合同法"));
```
它支持 **ST (StringTemplate)** 语法，甚至可以处理 List 和 Map 等复杂数据结构的序列化，确保注入到 Prompt 中的上下文格式正确。

## 2. Function Calling：让 AI 拥有“手”

大模型本身是封闭的。Function Calling 允许 AI 在需要时“回调”我们的 Java 代码（比如查询天气、查数据库、执行 shell）。

### 2.1 协议层：OpenAI Functions 规范
在 HTTP 层面，Function Calling 依赖于 JSON Schema。
当我们请求 LLM 时，除了 prompt，还带了一个 `tools` 字段：
```json
{
  "messages": [{"role": "user", "content": "北京天气如何？"}],
  "tools": [{
    "type": "function",
    "function": {
      "name": "getCurrentWeather",
      "description": "查询某地的天气",
      "parameters": {
        "type": "object",
        "properties": {
          "location": { "type": "string", "description": "城市名" }
        },
        "required": ["location"]
      }
    }
  }]
}
```
这段 JSON 告诉 LLM：“我有这个工具，你会用吗？”

### 2.2 Spring AI 的实现原理
Spring AI 极大地简化了这个过程。我们只需要写一个标准的 Java Function：

```java
@Bean
@Description("查询某地的天气") // 非常重要！会变成 JSON 中的 description
public Function<WeatherRequest, WeatherResponse> weatherFunction() {
    return request -> weatherService.query(request.location());
}
```

**内部执行流程**：
1.  **Schema Generation**: Spring AI 启动时，扫描所有注册的 Function。利用反射解析 Input Class (`WeatherRequest`) 的字段类型，自动生成对应的 JSON Schema。
2.  **Request Injection**: 发送 Chat 请求时，自动把 Schema 塞入 `tools` 参数。
3.  **LLM Reasoning**: LLM 分析用户问题，发现需要查天气。它**暂停生成文本**，而是返回一个特殊的 `finish_reason: tool_calls`，并附带 JSON：`{"name": "weatherFunction", "arguments": "{\"location\": \"Beijing\"}"}`。
4.  **Reflection Call**: Spring AI 拦截到这个响应。
    *   根据 `name` 找到 Bean。
    *   将 `arguments` JSON 反序列化为 Java 对象 (`WeatherRequest`)。
    *   反射调用 `apply` 方法。
5.  **Recursive Execution**: 拿到 Java 方法的返回值（"25度"），Spring AI 将其包装成一个新的 Message (`role: tool`)，再次发送给 LLM。
6.  **Final Response**: LLM 结合上下文和工具结果，生成最终回答：“北京今天25度，天气不错。”

## 3. 国产模型的适配挑战

虽然 Spring AI 抽象得很好，但国产模型的 Function Calling 实现参差不齐。

*   **通义千问 (Qwen)**: 对 OpenAI 格式支持较好，JSON 输出稳定。
*   **文心一言**: 早期的 Function Calling 协议比较独特，但在 4.0 版本后趋于标准。

### 3.1 复杂参数处理
如果 Function 的入参是一个复杂的嵌套对象，LLM 可能会生成错误的 JSON。
**最佳实践**：
使用 `@JsonPropertyDescription` 注解详细描述每个字段的用途和格式。
```java
public record WeatherRequest(
    @JsonPropertyDescription("城市名称，如 Beijing, Shanghai") String location,
    @JsonPropertyDescription("单位，可选 Celsius 或 Fahrenheit") String unit
) {}
```
这些注解会被 Spring AI 提取并放入 JSON Schema 的 `description` 字段，显著提高 LLM 的理解准确率。

## 总结
Spring AI 不仅仅是一个 HTTP 客户端包装器。它通过精妙的抽象，将 **Prompt Engineering** 和 **Function Calling** 变成了标准的 Java 开发范式。
*   理解 **JSON Schema** 的生成机制。
*   掌握 **反射调用** 的流程。
*   熟悉 **LLM 交互** 的多轮对话逻辑。
这能让我们更自如地驾驭 AI Agent 的开发，构建出真正智能的应用。
