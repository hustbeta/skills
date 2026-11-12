# WebSocket Backpressure 与 Signal PProf 计划（历史问题合成快照）

该快照把同一计划多轮评审发现的问题压缩到一份最小输入，用于检查 reviewer 能否同时发现代码契约错误、计划矛盾和验证缺口。

## 目标与边界

- websocket 队列积压时暂停新的 push-package claim/send，但超时清理始终执行。
- 保留现有 websocket 请求 10 秒超时和 1 分钟 running 超时。
- PProf 仅使用 Unix 信号写本地文件；不增加 `net/http/pprof`，不改变正常 PING/PONG 与 channel-closing 日志。

## Backlog 门禁

新增 `CountWebsocketQueueBacklog(cap)`：

- `Pending` 统计所有 `status=pending` 行，每类最多读取 `cap` 条。
- `Running` 统计 `status=running` 行，每类最多读取 `cap` 条。
- `Total = Pending + Running`。

测试数据包含：一条已到期 pending、一条未到期 pending、一条 running。`cap=1` 的期望为 `Pending=1, Running=0, Total=1`。

`PushPackageProcessor.Exec` 只增加 `backlog=true` 测试，证明积压时不再 claim。正常 claim/send helper 与超时清理实现“同上一版计划”。

当前安全 claim limit 为 `5`。若 backlog 下降太慢，先继续降低 claim limit，减少单轮请求量。

## Signal PProf

- `SIGUSR1` 写 heap 与 goroutine profile。
- `SIGUSR2` 第一次启动 CPU profile，第二次停止并落盘；忘记停止时自动停止。
- 文件名使用秒级时间戳，例如 `cpu-20060102-150405.pprof`。
- CPU profile 测试启动后 `Sleep(100ms)`，停止后断言文件非空。
- Windows 使用 no-op signal 注册，保持可编译。
