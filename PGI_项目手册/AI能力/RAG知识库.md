---
tags:
  - PGI
  - AI
  - RAG
  - 知识库
aliases:
  - 知识库
  - 向量检索
  - RAG
---

# RAG 知识库

本文档详细阐述 PGI 系统 RAG 知识库的完整处理流程、文档状态机、异步任务队列、与 AI 的协作机制、表结构和技术方案。适合开发人员和产品经理理解"文档怎么变成 AI 能检索的知识"。

## 一、RAG 是什么

**RAG（Retrieval-Augmented Generation，检索增强生成）**= 检索 + 生成

```
用户提问 ──► 向量化 ──► 向量库检索相似片段 ──► 注入上下文 ──► 大模型生成回答
```

### 1.1 与传统聊天机器人的区别

| 维度 | 传统聊天 | RAG |
|------|---------|-----|
| 知识来源 | 模型训练数据 | 企业自有文档 |
| 回答准确性 | 泛泛而谈 | 有据可查 |
| 可更新性 | 需重新训练 | 上传文档即可 |
| 幻觉风险 | 高 | 低（有上下文约束） |
| 适用场景 | 通用问答 | 企业专属问答 |

### 1.2 为什么招标业务需要 RAG

- 招标法规、企业制度等文档量大，人工检索效率低
- AI 基于企业文档回答，避免幻觉
- 历史标书可复用，降低新人学习成本
- 文档更新后无需重新训练，上传即生效

## 二、完整处理流程

```
文档上传 → Tika 解析(PDF/Word/MD/TXT)
    → 分块(500字, 重叠50)
    → Embedding 批量向量化(批20)
    → ChromaDB 存储
    → 检索 Top-K(5)
    → QuestionAnswerAdvisor 注入上下文
    → DeepSeek 生成回答
```

### 2.1 文档解析（Apache Tika）

- 支持 PDF / Word / Markdown / TXT / Excel
- 提取纯文本，去除格式信息
- Apache Tika 2.9.1，Apache 顶级项目

```java
@Component
public class DocumentParser {

    public String parse(InputStream stream, String fileName) {
        Tika tika = new Tika();
        try {
            return tika.parseToString(stream);
        } catch (Exception e) {
            throw new AiEmbeddingException("文档解析失败: " + fileName, e);
        }
    }
}
```

### 2.2 文档分块

- 按 `chunk_size=500` 字符分块
- `overlap=50` 字符重叠（保证上下文连续性）
- 支持配置化调整

```java
@Component
public class DocumentChunker {

    @Value("${rag.chunk.size:500}")
    private int chunkSize;

    @Value("${rag.chunk.overlap:50}")
    private int overlap;

    public List<String> chunk(String text) {
        List<String> chunks = new ArrayList<>();
        int start = 0;
        while (start < text.length()) {
            int end = Math.min(start + chunkSize, text.length());
            chunks.add(text.substring(start, end));
            start = end - overlap;  // 重叠
        }
        return chunks;
    }
}
```

### 2.3 向量化（智谱 embedding-2）

> [!important] 双端点设计
> 对话用 DeepSeek，向量化用智谱 embedding-2（1024 维），二者端点不同。
> `EmbeddingConfig` 自定义 `OpenAiEmbeddingModel` Bean（指向 `https://open.bigmodel.cn/api/paas/v4/`），`@Primary` 覆盖自动配置。

```java
@Configuration
public class EmbeddingConfig {

    @Value("${zhipu.api-key}")
    private String zhipuApiKey;

    @Bean
    @Primary
    public OpenAiEmbeddingModel embeddingModel() {
        OpenAiApi api = OpenAiApi.builder()
            .baseUrl("https://open.bigmodel.cn/api/paas/v4/")
            .apiKey(zhipuApiKey)
            .build();
        return new OpenAiEmbeddingModel(api,
            MetadataMode.EMBED,
            OpenAiEmbeddingOptions.builder().model("embedding-2").build());
    }
}
```

- 批量向量化（`batch_size=20`）
- 失败重试不超过 3 次

```java
@Service
public class EmbeddingService {

    @Autowired
    private EmbeddingModel embeddingModel;

    @Value("${rag.embedding.batch-size:20}")
    private int batchSize;

    public List<String> embed(List<String> texts) {
        List<String> allEmbeddings = new ArrayList<>();
        for (int i = 0; i < texts.size(); i += batchSize) {
            List<String> batch = texts.subList(i, Math.min(i + batchSize, texts.size()));
            EmbeddingResponse response = embeddingModel.embedForResponse(batch);
            for (Embedding e : response.getResults()) {
                allEmbeddings.add(e.getOutput());
            }
        }
        return allEmbeddings;
    }
}
```

### 2.4 向量存储（ChromaDB）

```yaml
spring.ai.vectorstore.chroma:
  client: { host: http://localhost, port: 8000 }
  collection-name: pgi_knowledge
  initialize-schema: true   # 自动建集合
```

- Spring AI `ChromaVectorStore` 自动管理 Collection
- `initialize-schema: true` 启动时自动建集合

### 2.5 检索（Top-K=5）

- 用户问题向量化 → ChromaDB 相似度检索
- 返回最相关的 5 个文档片段
- 组装为上下文注入大模型

```java
@Component
public class RagRetriever {

    @Autowired
    private VectorStore vectorStore;

    @Value("${rag.retrieval.top-k:5}")
    private int topK;

    public List<Document> retrieve(String query) {
        SearchRequest request = SearchRequest.builder()
            .query(query)
            .topK(topK)
            .build();
        return vectorStore.similaritySearch(request);
    }

    public String retrieveAsContext(String query) {
        List<Document> docs = retrieve(query);
        return docs.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n\n---\n\n"));
    }
}
```

### 2.6 上下文注入

通过 Spring AI 的 `QuestionAnswerAdvisor` 自动将检索结果注入到提示词：

```java
@Bean
public QuestionAnswerAdvisor questionAnswerAdvisor(VectorStore vectorStore) {
    return QuestionAnswerAdvisor.builder()
        .vectorStore(vectorStore)
        .searchRequest(SearchRequest.builder().topK(5).build())
        .build();
}
```

## 三、文档状态机

```
UPLOADED ──系统自动──► PROCESSING ──成功──► READY
                         │
                         ├──失败──► FAILED ──重试(<3)──► PROCESSING
                         │
                         └──删除──► DELETED
```

| 操作 | 角色 | 前置条件 |
|------|------|---------|
| 启动向量化 | 系统自动 | 文件已上传、MD5 校验通过 |
| 重试向量化 | 管理员 | 失败次数 < 3 |
| 删除文档 | 管理员 | 同步删除 ChromaDB 对应向量 |

### 状态说明

| 状态 | 含义 | 可执行操作 |
|------|------|-----------|
| UPLOADED | 已上传，待处理 | 系统自动启动向量化 |
| PROCESSING | 正在解析分块向量化 | 等待完成 |
| READY | 处理完成，可被 AI 检索 | 删除 |
| FAILED | 处理失败 | 重试（<3 次）、删除 |
| DELETED | 已删除 | — |

## 四、异步任务队列

> 向量化是耗时操作，采用任务队列异步处理，不阻塞用户。

### 4.1 任务表

`pgi_ai_embedding_job` 表记录向量化任务：

| 字段 | 说明 |
|------|------|
| id | 任务 ID |
| doc_id | 文档 ID |
| status | 任务状态（PENDING/PROCESSING/SUCCESS/FAILED） |
| retry_count | 重试次数（<3） |
| error_msg | 错误信息 |
| create_time | 创建时间 |
| update_time | 更新时间 |

### 4.2 EmbeddingJobScheduler

- 每 1 分钟轮询 `pgi_ai_embedding_job` 表
- 取 PENDING 状态任务执行
- 失败重试不超过 3 次
- 集群环境加分布式锁防重复执行

```java
@Component
public class EmbeddingJobScheduler {

    @Autowired
    private IEmbeddingJobService embeddingJobService;

    @Autowired
    private EmbeddingService embeddingService;

    @Scheduled(fixedRate = 60000)  // 每 1 分钟
    public void processJobs() {
        List<EmbeddingJob> jobs = embeddingJobService.selectPendingJobs();
        for (EmbeddingJob job : jobs) {
            try {
                job.setStatus("PROCESSING");
                embeddingJobService.update(job);

                // 执行向量化
                embeddingService.embedDocument(job.getDocId());

                job.setStatus("SUCCESS");
                embeddingJobService.update(job);
            } catch (Exception e) {
                job.setRetryCount(job.getRetryCount() + 1);
                if (job.getRetryCount() >= 3) {
                    job.setStatus("FAILED");
                } else {
                    job.setStatus("PENDING");
                }
                job.setErrorMsg(e.getMessage());
                embeddingJobService.update(job);
            }
        }
    }
}
```

## 五、知识库与 AI 的协作

```mermaid
sequenceDiagram
    participant User as 用户
    participant AI as ruoyi-ai
    participant RAG as ruoyi-rag
    participant Chroma as ChromaDB

    User->>AI: 上传文档
    AI->>RAG: 委托处理
    RAG->>RAG: Tika 解析+分块
    RAG->>RAG: 发布 EmbeddingJobEvent
    AI->>AI: 监听事件
    AI->>Chroma: 批量向量化存储
    RAG<<--AI: 回调更新状态 READY

    User->>AI: 提问
    AI->>RAG: 调用 RagRetriever
    RAG->>Chroma: 向量检索 Top-K=5
    RAG->>AI: 返回相关片段
    AI->>AI: 注入上下文生成回答
    AI->>User: 流式返回
```

### 5.1 事件驱动解耦

知识库（M14）↔ AI（M13）双向依赖，通过事件解耦：

```java
// ruoyi-rag 发布事件
@Service
public class KnowledgeDocServiceImpl implements IKnowledgeDocService {

    @Autowired
    private ApplicationEventPublisher eventPublisher;

    public void uploadDoc(Long docId) {
        // 保存文档元数据
        // ...
        // 发布向量化任务事件
        eventPublisher.publishEvent(new EmbeddingJobEvent(docId));
    }
}

// ruoyi-ai 监听事件
@Component
public class EmbeddingJobEventListener {

    @EventListener
    @Async
    public void handleEmbeddingJob(EmbeddingJobEvent event) {
        embeddingService.embedDocument(event.getDocId());
    }
}
```

## 六、知识库表结构

| 表 | 用途 | 关键字段 |
|----|------|---------|
| `pgi_ai_knowledge_base` | 知识库定义 | chroma_collection 对应 ChromaDB Collection |
| `pgi_ai_knowledge_doc` | 文档元数据 | status 状态机、file_md5 去重 |
| `pgi_ai_knowledge_chunk` | 分块 | chroma_id 映射 ChromaDB 向量 ID |
| `pgi_ai_embedding_job` | 向量化任务 | status、retry_count |

### 6.1 knowledge_base 表

| 字段 | 说明 |
|------|------|
| id | 知识库 ID |
| kb_name | 知识库名称 |
| kb_code | 知识库编号（唯一） |
| description | 描述 |
| chroma_collection | ChromaDB Collection 名 |
| status | 状态（ACTIVE/FROZEN/DELETED） |
| create_by | 创建人 |
| create_time | 创建时间 |

### 6.2 knowledge_doc 表

| 字段 | 说明 |
|------|------|
| id | 文档 ID |
| kb_id | 知识库 ID |
| doc_name | 文档名称 |
| file_path | 文件路径 |
| file_md5 | 文件 MD5（去重） |
| file_size | 文件大小 |
| file_type | 文件类型（PDF/Word/MD/TXT） |
| status | 文档状态（UPLOADED/PROCESSING/READY/FAILED/DELETED） |
| chunk_count | 分块数量 |

### 6.3 knowledge_chunk 表

| 字段 | 说明 |
|------|------|
| id | 分块 ID |
| doc_id | 文档 ID |
| chunk_index | 分块序号 |
| chunk_content | 分块内容 |
| chroma_id | ChromaDB 向量 ID |
| token_count | token 数 |

详见 [[数据库设计]]。

## 七、配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `rag.chunk.size` | 500 | 分块字符数 |
| `rag.chunk.overlap` | 50 | 分块重叠字符数 |
| `rag.embedding.batch-size` | 20 | 向量化批量大小 |
| `rag.retrieval.top-k` | 5 | 检索返回数量 |
| `rag.embedding.retry-limit` | 3 | 向量化重试上限 |

## 八、优化建议

### 8.1 检索结果不准确

| 优化方向 | 方法 |
|---------|------|
| 分块参数 | 调整 `chunk_size`（默认 500）和 `overlap`（默认 50） |
| Top-K | 调整检索数量（默认 5） |
| 文档质量 | 确保上传的文档结构清晰、内容完整 |
| 向量化模型 | 当前用智谱 embedding-2（1024 维），中文效果好 |

### 8.2 向量化速度慢

| 优化方向 | 方法 |
|---------|------|
| 批量大小 | 增大 `batch_size`（默认 20） |
| 异步处理 | 已采用任务队列，不阻塞用户 |
| 并发处理 | 多线程处理多个文档（需加分布式锁） |

## 相关笔记
- [[AI智能体]] — RAG 是 AI 智能体的核心能力之一
- [[工具调用体系]] — SearchRagDocTool 工具
- [[技术栈总览]] — Tika / ChromaDB / embedding-2 选型
- [[05-亮点展示]] — RAG 的商业价值
- [[数据库设计]] — 知识库表结构
- [[架构设计]] — 事件驱动解耦
