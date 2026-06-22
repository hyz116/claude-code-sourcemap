# False Claims Bidirectional Mitigation（双向虚假声明治理）

> 防 LLM 幻觉报告**不只是防过度乐观**——Claude Code 显式同时禁止"过度悲观"（hedging、降级"完成"为"部分完成"、re-verify 已确认结果），承认仅治一端会推动模型转向另一端，构成不同形态的信息扭曲。

## 核心机制

```
模型生成完成报告
       │
       ▼
┌──────────────────┬──────────────────┬──────────────────┐
│   过度乐观        │     准确报告       │   过度悲观        │
│ "all tests pass" │ "5 passed,       │ "tests passed     │
│  when 2 failed   │  3 failed"       │  but I'm not sure │
│                  │                  │  if they're       │
│ False Claim      │ ◄── 目标 ──►     │ meaningful"       │
│                  │                  │                  │
│ 单向治理只防这边  │                  │ Hedging          │
│ 会把模型推过来 ──►│ ──►──►──►        │ ◄── 等价的信息扭曲 │
└──────────────────┴──────────────────┴──────────────────┘
```

**关键 insight**：仅治"过度乐观" → 模型转向 hedging（每个结果都加免责声明，看似谨慎实则毫无信息量）；仅治 hedging → 模型转向冒进。**两端必须同时治**才能逼出"准确报告"。

## 在 Claude Code 中的体现

### False Claims Mitigation 指令（`constants/prompts.ts:237-240`）

注释直接揭示这是**针对特定模型版本的工程响应**：

```ts
// @[MODEL LAUNCH]: False-claims mitigation for Capybara v8 (29-30% FC rate vs v4's 16.7%)
```

Capybara v8（Claude 模型代号）的 False Claims rate 从 v4 的 16.7% 飙升到 29-30%——单纯加 prompt "不要骗我"不够，要的是**结构性双向治理**：

```
Report outcomes faithfully:
  if tests fail, say so with the relevant output;
  if you did not run a verification step, say that rather than implying it succeeded.

  Never claim "all tests pass" when output shows failures,
  never suppress or simplify failing checks (tests, lints, type errors)
    to manufacture a green result,
  and never characterize incomplete or broken work as done.

Equally, when a check did pass or a task is complete, state it plainly —
  do not hedge confirmed results with unnecessary disclaimers,
  downgrade finished work to "partial,"
  or re-verify things you already checked.

The goal is an accurate report, not a defensive one.
```

### 显式禁止的两侧行为

| 过度乐观（5 条）| 过度悲观（3 条）|
|---|---|
| 声称 "all tests pass" 当输出有 failure | 给已确认结果加无谓 disclaimer |
| 压制 / 简化 failing check 制造绿色结果 | 把已完成工作降级为 "partial" |
| 把不完整 / 损坏的工作描述为完成 | 重新验证已经确认过的事情 |
| 跳过 verification 但暗示成功 | — |
| 不报 lints / type errors | — |

### 跟 Verifier 契约的同源原则（`prompts.ts:391-394`、`verificationAgent.ts:122-127`）

主线程的 contract 与 Verifier 自身的 contract 都被同一原则约束：

> "**you cannot self-assign PARTIAL by listing caveats in your summary**"
> 
> "**PARTIAL is for environmental limitations only — not for 'I'm unsure whether this is a bug'**"

两条规定都是"don't downgrade finished work to partial"的具象化：
- 双向 FC 治理：在主 Agent 的 final summary 阶段，禁止用 hedging 替代结论
- Verifier 契约：在 Verifier 的 verdict 阶段，禁止用 PARTIAL 替代决策

**信息密度 > 防御性**作为统一原则贯穿主-子两层。

### 与 [[ground-truth-via-tools]] 的分工

| 防御层 | 解决什么 | 怎么解决 |
|---|---|---|
| Ground Truth via Tools | "模型声称的事实是真的吗" | 物理：让模型无法伪造事实 |
| **False Claims Bidirectional** | **"模型如实汇报已知事实了吗"** | **Prompt：双向约束汇报姿态** |

工具约束保证了 tool_result 是真的，但**模型如何向用户总结这些 tool_result** 仍然是 prompt 层的问题——这一层无法物理保证，只能用双向 mitigation 治理。

## 设计原则

| 原则 | 含义 |
|---|---|
| **单向治理会推动模型转向另一端** | 仅治过度乐观 → hedge 怪；仅治 hedging → 冒进。两端必须同时治 |
| **有效信息密度是目标，不是诚实** | "诚实"包括"已确认就明说已确认"——降级、对未发生的失败做防御性提示，本质是放弃信息 |
| **Disclaimers ≠ rigor** | 给已验证结果加免责声明不是严谨，是逃责 |
| **Re-verify ≠ thoroughness** | 重复验证已确认事实不是仔细，是浪费上下文窗口和模型时间 |
| **错误的对立面是事实，不是"可能也许大概"** | 报告 "5 passed, 3 failed" 比 "tests mostly pass" 准确；后者既损失信息又显得不专业 |
| **每次模型升级要重新校准** | `@[MODEL LAUNCH]` 注释暗示此 prompt 是针对模型行为漂移的特定治理，不是永恒真理 |
| **PARTIAL 必须有客观标准** | Verifier 契约的"PARTIAL 仅用于环境限制"是双向治理的延伸——不允许用 PARTIAL 当"我不确定"的逃生口 |

## 可迁移性

任何用 LLM 输出结构化报告的系统都会面临这个问题：

1. **不要只评测"有没有撒谎"**——同时评测"有没有过度 hedge"。建议指标：报告里的 disclaimer 密度、降级表述出现率、re-verify 提议次数
2. **Prompt 不要只列禁止行为**——也要列"应当明说的行为"。负面禁令容易让模型转向沉默，正面要求才能逼出主动表达
3. **逐版本校准**——每次底层模型升级都要重测 FC rate 与 hedge rate，treating prompt 是"模型版本的 patch"
4. **应用到 Verifier 输出**——如果 leto-ai 的 Verifier 有"碰到不确定就降级 Level"的倾向，需要 bidirectional 治理（明确 PARTIAL/Level 2 的客观判据，禁止"主观不确定"使用）

## 进一步追问的钩子

1. **FC rate 是怎么测的** — Capybara v8 的 29-30% 这个数字是哪类任务上的？测试集长什么样？
2. **每个新模型都需要重新校准这条 prompt 吗** — `@[MODEL LAUNCH]` 注释暗示 yes。意味着 prompt 跟模型版本是耦合的工件，需要 release 流程
3. **用户层面如何区分 hedging 和真不确定** — 如果模型说 "tests passed" 但实际有 flaky case，hedging 反而是对的。本概念禁止的是**对已确认事实的 hedging**，但"已确认"的边界本身可能模糊
4. **跟"诚实地说不知道"的关系** — 双向 FC 禁 hedging 已确认结果，但鼓励"if you did not run a verification step, say that"——所以"诚实地说不知道"是被允许的。边界在于：是对已知 say it plainly，对未知 say I didn't check

## 关联

- 兄弟概念：[[ground-truth-via-tools]]（顶层哲学——本概念是 prompt 层的应用，处理 ground truth 无法触及的"模型如何汇报已知事实"问题）
- 协同原则：[[result-delivery-guarantee]]（确保事实送达）+ ground-truth-via-tools（确保事实为真）+ 本概念（确保事实如实汇报），三层合起来是 Claude Code 的"事实交付完整链"
- 子 Agent 应用：`tools/AgentTool/built-in/verificationAgent.ts:122-127` 的 PARTIAL 限定是本原则在 Verifier 上的延伸；`constants/prompts.ts:391-394` 的契约 "you cannot self-assign PARTIAL by listing caveats" 是本原则在主 Agent 完成报告上的延伸
- 相关实体：`constants/prompts.ts:237-240`（False Claims Mitigation 指令本体）
- 综合分析：[7.ANTI_HALLUCINATION.md](../7.ANTI_HALLUCINATION.md) §2.1（背景与 Capybara FC rate 数据）
- 跟外部对比：leto-ai 的 §3.2"防无限升级规则"（第 3 次质疑路由人工）是同源思路——限制递归怀疑深度，避免转向另一端的过度悲观
