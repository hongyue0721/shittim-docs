# kernel-task-creation-Medium — 纯 Task 创建规范化与 allocation 校验

> 难度：**Medium**（纯逻辑，但幂等/哈希/alloc 是跨层对账事实）。进入本文件夹即表示任务落在 `rust/crates/kernel-task-creation`。

## 目标

让 root/child 创建规范化、receipt/idempotency 哈希与 allocation 校验保持合同精确。完成标准 = 纯函数测试 + official fixtures 对账通过，无 IO 依赖。

## 开工前必读

1. 根 `AGENT.md`；`Medium/rust-workspace-Medium/AGENT.md`；
2. `specs/IMPLEMENTATION_CONTRACTS.md` §5.3 / §5.3.1（投影/JCS preimage/规范化）、`adr/0006`（child 权威与 task.create v2 迁移）。

## 边界与职责

**只拥有**：
- root / child proposal 规范化（`normalize_root_task_create` / `normalize_child_task_proposal`）；
- receipt / idempotency 的 canonical JCS preimage 与哈希投影；
- allocation 校验：`validate_root_task_create_allocation`（root 七 UUID + opaque correlation/dedup）、`validate_child_task_materialization_allocation`（child 十 UUID）；
- 外部关系 UUID 快照的强类型入口（`RootTaskCreateExternalUuidRefsV1` 等），强迫调用方显式解析 wire 文本。

**不拥有**：ID 分配、repository 读取、事务、SQLite/KCP 实现细节。

## 硬性规则

1. **幂等/receipt 投影排除 envelope 字段**：request ID、deadline、idempotency key、protocol version、message kind、auth 均不参与业务幂等等价（唯一事实源 IC §5.3.1）。修改投影字段集是合同变更。
2. allocation 校验必须：内部 UUID 互异、与 caller 外部 UUID 快照不碰撞、opaque ID 非空；**拒绝自由 UUID bag**，每个关系槽必须显式构造。
3. 纯函数：无 IO、无存储、无时间/随机。
4. root-only 与 child 经 Action 的边界由本 crate 输入形状保证（root 输入 `parent_task_id`/`task_id` 固定 null 语义），不得提供绕过入口。

## 已知未裁决偏差（审阅发现，非行为范本）

- `kernel-sqlite` 的 root 重放当前要求已存 provenance 的 `command_request_id` 等于新请求的 `request_id`，与「request ID 不参与幂等投影」的合同存在张力。**不得**为了迁就 repository 现状而在本 crate 的投影里重新引入 request ID；等维护者在 `specs/` 拍板后统一收敛。

## 测试资产

- 官方 fixtures：`schemas/fixtures/task/child_task_proposal_normalized_hash.v1.json`、`schemas/fixtures/task/task_creation_allocations.v1.json`（及 root 对应 fixtures）。规范化/哈希输出必须与 fixtures 对账。

## 验收命令

```bash
export TMPDIR="${TMPDIR:-$HOME/.cache/shittim-build-tmp}"
export CARGO_TARGET_DIR="${CARGO_TARGET_DIR:-$PWD/rust/target}"
cargo test --manifest-path rust/Cargo.toml -p kernel-task-creation
```

## 完成判定

- 聚焦测试 + fixtures 对账绿；全 workspace 门禁绿；
- 文档同步完成；提交/推送/镜像按 `Easy/commit-and-push-Easy/AGENT.md`。
