# WebSocket 出站统一调度设计（历史最小快照）

## 目标与约束

- 统一调度 `push-package`、`message-push`、应用白名单刷新和动作型消息。
- 保留现有批量发送优化；业务调用方使用消息级入口，不直接操作通用 outbox。
- 使用 MySQL outbox，不引入 Kafka、Redis Stream 或新的终端 ACK 协议。
- 采用前向切换；用户已接受旧 `system_websocket_*` 待发数据不迁移、不补偿。

## 消息与清理策略

- `notify_only`：只通知终端重新拉取事实，请求成功后删除 outbox 行。
- `snapshot`：claim 时记录 `payload_version`；请求成功且版本未变化时删除。
- `rebuild_on_send`：发送前根据 `resource_type/resource_id` 重建最新 payload，因此请求成功后按行 ID 删除，不再比较 `payload_version`。
- `keep_until_online`：离线时保留；`drop`：确认离线时删除。

dispatcher 每轮读取一次 `ClientOnlineMacCacheKey`。终端在缓存中且 `PostCmdNoWait` 返回 HTTP `ok` 时，本次消息视为已投递，可以按上述策略清理。

## 并发与索引

running 行允许新业务变更覆盖 payload，但保持 `status` 和 `claimed_at` 不变。claimant 完成发送后执行清理。

outbox 保留两类业务二级索引：

1. `(mac, message_kind, merge_key)` 唯一键，用于合并和终端清理。
2. `(status, mac, next_attempt_at)`，用于 pending keyset 消费，也复用于 stale-running 释放。

## 切换

同一发布窗口切换生产入口并停止旧消费者，避免双重发送。旧表历史行不自动迁移；稳定后删除旧 DAO、模型和定时任务。
