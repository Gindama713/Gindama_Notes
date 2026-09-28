<small>承接 [[5_Ai_Agent_部署训练项目使用]]，上一篇只讲了 Agent 的概念，这一篇动手用 [[LangChain]] 写一个真正能跑的 Agent，把前面学的 [[ToolUse_FunctionCalling|工具调用]]、[[Memory|记忆]]、[[ReAct]] 循环全部串起来</small>


## 为什么要用 LangChain

第 2 篇我们用纯 OpenAI SDK 写过多轮对话，那个能跑，但有个问题：**AI 只会说话，不会干活**。  
你问它"25 乘 16 等于多少"，它要么自己算（可能算错），要么老实承认"我算不了"。  
想要让它真正去查天气、算数学、读文件，你得自己写一大堆"判断它要不要调工具、调哪个工具、拿到结果怎么处理"的逻辑——这就是 [[ReAct]] 循环。

自己写这个循环很烦，LangChain 帮你把这套轮子造好了：

|对比项|纯 OpenAI SDK|LangChain|
|---|---|---|
|调工具|自己解析返回、自己拼请求|`@tool` 一行定义，框架自动调度|
|ReAct 循环|自己写 while 循环+终止条件|`AgentExecutor` 现成的|
|记忆|自己维护 messages 列表|框架帮你塞进 prompt|
|接 RAG/数据库|全靠手写|有现成的工具集成|
|学习成本|低|中（抽象层多）|

一句话：**写玩具用纯 SDK，写能干活的 Agent 用 LangChain**。  
代价是抽象层多、出 bug 不好调，但比起自己从零造轮子，还是省事。

## 安装

```
pip install langchain langchain-openai langchain-core
```

> 三个包：`langchain` 是主体，`langchain-openai` 是 OpenAI 兼容模型接入（MiniMax 也走这个），`langchain-core` 是底层核心。

## LangChain 的几个核心零件

先把名字认全，后面代码里一个一个对上：

|零件|作用|类比|
|---|---|---|
|LLM / ChatModel|大脑，负责推理|GPT、MiniMax-M3|
|Tool|手，干具体的活|计算器、搜索、读文件|
|Prompt|指令模板，告诉大脑怎么干|系统提示词 + 工具说明|
|Agent|大脑+规则的结合体，决定调哪个工具|决策中心|
|AgentExecutor|执行器，负责跑 [[ReAct]] 循环|while 循环的封装|
|Memory / chat_history|记忆，记住之前说过啥|messages 列表|

## 第一步：初始化大脑（LLM）

MiniMax 兼容 OpenAI 接口，所以直接用 `ChatOpenAI` 指向 MiniMax 的地址就行，和第 1 篇的配置逻辑一模一样：

```python
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv
import os

load_dotenv()

llm = ChatOpenAI(
    model="MiniMax-M3",
    api_key=os.getenv("MINIMAX_API_KEY"),
    base_url="https://api.minimaxi.com/v1",
    temperature=0
)
```

> `temperature=0` 是让 Agent 稳一点。Agent 调工具时我们不希望它"发挥创意"，要它老老实实按规则来。参见 [[Temperature]]。

## 第二步：定义工具（Agent 的手）

LangChain 定义工具最简单的方式是 `@tool` 装饰器，写个普通函数就行：

```python
from langchain_core.tools import tool

@tool
def calculate(expression: str) -> str:
    """做数学计算。输入数学表达式，比如 '25 * 16' 或 '100 / 4'"""
    try:
        result = eval(expression)
        return f"计算结果：{result}"
    except Exception as e:
        return f"计算失败：{e}"

@tool
def get_word_length(word: str) -> str:
    """返回一个单词或句子的字符长度"""
    return f"'{word}' 的长度是 {len(word)} 个字符"

tools = [calculate, get_word_length]
```

**重点**：那个三引号 `"""..."""` 不是摆设，它是工具的说明文档，**Agent 就是靠这段话判断"该不该调这个工具"的**。  
写得不清晰，Agent 就会乱调——这正是第 5 篇说的"工具选择错误"失败模式。

## 第三步：组装 Agent

```python
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有用的助手。遇到计算或需要工具的任务，请调用对应工具，不要自己瞎猜。"),
    MessagesPlaceholder(variable_name="chat_history", optional=True),
    ("human", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad")
])

agent = create_tool_calling_agent(llm, tools, prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)
```

几个关键点：
- `chat_history`：留位置给记忆，先空着，第四步用
- `agent_scratchpad`：**这个必须有**，Agent 在思考过程中调工具的中间过程会塞在这里
- `verbose=True`：打印 [[ReAct]] 的思考过程，学习阶段一定要开，能看清它"想→做→看"的每一步

## 第四步：跑起来

```python
result = agent_executor.invoke({"input": "25 乘以 16 等于多少？再告诉我 'LangChain' 有几个字母"})
print("最终回答：", result["output"])
```

开了 `verbose=True` 后，你会看到类似这样的输出（这就是 [[ReAct]] 循环的真实样子）：

```
> Entering new AgentExecutor chain...

Invoking: calculate with {'expression': '25 * 16'}
计算结果：400

Invoking: get_word_length with {'word': 'LangChain'}
'LangChain' 的长度是 9 个字符

> Finished chain.
最终回答：25 乘以 16 等于 400，'LangChain' 有 9 个字母。
```

看明白了吗？Agent 自己拆了任务、分别调了两个工具、拿到结果后拼成最终答案。**这就是第 5 篇讲的"观察-思考-行动"循环，现在它真的跑起来了**。

## 第五步：给 Agent 加记忆

上面那个 Agent 是失忆的——你下一句话它不记得上一句。解决办法是把对话历史塞进 `chat_history`：

```python
chat_history = []

def chat(user_input):
    # 把用户问题塞进历史
    chat_history.append({"role": "user", "content": user_input})
    # 调 Agent，注意把 chat_history 传进去
    result = agent_executor.invoke({
        "input": user_input,
        "chat_history": chat_history
    })
    # 把 AI 回答也塞进历史，下一轮才能记住
    chat_history.append({"role": "assistant", "content": result["output"]})
    return result["output"]

print(chat("我叫小蒋，正在学 LangChain"))
print(chat("我刚才说我叫什么？"))   # 它能答上"小蒋"，说明记住了
```

这其实就是第 2 篇多轮对话的思路，只不过换成了 Agent 版本。参见 [[Memory]]。

## 第六步：把 RAG 当工具接进来（串联全部知识）

这是最爽的一步——把第 4 篇的 [[4_RAG 检索增强生成|RAG]] 包成一个工具，让 Agent 自己决定什么时候去查知识库：

```python
import chromadb
from langchain_core.tools import tool

# 复用第4篇的 Chroma 知识库
chroma_client = chromadb.PersistentClient(path="./chroma_db")
collection = chroma_client.get_or_create_collection("my_docs")

@tool
def search_knowledge_base(question: str) -> str:
    """在本地知识库里查找相关信息。当用户问到关于Gindama、蓝桥杯、RAG、Chroma 等已知知识库内容时调用"""
    results = collection.query(query_texts=[question], n_results=3)
    docs = results["documents"][0]
    if not docs:
        return "知识库里没找到相关内容"
    return "\n".join(docs)

# 把 RAG 工具加进工具列表
tools = [calculate, get_word_length, search_knowledge_base]

# 重新组装 Agent（工具变了要重建）
agent = create_tool_calling_agent(llm, tools, prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# 现在它可以同时算数学、查知识库
result = agent_executor.invoke({"input": "Gindama在蓝桥杯拿了什么奖？顺便算一下 2024 + 1"})
print(result["output"])
```

跑一下你就会看到：Agent 先调 `search_knowledge_base` 查蓝桥杯，再调 `calculate` 算加法，最后把两个结果拼起来回答你。  
**到这里，第 1~5 篇学的所有东西——环境、[[3_理解AI后端+Prompt提示词]]、[[Embedding]]、[[Chroma]]、RAG、工具调用、记忆、ReAct——全部串进了一个能跑的程序里。**

## 工具的两种写法

上面用的是 `@tool` 装饰器，还有一种 `Tool` 类的写法：

|写法|例子|什么时候用|
|---|---|---|
|`@tool` 装饰器|`@tool\ndef calc(x):...`|绝大多数情况，简洁|
|`Tool` 类|`Tool(name="calc", func=calc, description="...")`|需要更细的控制（比如自定义参数校验）|

新手用 `@tool` 就够了，别给自己找麻烦。

## 踩坑提醒

|坑|现象|怎么解决|
|---|---|---|
|没写工具 docstring|Agent 乱调工具或不调|三引号说明必须写清楚，最好带示例|
|忘了 `agent_scratchpad`|报错或 Agent 卡住|prompt 里一定要有这个 placeholder|
|工具 description 太模糊|该调计算器却调了搜索|把"什么时候用"写明白|
|死循环|反复调同一个工具停不下来|`AgentExecutor` 里设 `max_iterations=5`|
|MiniMax 不返回工具调用|Agent 当没看见工具|确认模型支持 function calling，M3 支持|
|中文工具名乱码|工具名最好用英文，description 可以中文|name 用英文，docstring 用中文|

> 一个实用建议：先把上面第四步的 `verbose=True` 输出反复看几遍，看懂 Agent 的思考过程，比写十个 demo 都管用。[[ReAct]] 不是背出来的，是看它跑出来的。
