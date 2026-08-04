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
