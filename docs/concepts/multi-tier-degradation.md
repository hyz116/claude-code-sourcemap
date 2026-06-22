# Multi-Tier Degradation（多级降级梯队）

> 对同一类问题设计多级递进应对手段，从最轻量（几乎无损）到最激进（有损但保活），每级只在前一级失败或不适用时才触发。

## 核心机制

```
Level 1: 预防（在问题发生前削减风险）
Level 2: 轻量修复（问题出现，低代价修正）
Level 3: 标准恢复（修正失败，中等代价恢复）
Level 4: 降级运行（恢复失败，功能降级但不中断）
Level 5: 紧急救援（降级也不行，最后手段）
Level 6: 优雅终止（一切失败，给用户可操作的错误信息）
```

与传统 try-catch 的区别：每个 Level 有明确的触发条件、代价描述、成功判定、传递给下级的状态。

## 在 Claude Code 中的体现

### 上下文超长（6 级）

snip → microcompact → context-collapse → autocompact → reactive-compact → blocking-limit

阈值错开避免竞态：collapse@90% / autocompact@93% / blocking@95%

### API 调用失败（6 级）

streaming → non-streaming fallback → 分类重试 → 模型降级 → 持久重试 → CannotRetryError

### 模型输出无效（5 级）

Zod 结构验证 → 静默容错修正 → validateInput 语义校验 → 精确匹配多级容错 → 路径纠正

## 设计原则

| 原则 | 含义 |
|------|------|
| 从轻到重 | 先尝试最低代价手段 |
| 各级独立 | 每级可独立禁用（feature flag） |
| 预防 > 检测 > 恢复 | 能预防就预防 |
| 阈值错开 | 各级触发点避免竞态 |
| 暂扣不暴露 | 可恢复的失败先藏起来 |
| 防止死循环 | 每级有 once guard 或 [[circuit-breaker]] |
| 状态跨级传递 | 前一级结果影响后一级判断 |

## 关联

- 相关概念：[[circuit-breaker]]、[[withhold-then-recover]]
- 相关实体：`query.ts`（上下文梯队）、`withRetry.ts`（API 梯队）、`toolExecution.ts`（输出验证梯队）
- 综合分析：[1.CORE_LOOP.md](../1.CORE_LOOP.md)
