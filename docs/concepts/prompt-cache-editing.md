# Prompt Cache Editing API（提示缓存编辑 API）

> Anthropic 一个**非公开的 first-party-only beta**：通过 `cache_edits` 消息块声明对已缓存 prefix 里的 `tool_result` 进行删除。Claude Code 的 cached microcompact（[[context-compression-cascade]] 第 ② 级的 ant-only 路径）建立在这条 API 上。本页**从 client 代码反推**——服务端实现不可见，所有"为什么这样设计"的解释都是推断。

> **⚠ 认知边界**：本页对 API **形态**的描述（type 定义、placement 约束、5 层 gate）来自 client 源码，可信。本页对**服务端语义**的描述（"cache 不变"、"deletion 是 per-request transient"、"逻辑/物理删除分离"）是从 client 重发模式倒推的**推断**——服务端实现可能完全不同（见下面"替代解释"段）。读者需带这个折扣。

## 核心机制（client 行为，OBS）

```
单次请求里 client 发送（OBS — 直接源自代码）：
  { type:'cache_edits', edits:[{type:'delete', cache_reference:'tool_use_xyz'}] }
              ↓
  并把这块"钉"在它最初被插入的消息位置（pinCacheEdits）
              ↓
  下一轮请求 client 再次发送同一块到同一位置
              ↓
  Turn 3、4...持续 re-send（pinnedEdits 列表累积）
```

**服务端处理（INF/SPEC，需带折扣）**：

> 推断：服务端把 `cache_edits` 块解读为"对已 cached prefix 的删除指令"，使本次请求看到的上下文不包含被删的 tool_result，但**保留 cache prefix 命中**（不 invalidate）。
>
> 依据：注释 `cacheDeletionsPending: cache reads will legitimately drop — this is expected, not a break` 表明 cache 仍然有 read（说明 prefix 至少部分命中），且 read 量"合法下降"（说明被删的内容确实从结果中扣掉了）。
>
> **替代解释**（同样跟 client 行为兼容）：服务端可能定期物理重建 cache（5min/1hour TTL），重建时把 cache_edits 累加应用；或者服务端记录删除清单 + client re-send 是冗余幂等保险。本页采用"per-request transient"模型是**最简单的解释**，但未必正确。

## 在 Claude Code 中的体现

### API 契约（从 client 代码反推）

`services/api/claude.ts:3052-3055`：

```ts
type CachedMCEditsBlock = {
  type: 'cache_edits'
  edits: { type: 'delete'; cache_reference: string }[]
}
```

**两个新增 API 元素**：

| 元素 | 位置 | 作用 |
|---|---|---|
| `cache_edits` block | 用户消息的 content array | 声明本次请求要删的 tool_result |
| `cache_reference` 字段 | `tool_result` block 的属性 | 让服务端能用 `tool_use_id` 寻址要删的 result |

`cache_reference` 由 client 在序列化时自动加（`claude.ts:3201-3203`）：

```ts
msg.content[j] = Object.assign({}, block, {
  cache_reference: block.tool_use_id,   // ← 用 tool_use_id 作为缓存里的寻址 key
})
```

### Beta 头与启用矩阵

启用条件层层收紧（`claude.ts:1675-1689`）：

```ts
const useCachedMC =
  cachedMCEnabled &&                              // GrowthBook flag 启用
  getAPIProvider() === 'firstParty' &&            // 仅 first-party API
  options.querySource === 'repl_main_thread'      // 仅主线程会话
```

外加：
- `feature('CACHED_MICROCOMPACT')` —— compile-time DCE flag（公开包不打包这条路径）
- `isCachedMicrocompactEnabled()` —— GrowthBook runtime flag
- `isModelSupportedForCacheEditing(model)` —— 模型白名单（不是所有模型支持）

被排除的场景：
- **3P providers**（Vertex / Bedrock）—— beta 不可用
- **subagents**（querySource 非 main_thread）—— forked agent 不用
- **不支持的模型** —— 只有特定模型支持
- **3P 用户**（feature DCE 在公开 build 里删整段路径）

这是一个**多层 gate**——任何一层 false 就退化到 [[context-compression-cascade]] 的兜底 autocompact。

### Sticky-on Latch：header 与 body 解耦

最微妙的设计（`bootstrap/state.ts:234-237`）：

```
cacheEditingHeaderLatched: boolean | null

// Sticky-on latch for the cache-editing beta header. Once cached
// microcompact is first enabled, keep sending the header so mid-session
// GrowthBook/settings toggles don't bust the prompt cache.
```

**latch 锁住**：beta 头只要本会话出现过一次，就持续发送，**不论后续 GrowthBook flag 是否翻转**。

为什么？因为 prompt cache key 包含 beta header set——header 进出会导致 cache miss。如果 GrowthBook 中途把 cached MC flag 关掉，header 跟着消失 → cache miss。Latch 让 header 跟会话同生死。

但 body 内容（实际是否发 `cache_edits` 块）依然跟随动态 flag：

```ts
// 注释（claude.ts:1672-1674）：
// Cache editing beta: header is latched session-stable; useCachedMC
// (controls cache_edits body behavior) stays live so edits stop when
// the feature disables but the header doesn't flip.
```

**Header latched + body dynamic** —— 一个跟会话粒度，一个跟 flag 粒度，两条独立信号。

### 插入位置的硬约束

`cache_edits` 块在用户消息 content 里的位置不能随便放（`utils/contentArray.ts`）：

| 规则 | 原因 |
|---|---|
| 必须**在最后一个 tool_result 之后** | API 解析顺序要求 |
| 如果会成为最后一个块，自动追加 `{ type: 'text', text: '.' }` 占位 | "some APIs require the prompt not to end with non-text content" |
| `cache_reference` 块必须**严格在 last cache_control marker 之前** | API 要求 cache_reference "before or on" the last cache_control，"用 strict 'before' to avoid edge cases where cache_edits splicing shifts block indices" |

注释里 "strict before" 那一段揭示一个**防御性偏保守**的取舍：API 允许 ≤，但 client 选择 < 来防止"切片移位"边缘情况。

### Pinned Edits：Client 重发模式（OBS）

`claude.ts:3127-3162`：

```ts
// Re-insert all previously-pinned cache_edits at their original positions
for (const pinned of pinnedEdits ?? []) {
  // ... 把以前发过的 cache_edits 块重新插回原位
}

// Insert new cache_edits into the last user message and pin them
if (newCacheEdits && result.length > 0) {
  // ... 插入新的 cache_edits + pin 到该位置
  pinCacheEdits(i, newCacheEdits)
}
```

**Client 端的可观察行为**：每个 cache_edits 块"钉"在它最初被插入的消息位置，后续每次请求都重新发。

```
Turn 1: cache_edits = [delete tool_xyz]
Turn 2: cache_edits = [delete tool_xyz]                          (re-send)
Turn 3: cache_edits = [delete tool_xyz, delete tool_abc]         (累积新增)
...
```

**为什么 client 重发？— 几种可能解释**：

| 假设 | 隐含的服务端模型 | 反驳 / 困难 |
|---|---|---|
| **A. Per-request transient（本页起初的解读）** | 服务端 cache 不变，每个请求基于 cache_edits 投影 | 简洁优雅，但**没有直接证据**——只是兼容观察到的行为 |
| **B. Cache TTL 重建** | 5min/1hour TTL 到期重建 cache，重建时累加 cache_edits | 需要 client 重发才能让重建后的 cache 也排除这些。也兼容观察 |
| **C. Belt-and-suspenders** | 服务端持久化删除，但 client 重发是冗余幂等 | 多余但不矛盾 |
| **D. 重建跨 process 容错** | 服务端可能在不同 worker 间漂移，新 worker 没有删除记录 | 也兼容 |

**结论**：观察到的事实是 **client 维护一个 `pinnedEdits` 列表，每轮重新声明**。"为什么"层面有多种合理解释，本页起初采用的"per-request transient"模型只是其中之一。

> **延伸思考**：如果你要迁移这套思路（详见后面"可迁移性"段），可信的部分是 **"client 显式维护逻辑状态 + 重发"** 这个 pattern；不可信的部分是 **"因此服务端 stateless"** 这个推论。前者作为模式可迁移；后者作为"为什么"建议留作 open question。

### 跨块去重防 server error

`claude.ts:3112-3125`：

```ts
const seenDeleteRefs = new Set<string>()

const deduplicateEdits = (block: CachedMCEditsBlock) => {
  const uniqueEdits = block.edits.filter(edit => {
    if (seenDeleteRefs.has(edit.cache_reference)) {
      return false   // ← 跨所有 cache_edits 块全局去重
    }
    seenDeleteRefs.add(edit.cache_reference)
    return true
  })
  return { ...block, edits: uniqueEdits }
}
```

同一 `tool_use_id` 跨多个 cache_edits 块出现会被服务端拒。client 主动 dedupe 防止 round-trip 失败。

### Cache miss 的"合法下降"

cache_edits 删除会让 cache_read tokens 合法地降低——本来要看到的 tool_result 不在了。`promptCacheBreakDetection.ts:65-67`：

```ts
/** Set when cached microcompact sends cache_edits deletions. Cache reads
 *  will legitimately drop — this is expected, not a break. */
cacheDeletionsPending: boolean
```

**这是个观测系统的细节**：cache break detector 默认看到 cache_read 大降会告警（提示用户"上下文配置变了 → cache miss"）。但 cache_edits 是合法地砍内容——必须打标记让告警系统知道"这是预期行为"。

## 内部基础设施泄露

`addCacheBreakpoints` 的注释（`claude.ts:3078-3088`）暴露了 Anthropic 内部推理基础设施细节，**这些术语在公开文档里没有**：

> Exactly one message-level cache_control marker per request. **Mycro's** turn-to-turn eviction (**page_manager/index.rs: Index::insert**) frees local-attention KV pages at any cached prefix position NOT in **cache_store_int_token_boundaries**. With two markers the second-to-last position is protected and its locals survive an extra turn even though nothing will ever resume from there — with one marker they're freed immediately. For fire-and-forget forks (skipCacheWrite) we shift the marker to the second-to-last message: that's the last shared-prefix point, so the write is a no-op merge on **mycro** (entry already exists) and the fork doesn't leave its own tail in the **KVCC**.

可以反推：

| 术语 | 推测含义 |
|---|---|
| **Mycro** | Anthropic 的推理引擎代号（"page_manager/index.rs" 是 Rust 文件路径，说明 mycro 是 Rust 写的） |
| **KV pages** | KV cache 的分页存储 |
| **page_manager / Index::insert** | 管理 KV cache 页的数据结构操作 |
| **cache_store_int_token_boundaries** | "受保护的"缓存 token 边界集合——cache_control marker 落在这里 |
| **KVCC** | KV Cache Coordinator/Container？（行为像 LRU 总管） |

这些术语**仅在内部代码注释里**——意味着：
1. Claude Code client 跟服务端**深度耦合**，client 作者熟悉 Mycro 内部结构
2. 一些"奇怪的 client 行为"（比如 `markerIndex = messages.length - 2` for forks）是为了**配合服务端实现**，不是 API 文档要求的
3. 公开 SDK 用户**看不到这些 magic 数字背后的理由**

## 设计原则

> 下表里 **(OBS)** 标注的项是从代码直接观察的行为；**(INF)** 是合理推断；**(SPEC)** 是建立在前述弱推断上、需带折扣。

| 原则 | 置信度 | 含义 |
|---|---|---|
| **Header sticky-on，body dynamic** | OBS | 影响 cache key 的部分（header）会话内固定，控制行为的部分（body）跟随 flag。两条信号分开。注释明说 |
| **多层 gate 收紧适用范围** | OBS | feature DCE → GrowthBook flag → first-party → main-thread → 模型白名单——5 层全过才用 |
| **跨块全局去重** | OBS | 跨多个 cache_edits 块全局保证 tool_use_id 唯一 |
| **Strict before 比 ≤ 保守** | OBS | API 允许 cache_reference 在 cache_control "before or on"——client 选 strict before 防边缘 |
| **Cache miss 的合法/非法分类** | OBS | cacheDeletionsPending 标记防误报 |
| **Pinned + re-send 模式** | OBS | client 维护逻辑删除清单，每轮请求重发 |
| **Client/server 耦合到 server 实现路径** | OBS | client 注释直接引用 `page_manager/index.rs`——明显的实现耦合 |
| **Cache 状态保持纯净，删除是投影** | **SPEC** | 服务端 cache 不变；删除是 per-request 投影。**这是从 client 重发倒推的——见上文替代解释，未必正确** |
| **不让服务端记状态** | **SPEC** | 同上推断；如果服务端实际记录删除（仅让 client 重发做幂等），这条原则就不成立 |

## 跟其他概念的对比

| 维度 | 普通 prompt cache | cache_edits API |
|---|---|---|
| Cache invalidation 粒度（OBS） | 整段 prefix 改了就失效 | 选择性删除（cacheDeletionsPending 标记说明 prefix 至少部分仍 read 命中） |
| 删除持久性（SPEC） | 自然 TTL（5min/1hour） | client 维护 pinned 清单 + 重发；服务端语义不可见 |
| Client 复杂度（OBS） | 极低 | 高——pinned edits 维护、跨块去重、placement 约束 |
| 可用性（OBS） | 公开 | first-party 主线程 + 特定模型 + 内部 flag |

## 失效模式与边界

| 失效场景 | 后果 |
|---|---|
| Beta header 被 mid-session 移除 | Prompt cache miss——所以才有 sticky-on latch |
| `cache_edits` 块放错位置 | API 拒绝——所以有严格 placement |
| 重复删同一 tool_use_id | API 错误——所以跨块全局 dedupe |
| Pinned edits 漏 re-send | 被删 tool_result 重新出现在上下文——破坏 microcompact 效果 |
| 用 cache_edits 时 querySource 不是 main_thread | useCachedMC 短路为 false，块被 client 自己丢弃 |
| 服务端不识别 cache_edits | 整个块作为非法 content 拒绝（依赖 beta header 协商） |

## 可迁移性

按置信度分两组：

### 强可迁移（基于 OBS）

1. **Sticky-on latch 防 mid-session 抖动** — 任何影响 cache key / connection identity 的配置都可以 latch，避免 toggle 引发副作用。证据扎实
2. **Header / body 信号分离** — 影响连接身份的（header / cache key / TLS cert）跟会话同生死；控制行为的（body / config / feature flag）跟动态 flag。注释明说，证据扎实
3. **多层 gate 收紧实验** — 新 API beta 应该层层 gate（DCE flag → runtime flag → provider gate → role gate → model whitelist），不是一次性放开
4. **观测的合法/非法分类** — 监控系统会对所有"异常"告警，需要给"合法异常"一个旁路（`cacheDeletionsPending`）防误报
5. **Client 显式维护逻辑状态 + 重发** — pattern 本身可迁移；但**不要把"因此服务端 stateless"当推论**（后者是 SPEC）

### 弱可迁移（基于 SPEC，谨慎使用）

6. **"逻辑删除 vs 物理删除分离"** — ⚠ **本页起初基于推断的服务端模型**——如果服务端实际不是这样工作（比如它内部确实物理删除然后让 client 重发做容错），那这个 pattern 就不是 cache_edits 在示范的东西。**借鉴这条之前先想清楚你自己的服务端要做哪种**

针对 leto-ai 的迁移建议（用强可迁移那 5 条）：
- "订单已撤销"加 cancelled 标志通过查询投影——这是个**通用模式**，不需要 cache_edits 当依据，传统 RDB 就这么做
- Verifier 验证历史"无效化"加状态而非删——同上，通用模式
- ⚠ 不要把这些迁移决策**建立在对 cache_edits 服务端语义的特定理解上**——cache_edits 服务端可能是别的样子

## 进一步追问的钩子

1. **`isModelSupportedForCacheEditing` 的具体白名单** — 哪些模型支持？这跟模型推理 stack 的实现直接相关
2. **Cache TTL 与 cache_edits 互动** — 5min/1hour TTL 到了，pinned edits 还有意义吗？需要重新建立 prefix 后重新累积
3. **Mycro 的 KV page eviction 策略** — `cache_store_int_token_boundaries` 是 cache_control marker 落点的集合？落在 boundary 上的 token 永不被 evict？
4. **服务端 `cache_edits` 解析失败时的 fallback** — 整个请求失败还是降级处理？
5. **Sub-agent 为什么不用 cache_edits** — 性能上应该也能受益，但代码明确排除（querySource 非 main_thread）。是 cache 隔离设计还是别的考虑？
6. **跟 [[opusplan-tradeoff]] 的 200K 逃生口的关系** — 都是上下文管理的非显式边界，这两个机制有没有交集？

## 关联

- 上层概念：[[context-compression-cascade]]（cache_edits 是其中第 ② 级 microcompact 的 ant-only 实现路径——3P 用户走 no-op，ant 用户走这条 API）
- 协同机制：[[flag-vs-hardcode]]（CACHE_EDITING_BETA_HEADER 由 GrowthBook flag 控制，但加 sticky-on latch 是这条路径的特殊 patch）；[[circuit-breaker]] 的同源思路（自动 compaction 失败 → 兜底，cache_edits 也类似——5 层 gate 任一失败回退）
- 反面对照：autocompact（[[context-compression-cascade]] ④）是 destructive + 公开的；本概念是 projection + 内部
- 相关实体：`services/api/claude.ts:3052-3209`（API 编排）、`services/api/claude.ts:1184-1205`（5 层 gate）、`services/api/claude.ts:1670-1689`（latch 与 body 分离）、`bootstrap/state.ts:234-237`（latch 状态）、`utils/contentArray.ts`（placement 约束）、`services/api/promptCacheBreakDetection.ts:65-67`（合法 cache miss 标记）、`services/compact/microCompact.ts:276-292`（client 触发逻辑）
- 综合分析：[1.CORE_LOOP.md](../1.CORE_LOOP.md)（query 主循环）
