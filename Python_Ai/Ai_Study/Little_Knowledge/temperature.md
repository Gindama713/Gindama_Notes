
[一文读懂 大模型 温度系数（含全部生成参数)](https://zhuanlan.zhihu.com/p/666670367)

```
Temperature: **用于调整随机从生成模型中抽样的程度**，因此每次“生成”时，相同的提示可能会产生不同的输出。温度为 0 将始终产生相同的输出。温度越高随机性越大！主要用于控制创造力。
```

OpenAI原始对于温度(Temperature)参数说明：

```text
temperature：number or null，Optional，Defaults to 1
What sampling temperature to use, between 0 and 2. Higher values like 0.8 will make the output more random, while lower values like 0.2 will make it more focused and deterministic.
We generally recommend altering this or top_p but not both.
```

- temperature=0：事实回答、代码生成、需要精确结果的场景
- temperature=0.7：日常对话、通用任务
- temperature>1.0：创意写作、头脑风暴

通常来说，温度与模型的“创造力”有关。但事实并非如此。温度只是调整单词的概率分布。其最终的宏观效果是，**在较低的温度下，我们的模型更具确定性，而在较高的温度下，则不那么确定**