# Alias Tier Inheritance（别名同 tier 继承）

> 子 Agent 写 `model: 'opus'` 时，应该跟父对话保持**精确同一个 Opus 字符串**，而不是按当前 provider 的默认值重新解析。这条规则只针对 bare family alias 生效——一个被 issue #30815 揭出的 silent downgrade 修复。

## 核心机制

```
子 Agent 声明 model: '<alias>'
       │
       ▼
   是 bare family alias（opus / sonnet / haiku）？
       │
       ├─ NO（opus[1m] / opusplan / best）→ 走常规 parseUserSpecifiedModel
       │
       └─ YES → aliasMatchesParentTier 检查父 canonical
                  │
                  ├─ 父 model 含同 family → 复用父字符串（绕过 default 解析）
                  │
                  └─ 不含 → 走常规 parseUserSpecifiedModel
```

判据：bare alias 是**唯一只表达"我要 X 家族那个东西"**的写法，没有附加语义——所以**唯一可以安全等同于"父对话用的 X"**的别名。

## 在 Claude Code 中的体现

### 事故：issue #30815（Vertex 用户子 Agent 静默降级）

**用户配置**：Claude Code v2.1.68，Vertex AI（`CLAUDE_CODE_USE_VERTEX=1`），通过 `/model` 切到 Opus 4.6

**期望**：spawn 子 Agent `model: opus` 也运行 Opus 4.6

**实际（无补丁时）**：

| Alias | firstParty 用户 | Vertex/Bedrock 用户 |
|---|---|---|
| `sonnet` | Sonnet 4.6 | **Sonnet 4.5** |
| `opus` | Opus 4.6 | **Opus 4.1** ⚠️ 跨过两个大版本 |
| `haiku` | Haiku 4.5 | Haiku 4.5（同） |

UI 显示 "Opus 4.6"、system prompt 也写着 4.6——但子 Agent 跑的是 4.1。**完全沉默的降级**，没有日志、没有警告。

### 根因：provider 路径下 default 的结构性分裂（`utils/model/model.ts:105-128`）

```ts
export function getDefaultOpusModel(): ModelName {
  if (process.env.ANTHROPIC_DEFAULT_OPUS_MODEL) return ...
  if (getAPIProvider() !== 'firstParty') {
    return getModelStrings().opus46    // 修复后两边返回值相同
  }
  return getModelStrings().opus46
}
```

注释明确这种分裂会复发：

> kept as a separate branch even when values match, since 3P availability lags firstParty and **these will diverge again at the next model launch**.

事故时（v2.1.68）3P 分支返回 `opus41`。子 Agent 写 `model: 'opus'` → `parseUserSpecifiedModel('opus')` → `getDefaultOpusModel()` → 在 Vertex 上拿到 4.1。无错误：4.1 是合法 Opus 字符串，调得通。

`getDefaultSonnetModel()` **当前仍然分裂**（3P 返 4.5、firstParty 返 4.6）——证实"分裂会复发"不是过去式，是结构性常态。

### 补丁（`utils/model/agent.ts:110-122`）

```ts
function aliasMatchesParentTier(alias: string, parentModel: string): boolean {
  const canonical = getCanonicalName(parentModel)
  switch (alias.toLowerCase()) {
    case 'opus':   return canonical.includes('opus')
    case 'sonnet': return canonical.includes('sonnet')
    case 'haiku':  return canonical.includes('haiku')
    default:       return false
  }
}
```

调用点（`agent.ts:71, 90`）：

```ts
if (aliasMatchesParentTier(toolSpecifiedModel, parentModel)) {
  return parentModel    // ← 直接复用父字符串，跳过 default 解析
}
```

12 行修复——精确绕过 alias 解析，让父对话用的精确 model string（如 `claude-opus-4-6-20251015`）原样下发到子 Agent。

### 为什么只对 bare alias 生效（`agent.ts:107-108`）

> Only bare family aliases match. `opus[1m]`, `best`, `opusplan` fall through since they carry **semantics beyond "same tier as parent"**.

| Alias | 为什么不走这条路 |
|---|---|
| `opus[1m]` | `[1m]` 是显式 context window 选择，可能跟父不同 |
| `opusplan` | 带 plan mode 动态语义（[[opusplan-tradeoff]]），不是简单同 tier |
| `best` | 抽象层不同——"挑最强的"而非"跟父同款" |

bare `opus` 是唯一**只表达 family、不附带其他语义**的写法。

## 三类动态化驱动源（合上路由表的最后一缝）

把这条事故跟兄弟概念拉通看，**Claude Code 模型路由的"动态化"有三种独立驱动源**：

| 动态维度 | 触发源 | 例子 | 沉淀 |
|---|---|---|---|
| 用户状态切换 | 用户主动切 plan mode | `opusplan` 在 plan 时升 Opus | [[opusplan-tradeoff]] |
| Anthropic 实验 | GrowthBook flag | `tengu_plum_vx3` 把 WebSearch 降 Haiku | [[flag-vs-hardcode]] |
| **Provider 分裂** | **API provider 路径** | **`opus` 在 Vertex 默认是 4.1** | **本页** |

前两类是**有意引入的动态**——为产品功能或 A/B 实验。第三类**反过来**：static routing 在跨 provider 边界处**自然发生分裂**。设计的工作不是引入动态，而是用 `aliasMatchesParentTier` **局部抑制它**，让父子继承场景退化回 static。

## 设计原则

| 原则 | 含义 |
|---|---|
| **默认值是为冷启动设计的，不是为继承设计的** | `opus` 在"用户首次说 `--model opus`"和"子 Agent 想跟父保持一致"两个语境里**应该解析成不同东西**。区分语境是给 alias 加上下文敏感性 |
| **provider 分歧是结构性常态** | 注释里 "will diverge again at the next model launch" 承认每次新模型发布都会分裂。必须有 idiom 处理它，不能寄希望于"以后 3P 跟得上" |
| **UI 撒谎是真正的伤害** | 事故的核心不是 4.1 不能用——是 UI 显示 4.6。沉默降级比显式失败更危险 |
| **附加语义的 alias 不走快路** | `opus[1m]` / `opusplan` / `best` 全部 fall-through。承认它们表达的不只是 tier |
| **修补精确不扩散** | 补丁只动 12 行，没尝试让 firstParty 和 3P 默认值收敛。只在"父子继承的 bare alias"这一交点动手术 |

## 进一步追问的钩子

1. **Bedrock region prefix 的同源处理** — `getAgentModel` 里还有 `applyParentRegionPrefix` 逻辑（`agent.ts:58-67`），处理 Bedrock 跨 region inference profile 继承。这是"同 tier 继承"思路的另一应用——region 也是父子应该一致的维度
2. **`/model` 切换 vs settings.json 写死的差异** — 用户通过 `/model` 切换主对话模型时，`getMainLoopModelOverride()` 存的是什么字符串？是已解析的精确版本还是 alias？这决定补丁能不能起作用
3. **`CLAUDE_CODE_SUBAGENT_MODEL` 全局覆盖会绕过这个补丁** — env var 在 `getAgentModel` 最顶层（`agent.ts:43`）就 return 了。如果企业部署强制了 SUBAGENT_MODEL，`aliasMatchesParentTier` 永不执行——是 feature 还是 bug？

## 关联

- 上层概念：[[model-routing]]（这条修复属于第 ③ 层"子 Agent 模型解析"——补丁解决了 alias 在 provider 边界的歧义）
- 兄弟概念（动态化三类驱动源）：[[opusplan-tradeoff]]（user-driven）、[[flag-vs-hardcode]]（Anthropic-driven）、本页（provider-driven）
- 相关实体：`utils/model/agent.ts:110`（`aliasMatchesParentTier`）、`utils/model/model.ts:105-128`（provider-default 分裂位点）
- 外部参考：GitHub issue #30815（事故原始报告）
- 综合分析：[3.MULTI_AGENT.md](../3.MULTI_AGENT.md)（子 Agent 模型继承的整体位置）
