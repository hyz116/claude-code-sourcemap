# opusplan 取舍

> Claude Code 静态路由表里**唯一**真正按"会话当前状态"切模型的位面。它的设计回答了一个具体问题：**当用户既要规划质量又要执行成本，怎么办？**

## 核心机制

opusplan 是挂在 `MODEL_ALIASES` 里的特殊别名（`utils/model/aliases.ts:8`）。它有**两副面孔**：

```
静态展开（parseUserSpecifiedModel）：opusplan ≡ Sonnet
动态升级（getRuntimeMainLoopModel）：plan mode + 不超 200K → 升 Opus
```

也就是说：用户选 opusplan 等价于"默认 Sonnet 跑活，进 plan 模式时升 Opus 想方案"。

### 升级判定（`utils/model/model.ts:152`）

```ts
if (
  getUserSpecifiedModelSetting() === 'opusplan' &&
  permissionMode === 'plan' &&
  !exceeds200kTokens   // ← 关键的逃生口
) {
  return getDefaultOpusModel()
}
```

### 200K 逃生口（`utils/tokens.ts:159`）

```ts
const THRESHOLD = 200_000
const lastAsst = messages.findLast(m => m.type === 'assistant')
return usage ? getTokenCountFromUsage(usage) > THRESHOLD : false
```

含义：plan 时上一条 assistant message 已超 200K → 放弃 Opus 升级。Opus 4.6 默认窗口 = 200K，硬升反而爆窗口。

### 主循环每次重算（`query.ts:572`）

```ts
let currentModel = getRuntimeMainLoopModel({
  permissionMode,
  mainLoopModel: toolUseContext.options.mainLoopModel,
  exceeds200kTokens:
    permissionMode === 'plan' &&
    doesMostRecentAssistantMessageExceed200k(messagesForQuery),
})
```

每次进 main loop 都重算——这是整个体系唯一会"在循环里重新选模型"的位面。

### 子 Agent 的传递性（`utils/model/agent.ts:80`）

```ts
if (agentModelWithExp === 'inherit') {
  return getRuntimeMainLoopModel({
    permissionMode: permissionMode ?? 'default',
    mainLoopModel: parentModel,
    exceeds200kTokens: false,   // ← 故意写死 false
  })
}
```

子 Agent 跟着升 Opus，但**强制不查 200K**——子 Agent 是 fresh context，不该被父会话的 token 状况拖累。

## 在 Claude Code 中的体现

### 设计取舍清单

| 维度 | 取舍 | 含义 |
|---|---|---|
| **成本** | Opus ≈ 5× Sonnet 单价 | 赌"规划占小头、执行占大头"——规划用强模型买精度，执行用 Sonnet 省钱 |
| **能力差** | 规划需高抽象，执行只需照办 | 前提假设：plan mode 真能产出"足够明确"的方案，让 Sonnet 执行不再做大决策 |
| **窗口安全** | 200K 逃生口 | 长代码库调研可能已超 200K，硬升 Opus 反而爆窗口 |
| **传递性** | 子 Agent 跟着升、但不查 200K | 子 Agent fresh context，不被父会话拖累 |
| **可见性** | 不在默认 picker 里（`modelOptions.ts:498`） | 不主动推；只有用户已选了才动态加入选项——给"知道自己要什么"的高级用户 |
| **可发现性补偿** | 专门的 tip（`tipRegistry.ts:474`） | 开了 opusplan 但 3 天没用 plan mode → 提醒"按两次 shift+tab 才能激活"。tip 的存在反向暴露：用户会忘 |
| **状态可观测** | StatusLine 显示运行时模型 | 通过 `getRuntimeMainLoopModel` 重算后显示，避免"我现在花的是 Opus 还是 Sonnet"的认知失焦 |
| **不做"自动判难度"** | 用户显式给信号、系统规则化路由 | plan mode 是用户显式进入的状态，不是模型推断；比 LLM-as-router 可控、比"全程一个模型"精细 |

### 裂隙的代价（破坏静态路由后留下的足迹）

opusplan 在静态路由表上**主动开了一个口子**，代价散落在多处：

| 位置 | 代价 |
|---|---|
| `modelAllowlist.ts:76` | 字符串匹配陷阱：`"opus"` 别名前缀匹配会误匹 `"opusplan"`，必须改成 segment-boundary 检查 |
| `aliasMatchesParentTier`（`agent.ts:107`） | 注释明确写 "`opusplan` falls through，because it carries semantics beyond same-tier" |
| `parseUserSpecifiedModel`（`model.ts:458`） | 静态展开时直接当 Sonnet → 必须在静态/动态两条路径上保持一致语义 |
| `query.ts:572` | 主循环每次都重算 currentModel，即使 99% 的会话从不切模式 |
| `agent.ts:83` | 子 Agent inherit 必须重算，不能简单复制父字符串 |
| `StatusLine.tsx:39` | UI 渲染必须重新解析，否则用户看到的模型与实际不符 |
| `tipRegistry.ts:474` | 专门写一个 reminder tip 处理"忘了用 plan mode"的 UX 漏洞 |

**收益**：用户用一个别名替代了"手动 /model 切换"的认知和操作成本。
**代价**：破坏静态路由不变性，所有相关基础设施都要识别这个特例。

## 设计原则

1. **用户给暗示，系统规则化路由** — 别让 LLM 当 router；别让一个模型从头到尾。让用户用**显式的状态切换**（plan mode 进入/退出）给系统暗示，系统按规则升降
2. **静态路由 + 局部动态裂隙** — 整体保持静态可预测，仅在有强烈业务理由的位面开动态口子。每个口子都要明确说出"为什么"
3. **逃生口是动态路由的标配** — `exceeds200kTokens` 这种"动态降级"必须配套，否则升级反而引入新风险
4. **传递性显式声明** — 子 Agent 是否继承动态结果（这里：跟着升 Opus；但忽略 200K 检查）必须代码中清晰标注，不能默认
5. **可观测性补齐裂隙的 UX 成本** — StatusLine 实时显示 + 闲置 tip 提醒。动态路由的隐藏成本在 UX 上必须被显性化
6. **不主动推动、保留为高级特性** — 不在默认 picker 列出，是承认：动态路由对普通用户是"额外认知负担"，不应该是默认

## 可迁移性

任何"成本-能力-可控性"三角约束的 Agent 系统都能借鉴这个模式：
- 不要尝试 LLM 自动判难度——不可靠且引入额外延迟
- 让用户的"显式状态切换"成为模型路由的信号源
- 升级路径必须有逃生口（窗口/超时/失败回退）
- 给 UI 实时反馈"我现在花的是哪一档"

## 关联

- 上层概念：[[model-routing]]（opusplan 是模型路由四层中第 ④ 层"Plan Mode 微调"的具体实现）
- 相关实体：`utils/model/model.ts`（`getRuntimeMainLoopModel`、`parseUserSpecifiedModel`）、`utils/model/agent.ts`（子 Agent 传递性）、`utils/tokens.ts`（200K 检查）、`query.ts`（主循环重算位点）
- 综合分析：[1.CORE_LOOP.md](../1.CORE_LOOP.md)（主循环结构）、[3.MULTI_AGENT.md](../3.MULTI_AGENT.md)（子 Agent 模型继承）
