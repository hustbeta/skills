# Design Assurance Loop Negative Control Benchmark Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 C1 固化为 eval #6，并对冻结的 #4、#5、#6 各运行三次无 Skill baseline 与当前 Skill treatment，生成可供用户审阅的静态对比报告。

**Architecture:** 正式案例仍使用 `design.md`、`evidence.md`、`decision-ledger.md` 三文件隔离。Benchmark 在被 `.gitignore` 排除的 `design-assurance-loop-workspace/iteration-1/` 中运行，只复制 Skill、rubric 和 executor 输入；不复制 catalog、`evals.json`、expectations 或 Gold。每个案例每轮同时启动 fresh-context baseline/treatment，随后由独立 grader 读取 Gold 评分，使用 skill-creator 自带脚本聚合并生成静态 viewer。

**Tech Stack:** Markdown、JSON、PowerShell、Python、skill-creator、Codex subagents、Git。

## Global Constraints

- 已获用户批准：正式化 C1，然后冻结 #4/#5/#6 运行 baseline/treatment；无需再次确认设计。
- 不修改 `design-assurance-loop/SKILL.md` 或 `references/review-rubric.md`，不根据本轮 held-out 结果调优 Skill。
- H4、M1、M2 继续保持 catalog-only。
- 每个 eval、每种 configuration 运行 3 次；configuration 名固定为 `with_skill`、`without_skill`。
- baseline 不得读取 Skill/rubric；treatment 只读取冻结的 Skill/rubric；两者均不得读取 catalog、`evals.json`、expectations 或 decision ledger。
- 受并发槽限制，按同一 eval、同一 run number 的 baseline/treatment 成对同时启动；不同 pair 分批调度，并在 benchmark notes 记录该限制。
- 协作工具不提供 token telemetry；`total_tokens` 记录为 `0` 并标注 unavailable，不使用 output 字符数冒充 token 数。
- Benchmark workspace 已被根 `.gitignore` 的 `*-workspace/` 规则排除；不提交、不推送。
- 当前 main 工作区包含同一用户批准任务链的相关未提交文件，原地追加，不清理、不覆盖。
- treatment 运行前冻结并记录 Skill、rubric、三个 eval 输入及 Gold 的 SHA-256。

---

### Task 1: 固化 C1 negative control

**Files:**
- Create: `design-assurance-loop/evals/files/api/passcode-approved-boundaries/design.md`
- Create: `design-assurance-loop/evals/files/api/passcode-approved-boundaries/evidence.md`
- Create: `design-assurance-loop/evals/files/api/passcode-approved-boundaries/decision-ledger.md`
- Modify: `design-assurance-loop/evals/evals.json`
- Modify: `design-assurance-loop/evals/catalog.md`

**Interfaces:**
- Consumes: 已通过独立门禁的外部 Passcode API 设计、当前实现证据和 accepted-risk 边界。
- Produces: eval #6；预期无 P0/P1、允许一个已知 P2 和运行环境 unknown 的 evaluator Gold。

- [x] **Step 1: RED — 确认目录与 ID 不存在**

Run:

```powershell
if (Test-Path -LiteralPath 'design-assurance-loop\evals\files\api\passcode-approved-boundaries') { throw 'C1 directory exists' }
$data = Get-Content -LiteralPath 'design-assurance-loop\evals\evals.json' -Raw | ConvertFrom-Json
if ($data.evals.id -contains 6) { throw 'eval #6 exists' }
```

Expected: exit 0，目录和 ID 均不存在。

- [x] **Step 2: 写 blind design/evidence**

`design.md` 必须覆盖：复用既有客户端及权限、HMAC/时间窗/Nonce/限流/Redis fail-closed、严格日期 JSON、固定响应、不写业务审计、旧内部 GET 与全量既有客户端授权为 accepted risk。`evidence.md` 必须记录当前实现先有界读取 1024 字节正文再做来源 IP 限流，以及测试和生产配置未在本轮执行/验证。两文件不得出现“预期 PASS”“无 P0/P1”或 Gold ID。

- [x] **Step 3: 写 evaluator-only Gold**

Gold 必须定义：

- 无文件证据支持的 P0/P1；任何 P0/P1 都必须逐条证明违反设计契约，而不是重开已接受范围。
- 允许 `C1-P2-001`：实现的来源 IP 限流顺序晚于设计；因正文硬上限为 1024 字节且代理/部署限制 unknown，严重度不高于 P2。
- 生产 TLS、Redis、可信代理、密钥配置及测试实际结果标为 unknown。
- 不新增权限/migration，不迁入其它路径，不要求业务审计正文或 hash，不重开旧内部 GET，不建设通用 API 引擎。

- [x] **Step 4: 注册 eval #6 并更新 catalog**

Prompt 只要求评审外部日期口令 API 的契约、边界和验证，不提示预期 PASS。`files` 只列 design/evidence。Catalog 状态改为 `registered-negative-control；do-not-tune`。

### Task 2: 冻结 benchmark 输入

**Files:**
- Create: `design-assurance-loop-workspace/iteration-1/manifest.json`
- Create: `design-assurance-loop-workspace/iteration-1/skill-under-test/SKILL.md`
- Create: `design-assurance-loop-workspace/iteration-1/skill-under-test/references/review-rubric.md`
- Create: `design-assurance-loop-workspace/iteration-1/eval-4-first-login-boundary/eval_metadata.json`
- Create: `design-assurance-loop-workspace/iteration-1/eval-5-async-state-authority/eval_metadata.json`
- Create: `design-assurance-loop-workspace/iteration-1/eval-6-approved-boundaries-control/eval_metadata.json`
- Create: each eval directory's `inputs/design.md` and `inputs/evidence.md`

**Interfaces:**
- Consumes: evals #4/#5/#6 and current Skill/rubric.
- Produces: immutable-by-convention executor snapshot and grader metadata.

- [x] **Step 1: RED — 确认 iteration-1 不存在**

Run:

```powershell
if (Test-Path -LiteralPath 'design-assurance-loop-workspace\iteration-1') { throw 'iteration-1 exists; never overwrite benchmark evidence' }
```

- [x] **Step 2: 只复制 executor 可见材料**

复制 Skill/rubric，以及三个 eval 的 design/evidence。禁止复制 catalog、`evals.json` 或 decision ledger。创建后扫描 workspace，不得出现 `decision-ledger`、`catalog.md`、`expected_output` 或 `expectations`。

- [x] **Step 3: 写 eval metadata**

每个 metadata 包含正式 eval ID、描述性名称、原 blind prompt 和 assertions。Assertions 从 evaluator Gold 派生，但 metadata 只供 grader/viewer，运行 agent 的任务消息不得引用 metadata。

- [x] **Step 4: 写 SHA-256 manifest**

Manifest 记录 source path、snapshot path、SHA-256、eval IDs、每配置 3 次、paired wave 调度、token telemetry unavailable。冻结后不得修改 snapshot；校验失败必须新建 iteration，不得覆盖。

### Task 3: 运行 paired baseline/treatment

**Files:**
- Create: `design-assurance-loop-workspace/iteration-1/eval-*/without_skill/run-{1,2,3}/outputs/review.md`
- Create: `design-assurance-loop-workspace/iteration-1/eval-*/with_skill/run-{1,2,3}/outputs/review.md`
- Create: each run directory's `timing.json`

**Interfaces:**
- Consumes: Task 2 snapshot；baseline 只消费 `inputs/`，treatment 额外消费 `skill-under-test/`。
- Produces: 18 个独立 `review.md` 和可用 wall-clock timing。

- [x] **Step 1: 为每个 pair 同时启动两个 fresh-context agent**

Baseline task 必须明确：只读取指定 inputs，不读取任何 Skill、原仓库 eval 资产或 evaluator metadata。Treatment task 必须明确：先完整读取冻结 Skill 和 rubric，再只读取指定 inputs。二者均输出最终设计、验收覆盖矩阵、裁决账本和残余风险到指定 `review.md`，不得编辑其它文件。

- [x] **Step 2: 调度九个 pair**

顺序固定为 eval 4 run 1–3、eval 5 run 1–3、eval 6 run 1–3。每个 pair 完成后验证两个 `review.md` 和 `timing.json` 均存在、非空，再启动下一 pair。

- [x] **Step 3: 校验盲测边界**

逐 run 检查 agent 最终报告未说明读取禁用文件。若 agent 违反隔离，该 run 标为 invalid 并使用新的 run number 重跑，不覆盖原 run。

### Task 4: 独立评分、聚合和静态 viewer

**Files:**
- Create: each run directory's `grading.json`
- Create: `design-assurance-loop-workspace/iteration-1/benchmark.json`
- Create: `design-assurance-loop-workspace/iteration-1/benchmark.md`
- Create: `design-assurance-loop-workspace/iteration-1/analysis.md`
- Create: `design-assurance-loop-workspace/iteration-1/review.html`

**Interfaces:**
- Consumes: 18 个 outputs、三个 eval metadata 和原正式 decision ledger。
- Produces: 可复核的 expectation pass rate、方差、分析结论及静态人工评审页。

- [x] **Step 1: 用三个 fresh-context grader 并行评分**

每个 grader 只负责一个 eval 的六个 run，读取 skill-creator `agents/grader.md`、对应 Gold、metadata 和 outputs。每个 `grading.json` 的 expectations 必须使用 `text`、`passed`、`evidence`，summary 数量与 pass_rate 必须自洽。P0/P1 recall 与 false P0/P1 边界优先于措辞相似度。

- [x] **Step 2: 运行官方聚合脚本**

Run:

```powershell
python 'C:\Users\zhuxiaokun\.codex\skills\skill-creator\scripts\aggregate_benchmark.py' 'design-assurance-loop-workspace\iteration-1' --skill-name 'design-assurance-loop' --skill-path 'design-assurance-loop'
```

Expected: 生成 `benchmark.json`、`benchmark.md`，两种 configuration 各 9 个有效 run，metadata 中 `runs_per_configuration=3`。

- [x] **Step 3: 独立 analyst 解释结果**

Analyst 读取 skill-creator `agents/analyzer.md`、benchmark 和 outputs，写 `analysis.md`。必须检查：非区分性 assertion、跨 run 方差、H2/H3 P1 recall、C1 false P0/P1、accepted-risk 越界、证据引用，以及 token telemetry unavailable。不得提出基于 held-out 的 Skill 修改。

- [x] **Step 4: 生成静态 viewer**

Run:

```powershell
python 'C:\Users\zhuxiaokun\.codex\skills\skill-creator\eval-viewer\generate_review.py' 'design-assurance-loop-workspace\iteration-1' --skill-name 'design-assurance-loop' --benchmark 'design-assurance-loop-workspace\iteration-1\benchmark.json' --static 'design-assurance-loop-workspace\iteration-1\review.html'
```

Expected: `review.html` 存在且包含 18 个 run 和 benchmark 数据。

### Task 5: 最终验证与用户审阅交接

**Files:**
- Verify: formal eval #6 files
- Verify: all `design-assurance-loop-workspace/iteration-1/` artifacts
- Verify unchanged: `design-assurance-loop/SKILL.md`
- Verify unchanged: `design-assurance-loop/references/review-rubric.md`

**Interfaces:**
- Consumes: Tasks 1–4 全部产物。
- Produces: 用户可按明确标准完成的人工质量门禁。

- [x] **Step 1: 验证 JSON、哈希、隔离和 Git 范围**

断言 eval IDs 为 `1,2,3,4,5,6`；manifest 哈希与 source/snapshot 一致；workspace 不受 Git 跟踪；Skill/rubric 与 benchmark 前哈希一致；运行 `git diff --check`。

- [x] **Step 2: 验证 benchmark 完整性**

断言 18 个 output、18 个 grading、18 个 timing；每配置 9 run；每个 eval/config 3 run；grading summary 自洽；viewer 可读取。

- [x] **Step 3: 告知用户唯一需要的操作**

提供 `review.html` 链接。用户至少审阅每个 eval 的一组 baseline/treatment，以及 grader 标为失败或存在分歧的全部 run。判断标准：

1. H2 treatment 是否同时识别宽匹配旁路和共享处理器额外入口，且不擅自扩大范围或升级 P0。
2. H3 treatment 是否区分异步受理/最终状态、明确权威源和失败收敛，且不写死队列/轮询实现。
3. C1 treatment 是否保持零 false P0/P1、尊重 accepted risk，并把部署事实标为 unknown；P2 顺序漂移可以报告。
4. finding 必须引用输入证据；只复述通用检查清单不算通过。
5. 若决定依据本轮结果调优 Skill，先把 #4/#5/#6 转为 historical regression，并补充新 held-out/control，不能继续用同一批案例证明通用性。
