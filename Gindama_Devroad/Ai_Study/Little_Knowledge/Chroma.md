完整 [[4_RAG 检索增强生成]] 系统的分工：

LLM = 大脑（回答问题、思考）
chroma = 笔记本（存数据 + 查数据，没脑子）
[[Embedding]] 模型 = 翻译官（把"姜苏豪讨厌客气"转成数字，数字相近 ≈ 意思相近）

**Chroma 本身能做啥**（Python 代码）：

```python
import chromadb
# 1. 创建客户端（数据持久化到 ./chroma_db 文件夹）
client = chromadb.PersistentClient(path="./chroma_db")
# 2. 建一个"集合"（类似数据库的表，名字自定义）
collection = client.get_or_create_collection("my_docs")
# 3. 加文档（Chroma 会自动用默认 embedding 模型把文字转成向量）
collection.add(
    documents=["姜苏豪考了蓝桥杯国一", "他讨厌AI客气"],
    ids=["doc1", "doc2"]   # 每个文档一个唯一 ID
)
# 4. 查询（输入问题，Chroma 自动找最相似的 top-2 文档）
results = collection.query(
    query_texts=["姜苏豪有什么成绩"],
    n_results=2
)
# 返回: results 里会包含 "姜苏豪考了蓝桥杯国一" 这个文档
```


**Chroma 的关键限制**：
-  **默认 embedding 是英文**（all-MiniLM-L6-v2），中文检索效果差
- 想中文好得换 **bge-m3** / **bge-large-zh** 等中文 embedding
-  **Chroma 自己不会思考**，只能查不能答
