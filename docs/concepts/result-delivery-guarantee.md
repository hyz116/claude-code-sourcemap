# Result Delivery Guarantee（结果交付保障）

> 六层机制确保"系统处理完了但用户没收到/没感知到"不发生。

## 核心机制

从底层到用户界面逐层保障：

1. **Task Notification 注入** — [[task-notification-injection]]，物理不可拦截
2. **流式中断恢复** — 90s 看门狗 + 非流式 fallback + abort 时合成 tool_result
3. **大结果溢出磁盘** — 超阈值写文件，给模型预览+路径
4. **SDK 模式保障** — heldBackResult 暂扣 + do/while drain + 崩溃时发射终止消息
5. **重连状态修复** — WebSocket 重连清空 runningTaskIds（宁缺勿错）
6. **用户感知层** — OS 终端通知 + 6s 无交互触发桌面通知 + 应用内四级优先级队列

## 设计原则

| 原则 | 含义 |
|------|------|
| 不丢 | 流失败降级为非流式；崩溃也发终止消息 |
| 不重 | 原子 notified 标志；wx 独占写 |
| 不阻塞用户 | 通知 priority=later；暂扣结果等后台完成 |
| 可诊断 | 空结果注入说明文本；文件丢失返回原因 |
| 宁缺勿错 | 重连后清状态而非猜测 |

## 关联

- 相关概念：[[task-notification-injection]]、[[at-most-once-delivery]]、[[multi-tier-degradation]]
- 相关实体：`messageQueueManager.ts`、`print.ts`、`notifier.ts`、`sdkEventQueue.ts`
- 综合分析：[3.MULTI_AGENT.md](../3.MULTI_AGENT.md)
