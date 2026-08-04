# Skills Repository Initialization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 初始化可扩展的多 Skill 仓库，并提供可用的 `design-assurance-loop` 最小版本。

**Architecture:** 仓库级规则和文档留在根目录；每个 Skill 作为根目录下的独立单元，资源与评测数据就近存放。首个 Skill 只使用 Markdown、YAML frontmatter 和 JSON，不引入脚本、依赖或自定义 agent 配置。

**Tech Stack:** Markdown、YAML frontmatter、JSON、Git、PowerShell

## Global Constraints

- Skill 直接位于仓库根目录，不增加第二层 `skills/`。
- 通用设计和计划分别存放在 `docs/designs/` 与 `docs/plans/`。
- 不增加自定义 agent、自动化脚本、CI、空领域目录或许可证。
- 不推送远端；每次提交只包含对应任务文件。
- 新增代码时必须添加解释最终行为或非直观原因的注释；本计划仅新增声明式文本文件。
- 不增加外部依赖。

---

### Task 1: 仓库级说明与约束

**Files:**
- Create: `README.md`
- Create: `AGENTS.md`
- Create: `.gitignore`

**Interfaces:**
- Consumes: `docs/designs/2026-08-04-skills-repository-initialization.md` 中批准的目录约束。
- Produces: 后续 Skill 共用的仓库导航、贡献规则和本地忽略规则。

- [ ] **Step 1: 写入仓库说明**

Create `README.md`:

```markdown
# Skills

可复用 Agent Skills 集合。每个 Skill 位于仓库根目录，可独立阅读、复制和安装。

## Skill 清单

- [`design-assurance-loop`](design-assurance-loop/)：对非平凡设计方案执行独立评审、裁决、修订和收敛门禁。

## 目录约定

- `<skill-name>/SKILL.md`：Skill 入口与工作流。
- `<skill-name>/references/`：按需读取的规则和领域资料。
- `<skill-name>/evals/`：经过人工确认的回归用例及附件。
- `docs/designs/`：仓库与 Skill 的设计记录。
- `docs/plans/`：已确认设计对应的实施计划。

仅在出现真实内容时创建 `scripts/`、`assets/` 或新的评测分类。仓库级工具只有被多个 Skill 复用时才提取到根目录。
```

- [ ] **Step 2: 写入 Agent 维护规则**

Create `AGENTS.md`:

```markdown
# Repository Guidelines

## Skill 结构

- 每个 Skill 直接位于仓库根目录，目录名使用 kebab-case。
- 每个 Skill 必须包含 `SKILL.md`，且目录名与 frontmatter 的 `name` 一致。
- `SKILL.md` 只保留核心工作流；较长规则放入 `references/` 并明确读取时机。
- 仅在有实际内容时增加 `scripts/`、`references/`、`assets/` 或 `evals/`。

## 变更原则

- 编码、代码审查、调试或重构默认使用 `karpathy-guidelines` skill。
- 改动保持小而精准，不增加未经请求的抽象、依赖或预留结构。
- 非平凡修改前先定义可验证的成功标准。
- 新增或修改代码必须带有恰当注释；注释说明最终行为或非直观原因，不记录冗长变更历史。

## 验证

- 修改 `SKILL.md` 后检查 frontmatter 的 `name` 和 `description`。
- 修改 `evals/*.json` 后使用 JSON 解析器验证格式。
- 提交前运行 `git diff --check`，并确认暂存范围只包含当前任务。
```

- [ ] **Step 3: 写入最小忽略规则**

Create `.gitignore`:

```gitignore
.DS_Store
Thumbs.db
*.tmp
*-workspace/
```

- [ ] **Step 4: 验证文件内容和差异格式**

Run:

```powershell
rg -n "^# Skills$|design-assurance-loop|docs/plans" README.md
rg -n "karpathy-guidelines|git diff --check" AGENTS.md
git diff --check
```

Expected: 两次 `rg` 均匹配目标行；`git diff --check` 无输出且退出码为 0。

- [ ] **Step 5: 提交仓库级文件**

```powershell
git add README.md AGENTS.md .gitignore
git diff --cached --check
git commit -m "chore: 初始化 Skills 仓库规范"
```

### Task 2: Design Assurance Loop 工作流

**Files:**
- Create: `design-assurance-loop/SKILL.md`
- Create: `design-assurance-loop/references/review-rubric.md`

**Interfaces:**
- Consumes: 用户提供的设计文档、需求约束、代码根目录、已接受风险和明确非目标。
- Produces: 最终设计、验收覆盖矩阵、裁决账本、残余风险及评测候选。

- [ ] **Step 1: 写入 Skill 入口与有界评审流程**

Create `design-assurance-loop/SKILL.md`:

```markdown
---
name: design-assurance-loop
description: 对非平凡技术设计执行基于代码证据的独立评审、裁决、修订和收敛门禁。用户要求审查或完善设计方案、减少多轮人工复审、启动多个 reviewer、检查 P0/P1 风险，或希望 Codex 自行循环评审时使用。
---

# Design Assurance Loop

把设计交付视为有停止条件的验证流程，而非单次文档写作。目标是降低真实严重缺陷和人工干预，不以 finding 数量衡量质量。

## 前置输入

开始前确认以下内容可用：

- 当前设计文档路径。
- 代码或配置证据的根目录。
- 目标、非目标和验收标准。
- 必须保持的不变量与禁止发生的回归。
- 用户已接受的风险和仍需决策的未知项。

缺少会改变方案边界的输入时，先请求用户决策。其余可验证信息从代码、配置和现有文档读取。

## 工作流

1. 完整读取 `references/review-rubric.md`。
2. 将输入整理为 canonical brief：目标、非目标、现状证据、不变量、接口与数据约束、验收标准、已接受风险、未知项。
3. 沿真实代码路径建立 evidence index，覆盖入口、调用方、数据读写、旁路实现、异步副作用、错误处理、部署和回滚路径。未核实推断标记为 `unknown`。
4. 并行启动三个上下文独立、只读 reviewer：
   - 证据与契约 reviewer：核对需求、接口、兼容性、入口、调用方和副作用。
   - 故障与安全 reviewer：核对并发、事务、幂等、容量、安全、迁移、降级和回滚。
   - 可执行性与验证 reviewer：核对实施顺序、依赖、测试、部署、验收和可观测性。
5. reviewer 只接收 canonical brief、evidence index 和当前设计，不接收作者辩护或其他 reviewer 的推理。reviewer 按 rubric 输出 findings，不修改设计。
6. 父 agent 去重并逐条裁决 finding，记录 `accepted`、`rejected`、`duplicate`、`requires-user-decision`、`deferred` 或 `non-goal` 及理由。
7. 仅修订已接受的 P0/P1；P2/P3 保留在账本中，除非它们阻止 P0/P1 闭环或用户明确要求处理。
8. 修订后让受影响 reviewer 定向复查，再启动一个未参与前序评审的 final reviewer。final reviewer 只看原始约束、最终设计和证据，不看前序推理。
9. 最多执行三轮完整评审。满足质量门禁时输出 `PASS`；达到上限仍有已接受的 P0/P1 时输出 `NOT_CONVERGED` 和阻塞项，不得声称通过。

独立子代理不可用时，可以执行分角色串行检查，但必须标记 `REDUCED_INDEPENDENCE`，不得把结果描述为独立上下文评审。

## 质量门禁

- 完整评审轮次中没有已接受的 P0/P1。
- 每项验收标准均映射到设计章节、代码位置和验证方式。
- 所有已识别入口、旁路和副作用均有明确结论。
- 未知项、残余风险和回滚限制均已记录。
- 用户明确接受的风险和非目标未被重新扩入实施范围。

## 最终输出

按以下顺序交付：

1. 状态：`PASS`、`NOT_CONVERGED` 或 `REDUCED_INDEPENDENCE`。
2. 最终设计方案。
3. 验收标准覆盖矩阵。
4. 精简裁决账本。
5. 残余风险和待决策项。
6. 评测候选；仅收录人工确认的漏检、误报、回归或新 P0/P1，不自动写入正式评测集。
```

- [ ] **Step 2: 写入共享评审规则**

Create `design-assurance-loop/references/review-rubric.md`:

````markdown
# Design Review Rubric

## Finding 格式

每条 finding 使用以下字段：

```yaml
id: F-001
severity: P1
category: contract
violated_contract: "创建成功必须意味着记录可被后续查询读取"
evidence:
  - "internal/service/create.go:42 写入主记录"
  - "internal/service/create.go:61 在异步刷新前返回成功"
failure_scenario: "刷新失败时接口返回成功，但查询仍读不到新记录"
recommended_fix: "将可见性更新纳入成功条件，或明确返回异步状态并提供查询门禁"
confidence: high
disposition_candidate: accepted
```

证据不足时将 `confidence` 设为 `low`，并把候选结论设为 `requires-user-decision`；不得直接定为 P0/P1。

## 严重级别

- `P0`：可能造成不可恢复的数据损坏、重大安全事故、系统性不可用，或直接破坏核心业务不变量。
- `P1`：主要路径无法工作、接口或数据契约被破坏、存在高概率生产事故，且没有可靠规避方式。
- `P2`：局部功能、运维性、性能或测试质量问题，有明确规避或较低发生概率。
- `P3`：表达、维护性或非阻塞改进建议。

严重级别由失败影响和发生条件决定，不因修改成本高低而调整。

## 证据规则

- 优先引用具体代码、配置、API 契约、数据库约束和用户确认的需求。
- 每条 P0/P1 必须给出可复现失败场景，并说明证据如何支持结论。
- 推断与事实分开；无法读取的运行环境或外部系统标记为 `unknown`。
- 用户明确接受的风险与非目标只记录为 residual risk，不升级为缺陷。
- reviewer 不修改设计，不把个人偏好包装成契约问题。

## Reviewer 检查面

### 证据与契约

- 入口、调用方、旁路路径和异步副作用是否完整。
- 请求、响应、持久化、缓存和审计契约是否一致。
- 兼容性、版本门禁和历史数据行为是否明确。

### 故障与安全

- 并发、事务、幂等、重试和清理是否可能破坏数据。
- 容量、超时、降级、回滚和迁移是否存在不可控窗口。
- 信任边界、权限和敏感数据处理是否完整。

### 可执行性与验证

- 修改位置、依赖顺序和部署步骤是否可执行。
- 测试能否稳定证明验收标准，而非依赖模糊观察或固定等待。
- 监控、灰度、回滚触发条件和验证命令是否明确。

## 裁决与停止条件

父 agent 为每条 finding 记录 disposition 和依据。重复项合并但保留来源；需要用户改变范围或业务取舍的项目不得代替用户决定。

一次完整评审只有在已接受 P0/P1 为零时通过。三轮后仍未通过时输出 `NOT_CONVERGED`，列出阻塞 finding、已有证据和所需决策。
````

- [ ] **Step 3: 验证 frontmatter、核心门禁和差异格式**

Run:

```powershell
$skill = Get-Content -LiteralPath 'design-assurance-loop\SKILL.md' -Raw
if ($skill -notmatch '(?s)^---\r?\nname: design-assurance-loop\r?\ndescription: .+?\r?\n---') { throw 'invalid SKILL.md frontmatter' }
rg -n "NOT_CONVERGED|REDUCED_INDEPENDENCE|最多执行三轮" design-assurance-loop/SKILL.md
rg -n "^## Finding 格式$|^## 严重级别$|^## 证据规则$" design-assurance-loop/references/review-rubric.md
git diff --check
```

Expected: frontmatter 检查不抛错；两次 `rg` 均匹配全部目标；`git diff --check` 退出码为 0。

- [ ] **Step 4: 提交 Skill 工作流**

```powershell
git add design-assurance-loop/SKILL.md design-assurance-loop/references/review-rubric.md
git diff --cached --check
git commit -m "feat(design-assurance-loop): 添加评审闭环"
```

### Task 3: 回归评测骨架与最终验证

**Files:**
- Create: `design-assurance-loop/evals/evals.json`
- Create: `design-assurance-loop/evals/files/README.md`

**Interfaces:**
- Consumes: 已完成人工裁决的真实设计评审案例。
- Produces: 可解析的评测清单，以及按需添加案例附件的规则。

- [ ] **Step 1: 写入空的正式评测集**

Create `design-assurance-loop/evals/evals.json`:

```json
{
  "skill_name": "design-assurance-loop",
  "evals": []
}
```

空数组表示当前尚无完成人工确认的正式回归用例，避免把推测样例当成基线。

- [ ] **Step 2: 写入附件组织规则**

Create `design-assurance-loop/evals/files/README.md`:

```markdown
# Eval Files

这里只保存经过人工确认的评测附件。每个案例使用独立目录，并保留复现 finding 所需的最小设计、证据和裁决记录。

```text
files/<domain>/<case-name>/
├── design.md
├── evidence.md
└── decision-ledger.md
```

`<domain>` 按真实案例选择，例如 `backend`、`frontend`、`architecture` 或 `fullstack`。没有案例时不创建空分类目录。

案例必须同时定义：应发现的问题、必须引用的证据、应接受的修复，以及不得错误升级的已接受风险或非目标。评测候选经人工确认后才加入 `evals.json`。
```

- [ ] **Step 3: 解析 JSON 并检查完整目录**

Run:

```powershell
Get-Content -LiteralPath 'design-assurance-loop\evals\evals.json' -Raw | ConvertFrom-Json | Out-Null
rg --files -uu -g '!.git/**' | Sort-Object
git diff --check
```

Expected: JSON 解析成功；文件列表与设计目录一致；`git diff --check` 退出码为 0。

- [ ] **Step 4: 提交评测骨架**

```powershell
git add design-assurance-loop/evals/evals.json design-assurance-loop/evals/files/README.md
git diff --cached --check
git commit -m "test(design-assurance-loop): 添加评测骨架"
```

- [ ] **Step 5: 执行仓库最终验证**

Run:

```powershell
$skill = Get-Content -LiteralPath 'design-assurance-loop\SKILL.md' -Raw
if ($skill -notmatch '(?s)^---\r?\nname: design-assurance-loop\r?\ndescription: .+?\r?\n---') { throw 'invalid SKILL.md frontmatter' }
Get-Content -LiteralPath 'design-assurance-loop\evals\evals.json' -Raw | ConvertFrom-Json | Out-Null
git diff --check
git status --short --branch
git log --oneline --decorate -5
git remote get-url origin
```

Expected:

- frontmatter 与 JSON 检查成功。
- `git diff --check` 无输出。
- `git status` 只显示 `## main`。
- 最近提交包含设计、计划和三个初始化任务提交。
- `origin` 输出 `https://github.com/hustbeta/skills.git`。
