# AI 实战：从量化原理到 HNSW——构建私有化知识库的硬核技术

## 背景
在政务或金融等敏感领域，“数据不出域”是死命令。这意味着我们无法使用云端的 GPT-4，必须在本地部署 LLM。
但这不仅是运行一个 Docker 容器那么简单。如何在有限的显存下跑大模型？如何让检索速度在百万级文档中达到毫秒级？
本文将深入探讨 **GGUF 量化**、**HNSW 索引算法** 以及 **KV Cache** 管理。

## 1. 显存的魔法：模型量化 (Quantization)

DeepSeek-Coder-V2 有 236B 参数，即使是 Lite 版也有 16B。
如果使用 **FP16 (Half Precision)** 加载：
$$ \text{Size} = 16 \times 10^9 \times 2 \text{Bytes} \approx 32 \text{GB} $$
这超过了单张消费级显卡（如 RTX 4090 24G）的极限。

### 1.1 对称与非对称量化
我们使用的 `.gguf` 文件，通常采用了 **4-bit 量化 (Int4)**。
量化的本质是将高精度的浮点数映射到低精度的整数。

**非对称量化公式**：
$$ Q = \text{round}(S \times (R - Z)) $$
其中 $R$ 是原始浮点数，$S$ 是缩放因子 (Scale)，$Z$ 是零点 (Zero Point)。

**Block-wise Quantization**：
为了减少精度损失，GGUF 不会对整个模型使用同一个 $S$ 和 $Z$。它将权重矩阵切分为一个个 Block（例如 32 个权重一组），每组单独计算缩放因子。
这就是 `Q4_K_M` 中的 `K-Quant` 技术。它智能地混合了不同精度的量化策略：关键的 Attention 权重可能用 6-bit，而不重要的 FFN 权重用 4-bit。

### 1.2 效果验证
结果：16B 模型仅需 ~10GB 显存。
虽然参数精度下降了，但模型的**困惑度 (Perplexity)** 仅增加了 1%-2%。对于代码生成和文档问答任务，这种损失几乎感知不到。

## 2. 向量检索的内核：HNSW 算法

RAG 的核心是向量检索。在 Milvus 或 Chroma 中，最常用的索引类型是 **HNSW (Hierarchical Navigable Small World)**。

### 2.1 为什么不用暴力搜索 (Flat Search)？
暴力搜索需要计算 Query 向量与库中 100 万个向量的余弦相似度（点积运算）。
复杂度 $O(N \times D)$，其中 D 是向量维度（如 1024）。在百万数据下，延迟高达数百毫秒。

### 2.2 HNSW 原理解析
HNSW 是一种基于图的 **ANN (Approximate Nearest Neighbor)** 算法。
它结合了 **概率跳表 (Skip List)** 和 **NSW (Navigable Small World)** 图的特性。

**结构**：
它构建了一个**多层图结构**。
*   **第 0 层 (Bottom Layer)**: 包含所有数据节点，构建了细粒度的近邻连接。
*   **第 N 层 (Top Layer)**: 只有少量的入口节点，节点之间连接稀疏，跨度大。

**搜索过程**：
1.  从顶层入口出发。
2.  利用贪心算法，寻找距离 Query 最近的节点。
3.  一旦当前层找不到更近的节点，就“下沉”到下一层。
4.  在第 0 层进行精细搜索，找到最终的 Top K。

这种搜索方式就像“坐高铁到达城市 -> 坐地铁到达区域 -> 骑单车到达目的地”，复杂度降低到 $O(\log N)$。

**内存代价**：
HNSW 虽然快，但极吃内存。它需要维护大量的邻接表。对于千万级向量，建议使用 **IVF_FLAT** 或 **IVF_PQ** (Product Quantization) 索引来压缩内存。

## 3. RAG 系统的上下文窗口管理

DeepSeek-Coder 支持 128k Context Window，但这并不意味着我们可以无脑塞入整本书。

### 3.1 KV Cache 瓶颈
推理时，显存不仅存权重，还要存 **KV Cache**（Key-Value Cache）。
Transformer 在生成第 $t$ 个 token 时，需要用到前 $t-1$ 个 token 的 Key 和 Value 矩阵来计算 Attention。
Sequence Length 越长，KV Cache 越大。
$$ \text{KV\_Size} = 2 \times \text{Layers} \times \text{Hidden\_Dim} \times \text{Seq\_Len} \times \text{Batch\_Size} \times \text{Bytes} $$
对于 128k 长度，KV Cache 可能高达几十 GB，直接撑爆显存。

### 3.2 分块与重排 (Chunking & Rerank)
为了节省 Token，我们必须精简 Context。
1.  **Chunking**: 按语义切分文档（如 RecursiveCharacterTextSplitter）。
2.  **Retrieval**: 粗排。从向量库召回 Top 50 个相关切片。
3.  **Rerank**: 精排。使用专门的 Cross-Encoder 模型（如 bge-reranker-v2）对这 50 个切片与 Query 进行深度交互打分。
4.  **Selection**: 选出得分最高的 Top 5，喂给 LLM。

Rerank 模型虽然慢，但它能理解复杂的语义匹配，显著减少了“噪音”对 LLM 的干扰，解决了 RAG 中的“迷失中间 (Lost in the Middle)”现象。

## 4. Spring AI 的底层抽象

Spring AI 屏蔽了不同 Vector Store 的差异。
它通过 `Document` 对象统一封装了 `Content` (文本) 和 `Metadata` (元数据)。

**Function Calling 实现**：
当模型觉得需要查数据库时，它会输出一个特定的 JSON。Spring AI 解析这个 JSON，反射调用本地 Java Bean 的方法，拿到结果后再塞回给 LLM。
这个过程体现了 **Agent** 的雏形：LLM 是大脑，Java 代码是手脚。

## 总结
构建私有化 AI 知识库，本质上是在做 **Time-Space Trade-off**。
*   用 **量化** 牺牲精度换取显存。
*   用 **HNSW** 牺牲内存换取速度。
*   用 **Rerank** 牺牲计算时间换取准确率。
理解这些底层算法的边界，才能设计出既快又准的企业级 RAG 系统。
