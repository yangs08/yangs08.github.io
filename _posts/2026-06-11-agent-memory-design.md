---
layout: post
title: 在做 Agent 应用时的思考（二）：记忆不是把聊天记录塞满
date: 2026-06-11 10:00:00 +0800
tags: AI Agent 记忆 RAG
---

上一篇聊的是 Agent 应用第一步怎么拆。

这篇聊另一个很容易被低估的问题：

**Agent 到底怎么记东西。**

很多人一说 Agent 记忆，第一反应就是 RAG。

切文档，做 embedding，放进向量库，检索出来塞给模型。

这当然是一种记忆。

但真正做 Agent 应用时，会发现记忆不是一个向量库，而是一套分层系统。

有些信息只在当前这一步有用。

有些信息要在这轮任务里一直保留。

有些信息要跨会话保存。

有些信息不是知识，而是 Agent 做事的规则。

所以我现在会先问四个问题：

```text
这个信息要记多久？
谁会用它？
什么时候取出来？
取出来以后会不会影响决策？
```

![Agent 记忆分层示意](/images/posts/agent-memory-workbench.png)

<!--more-->

## 一、先把记忆分层

我会先把 Agent 的记忆分成几层：

```text
当前输入
  ↓
工作记忆
  ↓
情景记忆
  ↓
语义记忆
  ↓
规则记忆
```

### 当前输入

当前输入就是用户这一轮说的话、刚返回的工具结果、当前页面上看到的内容。

比如用户说：

```text
我这个订单三天没状态了，订单 ID 是 xxx。
```

这就是当前输入。

它用完就可以丢，不一定要长期保存。

### 工作记忆

工作记忆是当前任务正在用的状态。

比如电商客服 Agent 正在处理一个售后问题，它需要知道：

```text
order_id = xxx
当前意图 = 退款
订单状态 = 已发货
物流状态 = 3 天未更新
是否已创建工单 = 否
```

这些信息不应该只散落在聊天记录里。

最好放进结构化 State：

```json
{
  "order_id": "xxx",
  "intent": "refund_request",
  "order_status": "shipped",
  "logistics_status": "no_update_3_days",
  "ticket_created": false
}
```

工作记忆解决的是：

```text
当前任务做到哪一步了？
下一步该基于哪些已确认事实继续？
```

### 情景记忆

情景记忆记录的是过去发生过什么。

比如：

```text
这个用户上次也因为物流延迟申请过售后。
这个订单已经被人工客服介入过一次。
上一次类似问题，最终是补发解决的。
```

它不是通用知识，而是一次次具体经历。

情景记忆适合存数据库，按用户、订单、任务类型查询。

它解决的是：

```text
过去发生过什么，这次要不要参考？
```

### 语义记忆

语义记忆记录的是事实和知识。

比如：

```text
平台退款规则
商品售后政策
物流异常判定标准
公司内部 SOP
用户偏好
```

RAG 主要处理的就是这一层。

但语义记忆不等于向量库。

向量库只是读取语义记忆的一种方式。

有些信息更适合结构化存储：

```json
{
  "user_id": "u123",
  "preferred_language": "zh",
  "vip_level": "gold"
}
```

这种信息没必要 embedding，直接查字段更稳。

### 规则记忆

还有一类容易被忽略，叫规则记忆。

也就是 Agent 做事的方式。

比如：

```text
高风险退款不能自动承诺
回答客服问题要先查订单再判断
用户情绪激动时先安抚再解释规则
```

这类信息通常放在 system prompt、agent 配置、workflow 规则里。

它不是业务知识，而是 Agent 的行为边界。

所以 Agent 记忆不是一个向量库。

它至少包括：

```text
当前输入：这一步看到什么
工作记忆：当前任务做到哪
情景记忆：过去发生过什么
语义记忆：业务知识是什么
规则记忆：Agent 应该怎么做事
```

## 二、上下文窗口满了怎么办

做多轮 Agent，很快会遇到一个问题：

上下文越来越长。

用户聊了十几轮，工具调了十几次，Agent 中间还做了几次判断，prompt 很快就会变重。

这时候不能简单地把所有历史都塞进去。

我会用三种方式处理。

第一种，滑动窗口。

只保留最近几轮对话：

```text
保留最近 5 轮
更早的直接丢掉
```

适合闲聊、低风险问答。

第二种，摘要压缩。

把历史对话压成一个简短摘要：

```text
用户正在处理订单 xxx 的退款问题。
订单已发货，物流 3 天无更新。
用户希望退款，系统已建议创建工单。
```

适合对话很长，但后面还需要知道整体背景。

第三种，结构化抽取。

不是总结一段自然语言，而是抽出关键字段：

```json
{
  "order_id": "xxx",
  "intent": "refund_request",
  "confirmed_facts": [
    "order_shipped",
    "logistics_no_update_3_days"
  ],
  "next_action": "create_ticket"
}
```

我更喜欢第三种。

因为 Agent 应用里，我们很多时候不是要保留完整聊天，而是要保留会影响下一步决策的字段。

判断标准可以简单一点：

```text
只影响语气 → 可以丢
影响任务状态 → 写进 State
影响长期偏好 → 写进长期记忆
影响业务判断 → 写进结构化字段
```

## 三、RAG 和 Fine-tuning 怎么选

这个问题很常见。

我的判断是：

**知识问题优先 RAG，行为问题才考虑 Fine-tuning。**

比如电商客服 Agent 要知道最新退货政策。

这个政策可能每个月变。

那就应该放知识库，用 RAG 查。

因为你不可能每改一次政策就重新训练模型。

```text
政策常变
需要引用来源
需要权限隔离
需要随时更新
→ RAG
```

Fine-tuning 更适合另一类问题。

比如你希望模型稳定输出某种结构：

```json
{
  "intent": "...",
  "risk_level": "...",
  "next_action": "..."
}
```

或者希望它在某个领域任务上更稳定，比如分类、固定格式生成、特定语气和指令跟随。

所以我不会把 Fine-tuning 当知识库。

它更像是让模型形成一种能力或习惯。

```text
知识会变 → RAG
行为要稳 → Fine-tuning
两者都要 → RAG + Fine-tuning
```

比如客服 Agent 可以这样组合：

```text
RAG：查最新售后政策
Fine-tuning：稳定识别用户意图和输出结构化结果
```

## 四、Long Context 和 RAG 怎么选

Long Context 很诱人。

模型上下文越来越长，看起来好像可以直接把所有资料塞进去。

但长上下文解决的是「装得下」，不是「一定找得准」。

如果资料很少，比如一份合同、一份产品说明书、几页 SOP，直接塞进长上下文没问题。

```text
文档少
需要整体阅读
预算够
→ Long Context
```

但如果是企业知识库，几万篇文档、权限不同、版本不同、内容经常变，就不能全塞。

这时候应该用 RAG。

```text
知识库大
内容常变
需要权限过滤
成本敏感
→ RAG
```

实际项目里，我更倾向于混用：

```text
RAG 先检索
  ↓
rerank 精排
  ↓
把少量高相关内容放进 Long Context
  ↓
模型回答
```

也就是说，不是 Long Context 替代 RAG。

而是 RAG 负责筛选，Long Context 负责阅读。

## 五、Embedding 模型怎么选

Embedding 模型不要只看排行榜。

要看自己的语料和场景。

我会先问几个问题：

```text
中文多不多？
中英混合多不多？
文档长不长？
需不需要混合检索？
```

如果是中文或多语言场景，BGE-M3 是一个常见选择。

它的 M3 指的是：

```text
Multi-Lingual
Multi-Functionality
Multi-Granularity
```

也就是支持多语言、多种检索方式、长短文本。

如果是中英双语、跨语言 RAG，可以看 BCE-embedding。

它的重点是 bilingual 和 cross-lingual，也就是双语和跨语言检索。

如果是英文轻量场景，MiniLM-L6 这类模型很常用。

它体积小，适合轻量语义搜索、聚类、原型验证。

如果你完全在 OpenAI API 体系里，也可以直接用 `text-embedding-3-small` 或 `text-embedding-3-large`。

简单记：

```text
中文 / 多语言 / 长文档 → BGE-M3

中英双语 / 跨语言 RAG → BCE-embedding

英文轻量 / 本地小模型 → MiniLM-L6

OpenAI API 栈 / 少折腾 → text-embedding-3-small / large
```

但最后一定要用自己的数据评测。

不要只看模型名。

## 六、向量数据库怎么选

向量数据库也不要一上来就选最复杂的。

我会按项目阶段选。

本地开发、小项目原型，用 Chroma。

它上手快，适合先把 RAG 链路跑通。

生产环境、想要托管服务、关注性能和运维成本，可以看 Pinecone。

如果你很看重混合搜索，也就是向量 + 关键词，Weaviate 很适合放进候选。

如果你的系统本来就在 PostgreSQL 生态里，PgVector 很自然。

因为它能把向量检索放进原有 PG 体系里，权限、事务、备份、业务表关联都更顺。

简单记：

```text
本地原型 → Chroma

托管生产 → Pinecone

混合搜索 → Weaviate

已有 PG 体系 → PgVector
```

但这不是绝对排名。

真正要看：

```text
数据量
QPS
权限隔离
混合检索
运维能力
成本
```

## 七、普通 RAG 和 Agentic RAG 怎么选

普通 RAG 是一次检索：

```text
用户问题
  ↓
检索 top-k
  ↓
塞进 prompt
  ↓
模型回答
```

它适合标准问题。

比如：

```text
退货政策是什么？
会员权益有哪些？
发票怎么开？
```

Agentic RAG 是 Agent 参与检索过程。

它会自己判断：

```text
要不要检索？
用什么关键词检索？
检索结果够不够？
要不要换个问法再搜一次？
要不要读原文更长上下文？
```

它适合复杂问题。

比如：

```text
这个用户能不能退款？
这份合同里甲方有没有违约风险？
这几个政策互相冲突时应该听哪个？
```

这类问题不是一次 top-k 就能解决。

它需要多次查证、对比、补证据。

但 Agentic RAG 的代价也明显：

```text
更慢
更贵
更难调试
更依赖评估
```

所以不要默认所有问题都 Agentic。

```text
高频标准问答 → 普通 RAG

复杂、多跳、需要查证 → Agentic RAG
```

## 写在最后

Agent 的记忆，不是存得越多越好。

真正要设计的是四件事：

```text
什么信息要写？
写到哪里？
什么时候读？
读出来以后怎么影响决策？
```

比如电商客服 Agent：

```text
当前订单状态 → 写 State
历史投诉记录 → 写数据库
售后政策 → 写知识库 / RAG
用户偏好 → 写长期 profile
高风险规则 → 写系统 prompt / 规则引擎
```

记忆做不好，Agent 会变成两种极端。

一种是什么都不记，每轮都像第一次见用户。

另一种是什么都记，最后被一堆无关上下文拖慢、带偏。

真正能用的 Agent 记忆，应该是该记的能记住，该忘的能忘掉，该查的时候能查回来。

参考：

- [OpenAI Fine-tuning](https://platform.openai.com/docs/guides/fine-tuning)
- [OpenAI Embeddings](https://platform.openai.com/docs/guides/embeddings)
- [OpenAI Retrieval](https://platform.openai.com/docs/guides/retrieval)
- [LangGraph Memory](https://docs.langchain.com/oss/python/concepts/memory)
- [BGE-M3](https://huggingface.co/BAAI/bge-m3)
- [BCE Embedding](https://huggingface.co/maidalun1020/bce-embedding-base_v1)
- [Chroma](https://docs.trychroma.com/docs/overview/introduction)
- [Pinecone](https://docs.pinecone.io/guides/get-started/overview)
- [Weaviate Hybrid Search](https://docs.weaviate.io/weaviate/search/hybrid)
- [PgVector](https://github.com/pgvector/pgvector)
