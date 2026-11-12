# 执行池标识在线迁移设计

## 背景

任务调度服务当前在 `scheduled_tasks.execution_pool_name` 中保存执行池名称。名称可以由外部 API 提交，也可以由服务内部更新。执行池目录已经为每个池分配稳定 ID，计划把任务引用迁移到 `scheduled_tasks.execution_pool_id` 外键。

## 目标

- 零停机完成滚动发布。
- 保持外部 API 的请求和响应字段不变。
- 新调度器最终只按稳定 ID 选择执行池。
- 回滚窗口结束后删除旧名称列和兼容逻辑。

## 非目标

- 不建设通用数据库迁移平台。
- 不重构任务模型或执行池目录。
- 不永久兼容任意历史服务版本。

## 拟议数据模型

先新增可空列和索引：

```sql
ALTER TABLE scheduled_tasks ADD COLUMN execution_pool_id BIGINT NULL;
CREATE INDEX idx_scheduled_tasks_execution_pool_id
    ON scheduled_tasks(execution_pool_id);
```

外键和非空约束在数据迁移及读路径切换后启用。

## 拟议发布顺序

1. Expand：新增可空 `execution_pool_id` 和索引。
2. Deploy writer：滚动部署新版本。新版本继续接收名称字段，解析目录后同时保存名称和 ID；滚动窗口内旧版本仍按原逻辑运行。
3. Backfill：启动一次全量任务，把当前名称解析为 ID 并写入空 ID 记录；任务完成后记录 `backfill_completed=true`。
4. Cutover reader：在 `backfill_completed=true` 后切换新调度器，使其只读取 `execution_pool_id`。
5. Validate：启用非空约束和外键约束。
6. Contract：回滚窗口结束后删除 `execution_pool_name` 及兼容代码。

## 读取与回滚

新调度器读取到空 `execution_pool_id` 时使用平台默认执行池，避免单条异常记录阻塞扫描。若新调度器出现异常，发布系统回滚到仍读取名称列的旧版本；数据库新增列在回滚窗口内保留。

## 验收建议

- 记录全量回填扫描数、成功数、无法映射数和约束验证结果。
- 在预发布环境执行旧版、新版和混合版本的创建、更新与调度测试。
- 对无法唯一映射的历史名称输出清单，由上线流程处理。
- 读切换后抽样比较名称与 ID 指向的执行池。
