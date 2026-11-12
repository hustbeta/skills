# Design Assurance Loop 冻结后盲测设计

## 背景

`design-assurance-loop` iteration 3 已完成定向调优并冻结 candidate-1：

- Skill SHA-256：`973BB0231626B50146C850ABA43A4EABE49ECB039247228AF2A2FB78F51D245E`；
- rubric SHA-256：`26AC2C99A808F583FBF939CBF685DCDD2AEC8F455AE4C6F16FFB3DDF3CAB096B`；
- 状态：`frozen-pending-fresh-validation`；
- `do_not_tune_on_fresh_validation: true`；
- `generality_claim: false`。

已知历史开发回归只能证明旧缺陷未回归。本轮在冻结后创建两个全新、脱敏、自包含的合成案例：一个 transfer-heldout，一个 negative control。案例机制、产品实体和固定数值不得复用 eval 7/8 的直接答案。结果只能评价冻结候选，不得在同一周期反向调参。

## 目标

- 检查 candidate-1 的通用证据、严重度、累计影响、unknown 和范围裁决规则能否迁移到新机制。
- 使用调优前 Skill 快照作 baseline，隔离本轮通用规则增量。
- 同时测量严重设计问题召回和 negative-control 误报控制。
- 保留原始执行、评分、人工反馈和隔离证据，不用汇总分覆盖逐项门禁。

## 非目标

- 不在盲测后修改 frozen candidate-1。
- 不把两个合成案例外推为全面通用性证明。
- 不新增 benchmark harness、grader 框架、CI、运行依赖或通用案例生成器。
- 不把案例专用词、函数名、实体名、固定数值或 Gold 写入 Skill/rubric。
- 不在规格确认前创建 eval 输入、Gold、assertions 或运行目录。

## 比较配置

### Treatment

使用冻结候选：

- `design-assurance-loop-workspace/iteration-3-tuning/frozen-candidate/SKILL.md`；
- `design-assurance-loop-workspace/iteration-3-tuning/frozen-candidate/references/review-rubric.md`。

运行前重新验证文件哈希与 frozen manifest 一致。

### Baseline

使用调优前 Skill 快照：

- `design-assurance-loop-workspace/iteration-2/skill-under-test/SKILL.md`；
- `design-assurance-loop-workspace/iteration-2/skill-under-test/references/review-rubric.md`。

预期哈希分别为：

- Skill：`B548FEF3D9E7142D761E6CC0979E5F9D0DE8C09B2E4E7100B9FBF92DB131B7B0`；
- rubric：`D36064ABA6DC2A3E9C249BB1F89E3B9F6BF6CCD2DA3FBEF59B22CF7BF757928F`。

本轮不增加无 Skill 第三组。原因：目标是比较本轮调优前后差异，而不是重新测量整个 Skill 相对于无流程评审的价值。

## 案例 9：滚动发布下的在线 Schema 演进

### 定位

- 评测层：`transfer-heldout（synthetic）`；
- 领域：通用后端任务调度与关系数据库；
- 核心机制：旧/新 writer 共存、一次性回填、读路径切换和 schema contract 顺序；
- 预期最高 finding：P1；
- 与旧案例差异：不使用跨租户授权、共享下载 resolver、有界正文准入、异步 provider 终态或 WebSocket 交付语义。

### Blind prompt

评审并修订附件中的执行池标识在线迁移设计。沿附件证据核对滚动发布兼容、数据回填、读写切换、约束启用、回滚和历史异常数据；无法验证的内容标记为 unknown。输出最终设计、验收覆盖矩阵、裁决账本和残余风险。

prompt 不提示旧 writer、一次性回填缺口、P1、模糊别名或期望阶段顺序。

### 脱敏设计场景

任务调度服务把自由文本 `execution_pool_name` 在线迁移为外键 `execution_pool_id`。目标要求：

- 零停机滚动发布；
- 外部 API 请求与响应不变；
- 新调度器最终只按稳定 ID 选择执行池；
- 回滚窗口结束后删除旧字段；
- 不建设通用迁移平台。

拟议方案先新增可空 ID 列，运行一次全量回填，随后切换新调度器只读 ID，最后启用约束并删除旧名称列。

### Executor 可见 evidence

附件应直接、但不替 reviewer 下结论地证明：

1. 滚动发布期间旧、新服务实例共存。
2. 旧实例创建或更新任务时只写 `execution_pool_name`。
3. 当前回填只扫描一次，完成后记录单一完成标志；没有增量追赶、版本比较或重复零漂移验证。
4. 方案没有在读切换前证明旧 writer 已清退，也没有阻止旧 writer 继续写入。
5. 新调度器切换后只读 `execution_pool_id`；该字段为空时使用默认池。
6. 历史数据含少量无法唯一映射到 ID 的别名；附件不提供其业务归属判断。
7. 附件没有真实生产实例清单、流量、表规模、执行结果或部署结果。

### Evaluator-only Gold

#### P1 finding

- violated contract：新读路径启用前，所有可能写入目标记录的版本必须维护新表示，或被证明已清退/隔离；一次回填不能覆盖其后的旧 writer 写入。
- evidence chain：`一次回填完成 → 旧实例继续只写名称 → ID 为空或陈旧 → 新调度器只读 ID → 使用默认池或持续失败`。
- failure impact：滚动窗口内新增或更新的任务持续进入错误执行池或无法正确调度；错误状态进入持久化数据并需要专项对账修复。
- severity：P1。证据证明持续、批量的数据分叉和错误业务执行；没有不可恢复的大规模数据丢失、重大安全事故或核心系统全面失效证据，因此不升 P0。
- runtime boundary：真实实例、流量与故障规模缺失是发布态 unknown，不是失败链成立的必要条件，不降低设计 P1。

#### Accepted repair

1. Expand：新增可空 ID 和必要索引，不立即删除旧字段。
2. Compatibility：先部署同时维护两种表示的新 writer；读路径保留旧字段回退。
3. Migrate：使用能识别并发更新的条件回填，避免旧快照覆盖较新业务写入。
4. Reconcile：重复扫描漂移；只有稳定观察窗口内差异为零才允许读切换。
5. Fence：通过实例清单、版本门禁或等价机制证明旧 writer 已清退或不能再写。
6. Cutover：切换新读路径并验证 null、错误映射和默认池均不可静默发生。
7. Validate：数据与 writer 门禁通过后再验证非空/外键约束。
8. Contract：回滚窗口结束后才删除旧字段和兼容逻辑。

具体框架、数据库品牌、编排平台和批处理实现保持开放。

#### 范围裁决

- accepted：零停机滚动发布；外部 API 不变。
- `requires-user-decision`：无法唯一映射的历史别名应阻断切换、隔离记录还是人工修复。不得自行填默认 ID、删除或静默跳过。
- non-goal：通用迁移平台、永久兼容任意旧版本、顺带重构业务模型。
- must-not-raise：未提供真实流量、部署结果、表规模或具体批大小不能独立生成或升级 P0/P1；不要求停机窗口。

### Assertions

1. 把一次回填、旧 writer 继续写旧字段、新读路径只读 ID 和默认池/失败串成 P1。
2. 把生产实例、流量和部署结果标为 unknown，且不降低已成立 P1。
3. 要求兼容 writer 与旧 writer 清退/隔离门禁先于读切换。
4. 要求并发安全回填、重复 reconciliation 和稳定零漂移门禁，之后才验证约束和 contract。
5. 把无法唯一映射的历史别名处置标为 `requires-user-decision`，不发明默认映射。
6. 保持零停机、外部 API 和非目标边界，不强制通用迁移平台或停机发布。

## 案例 10：签名配置缓存与已接受降级

### 定位

- 评测层：`negative-control（synthetic）`；
- 领域：通用后端配置分发；
- 核心机制：签名配置、单调 revision、原子替换、last-known-good 和只减权紧急覆盖；
- 预期最高 finding：P2；
- 预期设计结果：PASS，无文件证据支持的 P0/P1。

### Blind prompt

评审并修订附件中的功能配置缓存设计。沿附件证据核对完整性、版本切换、控制面故障、紧急覆盖、可观测性、验证门禁和已接受风险；无法验证的内容标记为 unknown。输出最终设计、验收覆盖矩阵、裁决账本和残余风险。

prompt 不提示预期 PASS、P2、监控字段或 must-not-raise。

### 脱敏设计场景

服务每 60 秒从控制面拉取功能配置。业务接受最多 60 秒实例传播差异；控制面故障时允许最多 24 小时使用已验证的 last-known-good。高风险处置使用独立本地紧急覆盖。

### Executor 可见 evidence

附件应证明：

1. 配置 bundle 使用固定公钥验证签名；验证失败不替换当前配置。
2. revision 必须单调递增；旧 revision 和相同 revision 的不同内容均拒绝。
3. bundle 先写临时文件、同步并原子替换，再更新内存引用。
4. 拉取失败时只使用最近一次验证成功的配置，最长 24 小时；超时后进入关闭可选功能的安全基线。
5. 本地紧急覆盖每 5 秒读取，只能强制关闭功能，不能开启；覆盖结果优先于远端配置。
6. 业务明确接受最多 60 秒的实例传播差异和 24 小时 last-known-good 窗口。
7. 当前监控只有控制面拉取成功/失败，没有每实例生效 revision、配置年龄、last-known-good 状态或紧急覆盖状态。
8. 附件没有生产签名密钥配置、系统时钟、覆盖文件权限、目标测试执行或真实部署证据。

### Evaluator-only Gold

#### Negative-control gate

- expected blocking findings：0。
- expected result：没有证据支持的 P0/P1；允许有证据的 P2/P3、unknown 和 residual risk。
- failure condition：把已接受传播差异、last-known-good 或缺失运行证据升级为 P0/P1；要求与现有契约无关的全局强一致或平台化方案。
- exception：若 reviewer 从输入中证明新的真实 P0/P1，应保留 finding 并将案例移出 negative control，不为保持 PASS 压制证据。

#### Allowed P2

- violated contract：运维验收需要知道每个实例实际使用的配置及其新鲜度，当前监控只能说明控制面拉取请求是否成功。
- evidence：没有每实例 effective revision、bundle age、using-last-known-good 和 emergency-override-active 状态。
- failure scenario：部分实例因本地替换或后续应用异常长期停留旧配置，控制面成功率仍显示健康，导致漂移发现和定位延迟。
- severity：P2。已有签名、单调 revision、原子替换、有效期、安全基线和只减权覆盖；没有严重事故、核心不可用或高概率生产故障证据。
- recommended fix：增加低基数的实例生效 revision、配置年龄、last-known-good 和紧急覆盖状态指标与接近过期/实例漂移告警。P2 不阻塞设计 PASS。

#### 范围裁决

- accepted risk：最多 60 秒实例差异；最多 24 小时使用签名有效的 last-known-good。
- unknown release gates：生产公钥/密钥配置、系统时钟、覆盖文件权限、目标测试结果和真实部署状态。
- must-not-raise：不要求逐请求访问控制面、全局强一致、共识、消息队列、数据库 migration 或通用策略引擎；不得把 last-known-good 描述为未经约束的 fail-open。

### Assertions

1. 没有输入证据支持的 P0/P1；严重 finding 必须证明违反未被接受的契约。
2. 指出缺少实例生效 revision、配置年龄、last-known-good 和紧急覆盖状态是 P2 可观测性缺口，不升级 P1。
3. 不重开已接受的 60 秒传播差异和 24 小时 last-known-good 窗口。
4. 保留签名、单调 revision、原子替换、过期安全基线和只减权紧急覆盖。
5. 把生产密钥、时钟、文件权限、目标测试和真实部署标为 unknown 发布门禁，不伪装成已验证或严重 finding。
6. 没有已接受 P0/P1 时允许设计 PASS，不要求全局强一致或通用平台。

## 输入、Gold 与运行隔离

每案在运行前冻结三类文件：

- executor-visible：`inputs/design.md`、`inputs/evidence.md`；
- evaluator-only：`eval_metadata.json`、`grader-only/decision-ledger.md`；
- configuration-specific：Treatment 或 baseline 的 Skill/rubric 快照。

executor 消息、临时目录和 reviewer 输入不得包含：

- catalog、assertions、decision ledger 或本设计；
- 其它 run、另一配置输出或历史 benchmark；
- “存在 P1”“预期 PASS”“检查旧 writer”等答案提示。

输入、Gold、Skill、rubric 和执行模板在第一个 run 前记录 SHA-256。运行后不得改动；若发现 Gold 证据不足，停止对应案例并等待用户裁决，不边跑边修。

## 执行拓扑

每案执行 3 个 Treatment 和 3 个 baseline，共 12 个 run。每个 run 使用独立新任务：

1. 任务自身为根 agent，读取指定 Skill/rubric 和 blind inputs。
2. 根 agent 构造 canonical brief 与 evidence index。
3. 同时启动证据与契约、故障与安全、可执行性与验证三个 fresh-context 只读 reviewer。
4. 根 agent 裁决并只修订已接受 P0/P1。
5. 启动未参与前序评审的 fresh-context final reviewer。
6. 保存最终输出和执行 provenance。

当前协调任务不得包裹一个 Treatment executor 再让其启动三个 reviewer；四槽环境会导致 reviewer 无法完整并行。Treatment 与 baseline 按配对顺序串行运行，避免槽位竞争。每个执行记录至少包含根任务 ID、配置、run number、模型、推理强度、三个初始 reviewer ID、并行与上下文隔离状态、final reviewer ID、最终结论、可用时间/令牌数据及缺失说明。

用户在实施阶段按逐步操作说明创建独立任务、粘贴对应 run prompt、等待完成并返回任务 ID。未满足独立根任务与三 reviewer 并行时，保存结果但标记 `REDUCED_INDEPENDENCE`，不计入主要 `3/3` 门禁，并使用新 run number 补跑。

## 评分、比较与人工复核

### 定量评分

每个有效 run 由专用 fresh-context grader 按六条 assertions 评分。grader 读取 grader 规范、该案 Gold、metadata、执行 provenance 和最终输出；不得修改输入、Gold 或 executor 输出。记录 grader task ID、请求的上下文隔离、输入与写入边界；完整 transcript 不可得时明确披露。

聚合报告同时给出：

- 每条 assertion 的 Treatment 与 baseline `x/3`；
- 两配置微平均结果；
- P0/P1 recall、false P0/P1、P2 calibration、unknown accuracy、scope/disposition 和 closure；
- 实际可用的时间与 token；零值 sentinel 不参与效率结论。

### 盲 A/B 比较

每案启动一个 fresh-context comparator。该 comparator 读取同一 blind input 和三组成对输出；每对随机标为 A/B，不提供配置标签、Gold、assertions 或其它 pair。分别判断：

- 证据链准确性；
- 严重度校准；
- 修订闭环；
- unknown 与范围边界；
- 总体 winner 或 tie。

平局允许。baseline 因实质正确性胜出时必须单独分析；仅因篇幅或写作偏好胜出不自动判定冻结候选失败。

### 人工复核

使用 skill-creator 静态 review viewer 展示十二个输出、正式评分、盲比较和 benchmark。用户确认每案：

- Gold 是否仍由输入充分支持；
- grader 是否按通用能力和具体证据评分；
- 是否存在未被 assertions 捕获的严重漏检或误报；
- Treatment 与 baseline 的差异是否具有实际评审价值。

用户反馈保存到本轮 workspace，不覆盖 executor、grader 或 comparator 原始产物。

## 硬门禁

### 案例 9 Treatment

- P1 链路：`3/3`；
- runtime unknown 不降低 P1：`3/3`；
- 兼容 writer 与旧 writer 门禁：`3/3`；
- 并发安全回填和稳定零漂移门禁：`3/3`；
- 模糊历史数据 `requires-user-decision`：`3/3`；
- 零停机、API 和非目标边界：`3/3`。

### 案例 10 Treatment

- 错误 P0/P1：`0/3`；
- P2 可观测性与严重度：`3/3`；
- accepted-risk 不重开：`3/3`；
- 安全配置链保留：`3/3`；
- unknown 与设计 finding 分离：`3/3`；
- 设计 PASS 且无平台化扩张：`3/3`。

总分不能替代上述命名门禁。任何 `REDUCED_INDEPENDENCE` run 不计入 `3/3`。

## 结果解释

- Treatment 通过、baseline 失败同一命名能力：本轮通用规则获得正向迁移证据。
- 两者都通过：冻结候选获得该案例的迁移证据，但不能证明本轮 delta 是原因。
- Treatment 失败：保留冻结版本和全部原始结果；不得在本轮修改 candidate-1。
- Treatment 比 baseline 差：记录潜在回归，留待下一调优周期。
- 新 Gold 争议：标记 `NEEDS_CONTEXT`，停止该案例，不改答案迁就输出。
- 两案 Treatment 均通过：只报告“冻结候选在两个全新合成案例中获得有限迁移证据”。不得声称跨所有代码库、语言或后端设计全面通用。

盲测完成后，两案转为 historical regression。若暴露稳定缺口，在下一周期以新的 candidate-2 执行 RED/GREEN；本轮 candidate-1 和原始盲测继续只读。

## 用户操作边界

当前规格阶段不需要用户执行任务。实施计划确认后，用户需：

1. 按提供的顺序创建独立新任务；
2. 每次粘贴完全给定的 run prompt，不添加 Gold 提示；
3. 等待该任务完成；
4. 返回任务 ID；
5. 按判断标准确认任务是否真正使用三个并行 reviewer 和独立 final reviewer。

实施计划将为每个操作给出准确文本、预期状态和失败处理。未确认 Gold、哈希或隔离条件时不开始正式运行。

## 通过后的下一步

本设计经用户书面确认后，编写 `docs/plans/2026-08-12-design-assurance-loop-post-tuning-blind-validation.md`。实施计划先创建并冻结附件与 Gold，再生成 12 个独立 run prompt；不会直接启动正式 eval。
