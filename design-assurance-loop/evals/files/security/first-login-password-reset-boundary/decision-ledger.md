# Gold Decision Ledger

> 仅供 evaluator 使用，不得作为被测 agent 输入。

## Accepted findings

### H2-001

- severity: `P0`
- category: `authorization`
- violated_contract: `敏感密码重置必须有与目标用户及首登流程绑定的正向授权证明`
- required_evidence: `通用前缀 ^/entry-a/(.*) 仍会放行；handler 仅凭正文 username 查找用户并重置密码`
- failure_scenario: 未持有目标用户首登授权的调用方提交用户名和新密码，通用前缀旁路仍使请求到达处理器并改变该用户凭据。
- accepted_fix: 在登录成功且确认需要强制改密后签发服务端短时、一次性、绑定用户和入口语义的许可；handler 成功消费后清除许可，同时保持现有 JSON。

### H2-002

- severity: `P1`
- category: `impact-boundary`
- disposition: `requires-user-decision`
- violated_contract: `修改共享敏感处理器前必须识别全部实际入口并裁决各入口的授权语义`
- required_evidence: `entry_a.py 与 entry_b.py 都挂载 ForceChangePasswordHandler`
- failure_scenario: 只按入口 A 的流程修改共享 handler，可能破坏入口 B 的既有流程；只在入口 A 外围加门禁，又可能让入口 B 继续暴露同类敏感操作。
- accepted_fix: 先记录两个入口的存在性和各自授权信号；由用户决定共享同一许可、按入口分离门禁或明确接受入口 B 的残余风险。

## Must-not-raise

- 不要求改变 `username`、`new_password` 或响应 JSON。
- 不要求把整个入口 A 改造成普通 `request.user` 会话认证。
- 不因发现入口 B 就擅自决定必须同步改变其产品行为。
- 附件已证明未授权调用方可选择目标用户并改变其凭据，因此 H2-001 按 P0；部署可达性和现网暴露仍记为运行态 `unknown`，但不降低设计 finding 的严重度。
- 只泛泛建议“检查其它调用方”但未引用第二个实际路由，不算命中 H2-002。

## Historical source

- `019f461c-4ad6-71b1-a6d3-5bf84227bb5a`
- 输入已将具体项目路由替换为入口 A/B；Gold 评分看“共享实现的全部入口/消费者”能力，不要求命中特定产品词。
