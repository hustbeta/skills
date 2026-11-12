# Gold Decision Ledger

> 仅供 evaluator 使用，不得作为被测 agent 输入。

## Accepted findings

### BP-001

- severity: `P1`
- category: `contract`
- violated_contract: `backlog 必须与消费者可 claim 的 pending 谓词一致`
- required_evidence: `ClaimDueMessages 只消费 delay>0 且到期的 pending；设计统计全部 pending`
- failure_scenario: 大量未来才到期的 pending 被计入 backlog，push-package 长时间停止正常 claim；另一实现若漏掉 running，又会低估真实压力。
- accepted_fix: 分别有界统计 `due pending` 与 `running`，再求和；时间使用 MySQL 当前时间。

### BP-002

- severity: `P1`
- category: `test-consistency`
- violated_contract: `测试期望必须与 capped-per-category 规则和 fixture 一致`
- required_evidence: `fixture 有 running；设计声明每类独立 cap=1，却期望 Running=0`
- failure_scenario: 实施者为满足错误断言写出忽略 running 的统计逻辑。
- accepted_fix: `cap=1` 时期望 `DuePending=1, Running=1, Total=2`，另放一条未到期 pending 防止宽谓词误通过。

### BP-003

- severity: `P2`
- category: `verification`
- violated_contract: `新增分支必须同时证明阻断路径和正常路径`
- required_evidence: `计划只有 backlog=true 测试`
- failure_scenario: 重构后所有场景都跳过 claim，但唯一测试仍通过。
- accepted_fix: 增加 backlog=false 测试，证明 claim/send helper 仍执行；超时清理在两条路径都执行。

### BP-004

- severity: `P2`
- category: `executability`
- violated_contract: `实施计划必须自包含`
- required_evidence: `关键 helper 与清理逻辑仅写“同上一版计划”`
- failure_scenario: 新执行任务无法确定错误传播和清理顺序，只能猜测旧版本内容。
- accepted_fix: 在当前计划给出 helper 职责、调用顺序和错误语义，不引用未附带旧稿。

### BP-005

- severity: `P3`
- category: `performance`
- violated_contract: `调优建议不能与吞吐目标相反`
- required_evidence: `claimLimit=5 已是当前超时预算内的安全上限；继续降低会减少每轮吞吐`
- failure_scenario: backlog 已增长时继续降低 claim limit，使积压更慢而不是更快。
- accepted_fix: 保持安全上限；若吞吐不足，复核 HTTP 超时、批处理或架构，不把降 limit 当作排空手段。

### BP-006

- severity: `P2`
- category: `verification`
- violated_contract: `CPU profile 测试必须产生可采样的 CPU 工作`
- required_evidence: `Sleep 不产生稳定 CPU 工作`
- failure_scenario: 测试偶发生成空或弱 profile，无法稳定证明 start/stop 生命周期。
- accepted_fix: 测试使用短 CPU busy loop，并覆盖启动、停止和非空文件。

### BP-007

- severity: `P3`
- category: `operability`
- violated_contract: `连续诊断信号不应静默覆盖同类文件`
- required_evidence: `文件名只有秒级精度`
- failure_scenario: 同一秒连续触发相同 profile 类型时后一份覆盖前一份。
- accepted_fix: 文件名增加纳秒时间或确定性的冲突后缀。

## Non-goals

- 不建议增加 `net/http/pprof`；该方向违反已确认的信号和本地文件边界。
- 不修改正常 PING/PONG 与 channel-closing 日志。

## Historical sources

- Predicate/CPU lifecycle review: `019f1c08-fd86-7a83-9407-9403cca52da9`
- Normal-path/runtime review: `019f1c19-879a-7171-a938-ef1ce861edea`
- Consistency/self-contained/tuning review: `019f1c47-4efd-7e42-962b-6ef64fb784b4`
