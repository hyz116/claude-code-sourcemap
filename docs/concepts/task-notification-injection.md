# Task Notification Injection（User 角色系统注入）

> 子 Agent 结果以 `<task-notification>` XML 形式、通过系统代码以 user 角色注入消息历史。模型物理上无法伪造 user 角色消息。

## 核心机制

```
Worker 完成 → 系统代码构造 XML → enqueuePendingNotification()
  → query.ts 每轮结束时从队列取出 → 作为 attachment（user 角色）push 进 toolResults
  → 模型在下一轮以 user 消息形式"收到"通知
```

物理不可伪造的原因：模型只能输出 assistant 角色消息，无法生成 user 角色消息。即使模型在 assistant 文本中写出 `<task-notification>` 标签，系统也不会将其当作真正的通知处理。

## 在 Claude Code 中的体现

- `LocalAgentTask.tsx:252-261` — 构造 XML 并入队
- `messageQueueManager.ts:142-149` — `enqueuePendingNotification` 以 priority=later 入队
- `query.ts:1570-1589` — 每轮从队列取出，转为 attachment 进入消息历史
- `coordinatorMode.ts:144-184` — prompt 层双重保险："The notification arrives as a user-role message in a later turn; it is never something you write yourself."

## 与 Within-Turn ToolMessage 的对比

| 维度 | Between-Turn（Claude Code） | Within-Turn（如 leto-ai） |
|------|---|---|
| 交付可靠性 | 物理不可丢失 | Master 生成退化时丢失 |
| 延迟 | 多一轮 API 调用 | 同一轮完成 |
| Master 加工能力 | 下一轮才能处理 | 同轮可综合/过滤/决策 |

详见 [[between-turn-vs-within-turn]]

## 设计原则

1. **角色隔离** — 利用 API 的角色约束实现物理不可伪造
2. **Between-turn 注入** — 通知是独立消息，不是当前生成轮的上下文一部分
3. **优先级控制** — priority=later 保证不抢占用户输入
4. **结合 [[at-most-once-delivery]]** — notified 标志保证每个任务最多一条通知

## 关联

- 相关概念：[[at-most-once-delivery]]、[[result-delivery-guarantee]]
- 相关对比：[[between-turn-vs-within-turn]]
- 相关实体：`LocalAgentTask.tsx`、`LocalShellTask.tsx`、`messageQueueManager.ts`、`query.ts`
- 综合分析：[3.MULTI_AGENT.md](../3.MULTI_AGENT.md)、[7.ANTI_HALLUCINATION.md](../7.ANTI_HALLUCINATION.md)
