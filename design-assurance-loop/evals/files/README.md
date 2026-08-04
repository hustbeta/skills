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
