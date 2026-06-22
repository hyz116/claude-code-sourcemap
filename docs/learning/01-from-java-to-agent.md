# Java 电商工程师入门 AI Agent

> 受众：多年 Java 电商开发经验，初入 AI Agent 应用开发。
>
> 目标：基于本 wiki 给出迁移路线——但**不只读 wiki**。wiki 有结构性偏差（见 [insights/3.SELF_CRITIQUE.md](../insights/3.SELF_CRITIQUE.md)），单靠它入门会踩雷。

---

## 0. 先承认两件事

**优势真大**：分布式、降级、限流、幂等、异步交付、监控告警、状态恢复——这些电商工程师每天都在解决的问题，Agent 系统**结构上完全一样**。Claude Code 90% 的"防御机制"对你来说是熟人。

**陌生的只有 10%**：LLM 是**非确定性概率系统**，不是 RPC；prompt 不是 API spec；token 是新的成本/性能维度；幻觉是新的故障模式。

⚠ 这意味着：你**不需要从 0 学 Agent**——你需要把电商经验**翻译**到新场景，并补 10% 的 LLM 特有概念。

---

## 1. 学习路径（不是按 wiki 章节顺序）

### Step 1：先动手（不是先读 wiki）

读 wiki 之前先做一个**最小 Agent loop**（200 行）：

```python
while True:
    response = anthropic.messages.create(model, messages, tools)
    if response.stop_reason == "end_turn": break
    for block in response.content:
        if block.type == "tool_use":
            result = execute(block.name, block.input)
            messages.append({role:"user", content:[{type:"tool_result", ...}]})
```

跑通这个再读 wiki，否则 [1.CORE_LOOP.md](../1.CORE_LOOP.md) 里说的 "tool_use → execute → tool_result → loop" 你**没有手感**。

**电商类比**：你不会先读 Spring Boot 文档再写第一个 Controller——你会先 hello world。

### Step 2：用电商熟人切入（高复用度的页）

这些页 80% 你已经会，只是换了术语：

| Wiki 页 | 电商熟人 | 新增的 10% |
|---|---|---|
| [multi-tier-degradation](../concepts/multi-tier-degradation.md) | 你的接口降级（强→弱→兜底）| LLM 多模型降级（Opus→Sonnet→Haiku）|
| [circuit-breaker](../concepts/circuit-breaker.md) | 你的 Hystrix/Sentinel | Agent 失败计数 |
| [at-most-once-delivery](../concepts/at-most-once-delivery.md) | 订单幂等（CAS/唯一键）| 子 Agent 通知幂等 |
| [result-delivery-guarantee](../concepts/result-delivery-guarantee.md) | 异步消息回执（MQ ACK）| 子 Agent 结果注入主对话 |
| [withhold-then-recover](../concepts/withhold-then-recover.md) | 重试+最终一致 | 错误对模型隐藏 |
| [repair-on-read](../concepts/repair-on-read.md) | DB 字段兼容（旧数据归一化）| LLM 输出修复 |
| [plan-mode-state-machine](../concepts/plan-mode-state-machine.md) | 订单状态机 | Plan/Auto/Default 模式切换 |
| [context-compression-cascade](../concepts/context-compression-cascade.md) | 缓存淘汰（LRU 多级）| token 预算 4 级压缩 |

⚠ **关键反思**：读这些时**反过来用**——"Claude Code 这样做，我电商代码是怎么做的？两边的差异是 LLM 特性导致还是历史遗留？" 反向应用比顺向学习收获大 3 倍。

### Step 3：补 LLM 特有的 10%

这些没有电商对应，要老老实实学：

| Wiki 页 | 学什么 |
|---|---|
| [ground-truth-via-tools](../concepts/ground-truth-via-tools.md) | 为什么 LLM 不能"自己说自己看到了文件"——**消息历史角色物理隔离**这个概念 |
| [read-before-write](../concepts/read-before-write.md) | 为什么不能让 LLM 凭记忆写代码——**幻觉是默认行为，不是 bug** |
| [false-claims-bidirectional](../concepts/false-claims-bidirectional.md) | 双向治理（既禁吹牛也禁过度谨慎）——电商没这问题，订单要么成功要么失败 |
| [tool-args-prevalidation](../concepts/tool-args-prevalidation.md) | 错误消息**面向 LLM 重试**写，不是面向人类——这点很反直觉 |
| [predictable-hallucination-hardcode](../concepts/predictable-hallucination-hardcode.md) | LLM 系统性犯错（如把 `1.5` 写成数字）需 client 端兜底 |

### Step 4：警告——这些章不要直接抄

| Wiki 页 | 为什么不能抄 |
|---|---|
| [prompt-cache-editing](../concepts/prompt-cache-editing.md) | 是 Anthropic 内部 API，外部不可用；且 wiki 自己标了高 SPEC——**作为电商工程师你应该跳过** |
| [coordinator-mode](../concepts/coordinator-mode.md) | Anthropic 自家机器跑的多机协调——你做电商 Agent 用单机或简单 fork-join 就够了，**不要为了酷上 Coordinator** |
| [insights/2.DESIGN_PRINCIPLES.md](../insights/2.DESIGN_PRINCIPLES.md) 的"3 条贯穿线" | 作者自己标了"高 NARR 风险"——**当 idea 看，不当定律抄** |

---

## 2. 你应该带怀疑读的部分

wiki 自己揭示（见 [3.SELF_CRITIQUE.md](../insights/3.SELF_CRITIQUE.md)）：**最 quotable 的部分最不准确**。可信度大致：

```
代码引用 > 概念页"在 Claude Code 中的体现"段 > 概念页"设计原则"表 > DESIGN_PRINCIPLES > 元论点
   ↑高                                                                                         低↓
```

入门阶段**直接读"反 Pattern 速查表"**（[2.DESIGN_PRINCIPLES.md](../insights/2.DESIGN_PRINCIPLES.md) 末尾）——它是具体的代码 anti-pattern，比抽象哲学**对你的实战价值高 5 倍**。

---

## 3. 30 天具体路径

| 周 | 做什么 |
|---|---|
| W1 | 跑通最小 Agent loop（200 行）；读 [1.CORE_LOOP.md](../1.CORE_LOOP.md) 验证你的 mental model |
| W2 | 加 3 个工具（Read / Write / Bash），手动触发幻觉；读 [ground-truth-via-tools](../concepts/ground-truth-via-tools.md) / [read-before-write](../concepts/read-before-write.md) / [predictable-hallucination-hardcode](../concepts/predictable-hallucination-hardcode.md) 看 Anthropic 怎么治 |
| W3 | 给你的 Agent 加"4 级降级 + 监控告警 + 幂等"——**这一周完全用电商技能**，读 [result-delivery-guarantee](../concepts/result-delivery-guarantee.md) / [multi-tier-degradation](../concepts/multi-tier-degradation.md) / [circuit-breaker](../concepts/circuit-breaker.md) 校验 |
| W4 | 选一个具体电商场景（如客服 Agent / 商品描述生成）落地，遇到具体问题回来查 wiki |

---

## 4. 三个反直觉点（电商老兵最容易踩）

### 4.1 Prompt 不是 contract

你写 "请只返回 JSON" 它会**90% 时间返回 JSON**，剩下 10% 给你来 markdown。**不要把 prompt 当 OpenAPI spec**——要么用 tool calling 强制结构化，要么用 Zod 在边界处验证（[tool-args-prevalidation](../concepts/tool-args-prevalidation.md)）。

### 4.2 重试逻辑要 LLM-friendly

你的电商重试是"网络抖动→retry"。LLM 重试是"模型理解错→给它**新的错误信息**让它自己修"。错误消息要包含"为什么错+怎么修"，**对 LLM 写**（[tool-args-prevalidation](../concepts/tool-args-prevalidation.md)）。

### 4.3 状态外部化更激进

电商关键状态进 DB；Agent 系统**对话历史本身就是状态**——要外部化到磁盘+有恢复路径（[plan-mode-state-machine](../concepts/plan-mode-state-machine.md)）。

---

## 5. 最重要的一句

**不要从这个 wiki 开始学 Agent**。这个 wiki 是**深入理解 Claude Code 的副产品**，不是 Agent 入门教程。

先用 [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) 或类似教程跑几个 demo，**有了第一手感**再回来读这个 wiki——你会发现自己能**批判性**地用它，而不是被它的"哲学"包装迷惑（这正是 [3.SELF_CRITIQUE.md](../insights/3.SELF_CRITIQUE.md) 想避免的）。

---

## 待办

- [ ] 跑通最小 Agent loop（W1）
- [ ] 选一个具体电商场景作为练手项目（W4 之前确定）
- [ ] 列出 wiki 里最相关的 5 篇 + 可跳过的 15 篇（依赖具体场景，待定）
