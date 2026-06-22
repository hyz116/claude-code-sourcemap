# Between-Turn Injection vs Within-Turn ToolMessage

> Worker/子Agent 结果如何返回给主 Agent：两种架构选择的优劣和适用场景。

## 对比背景

当 Master Agent 委派任务给 Worker/子 Agent 后，Worker 的结果需要回到 Master 并最终到达用户。这有两种根本不同的架构路径。

## 对比维度

| 维度 | Between-Turn（Claude Code） | Within-Turn（如 leto-ai） |
|------|---|---|
| 实现 | task-notification 以 user 角色在下一轮注入 | Worker 结果作为 ToolMessage 进入当前轮 context |
| 交付可靠性 | 物理不可丢失（Master 无法拦截 user 消息） | Master 生成退化时结果丢失（单点故障） |
| 延迟 | 多一轮 API 调用 | 一轮完成（低延迟） |
| Master 加工能力 | 下一轮才能处理，难以一次性整合多 Worker | 同轮可综合、过滤、决策 |
| 多 Worker 整合 | 各 notification 到达时机不同，整合困难 | 多结果都在同一轮 context，可对比合并 |
| 用户体验 | 先看到"已委派"，等一段时间后才看到结果 | 连贯（委派→执行→输出一气呵成） |
| 异步支持 | 天然支持（Worker 可后台长时间运行） | 不支持（Worker 必须在当前轮内完成） |
| 可审计性 | notification 作为独立消息存在于历史中 | Master 可能裁剪/曲解结果，用户无从察觉 |
| 长上下文风险 | notification 是新一轮起点，不受前文影响 | Worker 大结果塞入 context 可能触发 LLM 退化 |
| Token 成本 | 多一轮 API 调用 | 省 token |

## 核心 Trade-off

```
Between-Turn：用延迟和成本换取交付可靠性
Within-Turn： 用可靠性风险换取延迟和整合能力
```

## 适用场景

| 场景 | 推荐 | 原因 |
|------|------|------|
| Worker 耗时长（>30s） | Between-Turn | 用户不应干等 |
| Worker 结果即最终答案（查询类） | Between-Turn 或 Within-Turn+fallback | 不需要 Master 加工 |
| 需要综合多个 Worker 结果 | Within-Turn | 多结果须同轮整合 |
| Master 需基于结果做条件决策 | Within-Turn | 决策须在同一推理链 |
| Worker 结果很大（>20KB） | Between-Turn 或 Within-Turn+摘要 | 防生成退化 |
| 对交付可靠性要求极高 | Between-Turn | 物理不可丢失 |
| 对延迟敏感 | Within-Turn | 少一轮调用 |

## 最务实的组合方案

对于大部分需要 Master 加工的场景（如 leto-ai），不切换到 Between-Turn，而是在 Within-Turn 基础上加装保底：

1. **正常路径**：Within-Turn（低延迟、可加工）
2. **预防**：长结果预判+摘要给 Master（降低退化概率）
3. **检测**：DeliveryGate 在循环退出前检查交付完整性
4. **降级**：检测到失败时 fallback 到 Worker 原始结果（模拟 Between-Turn 的"直送"效果）

## 关联

- 相关概念：[[task-notification-injection]]、[[result-delivery-guarantee]]、[[multi-tier-degradation]]
- 综合分析：[3.MULTI_AGENT.md](../3.MULTI_AGENT.md)
- 外部应用：leto-ai `empty-output-after-worker-delegation` case
