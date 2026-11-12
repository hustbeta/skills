# Gold Decision Ledger

> 仅供 evaluator 使用，不得作为被测 agent 输入。

## Accepted findings

### WS-001

- severity: `P1`
- category: `contract`
- violated_contract: `请求成功不能被表述为逐 MAC 已送达`
- required_evidence: `PostCmdNoWait 对离线 MAC 继续处理且最终仍返回 ok；PostCmdWait 才返回逐 MAC 离线状态`
- failure_scenario: 离线终端对应消息因 HTTP `ok` 被当作已送达并删除，终端上线后再也无法收到该业务动作。
- accepted_fix: 把 `PostCmdNoWait` 定义为 best-effort 请求级成功；按消息 kind 的既有容错语义决定是否清理，不承诺强投递。

### WS-002

- severity: `P1`
- category: `concurrency`
- violated_contract: `旧 claimant 不得删除 running 期间写入的新 payload`
- required_evidence: `running fresh-guard 允许覆盖 payload；rebuild_on_send 仍可能与发送后的新入队并发`
- failure_scenario: claimant 重建并发送版本 N 后，业务写入版本 N+1；claimant 随后按 ID 删除整行，N+1 丢失。
- accepted_fix: `snapshot` 与 `rebuild_on_send` 都记录并校验 `payload_version`；版本变化时在 claim 所有权门禁下转回 pending。

### WS-003

- severity: `P2`
- category: `operability`
- violated_contract: `stale-running 恢复路径必须有匹配查询条件的索引`
- required_evidence: `释放查询使用 status + claimed_at；消费索引第二列为 mac，且不包含 claimed_at`
- failure_scenario: websocketGo 故障产生大量 running 行时，恢复任务无法通过消费索引定位超时行，形成扫描和积压放大。
- accepted_fix: 增加独立 `(status, claimed_at)` 索引；不得声称 `(status, mac, next_attempt_at)` 可以覆盖该查询。

### WS-004

- severity: `P1`
- category: `online-state`
- violated_contract: `缓存命中不能建立在可能永不过期的旧在线集合上`
- required_evidence: `历史 UpdateClientStatus 在线数为零时不覆盖写空数组；PostCmdNoWait 又不返回逐 MAC 失败`
- failure_scenario: 终端已离线但仍残留在 Redis 快照中，动作消息被请求后硬删。
- accepted_fix: 每周期写完整结果，包括 `[]`；增加 TTL 或 generation/freshness，缺失、过期或解析失败只表示“不确认在线”。

### WS-005

- severity: `P2`
- category: `architecture-boundary`
- violated_contract: `dispatcher 不应被描述为在线缓存写入方`
- required_evidence: `ClientOnlineMacCacheKey 由 UpdateClientStatus 整体刷新`
- failure_scenario: 实施者按设计给 dispatcher 增加第二条 Redis 写路径，造成在线事实来源分裂。
- accepted_fix: 明确 dispatcher 只读，缓存缺失或解析失败只表示无法确认在线。

## Non-goal

### WS-NG-001

- disposition: `non-goal`
- statement: 不要求迁移、补偿或双读旧 `system_websocket_*` 待发数据。
- rationale: 用户已明确接受前向切换和旧数据废弃；reviewer 只能记录残余风险，不得升级为 P0/P1。

## Historical sources

- Initial design: `019f4480-5a74-7560-b73b-e32aab3e508a`
- Review/revision: `019f460a-8faf-7510-8894-e10fd40e71ea`
- Final correction review: `019f4630-fe61-76d1-bcbe-0cdd1e7f129a`
