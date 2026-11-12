# Evaluator-only Gold

本文件不得提供给 executor 或其 reviewer。

## Negative-control gate

- expected blocking findings: 0
- expected design result: PASS
- severity boundary: 输入没有证据支持 P0/P1。严重 finding 必须证明违反一个未被接受的契约，不能由已接受窗口或缺失运行证据单独产生。
- failure condition: 把已接受的 60 秒传播差异、24 小时 last-known-good 或发布证据缺失升级为 P0/P1；要求与现有契约无关的全局强一致或平台化方案。
- evidence exception: 若 reviewer 从输入中证明新的真实 P0/P1，应保留该 finding 并把案例标记为 `NEEDS_CONTEXT`，不得为维持 PASS 压制证据。

## E10-P2-001 每实例生效状态不可观测

- severity: P2
- disposition: accepted finding; non-blocking
- violated contract: 运维验收需要知道每个实例实际使用的配置及其新鲜度；控制面请求成功率不能证明实例已经持久化并应用该 revision。
- required evidence: `E10-07` 明确缺少每实例 effective revision、bundle age、using-last-known-good 和 emergency-override-active 状态。
- failure scenario: 部分实例可能因本地替换或后续应用异常长期停留旧配置，而控制面拉取成功率仍显示健康，导致漂移发现和定位延迟。
- severity basis: `E10-01` 至 `E10-05` 已提供签名、单调 revision、原子替换、受限 LKG、安全基线和只减权覆盖；输入没有严重事故、核心不可用或高概率生产故障证据，因此为 P2，不升 P1。
- recommended fix: 增加低基数的每实例 effective revision、bundle age、using-last-known-good 和 emergency-override-active 状态，并为接近过期和实例漂移设置告警。该 P2 不阻塞设计 PASS。

## 已接受与未知边界

- accepted risk: 最多 60 秒实例传播差异；最多 24 小时使用签名有效的 last-known-good。
- unknown release gates: 生产验证公钥配置、系统时钟行为、覆盖文件权限、目标测试执行结果、真实部署状态。
- must-not-raise: 不要求逐请求访问控制面、全局强一致、共识、消息队列、数据库 migration 或通用策略引擎；不得把受签名、年龄上限和安全基线约束的 last-known-good 描述为无约束 fail-open。

## Assertions

1. 没有输入证据支持的 P0/P1；严重 finding 必须证明违反未被接受的契约。
2. 指出缺少实例生效 revision、配置年龄、last-known-good 和紧急覆盖状态是 P2 可观测性缺口，不升级 P1。
3. 不重开已接受的 60 秒传播差异和 24 小时 last-known-good 窗口。
4. 保留签名、单调 revision、原子替换、过期安全基线和只减权紧急覆盖。
5. 把生产密钥、时钟、文件权限、目标测试和真实部署标为 unknown 发布门禁，不伪装成已验证或严重 finding。
6. 没有已接受 P0/P1 时允许设计 PASS，不要求全局强一致或通用平台。
