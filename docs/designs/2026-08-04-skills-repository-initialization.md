# Skills 仓库初始化设计

## 背景

本仓库用于维护 `design-assurance-loop` 及后续个人 Skill。远端仓库为空，默认分支为 `main`。

## 目标

- 每个 Skill 可独立阅读、复制和安装。
- 仓库级文档与 Skill 内容分离。
- 首个 Skill 保留评审规则和回归样例的扩展位置。
- 初始结构保持最小，不预建尚未使用的工具链。

## 非目标

- 本次不引入自定义 agent、自动化脚本或 CI。
- 本次不建立前端、后端、架构等空分类目录。
- 本次不选择许可证，也不推送远端。
- 本次只初始化 `design-assurance-loop`，不实现未来 Skill。

## 目录结构

```text
skills/
├── README.md
├── AGENTS.md
├── .gitignore
├── docs/
│   └── designs/
└── design-assurance-loop/
    ├── SKILL.md
    ├── references/
    │   └── review-rubric.md
    └── evals/
        ├── evals.json
        └── files/README.md
```

仓库本身已命名为 `skills`，因此 Skill 直接位于根目录，避免形成 `skills/skills/<skill-name>`。`docs/designs/` 使用通用名称，不与特定工作流或插件绑定。

## 文件职责

- `README.md`：说明仓库用途、Skill 清单和使用方式。
- `AGENTS.md`：约束仓库内 Skill 的组织、变更范围和验证要求。
- `.gitignore`：只忽略本仓库可能产生的临时文件和本地评测产物。
- `design-assurance-loop/SKILL.md`：定义设计方案的独立评审、裁决、修订和收敛门禁。
- `references/review-rubric.md`：保存 reviewer 共用的 finding 格式、严重级别和证据规则。
- `evals/evals.json`：保存经过人工确认的回归用例清单。
- `evals/files/README.md`：说明用例附件按实际领域按需建目录，不提交空分类。

## 扩展规则

新增 Skill 时，在仓库根目录新增 `<skill-name>/`。每个 Skill 必须包含 `SKILL.md`；仅在确有内容时增加 `scripts/`、`references/`、`assets/` 或 `evals/`。仓库级工具只有被两个以上 Skill 复用时才提取到根目录。

## 验证标准

- Git 分支为 `main`，`origin` 指向 `https://github.com/hustbeta/skills.git`。
- 目录与本设计一致，没有无内容的预留目录。
- `SKILL.md` 包含有效的 `name` 和 `description` frontmatter。
- `evals.json` 可被 JSON 解析器读取。
- Git 工作区只包含本次初始化相关文件。
