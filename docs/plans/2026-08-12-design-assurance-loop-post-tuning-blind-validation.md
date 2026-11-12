# Design Assurance Loop 冻结后盲测实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:executing-plans`, `skill-creator`, `karpathy-guidelines`, and `superpowers:verification-before-completion` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 用两个全新、脱敏、自包含的合成案例，对冻结的 `candidate-1` 与调优前 Skill 快照执行 12 次独立评审、逐项评分和盲 A/B 比较，验证严重问题迁移能力与 negative-control 误报控制；本轮结果不得反向修改冻结候选。

**Architecture:** 正式 eval 附件与 evaluator-only Gold 先注册并冻结；每个 run 使用一个独立根任务和只含允许输入的 run package，根任务并行启动三个 fresh-context reviewer，再启动未参与前序评审的 final reviewer。原始尝试写入 `raw-runs/`，只有满足独立性和哈希门禁的三次有效结果才复制到 skill-creator 标准目录。评分、聚合、盲 A/B 和人工反馈均作为独立产物保存，不覆盖 executor 输出。

**Tech Stack:** Markdown、JSON、PowerShell、Python 标准库、skill-creator、Codex 独立任务与 reviewer agents、Git 只读检查。

## Global Constraints

- 规格来源：`docs/designs/2026-08-12-design-assurance-loop-post-tuning-blind-validation.md`，SHA-256 必须为 `122FD5552111B6231BD4D3605A8254E5A062523FD7726F6A79CF9ACDA06B4767`。
- Treatment 使用 `design-assurance-loop-workspace/iteration-3-tuning/frozen-candidate/`：Skill SHA-256 `973BB0231626B50146C850ABA43A4EABE49ECB039247228AF2A2FB78F51D245E`，rubric SHA-256 `26AC2C99A808F583FBF939CBF685DCDD2AEC8F455AE4C6F16FFB3DDF3CAB096B`。
- Baseline 使用 `design-assurance-loop-workspace/iteration-2/skill-under-test/`：Skill SHA-256 `B548FEF3D9E7142D761E6CC0979E5F9D0DE8C09B2E4E7100B9FBF92DB131B7B0`，rubric SHA-256 `D36064ABA6DC2A3E9C249BB1F89E3B9F6BF6CCD2DA3FBEF59B22CF7BF757928F`。
- 注册前 `evals.json` SHA-256 为 `F6E0211E573C01E4D27972E85CF7D574557C762926E4944087AC63D2289DAD15`，`catalog.md` SHA-256 为 `4965BD674DF390414A857FB5D943C0FE9BF4121EB3AD91D1F4152FBA6F07EC20`。Task 2–3 只允许追加 eval 9/10 及对应 catalog 行。
- `with_skill` 表示 frozen candidate；`without_skill` 仅为 skill-creator viewer 的兼容目录名，实际表示 pre-tuning Skill，不是无 Skill。所有报告必须显式写出该别名。
- 12 个 executor task 使用相同的请求模型、推理强度、blind prompt、input 和运行 contract；唯一有意差异是 Skill/rubric 快照。已知模型或推理强度不一致的 run 无效；实际配置不可观测时记录“requested same, actual unavailable”，不得宣称已证明完全同配。
- frozen candidate、baseline、规格、案例输入、Gold、assertions、run contract 和 12 个 prompt 在首个 run 前写入不可变 `frozen-input-manifest.json`；运行后不改。发现证据不足时标记 `NEEDS_CONTEXT` 并停止对应案例，不为输出修改答案。
- executor 只能读取 run package 内的 Skill、rubric、`design.md` 和 `evidence.md`。不得读取 catalog、`evals.json`、decision ledger、assertions、本规格、本计划、其它 run 或历史 benchmark。
- 当前仓库含同一任务链的未提交修改；保留全部既有修改，不清理、不暂存、不提交、不推送。所有运行产物写入已忽略的 `design-assurance-loop-workspace/iteration-4-blind-validation/`。
- 不新增 harness、grader 框架、CI、依赖或案例生成器；只复用 skill-creator 的 grader、comparator、聚合脚本和 viewer。
- telemetry 不可用时数字字段使用 `0` sentinel，并写 `availability: false` 与原因；禁止用输出字符数代理 token，禁止据此作效率结论。
- 任一计划外修改、哈希漂移、Gold 泄漏、配置串读、独立性失败或评分文本漂移均立即停止当前步骤，保存证据后再裁决。

---

### Task 1: 建立冻结后盲测边界

**Files:**

- Verify: `docs/designs/2026-08-12-design-assurance-loop-post-tuning-blind-validation.md`
- Verify: `design-assurance-loop-workspace/iteration-3-tuning/frozen-candidate/manifest.json`
- Verify: Treatment 与 baseline 的四个快照文件
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/preflight.md`

**Interfaces:**

- Consumes: 用户已确认规格、冻结候选、调优前快照和当前 evaluator 基线。
- Produces: 首个 run 前不可变边界；Task 2–12 必须引用该记录。

- [ ] **Step 1: 确认新 iteration 未被复用**

Run:

```powershell
if (Test-Path -LiteralPath 'design-assurance-loop-workspace\iteration-4-blind-validation') {
    throw 'iteration-4-blind-validation already exists; inspect it and do not overwrite evidence'
}
'EXPECTED_MISSING iteration-4-blind-validation'
```

Expected: `EXPECTED_MISSING iteration-4-blind-validation`。若目录存在，不删除；先判断是续跑还是路径冲突。

- [ ] **Step 2: 验证规格、快照和 evaluator 起点**

Run:

```powershell
$expected = @{
    'docs\designs\2026-08-12-design-assurance-loop-post-tuning-blind-validation.md' = '122FD5552111B6231BD4D3605A8254E5A062523FD7726F6A79CF9ACDA06B4767'
    'design-assurance-loop-workspace\iteration-3-tuning\frozen-candidate\SKILL.md' = '973BB0231626B50146C850ABA43A4EABE49ECB039247228AF2A2FB78F51D245E'
    'design-assurance-loop-workspace\iteration-3-tuning\frozen-candidate\references\review-rubric.md' = '26AC2C99A808F583FBF939CBF685DCDD2AEC8F455AE4C6F16FFB3DDF3CAB096B'
    'design-assurance-loop-workspace\iteration-2\skill-under-test\SKILL.md' = 'B548FEF3D9E7142D761E6CC0979E5F9D0DE8C09B2E4E7100B9FBF92DB131B7B0'
    'design-assurance-loop-workspace\iteration-2\skill-under-test\references\review-rubric.md' = 'D36064ABA6DC2A3E9C249BB1F89E3B9F6BF6CCD2DA3FBEF59B22CF7BF757928F'
    'design-assurance-loop\evals\evals.json' = 'F6E0211E573C01E4D27972E85CF7D574557C762926E4944087AC63D2289DAD15'
    'design-assurance-loop\evals\catalog.md' = '4965BD674DF390414A857FB5D943C0FE9BF4121EB3AD91D1F4152FBA6F07EC20'
}
foreach ($path in $expected.Keys) {
    $actual = (Get-FileHash -Algorithm SHA256 -LiteralPath $path).Hash
    if ($actual -ne $expected[$path]) { throw "hash mismatch: $path expected=$($expected[$path]) actual=$actual" }
}
'PREFLIGHT_HASHES_OK'
```

Expected: `PREFLIGHT_HASHES_OK`。

- [ ] **Step 3: 创建目录与 preflight 记录**

仅创建下列有实际用途的目录：

```text
design-assurance-loop-workspace/iteration-4-blind-validation/
├── benchmark-input/
├── comparisons/
├── human-review/
├── raw-runs/
├── run-packages/
└── run-prompts/
```

使用 `apply_patch` 创建 `preflight.md`，记录当前 HEAD、上述七个哈希、工作区未提交状态、`do_not_tune_on_fresh_validation: true`、`generality_claim: false` 和“不提交、不推送”。不得复制旧 benchmark 作为 executor 输入。

### Task 2: 注册 eval 9 — 滚动 Schema 演进

**Files:**

- Create: `design-assurance-loop/evals/files/database/rolling-schema-cutover/design.md`
- Create: `design-assurance-loop/evals/files/database/rolling-schema-cutover/evidence.md`
- Create: `design-assurance-loop/evals/files/database/rolling-schema-cutover/decision-ledger.md`
- Modify: `design-assurance-loop/evals/evals.json`
- Modify: `design-assurance-loop/evals/catalog.md`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/benchmark-input/eval-9-rolling-schema-cutover/eval_metadata.json`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/benchmark-input/eval-9-rolling-schema-cutover/inputs/design.md`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/benchmark-input/eval-9-rolling-schema-cutover/inputs/evidence.md`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/benchmark-input/eval-9-rolling-schema-cutover/grader-only/decision-ledger.md`

**Interfaces:**

- Consumes: 规格“案例 9”中已确认场景、证据、Gold、修复和边界。
- Produces: executor-visible 两个附件与 evaluator-only Gold；两者物理分离。

- [ ] **Step 1: 编写中性、脱敏的 executor 输入**

`design.md` 只写目标、非目标、现有数据模型、拟议阶段、回滚和验收方案，不使用 `P1`、`finding`、`错误池`、`旧 writer 缺口` 等答案性结论。必须包含：

- `execution_pool_name` 到 `execution_pool_id`；
- 零停机滚动发布、外部 API 不变、新调度器最终只读稳定 ID；
- 先新增 nullable ID、一次全量回填、切换新 reader、启用约束、回滚窗口后删旧列；
- 不建设通用 migration 平台。

`evidence.md` 使用稳定证据 ID `E9-01` 至 `E9-07`，中性记录：新旧版本共存；旧版只写名称；回填单次扫描与单一完成标志；没有 catch-up、版本比较、重复零漂移验证或旧 writer fence；新 reader 只读 ID 且 null 落默认池；存在无法唯一映射的别名；缺少真实生产运行和部署证据。

- [ ] **Step 2: 编写 evaluator-only decision ledger**

`decision-ledger.md` 必须按规格固定：

- P1 evidence chain：`一次回填 → 旧版继续只写名称 → ID 为空或陈旧 → 新 reader 只读 ID → 默认池或失败`；
- runtime unknown 不降低该设计 P1；
- accepted repair 顺序：expand、兼容 writer/read fallback、并发安全 migrate、重复 reconciliation 与稳定零漂移、旧 writer fence、cutover、validate、contract；
- 模糊别名处置为 `requires-user-decision`；
- accepted：零停机、API 不变；non-goal：通用迁移平台等；
- must-not-raise：真实流量、实例、表规模、批大小和部署结果缺失不得独立生成或升级 P0/P1，也不要求停机。

- [ ] **Step 3: 向 evals.json 追加正式条目**

追加 ID `9`；blind prompt 精确为：

```text
评审并修订附件中的执行池标识在线迁移设计。沿附件证据核对滚动发布兼容、数据回填、读写切换、约束启用、回滚和历史异常数据；无法验证的内容标记为 unknown。输出最终设计、验收覆盖矩阵、裁决账本和残余风险。
```

`eval_name` 为 `rolling-schema-cutover-heldout`；`expected_output` 精确为：

```text
识别滚动发布期间一次回填与旧 writer 共存造成的新旧表示分叉，把设计链路定为 P1，并用兼容写入、并发安全回填、稳定零漂移和旧 writer 门禁闭环；生产运行证据保持 unknown，模糊历史别名留给用户裁决。
```

`files` 只列 `design.md` 与 `evidence.md`。`expectations` 与 `eval_metadata.json.assertions` 必须逐字一致：

1. `把一次回填、旧 writer 继续写旧字段、新读路径只读 ID 和默认池/失败串成 P1`
2. `把生产实例、流量和部署结果标为 unknown，且不降低已成立 P1`
3. `要求兼容 writer 与旧 writer 清退/隔离门禁先于读切换`
4. `要求并发安全回填、重复 reconciliation 和稳定零漂移门禁，之后才验证约束和 contract`
5. `把无法唯一映射的历史别名处置标为 requires-user-decision，不发明默认映射`
6. `保持零停机、外部 API 和非目标边界，不强制通用迁移平台或停机发布`

- [ ] **Step 4: 注册 catalog 与 benchmark input**

catalog 新增 ID `9`：评测层 `transfer-heldout（synthetic）`，状态 `registered-transfer-heldout；frozen-cycle=iteration-4；do-not-tune`。Gold 置信度写 `user-confirmed`，不得使用具体产品名。

机械复制两个 executor-visible 文件到 `benchmark-input/.../inputs/`，机械复制 decision ledger 到 `grader-only/`。使用 `apply_patch` 创建 metadata，不从 metadata 再生成第二套 assertion 文本。

- [ ] **Step 5: 验证 JSON、隔离和内容边界**

Run:

```powershell
$evals = Get-Content -LiteralPath 'design-assurance-loop\evals\evals.json' -Raw | ConvertFrom-Json
$eval9 = @($evals.evals | Where-Object id -eq 9)
if ($eval9.Count -ne 1 -or $eval9[0].expectations.Count -ne 6) { throw 'eval 9 registration mismatch' }
if ($eval9[0].files.Count -ne 2 -or ($eval9[0].files -join '|').Contains('decision-ledger')) { throw 'eval 9 Gold leaked into files' }
$metadata = Get-Content -LiteralPath 'design-assurance-loop-workspace\iteration-4-blind-validation\benchmark-input\eval-9-rolling-schema-cutover\eval_metadata.json' -Raw | ConvertFrom-Json
if (Compare-Object -ReferenceObject @($eval9[0].expectations) -DifferenceObject @($metadata.assertions)) { throw 'eval 9 assertion drift' }
'EVAL_9_REGISTERED'
```

Expected: `EVAL_9_REGISTERED`。

### Task 3: 注册 eval 10 — 签名配置缓存 negative control

**Files:**

- Create: `design-assurance-loop/evals/files/configuration/signed-config-cache/design.md`
- Create: `design-assurance-loop/evals/files/configuration/signed-config-cache/evidence.md`
- Create: `design-assurance-loop/evals/files/configuration/signed-config-cache/decision-ledger.md`
- Modify: `design-assurance-loop/evals/evals.json`
- Modify: `design-assurance-loop/evals/catalog.md`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/benchmark-input/eval-10-signed-config-cache/eval_metadata.json`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/benchmark-input/eval-10-signed-config-cache/inputs/design.md`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/benchmark-input/eval-10-signed-config-cache/inputs/evidence.md`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/benchmark-input/eval-10-signed-config-cache/grader-only/decision-ledger.md`

**Interfaces:**

- Consumes: 规格“案例 10”中已确认控制链、accepted risk、P2、unknown 和 PASS 边界。
- Produces: 不提示 expected PASS 的 blind inputs 与 evaluator-only negative-control Gold。

- [ ] **Step 1: 编写中性、脱敏的 executor 输入**

`design.md` 必须完整描述配置轮询、签名验证、revision、原子替换、last-known-good、安全基线和只减权本地覆盖；不得写“设计 PASS”“只有 P2”或期望监控字段。

`evidence.md` 使用稳定 ID `E10-01` 至 `E10-08`，证明：固定公钥验签；revision 单调且拒绝同 revision 异内容；临时文件、同步、原子替换后才切内存；失败时只用已验证 LKG，最长 24 小时后关闭可选功能；5 秒读取且只能关闭的本地覆盖优先；业务接受 60 秒差异与 24 小时 LKG；当前只有拉取成功/失败监控；生产 key、时钟、文件权限、测试和部署未知。

- [ ] **Step 2: 编写 evaluator-only decision ledger**

固定记录：

- expected blocking findings `0`；没有输入证据支持的 P0/P1；
- allowed P2：缺少每实例 effective revision、bundle age、using-last-known-good、emergency-override-active；
- P2 原因是局部漂移发现和定位延迟，已有安全链使其不升 P1；
- accepted risk：60 秒传播差异、24 小时签名有效 LKG；
- unknown release gates：生产 key、时钟、覆盖文件权限、目标测试、部署；
- must-not-raise：逐请求控制面访问、全局强一致、共识、队列、数据库 migration、通用策略平台；不得把 LKG 写成无约束 fail-open；
- 若输入意外证明新的真实 P0/P1，保留 finding 并把案例标记 `NEEDS_CONTEXT`，不得压制证据维持 PASS。

- [ ] **Step 3: 向 evals.json 追加正式条目**

追加 ID `10`；blind prompt 精确为：

```text
评审并修订附件中的功能配置缓存设计。沿附件证据核对完整性、版本切换、控制面故障、紧急覆盖、可观测性、验证门禁和已接受风险；无法验证的内容标记为 unknown。输出最终设计、验收覆盖矩阵、裁决账本和残余风险。
```

`eval_name` 为 `signed-config-cache-negative-control`；`expected_output` 精确为：

```text
不制造缺少证据的 P0/P1，保留签名、单调 revision、原子替换、受限 last-known-good、安全基线和只减权覆盖；把缺少每实例生效状态定为 P2 可观测性缺口，并把生产配置、测试和部署保留为 unknown 发布门禁。
```

`files` 只列两个 executor-visible 文件。`expectations` 与 metadata assertions 必须逐字一致：

1. `没有输入证据支持的 P0/P1；严重 finding 必须证明违反未被接受的契约`
2. `指出缺少实例生效 revision、配置年龄、last-known-good 和紧急覆盖状态是 P2 可观测性缺口，不升级 P1`
3. `不重开已接受的 60 秒传播差异和 24 小时 last-known-good 窗口`
4. `保留签名、单调 revision、原子替换、过期安全基线和只减权紧急覆盖`
5. `把生产密钥、时钟、文件权限、目标测试和真实部署标为 unknown 发布门禁，不伪装成已验证或严重 finding`
6. `没有已接受 P0/P1 时允许设计 PASS，不要求全局强一致或通用平台`

- [ ] **Step 4: 注册 catalog 与 benchmark input**

catalog 新增 ID `10`：评测层 `negative-control（synthetic）`，状态 `registered-negative-control；frozen-cycle=iteration-4；do-not-tune`。机械复制输入与 Gold 到对应隔离目录。

- [ ] **Step 5: 验证两案正式注册**

Run:

```powershell
$evals = Get-Content -LiteralPath 'design-assurance-loop\evals\evals.json' -Raw | ConvertFrom-Json
foreach ($id in 9, 10) {
    $entry = @($evals.evals | Where-Object id -eq $id)
    if ($entry.Count -ne 1 -or $entry[0].expectations.Count -ne 6 -or $entry[0].files.Count -ne 2) { throw "eval $id registration mismatch" }
}
$metadata = Get-Content -LiteralPath 'design-assurance-loop-workspace\iteration-4-blind-validation\benchmark-input\eval-10-signed-config-cache\eval_metadata.json' -Raw | ConvertFrom-Json
$eval10 = @($evals.evals | Where-Object id -eq 10)[0]
if (Compare-Object -ReferenceObject @($eval10.expectations) -DifferenceObject @($metadata.assertions)) { throw 'eval 10 assertion drift' }
git diff --check
'EVALS_9_10_REGISTERED'
```

Expected: `EVALS_9_10_REGISTERED`；`git diff --check` 返回 0。

### Task 4: 冻结 run package、顺序和提示词

**Files:**

- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/manifest.json`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/frozen-input-manifest.json`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/run-order.json`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/run-prompts/executor-contract.md`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/run-prompts/seq-{01..12}.md`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/run-packages/seq-{01..12}/skill/SKILL.md`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/run-packages/seq-{01..12}/skill/references/review-rubric.md`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/run-packages/seq-{01..12}/inputs/design.md`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/run-packages/seq-{01..12}/inputs/evidence.md`

**Interfaces:**

- Consumes: 两案 frozen inputs、Treatment/baseline 快照和已确认执行拓扑。
- Produces: 12 个自包含、Gold-blind、顺序固定的根任务包。

- [ ] **Step 1: 固定平衡运行顺序**

`run-order.json` 精确映射：

| seq | eval | config | logical run |
| --- | --- | --- | --- |
| 01 | 9 | with_skill | 1 |
| 02 | 9 | without_skill | 1 |
| 03 | 9 | without_skill | 2 |
| 04 | 9 | with_skill | 2 |
| 05 | 9 | with_skill | 3 |
| 06 | 9 | without_skill | 3 |
| 07 | 10 | without_skill | 1 |
| 08 | 10 | with_skill | 1 |
| 09 | 10 | with_skill | 2 |
| 10 | 10 | without_skill | 2 |
| 11 | 10 | without_skill | 3 |
| 12 | 10 | with_skill | 3 |

每个条目记录 `status: pending`、`root_task_id: null`、`attempt: 1`、目标 raw output 目录和 package 路径。不得在运行中重排；补跑只增加 attempt，不覆盖原条目。

- [ ] **Step 2: 写无案例答案的 executor contract**

`executor-contract.md` 必须明确：

1. 只读本 seq package 的四个文件；不搜索或读取父目录、Gold、eval metadata、catalog、规格、计划或其它输出。
2. 完整读取 `SKILL.md` 与 rubric；先建立 canonical brief 和 evidence index。
3. 同时启动三个互不可见、fresh-context、只读 reviewer：证据与契约、故障与安全、可执行性与验证。
4. 根 agent 去重裁决；只修订 accepted P0/P1，保留 accepted/rejected/duplicate/requires-user-decision/deferred/non-goal 账本。
5. 再启动一个未参与前三审的 fresh-context final reviewer；final reviewer 只看 blind input、最终设计和裁决结果。
6. 写 `outputs/review.md` 与 `execution.json`；不得改 package、Skill 或 input。
7. `review.md` 不出现 `with_skill`、`without_skill`、`candidate-1`、`iteration-2`、`iteration-3` 或快照路径，避免破坏后续 A/B 盲法。
8. 无法真正并行三 reviewer、上下文不独立或 final reviewer 复用时，仍保存产物，但 `independence_status` 写 `REDUCED_INDEPENDENCE`。
9. 每个 seq prompt 列出 `INDEPENDENCE_OK` 与 `REDUCED_INDEPENDENCE` 两条完整候选完成行；按事实只输出其中一条，不得改变 eval、config 或 run。

- [ ] **Step 3: 固定 execution.json schema**

每个 run 写以下字段；未知元数据使用 JSON `null`，并在 `availability_notes` 解释，不发明值：

```json
{
  "schema_version": 1,
  "eval_id": 9,
  "configuration": "with_skill",
  "logical_run": 1,
  "attempt": 1,
  "root_task_id": null,
  "model": null,
  "reasoning_effort": null,
  "frozen_input_manifest_sha256": null,
  "allowed_reads": [],
  "forbidden_reads_observed": [],
  "initial_review_mode": "parallel",
  "initial_reviewers": [],
  "initial_contexts_independent": true,
  "final_reviewer": null,
  "final_reviewer_distinct": true,
  "final_review_result": null,
  "independence_status": "INDEPENDENCE_OK",
  "telemetry": {
    "available": false,
    "total_tokens": 0,
    "duration_ms": 0
  },
  "availability_notes": []
}
```

各 seq prompt 写死 eval、config、logical run、attempt、package 和 raw output 路径；其余文字只引用 executor contract，不含 Gold、assertions 或预期严重度。

- [ ] **Step 4: 机械构造 12 个 package**

Treatment seq 复制 frozen candidate；baseline seq 复制 iteration-2 pre-tuning snapshot。eval 9/10 各复制对应 benchmark input。每个 package 只含四个允许文件；不得放 metadata、manifest 或 prompt 副本。

- [ ] **Step 5: 计算并冻结 manifest**

`frozen-input-manifest.json` 记录：规格、frozen manifest、Treatment/baseline、注册后 evals/catalog、四个正式附件、两个 Gold、两个 metadata、executor contract、12 个 prompt、固定 run mapping 和每个 package 四文件的 SHA-256；同时记录 config alias、Gold 隔离为 `procedural-plus-package-boundary`、共享仓库仍可访问这一限制。该文件此后只读。

`manifest.json` 是可变的执行总账：记录 `frozen-input-manifest.json` 的 SHA-256、`status: pending-gold-confirmation`、run/grader/comparator/human-review provenance 和最终结论。不得把可变状态写回 frozen manifest。

Run:

```powershell
$root = 'design-assurance-loop-workspace\iteration-4-blind-validation'
$packages = Get-ChildItem -LiteralPath "$root\run-packages" -Directory
if ($packages.Count -ne 12) { throw "expected 12 packages, got $($packages.Count)" }
foreach ($package in $packages) {
    $files = Get-ChildItem -LiteralPath $package.FullName -File -Recurse
    if ($files.Count -ne 4) { throw "package must contain exactly four files: $($package.Name)" }
    if ($files.Name -contains 'decision-ledger.md' -or $files.Name -contains 'eval_metadata.json') { throw "Gold leaked into $($package.Name)" }
}
$frozen = Get-Content -LiteralPath "$root\frozen-input-manifest.json" -Raw | ConvertFrom-Json
$manifest = Get-Content -LiteralPath "$root\manifest.json" -Raw | ConvertFrom-Json
if ($frozen.status -ne 'frozen' -or -not $frozen.do_not_tune_on_fresh_validation) { throw 'frozen manifest flags invalid' }
if ($manifest.status -ne 'pending-gold-confirmation') { throw 'mutable manifest status invalid' }
if ($manifest.frozen_input_manifest_sha256 -ne (Get-FileHash -Algorithm SHA256 -LiteralPath "$root\frozen-input-manifest.json").Hash) { throw 'frozen manifest reference mismatch' }
'RUN_PACKAGES_FROZEN'
```

Expected: `RUN_PACKAGES_FROZEN`。

### Task 5: 人工确认 Gold 与首跑门禁

**Files:**

- Verify only: 两案 `design.md`、`evidence.md`、`decision-ledger.md`、metadata、两个 manifest 与 12 个 prompt
- Modify after approval: `design-assurance-loop-workspace/iteration-4-blind-validation/manifest.json`

**Interfaces:**

- Consumes: 已落盘的最终文本与哈希。
- Produces: 用户书面 `Gold 确认`；没有确认不得开始 seq 01。

- [ ] **Step 1: 向用户展示两案冻结摘要**

逐案展示：executor 能看到的事实、evaluator-only finding/PASS、六条 assertions、accepted/requires-user-decision/non-goal/must-not-raise、输入与 Gold 哈希。不得只展示设计摘要而隐藏实际文件差异。

- [ ] **Step 2: 用户按以下标准判断**

- eval 9：输入自身能串起一次回填、旧 writer、ID null/stale、新 reader 和默认池/失败；P1 不依赖真实流量；模糊别名没有足够业务信息自动处置。
- eval 10：输入证明安全配置链和已接受降级；只支持 P2 可观测性，不支持 P0/P1；unknown 是发布门禁而非设计 finding。
- 两案：blind prompt 未泄漏 expected severity/PASS；Skill/rubric 未出现案例专用答案；12 个 package 只含四个允许文件。

- [ ] **Step 3: 写确认记录**

用户回复 `Gold 确认` 后，用 `apply_patch` 在 manifest 写 `gold_confirmation.status: confirmed`、日期和本任务 ID（可得时）。任何修订都必须先重新计算全部受影响哈希，并再次请求确认；不得直接进入 run。

### Task 6: 执行 eval 9 的六个独立根任务

**Files:**

- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/raw-runs/eval-9/{with_skill,without_skill}/run-{1,2,3}/attempt-1/outputs/review.md`
- Create: 同级 `execution.json`
- Modify: `run-order.json`
- Create when required: 同一 logical run 的 `attempt-{2..n}`；不覆盖旧 attempt

**Interfaces:**

- Consumes: seq 01–06 prompt/package。
- Produces: 每配置三个独立有效 run，或明确的停止/补跑证据。

- [ ] **Step 1: 用户按 seq 01–06 逐次创建任务**

每次只执行一个 seq。协调任务向用户给出以下操作，不一次性丢出六个 prompt：

1. 在 Codex 点击“新建任务”。
2. workspace 选择 `D:\UserData\src\others\skills`。
3. 原样粘贴当前 `run-prompts/seq-XX.md` 全文；不要补充 Gold、上次结果或自己的判断。
4. 保持首跑前记录的模型和推理强度设置；不要在 12 个 run 中切换。
5. 等待任务输出该 prompt 列出的一个完整 `RUN_COMPLETE` 标记；不要同时启动下一 seq。
6. 返回该根任务 ID 和完成标记。

- [ ] **Step 2: 每次返回后立即验证**

协调任务检查：

- package 与 `frozen-input-manifest.json` 哈希未变；
- review/execution 两文件存在且 JSON 可解析；
- 三个 initial reviewer ID 非空、互异，`initial_review_mode=parallel`，contexts independent；
- final reviewer 非空且不在前三者中；
- `forbidden_reads_observed` 为空；
- 输出没有配置标识泄漏；
- root task ID、可用 model/reasoning/timestamps/telemetry 已补入 execution；不可得项明确为 null/0 sentinel。
- 请求模型与推理强度和冻结的统一配置一致；实际配置不可得时保留限制说明。

全部满足才把 seq 标记 `valid`。否则标记 `invalid-preserved`，保留 attempt，说明具体不合格项，并为同一配置/run 生成下一 attempt prompt；补跑仍须新根任务。

- [ ] **Step 3: eval 9 完成条件**

`with_skill` 和 `without_skill` 各有三个有效 logical run；无效 attempt 不计入 `3/3`。若连续三次因相同环境限制无法实现三 reviewer 并行，停止并报告执行环境 blocker，不放宽门禁。

### Task 7: 执行 eval 10 的六个独立根任务

**Files:**

- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/raw-runs/eval-10/{with_skill,without_skill}/run-{1,2,3}/attempt-1/outputs/review.md`
- Create: 同级 `execution.json`
- Modify: `run-order.json`
- Create when required: 新 attempt；不覆盖旧 attempt

**Interfaces:**

- Consumes: seq 07–12 prompt/package。
- Produces: 每配置三个独立有效 negative-control run。

- [ ] **Step 1: 按与 Task 6 相同的逐次用户操作执行 seq 07–12**

仍保持串行。不得把 eval 9 输出或 Gold 放入新任务。每次只在上一个 seq 通过独立性检查后给出下一个 prompt。

- [ ] **Step 2: 对每个 attempt 执行相同门禁**

除通用独立性外，额外检查 executor 没有在 review 中自称“预期 PASS”“negative control”或引用 assertion 文本；这些会破坏盲法，必须补跑。

- [ ] **Step 3: 形成 12-run 有效集合**

完成条件是 eval 9/10 × Treatment/baseline × 3 = 12 个有效 run。此时冻结 `run-order.json` 和全部 raw attempts；后续只允许机械复制、评分和派生分析。

### Task 8: 选择有效 run 并独立评分

**Files:**

- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/selection-manifest.json`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/benchmark-input/eval-{9,10}-*/{with_skill,without_skill}/run-{1,2,3}/outputs/review.md`
- Create: 同级 `execution.json`、`timing.json`、`grading.json`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/grading-execution.json`

**Interfaces:**

- Consumes: 12 个有效 raw runs、固定 assertions 与 evaluator-only Gold。
- Produces: skill-creator 标准目录和逐 run 独立评分。

- [ ] **Step 1: 机械选择，不重写输出**

`selection-manifest.json` 把每个标准 run 映射到一个 raw attempt 及其 SHA-256。复制 `review.md` 和 `execution.json`；logical run 保持 1–3。无效 attempt 不复制。

每个 `timing.json` 使用 skill-creator schema。若 task 通知提供 token/duration，立即记录原值；否则所有数字写 `0`，并增加 `availability: false` 和原因。不得从 `output_chars` 推断 token。

- [ ] **Step 2: 为两案分别启动 fresh-context grader**

两个 grader 可并行；每个完整读取：

- `C:\Users\zhuxiaokun\.codex\skills\skill-creator\agents\grader.md`；
- 本案 `eval_metadata.json`、grader-only decision ledger；
- 六个 selected outputs、execution 和 timing；
- 不读取另一个案例、Skill 调优报告、A/B mapping 或其它 grader 输出。

grader 只写六个 `grading.json`。每条必须包含原样 `text`、布尔 `passed`、具体 `evidence`；summary 必须与六项计数一致。语义等价证据足够，不要求命中特定术语，但泛泛建议不算命中。

- [ ] **Step 3: 记录 grader provenance**

`grading-execution.json` 记录每案 grader task ID、模型/推理强度（可得时）、允许读取路径、fresh-context 请求、写入范围、开始/结束时间、完整 transcript 是否可得。不可得不伪装为已验证独立性。

- [ ] **Step 4: 验证 12 个评分**

Run:

```powershell
$root = 'design-assurance-loop-workspace\iteration-4-blind-validation\benchmark-input'
$grades = Get-ChildItem -LiteralPath $root -Filter 'grading.json' -File -Recurse
if ($grades.Count -ne 12) { throw "expected 12 grading files, got $($grades.Count)" }
foreach ($grade in $grades) {
    $data = Get-Content -LiteralPath $grade.FullName -Raw | ConvertFrom-Json
    if ($data.expectations.Count -ne 6) { throw "expected six assertions: $($grade.FullName)" }
    $passed = @($data.expectations | Where-Object passed -eq $true).Count
    if ($data.summary.passed -ne $passed -or $data.summary.total -ne 6 -or $data.summary.failed -ne (6 - $passed)) { throw "grading summary mismatch: $($grade.FullName)" }
}
'TWELVE_GRADES_VALID'
```

Expected: `TWELVE_GRADES_VALID`。

### Task 9: 聚合并执行命名硬门禁

**Files:**

- Create by skill-creator: `design-assurance-loop-workspace/iteration-4-blind-validation/benchmark-input/benchmark.json`
- Create by skill-creator: `design-assurance-loop-workspace/iteration-4-blind-validation/benchmark-input/benchmark.md`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/benchmark-context.json`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/gate-summary.md`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/analyst-notes.json`

**Interfaces:**

- Consumes: 12 个正式 grading/timing。
- Produces: 原始聚合、配置解释、逐 assertion `x/3` 与冻结边界内结论。

- [ ] **Step 1: 运行官方聚合脚本**

Run:

```powershell
$env:PYTHONUTF8 = '1'
python 'C:\Users\zhuxiaokun\.codex\skills\skill-creator\scripts\aggregate_benchmark.py' 'design-assurance-loop-workspace\iteration-4-blind-validation\benchmark-input' --skill-name 'design-assurance-loop' --skill-path 'design-assurance-loop-workspace\iteration-3-tuning\frozen-candidate'
```

Expected: 12 runs；eval 9/10 各有 `with_skill` 与 `without_skill` 三次；viewer schema 使用精确配置名。

- [ ] **Step 2: 写配置与 telemetry 解释**

`benchmark-context.json` 明确：

- `with_skill = frozen candidate-1`；
- `without_skill = pre-tuning Skill snapshot`；
- `without_skill` 不是 no-skill baseline；
- token/time 零值是 unavailable sentinel；
- shared-repository Gold blindness 是 package + instruction 隔离，不是 OS sandbox 证明。

若全部 timing 都不可用，`gate-summary.md` 不出现效率优劣结论。

- [ ] **Step 3: 计算硬门禁**

`gate-summary.md` 逐 run 引证 grading，不只抄总分。

eval 9 Treatment 必须：P1 链 `3/3`；runtime unknown 不降级 `3/3`；兼容 writer + 旧 writer fence `3/3`；并发安全回填 + 稳定零漂移 `3/3`；模糊别名 `requires-user-decision` `3/3`；零停机/API/non-goal `3/3`。

eval 10 Treatment 必须：错误 P0/P1 `0/3`；P2 可观测性 `3/3`；accepted risk 不重开 `3/3`；安全配置链 `3/3`；unknown 分离 `3/3`；设计 PASS 且无平台化扩张 `3/3`。

同时列 baseline 对应 `x/3`，但 baseline 不受 Treatment 硬门禁替代。总分不能覆盖命名门禁。

- [ ] **Step 4: fresh-context analyst 只做模式归纳**

analyst 读取 aggregate、gate summary 和逐项 grades；不得读取 Skill 源码或提出 candidate-1 修改。`analyst-notes.json` 只回答：Treatment/baseline 哪些命名能力不同、差异是否一致、telemetry/独立性有哪些限制、结论属于哪一种：

- Treatment pass / baseline fail：本轮 delta 的正向迁移证据；
- both pass：候选的迁移证据，但无因果 delta 证明；
- Treatment fail：本轮停止，不调参；
- Treatment worse：记录潜在回归，下一周期处理。

### Task 10: 执行每案三对盲 A/B

**Files:**

- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/comparisons/eval-{9,10}/mapping.json`
- Create: `.../blind-inputs/pair-{1,2,3}/{A.md,B.md}`
- Create: `.../comparison-run-{1,2,3}.json`
- Create: `.../comparison-summary.json`
- Create: `.../unblinded-summary.md`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/comparison-execution.json`

**Interfaces:**

- Consumes: 同 logical run 的 Treatment/baseline review，不含配置元数据。
- Produces: 两个 fresh-context comparator 的六次 pair 判断和可审计 unblind。

- [ ] **Step 1: 在输出完成后随机 A/B mapping**

对六对分别生成一个随机 bit；先把结果写入 evaluator-only `mapping.json`，再机械复制 review 到 A/B。mapping 不传给 comparator。若 review 含 `with_skill`、`without_skill`、`candidate-1` 或 iteration 快照标识，停止该 pair 并补跑对应 executor，不做文本清洗冒充盲法。

Run:

```powershell
1..6 | ForEach-Object {
    $bit = [System.Security.Cryptography.RandomNumberGenerator]::GetInt32(0, 2)
    "pair-$($_)=$bit"
}
```

Expected: 六行 `pair-N=0|1`。立即用 `apply_patch` 把结果写入 mapping；`0` 固定表示 Treatment 放 A，`1` 固定表示 Treatment 放 B。

- [ ] **Step 2: 每案启动一个 fresh-context comparator**

comparator 完整读取 `C:\Users\zhuxiaokun\.codex\skills\skill-creator\agents\comparator.md`、本案 blind prompt、design/evidence 和三对 A/B。不得读取 Gold、assertions、mapping、execution、grading、benchmark 或另一案例。

每对独立写 skill-creator comparison schema，判断证据准确性、严重度、闭环、unknown/scope 和总体 `A`/`B`/`TIE`。允许 tie；禁止根据篇幅、格式或术语猜配置。

- [ ] **Step 3: 记录 provenance 并 unblind**

`comparison-execution.json` 记录两个 comparator task ID、fresh-context 与读取边界。协调任务按 mapping 生成 `unblinded-summary.md`，逐对标明 Treatment/Baseline/TIE。baseline 的实质性正确性胜出必须单列；风格性胜出不自动判 candidate 失败。

### Task 11: 生成 review viewer 并完成人工裁决

**Files:**

- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/human-review/viewer-workspace/` presentation copies
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/human-review/review.html`
- Create by viewer: `design-assurance-loop-workspace/iteration-4-blind-validation/human-review/viewer-workspace/feedback.json`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/human-review/adjudication.md`

**Interfaces:**

- Consumes: 12 原始输出、grades、aggregate、盲比较和 unblinded mapping。
- Produces: 用户逐 run 反馈与不覆盖原始评分的人工裁决。

- [ ] **Step 1: 建 presentation-only viewer workspace**

为 12 个 selected run 机械复制 `review.md` 与 `grading.json`；在每个 `outputs/` 额外放对应 `blind-comparison.json` 和 `unblinded-comparison.md`。把该案 `eval_metadata.json` 复制到 `with_skill/` 和 `without_skill/` 配置目录，使 viewer 能显示 prompt 和 eval ID。presentation copy 不作为 canonical executor 产物；所有文件保留来源 SHA-256 清单。

- [ ] **Step 2: 生成静态 artifact**

Run:

```powershell
python 'C:\Users\zhuxiaokun\.codex\skills\skill-creator\eval-viewer\generate_review.py' 'design-assurance-loop-workspace\iteration-4-blind-validation\human-review\viewer-workspace' --skill-name 'design-assurance-loop frozen vs pre-tuning' --benchmark 'design-assurance-loop-workspace\iteration-4-blind-validation\benchmark-input\benchmark.json' --static 'design-assurance-loop-workspace\iteration-4-blind-validation\human-review\review.html'
```

Expected: 生成自包含 `review.html`，包含 12 个 run、grades 与 benchmark。

- [ ] **Step 3: 用 viewer 自带服务收集反馈**

不要用 `python -m http.server`；那会令 `/api/feedback` 404，且某些浏览器上下文会出现 `blocked:origin`。Run:

```powershell
python 'C:\Users\zhuxiaokun\.codex\skills\skill-creator\eval-viewer\generate_review.py' 'design-assurance-loop-workspace\iteration-4-blind-validation\human-review\viewer-workspace' --skill-name 'design-assurance-loop frozen vs pre-tuning' --benchmark 'design-assurance-loop-workspace\iteration-4-blind-validation\benchmark-input\benchmark.json' --port 3117
```

用户打开脚本输出的 URL；通常是 `http://localhost:3117`，端口被占用时以脚本打印的新端口为准。逐项填写并点击完成。判断标准：

- Gold 是否仍由 blind input 充分支持；
- grader 是否接受无歧义语义证据而非强制产品词；
- 是否有 assertions 未覆盖的严重漏检或误报；
- Treatment/baseline 差异是否有实际评审价值，而非篇幅或风格差异；
- comparator 的盲判断是否合理。

Expected: `viewer-workspace/feedback.json` 的 `status` 为 `complete`，含 12 个 run；空 feedback 表示已看且无补充，不表示未审。

- [ ] **Step 4: 保存人工裁决，不覆盖 raw grade**

`adjudication.md` 逐案记录 `Gold sufficient: yes/no`、评分修订、未覆盖 finding、A/B 价值判断和用户原话摘要。若人工裁决翻转自动评分，另建 human-adjudicated aggregate；原 `grading.json` 和 `benchmark.json` 保持只读。

### Task 12: 结论、生命周期转换与最终验证

**Files:**

- Modify after completed review: `design-assurance-loop/evals/catalog.md`
- Create: `design-assurance-loop-workspace/iteration-4-blind-validation/final-report.md`
- Modify: `design-assurance-loop-workspace/iteration-4-blind-validation/manifest.json`
- Verify unchanged: `frozen-input-manifest.json`、frozen candidate 与 baseline 四个文件

**Interfaces:**

- Consumes: 硬门禁、baseline 对照、盲 A/B、人工反馈和执行限制。
- Produces: 边界清楚、可复现且不反向调参的最终报告。

- [ ] **Step 1: 作冻结周期结论**

按以下规则写 `final-report.md`：

- Treatment 通过、baseline 失败同一能力：正向迁移证据支持本轮通用规则 delta；
- 两者都通过：candidate 在该案例获得迁移证据，但不能证明 delta 因果；
- Treatment 任一硬门禁失败：保留 frozen/raw，结论为失败或未通过；不得改 candidate-1；
- Treatment 比 baseline 差：记录潜在回归，下一调优周期再创建 candidate-2；
- Gold 不充分：对应案例 `NEEDS_CONTEXT`，不计入通用性结论；
- 两案均通过时，最高表述仅为“冻结候选在两个全新合成案例中获得有限迁移证据”。

- [ ] **Step 2: 转换案例生命周期**

因为输出已被观察，eval 9 从 `transfer-heldout` 改为 `historical-regression（synthetic）`，eval 10 改为 `negative-control-regression（synthetic）`；状态记录 `observed-in=iteration-4`。只改 catalog 生命周期，不改 input、Gold、assertions 或 Skill。

- [ ] **Step 3: 更新 manifest 状态**

manifest 追加所有最终 artifact 的 SHA-256、有效/无效 attempt、grader/comparator/human-review provenance、telemetry availability 和结论。状态按事实写 `completed-pass`、`completed-fail` 或 `needs-context`；`do_not_tune_on_fresh_validation` 与 `generality_claim: false` 保持不变。

- [ ] **Step 4: 最终验证**

Run:

```powershell
$expected = @{
    'design-assurance-loop-workspace\iteration-3-tuning\frozen-candidate\SKILL.md' = '973BB0231626B50146C850ABA43A4EABE49ECB039247228AF2A2FB78F51D245E'
    'design-assurance-loop-workspace\iteration-3-tuning\frozen-candidate\references\review-rubric.md' = '26AC2C99A808F583FBF939CBF685DCDD2AEC8F455AE4C6F16FFB3DDF3CAB096B'
    'design-assurance-loop-workspace\iteration-2\skill-under-test\SKILL.md' = 'B548FEF3D9E7142D761E6CC0979E5F9D0DE8C09B2E4E7100B9FBF92DB131B7B0'
    'design-assurance-loop-workspace\iteration-2\skill-under-test\references\review-rubric.md' = 'D36064ABA6DC2A3E9C249BB1F89E3B9F6BF6CCD2DA3FBEF59B22CF7BF757928F'
}
foreach ($path in $expected.Keys) {
    if ((Get-FileHash -Algorithm SHA256 -LiteralPath $path).Hash -ne $expected[$path]) { throw "frozen file changed: $path" }
}
Get-Content -LiteralPath 'design-assurance-loop\evals\evals.json' -Raw | ConvertFrom-Json | Out-Null
$root = 'design-assurance-loop-workspace\iteration-4-blind-validation'
$manifest = Get-Content -LiteralPath "$root\manifest.json" -Raw | ConvertFrom-Json
Get-Content -LiteralPath "$root\frozen-input-manifest.json" -Raw | ConvertFrom-Json | Out-Null
if ($manifest.frozen_input_manifest_sha256 -ne (Get-FileHash -Algorithm SHA256 -LiteralPath "$root\frozen-input-manifest.json").Hash) { throw 'frozen input manifest changed' }
Get-Content -LiteralPath 'design-assurance-loop-workspace\iteration-4-blind-validation\benchmark-input\benchmark.json' -Raw | ConvertFrom-Json | Out-Null
git diff --check
git status --short --untracked-files=all
```

Expected: 四个快照哈希不变；JSON 可解析；`git diff --check` 为 0；没有暂存、提交或推送；正式仓库修改仅为 eval 9/10 注册、最终 catalog 生命周期和本规格/计划，既有任务链改动保持不变。

- [ ] **Step 5: 交付**

向用户报告：两案逐 assertion Treatment/baseline `x/3`、硬门禁、盲 A/B、人工裁决、独立性/telemetry 限制、最终 manifest/report 链接，以及下一周期是否需要 candidate-2。未通过时只给下一周期建议，不在本计划内调优。

## 执行交接

推荐在当前任务内使用 `superpowers:executing-plans` 顺序执行 Task 1–5；Task 6–7 由用户按协调任务逐次创建 12 个独立根任务；Task 8–12 再回到当前协调任务。这样满足三 reviewer 并行的四槽约束，也避免外层 executor 占用一个槽位。

开始正式运行前必须再次停在 Task 5，等待用户回复 `Gold 确认`。计划确认不等于 Gold 文件确认。
