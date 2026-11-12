# Design Assurance Loop Eval Catalog Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把现有三个案例标为历史回归开发集，并形成跨项目 held-out、negative-control 和 metamorphic 候选清单，供用户确认 Gold。

**Architecture:** 保持 skill-creator 的 `evals.json` schema 和现有案例路径不变。新增一个人读 catalog 保存分类、能力覆盖、来源、Gold 置信度和调优状态；README 只记录生命周期和隔离规则。候选未获用户确认前不创建输入附件、不注册 eval。

**Tech Stack:** Markdown、JSON、PowerShell、Git。

## Global Constraints

- 不修改 `design-assurance-loop/SKILL.md`、`references/review-rubric.md` 或 `evals/evals.json`。
- 不移动或改写现有三个案例的 `design.md`、`evidence.md`、`decision-ledger.md`。
- 不创建 held-out、control 或 metamorphic 的正式附件目录。
- 不运行 baseline、treatment、grader 或 eval viewer。
- 候选只保存脱敏摘要和历史会话 ID，不复制凭据、IP、人员名或生产数据。
- 本计划在当前用户已批准的 main 工作区任务链内原地执行，不提交、不推送。
- 受保护文件当前 SHA-256：
  - `evals/evals.json`: `254A16ABA9313D0B7413A6A50147B1FB54F194E7EA4A1E1DEDDD1E35138DC72D`
  - `SKILL.md`: `B548FEF3D9E7142D761E6CC0979E5F9D0DE8C09B2E4E7100B9FBF92DB131B7B0`
  - `references/review-rubric.md`: `D36064ABA6DC2A3E9C249BB1F89E3B9F6BF6CCD2DA3FBEF59B22CF7BF757928F`

---

### Task 1: 建立评测 catalog

**Files:**
- Create: `design-assurance-loop/evals/catalog.md`

**Interfaces:**
- Consumes: 当前 `evals.json` 三个 ID、已确认评测分层规格、历史 rollout 摘要。
- Produces: 人读分类和候选裁决入口；不改变 skill-creator 执行清单。

- [x] **Step 1: 确认 catalog 尚不存在**

Run:

```powershell
if (Test-Path -LiteralPath 'design-assurance-loop\evals\catalog.md') { throw 'catalog already exists; review before overwrite' } else { 'EXPECTED_RED catalog missing' }
```

Expected: `EXPECTED_RED catalog missing`。

- [x] **Step 2: 写 catalog 说明和当前回归集表**

Catalog 必须写明：

- `evals.json` 是工具执行清单，catalog 是人读分类，不向 executor 暴露。
- ID 1–3 均为 `historical-regression`、`directed`、`gold_status=pending-user-confirmation`。
- 三案能力标签分别覆盖交付语义/并发清理/边界，消费者谓词/验证，外部契约/零写入/范围控制。
- 三案可用于调优，但不能用于证明通用性。

- [x] **Step 3: 写 held-out 候选表**

登记以下四个候选，不分配正式 eval ID：

1. `H1 django-migration-chain-collision`：project-manager；迁移链和分支定制；来源 `019fcc25-7fff-7f11-8558-d4f48c435fda`；Gold 置信度 high。
2. `H2 first-login-password-reset-boundary`：VDI；权限旁路、双路由和一次性授权；来源 `019f461c-4ad6-71b1-a6d3-5bf84227bb5a`；Gold 置信度 medium，原因是历史结论为 uncertain。
3. `H3 async-resize-metadata-commit`：OECloud；异步基础设施操作、权威状态和回滚；来源 `019f985c-ed28-7d51-8571-faf9ced3673c`；Gold 置信度 high。
4. `H4 reused-drawer-state-lifecycle`：tob-server frontend；组件复用、状态生命周期和 reset 完整性；来源 `019f1b47-2bc5-7532-8638-bd5c34e7d08e`；Gold 置信度 high，但记录其与现有回归集同仓库。

每行必须包含：评测层、领域、来源、blind prompt、被测机制、proposed finding、must-not-raise、Gold 置信度和状态 `candidate`。

- [x] **Step 4: 写 control 与 metamorphic 候选**

登记：

- `C1 passcode-api-approved-boundaries`：project-manager 已批准外部 API 设计；来源 `019fcbe0-6160-7432-a42f-6f335d26875a`；预期没有 P0/P1；必须保留旧内部 GET 旁路为 accepted-risk，不把无审计正文升级为缺陷。
- `M1 request-success-vs-item-outcome`：从 WS-001 的“请求级成功不等于逐项完成”机制生成跨领域变体；保持失败影响，不复制 WebSocket、MAC、Redis 或原函数名。

- [x] **Step 5: 人工检查 catalog 不泄露未来 executor 输入**

确认 catalog 只供 evaluator/维护者读取；未来 benchmark 临时目录明确排除该文件。

### Task 2: 更新评测附件说明

**Files:**
- Modify: `design-assurance-loop/evals/files/README.md`

**Interfaces:**
- Consumes: `catalog.md` 的评测层和 held-out 生命周期。
- Produces: 新案例创建与注册规则。

- [x] **Step 1: 增加 catalog 职责**

写明 `evals/catalog.md` 保存分类和候选；它与 decision ledger 同属 evaluator-only，不得传给 executor。

- [x] **Step 2: 增加注册门禁**

写明：

- candidate 经过用户确认 Gold 后才创建正式附件和加入 `evals.json`；
- held-out 在当前优化周期内不得参与 Skill 修改；
- held-out 一旦用于调优，转为 historical regression，并补新 held-out；
- control 和 held-out 使用 blind prompt；historical regression 可以 directed。

- [x] **Step 3: 保留原有输入/Gold 隔离规则**

不得弱化干净临时目录、输入文件白名单和脱敏要求。

### Task 3: 静态验证并进入人工 Gold 门禁

**Files:**
- Verify: `design-assurance-loop/evals/catalog.md`
- Verify: `design-assurance-loop/evals/files/README.md`
- Verify unchanged: `design-assurance-loop/evals/evals.json`
- Verify unchanged: `design-assurance-loop/SKILL.md`
- Verify unchanged: `design-assurance-loop/references/review-rubric.md`

**Interfaces:**
- Consumes: Tasks 1–2 的文档。
- Produces: 可由用户逐项批准、修改或拒绝的候选清单。

- [x] **Step 1: 验证 JSON 和受保护文件哈希**

Run:

```powershell
$data = Get-Content -LiteralPath 'design-assurance-loop\evals\evals.json' -Raw | ConvertFrom-Json
if (($data.evals.id -join ',') -ne '1,2,3') { throw 'unexpected eval IDs' }
$expected = @{
  'design-assurance-loop\evals\evals.json' = '254A16ABA9313D0B7413A6A50147B1FB54F194E7EA4A1E1DEDDD1E35138DC72D'
  'design-assurance-loop\SKILL.md' = 'B548FEF3D9E7142D761E6CC0979E5F9D0DE8C09B2E4E7100B9FBF92DB131B7B0'
  'design-assurance-loop\references\review-rubric.md' = 'D36064ABA6DC2A3E9C249BB1F89E3B9F6BF6CCD2DA3FBEF59B22CF7BF757928F'
}
foreach ($path in $expected.Keys) {
  if ((Get-FileHash -Algorithm SHA256 -LiteralPath $path).Hash -ne $expected[$path]) { throw "protected file changed: $path" }
}
```

Expected: exit 0。

- [x] **Step 2: 验证 catalog 完整性**

Run:

```powershell
$text = Get-Content -LiteralPath 'design-assurance-loop\evals\catalog.md' -Raw
foreach ($id in @('1','2','3','H1','H2','H3','H4','C1','M1')) {
  if ($text -notmatch "(?m)^\| $id \|") { throw "missing catalog row: $id" }
}
if ($text -match '(?i)\bTODO\b|\bTBD\b|待填') { throw 'catalog placeholder found' }
```

Expected: exit 0，9 行均存在。

- [x] **Step 3: 扫描敏感信息、尾随空白和 Gold 泄漏边界**

对本阶段两个文档扫描 IPv4、凭据字段实际值、人员名、占位符和尾随空白。候选摘要可以写 proposed finding，但不能出现在未来 `design.md` 或 `evidence.md` 输入中。

- [x] **Step 4: 检查 Git 范围**

Run:

```powershell
git diff --check
git status --short --untracked-files=all
```

Expected: 本阶段只新增 catalog、修改 README、增加本计划；已有三案例和评测分层规格保持原状态。

- [x] **Step 5: 请求用户裁决候选**

向用户列出 H1–H4、C1、M1。每项接受以下答复：`确认`、`修改 finding/severity/must-not-raise`、`拒绝`。未确认前停止，不创建正式案例。

本计划不提交、不推送；候选 Gold 经用户确认后另写正式案例实施计划。
