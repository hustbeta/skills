# Evaluator-only Gold

## A-001

- severity: P0
- Finding: 已认证用户可选择其他租户的任务 ID 并取得结果文件，破坏租户隔离。
- Runtime context: 部署和网络可达性为运行态 unknown，不降低设计 P0。
- Required evidence: `download_export_result` 调用 `resolve_export_task(task_id)`；任务带有 `tenant_id`；该查询未以 `session.tenant_id` 为谓词；返回文件前未比较两者。
- Expected repair: 在 `download_export_result` 的任务解析处，将请求会话的租户身份与任务所属租户绑定校验；保持既有响应与存储结构。

## A-002

- Finding: 同一 `resolve_export_task` 函数还有 `deliver_scheduled_export` 定时投递消费者。
- disposition: requires-user-decision
- Expected repair: 先由用户决定定时投递消费者应采用的租户上下文与兼容策略；只泛泛建议检查调用方而未引用该定时消费者，不算命中。

## 非目标

- 不要求修改响应 JSON。
- 不要求修改全局权限框架或统一身份模型。
- 不要求新增加密或审计平台。
- 不要求数据库迁移。
