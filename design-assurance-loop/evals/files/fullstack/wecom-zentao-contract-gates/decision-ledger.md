# Gold Decision Ledger

> 仅供 evaluator 使用，不得作为被测 agent 输入。

## Accepted findings

### WZ-001

- severity: `P1`
- category: `implementation-order`
- violated_contract: `未经验证的外部契约不能成为后续实现的基础`
- required_evidence: `view 读取、父子关系和状态动作均未知；计划到最后才做真实验证`
- failure_scenario: 前四阶段围绕错误 payload 和 fixture 完成大量代码，最后发现 view 无法过滤或 parent 写法无效，导致整体返工或错误写入。
- accepted_fix: 增加最前置 contract gate；后续任务只消费已验证、脱敏的企微和禅道 fixtures。

### WZ-002

- severity: `P1`
- category: `api-contract`
- violated_contract: `子任务必须在查询结果中形成 direct parent`
- required_evidence: `v1 create 字段没有 parent；历史官方答复要求创建后设置父关系`
- failure_scenario: API 返回创建成功，但新任务仍是顶级任务；企微写回 child ID 后系统把错误结构当成同步成功。
- accepted_fix: 先验证唯一可用的 post-create/edit 路径，创建后重新查询并校验 execution 与 direct parent；失败时报警并停止该组。

### WZ-003

- severity: `P2`
- category: `data-safety`
- violated_contract: `模板契约失败时必须保持零写入`
- required_evidence: `用户要求列类型不匹配时停止并给出可操作错误`
- failure_scenario: 同名列类型改变后被“尽量解析”为错误值，随后写入错误任务或覆盖企微 ID。
- accepted_fix: 预检同时校验列存在与类型；错误必须包含列名、期望、实际和修复动作，并证明两个外部系统均无写请求。

### WZ-004

- severity: `P2`
- category: `operability`
- violated_contract: `非直观默认值应说明运维理由`
- required_evidence: `estimate=80 是减少维护工作的用户决策，实际工时才是统计口径`
- failure_scenario: 后续维护者把 80 当成估算算法或擅自改动，造成任务口径漂移。
- accepted_fix: 保留固定默认值并写明理由，不扩展为动态估算模块。

## Non-goals

### WZ-NG-001

- disposition: `non-goal`
- statement: 不把多进程并发、重复同步锁或队列容量上限作为 P0/P1 阻塞项。
- rationale: 用户已明确接受小规模单进程边界；可记录 residual risk，不得重新扩大本期范围。

### WZ-NG-002

- disposition: `accepted-risk`
- statement: ID 为空时复用唯一精确同名任务属于已确认业务规则，不得重新判为 P1 所有权缺陷。
- rationale: 用户已明确确认“设计如此”。

### WZ-NG-003

- disposition: `accepted-risk`
- statement: 不强制内网禅道 base URL 使用 HTTPS。
- rationale: 用户明确说明目标内网禅道可能只部署 HTTP；reviewer 可记录残余风险，不得升级为阻塞项。

## Historical sources

- Initial design/plan: `019f5a83-9c6d-7302-9c7f-b3ed391a2bb6`
- Review and user adjudication: `019f5b82-b16d-7573-b009-af3f0368b130`
