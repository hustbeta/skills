# Evidence Index

以下模块名、权限名和业务域已经脱敏，但调用顺序与边界保持不变。

## 现有共享安全能力

- `security.authenticate_request()` 验证客户端、归属用户启用状态、`user.call_external_api`、HMAC、`±300` 秒时间窗、客户端限流和 600 秒 Nonce。
- HMAC 使用规范 method、path、client ID、timestamp、nonce、request ID 和正文 SHA-256，并使用常量时间比较。
- Redis 负责令牌桶和 Nonce 原子登记；异常映射为固定 service-unavailable 响应，不继续业务计算。
- 固定响应函数设置 `Cache-Control: no-store` 和请求 ID。

## 当前请求读取顺序

三个新视图当前采用相同顺序：

```text
body, request_id = security.read_json_request(request, max_bytes=1024)
source_ip = security.get_trusted_source_ip(request)
security.check_ip_rate_limit(source_ip)
parse_and_authenticate(body)
```

`read_json_request()` 先校验方法、Query String、媒体类型、字符集和长度，再有界读取最多 1024 字节。它与设计中的“取得来源 IP并限流后再读取正文”顺序不同。

## 路由与业务映射

- 三个版本化 POST 路由均已挂载到独立业务视图。
- 业务视图严格限制唯一字段，拒绝重复字段、未知字段、非法日期、时区后缀和非整点时间。
- 业务视图直接调用已有日期口令算法，并把结果格式化为六位字符串。
- 旧三个 GET 路由仍指向原内部视图，未增加登录或权限校验。

## 测试源与未验证环境

- 测试源覆盖成功值、前导零、严格 JSON/日期、HMAC、安全失败、不写业务审计和旧 GET 兼容。
- 本附件没有目标测试的实际执行结果。
- 本附件没有生产 TLS、Redis 可用性、可信代理 CIDR、反向代理正文限制或客户端密钥配置的运行证据。

## 已确认范围

- 复用既有客户端和用户权限，不新增模型、权限或 migration。
- 旧内部 GET 未认证旁路与所有既有客户端自动获得新能力均已明确接受。
- 不写业务审计，不记录日期、正文摘要或口令 hash。
- 不修改既有工具 API 契约，不建设通用 API 引擎。
