# Predictable Hallucination Hardcode（可预测幻觉硬编码修复）

> 对于"训练数据混合 / API 消毒 / 视觉相似 / 标注习惯"这类**结构性、可重复**的模型输出偏差，Claude Code 直接在代码里硬编码反向修复，不靠 prompt 禁止——但严格保持"修复对模型不可见"，API schema 与日志仍呈现正确的输入形式。

## 核心机制

```
模型出现某种偏差输出
       │
       ▼
   是结构性 / 可重复 / 可枚举的吗？
       │
       ├─ NO（一次性、上下文依赖、语义性）
       │     │
       │     └─► 走 prompt 约束 + 验证拦截（[[false-claims-bidirectional]] 等）
       │
       └─ YES（训练数据来源、API 中间层、视觉相似、标注习惯）
              │
              ▼
          硬编码反向修复
              │
              ├─ 在边界处转换（输入归一化 / 输出还原）
              ├─ 对模型不可见（schema 仍声明正确类型）
              └─ 严格保留报错路径（只在原匹配失败时尝试）
```

判据：**这种偏差是不是每次都会以同样方式出现？** 是 → 硬编码；否 → prompt 治理或硬约束拦截。

## 在 Claude Code 中的体现

### 六个样本

| 偏差 | 模型实际输出 | 应当输出 | 修复位置 | 根因 |
|---|---|---|---|---|
| Windows shell 语法 | `ls 2>nul` | `ls 2>/dev/null` | `utils/bash/shellQuoting.ts:108-128` | 训练数据混了 Windows CMD 与 POSIX |
| API 消毒标签 | `<fnr>` | `<function_results>` | `tools/FileEditTool/utils.ts:526-550` | Anthropic API 在投喂前消毒，模型看不到原标签 |
| 数字字符串化 | `"head_limit":"30"` | `"head_limit":30` | `utils/semanticNumber.ts` | JSON schema 训练里数字常被引号包裹 |
| 布尔字符串化 | `"replace_all":"false"` | `"replace_all":false` | `utils/semanticBoolean.ts` | 同上，且 `"false"` 字符串看起来像 false |
| 引号类型 | smart quote `"foo"` | straight quote `"foo"` | `tools/FileEditTool/utils.ts:73`（`normalizeQuotes`）| 渲染上下文中智能引号被注入 |
| 波浪号当近似 | `~100` 渲染成 ̶1̶0̶0̶ | `~100`（普通文本）| `utils/markdown.ts:25`（禁用 `del` tokenizer）| 模型用 `~` 表"约等于"，不是 markdown strikethrough |

### 样本一：Windows nul 的真实代价（`shellQuoting.ts:111-128`）

```ts
/**
 * The model occasionally hallucinates Windows CMD syntax (e.g., `ls 2>nul`)
 * even though our bash shell is always POSIX (Git Bash / WSL on Windows).
 * When Git Bash sees `2>nul`, it creates a literal file named `nul` -- a
 * Windows reserved device name that is extremely hard to delete and breaks
 * `git add .` and `git clone`. See anthropics/claude-code#4928.
 */
const NUL_REDIRECT_REGEX = /(\d?&?>+\s*)[Nn][Uu][Ll](?=\s|$|[|&;)\n])/g
```

**为什么这个事故必须硬编码修复**：失败代价**远超原始幻觉**——`nul` 是 Windows 保留设备名，创建一个文件名为 `nul` 的文件几乎无法删除，且会破坏 `git add` / `git clone`。靠 prompt 让模型"不要写 `2>nul`"会不可避免漏掉，必须代码兜底。

### 样本二：API 消毒标签的反向映射（`FileEditTool/utils.ts:526-550`）

```ts
const DESANITIZATIONS: Record<string, string> = {
  '<fnr>':  '<function_results>',
  '<n>':    '<name>',
  '</n>':   '</name>',
  '<s>':    '<system>',
  '</s>':   '</system>',
  '<r>':    '<result>',
  '</r>':   '</result>',
  '\n\nH:': '\n\nHuman:',
  '\n\nA:': '\n\nAssistant:',
  // ... 更多 META 标签
}
```

注释精彩：

> Since Claude can't see any of these strings (sanitized in the API), it'll output the sanitized versions in the edit response.

模型在 file edit 时只能看到 `<fnr>`，**它不知道这个东西消毒前是 `<function_results>`**。所以它写 old_string 时也只能写 `<fnr>`——这不是幻觉，是**信息缺失下的合理推测**。修复（`desanitizeMatchString`）只在 old_string 匹配失败时尝试还原，再次匹配。

### 样本三：semanticNumber 的精确边界（`utils/semanticNumber.ts`）

最有教学价值的代码片段——展示**"宽松解析" vs "硬编码修复"** 的边界：

```ts
export function semanticNumber<T extends z.ZodType>(inner: T = z.number()) {
  return z.preprocess((v: unknown) => {
    if (typeof v === 'string' && /^-?\d+(\.\d+)?$/.test(v)) {
      const n = Number(v)
      if (Number.isFinite(n)) return n
    }
    return v   // ← 不匹配的字符串原样传过——让 inner schema 报错
  }, inner)
}
```

**为什么不用 `z.coerce.number()`**：注释直说——

> z.coerce.number() is the wrong fix: it accepts values like "" or null by converting them via JS Number(), masking bugs rather than surfacing them.

**关键约束**：
1. 只匹配**严格数字字面量**（`/^-?\d+(\.\d+)?$/`），不接受 `""` / `null` / `"30abc"`
2. 不匹配时**原样传过**——让内层 schema 自己报错
3. `z.preprocess` 不影响 emit 给 API 的 schema，**模型仍被告知"这是 number"**

这条边界设计是整个模式的核心：**修复要窄，让"真正的输入错误"仍然报错**——不能为了治幻觉把所有字符串都当合法输入。

### 样本四：semanticBoolean 的反例哲学（`utils/semanticBoolean.ts`）

```ts
return z.preprocess(
  (v: unknown) => (v === 'true' ? true : v === 'false' ? false : v),
  inner,
)
```

注释亮点：

> z.coerce.boolean() is the wrong fix: it uses JS truthiness, so **"false" → true**.

JavaScript 的 truthiness 把非空字符串都当 true——`Boolean("false")` 返回 true。这是个**几乎所有人都会写错的 bug**。Claude Code 显式拒绝 `z.coerce.boolean()`，只接受字面量字符串 `"true"` / `"false"`。

### 样本五：normalizeQuotes 的"二次尝试"模式（`FileEditTool/utils.ts:73`）

```ts
function findActualString(fileContent, searchString): string | null {
  if (fileContent.includes(searchString)) return searchString  // 优先精确

  const normalizedSearch = normalizeQuotes(searchString)        // 退化到归一化
  const normalizedFile = normalizeQuotes(fileContent)
  const searchIndex = normalizedFile.indexOf(normalizedSearch)
  if (searchIndex !== -1) {
    return fileContent.substring(searchIndex, searchIndex + searchString.length)
  }

  return null   // ← 都不匹配就报错，不做模糊匹配
}
```

**精确优先 → 归一化次之 → 失败报错** 三段式。永远不做 fuzzy match——种子文档 §13 明说理由：

> 过度容错会让幻觉内容"碰巧"通过检查；宁可报错让模型重读文件。

### 样本六：markdown 禁用 strikethrough（`utils/markdown.ts:25`）

最简短的样本——禁用整个 markdown 特性：

```ts
marked.use({
  tokenizer: { del() { return undefined } },
})
```

注释：

> Disable strikethrough parsing - the model often uses ~ for "approximate" (e.g., ~100) and rarely intends actual strikethrough formatting

**统计博弈**：模型用 `~` 表"约等于"的频率远高于真想画删除线，整体禁用解析比上下文判断更稳。这种**整体性放弃 markdown 表达力**的取舍，前提是"模型很少需要真画删除线"——如果场景变了，决策也得变。

## 共同设计模式

把六个样本的共性提取出来：

1. **在边界处转换** — 输入归一化 / 输出还原，不动核心逻辑
2. **对模型不可见** — Zod `preprocess` / desanitize 都不影响 emit 给 API 的 schema；模型仍被告知正确类型
3. **优先精确匹配 → 退化才尝试修复** — 失败有清晰的报错路径，不做模糊兜底
4. **严格的修复范围** — 正则只匹结构化模式（`/^-?\d+(\.\d+)?$/`），不接受边界情况
5. **拒绝标准库的"宽松"模式** — `z.coerce.*` / `Boolean()` 都因"太宽"被显式拒绝
6. **代价驱动而非频率驱动** — Windows nul 不一定高频，但代价极高（破坏 git）必须修

## "硬编码修复"的反面：什么不该硬编码

不是所有偏差都该硬编码。判据：

| 不该硬编码 | 原因 | 走什么路径 |
|---|---|---|
| 模型编造测试结果 | 语义性、上下文依赖 | [[false-claims-bidirectional]] prompt 治理 |
| 模型猜测 URL 内容 | 信息真实性问题 | prompt 禁止 + 工具门控 |
| 模型编辑不存在的文件 | 输入错误而非偏差 | 硬约束（[[read-before-write]]）报错 |
| 模型选错工具 | 推理决策问题 | 工具描述优化 + ToolSearch |
| 模型对未读文件做编辑 | 故意违规而非偏差 | 物理拦截（[[ground-truth-via-tools]]）|

**核心边界**：硬编码只处理**模型本可以做对、但因为外部结构原因（API 消毒、训练数据混合）做错**的情形。模型**应当用判断力解决**的问题不能硬编码——会掩盖真实 bug。

## 设计原则

| 原则 | 含义 |
|---|---|
| **代价驱动 > 频率驱动** | Windows nul 不高频但代价极高（破坏 git）必须硬编码；smart quotes 高频但代价低（编辑失败重试），归一化即可 |
| **修复对模型不可见** | API schema 仍声明正确类型；日志仍显示规范输出。修复是**单向**的客户端兜底，不污染模型对正确输入的认知 |
| **窄于"宽松解析"** | 拒绝 `z.coerce.*` / 模糊匹配——它们会让真正的输入错误"碰巧通过"。修复必须有精确正则边界 |
| **退化路径而非首选路径** | 精确匹配优先，归一化是次选，模糊永远不要 |
| **保留报错通道** | 修复只对**已枚举的偏差**生效；不在枚举内的输入仍然报错。报错是修复链的最后一环，不能省略 |
| **统计博弈定决策** | 禁用 strikethrough 是赌"模型用 `~` 表近似的频率 >> 真想画删除线"。决策建立在场景统计上，场景变了决策也要变 |
| **接受信息不对称是根因** | API 消毒导致模型看不到原标签——这是架构事实，不能怪模型。desanitize 是**修复架构副作用**，不是修复模型缺陷 |

## 失效模式与边界

| 失效场景 | 后果 |
|---|---|
| 修复正则太宽 | 把合法输入当幻觉处理（如"包含 nul 的文件名"被错误重写）|
| 修复正则太窄 | 漏掉变体（`2>NUL` 大小写没覆盖）|
| 模型版本变了不再产生该偏差 | 修复变成 dead code，但通常不删——成本低、保留兼容性 |
| 新模型产生新偏差 | 需要新增硬编码——这是**滚动成本**。`@[MODEL LAUNCH]` 注释（[[false-claims-bidirectional]] 也提到）暗示存在迭代机制 |
| 偏差跟正常输入边界模糊 | 该走 prompt 治理而不是硬编码 |

## 可迁移性

任何 LLM 集成系统都会遇到**结构性偏差**：

1. **建立"已知偏差"清单** — 每次发现一个可重复的偏差，在文档里记录"模型实际输出 vs 应当输出"。这份清单本身就是宝贵资产
2. **优先在边界处修复** — 输入归一化（preprocess）/ 输出反向（postprocess），别动核心逻辑
3. **拒绝"宽松解析"诱惑** — 标准库的 coerce / fuzzy 函数往往太宽，要写自己的精确正则
4. **保留报错路径** — 修复链的终点必须是"不匹配就报错"，不能"全都接受"
5. **代价 ≠ 频率** — 评估优先级时看"出错时损失多大"而不是"多久出一次"
6. **应用到 leto-ai**：电商场景的可预测偏差举例——LLM 把 SKU 写错大小写、把价格的小数点写成全角句号、把"件"写成"个"——这些都是结构性、可硬编码修复的偏差

## 进一步追问的钩子

1. **`@[MODEL LAUNCH]` 标记** — 多个修复（FC mitigation、注释行为）带这个标记，意味着 Anthropic 内部有"模型升级 → 重测偏差 → 调整修复"的工程流程。这套流程具体长什么样？测试集是什么？
2. **修复的 sunset 机制** — 模型升级后旧偏差消失，老修复变 dead code。是否定期审计、删除？
3. **跨语言偏差** — 这些样本都是英文/JSON 上下文。中文/日文场景下有没有同类偏差（比如全角符号、引号习惯）需要新增修复？
4. **修复触发的 telemetry** — 每次 desanitize/normalizeQuotes 命中是否有打点？这是评估"修复是否仍必要"的数据基础

## 关联

- 上层概念：[[ground-truth-via-tools]]（修复模式仍服从"工具事实链"——只在工具边界处转换，不污染事实链本体）
- 协同机制：[[read-before-write]]（硬约束派——"模型未读不让写"）vs 本概念（修复派——"模型写错了静默修"）。两者覆盖不同失败类型 |
- 反面对照：[[false-claims-bidirectional]]（语义性偏差走 prompt 治理）；本概念是"结构性偏差走代码硬编码"的对偶
- 相关实体：`utils/bash/shellQuoting.ts:108-128`（Windows nul）、`tools/FileEditTool/utils.ts:526-550`（desanitize）+ `:73`（normalizeQuotes）、`utils/semanticNumber.ts`、`utils/semanticBoolean.ts`、`utils/markdown.ts:25`
- 综合分析：[7.ANTI_HALLUCINATION.md](../7.ANTI_HALLUCINATION.md) §6（已知幻觉的硬编码修复）+ §13 决策表
