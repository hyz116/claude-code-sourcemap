# Ground Truth via Tools（工具约束即事实来源）

> Claude Code 里有一条**观察到的一致性模式**：消息历史中的事实只能来自 `tool_result` 块——模型物理上无法伪造这些块。本页描述这条模式 + 它涉及的几个机制。
>
> **⚠ 认知边界**：本页起初标榜这是"防幻觉根哲学"+"所有其他机制都是这条的应用"——这是**过度归纳**。重新审视后：
> 1. 模型只是不能伪造**消息历史里的 tool_result**，但完全可以在 assistant text 里说假话——后者由 [[false-claims-bidirectional]] 处理，是正交问题，不是本概念派生
> 2. 防幻觉簇 7 篇里只有 2-3 篇真正是这条模式的应用（read-before-write、task-notification-injection），其他要么互补要么正交，[[predictable-hallucination-hardcode]] 甚至**部分矛盾**（client 端悄悄修改 input/output 违反"事实不可篡改"精神）
> 3. 没有证据 Anthropic 把这条作为"哲学"——更可能是多个独立工程决策**碰巧**都符合"消息历史不可伪造"。我把它包装成统一哲学是事后归纳
>
> 把本页当作"消息历史角色不可伪造"这一**单一模式**的描述，不要当作 Claude Code 防幻覺的元理论。

## 核心机制

```
模型输出
   │
   ├─ assistant 角色文本 ──► 流回用户（用户是真值仲裁者）
   │
   └─ tool_use block ──► 系统执行工具 ──► tool_result 注入消息历史 ──► 进入下轮上下文
                                          │
                                          ├─ 模型无法跳过执行
                                          ├─ 模型无法修改返回值
                                          └─ 历史中所有 tool_result 都是系统实际执行产物
```

**架构约束**（OBS）：模型只能输出 assistant 角色的消息（API 层面强制）。`user` / `system` / `tool_result` 三种角色都由系统注入——**模型物理上无法在消息历史里伪造这些角色的内容**。

⚠ 注意"无法伪造事实"用词过强。准确说法：模型能在 assistant text 里**声称**任何事实（"我刚跑了测试都通过了"是经典 false claim），但**不能**在消息历史里插入伪造的 tool_result / user message / system message。前者由 [[false-claims-bidirectional]] 治理，本概念只覆盖后者。

## 在 Claude Code 中的体现

### 唯一事实来源：tool_result 通道

`query.ts` 主循环（`src/query.ts:572` 周边）的结构强制：

```
assistant message (含 tool_use)
  → 系统执行 tool
  → toolResults.push(...)
  → state.messages = [...messagesForQuery, ...assistantMessages, ...toolResults]
  → 下一轮 query
```

模型唯一能"声称事实"的方式是发起 `tool_use`——而 tool_use 的输出由系统填回，模型不接触。

### 角色不可伪造的物理实现

| "事实" | 唯一来源工具 | 防伪造机制 |
|---|---|---|
| 文件内容 | `Read` | `readFileState` 缓存追踪谁读过什么；未读不能编辑（[[read-before-write]]） |
| 测试/命令输出 | `Bash` 的 stdout/stderr | 输出由系统捕获写入 tool_result，模型无法注入 |
| 远程内容 | `WebFetch` / `WebSearch` | 抓取由系统执行；模型不能猜 URL 内容（prompts.ts:183 显式禁止） |
| 子 Agent 结果 | `task-notification` XML | 用 **user 角色**注入主对话——模型只能写 assistant，物理无法伪造（[[task-notification-injection]]） |
| 文件状态（修改时间）| `getFileModificationTime` | stale read 时间戳 + 内容双重比对，防"基于过时记忆编辑" |
| MCP 外部数据 | MCP 工具调用 | 经 MCP transport 物理隔离 |

### 大输出"必须重新 Read"模式

`tools/Tool.ts` 里 `maxResultSizeChars` 约束：超大 tool_result 不直接喂给模型，而是溢出到磁盘 + 给模型一个文件路径预览。模型如果想要全文，必须再调一次 Read。

⚠ **主要动机不是防幻觉**——大概率是 token 成本 / 上下文窗口压力 / 流式延迟。"防止模型声称记得"是**附带效果**，不是设计初衷。把它放在本概念页里只是因为这个 side effect 跟"事实必须从 tool_result 来"对齐。

参考 [[result-delivery-guarantee]] 第 3 层（磁盘溢出）。

## 跟外部 Verifier 路线的关系

leto-ai 的"主动追溯验证链路"（`/leto-ai/.../retroactive-verification/architecture.md` §8.4）核心约束：

> **追溯验证的价值完全取决于验证路径与原始执行路径的独立程度。**

⚠ **本页起初把这描述为"对偶解法"——这个 framing 是我编的，过度对立化**。准确说法：两者都用多层防御，但**重心不同**：

| 维度 | leto-ai | Claude Code |
|---|---|---|
| Layer 1 前置 | 有（§2.1 环境/权限/约束预检） | 有（[[bash-command-classification]] / [[tool-args-prevalidation]]）|
| Layer 2 后置 | **重点投入**（独立 Verifier Agent，§5）| 有但相对弱（[[post-generation-verification-channels]] 的 Verifier worker）|
| Layer 3 巡检 | 有（§2.1 持续巡检） | 几乎没有（CLI 单会话） |
| 消息历史不可伪造 | 不强制（业务系统对 Agent 输出的信任度可调）| 强制（API 角色约束）—— 本概念 |

差异**不是"对偶"**，是**应用场景导致的侧重差异**：业务系统失败代价高 → 事后强 verifier；开发工具任务异质性高 → 事前强输入门控 + tool 可信性。两个团队都用了所有 4 层，只是分配权重不同。

**本概念在 Claude Code 这套防御里的位置**：是 Layer 2 跟 Layer 1 之间的一个**架构副产品**——API 角色限制让消息历史不可伪造。这只覆盖"消息历史"这一个面，不覆盖 prompt 层 / 输出汇报层的幻觉。

## 设计原则

> ⚠ 下面这些"原则"是**事后从代码归纳的口号**，不是 Anthropic 文档里的明确指引。每条左边的标签：**OBS** 表示能从代码直接验证的事实；**NARR** 表示我加的格言式包装。带 NARR 标签的条目作为思维启发可以读，但不要当作"Claude Code 设计师这样想的"证据。

| 原则 | 类型 | 含义 |
|---|---|---|
| **模型不是真值，工具是** | NARR | 我编的格言。**OBS 部分**是：消息历史里的 tool_result 由系统填，模型不接触；**NARR 部分**是把它升级成"任何断言必须由 tool_result 证实"——assistant text 里的断言是另一回事 |
| **物理约束优于诚信约束** | INF | 一个跨多个机制的真实模式（read-before-write / role不可伪造），但"极致体现"是我的话；某些机制（[[predictable-hallucination-hardcode]] 的 desanitize）反而违反这条 |
| **角色不可伪造是免费的强约束** | OBS+NARR | 角色不可伪造是 OBS（API 设计如此）；"免费 / 强 / 零成本" 这些价值判断是我加的 framing |
| **大输出必经"重新 Read"** | OBS+INF | 行为是 OBS（maxResultSizeChars 存在）；动机"防声称记得"是 INF——更可能的真因是 token 成本 |
| **历史可审计** | OBS | 消息历史包含所有 tool_result——这是真的 |
| **工具失败也是事实** | OBS+NARR | error tool_result 进历史是 OBS；"也是事实"的格言化是 NARR |

## 进一步追问的钩子

1. **工具实现 bug 怎么办** — 当 `Read` 自己有 bug 返回错误内容，整个事实链就被污染了。Claude Code 没有"双工具交叉验证"机制——这是这条哲学的代价
2. **用户提供的"事实"算不算 ground truth** — 用户在 prompt 里贴的 `"我已经测过了，修复有效"` 是不是事实源？模型应不应基于它跳过验证？看似 yes，但跟 ground-truth-via-tools 形成张力
3. **MCP 工具的可信度链条** — MCP 是用户配置的外部工具，Claude Code 信任它的输出，但 MCP 实现可能不严密。`tool_result` 的可信度边界在哪？
4. **流式工具结果的中间态** — 长跑工具的中途流式输出（StreamEvent）算事实吗？还是只有最终 tool_result 算？

## 关联

- ⚠ 本页**不是**防幻觉的"顶层哲学"——是防御链里"消息历史不可伪造"这一**单一面**的描述
- **真正应用本模式的概念**：[[read-before-write]]（工具门控让"凭记忆编辑"在历史层面被卡）、[[task-notification-injection]]（user 角色不可伪造）
- **互补而非派生**：[[result-delivery-guarantee]]（确保 tool_result 真的送达，是另一面问题）
- **正交问题**：[[false-claims-bidirectional]]（治 assistant text 里的虚假声明，本概念的物理约束完全不覆盖这块）
- **部分矛盾**：[[predictable-hallucination-hardcode]]（client 端悄悄修 input/output，违反"事实不可篡改"精神——但通过修复**已知偏差**而非伪造，可以辩称是边界外）
- **跟 leto-ai 的关系**：不是"对偶"，是同一多层防御里的**侧重差异**——leto-ai 重 Layer 2/3（业务失败代价），Claude Code 重 Layer 1 + API 角色约束（输入异质性）
- 相关实体：`src/query.ts`（主循环 tool_use → tool_result 强制结构）、`src/tools/Tool.ts`（`maxResultSizeChars`）、`src/services/SessionMemory/`（事实链跨会话持久化）
- 综合分析：[7.ANTI_HALLUCINATION.md](../7.ANTI_HALLUCINATION.md)（防幻觉整体架构）、[1.CORE_LOOP.md](../1.CORE_LOOP.md)（query.ts 循环结构）
