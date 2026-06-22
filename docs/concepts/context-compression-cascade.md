# Context Compression Cascade（上下文压缩四级级联）

> Claude Code 在主循环里串了 4 级递增的压缩——**先便宜后贵、先无损后有损**。每一级尝试把 token 拉到下一级阈值之下，能跳过就跳过，让最贵的 LLM-summary（autocompact）只在万不得已时跑。这是 [[multi-tier-degradation]] 模式在 token 经济场景的应用。

## 核心机制

```
每轮 query 主循环：
       │
       ▼
┌─────────────────────────────────────────────────────┐
│ ① Snip      丢弃旧 snippet（保留消息结构）            │  HISTORY_SNIP feature gate
├─────────────────────────────────────────────────────┤
│ ② Microcompact   清掉指定工具的 tool_result 内容       │  always tries (3P fallback to no-op)
├─────────────────────────────────────────────────────┤
│ ③ Context Collapse  把段落折叠成 summary（projection）│  CONTEXT_COLLAPSE feature gate
├─────────────────────────────────────────────────────┤
│ ④ Autocompact   LLM 总结整个历史（最后兜底）          │  always
└─────────────────────────────────────────────────────┘
       │
       ▼
   API 调用
```

每一级跑完都重新算 token 预算。如果已经低于 autocompact 阈值，autocompact 跳过——保留更细粒度的上下文。

## 在 Claude Code 中的体现

### 主循环编排（`query.ts:396-468`）

四级**强制按顺序**串行：

```ts
// ① Snip
if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
  snipTokensFreed = snipResult.tokensFreed   // ← 传给 autocompact 校准阈值
}

// ② Microcompact
const microcompactResult = await deps.microcompact(...)
messagesForQuery = microcompactResult.messages

// ③ Context Collapse — projection-based, 在 autocompact 前跑
if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
  const collapseResult = await contextCollapse.applyCollapsesIfNeeded(...)
  messagesForQuery = collapseResult.messages
}

// ④ Autocompact (最后兜底)
const { compactionResult } = await deps.autocompact(...)
```

注释里点出关键 ordering 决策（`query.ts:428-432`）：

> Project the collapsed context view and maybe commit more collapses. **Runs BEFORE autocompact so that if collapse gets us under the autocompact threshold, autocompact is a no-op and we keep granular context** instead of a single summary.

—— 整条级联的核心就是这条："越往后越贵越糙，能用早期的就别用后期的"。

### 各级详解

#### ① Snip（`feature('HISTORY_SNIP')`）

**做什么**：丢弃历史消息里的旧 snippet 内容，保留消息结构和 ID。  
**关键设计**：保留 **protected-tail assistant**——最后一条 assistant message 完整保留，因为 token usage 是从它的 usage 字段读的，不能动。

注释明示这影响其他级（`query.ts:397-399`）：

> snipTokensFreed is plumbed to autocompact so its threshold check reflects what snip removed; tokenCountWithEstimation alone can't see it (reads usage from the protected-tail assistant, which survives snip unchanged).

**释放 token 的能力**通过 `snipTokensFreed` 显式传给 autocompact——后者计算阈值时减掉这部分，否则会重复触发。

**3P 状态**：feature flag DCE'd 掉了——`snipCompact.ts` 在公开 npm 包里**根本不存在**，只在 ant 内部构建中。

#### ② Microcompact（`microCompact.ts`）

**做什么**：清掉指定工具的旧 `tool_result` 内容，留 `[Old tool result content cleared]` 占位符。

**只对 8 类工具生效**（`microCompact.ts:39-49`）：

```ts
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME, ...SHELL_TOOL_NAMES,
  GREP_TOOL_NAME, GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME, WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME,
])
```

**两条触发路径**：

1. **Time-based**（`microCompact.ts:267-270`）—— 主线程上次 assistant 已超过阈值时间，**服务端 prompt cache 已过期**——既然要 rewrite 整个 prefix，不如趁机清掉旧 tool_result 节省 rewrite 量。这是个**搭便车**优化：cache miss 已成定局，cleanup 顺手做免费的。
2. **Count-based**（cached MC，ant-only）—— 累积工具调用 N 个后清最老的。

**Cached microcompact**（ant-only，`feature('CACHED_MICROCOMPACT')`）的关键设计：

> Does NOT modify local message content (cache_reference and cache_edits are added at API layer)

通过 Anthropic API 的 cache-edit 特性**远程删除**而不动本地——保留 prompt cache 命中。这是只有 first-party API 才支持的优化。

**3P 状态**：legacy 客户端 microcompact 路径已被砍——注释明说：

> Legacy microcompact path removed — `tengu_cache_plum_violet` is always true. For contexts where cached microcompact is not available (external builds, non-ant users, unsupported models, sub-agents), no compaction happens here; **autocompact handles context pressure instead**.

也就是说：**3P 用户 ② 直接 no-op**，所有压力下沉到 ④ autocompact。这是 [[flag-vs-hardcode]] 里讲过的 graduation 案例——`tengu_cache_plum_violet` 实验成功后硬编码为 always true，旧路径删除。

#### ③ Context Collapse（`feature('CONTEXT_COLLAPSE')`）

**做什么**：把历史段落折叠成 summary blocks，但**原始消息保留在 REPL 的内存数组里**。

**关键设计：projection-based 而非 destructive**（`query.ts:432-439`）：

> Nothing is yielded — the collapsed view is a read-time projection over the REPL's full history. Summary messages live in the collapse store, not the REPL array. **This is what makes collapses persist across turns: projectView() replays the commit log on every entry.**

**两个数组分离**：
- REPL 主数组：所有原始消息，永不删
- Collapse store：折叠 commit log（"在 turn N 把消息 [x,y] 折成 summary S"）

每次进入 query 都跑 `projectView()` 重放 commit log——折叠状态**幂等**。这也是为什么折叠能跨 turn 保持：projection 是无状态的。

**跟 ② Microcompact 的根本区别**：microcompact 真的改了消息内容（清空 tool_result）；collapse **不动原始消息**，只是给主循环一个"看起来折叠了"的视图。

**3P 状态**：feature flag DCE'd 掉，`contextCollapse/` 目录在公开包里不存在。

#### ④ Autocompact（`autoCompact.ts`）

**做什么**：超过阈值时调 Claude **总结整个会话历史**，替换原消息流。

**阈值计算**（`autoCompact.ts:62, 72-91`）：

```ts
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000   // 基于 p99.99 实测
```

注释里 `MAX_OUTPUT_TOKENS_FOR_SUMMARY` 的 magic number 来自实测：

> Based on p99.99 of compact summary output being 17,387 tokens.

**用真实数据校准的 buffer**——不是拍脑袋的 20K。

**Circuit breaker**（`autoCompact.ts:67-70`）：

```ts
// Stop trying autocompact after this many consecutive failures.
// BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272)
// in a single session, wasting ~250K API calls/day globally.
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

历史包袱注释——加这个 circuit breaker 之前，**全球每天 25 万次浪费的 API 调用**。connecting back to [[circuit-breaker]] 概念。

**Compaction summary prompt**（`compact/prompt.ts`）—— LLM 总结时强制要求结构化输出：用户的 primary request、关键技术概念、修改的文件清单、当前工作状态、可选的下一步等。最后一段：

> If you need specific details from before compaction (like exact code snippets, error messages, or content you generated), read the full transcript at: {transcriptPath}

**逃生口设计**——总结后历史虽然丢了，但 transcript 还在磁盘。模型如果发现总结里缺信息，知道怎么去找原文。

### 成本/失真梯度

| 级别 | 计算成本 | 失真程度 | 对 prompt cache | 跨 turn 持久 |
|---|---|---|---|---|
| ① Snip | O(n) 本地 | 丢 snippet 内容，结构留 | 完全 invalidate（rewrite prefix）| 是（消息真改了）|
| ② Microcompact (time-based) | O(n) 本地 | tool_result 清成占位符 | 已经过期，搭车做（成本零）| 是 |
| ② Microcompact (cached, ant-only) | API cache_edit | 同上 | **保留**（cache_edit 特性）| 是 |
| ③ Context Collapse | O(n) 本地 + projection | 段落折叠成 summary | 取决于 projection 实现 | 是（commit log 重放）|
| ④ Autocompact | **LLM API call**（贵）| 整段历史 → 单一 summary | 完全 invalidate | 是（替换原消息流）|

代价从近零（snip）到一次 LLM 调用（autocompact）跨 4 个数量级。

## 公开 3P vs Ant 内部的差异

这是个值得圈出的发现——**4 级级联的实际可用版本因构建而异**：

| 级别 | 3P 公开版 | Ant 内部 |
|---|---|---|
| ① Snip | DCE 掉（无文件）| 有 |
| ② Microcompact | **no-op**（legacy 已删）| Cached MC（API cache_edits）|
| ③ Context Collapse | DCE 掉（无目录）| 有 |
| ④ Autocompact | **唯一可用**| 有 |

也就是说**3P 用户实际上只有 autocompact**——其他 3 级在公开包里要么被 DCE 删掉要么 no-op。注释明说："autocompact handles context pressure instead"。

设计含义：**Ant 用 4 级精细控制，3P 是兜底单级**。这跟[[flag-vs-hardcode]] 的策略一致——内部先实验，公开版保守用稳定路径。

## 关键设计决策

### 1. Time-based microcompact 搭便车 cache miss

`microCompact.ts:261-266` 注释：

> If the gap since the last assistant message exceeds the threshold, the server cache has expired and the full prefix will be rewritten regardless — so content-clear old tool results now, before the request, to shrink what gets rewritten.

**已经要 rewrite 了，不如顺手清**。Cache miss 是无可避免的成本；用这一次 cache miss 把消息也瘦身了，下一次反而 cache 命中更好。这是个**用必然成本换额外收益**的优化。

### 2. Projection vs destructive 的根本区别

② Microcompact 改原始消息（destructive），③ Context Collapse 不改（projection）。两者解决类似问题但不同思路：

- 改原始消息：简单但**不可逆**，且每个 turn 都要重新决定要不要改
- Projection：**幂等**——commit log 一旦写入，每次 projectView 都重现同样视图

这是个有意思的取舍：projection 复杂度更高但可恢复；destructive 简单但需要人为决定时机。Claude Code **两种都用**——不同级别用不同策略。

### 3. snipTokensFreed 的显式数据流

`query.ts:466` 把 ① snip 的释放量传给 ④ autocompact。为什么不让 autocompact 自己重新算？因为它读的是 protected-tail assistant 的 usage 字段——那个字段不会因为前面的消息被 snip 而更新。

**这是个层间通信**——下游级别需要知道上游级别**做了什么**才能做正确决策。如果不传 snipTokensFreed，autocompact 会以为 token 还很高，重复触发。

### 4. Autocompact circuit breaker 是事故驱动的

注释直接附上数据："1,279 sessions had 50+ consecutive failures (up to 3,272) in a single session, wasting ~250K API calls/day globally"。

修复就是**3 次连续失败停止**——超过这个次数说明 context 已经不可救药，再试只是浪费。这跟通用 [[circuit-breaker]] 思路同源。

## 设计原则

| 原则 | 含义 |
|---|---|
| **代价/失真梯度排序** | 4 级按"近零本地处理 → LLM 调用"递增成本，按"无损 → 大段总结"递增失真。**便宜的先尝试**，能用早期就别用后期 |
| **越早期，越无损** | Snip 只丢内容不改结构；Microcompact 只改特定工具的 result；Collapse 仅 projection；Autocompact 全替换。**保留细粒度上下文优于压缩**——只在必须时压缩 |
| **Projection vs destructive 各有适用** | 短命数据（snip / microcompact）destructive 简单；长命数据（collapse）projection 幂等可恢复 |
| **搭便车必然成本** | Time-based microcompact 在 cache miss 已成定局时做——**用 0 边际成本** 换额外收益 |
| **跨级显式数据流** | snipTokensFreed 显式传给 autocompact——下游不能默默假设上游什么也没做 |
| **Circuit breaker 防绝望循环** | Autocompact 失败 3 次就停——基于真实事故数据（每天浪费 25 万次 API 调用） |
| **Empirical sizing** | `MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000` 来自 p99.99 实测（17,387 tokens）；不拍脑袋 |
| **Summary 留逃生口** | Autocompact 提示里附上 transcriptPath——总结丢了细节，但模型可以自查原文 |
| **公开版保守，内部版精细** | 3P 只跑 autocompact 兜底；Ant 4 级全开。**不把实验性优化塞给外部用户** |

## 失效模式与边界

| 失效场景 | 后果 |
|---|---|
| Snip 错误丢了关键 snippet | 主 Agent 后续推理基于不完整上下文 |
| Microcompact 清的 tool_result 后续还要用 | 模型看到占位符 `[Old tool result content cleared]`，知道要重新调工具 |
| Context Collapse 折叠了不该折的段 | projection 可恢复——但 Agent 看不到原文，需要主动展开 |
| Autocompact summary 漏关键细节 | 兜底机制：transcriptPath 让模型查原文 |
| Autocompact 连续 3 次失败 | Circuit breaker 触发，UI 提示用户手动 `/compact` 或新会话 |
| Cached MC 跟 cache miss 冲突 | Time-based microcompact 优先短路 cached MC（"editing assumes a warm cache, and we just established it's cold"） |

## 跟其他概念的对比

| 概念 | 联系 | 区别 |
|---|---|---|
| [[multi-tier-degradation]] | 同源思路：递增梯度先轻后重 | 多级降级处理失败恢复；本概念处理 token 预算 |
| [[circuit-breaker]] | 直接应用——autocompact 3 次失败就停 | — |
| [[result-delivery-guarantee]] | tool_result 大输出溢出磁盘也是上下文管理；微 cleanup 跟它互补 | 那个是单 result 的输入侧；本概念是跨 result 的累积侧 |
| [[flag-vs-hardcode]] | `tengu_cache_plum_violet` 已毕业（"is always true"）的注释正是 microcompact 这条路径 | — |
| [[ground-truth-via-tools]] | 压缩后大输出"必须重新 Read"是事实链的应用 | — |

## 可迁移性

任何长会话 LLM 系统都会遇到 token 预算问题：

1. **不要只做"满了就 LLM-summarize"** — 那是兜底不是主策略。先用便宜的 destructive 操作（清旧 tool_result / 截断）把压力降下来
2. **跨级显式数据流** — 如果下游级别需要知道上游做了什么（释放了多少 token），别让它自己算，显式传
3. **搭便车必然成本** — 检查系统里有没有"反正要花的钱"（cache miss、网络重连、状态同步），把额外清理任务挂上去
4. **真实数据校准 buffer** — 不要拍脑袋写 `20_000`，跑一遍实际负载看 p99.99
5. **Circuit breaker 在哪都需要** — 任何会失败的循环操作（重试、压缩、轮询）都需要"绝望就停"的硬阈值
6. **Projection vs destructive 选其一** — 长命数据 projection；短命数据 destructive。**别两种混用**

针对 leto-ai 的电商 Agent 场景：
- Order history 长度无界 → 需要类似的级联压缩（最近 N 单详情 + 更老的 LLM-summary）
- 商品详情页 tool_result 累积 → microcompact 思路：保留最近查的、清掉旧的占位符
- Verifier Agent 跨 session 的 verification 历史 → context collapse 的 projection 思路：用 commit log 重放

## 进一步追问的钩子

1. **Snip 具体识别什么"snippet"** — 文件 snippet？代码段？工具结果片段？feature DCE 掉了实现，需要从 history 行为反推
2. **Cached microcompact 的 cache_edits API** — Anthropic API 的非公开特性。能远程删除特定 tool_result 而不 invalidate prompt cache，技术细节如何
3. **`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` 的实证数据**（注释说 1,279 sessions 50+ 连失，3,272 是单 session 最差）—— 这个数据如果能拿到详情，可以反推 prompt_too_long 的失败模式
4. **3P 用户的 token 预算管理实际效果** — 只有 autocompact 兜底意味着达到阈值前 0 缓冲——会不会更频繁触发 autocompact？性能数据在哪？
5. **Context Collapse 的 commit log 形态** — projection 模型听起来像 git，commit log 长什么样？冲突解决？

## 关联

- 上层概念：（无；本概念是上下文管理的顶层）
- 反面对照：[[multi-tier-degradation]]（失败恢复版本）vs 本概念（token 经济版本）；[[circuit-breaker]] 是本概念中 autocompact 失败处理的具体应用
- 协同机制：[[flag-vs-hardcode]]（`tengu_cache_plum_violet` 毕业是 microcompact 路径切换的样本）；[[result-delivery-guarantee]] 的磁盘溢出是输入侧管理，本概念是累积侧管理
- 相关实体：`services/compact/`（4 级模块所在）、`query.ts:396-468`（编排主循环）、`services/compact/microCompact.ts`（②）、`services/compact/autoCompact.ts`（④）、`services/compact/prompt.ts`（autocompact 总结 prompt）、`services/compact/timeBasedMCConfig.ts`（time-based 触发）
- 综合分析：[1.CORE_LOOP.md](../1.CORE_LOOP.md)（query.ts 主循环结构）
