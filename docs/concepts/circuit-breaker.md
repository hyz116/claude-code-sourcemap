# Circuit Breaker（断路器）

> 连续 N 次失败后停止尝试，防止局部故障放大为全局雪崩。保护的是系统整体，不是当前请求。

## 核心机制

与重试的区别：重试是"再试也许能成功"，断路器是"再试下去只会浪费资源"。

```
操作 → 失败 → count++ → count < N? → 重试
                                    → count >= N? → 断路（停止尝试）
成功 → count = 0（重置）
```

## 在 Claude Code 中的体现

| 位置 | 阈值 | 事故背景 |
|------|------|---------|
| `autoCompact.ts:67-70` | 3 次 | BQ 2026-03-10: 1,279 sessions 浪费 250K API/天 |
| `useReplBridge.tsx:40` | 3 次 | Datadog 2026-03-08: 单 client 占 17% 的 401 |
| `ccrClient.ts:68` | 10 次 | token 未过期但 server 拒绝（KMS hiccup） |
| `ccrSession.ts:24` | 5 次 | 30min poll ≈ 600 calls，1% 5xx 率 ≈ 6 次命中 |
| `withRetry.ts:54` | 3 次 529 | 容量级联时重试是 3-10× 网关放大 |
| auto-mode | 远程配置 | 服务端 kill switch |

## 设计原则

1. **阈值有量化依据** — 来自事故数据，不是拍脑袋
2. **区分确定性和瞬态** — 确定性失败直接退出，瞬态才走断路器
3. **恢复策略分级** — 可恢复（成功重置）vs 永久熔断（session 生命周期）
4. **保护的是系统** — 当前请求已失败，断路器保护后续所有请求
5. **日志标记断路时刻** — 便于事后排查

## 关联

- 相关概念：[[multi-tier-degradation]]（断路器是梯队中的"终止级"）
- 相关实体：`autoCompact.ts`、`withRetry.ts`、`useReplBridge.tsx`
- 综合分析：[7.ANTI_HALLUCINATION.md](../7.ANTI_HALLUCINATION.md)（pipeline 断路器思路）
