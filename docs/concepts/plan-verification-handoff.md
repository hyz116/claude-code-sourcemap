# Plan Verification Handoff（Plan 实施后的自动化验证交接）

> Plan 模式退出 → 模型实施 → 实施完后**自动提醒模型调 VerifyPlanExecution 工具**做独立验证。一个**状态机驱动 + 节流提醒**的小型设计——把 [[plan-mode-state-machine]] 的退出点连到 [[post-generation-verification-channels]] 的 verifier worker。Ant-only 实验功能，3P 完全 DCE 掉。

## 核心机制

```
ExitPlanMode（用户批准 plan）
      │
      ▼
1. setAppState({ pendingPlanVerification: { plan, started=false, completed=false } })
      │
      ▼
2. 主 Agent 开始实施（normal mode）
      │
      │  每轮 query 前检查 pendingPlanVerification
      │
      ├─► 已 started 或 completed → no-op
      │
      ├─► 未 started + 未到 reminder turn 周期 → no-op
      │
      └─► 未 started + 到 reminder turn 周期（每 10 turn）
              │
              ▼
          注入 verify_plan_reminder attachment：
            "You have completed implementing the plan.
             Please call the 'VerifyPlanExecution' tool directly
             (NOT the Agent tool or an agent)..."
              │
              │  模型读到提醒 → 调 VerifyPlanExecution
              ▼
3. VerifyPlanExecution 工具调用
   - 读 plan 文件（getPlan()）
   - spawn background verification agent
   - 设置 verificationStarted = true
              │
              ▼
4. Verification agent 跑完 → task-notification 回主对话
   - 设置 verificationCompleted = true
   - 后续轮次 reminder 不再触发
```

整个机制是**push-based 提醒**——不强制模型调用，只是在它"忘了"的时候定期戳一下。

## 在 Claude Code 中的体现

### 状态字段（`AppStateStore.ts:411-417`）

```ts
// Pending plan verification state (set when exiting plan mode)
// Used by VerifyPlanExecution tool to trigger background verification
pendingPlanVerification?: {
  plan: string
  verificationStarted: boolean
  verificationCompleted: boolean
}
```

3 个字段构成最小状态机：未开始 → 已启动 → 已完成。每个状态下 reminder 行为不同。

### 触发：Plan 退出时建立状态（`REPL.tsx:3082-3088`）

```ts
const shouldStorePlanForVerification =
  initialMsg.message.planContent &&
  "external" === 'ant' &&                  // ← 字面量比较，DCE 关键
  isEnvTruthy(undefined)

// ...
...(shouldStorePlanForVerification && {
  pendingPlanVerification: {
    plan: initialMsg.message.planContent!,
    verificationStarted: false,
    verificationCompleted: false,
  },
}),
```

**字面量 `"external" === 'ant'` 是个 DCE trick**——在 3P 构建里这是 `"external" === 'ant'` 永远 false，整个分支被 Bun 静态消除；ant 构建里同位置是 `"ant" === 'ant'` 永远 true。**编译时分支收窄**，不依赖 runtime feature flag。

### 节流提醒（`utils/attachments.ts:3894-3929`）

```ts
async function getVerifyPlanReminderAttachment(
  messages: Message[] | undefined,
  toolUseContext: ToolUseContext,
): Promise<Attachment[]> {
  if (
    process.env.USER_TYPE !== 'ant' ||
    !isEnvTruthy(process.env.CLAUDE_CODE_VERIFY_PLAN)
  ) {
    return []
  }

  const pending = appState.pendingPlanVerification
  if (!pending || pending.verificationStarted || pending.verificationCompleted) {
    return []        // 已开始或已完成 → 不提醒
  }

  // Only remind every N turns
  if (messages && messages.length > 0) {
    const turnCount = getVerifyPlanReminderTurnCount(messages)
    if (
      turnCount === 0 ||
      turnCount % VERIFY_PLAN_REMINDER_CONFIG.TURNS_BETWEEN_REMINDERS !== 0
    ) {
      return []
    }
  }

  return [{ type: 'verify_plan_reminder' }]
}
```

`TURNS_BETWEEN_REMINDERS = 10` —— 每 10 轮提醒一次。

`getVerifyPlanReminderTurnCount` 从消息历史里找 `plan_mode_exit` attachment 算"退出 plan 后过了几轮"。**计数器锚点是 plan 退出的具体消息**，不是简单的全局 turn counter——保证了 reminder 只在 plan 实施期间生效。

### Reminder 内容（`utils/messages.ts:4240-4250`）

```ts
case 'verify_plan_reminder': {
  const toolName =
    process.env.CLAUDE_CODE_VERIFY_PLAN === 'true'
      ? 'VerifyPlanExecution'
      : ''
  const content = `You have completed implementing the plan. Please call the "${toolName}" tool directly (NOT the ${AGENT_TOOL_NAME} tool or an agent) to verify that all plan items were completed correctly.`
  return wrapMessagesInSystemReminder([
    createUserMessage({ content, isMeta: true }),
  ])
}
```

注意提醒文本的两个细节：

1. **"directly (NOT the Agent tool or an agent)"** ——明确禁止主 Agent 把验证任务委托给子 Agent。**主 Agent 必须自己调 VerifyPlanExecution 工具**。这跟 [[coordinator-mode]] 里 "synthesize 不能 delegate" 是同源原则
2. **`wrapMessagesInSystemReminder`** ——包装成 system reminder 而非普通 user message，让模型识别这是**系统提示而非用户对话**

### Auto-Mode Allowlist 集成（`classifierDecision.ts:40-44`）

```ts
const VERIFY_PLAN_EXECUTION_TOOL_NAME =
  process.env.USER_TYPE === 'ant'
    ? (require('../../tools/VerifyPlanExecutionTool/constants.js') 
        as typeof import('../../tools/VerifyPlanExecutionTool/constants.js'))
        .VERIFY_PLAN_EXECUTION_TOOL_NAME
    : null
```

VerifyPlanExecution 加入 `isAutoModeAllowlistedTool`——auto mode 下**不需要 user prompt**就能执行。设计意图：plan 验证是**用户已经默认同意的工作流**（用户批准 plan 时同意了"实施 + 验证"全流程），auto mode 下别再问一次。

### 工具本体被 DCE 掉（`tools.ts:91-94`）

```ts
const VerifyPlanExecutionTool =
  process.env.USER_TYPE === 'ant'
    ? require('./tools/VerifyPlanExecutionTool/VerifyPlanExecutionTool.js')
        .VerifyPlanExecutionTool
    : null
```

3P 公开包里 **VerifyPlanExecutionTool 本体不存在**——目录、constants、prompt、UI 全被 Bun DCE 消除。能从 3P 源码看到的只是**散布在各处的 wire-up 代码**：

- 状态字段定义（AppStateStore.ts）
- 状态写入（REPL.tsx，被 `"external" === 'ant'` DCE）
- Reminder attachment 生成（attachments.ts，被 `process.env.USER_TYPE !== 'ant'` runtime gate）
- Auto-mode allowlist 加入（classifierDecision.ts）
- 提醒文本（messages.ts）
- Tool 注册（tools.ts，被 USER_TYPE 检查）

**Tool 实现完全在 3P 源码外**——这是个 ant-only 实验功能。但**接口面**完整保留了，3P 用户从代码上能推断这套 hook 的工作方式。

## 设计模式：State-Machine-Driven Reminder

剥离掉具体业务，这是个**通用 LLM 工作流自动化模式**：

```
某个事件发生（plan 退出 / 任务完成 / 状态变化）
       │
       ▼
建立"待执行"状态（state.pending = { taskRef, started: false, completed: false }）
       │
       ▼
每 N 轮检查状态
   ├─ pending && !started && !completed
   │     → 注入提醒 attachment
   ├─ started && !completed
   │     → no-op（避免提醒"正在做的事"）
   └─ completed
         → no-op（清理或保留）
```

跟其他类似机制对比：

| 模式 | 触发 | 节流 | 关闭条件 |
|---|---|---|---|
| **Plan verification reminder** | Plan 退出 | 每 10 轮 | started/completed |
| Verification nudge ([[flag-vs-hardcode]]) | TodoWrite/TaskUpdate 关任务 | loop-exit 唯一时机 | 任务名含 verify |
| Compaction reminder | token 用量 ≥ 25% | 每 N tokens | snip 标记出现 |
| Context efficiency nudge | growth without snip | 每 10k tokens | snip / compact 边界 |

**通用思路**：节流提醒比"每轮都说"有效得多——LLM 看到重复提醒会忽略；偶尔出现的提醒更可能驱动行为。

## 跟其他概念的连接

| 上游 | 关系 |
|---|---|
| [[plan-mode-state-machine]] | Plan 退出**触发**本机制建立 pendingPlanVerification |
| [[post-generation-verification-channels]] | VerifyPlanExecution 是第 ③ 通路（Verifier）的具体调用入口 |
| [[task-notification-injection]] | Verifier 的结果通过 task-notification 回流（设置 verificationCompleted） |
| [[coordinator-mode]] | "Don't delegate" 提醒文本跟 coordinator 的"never write 'based on findings'"是同源原则 |
| [[ground-truth-via-tools]] | Plan 内容存在文件系统（[[plan-mode-state-machine]] 的 plan file），verifier 读文件而非记忆 |

下游影响：

| 下游 | 关系 |
|---|---|
| Reminder 节流（10 轮） | 防止 reminder 噪音 → [[false-claims-bidirectional]] 的"信息密度优于防御性"同源 |
| Auto-mode allowlist | 用户进 auto 时同意了"按预设规则放行"，验证是预设规则的一部分 |

## 设计原则

| 原则 | 含义 |
|---|---|
| **状态机驱动 reminder** | 不是定时器、不是每轮硬性提醒——状态决定是否提醒 |
| **节流避免噪音** | 每 10 轮一次。LLM 看到重复提醒会忽略，间歇性的提醒效果更好 |
| **Reminder 锚点是事件而非全局** | turnCount 从 plan_mode_exit attachment 起算，不是 session 全局 |
| **`directly (NOT the Agent tool)` 强约束** | 主 Agent 必须自己调，不能委托——验证责任不可转移（同 [[coordinator-mode]] 原则）|
| **system_reminder 包装区分系统提示** | 模型能识别这是系统插入的提醒而非用户对话 |
| **Auto-mode allowlist 减摩擦** | 用户已经隐式同意"plan 实施 + 验证"全流程，不再 prompt |
| **DCE 三层**：feature flag / USER_TYPE / 字面量比较 | 实验功能在 3P 完全消除——不只是 runtime 跳过，连代码都不打包 |
| **Tool 实现可分离，wire-up 必须保留** | 即使 tool 本体 DCE，状态字段/属性钩子/常量都要在公开代码中保留——避免 wire-up 漂移 |

## 可迁移性

任何"事件 → 后续动作必须发生"的工作流：

1. **状态机 + reminder 模式通用** — 用户提交订单 → 必须发邮件确认 → 状态机 + 节流提醒驱动
2. **节流参数实测决定** — 10 轮是个值得测的参数。太频繁（每轮）模型忽略；太稀疏（每 100 轮）忘了。这种参数应该 A/B 测
3. **Reminder 锚点要绑事件而非全局** — "退出 plan 后 10 轮"比"会话第 N 轮"语义更清晰
4. **强约束语言指定执行者** — "do X yourself, NOT delegate" 模式可以用在任何"责任不可转移"场景
5. **包装系统提示让模型识别** — wrapMessagesInSystemReminder 让消息看起来不像用户对话——LLM 处理这两类内容方式不同
6. **Auto 模式 allowlist 跟工作流默认同意对齐** — 用户进 auto 时同意了什么？把那条工作流上的工具加进 allowlist，减少 prompt 摩擦

针对 leto-ai 的电商 Agent：
- "用户下单 → 必须发履约确认通知" 工作流可以参考：在状态机里押栈"待发通知"，节流 reminder agent 调 send_notification 工具
- "退款审批 → 必须财务核账" 同模式
- DCE 模式可迁移性较低（leto-ai 大概不用 Bun + bundle DCE），但**字面量比较的编译期分支收窄**思路通用：用 `process.env.NODE_ENV === 'test'` 这类字面量决定要不要包含某段代码

## 失效模式与边界

| 失效场景 | 后果 |
|---|---|
| 模型 plan 实施完没看到 reminder | 等到第 10 轮注入——可能用户已经在等结果 |
| 模型在 reminder 出现前自己调了 VerifyPlanExecution | started=true，后续不再 reminder（理想路径）|
| 模型把验证委托给子 Agent | reminder 会持续触发——但状态不会被子 Agent 设置（VerifyPlanExecution 工具是主 Agent 专属）|
| 用户中途取消 plan 不再实施 | pendingPlanVerification 仍然 set，未来会持续 reminder——需要手动清理或 session 结束自动清 |
| 多次 plan 退出（连续做两个 plan） | 后一次的 plan 覆盖前一次（state 字段单一） |
| 3P 用户启用 CLAUDE_CODE_VERIFY_PLAN env var | runtime gate 通过但 tool 不存在——reminder 引用 `""` 工具名，模型懵 |

## 进一步追问的钩子

1. **VerifyPlanExecution 工具内部行为** — DCE 掉了。它具体怎么 spawn 验证 agent？拿 plan 跟实际改的文件 diff 比较？
2. **`getVerifyPlanReminderTurnCount` 的具体定义** — 找 plan_mode_exit 后过了几轮——但如果中间 plan 退出又再进入，怎么算？
3. **`CLAUDE_CODE_VERIFY_PLAN` 的实际启用配置** — env var 不在 ant 默认 build 里？是 opt-in 实验？
4. **Reminder 的关闭机制** — 如果模型坚持不调 VerifyPlanExecution，reminder 永远刷？有没有最大次数？
5. **跟 Verifier Agent 的边界** — VerifyPlanExecution 跟 [[post-generation-verification-channels]] 第 ③ 通路 verifier worker 的关系——是 spawn verifier 的快捷入口，还是另一种验证机制？

## 关联

- 上游：[[plan-mode-state-machine]]（plan 退出是触发事件）；[[post-generation-verification-channels]]（VerifyPlanExecution 调用 verifier 通路）
- 协同机制：[[task-notification-injection]]（verifier 结果回流方式）；[[coordinator-mode]]（"don't delegate"原则同源）；[[flag-vs-hardcode]]（其他 nudge 机制如 verification nudge）
- 反面对照：[[false-claims-bidirectional]]（节流是"避免无谓重复"的一面，本概念是"提醒漏掉的事"——两面都用）
- 相关实体：`state/AppStateStore.ts:411-417`（pendingPlanVerification 字段）、`screens/REPL.tsx:3065-3088`（plan 退出建立状态）、`utils/attachments.ts:3892-3929`（reminder 节流逻辑）、`utils/messages.ts:4240-4250`（reminder 文本）、`utils/permissions/classifierDecision.ts:40-44`（auto-mode allowlist）、`tools.ts:91-94`（tool 注册的 USER_TYPE 守门）
- 综合分析：[7.ANTI_HALLUCINATION.md](../7.ANTI_HALLUCINATION.md)（防幻觉整体架构里的验证一节）
