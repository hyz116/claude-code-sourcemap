# Read-Before-Write（先读后写）

> 强制模型在编辑文件前必须先通过 FileReadTool 获取文件真实内容，消灭"凭记忆编辑"幻觉。

## 核心机制

```
FileEditTool.call(input)
  → readFileState.get(filePath) 存在？ → 否 → ERROR: "Read it first"
  → isPartialView？ → 是 → ERROR: "Read it first"（部分读取不算已读）
  → 文件修改时间 > 读取时间？ → 是 + 内容变了 → ERROR: "File modified since read"
  → old_string 在文件中存在？ → 否 → ERROR: "String not found"
  → 通过 → 执行编辑
```

## 在 Claude Code 中的体现

- `FileEditTool.ts:289-311` — readFileState 门控 + stale read 检测
- `FileWriteTool.ts:199` — 全量覆盖同样强制已读
- `fileStateCache.ts` — FileStateEntry 记录 timestamp、content、offset/limit、isPartialView

### isPartialView 的来源

| 场景 | isPartialView |
|------|--------------|
| 附件注入的文件内容 | true（可能与磁盘不一致） |
| 带 offset/limit 的部分读取 | true |
| 内存/CLAUDE.md 文件加载 | true |
| 完整 FileReadTool 调用 | false |

### Windows 时间戳误报处理

```typescript
// 完整读取 + 内容未变 → 放行（Windows 云同步/杀毒改时间戳但不改内容）
if (isFullRead && fileContent === readTimestamp.content) {
  // safe to proceed
}
```

## 设计原则

1. **硬约束优于 prompt** — 即使模型忽视 "read it first" 提示，代码也会拒绝
2. **部分读取不算已读** — 防止模型基于片段推断整体（本质是幻觉）
3. **stale read 防并发** — 用户/linter/其他 agent 在模型思考期间改了文件
4. **配合 old_string 精确匹配** — 双重 grounding：先证明"你读过"，再证明"你引用的内容确实存在"

## 关联

- 上层概念：[[ground-truth-via-tools]]（本概念是工具门控做幻觉预防的具体应用——文件内容只能从 Read tool 来）
- 相关概念：[[repair-on-read]]（互补关系：write 前验证 vs read 后修复）
- 相关实体：`FileEditTool.ts`、`FileWriteTool.ts`、`fileStateCache.ts`
- 综合分析：[7.ANTI_HALLUCINATION.md](../7.ANTI_HALLUCINATION.md)
