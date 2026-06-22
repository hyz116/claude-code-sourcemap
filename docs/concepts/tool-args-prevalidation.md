# Tool Args Prevalidation Pipeline（工具参数预执行校验管线）

> 在 LLM 输出 `tool_use` block 到实际 `tool.call()` 之间，Claude Code 嵌入了 **7 步串行校验**（外加 1 步推测性预热）。代码注释直白承认 **"surprisingly, the model is not great at generating valid input"**——这条管线是**面向 LLM 的输入消毒**，每步失败都以 `tool_result` + `is_error: true` 回流让模型自动重试。

## 核心机制

```
LLM 输出 tool_use { name, input }
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ ① Zod schema 校验 (tool.inputSchema.safeParse)              │
│    └─ 失败时附加 ② schemaNotSentHint 提示                    │
├─────────────────────────────────────────────────────────────┤
│ ③ validateInput() 工具语义校验                              │
├─────────────────────────────────────────────────────────────┤
│ ★ Speculative bash classifier 预热（仅 Bash，不阻塞）       │
├─────────────────────────────────────────────────────────────┤
│ ④ Strip _simulatedSedEdit 防伪造                           │
├─────────────────────────────────────────────────────────────┤
│ ⑤ backfillObservableInput 衍生字段（仅给 hook/permission）  │
├─────────────────────────────────────────────────────────────┤
│ ⑥ PreToolUse hooks（用户/系统钩子）                         │
├─────────────────────────────────────────────────────────────┤
│ ⑦ 权限检查（canUseTool / hasPermissionsToUseTool）          │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
   tool.call(callInput)
```

任何一步失败都**不抛异常**，而是构造 `user`-role 的 `tool_result` 消息回流到主循环——模型下一轮看到错误，自行决定重试 / 调整 / 放弃。

## 在 Claude Code 中的体现

文件：`services/tools/toolExecution.ts:613-948`

### ① Zod schema 校验（line 614）

```ts
// surprisingly, the model is not great at generating valid input
const parsedInput = tool.inputSchema.safeParse(input)
```

注释翻译：**"模型生成有效输入的能力出奇的差"**——这是整条管线存在的根本理由。

失败时返回的错误消息**面向 LLM 而非人类**：

```xml
<tool_use_error>InputValidationError:
  The required parameter `file_path` is missing
</tool_use_error>
```

`formatZodValidationError` 区分三类：缺失参数 / 多余参数 / 类型不匹配。

### ② Schema-not-sent hint（line 619-630）

延迟加载工具的特殊补丁——如果工具**不在已发送 schema 集合**里，模型只能"猜"参数格式。Zod 失败 + schema 未发送 → 附加提示：

```
This tool's schema was not sent to the API -- it was not in the
discovered-tool set. Without the schema in your prompt, typed parameters
(arrays, numbers, booleans) get emitted as strings and the client-side
parser rejects them. Load the tool first: call ToolSearch with
query "select:ToolName", then retry this call.
```

**关键设计**：错误消息把"模型为什么会犯这个错"和"怎么修"一起告诉它。模型下一轮就知道先调 ToolSearch。

### ③ validateInput() 工具语义校验（line 683）

22 个工具有独立的 `validateInput()`（详见种子文档 §11.3 表格）。Zod 检查结构（必填/类型），`validateInput` 检查语义（"文件存在吗 / 已读吗 / 索引越界吗"）。

`Tool.ts:495` 的注释明示分工：

> validateInput() — "这个调用在技术上能否执行？"（文件存在吗？索引越界吗？）
> checkPermissions() — "用户是否允许执行？"（权限规则、安全检查）
>                       只在 validateInput 通过后才调用

### ★ Speculative classifier 预热（line 740-752）

不是 gate，是**优化**——Bash 工具特有：

```ts
startSpeculativeClassifierCheck(
  command,
  appState.toolPermissionContext,
  abortSignal,
  isNonInteractiveSession,
)
```

代码注释：

> Speculatively start the bash allow classifier check early so it runs in parallel with pre-tool hooks, deny/ask classifiers, and permission dialog setup.

**设计模式**：拒绝 / 批准的判定需要 100ms+，不要让用户干等——在还没决定要不要弹对话框之前就让 classifier 跑起来，等真的要弹时结果已就绪。

### ④ `_simulatedSedEdit` 剥离（line 756-773）

```ts
// Defense-in-depth: strip _simulatedSedEdit from model-provided Bash input.
// This field is internal-only — it must only be injected by the permission
// system (SedEditPermissionRequest) after user approval.
```

这是个**双保险**：
1. Schema 用 `strictObject` 应该已经拒绝模型构造带 `_simulatedSedEdit` 的输入
2. **但运行时仍剥离**——防 schema 退化导致权限旁路

如果模型成功伪造这个字段，可以绕过 sed 的权限检查。即使概率极低，代价（任意命令执行）足够大，必须双保险。

### ⑤ backfillObservableInput（line 783-793）

最微妙的设计：**hook/permission 看到的 input ≠ tool.call() 收到的 input**。

```ts
let callInput = processedInput
const backfilledClone = tool.backfillObservableInput && ...
  ? { ...processedInput }
  : null
if (backfilledClone) {
  tool.backfillObservableInput!(backfilledClone)
  processedInput = backfilledClone   // ← 给 hook/permission 用
  // callInput 不受影响 ── tool.call() 仍收原 input
}
```

注释解释为什么：

> file tools overwrite file_path with expandPath — that mutation must not reach call() because **tool results embed the input path verbatim** (e.g. "File created successfully at: {path}"), and changing it alters the serialized transcript and VCR fixture hashes.

**核心问题**：
- Hooks/permissions 想看**完整路径**做决策（`/home/user/foo` 而非 `~/foo`）
- 但 `tool.call()` 必须看**原始路径**——返回值里要嵌入原 path 保持 transcript 一致

解法：分叉两条 input——观测路径展开给检查方，执行路径保持原样。

### ⑥ PreToolUse hooks（line 800-862）

用户在 `settings.json` 配的钩子。返回值有 5 类（line 810-861）：

| Hook 返回类型 | 作用 |
|---|---|
| `message` | 注入消息（progress / additionalContext） |
| `hookPermissionResult` | hook 直接给出 allow/deny 决定，跳过后续 permission 检查 |
| `hookUpdatedInput` | 修改 input 继续走流程 |
| `preventContinuation` | 工具执行后阻止主 Agent 继续推理（强制让用户介入） |
| `stop` | 立即终止，回 user-role 错误消息 |

**stop 路径**最值得注意：用户的 hook 可以直接终止工具调用——比如"我配了一个 lint 检查 hook，发现严重问题就 stop"。

### ⑦ 权限检查（line 921）

最后一步——`canUseTool` / `resolveHookPermissionDecision`。决定来源：
- hook 已经决定了（⑥的 `hookPermissionResult`）→ 直接用
- 否则跑 4 层规则（MDM / Global / Project / Local）+ 4 种 mode（default / auto / bypass / plan）

权限通过后才进 `tool.call(callInput)`。

## LLM-friendly Error Feedback 设计

整条管线的错误消息有三个共同特征：

1. **`<tool_use_error>` XML 包裹** — 让模型一眼识别这是工具错误而非 result
2. **`is_error: true` 标记** — Claude API 明确知道这是失败，不当成正常输出
3. **包含纠正建议** — `Did you mean foo.tsx?` / `Note: cwd is /path` / `Load the tool first: call ToolSearch...`

种子文档 §11.5 的反馈设计原则总结：

> - 错误消息面向模型编写（LLM-friendly），而非面向人类
> - 包含纠正建议帮助模型快速恢复
> - `is_error: true` 标记让模型知道需要重试而非继续

这跟 [[predictable-hallucination-hardcode]] 是**不同层的解法**：硬编码修复"已知偏差"自动改了；预校验把"不可自动修的偏差"以友好错误形式让模型自己改。

## 跟 leto-ai 的对比 + 可迁移模板

leto-ai §2.1 的 Layer 1 前置验证「环境/权限/约束预检」与本管线对应，但粒度更粗——只做"环境是否就绪"。Claude Code 7 步对应到 leto-ai 业务场景的迁移模板：

| Claude Code 步骤 | leto-ai 业务场景对应 |
|---|---|
| ① Zod 结构 | 工具调用参数的 JSON schema 验证 |
| ② Schema-not-sent hint | （工具集动态变化时才需要）|
| ③ validateInput 语义 | "SKU 是否存在 / 库存是否充足 / 用户权限是否覆盖该操作" |
| ⑤ backfillObservableInput | 同一个 input 给"风控"看完整路径，给"执行"看脱敏后版本 |
| ⑥ PreToolUse hooks | 业务方注入"促销期间不允许大额操作"这种动态规则 |
| ⑦ 权限检查 | RBAC / ABAC 鉴权 |

特别建议：

1. **结构 vs 语义分开**——`Zod` (③) 和 `validateInput` (③) 的分工是关键。技术校验（结构）通用工具就够，业务校验（语义）每个工具自己写
2. **失败回流而非抛异常**——把 Verifier/Agent 的 retry 逻辑收敛到"主循环看 `tool_result + is_error`"
3. **错误消息面向 LLM 编写**——leto-ai 现有的报错可能面向人类调试，对 Agent retry 不友好；改成"为什么错 + 怎么修"的格式，retry 成功率会显著上升
4. **观测变量分流**（⑤）——业务系统经常需要"风控/审计/执行"看到不同视角的 input，预先分流比事后改造更干净

## 设计原则

| 原则 | 含义 |
|---|---|
| **承认模型生成 input 不可靠** | 注释直说 "surprisingly, the model is not great"。设计建立在这个事实上，而非假设 |
| **失败回流，不抛异常** | 每步失败都构造 `user`-role tool_result，让主循环自然推进——模型下一轮看到自动 retry |
| **错误消息面向 LLM 设计** | XML 包裹 + is_error 标记 + 纠正建议三件套，让模型能 self-recover |
| **结构校验和语义校验分离** | 通用 Zod 处理结构（必填/类型/strictObject），每个工具自己实现语义（语义性失败的具体原因） |
| **观测 input 与执行 input 分流** | hook/permission 看完整路径，tool.call() 看原 path——保持 transcript 一致性 |
| **Defense-in-depth 是给"低概率高代价"的** | `_simulatedSedEdit` 双保险——schema 应该挡住，但代价（任意命令）大到值得运行时再剥一次 |
| **拒绝时间做有用工作** | Speculative classifier 在等 hook/permission 的同时跑——把延迟"藏"到原本就要等的时间里 |
| **Hooks 是开放扩展点** | PreToolUse 5 类返回值（message/permission/updatedInput/preventContinuation/stop）让用户能在任意位置介入 |

## 失效模式与边界

| 失效场景 | 后果 |
|---|---|
| Zod schema 太宽（缺 `strictObject`）| 防伪造的双保险只剩运行时剥离一层 |
| validateInput 跑了 expensive I/O | 校验阶段慢，整条管线延迟暴涨 |
| PreToolUse hook 跑挂 | 会触发 SLOW_PHASE_LOG_THRESHOLD_MS 日志，但不会自动超时——需要 hook 自己管 |
| 错误消息不含纠正建议 | 模型 retry 时仍然不知道怎么改，可能死循环 |
| backfillObservableInput 错误地修改了 callInput | 破坏 transcript 一致性 + VCR fixture hash 漂移（注释里点名） |

## 进一步追问的钩子

1. **22 个工具的 validateInput 共性能否抽象** — 文件类工具都做"存在性 + 已读检查"，能否做成 mixin 减少重复？还是各工具有特殊性必须分开
2. **PreToolUse hook 的超时与 cancel** — 用户配的 hook 跑挂会怎样？SLOW_PHASE_LOG 是观测，没看到 hard timeout
3. **错误消息会不会太啰嗦把 context 占满** — 极端场景：一个工具被错调 50 次，每次都注入 200 token 的 `<tool_use_error>`，对 context window 影响多大？
4. **管线本身被绕过的可能** — SDK 模式 / coordinator mode 下管线是否完整执行？某些 entry point 可能跳过

## 关联

- 上层概念：[[ground-truth-via-tools]]（管线确保进入"事实链"的输入是合法的——是事实链入口的守门人）
- 协同机制：[[predictable-hallucination-hardcode]]（自动修可预测偏差）vs 本概念（不可自动修的让模型 retry）；[[read-before-write]] 是 ③ validateInput 的具体应用
- 反馈层：[[false-claims-bidirectional]]（输出层防错）vs 本概念（输入层防错）；两者构成"输入-输出"双向治理
- 相关实体：`services/tools/toolExecution.ts:600-948`（管线主体）、`Tool.ts:476-495`（validateInput 与 checkPermissions 分工）、`utils/hooks.ts:643`（PreToolUse hook 处理）
- 综合分析：[2.TOOLS.md](../2.TOOLS.md)（工具系统整体）、[7.ANTI_HALLUCINATION.md](../7.ANTI_HALLUCINATION.md) §11
- 跟外部对偶：leto-ai 的 Layer 1 前置验证——本管线是其细粒度版本，可作为业务系统前置验证设计的迁移模板
