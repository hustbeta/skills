# Evidence Index

以下资源标识与环境信息已脱敏。

## 当前写入路径

服务函数的关键顺序为：

```text
provider.resize(resource_id, target_profile)
profile = EffectiveProfile.get(resource_id)
profile.disk = target.disk
profile.memory = target.memory
profile.cpu = target.cpu
profile.save()
```

调用方只检查 `provider.resize(...)` 是否同步返回或抛错，没有等待基础设施进入最终成功状态。

## 基础设施状态语义

- resize 提交成功表示提供方接受了异步操作，不表示迁移或规格切换已经完成。
- 提交后资源还会经历中间状态；迁移阶段失败时，提供方可以回滚到原规格。
- 业务表由应用维护，用于资源列表和详情展示；基础设施提供方维护实际资源规格及变更终态。

## 已观察失败样本

- 一次 resize 请求被提供方接受，并开始按目标规格申请资源。
- 后续迁移步骤失败，提供方把资源回滚到原规格。
- 应用业务表仍保存目标 CPU 和内存，资源详情因此与提供方终态不一致。
- guest 中读取到的 CPU/内存与原规格相符，但 guest 指标没有完整 flavor 标识。

## 现有恢复能力

- 可读取提供方最终资源记录及其规格标识。
- 业务表更新可以按资源 ID 和旧目标字段做条件更新。
- 当前拟议流程没有 pending 状态、最终成功回调、失败回滚写入或周期对账门禁。
