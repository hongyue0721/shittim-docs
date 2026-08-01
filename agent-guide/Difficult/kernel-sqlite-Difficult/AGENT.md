# kernel-sqlite-Difficult — 文件型 SQLite 持久化

> 难度：**Difficult**（事务/迁移/状态 owner，跨层一致性与安全核心，必须强模型 + 全量门禁 + 独立审查）。进入本文件夹即表示任务落在 `rust/crates/kernel-sqlite`。

## 目标

让持久化基座保持「唯一状态 owner + fail closed」：迁移精确、事务原子、事件只有命名 owner 能写、旧库拒绝、Lease/Stop 关系在 COMMIT 前闭合。完成标准 = migration/repository 改动通过全量门禁，且经 `review/` 独立审查。

## 开工前必读

1. 根 `AGENT.md`；`Medium/rust-workspace-Medium/AGENT.md`；
2. `specs/IMPLEMENTATION_CONTRACTS.md` §6.x（repository 闭集）、§13.6.x（切片合同）、§13.7（谓词）；`adr/0004`、`0006`、`0007`、`0008`、`0009`、`0011`；
3. **开工前必须用 `docs/IMPLEMENTATION_MATRIX.md` / `docs/PROGRESS.md` 确认当前状态**：本 crate 的完成度高频变化，本文件不复述，避免过时。

## 边界与职责

文件型 SQLite 持久化、migration、事务、repository、统一 v2 Outbox 存储与命名业务 owner 的原子编排；**不拥有** KCP、`agentd`、网络传输、Publisher loop。

**开工前核对点（勿凭记忆）**：当前已实现 owner（含 `acquire_lease` / `get_action_lease`）、未实现 owner（`begin_dispatch` / `release_or_expire_lease` / `activate_stop_fence` / `get_stop_fence` / child materializer / recovery orchestrator）、4c 剩余 2 项 High——一律以 MATRIX/PROGRESS 与源码为准。

## 硬性规则

1. **文件型唯一**：禁止 `:memory:`、URI filename、shared-cache memory mode（ADR-0004）。连接必须验证 WAL、`foreign_keys=ON`、显式非零 `busy_timeout`。
2. **migration 纪律**：descriptor v1（0003+）identity 三元组与 asset bytes 精确稳定；asset 用 `include_bytes!`；每个 pending migration 先 `BEGIN IMMEDIATE` 再重验 ledger 再执行；无 down migration、无自动备份 API。新 migration 取下一序号，禁止改写已发布 asset。
3. **旧库拒绝**：`reject_legacy_v1_business_data` → 稳定 `reinitialize-required`；禁止自动清库/隐式升级/读后补写（ADR-0009）。
4. **事务纪律**：公开业务写统一 `with_write_transaction`（`BEGIN IMMEDIATE`）；savepoint 失败必须 `ROLLBACK TO`+`RELEASE`，cleanup 失败 poison outer transaction；只有 `COMMIT` 成功才返回业务结果；sequence/position 失败不占号。
5. **事件写入无公开入口**：待写事件类型与 append 方法 crate-private，只能由命名业务 owner producer 使用；任何成功 Action 状态边恰好一条 `action.state_changed`（causation=`action_transition`）；禁止「改状态不写事件」的平行路径。
6. **CAS 是私有机械细节**：无公开裸状态迁移入口；`cas_transition_for_intent` 等只能由 owner 编排器驱动。
7. **readback 纪律**：commit 前 canonical JCS readback 逐项验证 allocation/写入事实；stored corruption → `stored_data_invalid`，不猜修。
8. **时间精度**：进入持久化的时间戳（`accepted_at` / `changed_at` / `acquired_at` / `expires_at` / `read_at` 等）必须 UTC 秒级，非零小数秒 fail closed（`contract_error`），禁止静默截断。
9. **Delegation 未落地**：非空 delegation 固定 fail closed `delegation_not_found`；禁止伪造 Delegation 查询结果。
10. **纯 crate 语义不复制**：状态合法性找 `domain-task`，Policy 找 `domain-policy`，normalize/hash 找 `kernel-task-creation`，投影找 `kernel-authorization`；repository 只编排，不重新实现。

## 已知未裁决偏差（审阅发现，非行为范本）

- **root 重放 request_id**：重放路径要求已存 provenance 的 `command_request_id` 等于新请求 `request_id`，与 IC §5.3.1「request ID 不参与幂等投影」存在张力。
- **生产时钟精度**：`kernel-kcp` 的 `SystemKernelClock` 直接返回含纳秒的系统时间，与本 crate 秒级精度门冲突（见 `Difficult/kernel-kcp-Difficult/AGENT.md`）。
- **`InvalidScopePattern` 信息丢失**：`domain-policy` 的 `input_kind`/`index` 在 `TaskCreationError → StoreError` 边界被压缩，KCP details 无法还原定位信息。

以上三条未经维护者在 `specs/` 拍板前，不得擅自改语义「修复」，也不得在新代码中仿效。

## 验收命令

```bash
export TMPDIR="${TMPDIR:-$HOME/.cache/shittim-build-tmp}"
export CARGO_TARGET_DIR="${CARGO_TARGET_DIR:-$PWD/rust/target}"
cargo test --manifest-path rust/Cargo.toml -p kernel-sqlite
```

migration/repository 闭集变更后必须：全 workspace 测试 + `scripts/check-schema.sh` + 更新 `docs/api/kernel-sqlite.md`、`docs/api/task-repository-contract.md`、MATRIX/PROGRESS，且必须过 `review/` 独立审查。

## 完成判定

- 聚焦 + 全 workspace 测试绿；clippy/fmt 绿；统一门全绿；
- `review/` 审查通过（代码 + 合同一致性 + 文档同步 + 发布门）；
- 提交/推送/镜像按 `Easy/commit-and-push-Easy/AGENT.md` 完成。
