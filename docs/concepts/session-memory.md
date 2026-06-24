# Session Memory（会话记忆：后台增量笔记 → 预构建压缩摘要）

> 一个**后台 forked 子 agent** 周期性把当前会话的关键信息抽取进一个固定模板的 markdown 文件（`summary.md`）；它的**首要消费者不是"以后读回来"，而是压缩**——当 autocompact 触发时，这份已经增量维护好的笔记**直接当摘要用**，省掉同步调 Claude 总结整段历史的那次昂贵调用。这是 [[context-compression-cascade]] ④ Autocompact 的"预计算"旁路，也是 [context-engineering-anthropic.md](../insights/context-engineering-anthropic.md) §② 结构化笔记在 Claude Code 里的一种**具体落地形态**。

> **信任标注**：本页 `OBS` = 源码直引（file:line）；`INF` = 我从代码合理推断；`SPEC` = 推测。读"设计原则/可迁移性"等抽象段时按"+1 怀疑度"。

## 核心机制（OBS）

```
主 REPL 每轮采样后（post-sampling hook，仅 repl_main_thread）：
        │
        ▼
  shouldExtractMemory(messages)?   ← 阈值判定
        │ 是
        ▼
  fork 一个隔离子 agent（runForkedAgent, querySource='session_memory'）
        │  权限锁死：只允许 Edit 这一个文件，其他工具一律 deny
        ▼
  子 agent 按模板把会话要点 Edit 进 {sessionId}/session-memory/summary.md
        │
        ▼
  记录 lastSummarizedMessageId（标记"截至哪条消息已入笔记"）

────────────────────────────────────────────────────────

压缩触发时（trySessionMemoryCompaction）：
  若 summary.md 有实质内容（非空模板）
     → 保留 lastSummarizedMessageId 之后的消息 + summary.md 当摘要
     → 不再同步调 Claude 总结（绕过 autocompact 的 LLM 调用）
  否则 → 回退 legacy/autocompact
```

两条独立的循环：**写**（后台增量抽取）与**用**（压缩时当摘要）。`lastSummarizedMessageId` 是两者的接缝——它标记笔记覆盖到哪条消息，压缩时据此切"已总结/未总结"边界。

## 在 Claude Code 中的体现（OBS）

### 存储（`utils/permissions/filesystem.ts:261-270`）

- 路径：`{projectDir}/{sessionId}/session-memory/summary.md` —— **每会话一个文件**，单一 markdown。
- 权限：目录 `0o700`、文件 `0o600`（私有）；创建用 `wx`（`O_CREAT|O_EXCL`）防并发覆盖（`sessionMemory.ts:194-200`）。

### 固定模板（`SessionMemory/prompts.ts:11-41`）

10 个固定 section，每个带 _斜体说明_：`Session Title` / `Current State` / `Task specification` / `Files and Functions` / `Workflow` / `Errors & Corrections` / `Codebase and System Documentation` / `Learnings` / `Key results` / `Worklog`。可被 `~/.claude/session-memory/config/template.md` 覆盖。

### 触发阈值（`sessionMemory.ts:134-181` + `sessionMemoryUtils.ts:32-36`）

默认配置（可被 GrowthBook `tengu_sm_config` 远程覆盖）：

| 参数 | 默认 | 含义 |
|---|---|---|
| `minimumMessageTokensToInit` | 10000 | 上下文达此 token 才首次初始化 |
| `minimumTokensBetweenUpdate` | 5000 | 距上次抽取的**上下文增长**门槛 |
| `toolCallsBetweenUpdates` | 3 | 距上次抽取的工具调用数门槛 |

触发条件（`:168-170`）：`(token门槛 且 工具调用门槛)` **或** `(token门槛 且 上一轮无工具调用=自然停顿点)`。注释强调 **token 门槛永远必需**——光满足工具调用数不会触发，防过度抽取。token 计数刻意复用 autocompact 的 `tokenCountWithEstimation`，保证两者口径一致（`:136-137`）。

### Fork + 权限锁（`sessionMemory.ts:272-350, 460-482`）

- 抽取跑在 **forked 子 agent**（`runForkedAgent`），隔离 context、不污染主线 prompt cache；只在 `repl_main_thread` 跑（子 agent/teammate 不跑）。
- `createMemoryFileCanUseTool(memoryPath)`：**只允许 `Edit` 且 `file_path === memoryPath`**，其它工具/路径一律 `deny`。子 agent 物理上只能改这一个文件——这是 [[ground-truth-via-tools]] 式的能力隔离，不靠 prompt 自律。
- 抽取 prompt（`prompts.ts:43-81`）反复强调：本指令**不属于用户对话**、笔记里不许提"note-taking"、只用 Edit 改笔记然后停、section 标题与斜体说明**逐字保留**、`Current State` **必须更新**（"critical for continuity after compaction"）。

### 预算管理（`prompts.ts:8-9, 164-196, 256-324`）

- `MAX_SECTION_LENGTH = 2000` tokens/section、`MAX_TOTAL_SESSION_MEMORY_TOKENS = 12000`。
- 超预算时 `generateSectionReminders` 把"哪些 section 超了、必须压缩"动态拼进 prompt；插入压缩流时 `truncateSessionMemoryForCompact` 在 section 边界硬截断，防笔记吃光 post-compact 预算。

### 压缩集成——本机制的真正落点（`compact/sessionMemoryCompact.ts:505-614`）

`trySessionMemoryCompaction`：
- 先 `waitForSessionMemoryExtraction()`（最多等 15s，stale>1min 不等，`sessionMemoryUtils.ts:89-105`）确保后台抽取落盘。
- `summary.md` 不存在或仍是**空模板**（`isSessionMemoryEmpty`）→ 返回 null 回退 legacy。
- 正常：按 `lastSummarizedMessageId` 切边界，保留其后消息 + 笔记当摘要；**恢复的会话**（无边界）：保留笔记当摘要、消息从头算。
- 文件头注释直白：`EXPERIMENT: Session memory compaction`。

### 读回路径（OBS）

除压缩外：`services/awaySummary.ts:38`（离开摘要）、`skills/bundled/skillify.ts:181`、`/summary` 命令手动触发（`sessionMemory.ts:387` `manuallyExtractSessionMemory`，绕过阈值）。

## 与文章 §② 的关系（INF —— 本页填的就是这个 wiki 缺口）

[context-engineering-anthropic.md](../insights/context-engineering-anthropic.md) §② 把结构化笔记描述为"**agent 自己**定期写 NOTES.md、维护 to-do list"（举例 Claude 玩宝可梦、Claude Code 的待办清单）。SessionMemory 落地后有两点**关键偏离**值得标注（INF）：

1. **不是 agent 自导，而是系统编排的后台抽取**。主 agent 不知道、也不参与；是 post-sampling hook fork 出一个**权限锁死**的子 agent 去写。文章举的"Claude Code 创建待办清单"其实是**另一个机制**——`TodoWriteTool`（agent 自导、在 context 内的工作记忆）。

2. **首要目的是喂压缩，不是"以后读回来"**。文章强调"notes 之后被拉回 context"；SessionMemory 主线是**预构建压缩摘要**——它存在的理由（`initSessionMemory` 仅在 `isAutoCompactEnabled()` 时注册，`:359-371`）就是给压缩用。

> **INF**：因此 Claude Code 里"结构化笔记"至少有**两种 flavor**：
> - **TodoWrite** —— agent 自导、in-context、工作记忆（对应文章举例）
> - **SessionMemory** —— 系统编排、out-of-context、压缩预计算（本页）
>
> 不要把 SessionMemory 等同于文章里"public beta 的 memory tool"——那是 Claude Developer Platform 的**用户可见 API 工具**，本机制是 CLI 内部管线，[命名相近但不是一回事]（防 [[predictable-hallucination-hardcode]] 式命名误导）。

## 关键设计决策

| 决策 | 证据 | 含义 |
|---|---|---|
| 抽取放后台 fork，不打断主线 | `sessionMemory.ts:1-5, 318` | 主对话零延迟；隔离 context 不污染主 prompt cache（OBS） |
| 子 agent 只能 Edit 一个文件 | `:460-482` deny-all-else | 能力隔离替代信任——即便抽取 agent 跑飞也只能动那一个文件（OBS） |
| token 门槛永远必需 | `:165-167` 注释 | 防"工具调用多但内容没长"时空抽取，省 fork 成本（OBS） |
| 复用 autocompact 的 token 口径 | `sessionMemoryUtils.ts:19-26` | 两个特性对"上下文多大"看法一致，避免一个触发一个不触发（OBS） |
| 笔记当摘要，绕过同步 LLM 总结 | `sessionMemoryCompact.ts:505-614` | **把昂贵的压缩摘要分摊到后台增量做**——压缩时摘要已就绪（INF：这是相对 autocompact 的核心优化） |
| 模板斜体说明逐字保留 | `prompts.ts:55-79` | 模板即"结构契约"，跨多次抽取保持 section 稳定，便于增量更新（INF） |
| `Current State` 强制更新 | `prompts.ts:69` | 压缩后续命的关键单点——其它 section 可跳过，这条不行（OBS） |

## 3P vs Ant 状态（OBS + INF）

- 门控是 **GrowthBook 运行时 flag** `tengu_session_memory`（默认 `false`，`sessionMemory.ts:80-82`），**不是** `bun:bundle` 的 `feature()` 编译期 DCE。
- **INF**：因此代码**确实打进 3P 公开包**（不像 [[context-compression-cascade]] 的 Snip/Collapse 被 DCE 删除），但默认 flag 关闭；压缩集成文件自标 `EXPERIMENT`。也就是说 3P 用户**默认拿不到**，但 Anthropic 可远程灰度开启。**SPEC**：是否已对部分 3P 用户开启，需实测，本页不下结论。

## 失效模式与边界

| 场景 | 后果（OBS/INF） |
|---|---|
| 后台抽取还没落盘就触发压缩 | `waitForSessionMemoryExtraction` 等最多 15s；超时/stale 就继续，可能用到稍旧的笔记（OBS） |
| `lastSummarizedMessageId` 在当前消息里找不到（消息被改） | 回退 legacy 压缩（`:554-560`）——切不准边界就不冒险（OBS） |
| 笔记超 12K 总预算 | 抽取 prompt 注入"必须压缩"指令 + 插入时硬截断；优先保 `Current State` / `Errors & Corrections`（OBS） |
| 抽取子 agent 试图调别的工具 | 权限层 deny，只能 Edit 笔记文件（OBS） |
| flag 关闭（3P 默认） | hook 不注册/早退，压力全下沉到 [[context-compression-cascade]] ④ autocompact（INF） |

## 可迁移性（INF/SPEC —— +1 怀疑度）

任何长会话 Agent 都要在"压缩时才同步总结"和"后台增量维护摘要"之间选：

1. **把压缩摘要前移到后台增量做** —— 不要等满了才同步调 LLM 总结（那是 [[context-compression-cascade]] 的兜底）。在自然停顿点用便宜的后台抽取增量更新一份结构化摘要，压缩时直接拿。
2. **固定模板 = 增量更新的契约** —— 让笔记有稳定 section，抽取 agent 每次只补内容不改结构，才能"增量"而非"重写"。
3. **抽取 agent 权限锁死单文件** —— 后台自动跑的 agent 风险最高，用能力隔离（只能 Edit 一个文件）而非 prompt 自律兜底，呼应 [[ground-truth-via-tools]]。
4. **写/用解耦 + 接缝指针** —— 用一个 `lastSummarizedMessageId` 当写循环与用循环的接缝，避免两边各算边界。
5. **区分两种记忆 flavor** —— in-context 工作记忆（TodoWrite 式）≠ out-of-context 压缩预算（SessionMemory 式），别混用一套机制硬扛两种需求。

> 对 leto-ai 电商 Agent：跨多轮的订单处理会话 → 后台增量维护"当前订单状态/已确认决策/失败重试"结构化笔记，会话压缩时直接当摘要，而非每次满了重新 LLM 总结。

## 进一步追问的钩子

1. `sessionMemoryCompact.ts` 还有 ~500 行未细读（`calculateMessagesToKeepIndex`、min/maxTokens 40K 上限的边界逻辑）——压缩保留窗口的精确算法。
2. `tengu_session_memory` 实际灰度范围（3P 是否已开）——需实测或 BQ 数据，当前 SPEC。
3. 与 [[plan-mode-state-machine]] 的 plan 文件、`agentMemory.ts` 的子 agent memory 三者关系——CC 里到底有几套"记忆"，边界在哪。

## 关联

- 上层/协同：[[context-compression-cascade]]（本机制是其 ④ autocompact 的预计算旁路）、[context-engineering-anthropic.md](../insights/context-engineering-anthropic.md)（官方 §② 结构化笔记，本页是其落地核查）
- 同源思路：[[ground-truth-via-tools]]（抽取子 agent 的权限锁是能力隔离的应用）、[[predictable-hallucination-hardcode]]（"session memory" vs "memory tool" 命名勿混）
- 相关实体：`services/SessionMemory/{sessionMemory,prompts,sessionMemoryUtils}.ts`、`services/compact/sessionMemoryCompact.ts`、`utils/permissions/filesystem.ts:261-270`、`utils/forkedAgent.ts`（runForkedAgent）
- 综合分析：[1.CORE_LOOP.md](../1.CORE_LOOP.md)（post-sampling hook 在主循环的位置）、[3.MULTI_AGENT.md](../3.MULTI_AGENT.md)（forked subagent 机制）
