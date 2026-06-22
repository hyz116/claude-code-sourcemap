# Q&A：模型路由与多 Provider 设计

> 基于源码阅读的问答式记录。关联 wiki：[model-routing](../concepts/model-routing.md)

---

## Q1: Claude Code 的多模型路由是怎么处理的？

**A**: 没有集中式"路由器"或 LLM-as-router，而是**分层解析管道**——通过优先级覆盖链确定模型，再传给 API 层。

### 模型解析优先级链

`getMainLoopModel()` @ `utils/model/model.ts:92`：

1. **会话运行时覆盖**（`/model` 命令）
2. **启动参数**（`--model` CLI flag）
3. **环境变量** `ANTHROPIC_MODEL`
4. **用户设置** `settings.model`
5. **内置默认**（Max/Team Premium → Opus 4.6，其他 → Sonnet 4.6）

### 别名系统

```typescript
['sonnet', 'opus', 'haiku', 'best', 'sonnet[1m]', 'opus[1m]', 'opusplan']
```

- `opusplan`：正常用 Sonnet，Plan Mode 时自动升为 Opus
- `[1m]` 后缀：1M 上下文窗口标记，API 调用前 strip 掉

### 运行时动态调整

| 场景 | 行为 |
|------|------|
| Plan Mode + `opusplan` | 升到 Opus |
| Plan Mode + `haiku` | 升到 Sonnet |
| Fast Mode | 不换模型，附加 `fastMode: true` flag（仅 Opus 4.6 + firstParty） |
| 连续 3 次 529 | 降级到 Sonnet（`FallbackTriggeredError`） |

### 子 Agent 模型解析

`getAgentModel()` @ `utils/model/agent.ts:37`：

1. `CLAUDE_CODE_SUBAGENT_MODEL` 环境变量（全局覆盖）
2. Tool call 的 `model` 参数
3. Agent 定义文件 frontmatter 的 `model` 字段
4. 默认 `'inherit'`（继承父线程）

`aliasMatchesParentTier()` 防止无意义降级：子 Agent 写 `model: 'opus'` 而父已经在 Opus 4.6 → 直接复用父的精确字符串。

### 完整流程

```
用户输入 (--model / /model / env / settings)
    │
    ▼
parseUserSpecifiedModel()    ← 别名 → 完整 ID
    │
    ▼
getMainLoopModel()           ← 优先级链
    │
    ▼
getRuntimeMainLoopModel()    ← Plan Mode 动态调整
    │
    ▼
query.ts currentModel        ← 本次迭代使用的模型
    ├── Fast Mode: 附加 fastMode flag
    ├── 529 过载: 降级到 fallbackModel
    ├── 子 Agent: getAgentModel() 独立解析
    └── Skill: resolveSkillModelOverride()
    │
    ▼
normalizeModelStringForAPI() ← 去掉 [1m] 后缀
    │
    ▼
anthropic.messages.create({ model: "claude-opus-4-6" })
```

### 设计哲学

不做智能路由或请求级分类。模型选择权完全交给用户/配置/上下文，代码只负责正确解析和传递。唯一的自动切换是 Plan Mode 升级和过载降级——两者都是确定性规则，没有 ML-based routing。

---

## Q2: Claude Code 是否有多 Provider 设计？

**A**: 有。支持 4 个 Provider，但不是经典接口多态，而是**环境变量驱动 + 共享 SDK 类型 cast + 散点式特性门控**。

### Provider 选择

```typescript
// utils/model/providers.ts
export type APIProvider = 'firstParty' | 'bedrock' | 'vertex' | 'foundry'

export function getAPIProvider(): APIProvider {
  // 优先级：bedrock > vertex > foundry > firstParty (env vars)
}
```

### 客户端工厂：伪多态

`services/api/client.ts` → `getAnthropicClient()`：

```typescript
// 动态 import 不同 SDK，强制 cast 为统一类型
return new AnthropicBedrock(args)  as unknown as Anthropic  // 有意的类型谎言
return new AnthropicVertex(args)   as unknown as Anthropic
return new AnthropicFoundry(args)  as unknown as Anthropic
return new Anthropic(args)                                   // firstParty 原生
```

调用侧永远只看到 `Anthropic` 类型，不需要关心底层 Provider。代价：某些 `Anthropic`-only 方法在 3P client 上不存在但不会类型报错。

### 认证分立

| Provider | 认证方式 |
|----------|---------|
| firstParty | OAuth / `ANTHROPIC_API_KEY` / `ANTHROPIC_AUTH_TOKEN` |
| Bedrock | AWS STS 临时凭证 / `AWS_BEARER_TOKEN_BEDROCK` / 可跳过 |
| Vertex | `google-auth-library` + `cloud-platform` scope / `gcpAuthRefresh` / 可跳过 |
| Foundry | `ANTHROPIC_FOUNDRY_API_KEY` / Azure AD `DefaultAzureCredential` |

### 特性门控：Provider 能力不对等

全代码库 **60+ 处** `getAPIProvider()` 散点检查。

**仅 firstParty：**
- Fast Mode、全局 Prompt Cache Scope、Cache Editing
- Bootstrap API、Settings Sync、Analytics、预连接优化

**firstParty + Foundry：**
- Structured Outputs、firstPartyOnly betas
- Adaptive thinking 对未知模型默认开启
- Web search（Foundry 全模型支持）

**Bedrock/Vertex 受限：**
- Beta headers 必须放 request body（Bedrock）
- `countTokens` 不可用（Bedrock 自行实现）
- Thinking 仅 Opus 4+ / Sonnet 4+
- 默认 Sonnet 版本滞后（4.5 非 4.6）

**Foundry 的定位**（源码注释）：
> "Generally, foundry supports all 1P features; however out of an abundance of caution, we do not enable any which are behind an experiment"

即 Foundry ≈ firstParty 去掉实验性特性。

### 模型 ID 翻译层

```typescript
// utils/model/configs.ts
CLAUDE_OPUS_4_6_CONFIG = {
  firstParty: 'claude-opus-4-6',
  bedrock:    'us.anthropic.claude-opus-4-6-v1',
  vertex:     'claude-opus-4-6',
  foundry:    'claude-opus-4-6',
}
```

Bedrock 额外动态查询 `ListInferenceProfiles` 获取实际 ARN，处理区域前缀继承。

### 架构评价

| 维度 | 评价 |
|------|------|
| 抽象程度 | 低——没有 Provider interface，靠 SDK cast + 散点 if/else |
| 扩展性 | 差——加新 Provider 需改动 60+ 处 |
| 实用性 | 高——差异本质是"哪些 feature 可用"而非"API 怎么调" |
| 为什么能行 | Anthropic SDK 已做协议适配，Code 只需处理 feature parity 差异 |

### 为什么不做经典抽象？

Provider 间的差异不在"怎么调 API"（SDK 已屏蔽），而在"哪些能力可用"。这种差异天然是散点式的，硬造统一 interface 反而增加认知负担。所以选择务实路线——**一个工厂函数 + 类型 cast + 散点门控**。
