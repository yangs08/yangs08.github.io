---
layout: post
title: Orchestrator 案例：订单 3 天没状态，用户要退款时 Agent 怎么流转
date: 2026-06-10 12:30:00 +0800
tags: AI Agent 架构 多Agent 电商客服
hidden: true
---

上一篇写的是 [Agent 应用第一步怎么拆]({% post_url 2026-06-10-agent-architecture-first-step %})。

这篇用一个具体场景，把它落下来。

用户进来说：

```text
我订单 3 天没状态了，订单 ID：xxx，我要退款。
```

这个问题看起来像一句普通客服咨询，但里面其实有三件事：

```text
查订单状态
  ↓
判断物流是否异常
  ↓
判断能不能退款
```

所以我不会让一个回复 Agent 直接回答「可以」或者「不可以」。

这个场景更适合用 Orchestrator。

也就是一个主客服 Agent 负责对外沟通和整体调度，后面几个专业 Agent 负责查事实、判断规则、控制风险。

<!--more-->

## 一、先看整体架构

这个系统大概长这样：

[![AI 电商客服 Agent 系统架构](/images/posts/agent-customer-service-architecture.png)](/images/posts/agent-customer-service-architecture.png)

这里有两个关键点。

第一，用户只和主客服 Agent 说话。

用户不需要知道后面有订单 Agent、物流 Agent、售后 Agent、风控 Agent。

第二，所有 Agent 的关键结论都写进 Shared State。

Agent 之间不是靠自然语言互相转述，而是通过 State 对齐事实。

## 二、主 Agent 先不要急着答复，而是先拆任务

用户这句话：

```text
我订单 3 天没状态了，订单 ID：xxx，我要退款。
```

主客服 Agent 先解析出几个关键信息：

```json
{
  "order_id": "xxx",
  "intent": ["order_status_query", "refund_request"],
  "user_claim": "订单3天没状态",
  "requested_action": "refund"
}
```

然后初始化 Shared State：

```json
{
  "order_id": "xxx",
  "intent": ["order_status_query", "refund_request"],
  "order_status": null,
  "logistics_status": null,
  "days_without_update": null,
  "logistics_abnormal": null,
  "refund_eligible": null,
  "risk_level": null,
  "next_action": null
}
```

这一步很重要。

因为系统不能只记住「用户要退款」。

它还要知道，现在有哪些事实已经确认，哪些事实还没确认。

## 三、哪些 Agent 可以并行，哪些必须等待

这个场景里，订单 Agent 和物流 Agent 可以先并行。

因为它们互不依赖。

订单 Agent 查订单是否支付、是否发货、是否签收。

物流 Agent 查物流轨迹、最后一次更新时间、是否长时间无更新。

[![订单 3 天无状态退款请求的 Agent 流转](/images/posts/agent-customer-service-refund-flow.png)](/images/posts/agent-customer-service-refund-flow.png)

但售后规则 Agent 不能一开始就拍板。

因为它要依赖订单和物流结果。

同样是用户说「我要退款」，订单未发货、订单已发货、订单已签收，对应的售后规则完全不一样。

所以这里不是所有 Agent 都一起跑。

能并行的并行，有依赖的等待。

## 四、每个 Agent 写自己负责的字段

订单 Agent 查完后，可能写入：

```json
{
  "order_status": "shipped",
  "paid": true,
  "seller_shipped_at": "2026-06-07",
  "product_type": "standard_goods"
}
```

物流 Agent 查完后，可能写入：

```json
{
  "logistics_status": "in_transit",
  "last_tracking_update_at": "2026-06-07",
  "days_without_update": 3,
  "logistics_abnormal": true,
  "abnormal_reason": "no_tracking_update_3_days"
}
```

这时候 State 里已经有了事实。

售后规则 Agent 再读取这些字段，判断退款资格：

```json
{
  "refund_eligible": false,
  "refund_reason": "goods_in_transit_refund_requires_seller_or_platform_review",
  "suggested_action": "create_logistics_delay_ticket",
  "need_human_review": true
}
```

风控 Agent 再补充风险判断：

```json
{
  "risk_level": "medium",
  "risk_reason": "refund_requested_before_delivery_with_logistics_abnormal",
  "blocked_auto_action": ["auto_refund"],
  "allowed_auto_action": ["create_ticket", "send_status_explanation"]
}
```

注意这里的重点。

风控 Agent 不是写一句「建议谨慎」。

它要明确写出：

```text
blocked_auto_action = auto_refund
```

这样主 Agent 后面就不能自动退款。

这就是结构化 State 的价值。

它把一句模糊建议，变成了系统能执行的限制。

## 五、主 Agent 最后基于 State 做决策

最后 Shared State 可能长这样：

```json
{
  "order_id": "xxx",
  "order_status": "shipped",
  "logistics_status": "in_transit",
  "days_without_update": 3,
  "logistics_abnormal": true,
  "refund_eligible": false,
  "need_human_review": true,
  "risk_level": "medium",
  "blocked_auto_action": ["auto_refund"],
  "allowed_auto_action": ["create_ticket", "send_status_explanation"],
  "next_action": "create_logistics_delay_ticket"
}
```

这时主客服 Agent 不能回复：

```text
好的，已经给你退款。
```

它应该回复：

```text
我帮你查了一下，这笔订单目前已经发货，在运输中，但物流确实已经 3 天没有更新。这个状态下我不能直接为你自动退款，我会先帮你提交物流异常核查/售后工单。核查后如果确认物流异常，可以继续走退款或补发处理。
```

如果系统支持自动建工单，Orchestrator 可以调用工单工具：

```json
{
  "type": "create_ticket",
  "ticket_type": "logistics_delay_refund_request",
  "order_id": "xxx",
  "reason": "no_tracking_update_3_days",
  "user_requested_action": "refund"
}
```

然后再把工单号写回 State，并回复给用户。

## 六、为什么这个场景适合 Orchestrator

因为这个场景不是简单问答。

用户一句话里同时有查询、判断、动作请求。

如果直接做成链式流程：

```text
用户
  ↓
订单 Agent
  ↓
物流 Agent
  ↓
售后 Agent
  ↓
风控 Agent
  ↓
回复 Agent
```

看起来也能跑。

但问题是链式流程太像传话。

前一个 Agent 的重点，可能在后一个 Agent 那里被改写、忽略、弱化。

尤其是「不能自动退款」「需要人工审核」这种限制，一旦变成自然语言在 Agent 之间传，很容易丢。

Orchestrator 的好处是，它让主 Agent 始终负责全局。

哪些任务可以并行，它决定。

哪些任务必须等 State 更新后再执行，它决定。

最后能不能退款，也不是某个回复 Agent 自己发挥，而是根据订单状态、物流状态、售后规则、风控结果一起判断。

这个场景里，智能不是体现在 Agent 们聊得多热闹。

而是体现在系统能不能稳稳地把一句用户请求，拆成事实、规则、风险和动作，然后再给出一个不会越权的回复。

这才是电商客服 Agent 真正难的地方。
