# 有界清单准入

新增外部入口 `POST /v1/bounded-manifests` 由 `handleBoundedManifest` 接收 JSON 清单；`readManifestJSONBody` 最多读取 8192 字节并解析 JSON。解析成功后，`admitManifestTenant` 检查租户准入，随后 `acquireManifestConcurrencySlot` 取得应用层并发配额；任一拒绝均返回固定响应。

既有集成密钥自动获得新清单类型授权。既有内部入口 `POST /internal/manifest-imports` 及 `handleInternalManifestImport` 保持不变，不采用上述新处理链。明确接受的风险是：不增加按类型授权或 migration，不记录业务正文，既有内部入口保持不变。
