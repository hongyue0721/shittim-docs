# review-contract — 合同一致性审查

> 审查「规范 ↔ Schema ↔ generated ↔ 实现 ↔ fixtures ↔ 文档」六层是否同事实。Shittim 的价值在跨层一致性，本合同审查是最重要的审查类型。

## 目标

确认没有任何一层与唯一事实源漂移：字段、枚举、状态机、错误码、投影形状、版本绑定。

## 逐层对账清单

### 1. spec ↔ Schema（`specs/` vs `schemas/source`）

- [ ] IC §13.x 声明的对象是否都有对应 source Schema 与 manifest 条目；
- [ ] 字段名/类型/默认值/枚举值与 spec 一致；无 spec 定义但 Schema 不存在的「孤儿字段」；
- [ ] `schema_version` 语义一致；retained vs component-native 归属正确；
- [ ] 数量硬门：manifest 总数与 retained/component-native 分布与 ADR/MATRIX 一致。

### 2. Schema ↔ generated（`schemas/` vs `kernel-contracts/src/generated/`）

- [ ] generated 与 source 完全对应（跑 `schema-tool generate` 两次无 diff 是硬证据）；
- [ ] generated 无手改痕迹；string enum `ALL` 顺序与声明一致。

### 3. generated ↔ 实现（Rust crate）

- [ ] 实现直接消费 generated 类型，无手写平行类型；
- [ ] 状态枚举、错误码、projection 形状与 generated 一致；
- [ ] 时间合同：canonicalize_rfc3339_seconds 语义被一致使用。

### 4. 实现 ↔ fixtures（`schemas/fixtures`）

- [ ] normalization/hash/projection/allocation 输出与官方 fixtures 逐字节对账；
- [ ] fixture 变更与 spec/实现同步，未出现「fixture 先行改、实现未跟」或反之。

### 5. 实现 ↔ 文档（MATRIX/PROGRESS/api）

- [ ] 文档宣称「已实现」的每个能力都能在代码中找到 owner + 调用链 + 测试；
- [ ] 文档宣称「未实现」的，代码中没有冒牌实现（如只有表/只有 Schema/只有 stub）；
- [ ] api 徽章与 MATRIX 一致；无字段级复述漂移。

### 6. 错误码与版本绑定

- [ ] IC §5.7 Error Catalog 是唯一来源：实现/测试无硬编码新 code/message/details；
- [ ] `METHOD_VERSION_BINDINGS` 与 IC §13.5 八方法集一致；`task.create` active=[2]；
- [ ] v1 create 在 production 路径被拒（`unsupported_schema_version`），无复活入口。

## 已知漂移高危点（审查时重点查）

1. Lease 状态：`acquire_lease` / `get_action_lease` 已实现 vs 文档仍写「未实现」——若发现即 High；
2. 切片 4c：11/11 High 已随 c11d3be 闭合（Approval 撤销 Lease + 真实远程验签）；再出现「4c 未完成/剩 2 项」旧表述即 High；
3. handler 覆盖夸大：文档写「八方法已支持」而 dispatcher 只有五方法注册——即 High；
4. 已知未裁决偏差（request_id 重放、时钟精度、invalid_scope_pattern details、trim）——不得在文档中写成已修复。

## 输出

按 `review-overview.md` §5 报告格式；每层对账给结论（一致 / 漂移：文件:行号 vs 文件:行号）。
