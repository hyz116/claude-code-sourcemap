# Repair-on-Read（读时修复）

> 不信任任何存储边界（磁盘、网络、模型输出），消费侧总是做归一化/修复后再使用。

## 核心机制

不是"写入时保证正确"，而是"读取时假设可能不正确并主动修复"。即使是自己写入的数据，读回来时也当外部不可信输入处理。

## 在 Claude Code 中的体现

| 边界 | 修复逻辑 |
|------|---------|
| 磁盘 session | `migrateLegacyAttachmentTypes`、`filterUnresolvedToolUses`、`filterOrphanedThinkingOnlyMessages`、strip invalid permissionMode |
| 模型 tool input | `safeParse` → `validateInput` → `desanitizeMatchString` → `normalizeQuotes` → `rewriteWindowsNullRedirect` → `semanticNumber` |
| API 错误消息 | `extractConnectionErrorDetails`（深度 5 cause chain）、HTML 错误页提取 `<title>` |
| 网络恢复后消息历史 | `ensureToolResultPairing`（每次 API 请求前运行） |

关键注释（`conversationRecovery.ts:173-184`）：
> "Strip invalid permissionMode values from deserialized user messages. The field is unvalidated JSON from disk and may contain modes from a different build."

## 设计原则

1. **消费侧负责** — 不依赖写入方保证格式正确
2. **容错边界严格** — 只修复可预测的已知偏差（引号、消毒标签），不做模糊匹配
3. **每次都修** — 不是"第一次读时修复"，而是每次读取都执行修复链（如 `ensureToolResultPairing`）
4. **forward-compat** — 假设未来版本可能改变格式，当前版本读取时能容忍

## 关联

- 相关概念：[[read-before-write]]（写前验证是"写时保证"的思路，repair-on-read 是互补的"读时修复"思路）
- 相关实体：`conversationRecovery.ts`、`FileEditTool/utils.ts`、`errorUtils.ts`
- 综合分析：[7.ANTI_HALLUCINATION.md](../7.ANTI_HALLUCINATION.md)
