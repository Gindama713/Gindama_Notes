### 什么是 Function Calling

简单说：Function Calling 就是 LLM 输出一个结构化的 JSON，告诉你它想调用哪个函数、传什么参数。
不是 LLM 真的去执行这个函数——它只是告诉你它想调用，执行的事还得由你来做。
整个流程通常是这样的：

- 1. 你告诉 LLM："这里有一些函数可以调用，每个函数的名字、参数、用途是……"
- 2. 用户发消息："帮我查一下明天北京的天气。"
- 3. LLM 回复："我要调用 get_weather 函数，参数是 city='北京', date='明天'。"
- 4. 你去调用这个函数（真的去查天气 API），得到结果。
- 5. 你把结果塞回给 LLM："刚才的函数调用返回了：温度 25 度，晴天。"
- 6. LLM 基于这个结果，给用户一个自然语言回答："明天北京晴天，温度 25 度，很舒适。"
看到了吗？LLM 负责想，你负责执行。

### 定义工具的 [[JSON]] Schema

要让 LLM 调用工具，你得先告诉它有哪些工具可用。

这个告诉的过程，就是用 JSON Schema 描述函数。

来看一个标准的函数定义格式：

```python
# 这是一个典型的工具定义（用 Python 字典表示，最终会转成 JSON）
tool_definition = {
    "type": "function",
    "function": {
        "name": "get_weather",  # 函数名
        "description": "查询指定城市指定日期的天气",  # 函数用途描述，LLM 会看这个
        "parameters": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "城市名称，比如'北京'、'上海'、'深圳'",
                },
                "date": {
                    "type": "string",
                    "description": "日期，格式为 YYYY-MM-DD，比如'2024-06-18'",
                },
                "unit": {
                    "type": "string",
                    "enum": ["celsius", "fahrenheit"],
                    "description": "温度单位，摄氏度或华氏度，默认摄氏度",
                },
            },
            "required": ["city", "date"],  # 必填参数
        },
    },
}

# 另一个工具：搜索
search_tool = {
    "type": "function",
    "function": {
        "name": "web_search",
        "description": "在互联网上搜索最新信息，适合查新闻、实时数据、未知知识",
        "parameters": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "搜索关键词或问题",
                },
                "num_results": {
                    "type": "integer",
                    "description": "返回多少条结果，默认 5",
                    "default": 5,
                },
            },
            "required": ["query"],
        },
    },
}

# 另一个工具：计算器
calculator_tool = {
    "type": "function",
    "function": {
        "name": "calculate",
        "description": "执行数学计算，支持加减乘除、幂运算等",
        "parameters": {
            "type": "object",
            "properties": {
                "expression": {
                    "type": "string",
                    "description": "数学表达式，比如'25 * 4 + 10'、'sqrt(16)'",
                },
            },
            "required": ["expression"],
        },
    },
}

print(f"已定义 {len([tool_definition, search_tool, calculator_tool])} 个工具：get_weather, web_search, calculate")
# 输出：已定义 3 个工具：get_weather, web_search, calculate
```
description 非常重要 LLM 就是靠这个描述来理解这个工具是干什么的、什么时候用。
参数也要描述清楚 比如 unit 有 enum 约束，LLM 就知道只能从这两个值里选。
required 标出必填项 LLM 会确保这些参数一定有值。

1. 模型绝对不会执行代码：它只输出调用指令
2. 所有真实操作都由你的 Python 代码完成，安全可控
3. 工具可以无限扩展：加新工具只需定义JSON + 写对应函数
4. 多步执行就是 ReAct 循环：模型会反复调用工具→拿结果→再调用
