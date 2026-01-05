# AI 实战：基于 DeepSeek 私有化部署构建政务政策问答库

## 背景
国内政务项目对数据安全有着极高的要求，“数据不出域”是底线。因此，SaaS 类的 AI 服务（如 ChatGPT、文心一言公有云版）通常无法直接用于处理涉密或敏感的内部公文。
但业务部门又迫切希望利用 AI 技术实现“政策文件智能问答”。本文将介绍如何利用国产开源模型 **DeepSeek (深度求索)** 配合 **Ollama** 实现完全离线的 RAG（检索增强生成）系统。

## 1. 为什么选择 DeepSeek + Ollama？

*   **国产之光 DeepSeek-Coder-V2**: DeepSeek 在中文理解和逻辑推理能力上表现优异，且开源了权重，允许商用。
*   **Ollama**: 一个极其轻量级的 LLM 运行框架，支持一键本地启动模型，提供兼容 OpenAI 格式的 API。
*   **纯内网环境**: 不需要访问外网，所有推理都在本地服务器（GPU/CPU）完成。

## 2. 环境搭建 (CentOS 7/Ubuntu)

### 2.1 部署 Ollama
在内网服务器上（建议配置 Nvidia 显卡，如 T4 或 A10）：
```bash
# 如果无法联网，需下载离线安装包
curl -fsSL https://ollama.com/install.sh | sh
```

### 2.2 加载模型
我们需要提前在有网环境下载好模型文件（`.gguf` 格式），然后传输到内网。
```bash
# 启动 DeepSeek 7B 版本（适合显存 16G 左右的机器）
ollama run deepseek-coder:6.7b
```
启动成功后，Ollama 会在 `localhost:11434` 监听。

## 3. RAG 核心架构：Spring AI + Vector Store

为了让 AI 理解政策文件，我们需要搭建 RAG 链路。

### 3.1 架构图
`公文PDF` -> `ETL(文本提取)` -> `Embedding(向量化)` -> `Milvus(向量库)` -> `检索(Search)` -> `LLM(DeepSeek)` -> `答案`

### 3.2 向量数据库选型
在政务场景下，通常推荐使用 **Milvus** 或 **PostgreSQL (pgvector)**。
这里以 Milvus 为例，它支持分布式部署，稳定性高。

### 3.3 代码实现 (Spring AI)

首先，配置 Spring AI 连接本地 Ollama：
```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        model: deepseek-coder:6.7b
```

然后，实现文档向量化（Embedding）：
```java
// 使用 Spring AI 的 VectorStore 抽象
@Autowired
VectorStore vectorStore;

public void ingestDocs(List<File> policyFiles) {
    for (File file : policyFiles) {
        // 1. 读取 Tika/PDFBox 解析文本
        Document doc = new Document(parsePdf(file));
        // 2. 存入向量库（自动调用 Embedding 模型）
        vectorStore.add(List.of(doc));
    }
}
```
*注意*：Embedding 模型也必须本地化！推荐使用 `m3e-base` (Moka Massive Mixed Embedding)，它是目前中文语义匹配效果最好的开源模型之一，同样可以通过 Ollama 或 ONNX 运行。

### 3.4 问答服务
```java
public String askPolicy(String question) {
    // 1. 检索相似文档
    List<Document> similarDocs = vectorStore.similaritySearch(
        SearchRequest.query(question).withTopK(3)
    );
    
    // 2. 构建提示词
    String context = similarDocs.stream().map(Document::getContent).collect(Collectors.joining("\n"));
    String prompt = "基于以下政策文件内容回答问题，不要通过互联网检索：\n" + context + "\n问题：" + question;
    
    // 3. 调用本地 DeepSeek
    return chatClient.call(prompt);
}
```

## 4. 常见的优化策略

1.  **PDF 表格解析**: 政务公文中包含大量表格，普通的 PDF 解析器会乱码。建议使用 **OCR** (如 PaddleOCR) 专门处理表格区域。
2.  **切片策略**: 简单的按字符切分会打断语义。建议按“章节”或“条款”切分（正则匹配 `第一条`、`1.1` 等）。
3.  **幻觉控制**: 在 Prompt 中明确约束：“如果文件中没有提到，请直接回答‘文件中未查询到相关规定’，严禁编造。”

## 总结
通过 **DeepSeek (推理)** + **M3E (向量)** + **Milvus (存储)** + **Spring AI (编排)**，我们成功在政务内网构建了一套完全私有化的政策问答助手。数据全程不落地，既满足了安全合规，又提升了办公效率。
