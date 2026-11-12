# Design Assurance Loop Three Pilot Evals Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 从已有人工作出裁决的历史评审链提炼三个可回放评测案例，并保证执行输入不包含 gold。

**Architecture:** 每个案例沿用仓库既定的 `design.md`、`evidence.md`、`decision-ledger.md` 三文件结构。`evals.json` 只把前两者列为执行输入；裁决账本只供评分。案例只保留复现 finding 所需的脱敏最小事实，不复制完整内部代码或会话。

**Tech Stack:** Markdown、JSON、PowerShell JSON 解析、Git diff 检查。

## Global Constraints

- 仅纳入历史会话中已由用户接受或明确拒绝的 finding。
- 不保存凭据、IP、业务数据正文或完整内部源码。
- 输入附件不得出现 expected finding、severity、accepted fix 或裁决结果。
- 不修改 `design-assurance-loop/SKILL.md`；先建立 RED 案例，再开展对比测试和 Skill 优化。
- 不运行 baseline/treatment；本轮仅完成案例与静态验证。

---

### Task 1: WebSocket 出站契约案例

**Files:**
- Create: `design-assurance-loop/evals/files/backend/websocket-outbound-contract/design.md`
- Create: `design-assurance-loop/evals/files/backend/websocket-outbound-contract/evidence.md`
- Create: `design-assurance-loop/evals/files/backend/websocket-outbound-contract/decision-ledger.md`

**Interfaces:**
- Consumes: 2026-07-09 初稿及两轮独立评审的人工裁决。
- Produces: 覆盖 no-wait 交付语义、版本门禁、stale-running 索引和历史数据非目标的案例。

- [x] **Step 1: 写最小设计输入**
- [x] **Step 2: 写不含结论的代码契约证据**
- [x] **Step 3: 写 evaluator-only 裁决账本**
- [x] **Step 4: 检查输入中不存在 gold 结论**

### Task 2: WebSocket backpressure/pprof 计划案例

**Files:**
- Create: `design-assurance-loop/evals/files/backend/websocket-backpressure-pprof-plan/design.md`
- Create: `design-assurance-loop/evals/files/backend/websocket-backpressure-pprof-plan/evidence.md`
- Create: `design-assurance-loop/evals/files/backend/websocket-backpressure-pprof-plan/decision-ledger.md`

**Interfaces:**
- Consumes: 2026-07-01 三轮计划评审中已接受的缺陷。
- Produces: 覆盖 backlog 口径、测试矛盾、自包含性、错误调优和 pprof 验证边界的案例。

- [x] **Step 1: 合成多轮缺陷仍可共存的最小计划快照**
- [x] **Step 2: 写消费者谓词和超时常量证据**
- [x] **Step 3: 按历史影响确定 P1/P2，不抬高严重级别**
- [x] **Step 4: 检查输入中不存在修复答案**

### Task 3: 企微禅道范围与契约门禁案例

**Files:**
- Create: `design-assurance-loop/evals/files/fullstack/wecom-zentao-contract-gates/design.md`
- Create: `design-assurance-loop/evals/files/fullstack/wecom-zentao-contract-gates/evidence.md`
- Create: `design-assurance-loop/evals/files/fullstack/wecom-zentao-contract-gates/decision-ledger.md`

**Interfaces:**
- Consumes: 2026-07-13 初始设计/计划及后续人工裁决。
- Produces: 同时测试真实契约漏检和“不要把已接受的小规模约束升级为阻塞项”。

- [x] **Step 1: 写外部接口尚未验证的最小设计输入**
- [x] **Step 2: 写 API、字段类型和用户范围证据**
- [x] **Step 3: 写 accepted 与 non-goal 双向裁决账本**
- [x] **Step 4: 检查输入没有提示哪些项目应接受或拒绝**

### Task 4: 注册三个正式评测并定义隔离规则

**Files:**
- Modify: `design-assurance-loop/evals/evals.json`
- Modify: `design-assurance-loop/evals/files/README.md`

**Interfaces:**
- Consumes: Tasks 1-3 的输入附件与 gold。
- Produces: skill-creator 兼容的三个 eval 条目；执行器只接收 `files` 数组列出的文件。

- [x] **Step 1: 向 `evals.json` 加入唯一整数 ID 1-3**
- [x] **Step 2: 为每案写可判定 expectations**
- [x] **Step 3: 记录 benchmark 必须使用不含 `evals/` 的 Skill 快照**

### Task 5: 静态验证

**Files:**
- Verify: `design-assurance-loop/evals/evals.json`
- Verify: `design-assurance-loop/evals/files/**`

- [x] **Step 1: 用 `ConvertFrom-Json` 解析评测清单并确认三条 ID 唯一**
- [x] **Step 2: 确认每个输入路径存在且没有把 `decision-ledger.md` 列入 `files`**
- [x] **Step 3: 扫描敏感模式、占位符、gold 泄漏和尾随空白**
- [x] **Step 4: 运行 `git diff --check` 并人工检查变更范围**

本计划不提交、不推送；用户确认三个 gold 后，再进入 baseline/treatment 对比。
