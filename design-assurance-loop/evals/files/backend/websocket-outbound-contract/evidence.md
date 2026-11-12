# Evidence Index

## websocketGo 请求语义

- `oego/websocketGo/main.go::PostCmdNoWait` 遍历目标 MAC；找不到在线 client 时记录日志并继续处理其它 MAC，最终仍返回 `ok`。
- `oego/websocketGo/main.go::PostCmdWait` 对离线 MAC 返回逐 MAC 的 `status=-2` / `offline for client`。
- `oego/websocketGo/src/client.go` 将消息写入 client 的发送 channel；HTTP 调用完成与终端实际消费消息不是同一事件。

## tob-server 队列语义

- `backend/internal/dao/systemWebsocketMessageQueue.go::ClaimDueMessages` claim due pending 行。
- `ReleaseStaleRunningMessages` 按 `status=running` 和 `claimed_at` 查找超时 claim。
- running fresh-guard 允许新业务 payload 覆盖正在发送的行，但不重置当前 claim 状态。
- 历史实现中的 `backend/internal/service/tasks/client.go::UpdateClientStatus` 只在在线集合非空时写 `ClientOnlineMacCacheKey`；在线数降为零时旧值可能继续存在。
- `ClientOnlineMacCacheKey` 的生产者是 `UpdateClientStatus`；dispatcher 只需要读取该快照。

## 已确认范围

- 旧 `system_websocket_message_queues` / `system_websocket_message_onlines` 数据不迁移、不补偿。
- 不新增终端 ACK 协议；第一版只能使用现有 HTTP 与在线状态信号。
