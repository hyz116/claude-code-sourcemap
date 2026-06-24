# 有效的上下文工程（Anthropic 官方一手来源）

> **来源**：Anthropic Engineering Blog, [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
> **发布**：2025-09-29
> **作者**：Anthropic Applied AI team（Prithvi Rajasekaran, Ethan Dixon, Carly Ryan, Jeremy Hadfield 等）
> **类型**：外部参考 / 官方设计哲学一手来源

---

## 关于本页的信任定位（务必先读）

本 wiki 绝大多数 concept 页是从 **反编译源码事后归纳**的设计假说（OBS/INF/SPEC），并在 [index.md](../index.md) 和 [2.DESIGN_PRINCIPLES.md](2.DESIGN_PRINCIPLES.md) 里反复声明"**不是 Anthropic 的设计哲学**"。

本页不同：它是 **Anthropic 自己发布的一手叙述（NARR/官方）**。因此——

- 对"**Anthropic 声称的设计意图与原则**"，本页是高可信来源（一手）。
- 但它仍是一篇**工程+市场博客**：里面的"我们观察到 / studies have uncovered"是 Anthropic 的内部经验与选择性引用，**未经本 wiki 独立验证**；不要把博客措辞当成对源码行为的逐行断言。
- 本页是补齐 concept 页缺失的**意图侧**证据——尤其是 [[context-compression-cascade]] 等页此前只能从代码反推"做了什么"，本文给出"**为什么**"。交叉映射见文末「与本 wiki 的对应」（那一节是我的 INF，已与原文分离）。

---

## 核心论点

上下文是一种**有限且边际收益递减的资源**。工作重心不是写出完美 prompt，而是在每一步**精选信号密度最高的最小 token 集合，使其最大化达成目标的概率**（"find the smallest set of high-signal tokens that maximize the likelihood of your desired outcome"）。全文可归结为这一条。

## 上下文工程 vs. 提示工程

| | 提示工程 Prompt Engineering | 上下文工程 Context Engineering |
|---|---|---|
| 对象 | 编写/组织指令（主要是 system prompt） | 管理**整个上下文状态**：system prompt + tools + MCP + 外部数据 + 消息历史 |
| 时态 | 一次性、离散任务 | **迭代**：每次决定"喂什么给模型"都重做一次精选 |
| 定位 | AI 工程早期主体 | 提示工程的**自然延续 / 超集**，多轮、长时程 agent 的必需 |

Anthropic 把上下文工程视为提示工程的**自然演进**：agent 在循环里不断产生可能相关的数据，必须周期性精炼"从这片不断膨胀的信息宇宙里，挑哪些进入有限的上下文窗口"。

## 为什么重要：注意力预算（attention budget）

- **上下文腐烂（Context Rot）**：token 数增加 → 模型从上下文准确召回信息的能力下降。所有模型通用，只是衰减陡缓不同（引用 Chroma 的 needle-in-a-haystack 研究）。
- **根因是架构性的**：Transformer 注意力对 n 个 token 是 **n² 两两关系**；上下文越长，注意力被"摊薄"。训练数据中长序列更稀少 → 模型对长程依赖的专用参数更少、经验更少。
- 位置编码插值（position encoding interpolation）等技术能让模型处理更长序列，但以 token 位置理解的退化为代价。
- 结论：这是**性能渐变（gradient）而非断崖（cliff）**——长上下文仍高度可用，只是信息检索与长程推理的精度下降。
- 类比：和人类**有限的工作记忆**一样，LLM 有"注意力预算"，每个新 token 都在消耗它。

## 优质上下文的解剖（逐组件）

**System Prompt——写在"恰当的海拔"（right altitude）**
"金发姑娘区间"，避开两个失败极端：
- 过低：硬编码脆弱的 if-else 逻辑 → 脆弱、维护成本高。
- 过高：空泛高层套话 / 假设了并不存在的共享上下文 → 模型拿不到具体信号。
- 最优：足够具体能引导行为，又足够灵活给模型强启发式。

实践：用 `<background_information>`、`<instructions>`、`## Tool guidance`、`## Output description` 等分区（XML 标签 / Markdown 标题），但随模型能力增强，精确格式越来越不重要。追求**最小但完整**——"minimal ≠ short"。先用最强模型跑最小 prompt，再针对观察到的失败模式补指令/示例。

**Tools——定义 agent 与信息/动作空间的契约**
必须 token 高效、且引导高效行为。像设计良好的代码库函数：自包含、健壮、用途无歧义、功能少重叠；输入参数描述清晰、契合模型强项。
- 典型失败模式：**臃肿工具集**——*"如果连人类工程师都说不清某情境该用哪个工具，agent 更不可能做对。"*
- 精简的最小工具集也利于长交互中可靠地维护与裁剪上下文。

**Examples（Few-shot）——强烈推荐，但要精选**
不要把所有边缘情况塞成一张"清单"。应精选**少量多样、典范**的示例。"对 LLM 而言，示例是胜过千言万语的'图'。"

总原则：所有组件（system prompt / tools / examples / 消息历史）都要**有信息量但紧凑（informative yet tight）**。

## 上下文检索与 agentic search：「即时（just-in-time）」

- agent 的简单定义（引 Simon Willison）：**LLM 在循环里自主使用工具**。模型越强，可放手的自主度越高。
- 趋势：从基于 embedding 的**推理前预检索**，转向 agent 持有**轻量引用**（文件路径、存储的查询、网页链接），运行时用工具**按需加载**数据。
  - Claude Code 即如此：写定向查询、存结果、用 `head`/`tail` 等 Bash 命令分析大数据，**从不把完整数据对象载入上下文**。
  - 模拟人类认知：我们不背诵整个语料，而用文件系统、收件箱、书签等外部组织/索引按需取用。
- **元数据本身是信号**：`tests/` 里的 `test_utils.py` 与 `src/core_logic/` 里的同名文件含义不同；文件夹层级、命名约定、时间戳都暗示用途与相关性。
- 实现**渐进式披露（progressive disclosure）**：每次交互产出的上下文为下一步决策提供线索（文件大小→复杂度、命名→用途、时间戳→相关性），逐层组装理解，只在工作记忆里留必要的。
- **权衡**：运行时探索比预计算检索慢，且需要精心设计的工具与启发式；否则 agent 会误用工具、追死胡同、错过关键信息。
- **混合策略常最佳**：先预取一部分提速，再按需自主探索。Claude Code 即混合——`CLAUDE.md` 朴素地预先放进上下文，glob/grep 即时取文件，绕过陈旧索引与复杂语法树。混合更适合内容不太动态的场景（法律、金融）。
- 总箴言：随模型变强，趋势是"**让聪明的模型聪明地行动**"，人为curation 递减；而 **"做能起作用的最简单的事"** 仍是最佳建议。

## 长周期任务（long-horizon tasks）的三种技术

> 当 token 数超过上下文窗口（如大型代码库迁移、综合研究，数十分钟到数小时连续工作），需要专门技术绕过窗口限制。等更大窗口不是答案——可预见的未来里，任何尺寸的窗口都受**上下文污染**与**相关性**问题影响。

**① 压缩（Compaction）**
会话接近窗口上限时总结其内容，用摘要重启一个新窗口。通常是长时程一致性的**首选杠杆**。
- Claude Code 做法：把消息历史交给模型总结，**保留架构决策、未解 bug、实现细节**，丢弃冗余工具输出/消息；agent 带着压缩后的上下文 **+ 最近访问的 5 个文件**继续。
- 调优要点：在复杂 agent trace 上仔细调 prompt——**先最大化召回**（不漏关键信息），**再提升精度**（剔除冗余）。
- 最安全的轻量形式：**清除旧的工具调用/结果**（tool result clearing）——工具早已在历史深处被调用过，何必再看原始结果？已作为 Claude Developer Platform 的特性发布。

**② 结构化笔记（Structured note-taking / agentic memory）**
agent 定期把笔记**持久化到上下文外的记忆**，之后再拉回。
- 极低开销的持久记忆：如 Claude Code 维护待办清单、自定义 agent 维护 `NOTES.md`，跨数十次工具调用追踪进度与依赖。
- 案例 **Claude 玩宝可梦**：跨数千步维护精确计数（"过去 1,234 步我在 1 号道路练级，皮卡丘升了 8 级，目标 10 级"），无需任何记忆结构提示，自发画出已探索区域地图、记录成就与战斗策略；上下文重置后读自己的笔记继续多小时训练。
- Sonnet 4.5 发布时，在 Claude Developer Platform 公开 beta 了**基于文件系统的 memory tool**，便于跨会话存取上下文外信息、积累知识库。

**③ 子 Agent 架构（Sub-agent architectures）**
不让单个 agent 跨整个项目维护状态，而用专职子 agent 在**干净的上下文窗口**处理聚焦任务。
- 主 agent 持高层计划做协调；子 agent 深挖技术细节或用工具找信息，**各自可能烧掉数万 token，但只返回 1,000–2,000 token 的提炼摘要**。
- 实现**关注点分离**：详细搜索上下文隔离在子 agent 内，主 agent 专注综合分析。在复杂研究任务上对单 agent 系统有显著提升（见 Anthropic 多 agent 研究系统博客）。

**选型取决于任务特征：**

| 技术 | 最适合 |
|---|---|
| 压缩 Compaction | 需要大量来回交互、维持对话流的任务 |
| 结构化笔记 Note-taking | 有清晰里程碑的迭代式开发 |
| 多 Agent 架构 | 并行探索有回报的复杂研究与分析 |

## 结论

上下文工程是用 LLM 构建系统方式的**根本转变**。模型越强，越不需要事无巨细的规约工程、自主度越高；但**把上下文当作宝贵、有限的资源**始终是构建可靠 agent 的核心。无论是为长时程做压缩、设计 token 高效的工具，还是让 agent 即时探索环境——指导原则不变：**找到信号密度最高的最小 token 集合，最大化目标达成的概率**。

---

## 与本 wiki 的对应（以下为我的 INF 映射，非原文内容）

把官方"意图侧"叙述对接到从源码反推的 concept 页——本文恰好是多页此前缺失的**一手意图证据**：

| 本文段落（官方意图） | 对应 wiki 页（源码反推的"做了什么"） | 关系 |
|---|---|---|
| 长周期 §① 压缩、"tool result clearing 是最轻触压缩" | [[context-compression-cascade]]（四级级联：Snip / Microcompact / Context Collapse / Autocompact） | 强对应。本文的"压缩 + 保留最近 5 个文件"≈ Autocompact；"清除旧 tool 结果"≈ Microcompact。**官方只讲了兜底压缩这一层**，与 concept 页发现的"3P 公开版实际只有 Autocompact、4 级精细控制是 Ant 内部"互为印证 |
| 长周期 §③ 子 agent，"各烧数万 token、只回 1–2K 摘要" | [3.MULTI_AGENT.md](../3.MULTI_AGENT.md)、[[coordinator-mode]] | 强对应。官方明确"关注点分离 / 隔离搜索上下文"，正是 Coordinator Mode 与 sub-agent 摘要注入的设计意图 |
| 即时检索、混合策略、`CLAUDE.md` 预载 + glob/grep 即时取 | [1.CORE_LOOP.md](../1.CORE_LOOP.md)、[2.TOOLS.md](../2.TOOLS.md) | 解释了工具体系"为什么"以轻量引用 + 按需加载为主 |
| 注意力预算 / 上下文腐烂 → 把上下文当有限资源 | [[multi-tier-degradation]]、[[result-delivery-guarantee]]（磁盘溢出） | 提供了"为何要分级压缩 / 大输出溢出磁盘"的认知前提 |
| 长周期 §② 结构化笔记 / memory tool | [[session-memory]]（已建页）+ `TodoWriteTool` | 已落地。**关键发现**：CC 有两种笔记 flavor——TodoWrite(agent 自导/in-context)对应文章举例，SessionMemory(系统编排/喂压缩)是文章未强调的另一形态 |

> 注意一处张力：官方文中的压缩描述是**单层兜底叙事**（面向 3P 开发者），而 [[context-compression-cascade]] 从源码挖出的是 **4 级级联**且明确区分了 3P/Ant。两者不矛盾——博客讲公开可用面，concept 页讲内部全貌。这正是"一手意图叙述"与"源码事后归纳"互补、而非互替的例子。

## 逐条交叉验证（源码 ground-truth · 以下为 OBS，附 file:line）

把文章对 Claude Code **实现的实质性断言**逐条拿到 `restored-src/src/` 比对。证据均为代码直引（OBS），判定标注信任级别。

| # | 文章断言（NARR） | 源码证据（OBS） | 判定 |
|---|---|---|---|
| 1 | 压缩后续带"最近访问的 5 个文件" | `services/compact/compact.ts:122` `POST_COMPACT_MAX_FILES_TO_RESTORE = 5`；`createPostCompactFileAttachments`（`:1415`）按 `b.timestamp-a.timestamp` 排序（`:1431`）+ `.slice(0,maxFiles)`（`:1432`） | ✅ 精确确认。但比"5 个"复杂：叠加 `POST_COMPACT_TOKEN_BUDGET`（`:1458`）+ per-file token cap + 跳过已在保留尾部的文件（`:1421`） |
| 2 | 压缩保留"架构决策 / 未解 bug / 实现细节"，丢冗余工具输出 | `services/compact/prompt.ts:61-77` `BASE_COMPACT_PROMPT` 强制 **9 段**结构；`:62` 明确含 "architectural decisions" | ✅ 确认但被文章简化——实际 9 段还含 All user messages / Pending Tasks / Current Work 等文章未提项 |
| 3 | "tool result clearing"是最轻触压缩 | `services/compact/microCompact.ts:36` `TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result content cleared]'` | ✅ 占位符字符串精确匹配。详见 [[context-compression-cascade]] ② |
| 4 | 压缩后留 transcript 逃生口 | `services/compact/prompt.ts:340-350` `transcriptPath` 注入摘要 | ✅ 确认（文章未提，与"高保真"精神一致） |
| 5 | 子 agent"只返回 1,000–2,000 token 提炼摘要" | `tools/AgentTool/prompt.ts:257`（"return a single message…not visible to user…send concise summary"）+ `:107`（"report in under 200 words"为可选建议） | ⚠️ **机制确认，数字未确认**：源码无 1000–2000 token 硬限制，是 emergent；文章用 "often" 已默认。**勿把 NARR 统计当 OBS 常量** |
| 6 | "CLAUDE.md 朴素地预先放进上下文" | `context.ts:153` 注释 "prepended to each conversation, and cached"；`getUserContext` 注入 `claudeMd`（`:185`） | ✅ "naively/prepended" 措辞精确 |
| 7 | glob/grep primitives 即时导航 | `GLOB_TOOL_NAME` / `GREP_TOOL_NAME` 工具存在 | ✅（trivial） |
| 8 | "用 head/tail 等 Bash 命令分析大数据" | `tools/BashTool/prompt.ts:287` "Read files: Use Read (**NOT** cat/head/tail)"；`:293-295` 把 head/tail 列为应避免命令 | ⚠️ **张力**（见下） |
| 9 | 结构化笔记 / agentic memory / memory tool | `tools/TodoWriteTool/TodoWriteTool.ts` ✅、`services/SessionMemory/` ✅、`tools/AgentTool/agentMemory.ts` ✅ | ✅ note-taking 机制确认；"memory tool public beta" 是 Developer Platform API 特性，不在 CLI 源码核查范围 |
| 10 | 子 agent 在"干净上下文窗口"工作 | `tools/AgentTool/runAgent.ts` fresh spawn + [[coordinator-mode]] | ✅ 架构确认 |

**两处需圈出的发现：**

- **① head/tail 张力（#8）**：文章把 head/tail 当 just-in-time 的卖点，但 CC 的 Bash prompt **明确劝阻**——`:287` 要求"读文件用 Read 工具，不要 cat/head/tail"。不严格矛盾（文章场景是*大型数据库数据分析*，Bash prompt 禁的是*读源文件*），但文章泛化措辞会误导：CC 实际语境里 head/tail 读文件是被反对的。属于**博客示例为讲故事而泛化、与产品实际引导脱节**。
- **② "1,000–2,000 token"是观察值非约束（#5）**：源码里子 agent 返回无 token 硬上限，仅 prompt 软引导。读者易把 NARR 统计误当 OBS 常量。

**总判定**：10 条中 **7 条精确确认、1 条确认但被简化、2 条有张力/数字未坐实**。文章在压缩机制（#1#2#3）上与源码高度吻合；在检索叙事（#8）上为讲故事做了泛化。这恰好印证页首信任定位：官方一手叙述（NARR）可信但需对照源码（OBS）校准。

## 关联

- 同类外部参考：[[1.LLM_WIKI_PATTERN]]（Karpathy 的 LLM Wiki 模式，另一个外部一手来源）
- 直接印证的 concept 页：[[context-compression-cascade]]、[[coordinator-mode]]、[[multi-tier-degradation]]、[[result-delivery-guarantee]]
- 相关综合分析：[1.CORE_LOOP.md](../1.CORE_LOOP.md)、[2.TOOLS.md](../2.TOOLS.md)、[3.MULTI_AGENT.md](../3.MULTI_AGENT.md)
- 方法论对照：[[2.DESIGN_PRINCIPLES]]（本页是"官方一手"，那页是"事后归纳假说"——两者信任级别不同，勿混用）
