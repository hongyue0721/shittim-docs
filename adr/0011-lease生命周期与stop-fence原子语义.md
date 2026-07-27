# ADR-0011：Action Lease 生命周期与 Stop Fence 原子语义

- 状态：accepted
- 日期：2026-07-27
- 实现状态徽章：**designed** — 决策已定，migration 0010 与 Lease/Stop owner 未实施。当前进度见 [`docs/IMPLEMENTATION_MATRIX.md`](../docs/IMPLEMENTATION_MATRIX.md) 与 [`docs/PROGRESS.md`](../docs/PROGRESS.md)。

## 背景

`approved → leased → in_flight → completed` 是副作用 Action 的唯一合法执行链，但 `kernel-sqlite` 没有任何 Lease/Resource Lock/Stop Fence 持久化。实施前的合同复核发现，既有蓝图草稿与 `specs/IMPLEMENTATION_CONTRACTS.md`、`specs/CORE_ARCHITECTURE.md` 之间存在九处未闭合的结构冲突：

1. 蓝图要求「Action 离开 `leased` 即无 Lease 行」，但合同要求执行事务在 `in_flight → completed` 时继续 CAS `holder + generation + expires_at`（IC §6.10.6）——`in_flight` 期间删除 Lease 会让 completion 失去 fencing 事实。
2. `ActionRequestV2.lease` 内联字段与拟建的 `action_leases` 关系表构成同事实双源，蓝图未定义镜像校验。
3. acquire 的「先递增 Action generation、再插 Lease、再状态 CAS」在双方都用严格 `BEFORE` trigger 时形成先后循环；SQLite 没有 commit-time deferred trigger。
4. `action_resource_locks` 只有单列 FK，锁行 generation 可与父 Lease 不一致。
5. `max_uses` 只有字段没有消费语义，Schema 允许 `>=0` 而 SQL 草案收紧为 `>=1`，两者都没有合同依据。
6. IC §5.5 明确「重复 `stop.activate` 返回当前同一 generation」，蓝图却写「插入/递增单例」。
7. `stop.status` 响应要求完整 `Actor`，蓝图只持久化 actor id，重启后无法诚实构造响应。
8. Emergency Stop 的 Action 副作用不止「删所有 Lease」（CORE §19.1）：`leased` 与 `in_flight` 必须分流到 `cancelled` 与 `unknown_side_effect`，且每条状态边都要有独立 intent 与事件。
9. Stop 批量转换需要 transition/event ID，但受影响集合只有进入 `BEGIN IMMEDIATE` 后才能确定；合同「所有 ID 由上层分配」与此存在 TOCTOU 冲突，蓝图没有答案。

本 ADR 一次性拍板这些语义，然后才允许写 migration 0010。未投产、无已发布 consumer，因此选择直接修订合同而非兼容叠加。

## 决策

### 1. Lease 生命周期跨 `leased | in_flight`

- 持久 Lease 行与 Action 内联 `lease` 字段只在 `leased | in_flight` 状态存在；其它状态两者都必须为空。
- `approved → leased` 创建 Lease 与全部 Resource Lock；`leased → approved | cancelled | unknown_side_effect` 三条退出边原子删除 Lease 并级联释放锁。
- `in_flight → completed | failed | unknown_side_effect` 三条退出边同样原子删除 Lease 并级联释放锁：completion 是 Lease 的正常消费，failed/unknown 是 Lease 的异常终结，都必须释放锁，否则资源会被永久占用。
- `domain-task` 目前只为三条 `leased` 退出边产生 `LeaseReleaseEffect`；实现时必须为三条 `in_flight` 退出边补充同一效果，SQLite 只消费 typed effect，不在仓储内复制状态图。

### 2. `max_uses` v1 固定为 1

- v1 持久化 Lease 必须 `max_uses = 1`；repository 拒绝其它值（`contract_invalid`）。
- 不实现消费计数列，不保留没有实现语义的假通用能力；未来确需多次消费时以新合同修订引入真实计数语义。
- Schema 收紧（`action_request.v2.json` 的 `lease.max_uses` 改为 `const 1`）随 migration 0010 实现提交一同落地并过生成链门禁；本 ADR 先冻结语义，禁止只在 SQL 层暗中收紧。

### 3. Approval invalidation 的 Action 侧效果

`invalidate_and_optionally_replace` 在同一事务内按 Action 当前状态分流：

- **approved**：不改状态。`approved` 记录的是「Policy 评估通过」的历史事实；当前可执行性由执行边界每次重读 current chain head 决定，`approval_invalidated` 已是正式执行前置错误（IC §5.5 与错误目录）。replacement 经新 approved resolution 解析后，Action 无需任何状态迁移即恢复可执行。
- **leased**：同事务撤销 Lease 与锁，驱动 `leased → cancelled`。`dispatch_certainty=not_started` 由 Kernel 内部可证：`status==leased` 证明 `begin_dispatch` 从未提交，而 dispatch 只能由 Kernel 发起。Action 终结；需要时由 Task 创建新 Action。
- **in_flight**：Approval invalidation 不打断在飞执行。授权在 dispatch 时被消费，invalidation 只阻止未来执行；只有 Stop Fence 能中断 `in_flight`。

禁止为回退而新增 `approved → pending` 或 `leased → pending` 边，也禁止把上述转换伪装成 `lease_expired`。

### 4. Stop v1 的原子范围

`activate_stop_fence` 在单一 `BEGIN IMMEDIATE` 中完成且只完成：

1. 插入 `stop_fence` 单例（首版 `generation = 1`）并写唯一一条 `stop_fence.activated`（global aggregate）；
2. 对每个受影响的副作用 Action（当前 `leased | in_flight`）：
   - `leased`：dispatch 未开始可证，`leased → cancelled`，原子删除 Lease 与锁；
   - `in_flight`：Fence 时刻外部结果按定义不确定，`in_flight → unknown_side_effect`，原子删除 Lease 与锁；后续外部查询/恢复由 Recovery 编排器负责，不在本事务；
   - 每条状态边都有独立 ActionTransitionIntent 与恰好一条 `action.state_changed`。
3. S0 只读诊断 Action 不受影响（CORE §19.1：不得误转 `unknown_side_effect`）；`approved` 无 Lease 不迁移，Fence 在执行边界阻断其 acquire。

以下事项**不在**该 SQLite 事务内，也不得以降级形式伪装完成：向 Extension 发送 cancel（post-commit intent）、Task 转 `paused | waiting_user`（当前没有 Task 状态 producer）、进入 Restricted Mode、Approval invalidation。Fence 持久保持期间所有执行边界一律 `stop_fence_active`，这在 v1 无清除方法的语义下已构成完整阻断。

重复 `stop.activate`：canonical readback 返回原 `generation/reason/activated_by/activated_at`，不写新事件、不消费新 allocation、绝不 `UPDATE generation`。只有未来独立的「清除后再次激活」合同才允许 generation > 1。

### 5. Stop 批量转换的动态 allocation

受影响 Action 集合只有进入 `BEGIN IMMEDIATE` 后才能确定（事务外先查再进事务是 TOCTOU）。因此 `activate_stop_fence` 接受 caller 注入的 **transaction-bound 纯内存 typed allocation source**，在事务内为每个受影响 Action 产出 `(transition_id, event_id, correlation, dedup_key, changed_at, causation_ref)`：

- allocator 必须纯内存、无 IO、无外部调用、失败即整事务回滚；
- 产出的每个 UUID 都登记进本命令唯一的 transaction-bound UUID purpose collector，与 persisted facts 同闭集校验；
- 这是「所有 ID 由上层预分配」规则的唯一正式例外，理由是集合不可预知；replay 从已提交 Fence/事件事实读回原 allocation，禁止重新分配。

### 6. Stop Actor 事实

`stop_fence` 持久化：canonical Actor JSON、`activated_by_actor_id` 镜像投影、`activated_from_entry_point`、`origin_ref`（nullable，非空必须验证 ContentOrigin 存在）、`activated_at`、`reason`。`stop.status` 响应从 canonical 快照构造完整 `Actor`；`stop_fence.activated` payload 只投影 actor id（Schema 已定）。

### 7. acquire / release 的触发器协议

SQLite 无 deferred trigger，不变量由 DB guards + repository 协议 + 事务末 canonical readback 共同承担，禁止设计互相要求「对方先完成」的触发器：

- **acquire**：先插入 `generation = G+1`、`acquired_at_action_revision = R+1` 的 Lease（插入 trigger 校验当前 Action 仍为 `R/G/approved`）→ 同事务 Action CAS 到 `revision=R+1, execution_generation=G+1, status=leased`，内联 lease 与 Lease 行逐字段相等（Action update trigger 校验 Lease 行存在且匹配）→ readback。
- **release**：先删除 Lease（级联删锁）→ Action CAS 离开 lease-bearing 状态且 `lease=null` → readback。
- `action_leases` 增加 `UNIQUE(action_id, generation)`；`action_resource_locks` 使用复合 `FOREIGN KEY(action_id, generation) REFERENCES action_leases(action_id, generation) ON DELETE CASCADE`。
- 禁止就地 UPDATE `holder/generation`；重取必须递增 `execution_generation` 后重新插入。

### 8. 双源一致性

`actions.record_json` 的内联 lease 与 `action_leases` 行是同一事实的两个投影：`leased | in_flight` 时两者同存且 `holder/generation/expires_at/max_uses` 逐字段相等、`generation == actions.execution_generation`；其它状态两者同空。任何不一致返回 `stored_data_invalid`，绝不自动补写。

### 9. child creation Audit 的 actor/entry_point（切片 5 前置拍板）

`ChildTaskMaterializationCommand` 携带 typed execution context `{actor, entry_point}`；child creation `AuditRecordV2(task.creation_recorded)` 的 actor/entry_point 取自该 typed 事实。禁止继承父 Task 的 actor——「最初创建父任务的人」不是「本次执行 child Action 的调用者」。仅确无可归因主体时允许 `actor = null + entry_point = system_internal`（与 challenge expiry audit 的既有规则一致）。

### 10. RemoteSignatureVerifier 边界

纯密码学（preimage RFC8785 JCS 重建 + Ed25519 RFC8032 pure mode 验证，签名输入为 JCS UTF-8 bytes 本身，不先 hash）放在 `kernel-authorization` 纯库，依赖经审计的 `ed25519-dalek`，禁止自制算法。SQLite owner 通过可信端口消费验证结果；`approval.rs` 不手写密码学。验签、expire/consume、resolution append、head CAS 仍按 IC §6.10.3 在同一事务。

### 11. `event.poll` 的未来升级

要返回 EventEnvelope v2 必须新增或升级 versioned response payload、`MethodVersionBinding`、typed handler 与 Conformance；retained `EventPollResponse v1` 永不返回 v2 行。升级切片完成前不得注册任何 poll handler（IC §5.5 已有同义规则，此处重申为决策）。

## 后果

- `docs/design/lease-stop-fence-blueprint.md` 按本 ADR 重写为 v2；migration 0010 的表与触发器以修订版为准。
- `specs/IMPLEMENTATION_CONTRACTS.md` 的 §5.5（stop 两方法）、§6.5（lease 字段语义）、§6.10.1（invalidation 的 Action 侧闭包）、§6.10.6（Lease/Stop repository 表面与 allocation 例外）同步修订。
- 实施单元划分：单元 0（本 ADR 与合同闭合）→ 单元 1（migration 0010 DDL + guards）→ 单元 2（acquire + 严格读取）→ 单元 3（dispatch/release + `domain-task` in_flight 效果）→ 单元 4（Stop owner）→ 单元 5（4c Approval/Lease 撤销闭合）。
- child materialization 的 mapping 表使用 migration **0011**；交接文档中过时的 `0009_child_materialization` 表述作废（0009 已是 Action-PD heads，0010 分配给 Lease/Stop）。
- 每单元独立验收 `0 Critical / 0 High / 0 Medium` 后才提交；任何与本 ADR 冲突的实现细节以本 ADR 修订为准，不得静默偏离。
