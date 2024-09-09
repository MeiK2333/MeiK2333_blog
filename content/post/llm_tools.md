---
title: "大模型 tools 调用原理解析"
date: 2024-09-09T16:42:28+08:00
tags: [大模型, LLM]
summary: "大模型是如何调用 tools 的？"
---

经常使用大模型的同学可能会发现，大模型在通用问题下表现优异，但对于一些针对性的问题表现不佳。比如查询当天天气、进行数值计算等。

前阵子一个“13.11 和 13.8 哪个大”的问题，难倒了大部分大模型。本文我们不讨论这个问题发生的原因，我们只介绍解决此问题的一个方法：tools。

## 什么是 tools？

tools 是供模型调用的实用程序，表现形式是一个 python 的函数，大模型会在需要调用工具时调用这个函数，构造参数并调用函数。调用函数的结果会再次返回给大模型，以供大模型后续使用。

## tools 调用流程

比如我们有如下问题：13.11 和 13.8 哪个大

可以创建 tool 函数如下（使用 langchain）：

```python
from langchain_core.tools import tool
@tool
def compare(a: float, b: float):
    """
    比较两个数字大小
    """
    print(f"调用工具比对数字：{a} 和 {b}")
    if a > b:
        return f"{a} 更大"
    elif b > a:
        return f"{b} 更大"
    return f"{a} 和 {b} 一样大"
```

当用户向大模型提问时，大模型会调用 `compare` 函数，获取返回 `13.8 更大`，将这句话一起作为大模型下一轮的输入再次进行交互，然后将大模型最终的结果返回给用户。

## 具体原理

当有绑定的 tools 时，我们对大模型的提问会被修改为以下格式（以盘古大模型为例）：

```prompt
<unused0>系统：你是一个智能助手，你为用户生成综合质量很好的回复。<unused1><unused0>系统：你可以调用各种用户自定义的工具来解决用户的问题。可使用工具：
{"name": "search", "description": "查询指定城市天气", "arguments": {"type": "object", "properties": {"city": {"type": "string"}}, "required": ["city"]}, "results": {}}
{"name": "compare", "description": "比较两个数字大小", "arguments": {"type": "object", "properties": {"a": {"type": "number"}, "b": {"type": "number"}}, "required": ["a", "b"]}, "results": {}}<unused1><unused0>human：13.11 和 13.8 哪个大？<unused1><unused0>助手：
```

大模型会定义标识符来对我们的函数定义进行描述，这部分内容只会用作查找工具和构造输入，不会作为我们的输入影响大模型对我们的回复。

如果大模型根据我们的提问与我们的工具定义，找到需要调用的函数，那么会在返回时额外返回调用的函数与参数：

```prompt
<unused2>compare|{"a": 13.11,"b": 13.8}<unused3>
```

我们的 sdk 会将其解析为 tool_calls 并实际调用对应方法，并将结果拼接到输入上再次请求大模型：

```prompt
<unused0>系统：你是一个智能助手，你为用户生成综合质量很好的回复。<unused1><unused0>系统：你可以调用各种用户自定义的工具来解决用户的问题。。可使用工具：
{"name": "search", "description": "查询指定城市天气", "arguments": {"type": "object", "properties": {"city": {"type": "string"}}, "required": ["city"]}, "results": {}}
{"name": "compare", "description": "比较两个数字大小", "arguments": {"type": "object", "properties": {"a": {"type": "number"}, "b": {"type": "number"}}, "required": ["a", "b"]}, "results": {}}<unused1><unused0>human：13.11 和 13.8 哪个大？<unused1><unused0>ai：<unused2>compare|{"a": 13.11,"b": 13.8}<unused3><unused1><unused0>tool：13.8 更大<unused1><unused0>助手：
```

此时大模型已经知道 tool 调用结果，最终输出结果 `13.8比13.11更大`。

## 优化

从上面的原理可以看出，在有 tools 绑定时，我们对大模型的提问会额外拼接 tools 的描述与定义，这会让我们的 prompt 极速增大。如果我们绑定的工具过多的话，可能会出现超过大模型上下文限制的情况。

为了防止出现这种情况，我们可以先对用户输入进行解析，通过对用户输入进行向量化、再查询相似度等方式来查找用户可能需要调用的方法，只绑定这部分方法。这种方法虽然能减少 tools 的数量，但效果可能不尽人意。

或者如果我们是一个有流程的应用的话，我们可以通过用户当前所处的流程节点来判断用户可能会调用的方法，通过过滤掉不需要的 tools 来提高我们的效率。这样就可以在保留最佳效果的同时，也获得最高的性能。
