---
tags:
  - PGI
  - AI
  - 智能体
aliases:
  - AI Agent
  - 对话系统
  - AI智能体
---

# AI 智能体

本文档详细阐述 PGI 系统 AI 智能体的能力全景、技术架构、SSE 流式对话、记忆管理、用量审计、操作确认与回滚、会话生命周期。适合开发人员和产品经理理解"AI 怎么工作、怎么记忆、怎么保证安全"。

## 一、AI 智能体能力全景

| 能力 | 说明 | 技术实现 |
|------|------|---------|
| **流式对话** | SSE 流式输出，边生成边显示 | Spring AI `ChatClient.stream()` + `SseEmitter` |
| **工具调用** | AI 自动调用 12 个业务工具 | Spring AI `@Tool` 注解 + `ToolCallback` |
| **短期记忆** | 记住最近 20 轮对话 | Redis `MessageWindowChatMemory` |
| **长期记忆** | 记住用户偏好/关键事实 | MySQL `pgi_ai_memory`（importance 衰减） |
| **RAG 检索** | 基于企业文档回答问题 | [[RAG知识库]] |
| **用量审计** | 记录每次调用的 token 和成本 | `UsageAdvisor` 拦截 |
| **操作确认** | 写操作前弹窗确认 | `ToolConfirmAdvisor` |
| **快照回滚** | 记录前后快照，支持回滚 | `pgi_ai_action_record` |

## 二、技术架构

```
用户消息
   │
   ▼
┌──────────────────────────────────┐
│        AgentOrchestrator          │  ← Agent 编排器
│  (组装上下文 + 路由 Agent)         │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│          ChatClient               │  ← Spring AI 2.0
│  ┌────────────────────────────┐  │
│  │  Advisors (拦截器链)        │  │
│  │  1. UsageAdvisor (用量统计) │  │
│  │  2. ToolConfirmAdvisor      │  │
│  │  3. QuestionAnswerAdvisor   │  │  ← RAG 上下文注入
│  └────────────────────────────┘  │
│  ┌────────────────────────────┐  │
│  │  Memory (记忆)              │  │
│  │  - 短期: Redis (20 轮)      │  │
│  │  - 长期: MySQL              │  │
│  └────────────────────────────┘  │
│  ┌────────────────────────────┐  │
│  │  Tools (12 个工具)          │  │
│  │  - bid (招标类)             │  │
│  │  - system (系统类)          │  │
│  │  - rag (检索类)             │  │
│  └────────────────────────────┘  │
└──────────────────┬───────────────┘
                   │
                   ▼
            DeepSeek 大模型
```

### 2.1 AgentOrchestrator（Agent 编排器）

- 负责组装上下文（用户信息、会话历史、业务对象）
- 路由到对应领域的 Agent（BidAgent / NewsAgent）
- 管理对话生命周期

### 2.2 ChatClient（Spring AI 2.0）

```java
@Configuration
public class ChatClientConfig {

    @Bean
    public ChatClient chatClient(ChatClient.Builder builder,
                                  ToolRegistry toolRegistry,
                                  ChatMemoryService memoryService) {
        return builder
            .defaultSystem("你是 PGI 招标智能助手...")
            .defaultTools(toolRegistry.getCallbacks())
            .defaultAdvisors(
                new UsageAdvisor(),
                new ToolConfirmAdvisor(),
                QuestionAnswerAdvisor.builder()
                    .vectorStore(vectorStore)
                    .build()
            )
            .build();
    }
}
```

## 三、SSE 流式对话

### 3.1 核心交互方式

前端基于 `fetch + ReadableStream + AbortController` 实现 SSE，支持中途取消。

### 3.2 前端实现要点

- 按 `data:` 字段分行解析
- 区分 `tool_call`（工具调用）与普通文本消息
- 单用户单连接，心跳 15 秒，超时 3600 秒

### 3.3 前端 SSE 实现

```javascript
// utils/sse.js
export async function streamChat(conversationId, message, callbacks) {
    const controller = new AbortController();

    const response = await fetch(`/api/ai/chat/${conversationId}`, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${getToken()}`,
            'X-Request-Id': uuid()
        },
        body: JSON.stringify({ message }),
        signal: controller.signal
    });

    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    let buffer = '';

    while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        buffer += decoder.decode(value, { stream: true });
        const lines = buffer.split('\n');
        buffer = lines.pop();

        for (const line of lines) {
            if (line.startsWith('data:')) {
                const data = JSON.parse(line.slice(5));
                if (data.type === 'tool_call') {
                    callbacks.onToolCall(data);
                } else {
                    callbacks.onMessage(data.content);
                }
            }
        }
    }

    return controller;
}
```

### 3.4 后端 SSE 实现

```java
@PostMapping("/api/ai/chat/{conversationId}")
public SseEmitter chat(@PathVariable Long conversationId,
                       @RequestBody ChatMessageDTO dto) {
    SseEmitter emitter = sseEmitterManager.create(conversationId);

    // 获取历史记忆
    List<Message> history = memoryService.getHistory(conversationId);

    chatClient.prompt()
        .user(dto.getMessage())
        .messages(history)
        .tools(toolRegistry.getCallbacks())
        .stream()
        .content()
        .subscribe(
            content -> emitter.send(SseEmitter.event().data(content)),
            error -> {
                emitter.completeWithError(error);
                sseEmitterManager.remove(conversationId);
            },
            () -> {
                emitter.complete();
                sseEmitterManager.remove(conversationId);
            }
        );

    return emitter;
}
```

### 3.5 SseEmitterManager

- 单用户单连接：同一用户同时只有一个 SSE 连接
- 心跳保活：15 秒发送一次心跳
- 超时自动关闭：3600 秒无活动自动关闭
- 连接关闭时主动释放资源

## 四、记忆管理

### 4.1 短期记忆（Redis）

| 维度 | 说明 |
|------|------|
| Key | `ai:memory:short:{conversationId}` |
| 存储 | 最近 20 轮对话（`MessageWindowChatMemory`） |
| TTL | 会话关闭后保留 24 小时 |
| 实现 | Spring AI `ChatMemory` 接口 |

```java
@Service
public class ChatMemoryService {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    public void addMessage(Long conversationId, Message message) {
        String key = "ai:memory:short:" + conversationId;
        List<Message> history = getHistory(conversationId);
        history.add(message);

        // 保留最近 20 轮
        if (history.size() > 40) {  // 20 轮 = 40 条消息（user+assistant）
            history = history.subList(history.size() - 40, history.size());
        }

        redisTemplate.opsForValue().set(key,
            JSON.toJSONString(history), 24, TimeUnit.HOURS);
    }

    public List<Message> getHistory(Long conversationId) {
        String key = "ai:memory:short:" + conversationId;
        String json = redisTemplate.opsForValue().get(key);
        return json != null ? JSON.parseArray(json, Message.class) : new ArrayList<>();
    }
}
```

### 4.2 长期记忆（MySQL）

| 维度 | 说明 |
|------|------|
| 表 | `pgi_ai_memory` |
| 写入时机 | 会话结束时提取关键信息 |
| 内容 | 用户偏好、关键事实、历史摘要 |
| 衰减 | `importance` 字段 30 天周期衰减 |
| 召回 | `access_count` 访问计数，高频记忆优先 |

```java
@Service
public class LongTermMemoryService {

    public void extractAndSave(Long conversationId, Long userId) {
        // 获取会话历史
        List<Message> history = chatMemoryService.getHistory(conversationId);

        // 调用大模型提取关键信息
        String summary = chatClient.prompt()
            .user("请从以下对话中提取用户偏好和关键事实:\n" + historyToString(history))
            .call()
            .content();

        // 保存到长期记忆
        AIMemory memory = new AIMemory();
        memory.setUserId(userId);
        memory.setContent(summary);
        memory.setImportance(1.0);  // 初始重要性
        memory.setAccessCount(0);
        memoryMapper.insert(memory);
    }
}
```

### 4.3 记忆衰减定时任务

- `MemoryDecayScheduler` 每天凌晨执行
- `importance < 阈值` 的长期记忆自动清理
- 衰减公式：`importance = importance * 0.95`（每天衰减 5%）

## 五、用量审计

### 5.1 成本可控

每次 AI 调用都记录 token 消耗和成本，企业可精准核算 AI 使用成本。

### 5.2 用量日志表

| 字段 | 说明 |
|------|------|
| token_input | 输入 token 数 |
| token_output | 输出 token 数 |
| cost_amount | 成本金额（按 DeepSeek 定价计算） |
| model_name | 模型名称 |
| conversation_id | 关联会话 |
| user_id | 调用用户 |
| call_time | 调用时间 |
| duration_ms | 耗时 |

### 5.3 UsageAdvisor 实现

```java
public class UsageAdvisor implements BaseAdvisor {

    @Override
    public ChatResponse adviseCall(ChatClient.ChatClientRequest request,
                                    CallChatOptions options) {
        long start = System.currentTimeMillis();
        ChatResponse response = nextAdvisor.adviseCall(request, options);
        long duration = System.currentTimeMillis() - start;

        // 记录用量
        Usage usage = response.getMetadata().getUsage();
        UsageLog log = new UsageLog();
        log.setTokenInput(usage.getPromptTokens());
        log.setTokenOutput(usage.getCompletionTokens());
        log.setCostAmount(calculateCost(usage));
        log.setDurationMs(duration);
        log.setConversationId(request.context().get("conversationId"));
        log.setUserId(SecurityUtils.getUserId());
        usageLogMapper.insert(log);

        return response;
    }

    private BigDecimal calculateCost(Usage usage) {
        // DeepSeek 定价: input $0.001/1K tokens, output $0.002/1K tokens
        BigDecimal inputCost = BigDecimal.valueOf(usage.getPromptTokens())
            .multiply(new BigDecimal("0.001")).divide(new BigDecimal("1000"));
        BigDecimal outputCost = BigDecimal.valueOf(usage.getCompletionTokens())
            .multiply(new BigDecimal("0.002")).divide(new BigDecimal("1000"));
        return inputCost.add(outputCost);
    }
}
```

## 六、操作确认与回滚

### 6.1 安全机制

写操作类工具（`risk_level='1'或'2'`）执行前必须用户确认。

### 6.2 流程

1. 工具执行时检查 `need_confirm`，若需确认则创建 `pgi_ai_action_record`（`confirm_status='0'`），暂停执行
2. 前端弹窗确认后调用 `/api/ai/action/confirm/{id}`
3. 确认后继续执行，记录 `before_snapshot` / `after_snapshot`
4. 支持回滚：`/api/ai/action/rollback/{id}`

### 6.3 风险等级定义

| risk_level | 含义 | need_confirm | 示例 |
|------------|------|-------------|------|
| 0 | 只读 | 否 | SearchProject |
| 1 | 低风险写 | 是 | AddProgress |
| 2 | 高风险写 | 是 | SubmitQualification |

### 6.4 示例交互

```
用户：帮我把 PGI-2026-001 项目进度更新为"评标中"
AI：[调用 AddProgressTool] 检测到写操作
    → 创建 action_record，弹窗："确认更新项目进度？"
用户：确认
AI：[执行工具] 记录 before/after 快照
    → "已更新项目进度为'评标中'"
用户：撤销这个操作
AI：[调用 rollback] 根据 before_snapshot 恢复
    → "已回滚，项目进度恢复为原状态"
```

### 6.5 action_record 表

| 字段 | 说明 |
|------|------|
| id | 记录 ID |
| conversation_id | 关联会话 |
| user_id | 操作用户 |
| tool_name | 工具名称 |
| tool_args | 工具入参 JSON |
| before_snapshot | 操作前数据快照 |
| after_snapshot | 操作后数据快照 |
| confirm_status | 确认状态（0 待确认/1 已确认/2 已拒绝） |
| rollback_status | 回滚状态（0 未回滚/1 已回滚） |

## 七、会话生命周期

### 7.1 状态流转

```
ACTIVE (活跃) ──30分钟无消息──► IDLE (空闲) ──7天无消息──► ARCHIVED (归档)
   ▲                              │
   └────新消息────────────────────┘
```

由 `ConversationIdleScheduler` 定时任务驱动自动流转。

### 7.2 会话表

| 字段 | 说明 |
|------|------|
| id | 会话 ID |
| conversation_code | 会话编号（唯一） |
| user_id | 用户 ID |
| agent_id | 智能体 ID |
| status | 状态（ACTIVE/IDLE/ARCHIVED） |
| last_message_time | 最后消息时间 |
| context_project_id | 上下文项目 ID（可选） |

## 八、AI 智能体定义

### 8.1 Agent 表

| 字段 | 说明 |
|------|------|
| id | 智能体 ID |
| agent_name | 智能体名称 |
| agent_code | 智能体编号 |
| system_prompt | 系统提示词 |
| model_name | 模型名称 |
| temperature | 温度 |
| memory_strategy | 记忆策略 |

### 8.2 预置 Agent

| Agent | 用途 | system_prompt 要点 |
|-------|------|-------------------|
| BidAgent | 招标领域助手 | 熟悉招标流程，能查询项目/资质/投标 |
| NewsAgent | 资讯领域助手 | 能搜索资讯、更新订阅（已移除） |

## 九、AI 接口清单

| 接口 | URL | 方法 | 说明 |
|------|-----|------|------|
| 创建会话 | /api/ai/conversation | POST | |
| 会话列表 | /api/ai/conversation/list | GET | |
| 关闭会话 | /api/ai/conversation/{id} | DELETE | |
| 发送消息 | /api/ai/chat/{conversationId} | POST | SSE 流式返回 |
| 历史消息 | /api/ai/message/list/{conversationId} | GET | |
| 智能体 CRUD | /api/ai/agent/** | * | |
| 工具 CRUD | /api/ai/tool/** | * | |
| 确认 AI 操作 | /api/ai/action/confirm/{id} | PUT | 写操作工具确认 |
| 回滚 AI 操作 | /api/ai/action/rollback/{id} | PUT | |
| 记忆 CRUD | /api/ai/memory/** | * | |
| 用量查询 | /api/ai/usage/list | GET | |
| 用量导出 | /api/ai/usage/export | GET | |
| 知识库管理 | /api/ai/knowledge/** | * | 委托 ruoyi-rag |

## 十、技术风险与应对

| 风险 | 应对 |
|------|------|
| Spring AI 2.0 API 变动 | 以官方文档为准，先写最小 demo 验证 |
| SSE 高并发资源泄漏 | 单用户单连接 + 心跳 + 超时关闭 |
| AI 写操作安全风险 | 写操作确认 + 快照回滚 + 全量日志 |
| 长上下文 token 暴增 | 短期记忆限制 20 轮，长期记忆衰减 |
| 大模型超时 | 指数退避重试（AiModelTimeoutException） |

## 相关笔记
- [[工具调用体系]] — AI 能调用哪些工具
- [[RAG知识库]] — AI 如何基于企业文档回答
- [[核心模块详解]] — AI 模块内部结构
- [[05-亮点展示]] — AI 能力的商业价值
- [[状态机设计]] — AI 会话状态机
- [[架构设计]] — AI 在架构中的位置
