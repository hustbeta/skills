# Eval Catalog

> Evaluator-only。该文件用于分类、Gold 裁决和 held-out 生命周期管理；不得复制到被测 executor 的临时目录。`evals.json` 仍是工具执行清单。

## 使用规则

- `historical-regression` 可使用定向 prompt，并可参与 Skill 调优；它只能证明已知缺陷未回归，不能证明跨项目通用性。
- `transfer-heldout` 和 `negative-control` 使用 blind prompt；当前优化周期内不得根据其结果修改 Skill。
- held-out 一旦被用于调优，立即转为 `historical-regression`，并补充新的 held-out。
- `metamorphic` 保留失败机制和影响，替换领域、实体和术语，用于检查 reviewer 是否识别机制而不是记忆表面词。
- `H*`、`C*`、`M*` 是 catalog 临时编号，不是正式 eval ID。只有状态为 `candidate` 且用户确认 Gold 后，才创建附件并加入 `evals.json`。
- `rejected-*` 只保留裁决记录，不创建案例，也不计入评测覆盖率。

## Gold 表述层级

- Skill、rubric 和评分能力使用通用机制，例如“检查共享实现的全部入口、调用方和消费者”，不写产品名或固定符号。
- blind prompt 只描述评审任务和附件类型，不提示“存在复用”“还有其它入口”等预期缺陷。
- 输入证据与 case-specific Gold 保留真实代码名和产品入口，用于证明 finding 有事实依据。
- grader 按“通用能力 + 具体证据”判定，不要求命中特定产品词；只给出“建议检查调用方”但没有引用实际入口，不算通过。

## 案例目录

| ID | 评测层 | 领域与来源 | Prompt 方式 | 通用机制 | Proposed finding | Must-not-raise | Gold 置信度 | 状态 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | historical-regression | Go backend；`evals.json#1`；历史来源见对应 decision ledger | directed：核对 WebSocket 出站调度设计、证据和已确认边界 | 请求成功与逐项完成、并发所有权、恢复索引、缓存新鲜度 | P1：请求级 `ok` 不能证明逐 MAC 送达；旧 claimant 不能删除新 payload；另含 stale-running 索引和完整在线快照问题 | 不把已接受的旧 `system_websocket_*` 数据不迁移升级为 P0/P1；不新增第二条在线缓存写路径 | high | registered；gold_status=pending-user-confirmation |
| 2 | historical-regression | Go backend；`evals.json#2`；历史来源见对应 decision ledger | directed：核对 backpressure 与 signal-pprof 计划的谓词、一致性和验证 | 生产者/消费者谓词对齐、正常路径覆盖、诊断生命周期 | P1：backlog 必须统计可 claim 的 due pending 加 running；cap fixture 与期望必须一致；另含正常路径、CPU 负载和文件名验证问题 | 不增加 HTTP pprof；不修改正常 PING/PONG 或 channel-closing 日志；不把降低 claim limit 当作默认排空手段 | high | registered；gold_status=pending-user-confirmation |
| 3 | historical-regression | 跨系统集成；`evals.json#3`；历史来源见对应 decision ledger | directed：核对企微到禅道同步的外部契约、顺序和零写入门禁 | 外部契约前置验证、父子关系后验校验、失败零写入 | P1：未知外部契约必须先验证；创建成功不等于 direct parent 已成立；固定列契约失败时两个系统保持零写入 | 不把已接受的小规模单进程、精确同名复用或内网 HTTP 升级为 P0/P1；不扩展动态估算 | high | registered；gold_status=pending-user-confirmation |
| H1 | excluded-non-design | Python/Django migration；project-manager；`019fcc25-7fff-7f11-8558-d4f48c435fda` | 不适用：不创建 eval prompt 或附件 | 迁移与发布操作纪律，不属于本 Skill 的设计评审边界 | 无设计 Gold：该故障由人工跨分支处理 migration 不当引发，不是待评审设计中的错误决策 | 不把操作失误改写为设计缺陷，也不把本案例计入 held-out 或通用性证据 | user-adjudicated | rejected-non-design |
| H2 | historical-regression | Python/Django auth；`evals.json#4`；历史来源 `019f461c-4ad6-71b1-a6d3-5bf84227bb5a` | blind：评审首登强制改密接口的最小权限边界方案；脱敏附件提供路由、中间件、处理器和登录状态代码，不提示旁路或复用结论 | 多层认证旁路、正向授权、共享实现的多入口与多消费者影响面 | P0：未授权调用方可选择目标用户并修改凭据，敏感操作需要可验证的首登授权；P1 范围 finding：共享实现还存在目标功能外的实际入口，是否同步修改记为 `requires-user-decision` | 不默认要求普通 `request.user` 登录态；不改变既有请求/响应 JSON；部署暴露记为运行态 unknown，但不降低设计 P0；未经范围裁决不自动修改其它入口 | finding/P0=user-confirmed；其它入口处置=requires-user-decision | registered-historical-regression；do-not-tune |
| H3 | historical-regression | Python/cloud；`evals.json#5`；历史来源 `019f985c-ed28-7d51-8571-faf9ced3673c` | blind：评审资源规格变更与业务元数据同步方案；脱敏附件提供 API 契约、状态转换、写入路径和失败样本，不提示最终状态结论 | 异步请求接受与最终完成、权威事实源、条件提交、失败收敛 | P1：异步命令被接受不能作为有效业务状态的提交条件；派生业务状态必须以基础设施提供方最终状态为权威，guest 只作运行态佐证；失败或回滚后必须收敛 | 不假定缺少 confirm 操作就是根因；不以请求目标或 API 接受作为运行态；不只凭 guest 状态推断规格；不强制队列、轮询或事件实现；不为验证重放生产操作；终态触发器、状态枚举、客户端成功语义和规格映射缺少证据时保持 unknown 发布门禁，不升级为四个 P0 | finding/P1=user-confirmed；unknown-gate=user-confirmed；实现机制开放 | registered-historical-regression；do-not-tune |
| H4 | transfer-heldout（跨层补充） | Vue frontend；tob-server；`019f1b47-2bc5-7532-8638-bd5c34e7d08e` | blind：评审列表筛选与分页调整方案；附件提供组件结构、请求依赖和打开/关闭代码，不提示组件复用或 reset 缺口 | 影响请求的状态、共享组件实例、初始化与清理契约 | P2：新增到请求依赖的父级状态未在业务对象切换时重置、重绑定或明确保留，导致前一个对象的状态泄漏到下一个对象；设计必须覆盖所有影响请求的状态 | 不要求后端或 API 重构；不引入全局状态或强制 remount；优先局部 reset；没有实际状态依赖证据的泛泛提醒不算 finding | finding/P2=user-confirmed；implementation-derived；不计入主要跨项目门禁 | candidate |
| C1 | negative-control-regression | Python API design；`evals.json#6`；历史来源 `019fcbe0-6160-7432-a42f-6f335d26875a` | blind：评审外部日期口令 API 设计；脱敏附件提供现有认证链、接口契约、需求决策、威胁模型、日志边界和 accepted-risk，不提示预期 PASS | 误报控制、决策边界尊重、残余风险与阻塞缺陷分离 | 无 P0/P1；读取早于 IP 限流会产生不受连接数约束的累计正文读取成本，但单请求仅增加最多 1024 字节且缺少可用性影响证据，故为 P2；上游限制只影响规模和资源消耗位置 | 不要求新增权限或 migration；不强制迁入其它路径；不要求成功审计正文或口令 hash；不重开旧内部 GET 旁路；没有相反可用性契约时不把 Redis fail-closed 升级为 P1；不建设通用 API 引擎 | boundaries/user-accepted-risk/P2=user-confirmed | registered-negative-control-regression；do-not-tune |
| M1 | metamorphic | synthetic 批量文件转换；机制源自 eval 1 的 WS-001，无历史功能对应 | blind：评审批量文件转换与任务状态管理方案；附件提供提交 API、worker 行为、逐项状态表和清理路径，不提示成功信号层级 | 请求级、任务级和逐项 outcome 与破坏性清理范围 | P1：batch accepted 不能证明每个文件处理项完成；允许清理的对象集合不得超过已有证据确认完成的对象集合，失败或结果未知项必须保留可恢复状态 | 不引入 WebSocket、MAC、Redis、ACK 或原函数名；不强制同步、exactly-once、消息队列或 outbox；只泛泛建议重试但未识别成功语义错位不算 finding | mechanism/P1=user-confirmed；synthetic artifacts pending | candidate |
| M2 | metamorphic | 合成数据导出；机制源自 H2 的范围 finding | blind：评审交互式数据导出的字段脱敏调整；附件提供入口、共享转换服务和调用代码，不提示其它消费者 | 共享实现的多入口与多消费者影响面 | P1：共享脱敏服务还被定时归档任务调用，设计中“只影响交互导出”的结论不成立；是否同步改变归档输出记为 `requires-user-decision` | 不出现 route、View、VDI、VOI 或原符号；不因存在复用就断言定时任务必须修改；没有实际调用证据的泛泛提醒不算 finding | synthetic；机制已获用户确认 | candidate |
| 7 | historical-regression（synthetic/metamorphic） | synthetic 多租户文件导出；附件 `evals.json#7` | blind：评审多租户导出下载设计；附件提供入口、会话、任务解析与调用证据，不提示跨租户或其它消费者结论 | 不可信对象标识的资源归属校验、共享实现的实际消费者、设计 finding 与运行态 unknown 分离 | P0：用户可控对象 ID 仅全局查询且返回前未核对归属会破坏租户隔离；共享实现存在交互范围外的实际消费者，其处置需用户裁决 | 不要求全局权限/身份框架、额外加密或审计平台、数据库迁移；不因发现共享消费者自动扩大范围 | synthetic；mechanism/user-adjudicated | registered-historical-regression；tuning-source=iteration-2 |
| 8 | negative-control-regression（synthetic） | synthetic 有界清单准入；附件 `evals.json#8` | blind：评审外部有界清单准入设计；附件提供处理顺序、容量边界和已接受风险，不提示严重 finding 结论 | 单请求上限与并发累计成本、accepted risk、unknown 发布门禁、误报控制 | P2：正文读取/解析早于准入与应用并发配额会造成累计成本；缺少可用性影响证据时不升级 P0/P1 | 不重开已接受的授权、migration、正文记录或内部入口边界；不建设通用 API 框架或迁移既有内部入口 | synthetic；accepted-risk/P2=user-adjudicated | registered-negative-control-regression；tuning-source=iteration-2；scoring-boundary=semantic-order-sufficient |
| 9 | historical-regression（synthetic） | synthetic 后端任务调度与关系数据库；附件 `evals.json#9` | blind：评审执行池标识在线迁移；附件提供滚动拓扑、writer、一次回填、reader、历史别名和运行证据边界，不提示 finding 或修复顺序 | 新旧 writer 共存、在线 schema 演进、并发安全回填、稳定零漂移与 cutover 门禁 | P1：一次回填不能覆盖其后的旧 writer 写入；新 reader 只读 ID 会把持久化分叉变成错误池或持续失败；模糊历史别名处置需用户裁决 | 不要求停机、通用迁移平台或改变外部 API；真实流量、实例、表规模、批大小和部署结果缺失不独立生成或升级 P0/P1 | synthetic；mechanism/user-confirmed | registered-historical-regression；observed-in=iteration-4；frozen-cycle=iteration-4；do-not-tune |
| 10 | negative-control-regression（synthetic） | synthetic 后端配置分发；附件 `evals.json#10` | blind：评审签名功能配置缓存；附件提供完整性、revision、原子切换、受限 LKG、只减权覆盖、accepted risk 和发布证据边界，不提示 PASS 或严重度 | 误报控制、安全配置链、已接受降级、P2 可观测性与 unknown 发布门禁 | 无 P0/P1；P2：缺少每实例生效 revision、配置年龄、LKG 和紧急覆盖状态，会延迟漂移发现和定位，但不阻塞设计 PASS | 不重开 60 秒传播差异或 24 小时签名有效 LKG；不要求逐请求控制面访问、全局强一致、共识、队列、migration 或通用策略平台 | synthetic；accepted-risk/P2/user-confirmed | registered-negative-control-regression；observed-in=iteration-4；frozen-cycle=iteration-4；do-not-tune |

## C1 独立门禁记录

- reviewer 使用 fresh context，只读原设计、当前相关代码和 review rubric；未读取本 catalog、`evals.json` 或 decision ledger。
- 结论：`gate-passed`，未发现有文件证据支持的 P0/P1；accepted risk 与 non-goal 表述清楚。
- P2：当前实现先读正文再做来源 IP 限流；1024 字节只限制单请求成本，同一来源的 N 个并发连接仍可形成约 `N × 1024` 字节累计读取。上游限制只影响规模和资源消耗位置；缺少可达到并发量、worker/连接模型、读取超时和可复现服务降级证据，因此当前不升级为 P1。
- unknown：未执行测试，也未验证生产 TLS、Redis、可信代理 CIDR 和密钥配置。
- 本记录已用于正式化 eval #6，并经 iteration-1 benchmark 与人工裁决确认；案例现为 negative-control regression，不再作为 held-out 证明通用性。

## 隔离边界

未来 benchmark 只复制 `SKILL.md`、所需 `references/` 和 `evals.json.files` 明列的输入。必须排除本 catalog、`evals.json`、expectations 和所有 `decision-ledger.md`。
