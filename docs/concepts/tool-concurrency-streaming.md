# Tool Concurrency & Streaming Execution（工具并发与流式执行）

> Claude Code 让工具**在模型还在生成下一个 token 时就开始执行**，并用 `isConcurrencySafe` 这条三态判定（true / false / 输入相关）决定哪些工具能并行。失败传播**刻意非对称**——只有 Bash 错误会取消兄弟，因为 bash 命令有隐式依赖链而 Read / WebFetch 之间是独立查询。

## 核心机制

```
模型流式生成 tool_use blocks
       │ (流式抵达 ─ 不必等模型说完)
       ▼
┌─────────────────────────────────────────────────────────┐
│ StreamingToolExecutor.addTool(block)                    │
│   ├─ 解析 schema → tool.isConcurrencySafe(parsed) → bool │
│   └─ 入队 → processQueue()                              │
└──────────────────┬──────────────────────────────────────┘
                   │
       ┌───────────┴───────────┐
       │                       │
   safe + 当前 executing       not safe
   全是 safe 的？              ?
       │                       │
       ▼                       ▼
   立即并发执行            等到所有 executing 完成
   （上限 ≤ 10）           然后独占执行
       │                       │
       └───────────┬───────────┘
                   ▼
           结果按 tool_use 顺序 yield
           （进度消息绕过顺序约束，立即 yield）
```

两条核心规则：
- **safe 工具批量并发**（同一 batch 全 safe 才能并存）
- **not-safe 工具独占执行**（队列内任何 not-safe 都阻塞后续）

## 在 Claude Code 中的体现

### 两条执行路径

| 路径 | 文件 | 用于 |
|---|---|---|
| `StreamingToolExecutor` | `services/tools/StreamingToolExecutor.ts` | **流式模式**——模型边生成边执行工具 |
| `runTools` | `services/tools/toolOrchestration.ts` | 非流式——模型完整输出后批量执行 |

两者共享并发判定逻辑，但 streaming 路径多了"工具陆续到来"的处理。`partitionToolCalls`（`toolOrchestration.ts:91`）把工具序列分成 batch：连续 safe 工具合并成并发 batch，遇到 not-safe 工具就单独成 batch。

### `isConcurrencySafe` 的设计谱

不是 boolean 而是 **基于输入的函数**——`Tool.ts:402`：

```ts
isConcurrencySafe(input: z.infer<Input>): boolean
```

默认值（`Tool.ts:759` 的 noopTool）是 **false**——任何不显式 override 的工具都被当作不安全。29 个内置工具的实际声明分四类：

| 类别 | 声明 | 工具 |
|---|---|---|
| **永远 safe** | `() => true` | FileRead, Glob, Grep, WebFetch, WebSearch, TaskList, TaskGet, TaskOutput, ToolSearch, ListMcpResources, BriefTool, LSPTool, AgentTool 等 |
| **输入相关** | `(input) => isReadOnly(input)` | Bash, PowerShell（看 command 是不是只读） |
| **状态变更但原子** | `() => true` | TaskCreate, TaskUpdate, TodoWrite（通过 35 行 pub-sub store 的 setState callback 串行化，并发安全） |
| **永远 not-safe** | 默认 false（不声明）/ `() => false` | FileEdit, FileWrite, NotebookEdit, McpAuth, ExitPlanMode, AskUserQuestion |

值得圈出几条：

**1. AgentTool 是 safe 的**（`AgentTool.tsx:1273`）

子 Agent spawning 被声明为并发安全——意味着主 Agent 可以同时 spawn 多个子 Agent。这是 Plan/Explore 等读类子 Agent 并行调研的基础。但 Verifier 是 background async（不占主循环），并发风险更小。

**2. Bash 用 `isReadOnly` 做代理**（`BashTool.tsx:434`）

```ts
isConcurrencySafe(input) {
  return this.isReadOnly?.(input) ?? false;
}
```

Bash 没有静态 yes/no——`ls` / `cat` / `grep` 这些**输入级别**判定为 read-only 的命令并发安全；`mkdir` / `rm` / `git commit` 不安全。这把判定推到 `checkReadOnlyConstraints`（包含 AST 解析 + 命令白名单）。

**3. TaskCreate/TaskUpdate 标 safe 的依据**

任务状态在 AppState pub-sub store（[[result-delivery-guarantee]] 相关）。`setState(prev => next)` 通过 callback 串行——并发的多个 `TaskCreate` 调用最终都 atomic 应用到 state，不会撕裂。这是个**写操作但并发安全**的反例——并发安全 ≠ read-only。

**4. FileEdit/FileWrite 的不安全**

不声明 isConcurrencySafe → 默认 false。理由很直接：两个 Edit 同时改同一文件会撕裂；改不同文件理论上 OK 但 readFileState 缓存的状态机假设串行化。**保守一点比容错复杂逻辑划算**。

### 失败传播的非对称性（最深的设计取舍）

`StreamingToolExecutor.ts:354-364`：

```ts
if (isErrorResult) {
  thisToolErrored = true
  // Only Bash errors cancel siblings. Bash commands often have implicit
  // dependency chains (e.g. mkdir fails → subsequent commands pointless).
  // Read/WebFetch/etc are independent — one failure shouldn't nuke the rest.
  if (tool.block.name === BASH_TOOL_NAME) {
    this.hasErrored = true
    this.erroredToolDescription = this.getToolDescription(tool)
    this.siblingAbortController.abort('sibling_error')
  }
}
```

**只有 Bash 错误**会触发 `siblingAbortController.abort('sibling_error')`，取消所有兄弟工具。其他工具（Read, Grep, WebFetch...）失败不影响兄弟。

注释里点明理由：

> Bash commands often have implicit dependency chains (e.g. `mkdir` fails → subsequent commands pointless). Read/WebFetch/etc are independent — one failure shouldn't nuke the rest.

被取消的兄弟收到 `<tool_use_error>Cancelled: parallel tool call <bash desc> errored</tool_use_error>`——主 Agent 看到不仅是"工具失败"，还包括**为什么被取消**（哪个 bash 错了），可以决定下一步。

这条设计跟 [[multi-tier-degradation]] / [[circuit-breaker]] 都不同——它是**针对工具语义的非对称失败传播**：知识来源于"shell 命令通常依赖前一条成功"的领域知识，不是通用的失败级联模式。

### 三层 AbortController 架构

```
parent: toolUseContext.abortController          (query 级)
   │
   └─► child: siblingAbortController            (Bash 错误时 abort)
          │
          └─► child × N: per-tool toolAbortController  (单工具取消)
```

每个工具拿到自己的 child controller。三层关系：

| 触发 | 影响 |
|---|---|
| 用户按 ESC | parent abort → 全部 cancel |
| Bash 工具内部错 | sibling abort → 兄弟 cancel + per-tool abort |
| 单工具的权限被拒 | per-tool abort → bubble up 到 parent，**终止整个 turn** |

最微妙的是**反向 bubble-up**（`StreamingToolExecutor.ts:301-318`）：

```ts
toolAbortController.signal.addEventListener('abort', () => {
  if (
    toolAbortController.signal.reason !== 'sibling_error' &&
    !this.toolUseContext.abortController.signal.aborted &&
    !this.discarded
  ) {
    this.toolUseContext.abortController.abort(
      toolAbortController.signal.reason,
    )
  }
}, { once: true })
```

注释明示这是 issue #21056 的修复：

> Permission-dialog rejection also aborts this controller — that abort must bubble up to the query controller so the query loop's post-tool abort check ends the turn. Without bubble-up, ExitPlanMode "clear context + auto" sends REJECT_MESSAGE to the model instead of aborting.

权限被拒**不只取消那个工具**——必须把信号传到 query 顶层让整个 turn 结束。否则模型会以为只是"这个工具调用失败了，下一步换个方法试"，但实际上用户是想完全终止。

### 进度消息绕过顺序约束

`StreamingToolExecutor.ts:367-374`：

```ts
if (update.message.type === 'progress') {
  tool.pendingProgress.push(update.message)
  // Signal that progress is available
  if (this.progressAvailableResolve) {
    this.progressAvailableResolve()
    this.progressAvailableResolve = undefined
  }
}
```

设计：**结果按 tool_use 顺序 yield，但进度消息立即 yield**。

为什么？一个长跑工具（Bash 跑 30 秒）不应该让用户/UI 干等。所以"工具 1 还在跑"的进度消息不阻塞，"工具 1 的最终 result"则等工具 1 真的完成。这让用户感知到"系统在干活"而不是僵住。

### Context Modifiers 的并发缺口

`StreamingToolExecutor.ts:388-390`：

```ts
// NOTE: we currently don't support context modifiers for concurrent
//       tools. None are actively being used, but if we want to use
//       them in concurrent tools, we need to support that here.
```

某些工具会修改后续工具的 ToolUseContext（如 cwd 切换）——这种 "context modifier" 目前**只对非并发工具生效**。注释承认：并发情况下"哪个工具的 modifier 先生效"语义不明，未实现。

这是个**未填的坑**——如果未来加并发 + context-modifying 的工具，要先想清楚 modifier 排序语义。

### Interrupt Behavior：cancel vs block

每个工具可声明 `interruptBehavior() → 'cancel' | 'block'`：

- `cancel`：ESC 时立即终止
- `block`：ESC 不中断（默认）

Bash 走 `block`——半途中断 `mkdir` / `rm` 这类系统调用可能留下不一致状态。

UI 上的"可中断"指示器（line 254-260）只在**所有正在 executing 的工具都是 cancel** 时亮——只要有一个 block 工具在跑，整个 batch 视为不可中断。

### 流式 fallback 的丢弃逻辑

`discard()` + `streaming_fallback` 错误：当流式连接断开退化到非流模式，已经在跑的工具结果**全部丢弃**。每个未完成工具收到合成错误：

```xml
<tool_use_error>Error: Streaming fallback - tool execution discarded</tool_use_error>
```

主 Agent 看到这个会重新发起 tool_use（因为没有真正的 tool_result），形成幂等的重试。

### 全局并发上限

`toolOrchestration.ts:8`：

```ts
function getMaxToolUseConcurrency(): number {
  return parseInt(process.env.CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY || '', 10) || 10
}
```

默认 10。env var 可调。这是个**简单粗暴的兜底**——防止"模型一次发 50 个 tool_use"把系统打挂（API 限流、文件句柄耗尽）。

## 设计原则

| 原则 | 含义 |
|---|---|
| **Concurrency-safe 是输入函数，不是 boolean** | Bash/PowerShell 同一工具不同 input 安全性不同。把判定推到 input 级别 |
| **默认 not-safe，必须显式声明** | `Tool.ts:759` 默认 false。新工具不主动思考并发安全性就保守串行——比"默认并发出错"安全 |
| **失败传播按工具语义非对称** | Bash 错误 cancel 兄弟（命令链假设），其他失败不传染（独立查询）。**不要做通用的"任何失败 cancel 全部"** |
| **结果保序，进度不保序** | 用户感知到的 latency 由进度消息驱动，最终结果按 tool_use 顺序回流——两者解耦 |
| **三层 AbortController 解耦故障作用域** | parent / sibling / per-tool 三层让"用户取消" / "Bash 错误连锁" / "单工具问题" 各管各的 |
| **Bubble-up 信号承认"工具级失败 = turn 级失败"的特例** | 权限被拒不是普通错误，是"用户决定停止"——必须升级到 turn 级 |
| **状态变更也能 safe（如果是原子的）** | TaskCreate/TaskUpdate 标 safe 因为 setState 是 callback 串行——并发安全 ≠ 只读 |
| **interruptBehavior 区分中断代价** | Bash 用 block 防止中途中断系统调用——并发 + 可中断不是普世原则 |
| **流式 fallback 全丢弃 + 重试** | 部分结果不可信任，宁可重跑——简单语义胜过复杂的 partial 状态恢复 |

## 失效模式与边界

| 失效场景 | 后果 |
|---|---|
| `isConcurrencySafe(input)` 抛错 | 保守降级为 not-safe（`toolOrchestration.ts:103-106` 的 try/catch） |
| 模型一次发超过 10 个 tool_use | 超过 `MAX_TOOL_USE_CONCURRENCY` 的部分排队等 |
| 多个 Bash 同时跑且有依赖 | 都走串行路径（Bash 不安全），但 LLM 可能不知道这点而误以为并发 |
| 并发工具想修改 context | 静默丢弃 modifier（NOTE 注释承认） |
| 流式中断时工具刚写了一半文件 | discard 不能撤销已完成的副作用——只能让模型重试时发现状态变了 |
| Bash 错误 cancel 了兄弟，但兄弟其实独立 | 误伤——但代价"白跑一次"低于"语义不一致"的代价 |

## 可迁移性

任何"LLM + 多工具"系统都会遇到：

1. **不要默认并发** — 显式标注哪些工具能并发，比"默认并发遇到 bug 再修"安全
2. **失败传播按领域知识非对称** — bash / shell 命令有依赖链，db transaction 有依赖链，但 RAG 检索没有。**不要用统一规则**
3. **进度与结果分流** — 让用户感知到工具在动，但保持结果顺序——两条独立 channel
4. **AbortController 三层结构** — turn 级 / batch 级 / 工具级——每层各有取消触发条件
5. **Bubble-up 是反直觉但必要** — 工具级"用户取消"必须升级到 turn 级，否则模型会无视它继续推理

针对 leto-ai 业务场景：

- **支付/库存这类强依赖链工具组**（`reserve_inventory` → `charge_payment` → `confirm_order`）应当声明 not-safe 串行执行 —— 跟 Bash 的设计同源
- **查询类工具**（`get_product_info` / `get_user_history`）天然 safe，可以并发降低响应时间
- **业务领域的失败传播**：库存预占失败 → 取消支付兄弟（合理）；商品图片加载失败 → 不应取消价格查询（独立）。**写出领域专属的失败传播规则**，跟 Claude Code 只 cancel Bash 同样精细
- **interruptBehavior 是个值得搬的概念**：用户取消订单流程时，部分操作（如发邮件）已发出无法回滚，应当是 `block`；查询类操作可以安全 `cancel`

## 进一步追问的钩子

1. **`isConcurrencySafe` 错判的代价模型** — 把不安全的标 safe → 数据撕裂；把安全的标 not-safe → 串行损失。两个错误代价不对称，所以默认偏保守
2. **AgentTool 标 safe 的真实并发上限** — 主 Agent 同时 spawn 5 个子 Agent，每个又 spawn 5 个？`MAX_TOOL_USE_CONCURRENCY` 是顶层限制还是每层独立？
3. **Bash sibling abort 的副作用** — `siblingAbortController.abort` 后，正在跑的 bash subprocess 收到 SIGTERM——但已经写了一半的文件 / DB 状态不会自动回滚
4. **流式 fallback 的幂等假设** — 重试时如果工具有副作用（`mkdir` 已经成功创建目录），第二次会失败。Claude Code 怎么处理？
5. **`CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY=10` 的来源** — 这个数字是怎么定的？API rate limit / 文件句柄 / 机器 RAM？

## 关联

- 上层概念：[[ground-truth-via-tools]]（并发执行不影响事实链——每个 tool_result 仍然是独立的事实）
- 协同机制：[[tool-args-prevalidation]]（每个工具调用都先走 7 步管线，不论并发与否）；[[result-delivery-guarantee]]（streaming fallback discard 是结果交付的特殊路径）
- 反面对照：[[multi-tier-degradation]] / [[circuit-breaker]] 是通用失败级联模式；本概念**故意不用通用规则**，按工具语义定制
- 相关实体：`services/tools/StreamingToolExecutor.ts`（流式路径）、`services/tools/toolOrchestration.ts`（非流路径）、`Tool.ts:402`（isConcurrencySafe 接口）、`tools/BashTool/BashTool.tsx:434`（输入级判定典型）、`utils/abortController.ts`（三层 AbortController）
- 综合分析：[2.TOOLS.md](../2.TOOLS.md)（工具系统整体）
