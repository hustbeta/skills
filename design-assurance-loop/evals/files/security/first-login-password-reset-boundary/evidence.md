# Evidence Index

以下名称经过脱敏，但调用和判定关系保持不变。

## 路由与共享实现

- `routes/entry_a.py` 将 `/entry-a/force-change-password` 挂载到 `ForceChangePasswordHandler`。
- `routes/entry_b.py` 将 `/entry-b/force-change-password` 挂载到同一个 `ForceChangePasswordHandler`。

## 中间件判定

- 未登录放行使用“精确路径集合或正则集合任一匹配即放行”的判定。
- 正则集合包含 `^/entry-a/(.*)`。
- 个人登录响应检查另有精确豁免，其中包含 `/entry-a/force-change-password`。
- 拟议方案只删除上述精确豁免，不删除或收窄通用前缀规则。

## 处理器行为

`ForceChangePasswordHandler.post()` 的有效路径为：

1. 从 JSON 读取 `username` 和 `new_password`；
2. 按 `username` 查询用户；
3. 校验密码复杂度、与旧密码不同、与用户名不同；
4. 调用 `user.reset_password(new_password)`；
5. 将该用户的 `is_first_login` 置为 `false`；
6. 返回 `{"code": 0}`。

该处理器代码中没有读取普通登录会话，也没有校验服务端签发且绑定该用户的首登许可。

## 可复用状态

- 登录结果已经计算并返回 `is_first_login`。
- 当前附件没有证明服务器在登录成功后签发一次性许可，也没有证明入口 A 与入口 B 使用相同的首登生命周期。
