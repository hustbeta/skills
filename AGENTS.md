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
