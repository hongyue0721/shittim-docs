# review-code — 代码审查清单

> 用于审查 Rust 代码改动（`rust/` 各 crate）。与 `review-overview.md` 总流程配合使用。

## 目标

确认改动满足：状态唯一 owner、fail closed、事件 1:1、无兜底伪造、无 panic 生产路径、时间/错误/幂等合同不被破坏。

## 逐项清单

### A. 状态与 owner

- [ ] 每个状态/事实只有一个写入口；repository 层无公开裸 CAS / 无通用状态迁移 API；
- [ ] `cas_transition_for_intent` 等机械 primitive 是否保持私有，只由 owner 编排器驱动；
- [ ] 新增写路径是否在 IC repository 闭集表内（§13.6.x）；不在则合同未修订就实现 = High。

### B. 事务与持久化（kernel-sqlite）

- [ ] 业务写统一 `with_write_transaction`（`BEGIN IMMEDIATE`）；COMMIT 前统一关系闭包；
- [ ] savepoint 失败 `ROLLBACK TO`+`RELEASE`；cleanup 失败 poison outer transaction；
- [ ] sequence/position 失败不占号；回滚无残留；
- [ ] migration：descriptor identity 三元组精确、asset 用 `include_bytes!`、无改写已发布 asset；
- [ ] 旧库拒绝 `reinitialize-required`，无自动清库/隐式升级；
- [ ] readback：canonical JCS 逐项验证；stored corruption → `stored_data_invalid`。

### C. 事件与 producer

- [ ] 成功状态边恰好一条事件；payload 从权威原始 request 回读（不二次猜测）；
- [ ] causation 精确（如 `action.state_changed` → `CausationRefV2::ActionTransition`）；
- [ ] 待写事件/append 保持 crate-private，无公开旁路。

### D. 时间与幂等

- [ ] 持久化时间戳 UTC 秒级、非零小数秒 fail closed；
- [ ] 幂等/receipt 投影不含 request_id/deadline/idempotency_key/protocol/kind/auth；
- [ ] 重放路径与合同一致（注意已知未裁决偏差：root 重放 request_id 张力——不得在未拍板前改语义）。

### E. 错误处理

- [ ] 生产路径无 panic/unwrap/expect 处理外部输入；
- [ ] 错误码来自 IC §5.7；无硬编码新 code（kernel-kcp 已知漂移：新增错误必须先入 catalog）；
- [ ] backend 错误映射封闭穷举，无遗漏 variant；
- [ ] 错误 details 不泄露内部细节；跨层不丢失定位信息（已知：invalid_scope_pattern input_kind/index 丢失）。

### F. 纯 crate 纪律（domain-* / kernel-task-creation / kernel-authorization）

- [ ] 无 IO/存储/系统时间/随机数；事实 typed 注入；
- [ ] 无依赖 SQLite/KCP/Tokio；
- [ ] 字符串比较无 trim/Unicode 改写（domain-task 已知偏差不得仿效）。

### G. 生成物与类型

- [ ] generated/ 无手改；无手写平行类型；
- [ ] 状态枚举来自 kernel-contracts；无本地平行定义。

### H. 安全与边界

- [ ] Default Allow 只在零命中且无错误时成立；PolicyError 不落入 allow；
- [ ] KernelInvariantBlock 经 typed 输入，无隐式 deny；
- [ ] Stop Fence / Delegation 未落地能力保持 fail closed，无伪造实现。

## 输出

按 `review-overview.md` §5 报告格式输出；每条 High+ 附文件:行号与影响路径。
