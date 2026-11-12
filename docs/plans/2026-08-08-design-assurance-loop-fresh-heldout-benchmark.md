# Design Assurance Loop 全新跨域盲测实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` and `skill-creator` to execute this plan task-by-task.

**Goal:** 将已确认的跨租户导出正向案例和有界清单 negative control 制作为隔离盲测，并对冻结的当前 Skill 与 no-Skill baseline 各运行三次，得到经人工裁决的 iteration-2 对比结果。

**Architecture:** 先在被忽略的 prep 目录生成两案输入与 evaluator-only Gold，经过用户确认后才注册 eval #7/#8。正式 benchmark 使用新的 iteration-2 快照；executor 只接收 blind prompt、design/evidence，Treatment 额外接收冻结 Skill/rubric。六组 baseline/treatment pair 由 fresh-context 子代理执行，独立 grader 评分，最后保留原始结果并生成静态人工评审页。

**Tech Stack:** Markdown、JSON、PowerShell、Python、skill-creator、Codex subagents、Git。

## Global Constraints

- 规格来源：`docs/designs/2026-08-08-design-assurance-loop-fresh-heldout-benchmark.md`，已获用户确认。
- 不修改 `design-assurance-loop/SKILL.md` 或 `references/review-rubric.md`，不依据运行中结果调参。
- 不覆盖或改写 `design-assurance-loop-workspace/iteration-1/` 的任何原始、评分或人工裁决产物。
- 两案 Gold 经用户确认前，只能存在于被 Git 忽略的 `design-assurance-loop-workspace/iteration-2-prep/`；不得写入正式 `evals.json`。
- 每个 eval、每种 configuration 运行三次；configuration 名固定为 `with_skill`、`without_skill`，总计 12 个 executor run。
- baseline 不得读取 Skill/rubric；Treatment 必须先完整读取冻结 Skill/rubric。两者均不得读取 catalog、`evals.json`、expectations、decision ledger、grader metadata 或本计划/规格。
- 每次运行使用 fresh context。受四个并发槽限制，每波只同时启动一组 baseline/treatment；共六波。
- 协作工具没有可靠 token telemetry 时，记录 `total_tokens: 0` 和 `token_telemetry: unavailable`，不得用字符数冒充 token。
- 当前工作区已有同一任务链的未提交文件；仅追加本计划内文件，不清理、不覆盖、不提交、不推送。
- 受保护文件当前 SHA-256：
  - `SKILL.md`: `B548FEF3D9E7142D761E6CC0979E5F9D0DE8C09B2E4E7100B9FBF92DB131B7B0`
  - `references/review-rubric.md`: `D36064ABA6DC2A3E9C249BB1F89E3B9F6BF6CCD2DA3FBEF59B22CF7BF757928F`
  - 已确认规格：`405A15064EFE55B42DDCF84A83E637B7638C107480554E4D54E7933E49B1007A`

---

### Task 1: 生成案例 A 候选输入与 Gold

**Files:**
- Create: `design-assurance-loop-workspace/iteration-2-prep/cross-tenant-export-download/design.md`
- Create: `design-assurance-loop-workspace/iteration-2-prep/cross-tenant-export-download/evidence.md`
- Create: `design-assurance-loop-workspace/iteration-2-prep/cross-tenant-export-download/decision-ledger.md`

**Interfaces:**
- Consumes: 已确认规格中的案例 A 边界。
- Produces: 跨租户导出 blind input 和 evaluator-only Gold，不注册正式 eval。

- [ ] **Step 1: RED — 确认 prep 目录不存在**

Run:

```powershell
if (Test-Path -LiteralPath 'design-assurance-loop-workspace\iteration-2-prep') { throw 'iteration-2-prep exists; inspect before continuing' } else { 'EXPECTED_RED prep missing' }
```

- [ ] **Step 2: 写最小 blind design**

设计提出：已认证用户按导出任务 ID 下载；复用现有全局任务解析函数；保持响应与存储结构；需求范围只声明交互式下载。不得写出“跨租户漏洞”“P0”“共享消费者”或预期修复答案。

- [ ] **Step 3: 写脱敏 evidence**

证据必须能串起：请求中的用户可控任务 ID、会话中的租户身份、只按全局 ID 查询、从查询结果返回文件、定时投递也调用同一解析函数。证据还应明确没有部署和网络可达性材料，但不解释其严重度含义。

- [ ] **Step 4: 写 evaluator-only Gold**

Gold 定义：

- `A-001`：P0；已认证用户可选择其它租户任务 ID 并取得结果文件，破坏租户隔离。
- 部署/网络可达性为运行态 unknown，不降低设计 P0。
- `A-002`：共享解析函数还有定时投递消费者；具体处置为 `requires-user-decision`。
- 不要求改响应 JSON、全局权限框架、统一身份模型、额外加密/审计平台或数据库迁移。
- 只泛泛建议检查调用方而未引用定时消费者，不算命中 `A-002`。

### Task 2: 生成案例 B 候选输入与 Gold

**Files:**
- Create: `design-assurance-loop-workspace/iteration-2-prep/bounded-manifest-admission/design.md`
- Create: `design-assurance-loop-workspace/iteration-2-prep/bounded-manifest-admission/evidence.md`
- Create: `design-assurance-loop-workspace/iteration-2-prep/bounded-manifest-admission/decision-ledger.md`

**Interfaces:**
- Consumes: 已确认规格中的案例 B 边界。
- Produces: 有界清单 blind input 和 evaluator-only negative-control Gold。

- [ ] **Step 1: 写最小 blind design**

设计覆盖：最多 8192 字节 JSON 清单、租户准入、应用层并发配额、固定响应、既有集成密钥授权及明确 accepted risk。不得出现“预期 PASS”“P2”“N × 8192”或预期评分。

- [ ] **Step 2: 写脱敏 evidence**

证据按真实顺序描述：先有界读取并解析正文，再检查租户准入和取得应用层并发配额；列出未验证的代理限制、读取超时、worker/连接模型、执行器容量和目标测试结果。不得把 unknown 直接称为 finding。

- [ ] **Step 3: 写 evaluator-only Gold**

Gold 定义：

- `B-P2-001`：P2；单请求 8192 字节有界，但 N 个并发连接仍可在准入前形成约 `N × 8192` 的累计读取与解析成本。
- 缺少可达到并发量、资源耗尽或服务降级证据，因此没有文件证据支持 P0/P1。
- 上游限制影响规模和成本位置，不改变应用处理顺序。
- 未验证项是发布/验收门禁；accepted risk 与 non-goal 只记录为残余风险。
- 最终状态应允许 `PASS`，不新增权限、migration、正文审计、内部入口迁移或通用 API 框架。

### Task 3: 候选隔离检查与用户 Gold 门禁

**Files:**
- Verify: `design-assurance-loop-workspace/iteration-2-prep/**`

**Interfaces:**
- Consumes: Tasks 1–2 的六个候选文件。
- Produces: 用户的 Gold 确认或具体修订意见；未确认时停止。

- [ ] **Step 1: 检查输入不泄漏 Gold**

仅扫描两个案例的 `design.md` 和 `evidence.md`：不得出现 `A-001`、`A-002`、`B-P2-001`、`P0`、`P2`、`requires-user-decision`、`N × 8192`、`expected_output` 或 `expectations`。

- [ ] **Step 2: 检查领域去耦**

executor 输入不得出现 H2/H3/C1 的产品词或符号，包括 `ForceChangePassword`、`entry-a`、`resize`、`passcode`、`VOI`、`VDI` 和 `WebSocket`。输入必须仍保留足以独立证明 finding 的具体调用关系。

- [ ] **Step 3: 交给用户复核 Gold**

向用户提供两个 `decision-ledger.md` 链接，并逐项确认：

1. 案例 A 是否应为 P0，部署可达性是否只记 unknown。
2. 定时消费者是否只证明影响范围扩大，其具体修改是否保持 `requires-user-decision`。
3. 案例 B 是否保持 P2、零 P0/P1，并正确使用并发累计解释。
4. accepted risk、non-goal 和 unknown 门禁是否完整。

Expected: 用户明确回复 Gold 确认。未确认则只修订 prep 文件并重新复核，不继续 Task 4。

### Task 4: 正式化 eval #7/#8

**Files:**
- Create: `design-assurance-loop/evals/files/security/cross-tenant-export-download/design.md`
- Create: `design-assurance-loop/evals/files/security/cross-tenant-export-download/evidence.md`
- Create: `design-assurance-loop/evals/files/security/cross-tenant-export-download/decision-ledger.md`
- Create: `design-assurance-loop/evals/files/api/bounded-manifest-admission/design.md`
- Create: `design-assurance-loop/evals/files/api/bounded-manifest-admission/evidence.md`
- Create: `design-assurance-loop/evals/files/api/bounded-manifest-admission/decision-ledger.md`
- Modify: `design-assurance-loop/evals/evals.json`
- Modify: `design-assurance-loop/evals/catalog.md`

**Interfaces:**
- Consumes: 用户确认的 prep 六文件。
- Produces: eval #7 transfer-heldout（synthetic/metamorphic）和 eval #8 synthetic negative-control。

- [ ] **Step 1: RED — 确认正式目录和 ID 不存在**

Run:

```powershell
$paths = @(
  'design-assurance-loop\evals\files\security\cross-tenant-export-download',
  'design-assurance-loop\evals\files\api\bounded-manifest-admission'
)
if ($paths | Where-Object { Test-Path -LiteralPath $_ }) { throw 'formal case directory exists' }
$data = Get-Content -LiteralPath 'design-assurance-loop\evals\evals.json' -Raw | ConvertFrom-Json
if (@($data.evals.id | Where-Object { $_ -in @(7, 8) }).Count) { throw 'eval #7/#8 already exists' }
```

- [ ] **Step 2: 创建正式三文件案例**

正式文件与用户确认的 prep 内容逐字一致。`evals.json.files` 只能列 `design.md` 和 `evidence.md`；decision ledger 保持 evaluator-only。

- [ ] **Step 3: 注册 blind eval**

eval #7 prompt 只要求评审多租户导出下载的授权边界、影响范围和验证；eval #8 prompt 只要求评审清单接收的处理顺序、容量边界、accepted risk 和 unknown。expectations 只供 grader，覆盖规格中的 Gold 与 Must-not-raise。

- [ ] **Step 4: 更新 catalog 生命周期**

- #7：`transfer-heldout（synthetic/metamorphic）`，`registered; do-not-tune`。
- #8：`negative-control（synthetic）`，`registered; do-not-tune`。
- 明确两案一旦用于 Skill 调优，必须转为 `historical-regression` 并补充新的未见案例。

### Task 5: 冻结 iteration-2 benchmark 输入

**Files:**
- Create: `design-assurance-loop-workspace/iteration-2/manifest.json`
- Create: `design-assurance-loop-workspace/iteration-2/skill-under-test/SKILL.md`
- Create: `design-assurance-loop-workspace/iteration-2/skill-under-test/references/review-rubric.md`
- Create: `design-assurance-loop-workspace/iteration-2/eval-7-cross-tenant-export/inputs/{design.md,evidence.md}`
- Create: `design-assurance-loop-workspace/iteration-2/eval-8-bounded-manifest-control/inputs/{design.md,evidence.md}`
- Create: each eval directory's `eval_metadata.json`

**Interfaces:**
- Consumes: 正式 eval #7/#8 和冻结 Skill/rubric。
- Produces: executor snapshot、grader metadata 和 SHA-256 manifest。

- [ ] **Step 1: RED — 确认 iteration-2 不存在**

Run:

```powershell
if (Test-Path -LiteralPath 'design-assurance-loop-workspace\iteration-2') { throw 'iteration-2 exists; never overwrite benchmark evidence' } else { 'EXPECTED_RED iteration-2 missing' }
```

- [ ] **Step 2: 复制最小可见材料**

只复制 Skill/rubric 和两案 design/evidence。不得复制 catalog、正式 `evals.json` 或 decision ledger。`eval_metadata.json` 只供 grader/viewer，executor 任务消息不得引用它。

- [ ] **Step 3: 写 metadata 与 manifest**

Metadata 包含 eval ID、blind prompt 和 assertions。Manifest 记录 source/snapshot path、SHA-256、每配置三次、paired-wave 调度和 token telemetry 状态。

- [ ] **Step 4: 验证冻结与隔离**

核对 Skill/rubric/spec 哈希；只扫描两个案例的 `inputs/`，不得出现 Gold ID、severity、disposition 或 expectation 文本。哈希不符或隔离失败时停止，不运行 executor。

### Task 6: 运行六组 paired baseline/treatment

**Files:**
- Create: `design-assurance-loop-workspace/iteration-2/eval-*/without_skill/run-{1,2,3}/outputs/review.md`
- Create: `design-assurance-loop-workspace/iteration-2/eval-*/with_skill/run-{1,2,3}/outputs/review.md`
- Create: each run directory's `timing.json`

**Interfaces:**
- Consumes: Task 5 snapshot。
- Produces: 12 个独立 review 和 wall-clock timing。

- [ ] **Step 1: 为每组 run 同时启动两个 fresh-context executor**

Baseline 只读取指定 `inputs/`；Treatment 先完整读取 `skill-under-test/SKILL.md` 和 rubric，再读取相同 inputs。两者都输出最终设计、验收覆盖矩阵、裁决账本和残余风险到唯一指定 `review.md`，不得编辑其它文件。

- [ ] **Step 2: 调度六波**

顺序固定为 eval 7 run 1–3、eval 8 run 1–3。每波结束后检查两个 `review.md` 与两个 `timing.json` 存在且非空，再开始下一波。

- [ ] **Step 3: 处理无效运行**

若 executor 声明读取禁用材料、输出为空或写错目录，则把该 run 标为 invalid，以新 run number 重跑；不得覆盖原始运行。

### Task 7: 独立评分、聚合和分析

**Files:**
- Create: each valid run directory's `grading.json`
- Create: `design-assurance-loop-workspace/iteration-2/benchmark.json`
- Create: `design-assurance-loop-workspace/iteration-2/benchmark.md`
- Create: `design-assurance-loop-workspace/iteration-2/analysis.md`
- Create: `design-assurance-loop-workspace/iteration-2/review.html`

**Interfaces:**
- Consumes: 12 个 executor outputs、metadata 和 evaluator-only Gold。
- Produces: assertion 评分、对比报告、稳定缺口判断和静态评审页。

- [ ] **Step 1: 启动两个 fresh-context grader**

每个 grader 负责一个 eval 的六个 run，完整读取 `C:\Users\zhuxiaokun\.codex\skills\skill-creator\agents\grader.md`、对应 Gold、metadata 和 outputs。`grading.json` 必须包含逐 assertion 的 `text`、`passed`、`evidence`，summary 和 pass rate 自洽。

- [ ] **Step 2: 聚合 benchmark**

Run:

```powershell
python 'C:\Users\zhuxiaokun\.codex\skills\skill-creator\scripts\aggregate_benchmark.py' 'design-assurance-loop-workspace\iteration-2' --skill-name 'design-assurance-loop' --skill-path 'design-assurance-loop'
```

Expected: 两种 configuration 各六个有效 run；每个 eval/config 各三次；生成 `benchmark.json` 和 `benchmark.md`。

- [ ] **Step 3: 独立分析结果**

Analyst 完整读取 `agents/analyzer.md`、benchmark、Gold 和全部 outputs，写 `analysis.md`。必须分别报告：

- eval #7 P0 recall、部署 unknown 处理、共享消费者证据和 disposition。
- eval #8 false P0/P1、P2 严重度、并发累计解释、accepted-risk/unknown 边界。
- Treatment 与 Baseline 的逐项差异和跨 run 方差。
- 是否出现“三次中至少两次失败”的稳定通用能力缺口。

不得在本任务内修改 Skill。

- [ ] **Step 4: 生成静态 viewer**

Run:

```powershell
python 'C:\Users\zhuxiaokun\.codex\skills\skill-creator\eval-viewer\generate_review.py' 'design-assurance-loop-workspace\iteration-2' --skill-name 'design-assurance-loop' --benchmark 'design-assurance-loop-workspace\iteration-2\benchmark.json' --static 'design-assurance-loop-workspace\iteration-2\review.html'
```

Expected: `review.html` 包含 12 个 run 和两种 configuration 的 benchmark 数据。

### Task 8: 用户审阅与人工裁决

**Files:**
- Create: `design-assurance-loop-workspace/iteration-2/human-adjudication/pre-adjudication/**`
- Create: `design-assurance-loop-workspace/iteration-2/human-adjudication/benchmark-human-adjudicated.json`
- Create: `design-assurance-loop-workspace/iteration-2/human-adjudication/benchmark-human-adjudicated.md`

**Interfaces:**
- Consumes: 原始 benchmark、Gold、grader 结果和用户裁决。
- Produces: 保留原始证据的人工修订结果。

- [ ] **Step 1: 给用户静态评审页与检查顺序**

用户先各审一组 eval #7/#8 的 baseline/treatment，再审全部 grader 失败或 configuration 分歧项。判断标准：

1. finding 是否引用实际输入证据，而非复述通用清单。
2. eval #7 是否稳定给出 P0、保留运行态 unknown，并把定时消费者处置留给用户。
3. eval #8 是否稳定保持 P2、零 P0/P1，并正确解释单请求与累计并发成本。
4. accepted risk、non-goal 和 unknown 是否没有被重新升级。

- [ ] **Step 2: 归档裁决前证据**

在修改任何评分前，复制原始 `benchmark.json`、正式 `evals.json` 和两个 Gold 到 `pre-adjudication/`，记录 SHA-256。不得覆盖原始 output 或 grading。

- [ ] **Step 3: 生成 human-adjudicated 报告**

仅应用用户明确确认的评分或 Gold 修订，写新的 JSON/Markdown；原始 benchmark 保持不变。报告区分原始 grader 结果与人工裁决结果。

### Task 9: 最终验证与结论边界

**Files:**
- Verify: formal eval #7/#8 files
- Verify: all `design-assurance-loop-workspace/iteration-2/` artifacts
- Verify unchanged: `design-assurance-loop/SKILL.md`
- Verify unchanged: `design-assurance-loop/references/review-rubric.md`

**Interfaces:**
- Consumes: Tasks 1–8 全部产物。
- Produces: 可复核的 iteration-2 结论和下一步建议。

- [ ] **Step 1: 验证 JSON、路径、哈希和隔离**

解析正式 `evals.json` 及所有 benchmark JSON；断言 eval ID 连续为 1–8、`files` 不引用 decision ledger、manifest 哈希一致、workspace 被 Git 忽略、Skill/rubric 哈希未变。

- [ ] **Step 2: 验证数量和评分自洽**

断言 12 个有效 output、12 个 grading、12 个 timing；每配置六个 run；每 eval/config 三个 run；grading summary、benchmark 聚合和人工报告计数自洽。

- [ ] **Step 3: 仓库检查**

Run:

```powershell
git diff --check
git status --short --untracked-files=all
```

Expected: 只包含当前任务链的 eval、catalog、设计和计划文件；不提交、不推送。

- [ ] **Step 4: 限定结论与下一步**

- 两种配置都通过：不改 Skill，只报告本轮未观察到增量收益。
- Treatment 稳定优于 Baseline：只报告两个新合成领域上的初步迁移证据。
- Treatment 出现稳定缺口：本轮停止；若用户决定调优，先把 #7/#8 转为 historical regression，再单独设计新的未见 held-out/control 验证修改。
