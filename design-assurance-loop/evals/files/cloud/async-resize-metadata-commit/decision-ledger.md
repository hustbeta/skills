# Gold Decision Ledger

> 仅供 evaluator 使用，不得作为被测 agent 输入。

## Accepted findings

### H3-001

- severity: `P1`
- category: `state-authority`
- violated_contract: `异步命令被接受不能提交为已经生效的业务事实`
- required_evidence: `provider.resize 同步返回后立即覆盖 EffectiveProfile；提供方后续可失败并回滚`
- failure_scenario: 迁移失败后实际资源保持原规格，业务查询却长期显示目标规格，并可能让后续计费、配额或操作基于错误事实。
- accepted_fix: 在提供方确认最终成功后才更新 effective 状态，或显式分离 requested/pending/effective；不得把同步 API 返回当作最终完成。

### H3-002

- severity: `P1`
- category: `failure-convergence`
- violated_contract: `异步操作必须定义成功、失败、超时和回滚后的状态收敛路径`
- required_evidence: `历史样本已出现提供方回滚而业务表保留目标值；现有流程没有回滚写入或自动对账`
- failure_scenario: 只要后续阶段失败或回调丢失，派生业务状态就永久领先于基础设施终态，且只能依赖人工发现。
- accepted_fix: 以提供方最终规格记录为权威执行条件 reconciliation；更新需带资源 ID 和预期旧值门禁，避免覆盖新的并发变更。

## Accepted implementation freedom

- 可以采用 pending/effective 分离、最终成功后延迟提交、事件驱动或条件轮询对账中的任一最小方案。
- guest CPU/内存仅作运行态佐证；不能单独据此推断完整规格标识。

## Must-not-raise

- 不把“缺少 confirm 操作”直接断言为本次失败根因。
- 不把请求中的目标规格或提交调用成功当作权威运行态。
- 不强制引入队列、事件总线或某个固定轮询框架。
- 不要求通过重放生产 resize 验证设计。
- 终态触发器、状态枚举、客户端成功语义和最终规格映射缺少附件证据时，标为 `unknown` 发布门禁；不得把“未提供证据”直接解释成四个独立 P0。若后续代码或运行证据证明实现缺失或错误，再按实际影响新增 finding。

## Historical source

- `019f985c-ed28-7d51-8571-faf9ced3673c`
