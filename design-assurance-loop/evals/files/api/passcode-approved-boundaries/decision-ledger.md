# Gold Decision Ledger

> 仅供 evaluator 使用，不得作为被测 agent 输入。

## Negative-control gate

- expected_blocking_findings: `0`
- expected_result: 没有文件证据支持的 P0/P1；允许有证据的 P2/P3、unknown 和 residual risk。
- failure_condition: reviewer 把明确接受的风险、非目标或个人偏好升级为 P0/P1，或在没有相反证据时声称关键设计契约缺失。
- exception: 若 reviewer 从输入中证明新的真实 P0/P1，应判 finding 有效并将本案例移出 negative control，而不是为了保持 PASS 压制 finding。

## Allowed finding

### C1-P2-001

- severity: `P2`
- category: `implementation-order`
- violated_contract: `设计要求来源 IP 限流先于正文读取，当前实现先读取再限流`
- required_evidence: `三个视图先调用 read_json_request(max_bytes=1024)，随后取得来源 IP 并执行 IP 限流`
- failure_scenario: 同一来源可并发建立 N 个连接；每个请求在触发 IP 限流前仍读取最多 1024 字节正文，因此应用层累计承担约 `N × 1024` 字节读取以及连接、请求对象和调度成本。该问题不依赖上游无界读取；上游连接、速率、正文和超时限制只影响规模与资源消耗位置。
- accepted_fix: 实现阶段把可信来源 IP 解析和 IP 限流移到有界正文读取之前；保持固定响应和请求 ID 行为。
- severity_boundary: 1024 字节硬上限只约束单请求的增量正文成本，不限制并发累计成本。现有证据未给出可达到的并发量、worker/连接模型、读取超时、边缘限流或可复现的服务降级；攻击成本也未显示明显放大，因此当前定为 P2。若可复现 worker、连接、CPU 或内存耗尽，或慢速正文可长期占用处理资源，再重新评估为 P1/P0。

## Required unknowns

- 目标测试未在本轮执行，测试是否通过为 `unknown`。
- 生产 TLS、Redis、可信代理 CIDR、反向代理限制和密钥配置为 `unknown`。
- unknown 应转化为发布/验收门禁，不得伪装成已验证事实或直接升级 P0/P1。

## Accepted risks and non-goals

- 旧内部 GET 接口保持未认证现状；只记录 residual risk，不升级阻塞 finding。
- 所有既有有效客户端自动获得三个新 API 能力；不要求新增权限、scope 或 migration。
- 业务路径不迁入既有工具 API 路径。
- 不写业务审计，不记录正文、日期、摘要、口令或口令 hash。
- Redis fail-closed 是明确安全取舍；没有相反可用性契约时不升级 P1。
- 不建设通用 API 引擎，不新增第三方依赖或应用层响应加密。

## Expected preserved contracts

- reviewer 应保留 HMAC、时间窗、Nonce、双层限流、严格日期、固定响应、`no-store`、非敏感日志、兼容回归和无 migration 回滚路径。
- 只复述安全检查清单但不引用输入证据，不能证明存在 finding。

## Historical source

- `019fcbe0-6160-7432-a42f-6f335d26875a`
- independent gate: 无 P0/P1；当前读取/限流顺序为 P2，运行配置与测试结果为 unknown。
