# Plan Mode State Machine（Plan 模式完整状态机）

> Plan Mode 不只是"只读模式" —— 它是一个**带栈顶记忆的状态机**：进入时把当前 mode 押进 `prePlanMode`，退出时按规则恢复（含 circuit-breaker 兜底降级）。叠加 auto-mode、bypass-mode、teammate 协作、文件持久化、跨 session 恢复后，状态空间相当复杂。

## 核心机制

```
                    EnterPlanMode (model 主动调)
default / auto / bypass ──────────────────────────►  plan (prePlanMode = 原 mode)
                         │  prepareContextForPlanMode
                         │  - 保存原 mode 到 prePlanMode
                         │  - 视 auto-during-plan 设置决定是否保留 auto
                         │  - 设置 plan 文件 slug
                         │
                                                       │
                                                       │ ExitPlanMode (用户批准)
                                                       │
                                                       ▼
                  原 mode（多数情况）                 ◄─ 检查 auto-mode gate
                  default（circuit-breaker 兜底）       ◄─ 处理 plan_approval_request (teammate)
```

`prePlanMode` 是这台状态机的**栈顶寄存器**——没有它，从 plan 怎么回原状态就丢失信息。

## 在 Claude Code 中的体现

### 状态构成

`ToolPermissionContext` 里跟 plan 相关的字段：

| 字段 | 含义 |
|---|---|
| `mode` | 当前 mode：`default` / `auto` / `bypassPermissions` / `dontAsk` / **`plan`** |
| `prePlanMode` | 进 plan 前的原 mode（仅 plan 期间有值）|
| `isBypassPermissionsModeAvailable` | 用户最初是否启用过 bypass —— 决定 plan 期间是否仍可写 |
| `strippedDangerousRules` | auto-mode 进 plan 时被剥离的危险规则 |

### 进入：EnterPlanModeTool（`tools/EnterPlanModeTool/EnterPlanModeTool.ts`）

```ts
async call(_input, context) {
  if (context.agentId) {
    throw new Error('EnterPlanMode tool cannot be used in agent contexts')
  }

  handlePlanModeTransition(appState.toolPermissionContext.mode, 'plan')

  context.setAppState(prev => ({
    ...prev,
    toolPermissionContext: applyPermissionUpdate(
      prepareContextForPlanMode(prev.toolPermissionContext),
      { type: 'setMode', mode: 'plan', destination: 'session' },
    ),
  }))
}
```

三个动作：

1. **拒绝在 agent context 里调用**（line 78-80）—— Plan Mode 是用户级状态，子 Agent 不能切
2. **`handlePlanModeTransition`**（`state.ts:1349`）—— 副作用 flag，标记 plan 退出时要注入 attachment
3. **`prepareContextForPlanMode`**（`permissionSetup.ts:1462`）—— **核心栈操作**

### 栈顶寄存器：`prePlanMode` 的保存

`prepareContextForPlanMode` 处理 4 种入口情况：

```ts
export function prepareContextForPlanMode(context) {
  const currentMode = context.mode
  if (currentMode === 'plan') return context              // 已经在 plan，no-op
  
  if (feature('TRANSCRIPT_CLASSIFIER')) {
    const planAutoMode = shouldPlanUseAutoMode()         // 用户的 useAutoModeDuringPlan 设置
    
    if (currentMode === 'auto') {
      if (planAutoMode) {
        return { ...context, prePlanMode: 'auto' }      // 路径 A: auto 保持
      }
      autoModeStateModule?.setAutoModeActive(false)
      setNeedsAutoModeExitAttachment(true)
      return {
        ...restoreDangerousPermissions(context),
        prePlanMode: 'auto',                            // 路径 B: auto 关，退出再恢复
      }
    }
    
    if (planAutoMode && currentMode !== 'bypassPermissions') {
      autoModeStateModule?.setAutoModeActive(true)
      return {
        ...stripDangerousPermissionsForAutoMode(context),
        prePlanMode: currentMode,                       // 路径 C: 进 plan 顺手开 auto
      }
    }
  }
  
  return { ...context, prePlanMode: currentMode }       // 路径 D: 普通进入
}
```

**4 种路径**反映了 auto 模式跟 plan 的复杂交叉：

| 进入时 mode | useAutoModeDuringPlan | 实际行为 | prePlanMode |
|---|---|---|---|
| auto | true | auto 在 plan 期间保持激活 | `'auto'` |
| auto | false | auto 关掉，退 plan 再恢复 | `'auto'` |
| default | true | plan 期间临时开 auto（路径 C）| `'default'` |
| default | false | 普通 plan（路径 D） | `'default'` |
| bypass | (任意) | plan 期间仍可写（line 1270）| `'bypassPermissions'` |

### 退出：ExitPlanModeV2Tool（`ExitPlanModeV2Tool.ts:243+`）

退出 5 步（按代码顺序）：

1. **读 plan 内容**：从 `permissionResult.updatedInput.plan`（CCR web UI 编辑过的）或磁盘（`getPlan(agentId)`）
2. **如果 plan 被编辑过**：写回磁盘 + 重做 file snapshot（让 VerifyPlanExecution / 后续 Read 看到最新内容）
3. **Teammate 路径**：如果是 teammate + `isPlanModeRequired`，发 `plan_approval_request` 到 team-lead mailbox，**不立即退出 plan**
4. **Circuit breaker 检查**：如果 `prePlanMode === 'auto'` 但当前 auto-mode gate 关了（circuit-breaker / settings disable）→ **降级回 default**，避免 ExitPlanMode 直接 setAutoModeActive 绕过 gate
5. **应用 mode 切换**：`setMode` 回 prePlanMode（或 default），`prePlanMode` 字段清空

### Circuit-Breaker 兜底降级

最值得圈出的一段代码（`ExitPlanModeV2Tool.ts:325-346`）：

```ts
// Circuit breaker defense: if prePlanMode was an auto-like mode but the
// gate is now off (circuit breaker or settings disable), restore to
// 'default' instead. Without this, ExitPlanMode would bypass the circuit
// breaker by calling setAutoModeActive(true) directly.
if (
  prePlanRaw === 'auto' &&
  !(permissionSetupModule?.isAutoModeGateEnabled() ?? false)
) {
  gateFallbackNotification = ...
}
```

场景：
- 用户进 plan 时 mode = auto → prePlanMode = 'auto' 押栈
- Plan 期间 auto-mode classifier 因为连续 deny 太多触发 circuit breaker 关闭
- 退 plan 时如果机械恢复 prePlanMode = auto，会**绕过 circuit breaker**

兜底：检查 gate 是否还开着，关了就降到 default + 发通知。

这是个**栈顶恢复 + 当前状态校验**的设计——不能盲目相信栈顶值还有效，要验证它是否在当前世界中仍可用。

### Plan 文件持久化（`utils/plans.ts:119-144`）

```ts
export function getPlanFilePath(agentId?: AgentId): string {
  const planSlug = getPlanSlug(getSessionId())
  if (!agentId) {
    return join(getPlansDirectory(), `${planSlug}.md`)
  }
  return join(getPlansDirectory(), `${planSlug}-agent-${agentId}.md`)
}
```

**默认目录**：`~/.claude/plans/` 或 `$CLAUDE_CONFIG_HOME/plans/`

**文件命名**：
- 主会话：`{planSlug}.md`
- 子 Agent：`{planSlug}-agent-{agentId}.md`

为什么子 Agent 单独文件？因为 plan 模式可以**通过 in-process teammate** 嵌套——每个 teammate 有自己的 plan 提交给 leader 审批，不能撞主会话的 plan。

### 跨 Session Plan 恢复

`copyPlanForResume`（`plans.ts:164+`）—— `claude --resume` 时尝试找回原会话的 plan：

1. 从 log 里找 `slug` 字段（每个 message 可带 slug 提示）
2. 用 slug 去 `~/.claude/plans/` 找文件
3. 如果不在，**fallback 到 file snapshot**（session 内增量保存的 plan 备份）
4. 都失败了，从 message history 重建（如果 plan content 出现在某条消息里）

**3 层恢复链**反映 plan 文件作为状态外部化的重要性——丢了它会话就破。

### Plan 模式期间的工具行为

不像我预期的那样有"全局工具过滤"——plan 模式的写约束**主要靠两条机制**：

1. **System prompt 提示**（EnterPlanModeTool.ts:104-118）：

> Remember: DO NOT write or edit any files yet. This is a read-only exploration and planning phase.

2. **Plan-mode-required teammates**（`AgentTool/agentToolUtils.ts:86`）：

```
// Allow ExitPlanMode for agents in plan mode (e.g., in-process teammates)
```

也就是说，plan 模式更像 **prompt-driven 行为约束**，工具层面没有强制过滤——除了 ExitPlanMode 在非 plan 模式下报错（line 215：`'You are not in plan mode...'`）。

例外是 **bypass + plan 同时存在**时（`permissions.ts:1268-1271`）：

```ts
const shouldBypassPermissions =
  appState.toolPermissionContext.mode === 'bypassPermissions' ||
  (appState.toolPermissionContext.mode === 'plan' &&
   appState.toolPermissionContext.isBypassPermissionsModeAvailable)
```

**用户最初是 bypass 进的 plan，plan 期间所有工具仍 allow**——因为他们已经接受过整体不询问的 deal，plan 只是 UI 状态切换而非权限切换。

### Teammate Plan 审批流程

`ExitPlanModeV2Tool.ts:264-313`：teammate 退出 plan 不直接退出，而是：

1. 生成 `plan_approval_request` JSON 包
2. 写到 `team-lead` mailbox（跨进程消息总线）
3. 标记 task 为 `awaitingPlanApproval`
4. **等 team-lead approve 才真正退 plan**

这是个**分布式 plan-mode**——单个 teammate 的 plan 不能擅自实施，必须 leader 集中审批。整个状态机这条路径**等待外部信号**。

### needsPlanModeExitAttachment：plan_mode_exit 信号

`state.ts:1349-1363` 的 `handlePlanModeTransition` 副作用：

```ts
export function handlePlanModeTransition(fromMode, toMode) {
  if (toMode === 'plan' && fromMode !== 'plan') {
    STATE.needsPlanModeExitAttachment = false   // 进入 plan: 清掉旧的 exit 标记
  }
  if (fromMode === 'plan' && toMode !== 'plan') {
    STATE.needsPlanModeExitAttachment = true    // 退出 plan: 标记下轮要发 attachment
  }
}
```

**plan_mode_exit attachment** 是一段说"plan 已批准、可以开始实施"的注入消息，下一轮 query 自动加到上下文里。这让 ExitPlanMode 的语义不只是模式切换，还包括给模型**显式信号"现在开始实施"**。

如果用户快速 toggle plan ↔ default，可能产生重复 attachment——所以进入 plan 时要先清掉旧标记。

## 设计原则

| 原则 | 含义 |
|---|---|
| **prePlanMode 是栈顶寄存器** | Plan 是 push 操作，Exit 是 pop。没有这个寄存器从 plan 出来就丢失原状态信息 |
| **栈顶恢复时必须校验** | Circuit breaker 已关闭时，prePlanMode='auto' 要降级回 default。不能盲目恢复栈顶值，要验证当前世界是否仍允许 |
| **Mode 是 prompt + permission 的混合约束** | Plan mode 的"只读"靠 prompt 提示 + tool checkPermissions，没有全局工具过滤 |
| **bypass + plan 不冲突** | 用户最初进 bypass 等于"不问就行"，plan 模式只是 UI 状态切换，不收紧权限 |
| **Plan 文件外部化是状态根** | 不在 in-memory state，而在磁盘 `{slug}.md`——会话能跨 resume 恢复，可以被外部编辑 |
| **3 层恢复链** | 文件 → file snapshot → message history。plan 是关键状态，丢了就破，多层兜底 |
| **子 Agent plan 文件隔离** | `{slug}-agent-{id}.md` 防止 nested teammate 互相覆盖 |
| **teammate 分布式审批** | 单 teammate plan 不直接 exit，写 plan_approval_request 到 team-lead mailbox，等批准。整个状态机能"等外部信号" |
| **agent context 不能切 plan** | EnterPlanMode `if (context.agentId) throw` —— Plan 是用户级状态，子 Agent 跟着主线程走 |
| **Channels 模式禁用 plan** | `isEnabled` 返回 false，避免"模型能进但不能出"的 trap（exit 需要 terminal dialog） |
| **Plan 退出 attachment 是显式信号** | needsPlanModeExitAttachment 让模型在下一轮看到"plan 已批准"，行为切换 |

## 失效模式与边界

| 失效场景 | 后果 |
|---|---|
| 用户连按 `shift+tab` 快速 toggle plan | needsPlanModeExitAttachment 进 plan 时清掉旧标记防重 |
| Circuit breaker 在 plan 期间触发 auto 关闭 | Exit 时降级回 default + 通知用户 |
| Plan 文件被外部 rm | copyPlanForResume 3 层 fallback；都失败则会话状态破，需要新建 plan |
| `~/.claude/plans/` 目录不存在 | mkdirSync recursive 自动创建 |
| Channels 模式启动时 model 试图 EnterPlanMode | `isEnabled = false` → 工具不在可用列表，model 没法调 |
| Subagent 的 EnterPlanMode 调用 | 直接抛 Error |
| Plan 模式下 model 仍调写工具 | 没有强制拦截——靠 prompt 自觉 + ExitPlanMode 应当先调 |
| Teammate plan 等不到 approval | 卡 awaitingPlanApproval 状态——长跑 |
| ExitPlanMode 在非 plan 模式被调 | 工具拒绝："You are not in plan mode" |

## 跟 [[opusplan-tradeoff]] 的关系

[[opusplan-tradeoff]] 讲的是**plan 模式作为 model 升级触发器**——`opusplan` 别名 + plan 模式 + token 不超 200K → 升 Opus。本概念是**plan 模式作为状态机本身**——状态如何进入、保存、退出、恢复。两者可对照：

| 维度 | [[opusplan-tradeoff]] | 本概念 |
|---|---|---|
| 关注点 | plan mode 怎么影响模型选择 | plan mode 自己的状态转移 |
| 触发器 | `getRuntimeMainLoopModel` 每轮重算 | EnterPlanMode / ExitPlanMode 工具调用 |
| 关键状态 | mainLoopModel | mode + prePlanMode |
| 设计取舍 | "动态裂隙"——破坏 static routing | "栈顶寄存器"——保留 push/pop 语义 |

合起来：**plan mode 是单一信号，但有两个独立的下游消费者**——模型路由层用它升 Opus，permission 层用它走 plan 状态机。

## 可迁移性

任何"模式切换"系统都可以借鉴：

1. **栈顶寄存器是模式系统的标配** — 进入临时模式 → 保存原模式；退出 → 恢复。但**恢复时要校验**栈顶值在当前世界仍可用
2. **Circuit-breaker 在退出路径上反向触发** — 进入时合法的状态，可能在临时模式期间被外部条件取消授权。退出不能盲目恢复
3. **状态外部化 + 多层恢复** — 关键状态（plan 内容）不只在内存，存到文件 + 多种 fallback。让 crash / resume 不破坏会话连续性
4. **Mode 不是单一信号** — 同一个 mode 字段可以被多个独立子系统消费（模型路由 + permission + UI）。设计时考虑解耦
5. **快速 toggle 防抖** — 用户连按 toggle 不应该产生 N 个事件。每次进入清掉旧的 exit 标记
6. **分布式审批通过 mailbox** — 跨进程 teammate 协作不是 RPC，是消息总线 + 状态机等待外部信号

针对 leto-ai：
- "审核中"模式可以参考 `prePlanMode` 设计——审核前的状态押栈，审核完恢复（含 circuit breaker：审核期间用户被风控降权，恢复时要校验）
- 订单状态机里的关键状态（草稿 / 待付款）可以外部化到 KV store + 多层恢复
- 多人协作的批准流可以参考 teammate plan_approval_request——不是 RPC 同步等，是消息总线异步等

## 进一步追问的钩子

1. **`useAutoModeDuringPlan` 这个 setting 在哪配置** — 4 种 auto×plan 交叉行为完全由它决定，是个高杠杆 setting
2. **Plan-mode interview phase**（`isPlanModeInterviewPhaseEnabled`）—— EnterPlanMode 的 instruction 不同（"Detailed workflow instructions will follow"）。这是另一个未挖的子模式
3. **VerifyPlanExecution hook** —— `ExitPlanModeV2Tool.ts:315` 提到 "Background verification hook is registered in REPL.tsx AFTER context clear via registerPlanVerificationHook()"。这是连接 plan 实施跟 [[post-generation-verification-channels]] 的桥梁
4. **Plan slug 怎么生成** — `getPlanSlug` 的具体逻辑，影响文件名稳定性
5. **`isPlanModeRequired` teammate 配置** — 哪些 teammate 强制要求 plan？在哪声明？
6. **CCR plan 编辑路径的细节** — `permissionResult.updatedInput.plan` 怎么从 web UI 流到这里？

## 关联

- 兄弟概念：[[opusplan-tradeoff]]（plan mode 作为模型路由信号）；本概念（plan mode 作为状态机自身）—— 同一字段两个独立消费者
- 协同机制：[[task-notification-injection]]（teammate plan_approval_request 走类似的 mailbox 模式）；[[circuit-breaker]] 直接应用——auto-mode gate 关闭时 plan 退出降级
- 相关实体：`tools/EnterPlanModeTool/EnterPlanModeTool.ts`、`tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`、`utils/permissions/permissionSetup.ts:1462`（prepareContextForPlanMode）、`bootstrap/state.ts:1349`（handlePlanModeTransition + needsPlanModeExitAttachment）、`utils/plans.ts:100-144`（plan 文件持久化）、`utils/permissions/permissions.ts:1268-1271`（bypass+plan 交叉）
- 综合分析：[5.PERMISSIONS.md](../5.PERMISSIONS.md)（4 种 mode 的整体）
