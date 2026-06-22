# At-Most-Once Delivery

> 通过原子 CAS 操作保证同一事件最多触发一次处理，宁缺勿重。

## 核心机制

两种实现：

- **内存级**：`updateTaskState` 内 check-and-set `notified` 标志（单进程竞态）
- **文件系统级**：`flag: 'wx'`（O_EXCL）原子创建文件（跨进程竞态）

## 在 Claude Code 中的体现

### notified 标志（6+ Task 实现）

```typescript
updateTaskState(taskId, setAppState, task => {
  if (task.notified) return task;        // 已通知，跳过
  shouldEnqueue = true;
  return { ...task, notified: true };    // 原子设置
});
if (!shouldEnqueue) return;              // 竞态失败方静默退出
```

竞态场景：任务完成 vs 用户取消 vs TaskStopTool vs backgrounding 同时触发。

### O_EXCL 文件锁（13 处使用）

`autoUpdater.ts`（更新锁）、`cronTasksLock.ts`（cron 锁）、`toolResultStorage.ts`（幂等写入）、`computerUseLock.ts`、`tasks.ts`、`sessionMemory.ts`、`teammateMailbox.ts` 等。

## 为什么选 At-Most-Once

**发送不可逆，不发送可恢复。** 重复通知→模型困惑/重复输出/无法撤回；丢失通知→用户可通过 TaskOutput 手动查询。

## 设计原则

1. **标志先于动作** — 先设 notified=true，再入队。崩溃安全。
2. **静默失败** — 竞态失败方不报错、不日志——这是预期行为
3. **OS 原语优于应用逻辑** — 跨进程用 O_EXCL，不自己实现分布式锁
4. **聚合优于逐条** — 批量 kill 时先标记所有 task 再发一条聚合通知

## 关联

- 相关概念：[[task-notification-injection]]、[[result-delivery-guarantee]]
- 相关实体：`LocalAgentTask.tsx`、`LocalShellTask.tsx`、`stopTask.ts`、`autoUpdater.ts`
- 综合分析：[3.MULTI_AGENT.md](../3.MULTI_AGENT.md)
