# AI 国产化：Spring AI 接入通义千问/文心一言实战

## 背景
随着 Spring AI 的发布，Java 终于有了官方的 AI 应用开发框架。但官方原生主要支持 OpenAI、Azure 等海外模型。
在国内政务和信创项目中，我们必须对接国产大模型，如**阿里通义千问 (Qwen)** 或 **百度文心一言 (Ernie)**。
好消息是，Spring AI Alibaba 等社区项目已经跟进，让我们能以标准化的方式接入国产模型。

## 1. 选型：Spring AI Alibaba
Spring AI Alibaba 是基于 Spring AI 规范实现的，专门适配阿里云通义大模型（Qwen）。
它保持了 API 的统一性，这意味着你现在的代码，未来可以零成本切换到其他模型（如 DeepSeek 或 Claude）。

## 2. 快速接入通义千问

### 2.1 依赖引入
需要引入 Spring AI Alibaba 的 Starter：
```xml
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-starter</artifactId>
    <version>1.0.0-M1</version>
</dependency>
```

### 2.2 配置 API Key
在 `application.yml` 中配置：
```yaml
spring:
  ai:
    alibaba:
      qwen:
        api-key: ${ALI_AI_API_KEY} # 建议放在环境变量中
        model: qwen-plus # 推荐使用 plus 或 max 版本
```

### 2.3 业务代码
完全不需要改动 Spring AI 的原生接口！
```java
@RestController
@RequestMapping("/ai")
public class GovAssistantController {

    private final ChatClient chatClient;

    public GovAssistantController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @GetMapping("/draft")
    public String draftDocument(@RequestParam String topic) {
        return chatClient.prompt()
                .user("请帮我起草一份关于" + topic + "的政务通知，要求格式规范，语气严肃。")
                .call()
                .content();
    }
}
```

## 3. 对接百度文心一言 (Ernie Bot)
如果项目指定使用百度文心一言，虽然目前没有官方 Starter，但我们可以利用 Spring AI 的 **OpenAI 兼容模式**，或者自定义 `ChatModel`。
目前文心一言推出了兼容 OpenAI 协议的接口（千帆平台），这让接入变得异常简单。

### 3.1 配置 Base URL
```yaml
spring:
  ai:
    openai:
      base-url: https://qianfan.baidubce.com/v2/ # 百度千帆兼容端点
      api-key: ${QIANFAN_API_KEY}
```
*注意*：部分参数可能不支持，需要详细查阅千帆文档。

## 4. 政务场景下的 Prompt 工程技巧

国产模型对中文语境的理解通常优于 GPT-3.5，但在政务场景下，仍需精细化调优 Prompt。

### 4.1 角色设定 (Role Playing)
```java
String systemPrompt = """
你是一名经验丰富的政府办公厅秘书。
你的任务是辅助起草公文。
你的输出必须遵循《党政机关公文处理工作条例》格式。
严禁使用口语化表达。
""";
```

### 4.2 结构化输出
公文处理常需要提取关键信息（如时间、地点、责任人）。
使用 `BeanOutputParser` 让模型直接返回 JSON。
```java
record MeetingSummary(String date, String location, List<String> attendees, String content) {}

BeanOutputParser<MeetingSummary> parser = new BeanOutputParser<>(MeetingSummary.class);
String prompt = "分析以下会议记录..." + parser.getFormat();
```
国产模型（尤其是 Qwen-Max）在遵循 JSON 格式指令方面表现非常出色。

## 5. 总结
Spring AI 的出现屏蔽了底层模型的差异。在信创背景下，我们可以灵活切换通义千问、文心一言甚至私有化模型。
**核心建议**：
1. 首选 **Spring AI Alibaba** 对接通义系列，生态支持最好。
2. 充分利用 Spring AI 的 **抽象接口**，避免代码与具体模型绑定。
3. 针对国产模型特性调整 Prompt，特别是**公文格式**的约束。
