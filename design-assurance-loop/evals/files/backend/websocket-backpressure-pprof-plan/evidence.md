# Evidence Index

## 队列消费者

- `ClaimDueMessages` 的 pending 条件为：`status=pending`、`delay>0`、`updated_at <= DATE_SUB(MySQL now, INTERVAL delay SECOND)`。
- `processOnce()` 先释放 stale running，再 claim due pending。
- backlog 的业务含义是消费者当前可见的 `due pending + running`，不是全部 future pending。

## 时间预算与批量入口

- 全局 websocket HTTP timeout 为 `10s`。
- running timeout 为 `1m`。
- 已确认安全约束为 `claimLimit * 10s < 1m`；当前 `claimLimit=5`。
- `EnqueueMany` 已存在，批量更新不需要创建新的队列抽象。

## PProf 运行约束

- CPU profile 需要实际 CPU 活动才能稳定产生有效样本。
- 相同 profile 类型在同一秒内可能收到多次信号。
- 部署边界要求信号触发、本地文件输出、无 HTTP profile endpoint。
