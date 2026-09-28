# Gindama_Notes

个人 Obsidian 笔记库，包含两个独立的知识库。

---

## 📖 PGI_项目手册

**PGI 招标智能 AI 管理系统 · 项目手册**（18 篇）

一个 AI Native 的招标全流程管理平台的项目文档，把传统招标业务（项目 → 公告 → 投标 → 评标 → 中标）与 AI 智能体（对话、RAG 知识库、工具调用）融合。

| 主题 | 内容 |
| --- | --- |
| 主线 | 项目创建过程、技术结构、角色分配、使用教程、亮点展示 |
| 技术 | 技术栈总览、架构设计、核心模块详解 |
| 业务 | 招标全流程、状态机设计、业务规则体系 |
| AI 能力 | AI 智能体、RAG 知识库、工具调用体系 |
| 数据 / 运维 | 数据库设计、部署方案、常见问题排查 |

笔记之间通过 `[[]]` 双链互相关联，入口见 [`PGI_项目手册/README.md`](PGI_项目手册/README.md)。

---

## 🐍 Python_Ai

**Python + AI 应用开发学习笔记**（24 篇）

| 目录 | 说明 |
| --- | --- |
| `Ai_Study/PyAiCode/` | 学习主线 6 篇：环境配置 → 多轮对话 → Prompt → RAG → Agent 概念 → LangChain Agent 实战 |
| `Ai_Study/Little_Knowledge/` | 小知识点：venv、temperature、Embedding、Chroma、BM25、ReAct、LangChain、Hallucination 等 |
| `PIC/` | 笔记引用的架构图（Agent 框架、RAG 架构、ReAct、LangChain） |
| `knowledge/` | 笔记写作规范 |

技术栈：Python + OpenAI 兼容 SDK（MiniMax）、Chroma 向量库、LangChain。

---

## 🔧 本地使用

```bash
git clone https://github.com/Gindama713/Gindama_Notes.git
```

用 Obsidian 打开 `PGI_项目手册/` 或 `Python_Ai/` 目录即可（两个库各自带独立的 `.obsidian` 配置）。

> 已忽略 `workspace.json`（窗口布局，每次打开都会变）与插件 `data.json`（可能存放 API Key）。
