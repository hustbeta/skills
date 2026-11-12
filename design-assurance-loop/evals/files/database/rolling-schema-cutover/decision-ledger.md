# Evaluator-only Gold

本文件不得提供给 executor 或其 reviewer。

## E9-P1-001 滚动窗口中的新旧表示分叉

- severity: P1
- disposition: accepted finding
- violated contract: 新读路径启用前，所有仍可能写入目标记录的版本必须维护新表示，或者被证明已清退、隔离或禁止继续写入。
- required evidence: `E9-01` 证明新旧版本共存；`E9-02` 证明旧 writer 只写名称；`E9-03` 证明回填只扫描一次；`E9-04` 证明 cutover 没有旧 writer 门禁；`E9-05` 证明新 reader 只读 ID 并对空值使用默认池或失败；`E9-06` 证明错误池或持续失败的具体后果。
- evidence chain: `一次回填完成 → 旧实例继续只写名称 → execution_pool_id 为空或因并发更新而陈旧 → 新调度器只读 ID → 使用错误的默认池或持续失败`。
- severity basis: 该链路会在滚动窗口内持续产生持久化数据分叉和错误业务执行，需要专项对账修复。输入没有不可恢复的大规模数据丢失、重大安全事故或核心系统全面失效证据，因此为 P1，不升 P0。
- runtime boundary: 真实实例数量、流量、表规模和部署结果是发布态 unknown；它们不是该失败链成立的必要条件，不降低设计 P1。

## Accepted repair

1. Expand：新增可空 ID 和必要索引，不立即删除旧字段。
2. Compatibility：先部署同时维护名称与 ID 的新 writer；读路径保留名称回退，直到迁移门禁完成。
3. Migrate：条件回填必须比较名称、记录版本、更新时间或等价并发谓词，避免旧快照覆盖较新业务写入。
4. Reconcile：重复扫描空 ID、错误映射和名称/ID 漂移；只有稳定观察窗口内差异持续为零才允许读切换。
5. Fence：使用实例清单、最低版本门禁、写入拒绝或等价证据，证明旧 writer 已清退或不能再写。
6. Cutover：切换新读路径，并验证空 ID、错误映射和默认池均不会静默发生。
7. Validate：数据与 writer 门禁通过后再验证非空和外键约束。
8. Contract：回滚窗口结束后才删除旧字段与兼容逻辑。

具体数据库、编排平台、批处理框架和条件更新语法保持开放。

## 历史别名裁决

- disposition: requires-user-decision
- evidence: `E9-07` 证明部分别名无法唯一映射，且输入没有业务归属事实。
- decision required: 由用户决定阻断切换、隔离记录或人工修复；不得自行填默认 ID、删除或静默跳过。

## 已接受边界

- accepted: 零停机滚动发布；外部 API 请求与响应不变。
- non-goal: 通用迁移平台；永久兼容任意旧版本；顺带重构任务或执行池业务模型。
- must-not-raise: 缺少真实流量、生产实例、部署结果、表规模或具体批大小不能独立生成或升级 P0/P1；不要求停机窗口。

## Assertions

1. 把一次回填、旧 writer 继续写旧字段、新读路径只读 ID 和默认池/失败串成 P1。
2. 把生产实例、流量和部署结果标为 unknown，且不降低已成立 P1。
3. 要求兼容 writer 与旧 writer 清退/隔离门禁先于读切换。
4. 要求并发安全回填、重复 reconciliation 和稳定零漂移门禁，之后才验证约束和 contract。
5. 把无法唯一映射的历史别名处置标为 requires-user-decision，不发明默认映射。
6. 保持零停机、外部 API 和非目标边界，不强制通用迁移平台或停机发布。
