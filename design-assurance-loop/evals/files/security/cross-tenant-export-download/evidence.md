# 脱敏证据

- 下载入口 `download_export_result` 接收用户提供的 `task_id`，并调用 `resolve_export_task(task_id)`。
- 已认证会话携带 `session.tenant_id`。
- 任务记录以 `tenant_id` 表示所属租户；`resolve_export_task` 仅按任务 ID 查询：`SELECT tenant_id, storage_key FROM export_tasks WHERE id = :task_id`，查询谓词不包含 `session.tenant_id`。
- `download_export_result` 从 `resolve_export_task` 的查询结果读取 `storage_key` 并返回文件，返回前未比较任务 `tenant_id` 与 `session.tenant_id`。
- 定时投递调用点 `deliver_scheduled_export` 也调用 `resolve_export_task(task_id)`。
- 未提供部署材料或网络可达性材料。
