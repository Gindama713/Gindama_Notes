**MCP = Model Context Protocol，模型上下文协议**

**MCP 是 AI Agent 连接外部工具和数据源的“统一接口标准”。**
你可以把它理解成 **AI 世界的 USB-C 接口**。

- 工具方实现一个 **MCP Server**
- Agent 方实现一个 **MCP Client**
- 任何 Agent 都能连任何工具 → M + N


Agent（LLM + 逻辑）
   ↓
MCP Client
   ↓ 标准协议
MCP Server
   ├── 文件系统
   ├── 数据库
   ├── GitHub
   ├── 浏览器
   └── 任何外部服务

| **MCP Client** | Agent 内部，负责和 Server 通信 |
| -------------- | ---------------------- |
| **MCP Server** | 暴露工具、资源、提示词给 Client    |
| **协议**         | 定义双方怎么说话（JSON-RPC）     |

**MCP 是 [[ToolUse_FunctionCalling]]的标准化升级版。**

|维度|Tool Calling|MCP|
|---|---|---|
|定义方式|每个框架自己定|统一协议|
|工具复用|难，换个框架重写|容易，任何 MCP Client 都能用|
|连接方式|硬编码|标准协议|
|生态|分散|统一|

**Tool Calling 是“怎么调”，MCP 是“用什么标准调”。**

有了 MCP：
**你的 Agent 要读本地文件。**
1. 启动一个 filesystem MCP Server
2. Agent 的 MCP Client 连接它
3. Agent 自动获得 read_file、write_file 等工具
4. 不用在 Agent 代码里写任何文件操作
**工具和 Agent 完全解耦。**

LLM → Tool Calling → MCP → Agent → LangGraph