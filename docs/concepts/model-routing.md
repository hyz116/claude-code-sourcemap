# Model Routing（模型路由）

> Claude Code 不在运行时按"任务难度"自动选模型——它是一张**写死的、声明式的**四层路由表。所谓"省钱靠 Haiku"是开发者编码时一个个挑出"这件事不值得用大模型"挂上去的。

## 核心机制

四个层次的路由判断，互不干扰：

```
                ┌─ ① 主循环对话    ──► 用户配置驱动（不按任务变）
一次 API 调用 ──┼─ ② 后台碎活      ──► 写死 getSmallFastModel() = Haiku
                ├─ ③ 子 Agent      ──► Agent 定义文件声明 + 运行时解析
                └─ ④ Plan Mode 微调 ──► 仅 opusplan / haiku 两个别名特殊处理
```

整张图的关键判断：**没有 LLM-as-router**。模型与任务的绑定关系在编码时就固化在源码里，运行时只做查表。

## 在 Claude Code 中的体现

### ① 主循环：用户配置驱动

`getMainLoopModel()` @ `restored-src/src/utils/model/model.ts:92` 的优先级：

1. `/model` 会话内切换（`getMainLoopModelOverride()`）
2. 启动时 `--model` 参数
3. `ANTHROPIC_MODEL` 环境变量
4. `settings.json` 的 `model` 字段
5. 内置默认（`getDefaultMainLoopModelSetting()` @ model.ts:178）：
   - **Max / Team Premium** → Opus 4.6（合规时挂 `[1m]`）
   - **Pro / Team Standard / Enterprise / PAYG** → Sonnet 4.6

整个会话只用一个主模型，跟任务复杂度无关。

### ② 后台碎活：硬编码 Haiku

`getSmallFastModel()` @ model.ts:36 = `ANTHROPIC_SMALL_FAST_MODEL` 或回退到 Haiku 4.5，通过 `queryHaiku()` @ `services/api/claude.ts:3241` 暴露成专用通道。

**所有调用点都是写死的**——开发者编码时就决定了"这件事配 Haiku 够了"：

| 用途 | 文件 |
|---|---|
| 生成会话标题 | `utils/sessionTitle.ts:87` |
| WebFetch 网页摘要 | `tools/WebFetchTool/utils.ts:503` |
| WebSearch 查询改写（受 GrowthBook flag `tengu_plum_vx3` 控制） | `tools/WebSearchTool/WebSearchTool.ts:280` |
| Shell 命令前缀识别（权限决策） | `utils/shell/prefix.ts:220` |
| MCP datetime 解析 | `utils/mcp/dateTimeParser.ts:68` |
| Feedback 摘要 | `components/Feedback.tsx:449` |
| 长 tool 结果总结 | `services/toolUseSummary/toolUseSummaryGenerator.ts:69` |
| 离开会话摘要 / 会话改名 | `services/awaySummary.ts:49` / `commands/rename/generateSessionName.ts:20` |
| API key 验证（最便宜的 ping） | `services/api/claude.ts:541` |
| token 估算 | `services/tokenEstimation.ts:277` |
| 用户 hook 默认模型 | `utils/hooks/execAgentHook.ts:118` / `execPromptHook.ts:79` |

共同特征：**短输入 · 无工具 · 确定性输出 · 不需要推理深度**。

### ③ 子 Agent：声明式 + 运行时解析

`getAgentModel(agentDef.model, parentModel, toolParam, mode)` @ `utils/model/agent.ts:37` 的优先级：

1. `CLAUDE_CODE_SUBAGENT_MODEL` 环境变量（全局覆盖）
2. AgentTool 调用方传的 `model` 参数（`z.enum(['sonnet','opus','haiku'])`，临时覆盖）
3. Agent 定义文件的 `model` frontmatter
4. 默认 `'inherit'`（继承父对话当前模型）

内置 Agent 的实际声明：

| Agent | model 声明 | 文件 |
|---|---|---|
| Plan | `'inherit'` | `tools/AgentTool/built-in/planAgent.ts:87` |
| Explore | 外部 `'haiku'` / ant `'inherit'` | `tools/AgentTool/built-in/exploreAgent.ts:78` |
| Verification | `'inherit'` | `tools/AgentTool/built-in/verificationAgent.ts:148` |
| statusline-setup | `'sonnet'` | `tools/AgentTool/built-in/statuslineSetup.ts:141` |
| claude-code-guide | `'haiku'` | `tools/AgentTool/built-in/claudeCodeGuideAgent.ts:119` |

**`aliasMatchesParentTier()` 补丁** @ agent.ts:110：如果子 Agent 写 `model: 'opus'` 而父对话已经在 Opus 4.6 上，子 Agent 直接复用父的精确模型字符串，避免被降到 provider default 的旧版 Opus（issue #30815 的真实事故）。

### ④ Plan Mode 微调（唯一的"动态"逻辑）

`getRuntimeMainLoopModel()` @ model.ts:145 是整套体系里**唯一**会按"当前会话状态"切模型的位面，但只对两个别名生效：

- `opusplan` + plan mode + 不超 200K → 升到 Opus（规划阶段用强模型）
- `haiku` + plan mode → 升到 Sonnet（Haiku 规划能力不够）

### ⑤ Fast Mode：速度层级而非模型切换

`utils/fastMode.ts` 实现的 Fast Mode **不是选另一个模型**，而是在同一模型上请求更快的推理通道：

- `getFastModeModel()` 返回 `'opus'`（可带 `[1m]`）
- `isFastModeSupportedByModel()` 仅对含 `'opus-4-6'` 的模型字符串返回 true
- 仅第一方 API 支持（Bedrock/Vertex/Foundry 不支持）
- API 层发送 `fastMode: true` sticky header latch（一旦开启在会话内持续，以保持 prompt cache key 不变）@ `services/api/claude.ts:1398-1428`
- `/fast` 命令如果当前模型不支持 fast mode，会**自动切到 Opus 4.6**

**限流保护**：遇到 429/529 时 `triggerFastModeCooldown()` 临时退回标准速度，冷却后恢复。

### ⑥ 529 过载降级

`services/api/withRetry.ts:327-351` 实现连续过载时的确定性降级：

```
连续 3 次 529 (MAX_529_RETRIES)
  → 抛出 FallbackTriggeredError { fallbackModel }
  → query.ts:894 捕获，currentModel = fallbackModel (Sonnet)
  → 用 Sonnet 重跑当前 query
```

这是主循环里**唯一的自动模型降级路径**，且只从 Opus → Sonnet 单向触发。与 [[multi-tier-degradation]] 的区别：后者是在同模型内逐步丢弃上下文，这里是真正换模型。

### ⑦ Provider 多云适配

`utils/model/providers.ts` 按环境变量确定 Provider：

```typescript
CLAUDE_CODE_USE_BEDROCK  → 'bedrock'
CLAUDE_CODE_USE_VERTEX   → 'vertex'
CLAUDE_CODE_USE_FOUNDRY  → 'foundry'
默认                      → 'firstParty'
```

`utils/model/configs.ts` 为每个模型维护 per-provider ID 映射：

```typescript
CLAUDE_OPUS_4_6_CONFIG = {
  firstParty: 'claude-opus-4-6',
  bedrock:    'us.anthropic.claude-opus-4-6-v1',
  vertex:     'claude-opus-4-6',
  foundry:    'claude-opus-4-6',
}
```

`getModelStrings()` @ `utils/model/modelStrings.ts` 在最终调 API 前，将逻辑模型名转为对应 Provider 的物理 ID。同时 `settings.json` 的 `modelOverrides` 允许企业用户覆盖为自定义 ARN。

Bedrock 子 Agent 继承父的 region prefix（`us.`/`eu.`），防止跨区域路由。

### ⑧ Skill 模型覆盖

`tools/SkillTool/SkillTool.ts:810-821` + `resolveSkillModelOverride()` @ model.ts:523：

Skill 的 `.md` 定义文件可在 frontmatter 声明 `model:`。解析逻辑：

1. Skill 声明的 model 已含 `[1m]` → 原样使用
2. 当前会话在 `[1m]` 且 Skill 的 model 支持 1M → 继承 `[1m]` 后缀（避免触发 autocompact）
3. 否则直接用 Skill 声明的 model

解析后的模型设为该 Skill 调用上下文的 `mainLoopModel`，调用结束后恢复。

### ⑨ 模型白名单（企业管控）

`utils/model/modelAllowlist.ts` → `isModelAllowed()` 实现三级匹配：

1. 家族别名（`"opus"`）→ 匹配该家族所有模型
2. 版本前缀（`"opus-4-5"`）→ 匹配该版本所有 build
3. 完整 model ID → 精确匹配

组织通过 `settings.availableModels` 限制用户可选范围。

## 设计原则

| 原则 | 含义 |
|---|---|
| **静态路由优于动态判别** | 用 LLM 当 router 既慢又贵，且引入新的不确定性。把绑定关系写进源码 |
| **大模型守主线** | 主对话模型一旦由用户选定，整个会话稳定不变，避免上下文跨模型迁移损耗 |
| **碎活下放到 Haiku** | 短输入 + 无工具 + 确定性输出的内部需求，全部硬编码用 Haiku |
| **Agent 模型声明式** | 子 Agent 在定义时拍板，不让父 Agent 在 runtime "看任务挑模型"——可预测、可测试 |
| **平级别名匹配父 tier** | `model: 'opus'` 复用父的精确串，防止跨 provider 的隐性降级 |
| **环境变量保留逃生口** | `ANTHROPIC_*_MODEL` / `CLAUDE_CODE_SUBAGENT_MODEL` 允许企业部署整体替换 |
| **GrowthBook 控制实验性切换** | `tengu_plum_vx3` 这种 flag 用来 A/B 测试"这件事 Haiku 够不够"，再决定是否硬编码 |

## 三个值得继续追问的点

1. ~~**`tengu_plum_vx3` flag 为什么存在**~~ — 已沉淀为 [[flag-vs-hardcode]]：核心判据是"输出是否回流主推理链"——Haiku 替换会污染主循环时必须 flag 测，纯内部管道直接写死。
2. ~~**`aliasMatchesParentTier` 补丁的事故**~~ — 已沉淀为 [[alias-tier-inheritance]]：Vertex 用户 Opus 4.6 子 Agent 被静默降级到 Opus 4.1 的事故修复；揭示模型路由动态化的第三种驱动源——provider 分裂。
3. ~~**`opusplan` 别名的取舍**~~ — 已沉淀为 [[opusplan-tradeoff]]：静态路由唯一的动态裂隙，含完整设计取舍清单与裂隙代价分析。

## 关联

- 相关概念（动态化三类驱动源）：[[opusplan-tradeoff]]（user-driven，第④层动态裂隙）、[[flag-vs-hardcode]]（Anthropic-driven，第②层 Haiku 替换判据）、[[alias-tier-inheritance]]（provider-driven，第③层 bare alias 同 tier 继承）；可对比 [[multi-tier-degradation]] 思考"为何模型选择不也做成多级降级"
- 相关概念（运行时调整）：[[circuit-breaker]]（Fast Mode 冷却机制类似断路器模式）、[[multi-tier-degradation]]（529 降级是跨模型版本的降级，区别于上下文内降级）
- 相关实体：`utils/model/model.ts`、`utils/model/agent.ts`、`utils/model/configs.ts`、`utils/model/providers.ts`、`utils/model/aliases.ts`、`utils/model/modelStrings.ts`、`utils/model/modelAllowlist.ts`、`utils/fastMode.ts`、`services/api/claude.ts`（`queryHaiku` / `queryWithModel`）、`services/api/withRetry.ts`（529 fallback）、`tools/SkillTool/SkillTool.ts`（Skill 模型覆盖）
- 综合分析：[0.ARCHITECTURE.md](../0.ARCHITECTURE.md)、[3.MULTI_AGENT.md](../3.MULTI_AGENT.md)
