---
layout: post
title: 在做 Agent 应用时的思考（一）：第一步怎么拆
date: 2026-06-10 12:00:00 +0800
tags: AI Agent 架构 多Agent
---

我在做 Agent 应用时，第一步不是先选框架。

我会先想三个问题：

1. 用单 Agent，还是多 Agent？
2. 如果用多 Agent，它们怎么协作？
3. Agent 之间怎么通信，State 怎么共享？

这三个问题没想清楚，后面选什么框架都容易跑偏。

<!--more-->

## 一、单 Agent 还是多 Agent

先判断一个问题：**能不能用一个 Agent 做完？**

比如做一个 AI 电商客服，用户问：

```text
我的订单到哪了？
```

这个流程是线性的：

```text
用户
  ↓
客服 Agent
  ↓
订单查询工具
  ↓
物流查询工具
  ↓
回复用户
```

这里虽然用到了订单系统和物流系统，但它们只是工具，不是 Agent。

查订单、查物流是确定性动作，输入 ID，返回状态，不需要独立决策。

但如果用户说：

```text
耳机用了两天坏了，我要退款，不然投诉。
```

这个问题就不一样了。

它至少包含几类判断：

```text
订单事实判断
  ↓
售后规则判断
  ↓
风险判断
  ↓
回复生成
```

这时候可以拆成：

```text
用户
  ↓
主客服 Agent
  ↓
订单 Agent
  ↓
售后 Agent
  ↓
风控 Agent
  ↓
主客服 Agent 回复用户
```

这里考虑多 Agent，不是因为步骤变多，而是因为责任变多。

订单 Agent 负责事实，订单是否存在、是否签收、签收几天。

售后 Agent 负责规则，是否符合退货、换货、维修条件。

风控 Agent 负责风险，是否涉及高频退款、投诉升级、异常赔付。

主客服 Agent 负责对话体验和最终回复。

判断标准：

```text
流程简单、目标单一、工具确定 → 单 Agent

职责不同、需要专业分工、风险更高 → 多 Agent
```

一句话：**不是步骤多就要多 Agent，而是责任不同才需要多 Agent。**

## 二、多 Agent 怎么协作

决定用多 Agent 后，先选协作方式，而不是马上堆角色。

### Workflow，固定流程

如果业务流程是固定的，就用 Workflow。

比如退货申请：

```text
确认订单
  ↓
判断是否在售后期
  ↓
收集退货原因
  ↓
创建售后单
```

这种流程顺序明确，Agent 可以负责意图识别、补全信息、生成回复，但流程本身应该固定。

### Graph，状态跳转

如果流程不是一条直线，而是会来回跳，就用 Graph。

比如用户一开始说退款，后来又说想换货，中间还要补材料。

这时候流程可能是：

```text
收集信息
  ↓
售后判断
  ↓
缺材料？ → 回到收集信息
  ↓
用户确认
  ↓
执行售后
  ↓
失败？ → 转人工
```

Graph 关心的是状态：

```text
当前在哪个状态？
下一步允许去哪里？
什么条件下回退？
什么条件下结束？
```

### Orchestrator，主 Agent 调度

如果任务不是固定流程，需要主 Agent 动态拆任务，就用 Orchestrator。

比如用户说：

```text
我这几个订单都有问题，你帮我看看怎么处理。
```

这时候主 Agent 需要决定查哪些订单、调用哪些子 Agent、哪些可以并行、最后怎么汇总。

```text
用户
  ↓
主 Agent
  ├─ 订单 Agent
  ├─ 售后 Agent
  └─ 风控 Agent
  ↓
主 Agent 汇总回复
```

Orchestrator 的核心是有一个总负责人，负责拆任务、派任务、收结果、做最终回复。

电商客服这种需要稳定交付的场景，我会更倾向这种模式。

### Peer-to-Peer，平等讨论

Peer-to-Peer 是多个 Agent 平等协作，没有明确主 Agent。

```text
策略 Agent ←→ 数据 Agent
    ↕           ↕
内容 Agent ←→ 增长 Agent
```

它适合头脑风暴、方案讨论、研究分析，不太适合客服这种需要明确结果的场景。

简单记：

```text
固定流程 → Workflow

状态跳转 → Graph

主 Agent 动态调度 → Orchestrator

开放讨论 → Peer-to-Peer
```

## 三、Agent 之间怎么通信，State 怎么共享

多 Agent 拆完后，重点不是有几个 Agent，而是它们怎么传信息。

最直觉的方式，是让 Agent 之间用自然语言聊天：

比如：

```text
售后 Agent：这个用户符合质量问题售后，但最好让他补充一段故障视频。

风控 Agent：这个用户历史退款次数偏高，建议人工审核。

回复 Agent：好的，已经为您申请退款。
```

问题是，前面两个 Agent 明明给了限制：

```text
需要补充视频
需要人工审核
```

但回复 Agent 最后直接承诺退款。

根因是自然语言太松散。

「建议人工审核」到底是强制规则，还是普通建议？

「可以退款」到底是已经可以执行退款，还是只是在规则上符合退款条件？

「需要补充视频」到底是必须补充，还是可选补充？

这些信息如果藏在一段话里，后面的 Agent 很容易理解错、漏读，或者为了生成更顺的回复，把关键限制忽略掉。

所以自然语言适合解释，不适合做系统协议。

我会把通信拆成两层：

```text
Message：派任务

State：存结果
```

Message 解决的是：

```text
我要让哪个 Agent 做什么？
```

比如主客服 Agent 要让售后 Agent 判断是否能退款，就发一个结构化任务：

```json
{
  "type": "check_refund_policy",
  "order_id": "123",
  "issue_type": "quality_problem",
  "signed_days": 2
}
```

Message 不保存最终结论，只负责触发一个 Agent 去做事。做完后，结果写进 State。

State 解决的是：

```text
当前任务已经确认了哪些事实？
系统下一步应该怎么走？
```

比如：

```json
{
  "order_status": "signed",
  "signed_days": 2,
  "refund_eligible": true,
  "required_material": ["fault_video"],
  "risk_level": "medium",
  "next_action": "ask_user_upload_video"
}
```

主客服 Agent 最后生成回复时，不读一堆聊天记录，而是读这个结构化 State。

它看到：

```text
refund_eligible = true
required_material = fault_video
risk_level = medium
next_action = ask_user_upload_video
```

所以它不能直接说「已经为您退款」。

它应该说：

```text
这笔订单在售后期内，可以为你申请售后。为了继续处理，需要你先补充一段故障视频，我们会根据材料继续审核。
```

区别是：

```text
Message 是过程里的任务分发。

State 是系统里的事实记录。
```

这样做有几个好处：

第一，信息不会丢。

每个 Agent 的关键判断都会落到字段里，而不是散落在自然语言对话里。

第二，系统容易调试。

如果最后回复错了，可以回看 State：

```text
是 refund_eligible 写错了？

是 risk_level 写错了？

还是主 Agent 没按 next_action 回复？
```

问题能定位到具体字段。

第三，可以做规则兜底：

比如只要：

```text
risk_level = high
```

系统就强制转人工。

不管回复 Agent 怎么想，都不能自动退款。

第四，可以控制写入权限：

每个 Agent 只写自己负责的字段：

```text
订单 Agent → order_status、signed_days

售后 Agent → refund_eligible、required_material、matched_rule

风控 Agent → risk_level、risk_reason

主客服 Agent → next_action、final_response
```

这里还会引出另一个问题：

**什么时候要共享上下文，什么时候让 Agent 保持独立上下文？**

不要默认共享所有上下文。共享越多，越容易互相污染。

比如订单 Agent 只需要知道订单 ID、用户身份、要查哪些订单字段。

它不一定需要看到用户整段情绪化表达：

```text
你们太离谱了，再不退款我就投诉。
```

订单 Agent 本来只应该查事实，看到太多情绪信息，反而可能开始解释、安抚，输出不属于它职责范围的内容。

所以我会把上下文分成两类。

第一类是共享上下文，放所有 Agent 都需要对齐的事实：

```text
order_id
user_id
当前意图
已确认事实
关键风险标记
下一步动作
```

第二类是独立上下文，放某个 Agent 完成自己职责时才需要的局部信息：

比如：

```text
订单 Agent 只关心订单字段和订单系统返回。

售后 Agent 只关心商品类型、签收时间、问题类型、售后规则。

风控 Agent 只关心退款历史、投诉记录、异常行为。

主客服 Agent 才需要完整对话和用户情绪。
```

Shared State 不是把所有聊天记录复制给所有 Agent。

它更像一张公共白板，只放跨 Agent 决策必须依赖的事实。每个 Agent 仍然保留自己的独立上下文。

订单 Agent 不会被用户情绪带偏。

风控 Agent 不会被客服话术干扰。

售后 Agent 不会因为主 Agent 想安抚用户，就放宽规则。

判断原则：

```text
影响多个 Agent 决策的事实 → 共享上下文

只服务某个 Agent 自己判断的信息 → 独立上下文
```

共享上下文解决协作问题，独立上下文保证职责边界。

所以多 Agent 通信不是让 Agent 互相「聊明白」，而是通过 Message 分工，通过 State 对齐事实。

## 写在最后

Agent 架构第一步，先定三件事：责任边界、协作方式、状态流转。

这三件事定清楚，再加 Agent。否则 Agent 越多，问题越难排查。

关联案例：[Orchestrator 案例：订单 3 天没状态，用户要退款时 Agent 怎么流转]({% post_url 2026-06-10-agent-customer-service-refund-case %})
