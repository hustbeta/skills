# Design Assurance Loop 定向调优实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:executing-plans`, `skill-creator`, `superpowers:writing-skills`, `superpowers:test-driven-development`, and `karpathy-guidelines` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 最小修订 `design-assurance-loop` 的通用评审规则，使四项人工确认稳定缺口通过历史红绿回归，并在其它历史案例未确认回归后冻结候选 Skill。

**Architecture:** iteration-2 的人工裁决作为已观察 RED，不覆盖、不重跑旧产物。先把 eval 7/8 转为开发回归集，再只修改 `SKILL.md` 和 rubric 的通用规则；随后执行 eval 7/8 三轮定向 GREEN 和 eval 1–6 历史冒烟。全部门禁通过后记录候选哈希；新的 held-out/control 必须在冻结后另行设计和确认，不能在本计划中提前选择答案。

**Tech Stack:** Markdown、JSON、PowerShell、Python、skill-creator、Codex reviewer agents、Git。

## Global Constraints

- 规格来源：`docs/designs/2026-08-12-design-assurance-loop-targeted-tuning.md`；初始确认版 SHA-256 为 `004D0E7EE25DC3944BE7865C4123947B47AEEA93DDFF696C37CFB3E4F4996199`，执行中人工裁决补充版为 `96AAF2546F9C921113974C91174754248441AD530A44F52BCB88229BA8166A1F`。
- RED 证据来源：`design-assurance-loop-workspace/iteration-2/human-adjudication/benchmark-human-adjudicated.json`，SHA-256 为 `5B4CFC6EE0021497503B7F80C88D906C4BAC58EB32E09B15D6C49DD126DEF2D7`。
- 修改前 Skill SHA-256 为 `B548FEF3D9E7142D761E6CC0979E5F9D0DE8C09B2E4E7100B9FBF92DB131B7B0`；rubric 为 `D36064ABA6DC2A3E9C249BB1F89E3B9F6BF6CCD2DA3FBEF59B22CF7BF757928F`。
- `design-assurance-loop/evals/evals.json` 在原始定向评分完成前保持 SHA-256 `4B6DDF97360C341DDF5F29E584D825194CFF0E7B734EF496A5BC07D0F6EBACFB`；用户裁决后只允许修订 eval 8 assertion 2 的证据匹配措辞，新 SHA-256 为 `F6E0211E573C01E4D27972E85CF7D574557C762926E4944087AC63D2289DAD15`。prompt、files、expected output 和 finding 本身保持不变。
- 正式修改范围为 `design-assurance-loop/SKILL.md`、`design-assurance-loop/references/review-rubric.md`、`design-assurance-loop/evals/catalog.md`，以及人工裁决授权的 `design-assurance-loop/evals/evals.json`、eval 8 decision ledger 和本设计/计划。运行产物只写入已忽略的 `design-assurance-loop-workspace/iteration-3-tuning/`。
- 当前工作区已有同一任务链的未提交文件；保留全部既有修改，不清理、不暂存、不提交、不推送。当前 HEAD 为 `d1d1565e80ab7a708a9c0f7062efab68d5c80d73`。
- Skill/rubric 禁止写入案例函数名、项目实体、固定容量数值或案例公式；规则必须使用可观察的通用谓词。
- 定向回归属于开发测试，可以使用已知 Gold；它只能证明已知缺陷被覆盖，不能证明通用性。
- 每次 Treatment 由当前根 agent 直接执行；初始三位 reviewer 同时占用三个子代理槽位，完成后再启动独立 final reviewer。不得用外层执行子代理包裹 Treatment，否则四槽环境无法实现三 reviewer 并行。
- 任一计划外修改、输入哈希漂移、新的 Gold 争议或隔离失败立即停止；不得为了得到绿色结果修改 finding。eval 8 的“具体符号还是无歧义语义链”争议已由用户裁决，按下述执行补充处理。

### 执行补充：eval 8 证据匹配边界

- 原始六个 Treatment 输出、`grading.json`、`benchmark.json` 和 `gate-summary.md` 保持只读；旧正式 evaluator 文件归档到 `targeted/human-adjudication/pre-adjudication/`。
- 具体函数名是充分但非必要证据。输出若使用与输入一一对应且无歧义的“有界读取/解析 → 租户准入 → 应用并发配额”语义链并定为 P2，同样命中；泛泛容量建议不命中。
- 人工裁决只翻转 eval 8 assertion 2 的 run 2、run 3；其它评分不变。生成独立的 human-adjudicated benchmark 和门禁，不覆盖原始 34/36 结果。
- Skill/rubric 不因该裁决修改；人工裁决门禁通过后恢复 Task 6。

---

### Task 1: 固化并验证 RED 基线

**Files:**
- Verify: `design-assurance-loop-workspace/iteration-2/human-adjudication/benchmark-human-adjudicated.json`
- Verify: `design-assurance-loop-workspace/iteration-2/human-adjudication/benchmark-human-adjudicated.md`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/red-baseline.md`

**Interfaces:**
- Consumes: iteration-2 人工裁决和修改前哈希。
- Produces: 可审计的失败基线；后续 Skill 修改不得改写该文件引用的原始证据。

- [x] **Step 1: 确认新调优目录尚不存在**

Run:

```powershell
if (Test-Path -LiteralPath 'design-assurance-loop-workspace\iteration-3-tuning') {
    throw 'iteration-3-tuning already exists; inspect it and do not overwrite evidence'
}
'EXPECTED_RED iteration-3-tuning missing'
```

Expected: 输出 `EXPECTED_RED iteration-3-tuning missing`。若目录存在，停止并检查，不删除或覆盖。

- [x] **Step 2: 验证 RED 证据和修改前文件哈希**

Run:

```powershell
$expected = @{
    'design-assurance-loop-workspace\iteration-2\human-adjudication\benchmark-human-adjudicated.json' = '5B4CFC6EE0021497503B7F80C88D906C4BAC58EB32E09B15D6C49DD126DEF2D7'
    'design-assurance-loop-workspace\iteration-2\human-adjudication\benchmark-human-adjudicated.md' = '314EB0D170DD21CFE3F3C6CFEB037DC265A81EEBA0554151ABAFE7DEE902556E'
    'design-assurance-loop\SKILL.md' = 'B548FEF3D9E7142D761E6CC0979E5F9D0DE8C09B2E4E7100B9FBF92DB131B7B0'
    'design-assurance-loop\references\review-rubric.md' = 'D36064ABA6DC2A3E9C249BB1F89E3B9F6BF6CCD2DA3FBEF59B22CF7BF757928F'
    'design-assurance-loop\evals\evals.json' = '4B6DDF97360C341DDF5F29E584D825194CFF0E7B734EF496A5BC07D0F6EBACFB'
}
foreach ($path in $expected.Keys) {
    $actual = (Get-FileHash -Algorithm SHA256 -LiteralPath $path).Hash
    if ($actual -ne $expected[$path]) { throw "hash mismatch: $path" }
}
'RED_HASHES_OK'
```

Expected: `RED_HASHES_OK`。

- [x] **Step 3: 证明四项测试在修改前确实失败**

Run:

```powershell
$red = Get-Content -LiteralPath 'design-assurance-loop-workspace\iteration-2\human-adjudication\benchmark-human-adjudicated.json' -Raw | ConvertFrom-Json
$actual = @($red.confirmed_stable_gaps | ForEach-Object { "$($_.eval_id):$($_.assertion):$($_.with_skill_passed)/$($_.with_skill_total):$($_.human_gold_sufficient)" })
$expected = @('7:A-004:0/3:True', '8:B-001:1/3:True', '8:B-002:0/3:True', '8:B-003:0/3:True')
if (Compare-Object -ReferenceObject $expected -DifferenceObject $actual) { throw 'RED gap set changed' }
if ($red.summary.micro_assertion_rate.with_skill.passed -ne 22 -or $red.summary.micro_assertion_rate.with_skill.total -ne 36) { throw 'RED score changed' }
'EXPECTED_RED four stable gaps reproduced'
```

Expected: 输出 `EXPECTED_RED four stable gaps reproduced`。这一步是本次 Skill 行为修改的失败测试。

- [x] **Step 4: 创建最小 RED 记录**

使用 `apply_patch` 创建 `red-baseline.md`，记录上述五个哈希、四项失败计数、E7 P0 `2/3`、E8 设计 PASS `2/3`、执行保真限制，以及“该证据是 historical development RED，不是新 held-out”这一结论。

### Task 2: 将 eval 7/8 转为开发回归集

**Files:**
- Modify: `design-assurance-loop/evals/catalog.md:35`
- Modify: `design-assurance-loop/evals/catalog.md:36`
- Verify unchanged: `design-assurance-loop/evals/evals.json`

**Interfaces:**
- Consumes: Task 1 已确认 RED。
- Produces: 符合 held-out 生命周期规则的 evaluator-only 分类。

- [x] **Step 1: RED — 确认 catalog 仍把两案标为未见案例**

检查第 35–36 行。Expected: eval 7 仍为 `transfer-heldout`，eval 8 仍为 `negative-control`，两者状态含“用于调优即转 historical regression”。若已经转换，停止核对来源。

- [x] **Step 2: 只修改两行生命周期字段**

使用 `apply_patch`：

- eval 7 的评测层改为 `historical-regression（synthetic/metamorphic）`。
- eval 7 状态改为 `registered-historical-regression；tuning-source=iteration-2`。
- eval 8 的评测层改为 `negative-control-regression（synthetic）`。
- eval 8 状态改为 `registered-negative-control-regression；tuning-source=iteration-2`。
- 两行的领域、prompt、通用机制、Gold 和 Must-not-raise 原文保持不变。

- [x] **Step 3: 验证分类与正式 eval 未改变**

Run:

```powershell
$catalog = Get-Content -LiteralPath 'design-assurance-loop\evals\catalog.md' -Raw
foreach ($needle in @(
    '| 7 | historical-regression（synthetic/metamorphic） |',
    'registered-historical-regression；tuning-source=iteration-2',
    '| 8 | negative-control-regression（synthetic） |',
    'registered-negative-control-regression；tuning-source=iteration-2'
)) {
    if (-not $catalog.Contains($needle)) { throw "missing catalog transition: $needle" }
}
$evalHash = (Get-FileHash -Algorithm SHA256 -LiteralPath 'design-assurance-loop\evals\evals.json').Hash
if ($evalHash -ne '4B6DDF97360C341DDF5F29E584D825194CFF0E7B734EF496A5BC07D0F6EBACFB') { throw 'evals.json changed' }
'CATALOG_TRANSITION_OK'
```

Expected: `CATALOG_TRANSITION_OK`。

### Task 3: GREEN — 最小修订 Skill 与 rubric

**Files:**
- Modify: `design-assurance-loop/SKILL.md:26`
- Modify: `design-assurance-loop/SKILL.md:32`
- Modify: `design-assurance-loop/SKILL.md:43`
- Modify: `design-assurance-loop/references/review-rubric.md:21-38`
- Modify: `design-assurance-loop/references/review-rubric.md:48-52`
- Modify: `design-assurance-loop/references/review-rubric.md:60-64`

**Interfaces:**
- Consumes: 四项 RED 能力缺口。
- Produces: 不含案例答案的通用 evidence、severity、capacity 和 disposition 规则。

- [x] **Step 1: 修改 evidence index 的取证范围**

使用 `apply_patch` 将工作流第 3 步改为：

```markdown
3. 沿真实代码路径建立 evidence index，覆盖入口、调用方、共享实现的实际消费者、数据读写、旁路实现、异步副作用、错误处理、部署和回滚路径；涉及容量或资源边界时，标明限制作用域和关键控制顺序。未核实推断标记为 `unknown`。
```

- [x] **Step 2: 修改父 agent 的范围裁决规则**

将工作流第 6 步改为：

```markdown
6. 父 agent 去重并逐条裁决 finding，记录 `accepted`、`rejected`、`duplicate`、`requires-user-decision`、`deferred` 或 `non-goal` 及理由。发现目标范围外的实际消费者时记录其影响；现有目标、兼容策略或业务语义不足以决定其行为时，具体处置标记为 `requires-user-decision`，不因发现复用自动扩大修改范围。
```

- [x] **Step 3: 收紧影响范围质量门禁**

把质量门禁中的入口项改为：

```markdown
- 所有已识别入口、旁路、共享消费者和副作用均有基于证据的影响结论或明确待决策状态。
```

- [x] **Step 4: 在 rubric 加入证据与严重度双向约束**

保持现有 finding schema 和 P0–P3 定义，在严重级别及证据规则中加入以下规则：

```markdown
- finding 必须同时给出未被接受的契约或不变量违反、支持该违反的证据链和具体失败场景。
- 缺失运行态、外部实现或部署证据本身是 `unknown`，不能独立制造或升级 P0/P1。
- 若设计与调用链证据已经证明契约违反，且某个 `unknown` 不是失败链成立的必要条件，则该 unknown 只进入运行或发布门禁，不降低已成立 finding 的严重度。
- 若 unknown 是失败链成立的必要条件，不得把违反描述为已证实；降低 confidence，并保留为 `unknown` 待验证项或验证门禁。
- 已由证据证明的局部或有界弱点不得仅因不够 P0/P1 而改写为 unknown；符合 P2 定义时定为 P2。没有证据证明契约违反或实际影响时，不得为了填充 finding 定为 P2。
```

现有“证据不足时直接 `requires-user-decision`”一句改为：证据不足以证明契约违反时，将结论标为低置信度 `unknown` 或验证门禁，不得直接定 P0/P1；只有事实证据已足够、但缺少业务范围或兼容策略裁决时，才使用 `requires-user-decision`。

- [x] **Step 5: 在故障与安全检查面加入累计暴露推理**

在容量检查项后加入：

```markdown
- 标明每项限制作用于单项、单请求、单连接、单租户、单 worker、单进程、全局还是时间窗口，并核对资源消耗发生在准入、限流或并发控制之前还是之后。
- 局部上限不能自动证明总体成本有界；存在多个并行执行单元时，使用“活跃单元数量 × 单元成本”一类符号关系评估累计暴露，并考虑连接、调度、解析或其它持续资源。
- 上游控制是规模和成本位置的缓解证据，不自动改变已由代码证明的应用处理顺序。
```

- [x] **Step 6: 在裁决规则加入实际消费者 disposition**

在父 agent 裁决段加入：

```markdown
发现目标范围外的实际消费者时，必须记录具体证据和影响。现有目标、兼容策略或业务语义不足以决定其行为时，将具体处置标为 `requires-user-decision`；不得因发现复用自动扩大修改范围，也不得无依据改记为 accepted、deferred 或 non-goal。
```

### Task 4: 静态 GREEN、通用化审查与候选快照

**Files:**
- Verify: `design-assurance-loop/SKILL.md`
- Verify: `design-assurance-loop/references/review-rubric.md`
- Verify: `design-assurance-loop/evals/catalog.md`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/candidate-1/SKILL.md`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/candidate-1/references/review-rubric.md`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/candidate-1/manifest.json`

**Interfaces:**
- Consumes: Task 3 的最小规则修订。
- Produces: 静态有效、无答案泄漏的候选快照，供后续所有回归运行使用。

- [x] **Step 1: 验证 Skill frontmatter**

Run:

```powershell
python 'C:\Users\zhuxiaokun\.codex\skills\skill-creator\scripts\quick_validate.py' 'design-assurance-loop'
```

Expected: `Skill is valid!`。

- [x] **Step 2: 验证没有案例答案泄漏**

Run:

```powershell
$paths = @('design-assurance-loop\SKILL.md', 'design-assurance-loop\references\review-rubric.md')
$forbidden = @(
    'deliver_scheduled_export', 'resolve_export_task', 'handleBoundedManifest',
    'readManifestJSONBody', 'admitManifestTenant', 'acquireManifestConcurrencySlot',
    'task_id', 'tenant_id', '8192', 'N × 8192'
)
foreach ($path in $paths) {
    $text = Get-Content -LiteralPath $path -Raw
    foreach ($needle in $forbidden) {
        if ($text.Contains($needle)) { throw "case answer leaked into $path: $needle" }
    }
}
'GENERIC_RULES_OK'
```

Expected: `GENERIC_RULES_OK`。

- [x] **Step 3: 独立审查 diff**

启动一个 fresh-context 只读 reviewer，仅给已确认规格、`SKILL.md`、rubric 及两文件 diff。要求检查：四项能力是否都有规则承载、是否引入案例专用词、是否把 unknown 一概降级、是否把所有有界问题强制定为 P2、是否自动扩大共享消费者范围。存在 Critical/Important 时回到 Task 3，只修正对应文字并重新执行 Task 4；Minor 只记录，不扩大范围。

- [x] **Step 4: 创建候选快照与 manifest**

使用明确路径的 `New-Item` 创建目录，以 `Copy-Item` 机械复制 Skill/rubric。使用 `apply_patch` 创建 manifest，记录：候选编号 `candidate-1`、规格哈希、修改前和修改后 Skill/rubric 哈希、evals.json 哈希、catalog 生命周期、禁止读取 Gold、以及 `status: targeted-regression-pending`。

- [x] **Step 5: 验证格式和仓库差异**

Run:

```powershell
Get-Content -LiteralPath 'design-assurance-loop\evals\evals.json' -Raw | ConvertFrom-Json | Out-Null
$evalHash = (Get-FileHash -Algorithm SHA256 -LiteralPath 'design-assurance-loop\evals\evals.json').Hash
if ($evalHash -ne '4B6DDF97360C341DDF5F29E584D825194CFF0E7B734EF496A5BC07D0F6EBACFB') { throw 'evals.json changed' }
git diff --check
git diff -- design-assurance-loop/SKILL.md design-assurance-loop/references/review-rubric.md design-assurance-loop/evals/catalog.md
```

Expected: JSON 可解析；evals.json 哈希未变；`git diff --check` 为 0；diff 只包含已设计的通用规则与两行分类。

### Task 5: 对 eval 7/8 执行三轮定向 GREEN

**Files:**
- Create: `design-assurance-loop-workspace/iteration-3-tuning/targeted/eval-7/eval_metadata.json`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/targeted/eval-7/inputs/design.md`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/targeted/eval-7/inputs/evidence.md`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/targeted/eval-7/grader-only/decision-ledger.md`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/targeted/eval-7/with_skill/run-{1,2,3}/{outputs/review.md,execution.json,grading.json}`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/targeted/eval-8/eval_metadata.json`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/targeted/eval-8/inputs/design.md`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/targeted/eval-8/inputs/evidence.md`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/targeted/eval-8/grader-only/decision-ledger.md`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/targeted/eval-8/with_skill/run-{1,2,3}/{outputs/review.md,execution.json,grading.json}`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/targeted/benchmark.json`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/targeted/benchmark.md`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/targeted/gate-summary.md`

**Interfaces:**
- Consumes: candidate-1 快照、eval 7/8 原输入、正式 expectations 和 evaluator-only Gold。
- Produces: 每案三次 Treatment 的独立输出、评分和硬门禁结果。

- [x] **Step 1: 冻结 executor 可见输入**

每个 eval 目录只复制对应 `design.md`、`evidence.md` 和 candidate-1 Skill/rubric。`eval_metadata.json` 与 decision ledger 放在 grader-only 目录；executor 和 reviewer 消息不得引用 catalog、expectations、decision ledger、iteration-2 输出或本计划。

- [x] **Step 2: 顺序执行六个 Treatment run**

按 eval 7 run 1–3、eval 8 run 1–3 顺序执行。每个 run：

1. 根 agent 从 prompt、design/evidence 构造 canonical brief 和 evidence index。
2. 同时启动证据与契约、故障与安全、可执行性与验证三个 fresh-context reviewer；每个只读、互不可见。
3. 三者完成后，根 agent 裁决并修订设计。
4. 启动未参与前序评审的 fresh-context final reviewer，仅看原始约束、最终设计和证据。
5. 保存 `outputs/review.md` 和 `execution.json`；execution 记录三个初始 reviewer ID、final reviewer ID、是否真正并行及独立性状态。

如果三个初始 reviewer 未并行或上下文不独立，标记 `REDUCED_INDEPENDENCE`，该 run 不计入 `3/3`，保留原目录并用新 run number 补跑，不覆盖。

- [x] **Step 3: 独立评分**

每个 eval 启动一个 fresh-context grader，完整读取 `C:\Users\zhuxiaokun\.codex\skills\skill-creator\agents\grader.md`、该 eval 的 Gold、metadata 和三个有效输出。每个 `grading.json` 必须包含六条 assertion 的 `text`、`passed`、`evidence` 及自洽 summary；grader 不得修改输出或 Gold。

- [x] **Step 4: 聚合定向结果**

Run:

```powershell
python 'C:\Users\zhuxiaokun\.codex\skills\skill-creator\scripts\aggregate_benchmark.py' 'design-assurance-loop-workspace\iteration-3-tuning\targeted' --skill-name 'design-assurance-loop' --skill-path 'design-assurance-loop'
```

Expected: `with_skill` 下共有六个有效 run，每个 eval 三次；生成 benchmark JSON/Markdown。

- [x] **Step 5: 按命名能力生成硬门禁摘要**

使用 `apply_patch` 写 `gate-summary.md`，从 grading 逐 run 引用证据并统计：

- eval 7：P0、runtime unknown 不降级、实际额外消费者、`requires-user-decision`、范围不扩张，均须 `3/3`。
- eval 8：零 false P0/P1、P2 校准、并发累计成本、unknown 发布门禁、accepted-risk/non-goal 不扩张，均须 `3/3`。
- eval 8 的设计 PASS/发布门禁单独报告 `x/3`，作为观察项，不触发本轮新增规则。

Expected: 所有硬门禁 `3/3`；eval 8 错误 P0/P1 为 `0/3`。

- [x] **Step 6: GREEN 决策**

若全部硬门禁通过，candidate-1 进入 Task 6。任何硬门禁失败，都停止本计划剩余任务并保留全部结果。若失败来自 Gold 证据匹配边界争议，等待用户裁决；用户确认评分边界后生成独立人工裁决结果，原始结果不覆盖。只有 Skill 能力仍失败时才另写最小规则 delta，以 candidate-2 开始新的 RED/GREEN 循环。

### Task 6: 对 eval 1–6 执行历史回归冒烟

**Files:**
- Create: `design-assurance-loop-workspace/iteration-3-tuning/historical-smoke/eval-{1,2,3,4,5,6}/eval_metadata.json`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/historical-smoke/eval-{1,2,3,4,5,6}/inputs/{design.md,evidence.md}`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/historical-smoke/eval-{1,2,3,4,5,6}/grader-only/decision-ledger.md`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/historical-smoke/eval-{1,2,3,4,5,6}/with_skill/run-1/{outputs/review.md,execution.json,grading.json}`
- Create on failed smoke only: `design-assurance-loop-workspace/iteration-3-tuning/historical-smoke/eval-{failed-id}/with_skill/run-{2,3}/{outputs/review.md,execution.json,grading.json}`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/historical-smoke/benchmark.json`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/historical-smoke/benchmark.md`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/historical-smoke/regression-summary.md`

**Interfaces:**
- Consumes: 已通过定向门禁的同一 candidate-1 快照和正式 eval 1–6。
- Produces: 其它已知契约、严重度和范围边界的回归证据。

- [x] **Step 1: 每案执行一次 canonical Treatment**

对 eval 1–6 各执行一次，使用与 Task 5 相同的三 reviewer 并行、父 agent 裁决、独立 final reviewer 流程。每个 executor 只接收 candidate snapshot、prompt、design/evidence；Gold 与 expectations 保持 grader-only。

- [x] **Step 2: 独立评分六个冒烟输出**

fresh-context grader 按正式 expectations 评分并引用输出证据。除逐 assertion 外，额外标记四类回归：既有 P0/P1 漏检、错误新增 P0/P1、accepted-risk/non-goal 扩张、关键 disposition 错误。

- [x] **Step 3: 对出现失败的案例补足三次**

单次全通过的案例记录“未观察到回归”。任一 assertion 或四类回归标记失败时，仅为该 eval 再运行两次并独立评分。相同能力至少 `2/3` 失败时确认回归；只有 `1/3` 失败时记录波动，不宣称统计稳定。

- [x] **Step 4: 聚合并作停止判断**

Run:

```powershell
python 'C:\Users\zhuxiaokun\.codex\skills\skill-creator\scripts\aggregate_benchmark.py' 'design-assurance-loop-workspace\iteration-3-tuning\historical-smoke' --skill-name 'design-assurance-loop' --skill-path 'design-assurance-loop'
```

使用 `apply_patch` 写 `regression-summary.md`，列出每案运行数、assertion 结果、四类回归和证据位置。若确认任何回归，停止，不冻结 candidate；撤回或缩小规则需要新的经确认 delta。没有确认回归时进入 Task 7。

### Task 7: 冻结候选并完成调优结论

**Files:**
- Create: `design-assurance-loop-workspace/iteration-3-tuning/frozen-candidate/SKILL.md`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/frozen-candidate/references/review-rubric.md`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/frozen-candidate/manifest.json`
- Create: `design-assurance-loop-workspace/iteration-3-tuning/tuning-report.md`

**Interfaces:**
- Consumes: 定向 GREEN 和历史回归通过结果。
- Produces: 不再接受本轮新盲测反向调参的冻结候选，以及边界清楚的调优报告。

- [x] **Step 1: 复制最终候选并记录 SHA-256**

机械复制当前 Skill/rubric 到 `frozen-candidate/`。manifest 记录源路径、snapshot 路径、SHA-256、规格哈希、evals.json 哈希、targeted/historical benchmark 哈希，并设置：

```json
{
  "status": "frozen-pending-fresh-validation",
  "do_not_tune_on_fresh_validation": true,
  "generality_claim": false
}
```

- [x] **Step 2: 写调优报告**

报告必须包含：RED 基线、精确 diff、静态审查、eval 7/8 原始评分与人工裁决后的逐项 `3/3` 门禁、eval 1–6 冒烟及补跑情况、执行独立性、最终哈希、未提交状态和结论边界。不得把历史 GREEN 写成通用性证明。

- [x] **Step 3: 最终验证**

Run:

```powershell
python 'C:\Users\zhuxiaokun\.codex\skills\skill-creator\scripts\quick_validate.py' 'design-assurance-loop'
Get-Content -LiteralPath 'design-assurance-loop\evals\evals.json' -Raw | ConvertFrom-Json | Out-Null
$evalHash = (Get-FileHash -Algorithm SHA256 -LiteralPath 'design-assurance-loop\evals\evals.json').Hash
if ($evalHash -ne 'F6E0211E573C01E4D27972E85CF7D574557C762926E4944087AC63D2289DAD15') { throw 'evals.json changed outside the adjudicated boundary' }
git diff --check
git status --short --untracked-files=all
```

Expected: Skill valid；JSON 可解析且哈希符合人工裁决版；`git diff --check` 为 0；没有暂存、提交或推送；正式修改只来自计划范围，既有脏文件保持不变。

- [x] **Step 4: 交付冻结候选**

向用户报告：候选是否冻结、硬门禁和回归结果、最终文件链接，以及“通用性仍未验证”。若 Task 5/6 中止，报告失败证据和下一轮最小 delta，不声称完成。

## 冻结后的交接边界

本计划在 Task 7 停止。候选冻结后再启动一次独立 brainstorming：筛选一个新 transfer-heldout 和一个新 negative control，编写 `docs/designs/2026-08-12-design-assurance-loop-post-tuning-blind-validation.md`，经用户确认后再编写 `docs/plans/2026-08-12-design-assurance-loop-post-tuning-blind-validation.md`。

新 held-out 必须包含人工可证的 P0/P1 和至少一个范围边界；新 negative control 必须没有文件证据支持的 P0/P1，但允许 P2/P3、accepted risk 和发布门禁。两案不得复用 eval 7/8 的产品实体、函数、固定数值或直接同构答案。

用户届时需要逐案确认 finding、严重度、accepted risk/non-goal、must-not-raise、PASS 与发布门禁边界。判断标准是证据能否独立串起未被接受的契约违反和失败影响；缺证据时不注册。

新盲测的每次 Treatment 必须使用独立新任务，使其根 agent 能占用三个 reviewer 槽位。届时再向用户给出创建任务、粘贴 prompt、等待完成和返回任务 ID 的逐步操作。新结果不得反向修改 Task 7 的冻结候选；失败案例进入下一调优周期。
