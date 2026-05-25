---
layout: post
title: 为什么向量检索不能直接喂给大模型？——从信息瓶颈到 RAG 工程实践
date: 2026-05-25
tags: RAG LLM 检索 工程
---

RAG 系统的第一版往往是这样：文档切块 → embedding → 向量检索 top-5 → 拼进 prompt → 大模型回答。本地测几个问题觉得"挺准的"，上线就崩。这篇文章不讲"有什么坑要避"，而是从信息论的约束出发，推导出为什么这套做法必然出问题，然后给出可落地的工程方案。

<!--more-->

## 一、一个"合理"的做法，为什么上线就错

先定义问题。一个朴素 RAG 检索长这样：

```python
def answer_v1(question):
    q_vec = embed(question)
    chunks = vector_db.search(q_vec, top_k=5)
    context = "\n\n".join(c["text"] for c in chunks)
    return call_llm(
        f"根据以下资料回答问题。\n资料:{context}\n问题:{question}")
```

逻辑直白到无可挑剔：把问题转向量，找最接近的 5 段，拼起来喂给模型。上线后四类问题：

1. **答案不对**——top-5 里没有含答案的那段，模型只能瞎编
2. **top-5 充满"像而没用"的段落**——词面重叠多但没回答问题
3. **K 调大反而更差**——从 5 调到 20，答案质量不升反降
4. **结果不稳定**——同一问题有时对有时错

表面原因各有不同，但根子是一个：**认为向量检索返回的 top-K 就是精确的相关性排序**。这个认知是错的，而且错在信息论层面——不是模型不够好，是 bi-encoder 的架构决定了它必然丢失精度。

## 二、信息瓶颈：bi-encoder 的理论上限

### 2.1 有损压缩的必然性

Bi-encoder 的工作方式：把一段文本（无论是问题还是文档）编码成一个固定长度的向量。这个向量长度通常只有 768 或 1024 维——不管原文是 50 个字还是 5000 个字，都压缩到同一个固定大小。

这里有一个根本性的信息论约束：**压缩必然导致信息丢失**。一篇 2000 字的文档包含的信息量远大于一个 768 维向量能承载的，编码过程必须"丢掉细节、只留大意"。而"这段文档有没有回答这个具体问题"，恰恰属于细节层面的判断。

两段文档都可能被编码成"关于会员功能"的向量，但一段讲开通步骤、一段讲权益介绍——bi-encoder 无法区分，因为细节已经在压缩时丢掉了。

### 2.2 预先编码 vs 现场阅读

更深一层的差异在于编码时机：

```
Bi-encoder：
  文档向量在离线时预先算好 → 压缩时不知道用户会问什么
  → 只能保留"大意"，丢失细节
  → 来任何问题都只能拿这个压缩后的向量去比

Cross-encoder：
  等查询来了才阅读 → 把问题和文档拼在一起整体看
  → 问题中的每个词可以和文档中的每个词交互
  → 可以判断"这段有没有正面回答这个问题"
```

用招聘来类比：bi-encoder 像 HR 用关键词筛简历——"精通 Java、5 年经验"这些标签能对上，但分不清"这个人真的能干活"还是"简历里堆了一堆热门词"。cross-encoder 像面试官一对一深谈——结合具体岗位要求来判断，慢，但是准。

### 2.3 一个简单的实验验证

你可以在自己的数据上验证这个信息瓶颈。取一批"问法不同但召回同一段文档"的 query，观察向量检索的排序稳定性。你会发现，对于同一段文档，稍微改变问法（"怎么开通" vs "开通步骤" vs "如何开始使用"），向量检索给出的距离分数波动很大——这就是细节丢失的表现：两个意思不同但氛围相近的短语，在压缩后的向量空间里可能比真正匹配的短语更近。

## 三、工程解法：两阶段检索的完整落地

理解了信息瓶颈，解法就清晰了：**不要让 bi-encoder 独自承担"选出最终资料"的重任。** 把检索拆成两个阶段，各司其职。

### 3.1 系统架构

```
用户问题
  │
  ├─ [可选] 查询改写 → 多条查询分别召回 → 合并去重
  │
  ├─ 阶段一：召回（Recall）
  │    工具：向量检索（bi-encoder）
  │    目标：要全，不漏
  │    K：100~200
  │    职责：把"亿"量级收敛到"百"
  │
  ├─ 阶段二：精排（Rerank）
  │    工具：cross-encoder
  │    目标：要准，排序
  │    职责：从"百"里精挑出最相关的几条
  │
  ├─ 相关性阈值截断 → 分数不够的不要
  │
  └─ 上下文预算装填 → 按 token 上限装满即止
       │
       └─ 最终资料 + 问题 → 大模型
```

### 3.2 召回阶段：K 怎么选

```python
def recall(question, recall_k=150):
    q_vec = embed(question)
    candidates = vector_db.search(q_vec, top_k=recall_k)
    return candidates
```

K 的选择是一个权衡，核心原则是：**宁可宽，不可漏**。推荐做法：

- **知识库 < 10 万条**：K = 50~100
- **知识库 10 万~100 万条**：K = 100~200
- **知识库 > 100 万条**：K = 200~500（配合分片检索）

实际工程中可以这样测试：抽样一批 query，人工标注正确答案所在文档，然后看不同 K 值下的召回率（Recall@K）。画出曲线，找到"召回率开始饱和"的那个拐点——再增大 K 召回率不再明显提升，那就是合适的值。

### 3.3 精排阶段：cross-encoder 的工程集成

```python
from sentence_transformers import CrossEncoder

# 模块级，只加载一次 —— 每个请求都 new 一个模型是致命错误
RERANKER = CrossEncoder("BAAI/bge-reranker-v2-m3")

def rerank(question, candidates, batch_size=32):
    pairs = [(question, c["text"]) for c in candidates]
    scores = RERANKER.predict(pairs, batch_size=batch_size)
    for c, s in zip(candidates, scores):
        c["rerank_score"] = float(s)
    return sorted(candidates, key=lambda c: c["rerank_score"], reverse=True)
```

**工程要点：**

1. **批量推理**：不要逐条调用 predict——每条都是独立的前向传播，100 条就是 100 次，延迟放大几十倍。一次性传入所有 pair，cross-encoder 在内部并行计算。

2. **模型常驻**：进程启动时加载一次，所有请求复用。cross-encoder 模型权重加载需要几秒，每个请求都加载是灾难。

3. **GPU/CPU 决策**：
   - 候选量 < 100、QPS < 10 → CPU 够用，延迟 100~300ms
   - 候选量 > 100 或 QPS > 50 → 必须上 GPU，否则延迟不可接受
   - 使用 `model.to('cuda')` 或 `model.half()` 可进一步压延迟

### 3.4 阈值截断：宁可不说，不可胡说

重排之后不是无脑取 top-N，而是按分数截断：

```python
def select_by_threshold(ranked, score_threshold=0.3, max_n=5):
    picked = []
    for c in ranked:
        if c["rerank_score"] < score_threshold:
            break  # 已按降序排列，后面的分数只会更低
        picked.append(c)
        if len(picked) >= max_n:
            break
    return picked
```

**阈值如何校准：**

1. 收集 200~500 条"问题-文档"pair
2. 人工标注每条是否"真正回答了问题"（二分类）
3. 在不同阈值下计算 precision/recall
4. 选择 F1 最高的阈值

不同模型、不同领域的阈值差异很大。BGE-reranker-v2-m3 在中文知识库上通常 0.2~0.4 是合理范围，但必须用自己的数据校准。

### 3.5 上下文预算：多即是噪声

```python
def build_context(picked, token_budget=2000):
    parts, used = [], 0
    for c in picked:
        t = count_tokens(c["text"])
        if used + t > token_budget:
            break
        parts.append(c["text"])
        used += t
    return "\n\n".join(parts)
```

这里有一个常见的直觉错误：模型答不好 → 给它的资料太少 → 调大 K。但大模型读上下文不是"从一堆资料里精准挑出有用的那段"——它会受到所有内容的综合影响。多塞一段次相关的内容，不是多一份"参考"，而是多一份"噪声"。K 调大反而更差的真相就是：有用的信息被淹没了。

### 3.6 完整的 v2 实现

```python
def answer_v2(question):
    # 召回
    candidates = recall(question, recall_k=150)
    if not candidates:
        return "知识库中没有找到相关资料。"
    
    # 精排
    ranked = rerank(question, candidates)
    
    # 阈值截断
    picked = select_by_threshold(ranked, score_threshold=0.3)
    if not picked:
        return "知识库中没有足够相关的资料来回答这个问题。"
    
    # 预算装填
    context = build_context(picked, token_budget=2000)
    
    return call_llm(
        f"根据以下资料回答问题。\n资料:{context}\n问题:{question}")
```

同样的 top-5，v1 取的是向量检索的前 5（粗筛结果），v2 取的是 150 条经 cross-encoder 精排后的前 5——天差地别。

## 四、Cross-encoder 选型与性能实战

### 4.1 主流 cross-encoder 对比

| 模型 | 参数量 | 推理速度（CPU） | 中文效果 | 适用场景 |
|------|--------|----------------|----------|---------|
| BAAI/bge-reranker-v2-m3 | 568M | 中等 | 优秀 | 通用推荐，平衡之选 |
| BAAI/bge-reranker-base | 278M | 快 | 良好 | 低资源场景 |
| BAAI/bge-reranker-large | 560M | 中等 | 优秀 | 精度优先 |
| Cohere rerank v3 | API | N/A | 良好 | 不想自部署 |
| Jina Reranker v2 | 278M | 快 | 良好 | 多语言场景 |

### 4.2 延迟优化策略

线上实测数据（100 条候选，CPU 推理）：

| 优化 | 延迟 | 备注 |
|------|------|------|
| 逐条推理（无优化） | ~10s | 不可接受 |
| 批量推理 batch=32 | ~800ms | 主要优化手段 |
| 批量推理 + FP16 | ~500ms | 精度几乎无损 |
| batch=32 + GPU | ~80ms | 高 QPS 必需 |

### 4.3 选型决策树

```
需要自部署？
  ├─ 否 → 用 Cohere/Jina API
  └─ 是 → QPS > 50？
       ├─ 是 → GPU + bge-reranker-base（batch=64）
       └─ 否 → CPU + bge-reranker-v2-m3（batch=32）
```

## 五、进阶工程实践

### 5.1 查询改写：堵住召回的最后漏洞

召回漏掉正确答案，往往是"问法和写法对不上"。用户说"这功能怎么开"，文档里写的是"启用该选项的步骤"——原问题可能召不到。

```python
def expand_query(question):
    rewrites = call_small_llm(
        f"把下面的问题改写成 3 个意思相同但说法不同的查询，"
        f"每行一个，不要解释：\n{question}"
    ).strip().split("\n")
    return [question] + [r.strip() for r in rewrites if r.strip()]

def multi_recall(question, recall_k=150):
    seen, pool = set(), []
    for q in expand_query(question):
        for c in vector_db.search(embed(q), top_k=recall_k):
            if c["id"] not in seen:
                seen.add(c["id"])
                pool.append(c)
    return pool
```

注意：查询改写本身会调用一次语言模型，增加延迟，所以改写模型要够小。实测 qwen2.5-1.5b 级别就能产生有效的改写，延迟控制在 100ms 内。

### 5.2 Chunk 策略与重排的相互影响

文档切块大小直接影响重排效果：

- **切太大（>1000 tokens）**：一块里混了相关和不相关的内容，cross-encoder 只能给一个整块分数，无法精确定位到回答位置
- **切太小（<100 tokens）**：可能把一个完整答案拆散，哪一段都缺少上下文，cross-encoder 判断不出是否相关

建议：
- 基础 chunk 大小：256~512 tokens
- 配合滑动窗口重叠（overlap）：每个 chunk 与前一个重叠 10~20%，减少"刚好切掉答案"的概率
- 重排后取 top-N 个 chunk，必要时做二次合并

### 5.3 混合检索：当纯语义不够用时

某些场景下向量检索效果很差：
- **专有名词/缩写**（"这个 bug 的 RCA 是什么"）——语义编码不擅长精确匹配
- **代码/版本号**（"v3.2.1 的 API 变更"）——向量检索对精确数字不敏感
- **中英文混杂**——部分 embedding 模型跨语言能力有限

解决方案：**混合检索**——向量检索 + 关键词检索（BM25）并行，结果按加权合并。

```python
def hybrid_recall(question, recall_k=100, alpha=0.5):
    # 向量检索
    q_vec = embed(question)
    dense_results = vector_db.search(q_vec, top_k=recall_k)
    
    # 关键词检索
    sparse_results = bm25_search(question, top_k=recall_k)
    
    # 加权融合（按 rank 加权）
    combined = {}
    for rank, doc in enumerate(dense_results):
        combined[doc["id"]] = combined.get(doc["id"], 0) + (1 - alpha) / (rank + 60)
    for rank, doc in enumerate(sparse_results):
        combined[doc["id"]] = combined.get(doc["id"], 0) + alpha / (rank + 60)
    
    sorted_ids = sorted(combined, key=combined.get, reverse=True)
    return [d for d in dense_results + sparse_results 
            if d["id"] in set(sorted_ids[:recall_k])]
```

`alpha` 控制稠密/稀疏的权重，推荐从 0.3~0.5 开始调。在实际项目中，混合检索通常能在纯向量检索基础上提升 5~15% 的召回率。

### 5.4 延迟预算的分配决策

一个端到端 RAG 查询的时间分布（实测数据）：

| 环节 | 耗时 | 占比 |
|------|------|------|
| 向量检索（召回 150 条） | ~30ms | 3% |
| 查询改写（小模型） | ~100ms | 10% |
| Cross-encoder 精排（batch，CPU） | ~500ms | 52% |
| 大模型推理 | ~300ms | 31% |
| 其他（预处理、阈值等） | ~30ms | 3% |

**关键洞察**：精排是最大的延迟瓶颈。优化的杠杆在于：
- cross-encoder batch size 调优（不是越大越好，会线性增加延迟）
- 精排候选量不是越多越好——召回 100 条时精排 100 条，召回 500 条你还是要精排 500 条
- 如果延迟要求 < 1s，考虑减小 recall_k 到 50，或上 GPU

## 六、总结

回到开头的问题：为什么向量检索不能直接喂给大模型？

**根本原因**：bi-encoder 的信息瓶颈——固定长度向量的有损压缩决定了它必然丢失细节，而"有没有回答这个具体问题"恰恰需要细节层面的判断。这不是模型不够好，而是架构的上限。

**工程解法不是单一的"加个重排"，而是一套流水线：**

1. **召回放宽**：K 设到 100~200，宁可宽不可漏
2. **重排接力**：cross-encoder 精排，批量推理控制延迟
3. **阈值截断**：分数不够的不要，宁可不说不可胡说
4. **预算装填**：按 token 上限装填，控噪声
5. **查询改写**：弥补问法和写法的 gap
6. **混合检索**：语义 + 关键词双通道

最后一句：RAG 检索的可靠，不是靠一个模型神通广大，而是让每个环节做自己擅长的事，接力完成。
