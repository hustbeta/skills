# 交互式导出下载设计

已认证用户可通过 `download_export_result` 按导出任务 ID 下载任务结果文件。

- `download_export_result` 接收请求中的 `task_id`。
- 下载入口调用现有全局任务解析函数 `resolve_export_task(task_id)`。
- 解析结果按既有响应与存储结构返回文件。

范围仅涵盖交互式下载，不包含定时投递或其他触发方式。
