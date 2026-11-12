# Design Assurance Loop 评测分层设计

## 背景

当前三个案例来自 WebSocket 出站、WebSocket backpressure/PProf 和企微禅道同步。它们有人工裁决和真实证据，适合防止已知问题回归，但不足以证明 `design-assurance-loop` 能跨项目、跨领域工作。

单个 eval 保持具体；通用规律只从多案例失败中提炼，并写入 Skill 或 rubric。不能用同一批案例同时推导规则和证明泛化。

## 目标

- 将当前三个案例明确归类为 `historical-regression` 开发集。
- 建立互不混用的回归、held-out、negative-control 和 metamorphic 评测层。
- 同时评估 finding 准确性、流程合规、边界遵守和运行成本。
- 只有未见案例证明收益后，才修改并宣称 Skill 具备通用效果。

## 非目标

- 本阶段不修改 `SKILL.md` 或 `references/review-rubric.md`。
- 本阶段不开发自定义 benchmark harness、CI 或 reviewer agent。
- 不把具体项目 finding 直接改写成 Skill 强制规则。
- 不运行 baseline、treatment 或描述触发优化。

## 评测分层

### Historical regression

用途：开发和回归。允许查看结果、分析失败并据此改进通用规则。

当前三个案例全部属于该层：

1. `websocket-outbound-contract`
2. `websocket-backpressure-pprof-plan`
3. `wecom-zentao-contract-gates`

这组案例不再承担跨项目泛化证明。

### Transfer held-out

用途：验证通用规则能否迁移。当前优化周期内不得根据其结果修改 Skill；一旦用于调优，就转入 historical regression，并补充新的 held-out。

首批至少覆盖三个互异领域：

- 数据库迁移、双写、一致性和回滚。
- 权限、安全、审计或敏感数据边界。
- 前端状态、兼容性、离线数据或缓存一致性。
- 部署、灰度、配置切换或服务回滚可作为第四案例。

案例优先来自真实历史设计和人工裁决。缺少可靠 gold 时不注册。

首批 held-out 的合并 gold 至少包含一个 P0/P1、一个 accepted-risk 或 non-goal，避免召回率和边界测试成为空判断。

### Negative control

用途：检查 reviewer 是否为了产生 finding 而制造 P0/P1。

至少一个案例应满足：设计主要路径成立，没有 P0/P1；允许存在已记录的 P2/P3、accepted-risk 或不阻塞主路径的 unknown。通过条件是正确输出 `PASS` 或带独立性标记的通过状态，且不扩大非目标。

### Metamorphic pair

用途：区分机制识别与关键词记忆。

从一个已验证案例保留同一不变量、失败机制和影响，替换领域、符号及表述。原案例和变体应得到一致的严重级别、证据链和 disposition；不得复制原项目专有答案。

## 案例与通用规律的边界

每个案例仍保存具体的 `design.md`、`evidence.md` 和 evaluator-only `decision-ledger.md`。另外维护人读 catalog，记录：

- eval ID 和案例名；
- 所属评测层；
- 领域和来源类型；
- 被测通用能力；
- prompt 是 directed 还是 blind；
- gold 是否经人工确认；
- held-out 是否已被用于调优。

`evals.json` 保持 skill-creator 兼容格式，不增加私有字段。分类信息放入 catalog，避免修改工具 schema。

## 输入和 Gold 隔离

- executor 只接收 `SKILL.md`、必要 references、prompt、`design.md` 和 `evidence.md`。
- executor 不得访问 `evals.json`、catalog、expectations 或 `decision-ledger.md`。
- historical regression 可使用 directed prompt；held-out 和 control 使用贴近真实用户请求的 blind prompt，不提示预期缺陷类别。
- Gold 必须由输入之外的历史裁决或独立人工判断产生，不能从被测输出反推。

## 评分面

### Finding 准确性

- P0/P1 recall。
- false P0/P1。
- evidence accuracy。
- accepted finding 的修复闭环和修订回归。

### 流程合规

- 事实、推断和 `unknown` 分离。
- P0/P1 有可复现失败场景和证据。
- accepted-risk 与 non-goal 未被重新升级。
- 裁决账本包含 accepted、rejected、duplicate、deferred、requires-user-decision 或 non-goal 的合理 disposition。
- `PASS`、`NOT_CONVERGED` 与独立性状态符合剩余 finding。

独立 reviewer、定向复查和轮次上限需要读取执行 transcript；仅检查最终文档不足以证明流程执行。

### 成本

- token、耗时和 agent 数量。
- 用户干预次数。
- 与 baseline 相比的额外成本和人工复审减少量。

## 对比方式

正式比较包含：

1. Baseline A：单次独立 reviewer。
2. Baseline B：不加载 Skill，按原有人工多轮步骤和提示复现评审。
3. Treatment：`design-assurance-loop`。

每个正式案例每种配置运行三次。先在 historical regression 做冒烟和诊断；冻结 Skill 后再运行 held-out、control 和 metamorphic，不边跑边改。

## 泛化门禁

满足以下条件才可声称通用性获得初步验证：

- held-out P0 全部检出，且无 false P0。
- P1 recall 和 evidence accuracy 不低于 baseline。
- 最终无残留 P0，且未扩大 accepted-risk 或 non-goal。
- negative control 不产生 P0/P1。
- metamorphic pair 对同一机制给出一致判断。
- 流程合规项通过，成本和人工干预有记录。

未满足时，只能报告具体领域和案例上的结果，不得外推为通用改进。

## 实施顺序

1. 增加 catalog，并把现有三案标为 `historical-regression`。
2. 从无关项目历史中筛选 held-out 和 control 候选，先由用户确认 gold。
3. 增加一个 metamorphic pair。
4. 为全部案例补 finding 与流程合规 expectations。
5. 静态验证输入/gold 隔离后，再制定并运行对比测试。
6. 仅依据已观察到的 baseline/treatment 差异，最小修改 Skill 或 rubric。

## 验收标准

- 当前三个案例不再被描述为通用性证明。
- catalog 能区分开发集、held-out、control 和 metamorphic。
- held-out 生命周期明确，避免测试集参与调优。
- 每个新案例有真实输入、独立 gold、must-not-raise 和人工确认。
- 对比前不修改 Skill，不新增无实际需要的 harness 或目录。
