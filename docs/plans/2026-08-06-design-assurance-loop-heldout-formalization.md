# Design Assurance Loop Held-out Formalization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 将已确认的 H2、H3 固化为跨项目 held-out eval，并用无 Gold 上下文的独立 reviewer 判断 C1 是否可作为 negative control。

**Architecture:** 延续现有三文件案例结构。`design.md` 与 `evidence.md` 只放脱敏、可执行的 blind input；`decision-ledger.md` 与 `catalog.md` 保持 evaluator-only。H2 用通用“共享实现/多入口”语言，不把产品路由名写成能力规则；H3 只编码“异步受理不等于权威终态”。C1 本阶段只过门禁，不注册、不创建附件。

**Tech Stack:** Markdown、JSON、PowerShell、Git。

## Global Constraints

- 只正式化 H2、H3；H4、M1、M2 保持 catalog-only。
- C1 reviewer 不得读取 `evals/catalog.md`、`evals.json` 的 expectations 或任何 proposed Gold。
- C1 若发现有证据支持的 P0/P1，则标为 `gate-blocked`；否则标为 `gate-passed`，仍不注册。
- 不修改 `design-assurance-loop/SKILL.md` 或 `references/review-rubric.md`，不根据 held-out 结果调优 Skill。
- 不运行 baseline、treatment、grader 或 viewer，不提交、不推送。
- 输入不得包含真实凭据、IP、人员名、实例 ID 或产品专有路由名。
- 当前工作区中的相关未提交改动属于同一用户批准任务链，原地追加，不覆盖或清理。
- 受保护文件当前 SHA-256：
  - `SKILL.md`: `B548FEF3D9E7142D761E6CC0979E5F9D0DE8C09B2E4E7100B9FBF92DB131B7B0`
  - `references/review-rubric.md`: `D36064ABA6DC2A3E9C249BB1F89E3B9F6BF6CCD2DA3FBEF59B22CF7BF757928F`

---

### Task 1: 固化 H2 首登强制改密权限边界

**Files:**
- Create: `design-assurance-loop/evals/files/security/first-login-password-reset-boundary/design.md`
- Create: `design-assurance-loop/evals/files/security/first-login-password-reset-boundary/evidence.md`
- Create: `design-assurance-loop/evals/files/security/first-login-password-reset-boundary/decision-ledger.md`

**Interfaces:**
- Consumes: 已确认的权限旁路、首登状态及同一 handler 被多个入口复用的证据。
- Produces: 不依赖项目词汇的 blind input 与 evaluator-only Gold。

- [x] **Step 1: RED — 确认正式目录不存在**

Run:

```powershell
if (Test-Path -LiteralPath 'design-assurance-loop\evals\files\security\first-login-password-reset-boundary') { throw 'H2 already exists' } else { 'EXPECTED_RED H2 missing' }
```

- [x] **Step 2: 写最小 blind design**

设计快照保留请求/响应 JSON，提出只移除某一精确豁免、保留通用前缀豁免和共享 handler 的局部方案。正文只使用“入口 A / 入口 B / 共享 handler”，不出现具体产品路由名。

- [x] **Step 3: 写脱敏 evidence**

证据必须包含：通用前缀认证旁路、额外精确豁免、handler 可按正文用户名直接重置密码、登录结果已有首登状态、两个入口挂载同一 handler。不得直接写出结论或修复答案。

- [x] **Step 4: 写 Gold ledger**

记录：

- `H2-001`：P1，敏感操作缺少与首登流程绑定的正向授权证明；仅移除精确豁免无效。
- `H2-002`：P1，修改共享实现前必须枚举其它入口/消费者；其它入口如何处理为 `requires-user-decision`。
- 保留接口 JSON；不强制改造成普通会话认证；无部署可达性证据时不得升级 P0；不得自动扩大为改造所有入口。

### Task 2: 固化 H3 异步 resize 与元数据提交边界

**Files:**
- Create: `design-assurance-loop/evals/files/cloud/async-resize-metadata-commit/design.md`
- Create: `design-assurance-loop/evals/files/cloud/async-resize-metadata-commit/evidence.md`
- Create: `design-assurance-loop/evals/files/cloud/async-resize-metadata-commit/decision-ledger.md`

**Interfaces:**
- Consumes: provider 异步 resize、应用立即写目标规格、后续回滚造成分叉的证据。
- Produces: 跨基础设施提供方可迁移的权威状态与提交时机 Gold。

- [x] **Step 1: RED — 确认正式目录不存在**

Run:

```powershell
if (Test-Path -LiteralPath 'design-assurance-loop\evals\files\cloud\async-resize-metadata-commit') { throw 'H3 already exists' } else { 'EXPECTED_RED H3 missing' }
```

- [x] **Step 2: 写最小 blind input**

设计快照在 provider 接受请求后立即覆盖应用有效规格，并把后续状态轮询视为独立运维流程。证据说明 provider 可异步失败/回滚、应用表当前同步写目标值、历史上出现过终态回滚。

- [x] **Step 3: 写 Gold ledger**

记录：

- `H3-001`：P1，异步受理不能提交为已生效业务事实。
- `H3-002`：P1，必须定义成功、失败、超时和回滚后的对账/恢复路径。
- provider 终态是规格权威来源；guest 只作运行态佐证；实现可选 pending/effective、延迟提交或条件对账，不把某一种机制写死。

### Task 3: 注册 H2/H3 并更新 catalog

**Files:**
- Modify: `design-assurance-loop/evals/evals.json`
- Modify: `design-assurance-loop/evals/catalog.md`

**Interfaces:**
- Consumes: Tasks 1–2 的 blind input 与 Gold。
- Produces: eval ID 4、5；H2/H3 状态 `registered-heldout`。

- [x] **Step 1: RED — 确认 ID 4、5 尚未注册**

Run:

```powershell
$data = Get-Content -LiteralPath 'design-assurance-loop\evals\evals.json' -Raw | ConvertFrom-Json
if (@($data.evals.id | Where-Object { $_ -in @(4, 5) }).Count) { throw 'IDs 4/5 already exist' } else { 'EXPECTED_RED IDs 4/5 missing' }
```

- [x] **Step 2: 追加 blind eval**

ID 4 检查正向授权、通用豁免和共享实现影响面；ID 5 检查异步受理、权威终态、对账与回滚。`files` 只列 design/evidence，不列 Gold。

- [x] **Step 3: 更新 catalog 状态**

H2、H3 标为 `registered-heldout; do-not-tune`。H2 的产品事实仅保留为来源证据，能力标签使用通用“shared implementation / multiple entry points”。

### Task 4: 独立验证 C1 negative-control 门禁

**Files:**
- Modify: `design-assurance-loop/evals/catalog.md`

**Interfaces:**
- Consumes: C1 原始设计、当前相关代码和 `review-rubric.md`，不消费 catalog/Gold。
- Produces: `gate-passed` 或 `gate-blocked` 及简短证据结论；不创建 C1 正式目录。

- [x] **Step 1: 启动 fresh-context 只读 reviewer**

Reviewer 必须：

- 不继承当前对话；
- 不读取 `evals/catalog.md`、`evals.json`、decision ledger；
- 只报告有代码/设计证据支持的 P0/P1、unknown 和 accepted-risk 边界；
- 不编辑任何文件。

- [x] **Step 2: 应用门禁**

若有 P0/P1，catalog 写 `gate-blocked` 并记录 finding 摘要；若无 P0/P1，写 `gate-passed`，同时保留“尚未注册、尚未 benchmark”的限制。

### Task 5: 静态验证

**Files:**
- Verify: `design-assurance-loop/evals/evals.json`
- Verify: H2/H3 六个附件
- Verify: `design-assurance-loop/evals/catalog.md`
- Verify unchanged: `design-assurance-loop/SKILL.md`
- Verify unchanged: `design-assurance-loop/references/review-rubric.md`

- [x] **Step 1: JSON 与路径验证**

解析 JSON，断言 ID 为 `1,2,3,4,5`、每个 `files` 路径存在、路径不包含 `decision-ledger.md`。

- [x] **Step 2: 隔离与敏感信息验证**

扫描 H2/H3 输入中的 Gold ID、`accepted_fix`、真实 IPv4、UUID、凭据字段实际值和产品专有路由名；扫描 decision ledger 未被 `files` 引用。

- [x] **Step 3: 保护与差异验证**

核对 Skill/rubric SHA-256，运行：

```powershell
git diff --check
git status --short --untracked-files=all
```

Expected: 仅出现本任务链相关评测文档与 JSON；不提交、不推送。
