# 脱敏核对记录

新增外部入口 `POST /v1/bounded-manifests` 的 `handleBoundedManifest` 调用 `readManifestJSONBody` 有界读取并解析正文；成功后调用 `admitManifestTenant` 检查租户准入；准入后调用 `acquireManifestConcurrencySlot` 取得应用层并发配额。既有内部入口 `POST /internal/manifest-imports` 的 `handleInternalManifestImport` 保持原处理链。

未验证：代理限制、读取超时、worker/连接模型、执行器容量，以及目标测试结果。上述项目待发布或验收时逐项核对。
