# Withhold-Then-Recover（暂扣恢复）

> 可恢复错误不暴露给用户，先暂扣并尝试修复；全部修复手段耗尽后才 surface。

## 核心机制

```
API 返回错误（413 / max_output_tokens / media_size）
  → 判断：可恢复？
    → 是：withhold（不 yield 给用户）→ 尝试恢复 → 成功？continue 循环
    → 否：直接 yield
  → 恢复全部失败：surface error + 退出
```

关键：withheld 的错误仍 push 进 `assistantMessages`，以便恢复逻辑能找到它。

## 在 Claude Code 中的体现

`query.ts:788-825` — 413/max_output_tokens/media_size 三种错误被 withhold：

1. **collapse drain**（便宜）→ 如果 context-collapse 有未提交的 commits，先 drain
2. **reactive compact**（贵）→ 全量压缩后重试
3. **恢复耗尽** → yield 错误 + executeStopFailureHooks + return

防护：`hasAttemptedReactiveCompact = true` 防止无限循环。

注释原文：
> "Recovery exhausted — surface the withheld error now."
> "Do NOT fall through to stop hooks: the model never produced a valid response."

## 设计原则

1. **用户只看到最终结果** — 要么静默恢复成功，要么所有手段耗尽后报错
2. **单次保护** — once guard 防止恢复本身进入死循环
3. **不吞错误** — withhold ≠ 丢弃，错误始终保留在消息链中可被后续逻辑访问
4. **恢复与 stop hooks 互斥** — 未产出有效响应时不执行 stop hooks，防止 hook 注入更多 token 加剧问题

## 关联

- 相关概念：[[multi-tier-degradation]]（withhold-then-recover 是梯队中的"检测+恢复"段）、[[circuit-breaker]]
- 相关实体：`query.ts`、`reactiveCompact.ts`、`contextCollapse/`
- 综合分析：[1.CORE_LOOP.md](../1.CORE_LOOP.md)
