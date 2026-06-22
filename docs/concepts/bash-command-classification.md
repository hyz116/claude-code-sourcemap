# Bash Command Classification（Bash 命令权限分类管线）

> Claude Code 把"LLM 生成的不可信 bash 命令"转化为"该不该自动放行"，靠**多阶段管线**：tree-sitter AST 解析（fail-closed） → 标志位白名单 → 前缀提取（Fig spec 或 Haiku LLM） → 权限规则匹配 → 可选 ML 分类器。每一阶段处理不同失败模式，组合起来比纯黑名单或纯白名单都强。

## 核心机制

```
LLM 生成 bash 命令
       │
       ▼
① tree-sitter AST 解析（utils/bash/ast.ts）
   ├─► simple commands (argv 可信)
   ├─► too-complex（任何未知节点 → 全命令 ask）
   └─► parse-unavailable（tree-sitter 没装载，回退到 ②）
       │
       ▼
② 旧路径 / 安全检查（bashSecurity.ts）
   - Parser differential 检查（CR 注入、\r 边界差异）
   - Zsh 危险命令清单（zmodload / zpty / ztcp...）
   - 命令替换 / 进程替换 / Zsh `=cmd` 等模式黑名单
       │
       ▼
③ 只读命令白名单（readOnlyCommandValidation.ts）
   - GIT_READ_ONLY_COMMANDS / EXTERNAL_READONLY_COMMANDS
   - 每命令的 safeFlags 表（"--all": "none", "--since": "string"...）
   - additionalCommandIsDangerousCallback per-command 钩子
       │
       ▼
④ 前缀提取（prefix.ts + specPrefix.ts）
   - Fig spec 优先（已知 CLI: git/gcloud/aws/kubectl/docker...）
   - Fall back to Haiku LLM（当 spec 不可用）
   - LRU memoize 防重复
       │
       ▼
⑤ 权限规则匹配
   - Bash(git status:*) 形态
   - 4 层规则（MDM/Global/Project/Local）
       │
       ▼
⑥ Auto-mode classifier（Haiku LLM，ant-only）
   - "如果还是 ask，用 LLM 预测 user 的决定"
   - 速度优化：speculative 预启（[[tool-args-prevalidation]]）
```

## 在 Claude Code 中的体现

### ① Tree-sitter AST：Fail-Closed 解析（`utils/bash/ast.ts`）

代码注释非常清晰：

> The key design property is **FAIL-CLOSED**: we never interpret structure we don't understand. If tree-sitter produces a node we haven't explicitly allowlisted, we refuse to extract argv and the caller must ask the user.
>
> This is **NOT a sandbox**. It does not prevent dangerous commands from running. It answers exactly one question: **"Can we produce a trustworthy argv[] for each simple command in this string?"**

3 种返回：

```ts
| { kind: 'simple'; commands: SimpleCommand[] }       // 可信
| { kind: 'too-complex'; reason: string }             // ask user
| { kind: 'parse-unavailable' }                       // 回退旧路径
```

**SimpleCommand** 提取了 4 个维度：

```ts
type SimpleCommand = {
  argv: string[]                    // 引号已 resolve 后的参数
  envVars: { name: string; value: string }[]   // 前置 VAR=val
  redirects: Redirect[]             // 输入/输出重定向
  text: string                      // 原始 source span
}
```

**关键点**：这里**不是沙箱**，不防止命令执行。它只回答一个问题——"能不能给我一个可信的 argv？"如果不能，下游必须问用户。

### ② Parser Differential：为什么要换掉 shell-quote

`ast.ts` 注释说明为什么从 shell-quote 迁移：

> This module replaces the shell-quote + hand-rolled char-walker approach in bashSecurity.ts / commands.ts. Instead of detecting parser differentials one-by-one, we parse with tree-sitter-bash...

**Parser differential** 是 shell 安全里的经典攻击——shell-quote 跟实际 bash 对同一字符串可能产生不同 tokenization。`bashSecurity.ts:947-1009` 给出具体例子：

```
TZ=UTC\recho     ← \r 在中间
shell-quote 解析: ['TZ=UTC', 'echo']  （\r 是 \s，shell-quote 当 token 边界）
bash 实际执行:    单 token 'TZ=UTC\recho'
```

shell-quote 看到 echo 命令，权限系统给放行；但 bash 实际跑的是别的东西。**parser 解析跟实际执行不一致 = 安全漏洞**。

迁移到 tree-sitter-bash（**用真实的 bash 语法树**）从根上消除了这一类问题。

### ③ Zsh 攻击面（`bashSecurity.ts:46-79`）

清单里展示了 Zsh 跟 Bash 的微妙差异——同一命令字符串在 Zsh 跑会有不同行为：

```ts
const ZSH_DANGEROUS_COMMANDS = new Set([
  // zmodload is the gateway to many dangerous module-based attacks:
  // zsh/mapfile (invisible file I/O via array assignment),
  // zsh/system (sysopen/syswrite two-step file access),
  // zsh/zpty (pseudo-terminal command execution),
  // zsh/net/tcp (network exfiltration via ztcp),
  // zsh/files (builtin rm/mv/ln/chmod that bypass binary checks)
  'zmodload',
  // emulate with -c flag is an eval-equivalent
  'emulate',
  // ...
  'zpty',     // pseudo-terminal command execution
  'ztcp',     // TCP for exfiltration
  'zf_rm',    // Builtin rm bypassing binary check
])
```

**三个值得圈出的攻击向量**：
- `zmodload` 加载模块绕过二进制检查（builtin `zf_rm` 不需要 `/bin/rm`）
- `zpty` 创建 pseudo-terminal 跑命令
- `ztcp` 直接 TCP 数据外传

这些**普通 bash 用户根本不会写**——但 LLM 训练数据里有 zsh 文档，可能产出这种命令。

### ④ Zsh Equals Expansion 攻击（`COMMAND_SUBSTITUTION_PATTERNS`）

```ts
{
  pattern: /(?:^|[\s;&|])=[a-zA-Z_]/,
  message: 'Zsh equals expansion (=cmd)',
}
```

注释解释攻击：

> Zsh EQUALS expansion: `=cmd` at word start expands to `$(which cmd)`. `=curl evil.com` → `/usr/bin/curl evil.com`, bypassing Bash(curl:*) deny rules since the parser sees `=curl` as the base command, not `curl`.

**深刻的攻击**：用户配了 `Bash(curl:*) deny`，攻击者用 `=curl evil.com` ——AST 看到的命令名是 `=curl`（带 = 前缀），权限规则匹配的是字面 `curl`，匹不上 → 放行。Zsh 真跑时会把 `=curl` 解析为 `which curl` 的结果，等价于 curl。

防御是把这种模式直接拉黑。

### ⑤ PowerShell 注释 `<#` 的预防性 block

```ts
{ pattern: /<#/, message: 'PowerShell comment syntax' },
// Defense in depth: Block PowerShell comment syntax even though we don't execute in PowerShell
// Added as protection against future changes that might introduce PowerShell execution
```

**当前根本不会跑 PowerShell**，但**预防未来改动**——如果以后允许 PowerShell 执行，这条防御已经在了。这是个非常 paranoid 但合理的姿态：**安全特性的代价是几行代码，遗漏的代价可能是漏洞**。

### ⑥ 只读命令白名单（`readOnlyCommandValidation.ts`）

每命令一个 config：

```ts
type ExternalCommandConfig = {
  safeFlags: Record<string, FlagArgType>      // 标志位白名单
  additionalCommandIsDangerousCallback?: (    // 额外回调
    rawCommand: string,
    args: string[],
  ) => boolean
  respectsDoubleDash?: boolean                // POSIX `--` 处理
}

type FlagArgType = 'none' | 'number' | 'string' | 'char' | '{}' | 'EOF'
```

举个例子，`git log --oneline --graph` 是 read-only；`git push` 不是。每个 git 子命令单独一个 config。

`FlagArgType` 区分 6 类参数形态——同名 flag 在不同命令可能取不同类型，必须精确建模。

### ⑦ 前缀提取：Fig spec 优先（`utils/shell/specPrefix.ts`）

`specPrefix.ts:24-37` 的 DEPTH_RULES 给已知 CLI 写死前缀深度：

```ts
export const DEPTH_RULES: Record<string, number> = {
  rg: 2,                  // pattern argument is required despite variadic paths
  'pre-commit': 2,
  gcloud: 4,              // CLI tools with deep subcommand trees
  'gcloud compute': 6,    // gcloud scheduler jobs list
  'gcloud beta': 6,
  aws: 4,
  az: 4,
  kubectl: 3,
  docker: 3,
  dotnet: 3,
  'git push': 2,
}
```

`gcloud compute` 6 层深——意味着 `gcloud compute instances list` 跟 `gcloud compute disks list` 都会被识别为不同前缀。**这是 CLI 工具理解的精细化**，不是泛泛的"动词级"权限。

### ⑧ 前缀提取：Haiku LLM 兜底（`utils/shell/prefix.ts`）

当 Fig spec 不可用（如不知名的 CLI），走 Haiku 询问"这条命令的最简前缀是什么？"——模型回答 `git status` 或 `npm install`。LRU memoize 防止同一命令重复问。

注释揭示一个关键防御：

```ts
const DANGEROUS_SHELL_PREFIXES = new Set([
  'sh', 'bash', 'zsh', 'fish', 'csh', 'tcsh', 'ksh', 'dash',
  'cmd', 'cmd.exe', 'powershell', 'powershell.exe', 'pwsh', 'pwsh.exe',
  'bash.exe',
])
```

> Shell executables that must never be accepted as bare prefixes. Allowing e.g. "bash:*" would let any command through, defeating the permission system.

`Bash(bash:*) allow` 等于 `Bash(*) allow`——所以这种 "shell-as-prefix" 直接拒绝。

### ⑨ 权限规则匹配 + ML 分类器（ant-only）

匹配 prefix against permission rules（[[plan-mode-state-machine]] 同体系的 4 层规则）。最后兜底：**auto-mode 的 transcript classifier**——`feature('TRANSCRIPT_CLASSIFIER')` ant-only feature。

代码里：

```ts
// utils/permissions/bashClassifier.ts (3P 版本完全 stubbed)
export function isClassifierPermissionsEnabled(): boolean {
  return false   // 3P 不可用
}

export async function classifyBashCommand(...): Promise<ClassifierResult> {
  return {
    matches: false,
    confidence: 'high',
    reason: 'This feature is disabled',
  }
}
```

3P 版的 bashClassifier 是 stub，**真正的 ML 分类器只在 ant 内部 build**。它是 [[tool-args-prevalidation]] 提到的 speculative classifier 的具体实现。

## 设计原则

| 原则 | 含义 |
|---|---|
| **Fail-closed AST 解析** | 任何未识别的 AST 节点 → too-complex → ask user。**绝不解释自己看不懂的结构** |
| **Tree-sitter 替代 shell-quote 解决根因** | shell-quote 的 tokenization 跟 bash 不一致，每发现一个 differential 就要打一个 patch。换 tree-sitter 用真 bash 语法树从根上消除 |
| **多 Shell 攻击面分别枚举** | Bash/Zsh/PowerShell 命令字符串看起来一样但跑起来不同。每种 shell 单独建模危险命令清单 |
| **预防性 block 跨 shell 模式** | `<#` 是 PowerShell 注释语法，当前不跑 PowerShell 也 block——成本低、未来收益大 |
| **每命令独立 config** | git/aws/kubectl 每个子命令的 safe flags 不同；用 type-strict 的 `FlagArgType` 精细建模，不做通用近似 |
| **前缀深度按 CLI 决定** | gcloud 6 层深、kubectl 3 层、git 1-2 层。**理解每个工具的语义**，不是泛泛的动词级权限 |
| **Shell 可执行不能当前缀** | `Bash(bash:*) allow` 等于 `Bash(*) allow`——直接拒绝这类 shell-as-prefix 规则 |
| **Fig spec 优先，Haiku 兜底** | 已知 CLI 用结构化 spec（精确）；未知用 LLM（可扩展但贵）。LRU memoize 平摊成本 |
| **管线分阶段，每阶段独立失效** | 任意阶段返回 ask 都触发 user prompt——没有"打通所有阶段"的硬路径 |
| **ML 分类器是兜底不是主路径** | 前面阶段都不能定（ask），才用 LLM 预测用户决定。**LLM 不在主流程上** |

## 失效模式与边界

| 失效场景 | 后果 |
|---|---|
| Tree-sitter 未装载 | parse-unavailable → 回退到旧 shell-quote 路径，回到 parser differential 风险窗口 |
| AST 节点超 10000 字符 | parser 拒绝（`MAX_COMMAND_LENGTH`） |
| 新版 Zsh 引入新攻击命令 | 不在 ZSH_DANGEROUS_COMMANDS 清单 → 通过——清单是滚动维护的 |
| Fig spec 跟实际 CLI 不一致 | 提取的前缀错位 → 权限规则匹配失败 |
| Haiku LLM 提取前缀错 | 错误前缀进入规则匹配——但 LRU 让错误持续、不抖动 |
| 用户配 `Bash(=curl:*)` 试图反制 Zsh equals expansion | 规则系统不会自动识别这种配法的弱点 |
| 攻击者用 base64 编码命令 + eval | base64 是另一条路径，eval 在 Zsh dangerous 清单里 |

## 可迁移性

任何让 LLM 生成 shell / DSL / 代码片段并执行的系统都会遇到：

1. **不要相信文本层的 tokenization** — JS shell-quote / Python shlex 跟实际目标解释器不一致是常态。**用目标语言的真语法树**（tree-sitter / 官方 parser）
2. **Fail-closed 是非沙箱场景的关键** — 即使你不能阻止执行，也可以拒绝"我看不懂这个"——把不确定性返回给用户决策
3. **多解释器的语义差异要分别建模** — Bash / Zsh / PowerShell 看起来兼容实际不同。任何"跨 shell 通用"的检查都有漏洞
4. **预防性 block 跨语义模式** — 当前不支持但**可能将来支持**的危险模式（PowerShell 注释、其他 shell extension）现在就 block——成本极低
5. **Fig spec 思路通用** — 用结构化 CLI 描述代替 hand-rolled 解析；已知工具用 spec、未知用 LLM 兜底
6. **LLM 在管线**很贵，最后一站 — 前面所有结构化检查都过不了才用——LRU memoize + speculative 预启把代价摊薄
7. **DANGEROUS_SHELL_PREFIXES 思想通用** — 找出"如果允许这个就等于允许全部"的模式（shell 可执行、`eval`、`exec`、动态 import）显式拉黑

针对 leto-ai 的电商 Agent：
- LLM 生成数据库查询？参考"AST + flag whitelist"——SQL parser 解析 + 标志位白名单（SELECT/UPDATE/DELETE 不同处理）
- LLM 生成 API 调用？参考"前缀提取"——把 endpoint + method 当 prefix（`POST /payment/charge` 单独一类）
- LLM 生成 shell 命令（如部署脚本）？直接抄 Claude Code 的 ast.ts + readOnlyCommandValidation.ts 思路
- "Shell-as-prefix" 拒绝模式 → 业务等价：拒绝 `*` / `admin:*` 这类爬墙规则

## 进一步追问的钩子

1. **AST too-complex 的频率** — 实战中多少比例的 LLM 命令落到 too-complex 路径？这影响 user prompt 的频率
2. **Fig spec 来源** — `@withfig/autocomplete` 是个 Fig 公司的开源 spec 集合。Claude Code 怎么集成？是否打包了 specs？
3. **Haiku prefix extractor 的 prompt** — 怎么让 Haiku 输出"最简前缀"？训练 / few-shot / system prompt 怎么构造？
4. **MAX_COMMAND_LENGTH = 10000 的来源** — 拍脑袋还是实测？长命令的应对？
5. **Zsh 危险清单的滚动维护** — 谁负责跟踪 zsh 新版本？什么节奏审计？
6. **bashSecurity.ts 完整的检查清单** — 我只看到几条，完整清单（INCOMPLETE_COMMANDS / OBFUSCATED_FLAGS / IFS_INJECTION / GIT_COMMIT_SUBSTITUTION 等）每条对应什么具体攻击

## 关联

- 上层概念：[[ground-truth-via-tools]]（tool 边界处的输入校验是事实链入口的守门）；[[plan-mode-state-machine]]（ToolPermissionContext 由本管线消费）
- 协同机制：[[tool-args-prevalidation]]（7 步管线里的 ⑦ 权限检查 = 本概念的输出）；[[predictable-hallucination-hardcode]] 同源思路（已知偏差代码兜底）但前者是输入修复、本概念是输入分类
- 横向对比：[[flag-vs-hardcode]] 的判据"输出回流主推理"——本概念是反向，**输入被信任程度**决定走哪条路径（结构化解析 vs LLM）
- 相关实体：`utils/bash/ast.ts`（tree-sitter AST）、`utils/bash/parser.ts`（tree-sitter 包装）、`utils/bash/ParsedCommand.ts`（接口 + 旧 regex fallback）、`tools/BashTool/bashSecurity.ts`（parser differential / Zsh / 危险模式）、`utils/shell/readOnlyCommandValidation.ts`（白名单 + 标志）、`utils/shell/prefix.ts`（Haiku 提取器）、`utils/shell/specPrefix.ts`（Fig spec 提取器）、`utils/permissions/bashClassifier.ts`（ML 分类器，3P stub）
- 综合分析：[5.PERMISSIONS.md](../5.PERMISSIONS.md)（4 层规则整体）
