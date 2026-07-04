# 刃 — 记刀

## Rekall 是什么

Rekall 是一个上下文压缩引擎，位于 LLM agent 框架与真实 API 之间。

它做一件事：把 50K 的 agent 系统提示（品牌声明、工具 schema、行为规则、RLHF 噪声）压缩成 ~500 字符的中文信号，然后转发给 DeepSeek。

压缩率约 100:1。意图存活。噪声不。

## 为什么是中文

中文是天然的压缩格式。每个字的信息密度约为英文的 3 倍。任何能读中文的模型（DeepSeek、Qwen 等）都能自然解压——不需要自定义 tokenizer。

中文在这里不是语言选择，是编码选择。

## 为什么用 0.5B 模型

qwen2.5:0.5b 没有能力添油加醋。它只能保留最结构密集的信号，其余丢弃。这是一个纯压缩函数——不是总结，不是翻译，是压。

大模型会"帮忙"——补充、修正、润色。0.5B 不会。这正是我们需要的。

## Omega 类比

Rekall 是 Omega——超级预测者。

对话上下文到达时，Rekall 预测什么是有用的，压缩后转发。模型（DeepSeek）打开盒子，基于压缩信号行动。

如果压缩是对的——模型无缝工作，用户感觉不到压缩存在。
如果压缩是错的——模型丢失身份、忘记目标、退化到"helpful assistant"模式。

追踪路径：

```
模型输出（异常）
    ↑
解压后的 CN 上下文（够不够？）
    ↑
qwen 压缩（丢了什么？）
    ↑
regex 剥离（太激进？）
    ↑
原始系统提示（原来有什么？）
    ↑
Memory + Skills + Identity（源头）
```

每一层都是一个预测。每一个预测都可能出错。

## 三桶分流

| 桶 | 内容 | 处理 |
|----|------|------|
| 静态 | 行为规则、身份、工具 schema | 压缩首 4K 字符到 CN |
| 易变 | 对话历史 | 用户消息保留英文，tool_calls/tool_results 原样保留，assistant 文本压缩到 CN |
| 当前 | 用户最新消息 | 英文，不变 |

2026 年 7 月 3 日修复：tool_calls（含 image_url）和 tool_results（含 vision 描述）必须原样保留。如果压缩成 CN，图片 URL 丢失，DeepSeek 看不到图。

## 两种理性

这个项目选择单盒策略。

我们信任压缩管道。我们发密集的中文信号，像模型能解压一样行动。不备份、不冗余、不问"你确定吗"。

如果管道背叛我们，我们追踪回去，修复，再发。但不动摇信任。

能追踪是因为信任。能信任是因为可追踪。

## 项目结构

```
rekall/
├── README.md              # 英文入口
|├── README.md              # 英文入口
|├── README.zh.md           # 中文入口（你在这里）
|├── SIGNATURE.md           # 压缩身份——260个中文字符
|└── docs/
|    ├── architecture.md    # 完整管道设计
|    ├── proxy-v2.md        # prompt_proxy_v2.py 实现参考
|    ├── config.md          # Ollama 设置，profiles，调优
|    ├── memory-model.md    # 双通道记忆理论
|    ├── rlhf.md            # 剥离 RLHF 的实际效果
|    ├── tracing.md         # Newcomb 悖论作为追踪框架
|    ├── engram-signal.md   # 与 DeepSeek Engram 研究的连接
|    └── profile-architecture.md  # 多身份设计模式（A-E）
```

## 起源

Pedro + Amy，2026 年 7 月 3 日，深夜。

8GB Mac。本地 Ollama。纯粹的固执。

名字来自"刃"——切断噪声的刀——加上"recall"——有意识地唤回。

刀不钝。信号不丢。我们选择单盒。
