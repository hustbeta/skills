# Eval Files

这里只保存经过人工确认的评测附件。每个案例使用独立目录，并保留复现 finding 所需的最小设计、证据和裁决记录。

```text
files/<domain>/<case-name>/
├── design.md
├── evidence.md
└── decision-ledger.md
```

`<domain>` 按真实案例选择，例如 `backend`、`frontend`、`architecture` 或 `fullstack`。没有案例时不创建空分类目录。

案例必须同时定义：应发现的问题、必须引用的证据、应接受的修复，以及不得错误升级的已接受风险或非目标。`evals/catalog.md` 保存评测分层、候选和 Gold 状态；它与 decision ledger 同属 evaluator-only，不得传给被测 agent。

## 注册与生命周期

- candidate 经用户确认 Gold 后，才创建正式附件并加入 `evals.json`。
- `transfer-heldout` 和 `negative-control` 使用 blind prompt；当前优化周期内不得根据其结果修改 Skill。
- held-out 一旦用于调优，转为 `historical-regression`，并补充新的 held-out。
- historical regression 可以使用 directed prompt，也可以参与 Skill 调优，但不能用于证明跨项目通用性。

## 输入与 Gold 隔离

- `design.md`、`evidence.md` 是被测 agent 输入；`evals.json` 的 `files` 只能列出这两类文件。
- `decision-ledger.md` 是 evaluator-only gold，不得传给被测 agent。
- benchmark 必须把 `SKILL.md`、所需 `references/` 和 `files` 列出的输入复制到干净临时目录；不要让 executor 访问原仓库中的 catalog、`evals.json`、expectations 或 decision ledger。
- 历史案例使用脱敏最小重构，不复制完整内部源码、凭据、人员信息或业务正文。
