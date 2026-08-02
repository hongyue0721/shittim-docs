# task.create 幂等重放偏差 — 决策输入

> 状态：**决策输入（未拍板）**。本文只整理事实、冲突与选项，不替代 `specs/` 合同，不构成 ADR。由维护者拍板后在 `specs/IMPLEMENTATION_CONTRACTS.md`（及必要的 ADR）收敛，届时删除本文件或降级为链接。

## 1. 背景

`agent-guide/Medium/kernel-task-creation-Medium/AGENT.md` 记录了审阅发现的已知未裁决偏差：

> kernel-sqlite 的 root 重放当前要求已存 provenance 的 `command_request_id` 等于新请求的 `request_id`，与「request ID 不参与幂等投影」的合同存在张力。不得为了迁就 repository 现状而在本 crate 的投影里重新引入 request ID；等维护者在 `specs/` 拍板后统一收敛。

`kernel-task-creation` 侧（投影）已确认无违反：`RootTaskCreateIdempotencyProjectionV1` 只包含 actor / command_type / context / entry_point / expected_revision:NullOnly / payload / schema_version / task_id:NullOnly，无任何 envelope 元数据（request ID、deadline、idempotency key、protocol version、message kind、auth 均不参与）。本文件核对的是 repository 侧现状与合同的关系。

## 2. 合同事实（唯一事实源）

- `specs/IMPLEMENTATION_CONTRACTS.md` §5.3.1：幂等/receipt 投影排除 request ID、deadline、idempotency key、protocol version、message kind、auth。合法改变 `request_id` 时两 hash 均 `same`（业务幂等等价）。
- §5.3.1 fixture 权威矩阵：`task_create_normalized_hash.v2.json` tamper case 1/2/3 中，改变 `deadline` / `request_id` / `idempotency_key` → receipt 与 idempotency 两 hash 均 `same`。
- §5.5 重放语义：commit 成功但响应丢失时，以同一 Action ID/generation/idempotency 重放，读取并 canonical verify 已存在 child，返回原 child（root 同理：返回已存在任务，不创建新事实）。

## 3. repository 现状（实测证据）

`rust/crates/kernel-sqlite/src/root_task_create_v2.rs`：

- **幂等查找**（约 303 行起）：以 `idempotency_key`（scoped by actor ID + entry point + command type）查 `root_task_create_idempotency_v2`。
- **命中校验**：存库投影必须 canonical 且 hash 自洽；`actor_id` / `entry` / `command_type` 匹配；`stored_hash == prepared.projection.idempotency.sha256` 且 `projection_json == expected_projection`，否则 `IdempotencyConflict`（"idempotency key was used for different task facts"）。**此段与合同一致**：业务幂等以投影为准，request ID 不参与。
- **重放 bundle 校验**（约 363–372 行）：`stored_provenance_id != provenance_id || command_request_id != command.envelope.request_id` → `stored_invalid()`（"Lease/Action projection is not closed" 同款 `StoredDataInvalid`）。

## 4. 冲突场景

同一业务 payload（相同幂等投影 hash）+ 相同 `idempotency_key` + **不同 `request_id`** 重放：

1. 幂等表命中，投影 hash 与精确投影均一致 → 通过业务幂等判定；
2. 进入重放 bundle 校验：`command_request_id`（首次执行时存入 provenance）≠ 新请求 `request_id` → 返回 `StoredDataInvalid`。

后果：
- 客户端在响应丢失后**换新 request_id 重试同一业务**，得到的是内部错误（`stored_data_invalid`），而不是合同承诺的"返回原任务（Replayed）"或明确的 `IdempotencyConflict`；
- `stored_data_invalid` 在系统语义里表示"已提交数据损坏"（见 Safe Recovery 语境），用它表达"重放关联不匹配"会误导恢复路径；
- 若客户端同时换掉 `idempotency_key`，则幂等表查不到行，直接走创建路径生成**第二个任务**——这与"一个业务事实只存在一次"的意图冲突（但 `idempotency_key` 本身是 caller 选择的关联键，换键后无法关联是合同允许的边界）。

## 5. 选项

### A. 保持现状（重放必须同 request_id）

- 语义：`request_id` 是"同一次命令执行"的关联键，重放必须是同一次执行的 retry；投影决定业务等价，repository 的 request_id 检查决定"是否同一次执行"。
- 优点：重放关联最强，恢复语义清晰（同一次执行的 retry 才能拿到原任务）。
- 缺点：与 §5.3.1 的表述张力仍在（需在合同中明确"投影不参与、但 repository 重放关联校验可参与"）；`stored_invalid` 错误码语义不当，应改为显式错误。

### B. 重放不检查 request_id

- 语义：业务幂等完全由投影 hash + idempotency_key 决定，request_id 只做日志记录。
- 优点：与"request ID 不参与幂等"字面完全一致；换 request_id 重试稳定返回原任务。
- 缺点：失去"同一次执行"强关联；provenance 的 `command_request_id` 字段降级为仅记录；需确认是否有审计/对账依赖它。

### C. 合同澄清 + 错误语义细化（倾向建议）

- 在 IC §5.3.1 / §5.5 明确两段式语义：**幂等投影是业务等价唯一判据（request ID 等 envelope 字段不参与）**；**repository 可额外要求同 request_id 才允许 Replayed**，且不匹配时必须返回**显式错误**（复用 `IdempotencyConflict` 或新增 typed code，例如 `replay_request_mismatch`），不得使用 `stored_data_invalid`。
- 优点：合同与实现各归其位；错误可诊断；恢复路径不会被误导。
- 缺点：需要一次 specs 修订 + repository 错误映射小改 + 对应测试/fixture 更新。

### D. 换 request_id 视为幂等成功（返回原任务）

- 语义：投影一致 + idempotency_key 一致即视为同一次业务，无论 request_id，返回原任务（Created/Replayed 语义按 §5.5）。
- 优点：对客户端最宽容，天然满足"不创建第二个任务"。
- 缺点：`request_id` 关联审计价值下降；与"command_request{id}" causation 的对应关系弱化（首次事件的 causation 指向首次 request_id，重放返回时客户端拿到的 causation 与自己的 request_id 不同，需在响应中说明）。

## 6. 建议

倾向 **C**：合同明确"投影是业务等价唯一判据，repository 重放关联校验可用 request_id 但必须显式错误"，同时把 `stored_invalid` 的错误映射修正为 typed code。理由：不改投影（`kernel-task-creation` 无违反）、不削弱重放关联、错误可诊断。

最终取舍（A/B/C/D）由维护者拍板。

## 7. 拍板后的收敛动作（供参考）

1. 修订 `specs/IMPLEMENTATION_CONTRACTS.md` §5.3.1 / §5.5 相应段落；
2. 若选 C：`kernel-sqlite` 重放校验错误映射为 typed code + 测试更新；
3. 若选 B/D：删除/降级 provenance `command_request_id` 校验，更新重放测试；
4. 更新 `docs/PROGRESS.md` / `docs/IMPLEMENTATION_MATRIX.md` 与相关 API 文档；
5. 删除本决策输入文件或改为 ADR 链接。

## Critical Files

- `specs/IMPLEMENTATION_CONTRACTS.md`（§5.3.1、§5.5）
- `rust/crates/kernel-sqlite/src/root_task_create_v2.rs`（约 303–372 行）
- `rust/crates/kernel-task-creation/src/normalization.rs`（投影构造，无违反）
- `schemas/fixtures/kcp/task_create_normalized_hash.v2.json`（tamper case 1/2/3）
