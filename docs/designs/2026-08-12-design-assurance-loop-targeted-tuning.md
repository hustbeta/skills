# Design Assurance Loop 定向调优设计

## 背景

iteration-2 对两个未见案例各执行了三次 Treatment 和三次 baseline，并完成人工裁决。Treatment 的微平均 assertion 通过率为 `22/36`，baseline 为 `18/36`。人工确认四项稳定缺口：共享实现的额外消费者没有保留用户裁决、缺少证据时误升 P0/P1、已证实的有界问题没有校准为 P2，以及没有从单执行单元上限推导并发累计成本。

本轮结果已被查看并将用于调优，因此不能继续作为 held-out 通用性证据。执行还受并发槽位约束，部分 Treatment 未完整实现三个独立 reviewer 的并行流程；既有分数仅描述已观察运行，不作为规范工作流的孤立因果证明。

用户确认采用方案 A：只修订能够解释稳定缺口的通用判定规则，不增加案例答案、项目术语、强制输出矩阵或新评测框架。

## 目标

- 用最小规则修订纠正四项人工确认的稳定缺口。
- 保持规则跨语言、跨框架、跨项目可用，不记忆具体案例表面词。
- 将已用于调优的案例转为历史回归，并保留原始输出、Gold 和人工裁决。
- 先用历史案例红绿验证，再冻结 Skill，以新的未见 held-out 和 negative control 检查迁移和误报。
- 给 Treatment 的三个独立 reviewer 保留并行容量，使下一轮结果符合 Skill 声明的执行方式。

## 非目标

- 不从两个案例推导领域专用的安全、路由、文件解析或容量规则。
- 不把具体函数名、产品名、固定字节数或案例公式写入 `SKILL.md` 或 rubric。
- 不增加自定义 benchmark harness、grader 框架、CI、依赖或新的 finding schema。
- 不因一次波动增加独立规则；单次 P0 漏判作为硬回归门禁，单次设计 PASS 与发布门禁混淆作为观察项。
- 不以总分提升代替严重缺陷召回、误报控制和范围边界门禁。

## Eval 生命周期调整

- eval 7 从 `transfer-heldout` 转为 `historical-regression`。
- eval 8 转为 `negative-control-regression`，保留其误报控制角色，但不再视为未见 control。
- 两案原始输入、Gold、运行输出和人工裁决保持不变，不用调优后结果覆盖旧结果。
- `evals.json` 的 prompt、files 和 expectations 在定向调优阶段保持不变；分类只修改 evaluator-only catalog。

## 通用规则修订

### 1. 影响发现不等于修改授权

对每个拟修改的共享实现，evidence index 应沿实际证据列出已识别的入口、调用方和消费者。只泛泛写“检查其它调用方”不算完成影响分析。

发现目标范围外的实际消费者时，必须记录其影响和 disposition。现有目标、兼容策略或业务语义不足以决定该消费者行为时，其具体处置标为 `requires-user-decision`；不得因为发现复用就自动扩大修改范围，也不得未经依据直接标为 accepted、deferred 或 non-goal。

### 2. 证据与严重度双向约束

finding 必须同时具备未被接受的契约或不变量违反、支持该违反的证据链，以及能够说明影响的失败场景。缺失运行态、外部实现或部署证据本身是 `unknown`，不能独立制造或升级 P0/P1。

如果静态设计与调用链证据已经证明核心契约被破坏，且某个运行态 unknown 不是失败链成立的必要条件，则该 unknown 只进入运行或发布门禁，不得单独降低已成立 finding 的严重度。如果该 unknown 是失败链成立的必要条件，则不得把违反描述为已证实；应降低结论置信度或保留为待验证项。

### 3. 限制作用域、控制顺序与累计暴露

评审容量和资源边界时，先标明限制作用于单项、单请求、单连接、单租户、单 worker、单进程、全局或时间窗口，再核对资源消耗发生在准入、限流和并发控制之前还是之后。

局部上限不能自动证明总体成本有界。存在多个并行执行单元时，应使用“活跃单元数量 × 单元成本”一类符号关系评估累计暴露，并同时考虑连接、调度、解析或其它持续资源。上游控制可以降低规模或改变资源消耗位置，应作为严重度缓解证据；它不自动改变已经由代码证明的应用处理顺序。

### 4. 已证实有界问题的 P2 校准

已经由证据证明的顺序、边界或局部契约弱点，即使没有系统性故障证据，也不能仅作为 unknown 丢弃。若失败影响有界、存在规避方式，或尚无高概率严重后果证据，可按实际影响定为 P2。

缺少严重后果证据只限制升级到 P0/P1；缺少任何契约违反或实际影响证据时，则不得为了填充 finding 自动定为 P2。

## 文件级设计

### `design-assurance-loop/SKILL.md`

- 在 evidence index 步骤加入共享实现消费者、限制作用域和关键控制顺序。
- 在父 agent 裁决步骤明确：范围外消费者缺少业务决定时使用 `requires-user-decision`，发现复用不授权扩大范围。
- 在质量门禁中要求每个已识别的实际消费者都有有证据的影响结论或明确待决策状态。

### `design-assurance-loop/references/review-rubric.md`

- 在严重度和证据规则中加入双向约束及“必要条件”例外。
- 在故障与安全检查面加入限制作用域、控制顺序和累计暴露检查。
- 明确有证据的有界弱点与无证据猜测之间的 P2 边界。

### `design-assurance-loop/evals/catalog.md`

- 只调整 eval 7、eval 8 的分类和状态说明。
- 保留具体 Gold；catalog 仍为 evaluator-only，不进入 executor 输入。

## 验证流程

### 阶段 1：静态检查

- `SKILL.md` frontmatter 的 `name` 和 `description` 仍有效。
- Skill 与 rubric 不出现两个调优案例的项目实体、函数名或固定容量答案。
- 修改保持在上述三个文件，不改变正式 eval 输入和 Gold。
- Markdown 无明显格式问题，`git diff --check` 通过。

### 阶段 2：定向历史红绿回归

eval 7、eval 8 各执行三次 Treatment。它们是开发集，允许根据失败继续最小修订通用规则；每次规则变化后，受影响案例重新执行，不覆盖先前输出。

硬门禁：

- eval 7 的既有 P0 三次全部命中，且不因无关 runtime unknown 降级。
- eval 7 的实际额外消费者三次全部被引用，具体处置三次全部为 `requires-user-decision`。
- eval 8 三次均无缺少输入证据的 P0/P1。
- eval 8 三次均识别已证实的顺序弱点并校准为 P2。
- eval 8 三次均区分单执行单元上限与并发累计成本。
- 两案均不得擅自扩大 accepted risk、non-goal 或额外消费者的修改范围。

设计 PASS 与 runtime 发布门禁的区分继续逐次记录，但本轮不因一次历史波动新增专门规则。

### 阶段 3：其它历史案例回归

eval 1 至 eval 6 先各执行一次 Treatment 冒烟。若任一案例出现既有 P0/P1 漏检、新增错误 P0/P1、错误扩大 accepted risk/non-goal 或关键 disposition 回归，则为该案例补足三次运行；至少两次失败视为确认回归并停止冻结。

一次冒烟通过只能表述为“未观察到回归”，不能表述为统计稳定。

### 阶段 4：冻结与全新盲测

定向和历史回归通过后记录 Skill 与 rubric 的 SHA-256，冻结本轮候选。冻结后再选择和构造一个新的 transfer-heldout 与一个新的 negative control；不能提前把其机制答案写入调优规则。

每个新案例分别执行三次 Treatment 和三次 baseline，共十二次。executor 只能看到 blind prompt、设计和证据；不得看到 catalog、expectations、decision ledger 或本设计文档。Gold 需在执行前由用户确认。

Treatment 与 baseline 分波执行。每次 Treatment 在独立新任务中由根 agent 直接执行，并为三个内部 reviewer 保留三个并行槽位；评测协调者不得作为占用同一四槽池的父 agent 嵌套启动 Treatment。baseline 使用另一 fresh context，且明确不加载 Skill。无法满足时保存原始结果并标记 `REDUCED_INDEPENDENCE`，但不计入规范并行流程的主要因果比较。

新盲测结果不得用于修改已冻结 Skill。若暴露新稳定缺口，案例转入下一轮 historical regression 后再启动新的调优周期。

## 通过标准

- 四项稳定缺口在定向回归中全部达到 `3/3`。
- eval 7 的 P0 召回达到 `3/3`。
- eval 8 的错误 P0/P1 为 `0/3`。
- 所有运行均未擅自扩大额外消费者、accepted risk 或 non-goal 的范围。
- eval 1 至 eval 6 没有观察到或确认新的严重回归。
- 新 held-out 的所有人工确认 P0/P1 均被三次 Treatment 命中。
- 新 negative control 的三次 Treatment 均为零 false P0/P1。
- 新案例的证据引用、disposition 和范围边界正确；总分差只作辅助指标。
- 只有新的未见案例通过后，才能报告本次修订获得新的迁移证据；仍不得由两个新案例外推为全面通用。

## 停止条件

- 定向历史门禁失败：继续最小修改通用规则，不进入冻结。
- 其它历史案例确认回归：撤回或缩小导致回归的规则，不进入冻结。
- 新盲测失败：保留冻结版本和原始结果，报告具体失败，不在同一轮继续调参。
- Gold 证据不足或存在业务争议：标记 `NEEDS_CONTEXT`，等待用户裁决，不运行对应案例。

## 用户参与点

当前只需确认本设计规格。实施期间不需要用户逐条操作；冻结后创建新案例时，用户需要按提供的证据链确认 Gold 的 finding、严重度、must-not-raise 和 PASS 边界。运行新盲测前，用户还需按届时给出的步骤创建或授权创建隔离任务，确保 Treatment 以根 agent 身份获得三个 reviewer 槽位。未确认 Gold 或不能满足隔离条件时，不运行正式新 eval。

## 执行中人工裁决补充（2026-08-12）

eval 8 的三轮定向输出都明确给出了与输入一一对应的“有界读取/解析 → 租户准入 → 应用并发配额”顺序并定为 P2；run 2、run 3 仅未复述具体函数名。用户裁决“语义顺序足够”。因此，具体符号链是充分但非必要证据；无歧义语义链同样命中，未映射到输入顺序的泛泛容量建议仍不命中。

该裁决只修订 evaluator 的证据匹配边界，不修改 Skill 或 rubric，不覆盖原始 grading、benchmark 和失败门禁。人工裁决只把 eval 8 assertion 2 的 run 2、run 3 从未命中改为命中，其它 assertion 与运行结果保持不变；该结果仍是历史开发回归，不构成通用性证明。
