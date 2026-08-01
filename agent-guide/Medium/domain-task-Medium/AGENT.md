# domain-task-Medium — 纯 Task/Action 状态机

> 难度：**Medium**（纯领域逻辑，不碰持久化，但状态机语义是核心合同）。进入本文件夹即表示任务落在 `rust/crates/domain-task`。

## 目标

让 Task/Action 状态迁移合法性、evidence 门与 Lease typed effects 保持合同精确。完成标准 = 状态机改动有纯逻辑测试覆盖，且不引入 IO/持久化依赖。

## 开工前必读

1. 根 `AGENT.md`；`Medium/rust-workspace-Medium/AGENT.md`；
2. `specs/CORE_ARCHITECTURE.md`（Task/Action 状态机、副作用分类）、`specs/IMPLEMENTATION_CONTRACTS.md` §6.14 / §13.6.11、`adr/0011`（Lease typed 效果）、`adr/0008`（Event intent / causation）。

## 边界与职责

**只拥有**：
- Task / Action 状态迁移合法性（`is_*_transition_allowed`）与迁移应用（`apply_*_transition`）；
- `revision` / `plan_version` 算术与 evidence 门；
- Policy 评估结果在 pending Action 上的投影效果（`apply_policy_evaluation_outcome`）；
- Lease 生命周期 typed 效果闭集（`LeaseReleaseReason` / `LeaseReleaseEffect` / `DispatchCertainty`）；
- Task/Action Event intent 投影（只描述意图，不分配 Event 事实）；
- Recovery candidate 合法性（`validate_recovery_candidate_kind` / `validate_retry_original_candidate`）与补偿 Action 草稿校验。

**不拥有**：持久化、SQLite、Event envelope/position 分配、ID/时间分配、Policy matcher（在 `domain-policy`）、KCP、UI、Extension。

## 硬性规则

1. 状态枚举一律来自 `kernel-contracts` generated types，禁止本地平行定义。
2. 所有迁移先经 `is_*_transition_allowed` 判定再 `apply_*`；非法边返回结构化 `DomainTaskError`，fail closed。
3. evidence 门缺失/为空是硬错误，不是可跳过的警告。
4. 需要 Lease/Lock 效果的边只产出 typed effect；消费由 `kernel-sqlite` owner 完成，本 crate 不得伪造「已释放」事实。
5. 业务字符串比较遵循合同精确语义：**不做 trim、不做 Unicode 改写、按完整内容匹配**。新增比较逻辑必须遵守此条。
6. 纯函数：无 IO、无存储、无系统时间、无随机数。

## 已知未裁决偏差（审阅发现，非行为范本）

- `task.rs` 成功判据匹配对 criterion content 做了 `trim()`，与合同「精确多重集合、不改写」语义冲突。未经维护者在 `specs/` 拍板前：**不得**擅自「修复」（可能破坏既有测试基线），也**不得**在新代码中仿效该写法。

## 验收命令

```bash
export TMPDIR="${TMPDIR:-$HOME/.cache/shittim-build-tmp}"
export CARGO_TARGET_DIR="${CARGO_TARGET_DIR:-$PWD/rust/target}"
cargo test --manifest-path rust/Cargo.toml -p domain-task
```

改动状态机边/枚举后同步跑全 workspace 测试与 `scripts/check-schema.sh`（见 `Medium/rust-workspace-Medium/AGENT.md`），并检查 `docs/api/domain-task.md` 徽章与 MATRIX/PROGRESS 是否仍成立。

## 完成判定

- 聚焦测试（含 `tests/action_matrix.rs`）绿；全 workspace 绿；门禁全绿；
- 文档同步完成；提交/推送/镜像按 `Easy/commit-and-push-Easy/AGENT.md`。
