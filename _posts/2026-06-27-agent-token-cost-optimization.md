---
layout: post
title: 关于 Token 优化
date: 2026-06-27 08:30:00 +0800
tags: AI Agent Token 成本优化
---

最近在看 Agent 应用里的 Token 成本优化。

我一开始以为，这个问题可能就是把 Prompt 写短一点，或者换个便宜模型。

但查了一圈主流做法之后，发现不是。

实际业务里优化 Token 成本，通常不是靠一个技巧，而是分四层做：

```text
模型层：用更便宜的方式调用模型
上下文层：减少每次塞给模型的内容
流程层：减少不必要的模型调用
监控层：知道钱到底花在哪里
```

![Token 成本优化的四个阀门](/assets/agent-token-cost-optimization-illustrations/01-four-cost-valves.png)

也就是说，Token 成本不是最后抠出来的。

它是架构设计的一部分。

<!--more-->

## 一、Prompt / Context Caching，缓存稳定前缀

这是现在各大厂商都在推的主流方案。

适合缓存的内容一般是这些：

```text
System Prompt
工具说明
业务规则
Few-shot 示例
长文档背景
多轮对话里的稳定历史
```

这些内容有一个共同点：

```text
很长，但变化不大。
```

比如一个客服 Agent，每一轮请求都要带上：

```text
客服行为规范
工具调用说明
售后规则
回复口径
安全边界
```

如果每次都让模型重新处理一遍，成本和延迟都会浪费。

OpenAI 的 Prompt Caching 会在请求里返回 `cached_tokens`，可以用来观察命中了多少缓存 token。它要求 prompt 至少 1024 tokens 才会产生缓存命中统计。

参考：[OpenAI Prompt Caching](https://developers.openai.com/api/docs/guides/prompt-caching)

Anthropic 的 Prompt Caching 更明确支持自动缓存和显式 cache breakpoint，适合长对话、多工具 Agent、重复指令集。它的文档里提到，缓存命中可以减少处理时间和成本，默认缓存 5 分钟，也支持 1 小时缓存。

参考：[Anthropic Prompt Caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)

AWS Bedrock 也支持 Prompt Caching，强调适合「长且重复的上下文」，比如用户上传文档后反复问问题。

参考：[Amazon Bedrock Prompt Caching](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html)

所以 Prompt 结构也要设计。

一个简单原则是：

```text
稳定内容放前面，动态内容放后面。
```

![Prompt Caching 缓存稳定前缀](/assets/agent-token-cost-optimization-illustrations/02-prompt-cache-box.png)

比如：

```text
System Prompt
工具定义
业务规则
Few-shot 示例
---
当前用户输入
当前 State
本轮工具返回
```

能缓存 System Prompt、工具说明、业务规则，就不要每轮都让模型重新处理。

这不是 Prompt 小技巧。

这是成本结构设计。

## 二、模型路由，小任务不用大模型

OpenAI 的成本优化文档里提到几个方向：

```text
减少请求
减少 token
选择更小的模型
```

参考：[OpenAI Cost Optimization](https://developers.openai.com/api/docs/guides/cost-optimization)

实际项目里，模型一般会分层使用。

大概是这样：

```text
规则 / 代码：订单号提取、手机号校验、权限判断
小模型：意图识别、分类、字段抽取
中模型：工具选择、摘要、普通回复
大模型：复杂推理、高风险判断、长文生成
```

比如一个客服 Agent：

```text
用户输入
  ↓
规则提取订单号
  ↓
小模型识别意图
  ↓
工具查订单 / 物流
  ↓
中模型生成回复
  ↓
高风险投诉才上大模型或转人工
```

不是所有节点都要用最强模型。

Agent 成本优化的第一刀，是把任务拆细，然后按难度路由模型。

比如：

```text
查订单、查物流、发票怎么开
```

这种高频标准问题，不需要大模型每次完整推理。

但如果用户说：

```text
你们物流一直不更新，我已经投诉平台了，现在要求退款加赔偿。
```

这种涉及投诉、退款、赔偿和风险边界的问题，才值得上更强模型，甚至直接转人工。

所以模型路由不是单纯省钱。

它是在问：

```text
这个节点到底需要多少智能？
```

## 三、RAG 上下文裁剪，不要无脑塞 top-k

主流 RAG 成本优化不是「少检索」，而是：

```text
多召回，精筛选，少塞入。
```

常见做法是：

```text
Query rewrite
Hybrid search
Rerank
相关性阈值过滤
去重
按 token budget 装填
只取命中字段
长文档先摘要
表格先查询再截取子表
```

核心是：

```text
向量库可以多召回，但进 Prompt 的内容必须少而准。
```

很多 RAG 系统一开始会这么做：

```text
向量检索 top-5
  ↓
直接拼进 Prompt
  ↓
让模型回答
```

如果答不好，就把 top-5 改成 top-10。

然后成本更高，效果还不一定更好。

因为多塞进去一段不相关内容，不是多一份参考，而是多一份噪声。

比如用户问：

```text
耳机签收两天坏了能不能退？
```

系统不应该把整篇售后规则文档都塞给模型。

更合理的是先召回更多候选，再用 Reranker 精排，只留下和「耳机」「签收两天」「质量问题」「退换货」相关的片段。

如果命中的是大表格，也不要把整张表喂给模型。

先查询相关行列，再把子表交给模型。

这块其实可以和我之前写的那篇 RAG 文章接上：

[为什么向量检索不能直接喂给大模型？——从信息瓶颈到 RAG 工程实践]({% post_url 2026-05-25-why-vector-search-cannot-feed-llm-directly %})

RAG 省 token 的本质，不是少找资料。

而是只把真正能回答问题的证据放进上下文。

## 四、Agent 上下文裁剪，保留必要 State，不保留全部历史

我还查到一篇很贴 Agent 场景的研究，结论挺有意思。

在企业工具调用 Agent 里，完整保留所有历史并不一定最好。

研究里对比了完整上下文、只保留最近工具交互、加摘要等方式，发现「保留最近工具调用 + 压缩摘要」可以同时提升成功率并减少 token。

参考：[Less Context, Better Agents](https://arxiv.org/abs/2606.10209)

这点非常适合 Agent 应用。

因为 Agent 不是记得越多越好。

它需要的是当前任务还用得上的 State，而不是所有历史对话。

比如一个客服 Agent，真正需要保留的可能是：

```text
order_id
intent
tool_results
risk_level
next_action
```

可以丢弃或摘要的是：

```text
寒暄
重复确认
过期工具结果
无关上下文
旧分支推理
```

这和人做事很像。

你处理一个退款工单时，不需要记住用户前面每一句情绪化表达。

你需要知道的是：

```text
订单是什么
现在查到了什么
规则怎么判断
风险等级是什么
下一步该做什么
```

所以 Agent 上下文优化，不是机械截断历史。

而是把历史变成结构化 State。

## 五、Batch / 异步任务，用更便宜的处理方式

还有一类优化，不在实时链路里。

OpenAI Batch API 官方写的是，异步批处理可以比同步 API 低 50% 成本，适合不需要立即返回的任务，比如评测、大规模分类、embedding 内容库。

参考：[OpenAI Batch API](https://developers.openai.com/api/docs/guides/batch)

适合走 Batch 的任务：

```text
离线评测
批量生成摘要
批量打标签
知识库 embedding
日志清洗
用户画像摘要更新
```

不适合走 Batch 的任务：

```text
实时客服回复
用户正在等待的对话
高优先级工具决策
```

这块在业务里很重要。

很多团队一开始会把所有东西都塞进实时链路。

用户问一句，系统顺手做：

```text
回复生成
会话摘要
用户画像更新
工单标签生成
评测打分
日志归因
```

听起来很自动化。

但用户在等，成本也在烧。

更合理的是拆开：

```text
实时链路：只完成当前用户任务

离线链路：做总结、归因、评测、画像更新
```

一句话：

```text
实时链路用同步，离线链路用 Batch。
```

不要拿实时模型调用去跑后台清洗任务。

## 六、Trace + Token 成本监控，先知道钱花在哪

最后一层是监控。

没有 Trace，就不知道钱花在哪。

LangSmith 文档里提到，Trace 里需要记录 provider、model、token counts、cost，才能做成本展示和分析。它支持记录 input tokens、output tokens、total tokens，以及 cache read / cache creation 等细分字段。

参考：[LangSmith Log LLM Calls](https://docs.langchain.com/langsmith/log-llm-trace)

实际项目里，我会按这些维度看：

```text
每个节点花多少 token
每个工具调用前后花多少 token
哪个 Prompt 最贵
哪个用户场景最贵
RAG context 平均长度
cache hit rate
每次任务平均成本
成本 / 成功任务
```

这里最有价值的指标不是总 token。

而是：

```text
Cost per successful task
```

![Trace 里找到 token 黑洞](/assets/agent-token-cost-optimization-illustrations/03-trace-token-holes.png)

因为有些 Agent token 花得多，但任务完成了，也许是值得的。

有些 Agent token 花得也多，最后还没解决问题，这才是真正浪费。

比如你看到某类售后问题：

```text
平均 12000 tokens
任务成功率 30%
最后 60% 转人工
```

这时候就要回头看：

```text
是不是上下文塞太多？
是不是 RAG 检索噪声太多？
是不是低置信度没有提前转人工？
是不是用了不必要的大模型？
是不是每轮都重复读工具说明？
```

Token 优化一定要和评测监控连在一起。

只看成本，不看成功率，会把系统优化坏。

只看成功率，不看成本，系统上线后会贵到不可持续。

## 七、主流落地组合

把上面的方案放到一起，大概是这张表：

| 层级 | 主流做法 | 解决什么 |
|---|---|---|
| Provider | Prompt caching / Batch API | 重复上下文、离线任务成本 |
| Model | 小模型路由 / 大模型兜底 | 不同任务不同成本 |
| Context | 摘要、裁剪、State 化 | 减少无效输入 token |
| RAG | Rerank、阈值、预算装填 | 减少知识库噪声 |
| Agent | 节点化、提前结束、工具优先 | 减少不必要模型调用 |
| Observability | LangSmith / Langfuse / Phoenix | 找到 token 黑洞 |

所以这篇文章真正想说的是：

```text
Token 优化不是把 Prompt 写短。
```

而是让模型少看不该看的东西，少做不该做的事。

更具体一点：

```text
稳定内容缓存
简单任务路由到小模型或规则
RAG 只塞必要证据
Agent 只保留必要 State
离线任务走 Batch
用 Trace 找到成本黑洞
```

这些东西连起来，Token 成本才真的能降下来。

如果只在最后改一句 Prompt，基本救不了一个上下文设计粗糙的 Agent 系统。
