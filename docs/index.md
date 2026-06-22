# Wiki Index

> 本索引按类型分区，每条一行摘要。LLM 查询时先读此文件定位相关页面。

---

## 模块综合分析（种子文档）

| 页面 | 摘要 |
|------|------|
| [0.ARCHITECTURE.md](0.ARCHITECTURE.md) | 整体架构分层：入口 → 核心循环 → 工具体系 → 多Agent → 状态 → 权限 → Skill |
| [1.CORE_LOOP.md](1.CORE_LOOP.md) | query.ts + QueryEngine.ts 的 Agent 循环：API 请求 → 流式解析 → tool_use → 执行 → tool_result → 循环 |
| [2.TOOLS.md](2.TOOLS.md) | 40+ 工具的统一抽象（Tool.ts）、ToolUseContext 依赖注入、流式 AsyncGenerator 输出 |
| [3.MULTI_AGENT.md](3.MULTI_AGENT.md) | AgentTool 四种执行模式（sync/async/remote/swarm）、task-notification 结果注入、Coordinator Mode |
| [4.STATE_MANAGEMENT.md](4.STATE_MANAGEMENT.md) | 35 行 pub/sub store、AppState 80+ 字段、onChangeAppState 副作用集中化 |
| [5.PERMISSIONS.md](5.PERMISSIONS.md) | 四层规则优先级（MDM→全局→项目→本地）、四种模式、Bash AST 解析 + ML 分类器 |
| [6.SKILLS.md](6.SKILLS.md) | 三种来源（内置/磁盘/MCP）、inline vs fork 执行、SkillTool 调度 |
| [7.ANTI_HALLUCINATION.md](7.ANTI_HALLUCINATION.md) | 五层防幻觉：Prompt 禁令 → 工具验证 → 输入修正 → 架构约束 → 时间戳校验 |

## 高维综合（insights/）

| 页面 | 摘要 |
|------|------|
| [insights/0.CONCEPT_ANATOMY.md](insights/0.CONCEPT_ANATOMY.md) | 八维概念解剖：辩证刀、形式刀、历史刀等切面透视 Claude Code |
| [insights/1.LLM_WIKI_PATTERN.md](insights/1.LLM_WIKI_PATTERN.md) | Karpathy 的 LLM Wiki 模式：持久化知识库的增量构建方法论 |
| [insights/2.DESIGN_PRINCIPLES.md](insights/2.DESIGN_PRINCIPLES.md) | 24 篇 concept 页**事后归纳的设计假说**（带认知边界框：明确不是 Anthropic 的设计哲学）：5 簇假说 + 反 pattern 速查 + 3 条校准后的贯穿线 |
| [insights/3.SELF_CRITIQUE.md](insights/3.SELF_CRITIQUE.md) | 4 轮自我批判后浮现的研究方法论失败模式：9 类归纳偏差 + 抽象阶梯（4 层）+ 写作检查清单 + 5 个认识论原因。**本 wiki 可信度结构的元说明** |

## 概念页（concepts/）

> ⚠ **关于"簇"分组**：[insights/2.DESIGN_PRINCIPLES.md](insights/2.DESIGN_PRINCIPLES.md) 和操作日志里多次提到"防幻觉簇 7 篇 / 权限簇 4 篇"等分组——这是我**事后归纳**的 reading aid，**不是 Claude Code 的内部结构**。两个簇被发现过度归纳：
> - **"防幻觉簇 7 篇"** 实际是 4 个子主题（真防幻觉 3 / 输入输出质量 2 / 交付可靠性 1 / 架构事实 1）；result-delivery-guarantee 和 predictable-hallucination-hardcode 跟 hallucination 关系松，被借了标签
> - **"权限/Mode 簇 4 篇"** 实际是 2 个子主题（权限 2 / 工作流模式 2）；coordinator-mode 和 plan-verification-handoff 跟权限关系弱，因为文件目录相近被归一起
>
> 把分组当**索引便利**，不当**本体论**。具体页之间的真实关系看每页的"关联"段。

| 页面 | 摘要 |
|------|------|
| [concepts/multi-tier-degradation.md](concepts/multi-tier-degradation.md) | 多级降级梯队：从轻到重逐级退化，每级独立触发/代价/成功判定 |
| [concepts/circuit-breaker.md](concepts/circuit-breaker.md) | 断路器：连续 N 次失败后停止尝试，防止局部故障放大为全局雪崩 |
| [concepts/at-most-once-delivery.md](concepts/at-most-once-delivery.md) | At-Most-Once：原子 CAS（notified 标志 + O_EXCL 文件锁）保证不重复 |
| [concepts/read-before-write.md](concepts/read-before-write.md) | Read-Before-Write：强制模型先读文件再编辑，消灭"凭记忆编辑"幻觉 |
| [concepts/withhold-then-recover.md](concepts/withhold-then-recover.md) | 暂扣恢复：可恢复错误不暴露给用户，先尝试修复，全部失败后才 surface |
| [concepts/repair-on-read.md](concepts/repair-on-read.md) | 读时修复：不信任任何存储边界，消费侧总是做归一化/修复 |
| [concepts/task-notification-injection.md](concepts/task-notification-injection.md) | User 角色注入：子 Agent 结果以 user 角色系统注入，主 Agent 物理不可伪造 |
| [concepts/result-delivery-guarantee.md](concepts/result-delivery-guarantee.md) | 结果交付保障：六层机制确保"系统完成但用户未感知"不发生 |
| [concepts/model-routing.md](concepts/model-routing.md) | 模型路由：四层静态路由表（主循环/后台碎活/子Agent/Plan微调）— 不按任务难度自动选 |
| [concepts/opusplan-tradeoff.md](concepts/opusplan-tradeoff.md) | opusplan 取舍：静态路由唯一的动态裂隙——plan 升 Opus、执行用 Sonnet，附 200K 逃生口与传递性 |
| [concepts/flag-vs-hardcode.md](concepts/flag-vs-hardcode.md) | Flag vs Hardcode：用 tengu_plum_vx3 解剖"输出是否回流主推理"判据、打包改/毕业机制/命名约定 |
| [concepts/alias-tier-inheritance.md](concepts/alias-tier-inheritance.md) | 别名同 tier 继承：issue #30815 事故剖析——bare alias 在子 Agent 应继承父字符串而非按 provider 默认重新解析 |
| [concepts/ground-truth-via-tools.md](concepts/ground-truth-via-tools.md) | 工具约束即事实来源：防幻觉根哲学——模型声称的事实没有 tool_result 就不可能进入执行路径，物理隔离替代诚信约束 |
| [concepts/false-claims-bidirectional.md](concepts/false-claims-bidirectional.md) | 双向虚假声明治理：同时禁止过度乐观（FC）和过度悲观（hedging），单向治理会推动模型转向另一端 |
| [concepts/post-generation-verification-channels.md](concepts/post-generation-verification-channels.md) | 生成后检测多通路：LSP 被动 / IDE diff / Verifier / Hook / Prompt 五通路对比；故意没有"自动跑功能验证"的统一通道 |
| [concepts/predictable-hallucination-hardcode.md](concepts/predictable-hallucination-hardcode.md) | 可预测幻觉硬编码修复：6 个样本（Windows nul / desanitize / semanticNumber/Boolean / smart quotes / 禁 strikethrough），代价驱动 + 修复对模型不可见 + 拒绝宽松解析 |
| [concepts/tool-args-prevalidation.md](concepts/tool-args-prevalidation.md) | 工具参数预执行 7 步管线：Zod / hint / validateInput / 防伪造剥离 / 双 input 分流 / hooks / 权限——LLM-friendly 错误回流让模型自动 retry |
| [concepts/tool-concurrency-streaming.md](concepts/tool-concurrency-streaming.md) | 工具并发与流式执行：isConcurrencySafe 三态判定 + 失败传播按工具语义非对称（仅 Bash 错误 cancel 兄弟）+ 三层 AbortController + 进度与结果分流 |
| [concepts/context-compression-cascade.md](concepts/context-compression-cascade.md) | 上下文压缩四级级联：Snip / Microcompact / Context Collapse / Autocompact，按代价/失真梯度排序——3P 公开版只跑 Autocompact 兜底，4 级精细控制是 Ant 内部 |
| [concepts/prompt-cache-editing.md](concepts/prompt-cache-editing.md) | Prompt Cache Editing API：非公开 first-party-only beta，cache_edits 块远程删除 tool_result 不 invalidate cache + sticky-on latch + pinned edits + 内部 Mycro/KVCC 术语泄露 |
| [concepts/plan-mode-state-machine.md](concepts/plan-mode-state-machine.md) | Plan Mode 完整状态机：prePlanMode 栈顶寄存器 + 4 种 auto×plan 交叉 + Circuit-breaker 退出降级 + 文件持久化 3 层恢复 + Teammate 分布式审批 |
| [concepts/bash-command-classification.md](concepts/bash-command-classification.md) | Bash 命令权限分类：tree-sitter AST 解析（fail-closed） + Zsh/PowerShell 危险模式 + Fig spec 前缀深度 + Haiku 兜底提取——多 shell 攻击面分别建模 |
| [concepts/coordinator-mode.md](concepts/coordinator-mode.md) | Coordinator 模式：主 Agent 不执行只编排 + worker 强制 async + 跨 worker 隔离 + Scratchpad 共享 + Synthesize 禁止 lazy delegation |
| [concepts/plan-verification-handoff.md](concepts/plan-verification-handoff.md) | Plan 实施后自动验证交接：状态机 + 每 10 轮节流 reminder + "directly NOT delegate" 强约束 + DCE 三层（feature/USER_TYPE/字面量比较） |

## 实体页（entities/）

（待创建 — 下次深入具体文件时逐步建立）

## 对比页（comparisons/）

| 页面 | 摘要 |
|------|------|
| [comparisons/between-turn-vs-within-turn.md](comparisons/between-turn-vs-within-turn.md) | Worker 结果交付：between-turn 注入（不可丢失但高延迟）vs within-turn ToolMessage（低延迟但有单点故障） |

---

## 学习笔记（learning/）

> 消费侧个人学习记录，**不是** wiki 主体的一部分。允许个人化、强主观，不需要遵循 wiki 的研究纪律。

| 页面 | 摘要 |
|------|------|
| [learning/README.md](learning/README.md) | 学习笔记索引和使用约定 |
| [learning/01-from-java-to-agent.md](learning/01-from-java-to-agent.md) | Java 电商工程师入门 AI Agent 的迁移路线（30 天具体路径 + 反直觉点）|

---

## 元文件

| 文件 | 用途 |
|------|------|
| [SCHEMA.md](SCHEMA.md) | Wiki 行为约定（页面格式、命名、工作流） |
| [index.md](index.md) | 本文件 |
| [log.md](log.md) | 操作日志（append-only） |
