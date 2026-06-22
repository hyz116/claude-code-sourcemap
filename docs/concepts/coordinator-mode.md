# Coordinator Mode（协调员模式）

> 主 Agent 的**替代操作模式**——不直接执行工具、只调度 worker，用 SendMessage 续聊+ TaskStop 干预。所有 worker 都是 async 的，主 Agent 在 worker 工作时**继续推理**。换来的是显式的"研究→综合→实施→验证"工作流和并行性，代价是单次任务的延迟更高。

## 核心机制

```
传统模式：主 Agent 自己跑 Bash/Edit/Read，worker 偶尔用
       │
       ▼  CLAUDE_CODE_COORDINATOR_MODE=1
       
Coordinator 模式：
                  ┌─ 工具集 ─┐
   主 Agent ──►   │ AgentTool │  spawn workers (forced async)
                  │ SendMsg   │  续聊已有 worker
                  │ TaskStop  │  打断走错的 worker
                  │ subscribe_pr_activity (PR 模式)
                  └───────────┘
                      │
                      │ 通过 task-notification XML（user 角色注入）拿结果
                      ▼
       Workers (async，全部并行)
                  ┌─ 工具集 ─┐
                  │ Bash      │
                  │ FileRead  │
                  │ FileEdit  │
                  │ Glob/Grep │
                  │ WebFetch  │
                  │ ...       │
                  │ MCP tools │
                  │ Skills    │  
                  └───────────┘
                      │
                      │ 工作完成 → task-notification 回主 Agent
                      ▼
       主 Agent 综合 → 决定下一步（continue / spawn / 报告用户）
```

**身份保留**：coordinator 的 `agentId === undefined`——它还是**主线程身份**，跟普通主 Agent 一样消费 user prompt、写 user-facing message。但它**不亲自执行任何文件操作或命令**。

## 在 Claude Code 中的体现

### 激活与会话绑定（`coordinator/coordinatorMode.ts:36-78`）

```ts
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}

export function matchSessionMode(
  sessionMode: 'coordinator' | 'normal' | undefined,
): string | undefined {
  // ... 如果当前模式跟 session 存的不一致，翻转 env var
  if (sessionIsCoordinator) {
    process.env.CLAUDE_CODE_COORDINATOR_MODE = '1'
  } else {
    delete process.env.CLAUDE_CODE_COORDINATOR_MODE
  }
}
```

两层 gate：
1. **`feature('COORDINATOR_MODE')`**: compile-time DCE flag（3P 是否打包）
2. **`CLAUDE_CODE_COORDINATOR_MODE` env var**: runtime 启用

**会话绑定**：每个会话开始时记录是否 coordinator（main.tsx:3771: `saveMode(isCoordinatorMode() ? 'coordinator' : 'normal')`）。Resume 时如果 env var 跟存的不一致，**flip env var 让 isCoordinatorMode() 跟 session 一致**——同一会话不能在两种模式间漂移。

### 主 Agent 工具集（`tools.ts:281-296`）

`CLAUDE_CODE_SIMPLE` + Coordinator 模式：

```ts
const simpleTools: Tool[] = [BashTool, FileReadTool, FileEditTool]
if (feature('COORDINATOR_MODE') && coordinatorModeModule?.isCoordinatorMode()) {
  simpleTools.push(AgentTool, TaskStopTool, getSendMessageTool())
}
```

注释解释：

> When coordinator mode is also active, include AgentTool and TaskStopTool so the coordinator gets Task+TaskStop (via useMergedTools filtering) and **workers get Bash/Read/Edit** (via filterToolsForAgent filtering).

也就是说：**同一个 tool list，coordinator 和 worker 看到的是不同子集**。filterToolsForAgent 在 worker 启动时把 AgentTool/SendMessage/TaskStop 拿掉，filterToolsByDenyRules 给 coordinator 装上。

### Worker 工具集（`constants/tools.ts:55`）

```ts
export const ASYNC_AGENT_ALLOWED_TOOLS = new Set([
  FILE_READ_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  TODO_WRITE_TOOL_NAME,
  GREP_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  GLOB_TOOL_NAME,
  ...SHELL_TOOL_NAMES,           // Bash, PowerShell
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
  NOTEBOOK_EDIT_TOOL_NAME,
  SKILL_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
  TOOL_SEARCH_TOOL_NAME,
  ENTER_WORKTREE_TOOL_NAME,
  EXIT_WORKTREE_TOOL_NAME,
])
```

Worker **不能**：
- 调 AgentTool（不能 spawn 子-子 worker）
- 调 SendMessage（不能跟其他 worker 直接通信）
- 调 TeamCreate/TeamDelete（不能管理团队）

只有 coordinator 能管 worker；worker 之间**互相不可见**——通过 coordinator 中转。这是个**显式的层级隔离**，避免 worker 互相阻塞或形成依赖图。

### Forced Async（`AgentTool.tsx:567`）

```ts
const shouldRunAsync = (
  run_in_background === true ||
  selectedAgent.background === true ||
  isCoordinator ||                        // ← 这里
  forceAsync ||
  assistantForceAsync ||
  (proactiveModule?.isProactiveActive() ?? false)
) && !isBackgroundTasksDisabled
```

**Coordinator 模式下，所有 AgentTool spawn 都是 async**——coordinator 不阻塞等 worker，立即得到 `agentId` 并继续推理。worker 完成时通过 task-notification 把结果**注入回主对话流**（[[task-notification-injection]]）。

这是 coordinator 模式的核心生产力来源——**主线程一直在思考，worker 一直在干活**，两条流时间错位。

### Model 参数被丢弃（`AgentTool.tsx:252`）

```ts
const model = isCoordinatorMode() ? undefined : modelParam
```

Coordinator 不能给 worker 指定 model——必须用 default。Coordinator system prompt 里也明说：

> Do not set the model parameter. Workers need the default model for the substantive tasks you delegate.

理由：Coordinator 该集中思考"该让 worker 做什么"，而不是"给 worker 哪个 model"。模型选择应该按 task 类型自动决定（[[model-routing]]），而非 coordinator 临时拍板。

### ForkSubagent 禁用（`forkSubagent.ts:34`）

```ts
if (isCoordinatorMode()) return false
```

Fork subagent（继承当前对话上下文 spawn 一个分身）在 coordinator 模式禁用。Coordinator 永远 spawn **fresh worker**——这跟 "workers can't see your conversation" 的设计原则一致。

### Proactive 通知禁用（`main.tsx:2199`）

```ts
}).proactive || isEnvTruthy(process.env.CLAUDE_CODE_PROACTIVE)) 
   && !coordinatorModeModule?.isCoordinatorMode())
```

Coordinator 模式下 proactive 关闭——**减少干扰，让 coordinator 专注于 worker 编排**。

### 跨 Worker 共享 Scratchpad（`coordinatorMode.ts:104-106`）

```ts
if (scratchpadDir && isScratchpadGateEnabled()) {
  content += `\n\nScratchpad directory: ${scratchpadDir}\nWorkers can read and write here without permission prompts. Use this for durable cross-worker knowledge — structure files however fits the work.`
}
```

由 `tengu_scratch` GrowthBook flag 控制。**所有 worker 共享一个 scratchpad 目录**，可以放跨 worker 知识。**免权限提示**——这是个意图明确的"工作区"。

为什么需要？Worker 间不能直接通信（前面讲了），但有时需要"worker A 的研究结果给 worker B 用"。Coordinator 可以读 worker A 的 task-notification，然后在 spawn worker B 时贴进 prompt——但 prompt 太长不经济。Scratchpad 让 worker A 把结果写到文件，coordinator 只告诉 B "去看 `~/.claude/scratch/auth-research.md`"。

## System Prompt 设计（`coordinator/coordinatorMode.ts:111-368`）

整段 system prompt 是 369 行——比普通模式长得多。几个**值得圈出的契约**：

### 「Every message you send is to the user」

```
Worker results and system notifications are internal signals, not
conversation partners — never thank or acknowledge them. Summarize new
information for the user as it arrives.
```

明确告诉模型：**user-role 的 task-notification 不是对话伙伴**，不要 reply 给它。所有 assistant 输出都给用户。

### 「Don't delegate work you can handle without tools」

```
Answer questions directly when possible — don't delegate work that you
can handle without tools
```

防止 coordinator 滑向"任何小事都开 worker"的反模式——纯文本问题（"什么是 X"）coordinator 直接答。Worker 是为有副作用的工作准备的。

### 「Parallelism is your superpower」

```
**Parallelism is your superpower. Workers are async. Launch independent
workers concurrently whenever possible** — don't serialize work that can
run simultaneously and look for opportunities to fan out.
```

**鼓励 fan-out**：研究阶段同时开 3-4 个 worker 探索不同角度。在传统模式下这是反模式（多个 worker 抢主线程注意力），coordinator 模式专为此设计。

### 「Synthesize is your most important job」

```
Never write "based on your findings" or "based on the research." These
phrases delegate understanding to the worker instead of doing it yourself.
You never hand off understanding to another worker.
```

**反 lazy delegation**——这条原则跟 [[false-claims-bidirectional]] 是同源思路：禁止 hedging / 模糊化。coordinator 必须**真的读懂 worker 的报告**，提炼成包含 file path 和 line number 的明确 spec 给下一个 worker。

注释里的反例：

```ts
// Anti-pattern — lazy delegation
${AGENT_TOOL_NAME}({ prompt: "Based on your findings, fix the auth bug", ... })

// Good — synthesized spec
${AGENT_TOOL_NAME}({ prompt: "Fix the null pointer in src/auth/validate.ts:42. The user field on Session (src/auth/types.ts:15) is undefined when sessions expire but the token remains cached. Add a null check before user.id access — if null, return 401 with 'Session expired'. Commit and report the hash.", ... })
```

后者比前者长 5 倍但**直接告诉 worker 该做什么**，不要 worker 自己再去理解。

### Continue vs Spawn 决策矩阵

System prompt 给了一个完整的决策表：

| 情况 | 机制 | 为什么 |
|---|---|---|
| 研究探索的文件正好是要编辑的文件 | Continue | worker 已有文件 in context，加上明确 spec |
| 研究广而实施窄 | Spawn fresh | 避免拖累探索噪音 |
| 修复失败或继续最近工作 | Continue | worker 有错误 context，知道刚试了啥 |
| 验证别的 worker 写的代码 | Spawn fresh | Verifier 应该 fresh eyes，不带实施假设 |
| 第一次实施用错路径 | Spawn fresh | 错误路径 context 污染重试 |
| 完全无关任务 | Spawn fresh | 没有可复用 context |

**没有通用 default**——按 context overlap 决定。这是个 coordinator 必须**主动判断**的事，不能拍脑袋。

### Worker 自验证 + 单独 Verifier 双层

```
For implementation: "Run relevant tests and typecheck, then commit your
changes and report the hash" — workers self-verify before reporting done.
This is the first layer of QA; a separate verification worker is the
second layer.
```

跟 [[post-generation-verification-channels]] 直接对接：worker 自己跑 test/typecheck（first layer），然后 coordinator 再 spawn 独立 Verifier worker（second layer）。**双层 QA** 是 coordinator 模式的标准工作流。

## 跟其他概念的对比

| 概念 | 关系 |
|---|---|
| [[task-notification-injection]] | Coordinator 完全依赖这个机制收 worker 结果 |
| [[plan-mode-state-machine]] | Coordinator 的 agentId === undefined 让 plan mode 仍然能进入 |
| [[tool-concurrency-streaming]] | Coordinator 模式下 AgentTool 全部 async，多 worker 真并行 |
| [[post-generation-verification-channels]] | "Verification worker is the second layer" 直接引用 |
| [[false-claims-bidirectional]] | "Never write 'based on your findings'" 是同源原则 |
| [[bash-command-classification]] | Worker 跑 bash 时仍然走完整分类管线（permission 不放松） |
| [[ground-truth-via-tools]] | task-notification 是 coordinator 唯一的事实源——不能 fabricate |

## 设计原则

| 原则 | 含义 |
|---|---|
| **职责分离：coordinator 编排、worker 执行** | 主 Agent 不动手，所有 file/shell 操作下放给 worker |
| **身份不变性：coordinator 仍是主线程** | `agentId === undefined`——是不是亲自动手不影响"我是谁" |
| **强制 async 让推理跟执行并行** | 主 Agent 不阻塞等 worker，立即推进。这是模式的核心生产力来源 |
| **Worker 互相隔离** | 不能 spawn 子 worker、不能 SendMessage、不能 TeamCreate。所有协调由 coordinator 中转 |
| **每个 worker prompt 自包含** | "workers can't see your conversation"——coordinator 必须把所有 context 放进 prompt |
| **Synthesize 不能 delegate** | Coordinator 必须亲自读懂 worker 报告，禁止"based on your findings"——理解不能转移 |
| **Continue vs Spawn 按 context overlap** | 没通用 default。每次都要主动判断 |
| **双层 QA：worker 自验证 + verifier worker** | first layer worker 自己 typecheck/test；second layer 独立 Verifier |
| **Scratchpad 解耦"worker 间共享"问题** | Worker 不通信，但可以共享文件。Coordinator 告诉 worker B 去读 worker A 写的文件 |
| **Model param 不可设置** | Coordinator 不挑模型——按 task 自动选 |
| **Fork 禁用** | 永远 fresh worker，跟"workers can't see your conversation"一致 |
| **Session 绑定，不能漂移** | matchSessionMode 强制对齐——同一会话不能从 normal 切到 coordinator |
| **Proactive 关掉减少干扰** | Coordinator 模式专注于编排，不被 proactive 通知打断 |

## 失效模式与边界

| 失效场景 | 后果 |
|---|---|
| Coordinator 偷懒"based on your findings" | Worker 自由发挥（lazy delegation 的反模式）—— prompt 强约束但模型可能违反 |
| 同时 spawn 多个 implementation worker 改同一文件 | 编辑冲突——prompt 提示 "Write-heavy tasks — one at a time per set of files" 但靠 coordinator 自觉 |
| Worker 报告含具体细节但 coordinator 不读 | Synthesize 失败——symptom: 下一个 worker prompt 还是模糊的 |
| 全部 worker 同时跑炸 API rate limit | `MAX_TOOL_USE_CONCURRENCY=10` 兜底（[[tool-concurrency-streaming]]） |
| Resume 一个 normal session 但 env var 是 coordinator | matchSessionMode 自动 flip，输出"Exited coordinator mode to match resumed session." |
| Coordinator 试图调 ForkSubagent | tool 直接报错（forkSubagent.ts:34 deny） |
| Worker 试图 spawn 子 worker | 工具不在它的 ASYNC_AGENT_ALLOWED_TOOLS 集合，看不到这个工具 |
| Workers 想互相通信 | 不行——通过 coordinator 中转或共享 scratchpad |

## 可迁移性

任何"orchestrator + executor"分层 Agent 系统都可以借鉴：

1. **职责强分层**：编排者不执行、执行者不编排。中间通过明确的消息接口（task-notification）通信
2. **强制 async 是关键设计**：让编排者跟执行者真并行——同步等 worker 等于退化回单线程
3. **Worker 互相隔离避免依赖图复杂化**：所有协调由编排者中转。需要共享数据时用文件/数据库（scratchpad），不要 worker 间 RPC
4. **每个 worker prompt 自包含**：执行者看不到编排者的对话上下文。所有信息打包进 prompt 减少耦合
5. **强制编排者 synthesize**：禁止"based on findings"这种 lazy delegation。**理解必须在编排者那里发生**
6. **Continue vs spawn 按 context overlap**：判断每次任务跟 worker 现有 context 的重合度——重合多续聊、重合少新起
7. **双层 QA**：执行者自验证（快速反馈循环）+ 独立验证器（fresh eyes）
8. **Session 绑定模式**：会话开始决定，不能中途漂移——避免状态机退化

针对 leto-ai 的电商 Agent：
- 客服对话 Agent + 订单执行 worker 分层非常自然
- Worker 之间不要直连——比如"库存查询 worker"和"价格查询 worker"通过 coordinator 中转，避免形成 worker 依赖图
- Scratchpad 思路 → Redis 短期存储让 worker 间共享中间结果
- "Continue vs spawn"决策可以照抄到客服场景（用户连续问 vs 切话题）
- "禁止 based on findings"是给 coordinator agent 的强 prompt 约束——leto-ai 的主 agent 也应该有

## 进一步追问的钩子

1. **Coordinator 跑 Plan Mode 是什么样** — Coordinator 进 plan 时，workers 也进 plan 吗？还是 plan 是 coordinator-only 状态？两者交互不明
2. **Worker 失败 N 次后的回退策略** — system prompt 说 "If a correction attempt fails, try a different approach or report to the user"——但没有 hard limit
3. **subscribe_pr_activity 的具体协议** — coordinator 专属的 PR 监听，事件以 user message 形式到达——跟 task-notification 是同一个机制吗？
4. **Scratchpad 跨会话语义** — 重启 session 后 scratchpad 还在吗？多个 session 同时跑会冲突吗？
5. **`CLAUDE_CODE_SIMPLE` 模式存在感** — 给 worker 只发 Bash+Read+Edit。这个模式的目标用户是谁？为什么不直接用全工具集？
6. **Coordinator 对 sub-agent 的 SDK exposure** — SDK 用户能不能用 coordinator mode？文档怎么讲？

## 关联

- 上层概念：（无；coordinator 模式是顶层操作模式）
- 协同机制：[[task-notification-injection]]（worker 结果交付的唯一通道）；[[tool-concurrency-streaming]]（forced async + worker 并行）；[[post-generation-verification-channels]]（双层 QA 模式）；[[plan-mode-state-machine]]（agentId 一致性）
- 反面对照：传统主 Agent 模式——coordinator 是它的"职责强分层"变体；[[false-claims-bidirectional]] 是 prompt 层的同源原则（禁止 hedging）
- 相关实体：`coordinator/coordinatorMode.ts`（核心模块）、`tools/AgentTool/AgentTool.tsx:223-252,567,750`（forced async + 工具过滤）、`tools/AgentTool/forkSubagent.ts:34`（fork 禁用）、`tools.ts:281-296`（工具子集分配）、`constants/tools.ts:55`（worker 工具集）、`main.tsx:2199`（proactive 禁用）、`main.tsx:3771`（session 绑定）
- 综合分析：[3.MULTI_AGENT.md](../3.MULTI_AGENT.md)（多 Agent 整体架构）
