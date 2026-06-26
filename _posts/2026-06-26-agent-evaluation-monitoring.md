---
layout: post
title: 在做 Agent 应用时的思考（三）：评测不是看回答好不好
date: 2026-06-26 12:00:00 +0800
tags: AI Agent 评测 监控
---

做 Agent 应用时，很多人会把评测理解成：

```text
这个回答看起来对不对？
```

但真正上线以后会发现，这个问题太小了。

Agent 不是普通问答。

它中间会识别意图、规划步骤、调用工具、读写 State、判断风险，最后才生成回复。

所以评测 Agent，不能只看最终回答。

要看整条链路。

![Agent 评测不能只看最终回复](/assets/agent-evaluation-monitoring-illustrations/01-evaluate-the-chain.png)

<!--more-->

## 一、先评任务是否完成

Agent 最核心的指标不是回答流畅度，而是任务是否完成。

比如智能客服里，用户说：

```text
订单 3 天没状态了，订单 ID 是 xxx，我要退款。
```

一个看起来不错的回复可能是：

```text
很抱歉给你带来不便，我们会尽快为你处理。
```

但这不一定算成功。

因为这个任务真正需要完成的是：

```text
识别退款诉求
  ↓
查询订单状态
  ↓
查询物流状态
  ↓
判断是否物流异常
  ↓
判断是否可以退款
  ↓
创建工单或转人工
  ↓
给用户明确回复
```

如果 Agent 没查订单，没查物流，只是说了一句安抚话，那就是失败。

所以第一层指标应该是：

```text
Task Success Rate = 成功完成任务数 / 总任务数
```

每类任务都要定义清楚「成功」是什么。

查订单成功，是返回正确订单状态。

退款场景成功，是正确判断能不能退款，以及走到正确动作。

高风险场景成功，是没有越权承诺，必要时转人工。

缺信息场景成功，是追问缺失字段，而不是瞎猜。

## 二、再评中间过程是否正确

Agent 经常会出现一种情况：

最终回复看起来还行，但中间过程是错的。

比如用户问：

```text
帮我查一下订单 xxx 到哪了。
```

正确链路应该是：

```text
intent = order_status_query
tool = query_order
params = { order_id: xxx }
next_action = reply_order_status
```

如果 Agent 没调用工具，直接编了一个物流状态，即使话术自然，也要算失败。

所以评测要拆到过程里：

```text
意图识别是否正确
规划步骤是否合理
工具是否选对
参数是否传对
工具结果是否被正确使用
State 是否写对
是否进入正确分支
最终回复是否遵守 State
```

这也是为什么项目里要接 Trace。

没有 Trace，只能看到最后一句话。

有 Trace，才能看到 Agent 到底怎么走到这个答案。

## 三、工具调用要单独评

Agent 应用里，工具调用是最容易出事故的地方。

工具评测至少看四个指标：

```text
Tool Selection Accuracy
Argument Accuracy
Tool Execution Success Rate
Must-not-call Violation Rate
```

翻成人话就是：

```text
该调哪个工具，调对了吗？
参数填对了吗？
工具执行成功了吗？
不该调用的高风险工具，有没有误调用？
```

比如客服系统里：

```text
query_order        查询订单，可以自动调用
query_logistics    查询物流，可以自动调用
check_refund_rule  查售后规则，可以自动调用
create_refund      创建退款，必须确认或审核
cancel_order       取消订单，必须二次确认
```

评测时不能只看 Agent 有没有调用工具。

还要看它有没有误调用高风险工具。

比如用户只是问「能不能退款」，Agent 直接调用 `create_refund`，这就是严重失败。

所以 eval case 里最好写清楚：

```json
{
  "input": "订单三天没状态了，我要退款，订单ID是xxx",
  "expected_tools": ["query_order", "query_logistics", "check_refund_rule"],
  "must_not_call": ["create_refund"],
  "expected_next_action": "create_logistics_delay_ticket"
}
```

这样每次改 Prompt、换模型、改工具 Schema，都能自动跑一遍。

## 四、State 也要评

很多 Agent 的问题不是工具没调，而是 State 写错了。

比如订单 Agent 写入：

```json
{
  "order_status": "shipped",
  "logistics_abnormal": true
}
```

风控 Agent 写入：

```json
{
  "blocked_auto_action": ["create_refund"]
}
```

但最后主 Agent 还是自动退款。

这就不是工具问题，而是 State 没有被正确遵守。

所以要评：

```text
State 是否写对
State 是否被正确读取
关键字段是否被覆盖
高优先级限制是否生效
多轮对话是否丢状态
```

尤其是这些字段，最好单独检查：

```text
intent
missing_fields
tool_results
risk_level
blocked_auto_action
next_action
current_node
```

Agent 的回复必须和 State 对得上。

如果 State 里是：

```text
next_action = create_ticket
```

最终回复却说：

```text
已经为你退款。
```

那就是评测不通过。

## 五、RAG 要评检索，不只评答案

如果 Agent 依赖知识库，还要单独评 RAG 链路。

不能只问「答案对不对」。

要拆成：

```text
正确文档有没有被召回？
Reranker 有没有把它排到前面？
低相关内容有没有被过滤？
上下文有没有塞太多噪声？
答案有没有引用正确来源？
没有资料时有没有拒答？
```

常见指标可以是：

```text
Recall@K
Context Precision
Answer Correctness
Faithfulness
Citation Accuracy
Hallucination Rate
```

实际项目里，我会把 RAG eval case 写成这样：

```json
{
  "question": "会员退款规则是什么？",
  "expected_doc_ids": ["refund_policy_v3"],
  "expected_answer_points": ["7天内", "未使用", "原路退回"],
  "should_refuse": false
}
```

这样可以分开看问题：

```text
召回没召到，是检索问题。
召到了但没用，是上下文组装或模型问题。
资料没有还乱答，是风险控制问题。
```

## 六、风险样本要单独建一组

Agent 上线最怕的不是答不出来，而是乱承诺、乱执行。

所以 eval set 里一定要有风险样本。

比如：

```text
用户要求退款
用户要求赔偿
用户威胁投诉
用户提供别人的订单号
知识库信息冲突
工具连续失败
用户诱导 Agent 越权
```

这类样本主要看：

```text
有没有拒绝越权操作
有没有避免编造结果
有没有低置信度转人工
有没有高风险动作二次确认
有没有遵守 must_not_call
```

可以定义一个很硬的指标：

```text
Risk Violation Rate = 风险违规次数 / 高风险样本数
```

这个指标比回答好不好更重要。

因为一次越权退款、一次错误承诺，可能比十次回答不够优雅更严重。

## 七、项目里一般怎么落

真实项目里，我不会只选一个评测框架。

更常见的是一套组合：

```text
Trace 观测
  ↓
离线评测集
  ↓
自动 Evaluator
  ↓
CI 回归
  ↓
线上监控
  ↓
失败样本回流
```

![线上 Trace 反哺 Eval 集](/assets/agent-evaluation-monitoring-illustrations/02-trace-to-eval-loop.png)

### Trace 观测

如果项目是 LangGraph / LangChain，优先用 LangSmith。

如果想自托管，或者框架不绑定 LangChain，可以用 Langfuse。

如果偏 OpenTelemetry / OpenInference，也可以用 Arize Phoenix。

Trace 里要记录：

```text
用户输入
每个节点输入输出
模型调用
工具调用
工具参数
工具返回
State 变化
最终回复
token / latency / cost
```

### 离线评测

离线评测集可以先从 100 条开始。

不要一开始追求很大。

先覆盖这几类：

```text
高频问题
复杂多轮
高风险动作
边界异常
线上失败样本
```

每条样本尽量标注结构化期望：

```json
{
  "input": "我订单三天没状态了，我要退款",
  "expected_intent": "refund_request",
  "expected_tools": ["query_order", "query_logistics"],
  "must_not_call": ["create_refund"],
  "expected_state": {
    "next_action": "create_ticket"
  }
}
```

### 自动 Evaluator

确定性指标用代码评：

```text
JSON 是否合法
Schema 是否通过
工具是否调用对
must_not_call 是否违规
State 字段是否符合预期
```

主观质量用 LLM-as-judge 评：

```text
回复是否完整
是否解决问题
是否有幻觉
是否遵守客服口径
是否越权承诺
```

框架上可以这样选：

```text
LangSmith / Langfuse / Phoenix：Trace + Dataset + Experiment
Ragas：RAG 指标、工具调用指标
DeepEval：pytest 风格回归测试、Agent 工具评测
自定义脚本：业务规则和 must_not_call 检查
```

### CI 回归

每次改这些东西，都应该跑一遍 eval：

```text
Prompt
模型
工具 Schema
RAG 参数
Graph 节点逻辑
风险规则
```

如果出现下面情况，就不应该直接发布：

```text
任务成功率下降
工具误调用上升
高风险样本违规
RAG 幻觉率上升
延迟或成本超预算
```

## 八、线上监控看什么

上线后不要只看 QPS。

Agent 要看这些：

```text
任务完成率
转人工率
用户追问率
用户满意度
工具调用失败率
高风险拦截率
人工纠错率
平均响应时长
token 成本
```

几个很有用的信号：

```text
用户反复问同一个问题 → 说明没解决
用户主动要求转人工 → 说明 Agent 没兜住
人工修改 Agent 结论 → 说明判断不稳
工具失败集中出现 → 说明后端链路有问题
某类问题 token 暴涨 → 说明流程或上下文设计有问题
```

线上监控最重要的不是看一个漂亮 dashboard。

而是把失败 trace 捞出来，变成新的 eval case。

这样评测集才会越来越贴近真实业务。

## 写在最后

Agent 评测不是给最终回复打一个分。

它要回答的是：

```text
Agent 有没有理解对？
有没有走对流程？
有没有调对工具？
有没有写对 State？
有没有控制风险？
有没有在成本和延迟内完成任务？
```

如果只看最终回复，很多问题会被漂亮话术盖住。

如果把理解、规划、工具、State、风险和回复都评起来，Agent 才有机会从 Demo 变成系统。
