# 项目长期记忆 — Python_Ai Obsidian 笔记库

## 项目性质
用户的 Python+AI 学习笔记库（Obsidian vault），用户正在系统学习 AI 应用开发。

## 学习路径（PyAiCode 主线）
1. 初始环境配置（venv + OpenAI SDK + MiniMax API）✅
2. 多轮对话管理（messages 列表维护）✅
3. 理解AI后端 + Prompt 提示词（AI/ML/DL、提示词结构、幻觉、安全）✅
4. RAG 检索增强生成（Chroma + Embedding 实战代码）✅
5. Ai_Agent 概念（ReAct、LangChain 介绍、工具调用、记忆、失败模式）✅
6. LangChain Agent 实战（2026-07-17 新增，动手写 Agent）✅

## 配套小知识点（Little_Knowledge）
venv、temperature、Memory、Hallucination、Chroma、JSON、ToolUse_FunctionCalling、ReAct、Embedding、LangChain

## 用户写作习惯（撰写笔记时必须遵守）
- `<small>` 标签做开头说明，`# GOAL` + `[√]` 打勾清单
- 大量表格对比（3-4列），代码块用 ```python
- 代码：MiniMax API（`api.minimaxi.com/v1`，model=`MiniMax-M3`）+ `load_dotenv()` + `os.getenv`
- 中文口语化注释，markdown 正文行尾两个空格换行
- 双链 `[[小知识点]]` 指向 Little_Knowledge
- 图片 `![[xxx.png]]`，文本流程图用 ``` 代码块
- 结尾必有"实用建议"或总结性 `>` 引用
- 几乎不用 emoji，举例贴近生活（蓝桥杯、江苏昊、订机票、SpaceX）

## 技术栈
- Python + OpenAI 兼容 SDK（MiniMax）
- Chroma 向量数据库（RAG）
- LangChain（Agent 框架）
- 密钥管理：dotenv + .env 文件
