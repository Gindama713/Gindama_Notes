 [[4_RAG 检索增强生成]]] 里专门用来**给初步召回的结果重新打分、重新排序**的模型。
 
用户问题
   ↓
向量检索 + [[BM25]] 混合召回
   ↓
召回 Top 50
   ↓
Reranker 精排
   ↓
Top 5 给 LLM

召回阶段要尽量保证：

> **所有可能相关的，都进 Top 50。**

Reranker 再在这 50 个里：

> **把最相关的排最前，把噪音踢出去。**

**BGE Reranker**
它采用 **Cross-Encoder（交叉编码器）** 结构，会把“用户问题”和“候选文档”**拼接在一起**输入模型，让它们进行深度的、逐词级别的语义交互，然后直接输出一个 0~1 之间的相关性分数[](https://aihub.caict.ac.cn/models/BAAI/bge-reranker-v2-m3#1)[](https://developer.aliyun.com/article/1764001#2)。这种深度匹配比向量相似度精确得多，能有效过滤噪声。