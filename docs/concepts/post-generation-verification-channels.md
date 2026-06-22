# Post-Generation Verification Channels（生成后检测多通路）

> Claude Code 在"模型生成 tool_use 或 final answer 之后"埋了五条独立检测通路，分别捕获不同失败模式。**故意没有"自动跑功能验证"的统一通道**——这是有意识的设计取舍：自动验证太贵且易误判，让"该深度验证"的判断保留给主 Agent。

## 核心机制

```
模型生成
   │
   ├─ 输出 tool_use ──► [Edit/Write 类工具]
   │                        │
   │                        ▼
   │                    ① LSP 被动诊断（IDE → registerPendingLSPDiagnostic）
   │                    ② IDE 诊断追踪（baseline diff，only 新增）
   │                    ④ PostToolUse Hooks（用户配置 shell command）
   │
   ├─ 输出 tool_use ──► [其他工具] ──► ④ PostToolUse Hooks
   │
   └─ 准备 final report
            │
            ▼
        ⑤ Prompt 软约束（False Claims Mitigation 持续约束）
        ③ Verification Agent（主 Agent 主动 spawn / nudge 触发）
```

五条通路按**触发自动度**分层：① ② ④ 自动（有相应基础设施时），③ 半自动（契约 + nudge），⑤ 软约束。

## 在 Claude Code 中的体现

### 五通路总览表

| # | 通路 | 触发 | 检测什么 | 自动？ | 覆盖范围 | 关键文件 |
|---|---|---|---|---|---|---|
| ① | LSP 被动诊断 | LSP server 推送 `publishDiagnostics` | 类型错误 / 语法 / lint | 是（有 LSP 时）| IDE 模式 | `services/lsp/LSPDiagnosticRegistry.ts` |
| ② | IDE 诊断追踪 | 编辑前 baseline + 编辑后 diff | **因编辑新增**的诊断（过滤已存在）| 是（有 IDE RPC 时）| VS Code / JetBrains | `services/diagnosticTracking.ts` |
| ③ | Verification Agent | 模型主动 spawn / `/verify` / loop-exit nudge | 功能正确性 / 边界 / 对抗性 | 否（契约 + nudge）| 所有环境 | `tools/AgentTool/built-in/verificationAgent.ts` |
| ④ | PostToolUse Hooks | 每次 tool 执行后按 settings.json 配置 | 用户自定义（典型：lint/format/test）| 是（配置后）| 所有环境 | `utils/hooks.ts:643` |
| ⑤ | Prompt 软约束 | 每轮 system prompt 持续约束 | 模型自我汇报的诚实性 | 软约束 | 所有环境 | `constants/prompts.ts:240` |

### ① LSP 被动诊断

`registerLSPNotificationHandlers` (`passiveFeedback.ts:125`) 监听所有连接 LSP server 的 `textDocument/publishDiagnostics` 通知，写进 `LSPDiagnosticRegistry`：

```ts
// 限流避免淹没上下文
const MAX_DIAGNOSTICS_PER_FILE = 10
const MAX_TOTAL_DIAGNOSTICS = 30
const MAX_DELIVERED_FILES = 500   // LRU 跨轮去重
```

`getLSPDiagnosticAttachments`（`attachments.ts:2883`）转成 attachment 注入下一轮 query，模型在下一轮看到错误后**自行决定是否修复**——不强制。

**关键设计**：跨轮 LRU 去重——同一个错误诊断只投递一次，避免每轮都看到同样的 lint 噪音。

### ② IDE 诊断追踪（baseline diff）

跟 ① 不同——这是**主动询问 IDE**而非被动接收。`DiagnosticTrackingService`（`diagnosticTracking.ts:30`）：

```ts
beforeFileEdited(filePath)   // 编辑前通过 MCP RPC 调用 IDE，存基线
getNewDiagnostics()          // 编辑后再问 IDE，diff
  // 只返回 baseline 里没有的新增 diagnostic
```

**核心设计**：**只报新增**。文件原本就有的 100 条 lint 错误不会污染信号——只关心"我这次编辑引入了什么新问题"。这跟通用 LSP 推送形成互补：LSP 看全局，diagnostic-tracking 看 delta。

### ③ Verification Agent

跟前面已沉淀的 [[flag-vs-hardcode]] / [[false-claims-bidirectional]] 直接关联——是**唯一能验证功能正确性**的通道。

设计三特征：
- **disallowedTools 物理只读**（`verificationAgent.ts:139-145`）：禁 Edit/Write/Notebook/AgentTool
- **PASS/FAIL/PARTIAL machine-parseable verdict**（`verificationAgent.ts:117-127`）
- **强制 adversarial probe**：每次 PASS 前必须有至少一条对抗性测试（boundary/concurrency/idempotency/orphan）

为什么**不自动触发**：跑功能验证可能要起 dev server / 跑测试集 / 浏览器自动化，每次编辑都跑代价巨大；且谁来判断"这次改动需要多深的验证"——这个判断本身需要语义理解，自动化反而引入新问题。

替代方案是**契约 + nudge 提醒**：
- system prompt 契约（`prompts.ts:391-394`）："non-trivial 实现必须 spawn Verifier"
- TodoWrite/TaskUpdate 在 loop-exit 注入 nudge（`TodoWriteTool.ts:78-86`）

### ④ PostToolUse Hooks

`utils/hooks.ts:643` 的 `PostToolUse` event。用户在 `settings.json` 里配置，**每次 tool 执行后按 matcher 过滤触发**：

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write",
      "hooks": [{ "type": "command", "command": "eslint --fix $FILE" }]
    }]
  }
}
```

Hook 是 **shell 命令**——stdout 作为 attachment 反馈给模型，模型下一轮看到。

设计哲学：**Anthropic 不假设知道用户的 stack**。开放扩展点——你的 Java 项目用 Maven、Python 用 ruff、Rust 用 cargo check，自己配。

零配置时此通道为零——典型用户可能从来不用。但对中重度用户（CI 严格的项目），这是最强的"每个 tool 都自动 lint"通道。

### ⑤ Prompt 软约束

[[false-claims-bidirectional]] 已细谈。这是唯一**针对模型自我汇报**的通道，其他四个都是针对外部副作用。

## 跟 leto-ai 的对比

leto-ai 架构（`/leto-ai/.../retroactive-verification/architecture.md`）的三层验证闭环：

| leto-ai Layer | 对应 Claude Code 通道 | 关键差异 |
|---|---|---|
| Layer 1 前置验证 | （不在本概念范围 — 见 [[ground-truth-via-tools]] / 权限系统）| Claude Code 的前置在工具调用前 |
| Layer 2 后置验证 | ③ Verification Agent | leto-ai 系统编排触发；Claude Code 模型契约触发 |
| Layer 3 持续巡检 | **没有对应** | Claude Code 是 CLI 单会话，无跨会话巡检需求 |
| 旁路检测（外延）| ① LSP 被动 / ② IDE diagnostic / ④ PostToolUse Hook | leto-ai 设计文档明说旁路检测不在本方案范围；Claude Code 把它当作集成点之一 |

**最关键的设计差异**：

leto-ai 的 Verifier 是**编排层确定性触发**（按状态机：`VERIFYING → VERIFIED/FAILED`）；Claude Code 的 Verifier 是**主 Agent 契约性触发**（system prompt 约定 + nudge 提醒，但没有代码强制）。前者更适合"业务流程稳定"的场景，后者更适合"任务异质性高，无法穷举触发条件"的开发工具场景。

## 失效模式

每条通路有清晰失效边界，互不替代：

| 通路 | 失效场景 |
|---|---|
| ① LSP | 没连 LSP server（纯 CLI 用户）；LSP server 自己有 bug；动态语言的 runtime bug LSP 不可能 catch |
| ② IDE Diagnostic | 没连 IDE（CLI 模式）；MCP RPC 失败；baseline 对比依赖 IDE 状态稳定 |
| ③ Verifier | 主 Agent 没调用（[[false-claims-bidirectional]] 提到的 nudge 启发式可被绕过）；Verifier 自己也是 LLM 也会偷懒（system prompt 自我反幻觉是补丁）|
| ④ Hook | 用户没配置；hook 命令出错；hook 输出太长被截断 |
| ⑤ Prompt | False Claims rate 随模型版本漂移；hedging 当 fallback；模型可能"假装看不见"约束 |

## 设计原则

| 原则 | 含义 |
|---|---|
| **多通路独立** | 一个失效，其他不受影响。覆盖叠加而非互替 |
| **自动通道 + 手动通道并存** | 自动覆盖广度（LSP/IDE/Hook），手动覆盖深度（Verifier）。两者无法相互替代 |
| **不要做"全局自动功能验证"** | 跑测试 / 浏览器自动化太贵；"哪种验证"的判断需要语义理解；自动化引入新问题。把功能验证留给契约触发的 Verifier |
| **新增 vs 全量诊断要分开** | IDE diagnostic 只报"因这次编辑新增的"——比"全量 lint" 信号纯净一个数量级 |
| **限流是必需品** | LSP 限流 30 条 / 文件 10 条 / LRU 500 文件——防止初次打开未格式化项目时诊断淹没上下文 |
| **用户配置作为开放扩展点** | Hook 把"哪些自动检查"决策权交给用户。Anthropic 不假设知道用户的 stack |
| **每条通路只解决自己擅长的问题** | LSP 看静态、Verifier 看功能、Hook 看用户专项、Prompt 看汇报诚实性。**不要让一个通路兜底所有失败模式** |

## 可迁移性

任何 LLM Agent 系统都可以问的几个问题：

1. **我有几条独立验证通路？**——只有 1 条意味着这条失效就完全裸奔
2. **它们各自的失效场景是否独立？**——如果"LSP 失效"和"IDE 失效"是同源（都依赖 IDE 连接），不算独立
3. **我有"只报新增 vs 报全量"的设计吗？**——baseline diff 思路（IDE diagnostic ②）能让信号纯净 10 倍以上
4. **用户能不能扩展？**——PostToolUse Hook 这种开放点把"我的 stack 我自己配"的灵活性给到用户
5. **我有 catch 所有失败的"自动 + 全覆盖"通路吗？**——如果有，警惕：可能太贵、可能误判多、可能在掩盖应当让人决策的判断

特别针对 leto-ai 业务场景：

- **Layer 3 持续巡检** 在 Claude Code 缺失，但 leto-ai 需要——因为电商场景**有跨会话状态**（订单、库存）。这是业务特性决定的，不是 Claude Code 设计缺陷
- **Verifier 触发方式**：leto-ai 用编排层确定触发（业务规则可枚举），Claude Code 用契约触发（开发任务无法穷举）。两者都对——选谁取决于你能不能枚举触发条件
- **PostToolUse Hook 等价物**：leto-ai 可以考虑给运营/SRE 一个"在工具执行后注入自定义检查"的开放点，作为"业务方知道的失败模式" 而 Anthropic 不知道的扩展点

## 进一步追问的钩子

1. **多通路同时报告同一问题怎么去重** — LSP 报 type error + Hook 跑 tsc 也报同一个错，会不会重复刷屏？目前看 LSP 有 LRU 跨轮去重，但跨通路去重没看到机制
2. **Hook 输出长度怎么处理** — 用户配的 hook 可能输出几 MB；attachment 注入有 size cap 吗？跟 [[result-delivery-guarantee]] 的磁盘溢出相关
3. **LSP server 自己 hallucinate 怎么办** — TypeScript LSP 偶尔报错的 bug 是真实存在的；Claude Code 完全信任 LSP 的判断，没有"双 LSP 交叉验证"
4. **Verifier 跟 LSP 的诊断重复时哪个算数** — Verifier 通过功能测试但 LSP 报 type error，主 Agent 该如何决策？是个 prompt 边界问题

## 关联

- 上层概念：[[ground-truth-via-tools]]（本概念是工具事实链的"质量保障侧"——确保工具结果之后还能被外部检查）、[[false-claims-bidirectional]]（通道 ⑤ 的核心）
- 直接关联：[[task-notification-injection]]（通道 ③ 的结果交付机制）、[[result-delivery-guarantee]]（通道 ③ 的交付保障）、[[flag-vs-hardcode]]（通道 ③ 的 nudge 触发也是 flag 控制 `tengu_hive_evidence`）
- 失败恢复簇关联：[[multi-tier-degradation]]（多通路并存可视为"分级覆盖"的一种形态——但这里覆盖的是不同失败维度，不是同一失败的不同强度）
- 相关实体：`services/lsp/`（LSP 通道）、`services/diagnosticTracking.ts`（IDE 通道）、`tools/AgentTool/built-in/verificationAgent.ts`（Verifier 通道）、`utils/hooks.ts:643`（PostToolUse 通道）、`constants/prompts.ts:240`（FC mitigation）
- 综合分析：[7.ANTI_HALLUCINATION.md](../7.ANTI_HALLUCINATION.md) §12（生成后检测能力的真实边界）
- 跟外部对偶：leto-ai 的"三层验证闭环"（Layer 1 前置 / Layer 2 后置 / Layer 3 巡检）—— Claude Code 在 Layer 2 与 leto-ai 同思路但触发机制不同，没有 Layer 3 但把 leto-ai 视为"外延旁路"的部分（LSP/Hook）做成核心通道
