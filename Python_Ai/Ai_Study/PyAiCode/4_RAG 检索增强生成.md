`RAG（Retrieval-Augmented Generation，检索增强生成）是一种让大语言模型基于特定知识库回答问题的技术`

## RAG 架构全貌
离线索引阶段 + 在线检索阶段
![[RAG架构图.png]]

```
[用户问题]
    ↓
[embedding 模型]   ← 把问题转成一串数字（向量）
    ↓
[向量数据库]       ← 用相似度找出 top-k 相关文档片段
    ↓
[拼 prompt]        ← 系统提示词 + 检索片段 + 用户问题
    ↓
[LLM]              ← 看资料 + 看问题，生成答案
```


#### 离线索引阶段（数据准备）

这个阶段是在后台运行的，用户不会直接看到。

它的任务是把你的文档处理成可以快速检索的格式。

步骤如下：

- 1. **文档加载**：读取 PDF、TXT、DOCX、网页等各种格式的文档。
    
- 2. **文档切分**：把长文档分割成小的文本块（Chunk），通常是几百到一千个词。
    
- 3. **向量化**：用 Embedding 模型把每个文本块转换成向量（一串数字）。
    
- 4. **存储**：把向量和原始文本一起存入向量数据库。
    

这个阶段只需要做一次，或者当文档更新时重新运行。

#### 在线检索阶段（用户查询）

这个阶段是用户提问时实时发生的。

步骤如下：[[Embedding]]

- 1. **问题向量化**：用同一个 Embedding 模型把用户的问题也转换成向量。
    
- 2. **相似度检索**：在向量数据库中找出和问题向量最接近的几个文本块。
    
- 3. **构建 Prompt**：把检索到的文本块作为上下文，和用户的问题一起组装成 Prompt。
    
- 4. **LLM 生成**：把 Prompt 送给 LLM，让它基于上下文回答。
    
- 5. **返回答案**：把 LLM 的回答返回给用户，通常还会附上引用的文档来源。



| 场景                  | 推荐方案                                |
| ------------------- | ----------------------------------- |
| 知识 < 几千字            | [[Memory]] 够用                       |
| 知识 > 几万字            | 上 RAG                               |
| 搭 RAG 系统            | [[Chroma]] + embedding 模型 + LLM 三件套 |
| 个人 AI 助手（既要懂你又要查资料） | **Memory + RAG 叠加**                 |

```python
import chromadb  
from openai import OpenAI  
from dotenv import load_dotenv  
import os  
  
load_dotenv()  
  
chroma_client = chromadb.PersistentClient(path="./chroma_db")  
collection = chroma_client.get_or_create_collection("my_docs")  

collection.add(  
    documents=[  
        "Jiang Suhao won first prize in Lanqiao Cup Python B group",  
        "He is studying AI backend and RAG",  
        "He wants to study abroad in Hong Kong",  
        "Chroma is a vector database for RAG",  
        "RAG means retrieval-augmented generation"  
    ],  
    ids=["1", "2", "3", "4", "5"]  
)  
  
    api_key=os.getenv("MINIMAX_API_KEY"),  
    base_url="https://api.minimaxi.com/v1"  
)  
  
def rag_ask(question):  
    # 1.语义检索：找出知识库中和问题最相关的3段内容  
    search_result = collection.query(  
        query_texts=[question],  
        n_results=3  
    )  
    # 把检索到的多条文本拼接成参考上下文  
    context_text = "\n".join(search_result["documents"][0])  
  
    # 构造提示词，强制AI只能根据给的资料回答  
    prompt = f"""Answer strictly based on the provided context, do not make up content.  
Context reference:  
{context_text}  
User question: {question}  
Answer:"""  
  
    # 调用MiniMax模型生成答案  
    resp = llm_client.chat.completions.create(  
        model="MiniMax-M3",  
        messages=[{"role": "user", "content": prompt}]  
    )  
    return resp.choices[0].message.content  
  
    ans1 = rag_ask("What did Jiang Suhao win?")  
    print("问题1答案：", ans1)  
  
    ans2 = rag_ask("What is RAG?")  
    print("问题2答案：", ans2)
```

```
问题1答案： <think>The user is asking what Jiang Suhao won. Based on the context, it states that "Jiang Suhao won first prize in Lanqiao Cup Python B group." I should answer strictly based on the provided context without making up content.</think>

According to the context, Jiang Suhao won first prize in the Lanqiao Cup Python B group.
问题2答案： <think>The user is asking about what RAG is, based on the provided context. Let me look at the context clues:

1. "Chroma is a vector database for RAG" - This tells us RAG uses a vector database like Chroma
2. "RAG means retrieval-augmented generation" - This directly answers what RAG is
3. "He is studying AI backend and RAG" - This tells us RAG is related to AI backend

Based on the context, RAG stands for "retrieval-augmented generation." I should answer strictly based on what's in the context, without making up additional content.</think>

Based on the provided context, **RAG** stands for **Retrieval-Augmented Generation**.

Additionally, from the context, it is mentioned that:
- RAG involves the use of a **vector database**, with **Chroma** being an example of one.
- RAG is a topic of study in **AI backend**.
```

## 所以完整的[[Recall and precision]]提升手段，按性价比排序

| 优先级 | 手段                                  | 成本  | 效果  |
| --- | ----------------------------------- | --- | --- |
| 1   | 优化切块（语义完整）                          | 低   | 高   |
| 2   | 加文档标题/章节注释                          | 极低  | 中高  |
| 3   | 混合检索（向量+[[BM25]]）                   | 低   | 高   |
| 4   | 查询改写                                | 低   | 中高  |
| 5   | 加 Reranker [[Contextual Retrieval]] | 中   | 高   |
| 6   | LLM 生成上下文（Contextual Retrieval）     | 高   | 很高  |
| 7   | 换更好的 [[Embedding]] 模型               | 中   | 中   |
