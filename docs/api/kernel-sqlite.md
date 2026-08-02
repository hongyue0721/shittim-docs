# kernel-sqlite 内部 Rust API

`rust/crates/kernel-sqlite`是文件型SQLite持久化crate，不是KCP或外部SDK API。当前实现 migration 0001–0011；其中 0010 已建立 Action Lease、Resource Lock 与 Stop Fence 的持久化基座、关系守卫及统一 COMMIT 前闭包，Lease 全链 owner（`acquire_lease` / `get_action_lease` / `begin_dispatch` / `release_or_expire_lease` / `activate_stop_fence` / `get_stop_fence`）与 Approval 撤销 Lease、真实远程验签均已落地；0011 建立 child materializer 持久化基座（`verification_results` / `child_materializations`，mapping 五钉闭包 FK）。

## 打开与migration

```rust
let config = SqliteConfig::new(Duration::from_secs(5))?;
let store = SqliteStore::open("/var/lib/shittim/kernel.sqlite3", config)?;
```

- 只接受普通文件路径；拒绝空路径、`:memory:`及`file:` URI。
- 每个连接验证`foreign_keys=ON`、显式非零`busy_timeout`与WAL。
- 并发首次打开同一新文件：`SqliteStore::open` 对 `SqliteBusy` 在 3×`busy_timeout` 窗口内整体重试（经 `config::retry_on_busy` 辅助）。WAL 引导、外键与迁移均可幂等重跑，重试不会重复迁移；窗口耗尽仍以 `SqliteBusy` 在源头暴露，非 busy 错误从不重试。这是为了覆盖 SQLite busy handler 不处理的锁升级路径，以及满负载下单次迁移单元超过一个 busy_timeout 窗口的偶发失败。
- migration definitions分为`LegacySql`（0001/0002）与`DescriptorV1`（0003+）；所有SQL使用`include_bytes!`原始asset bytes。
- 0001/0002 ledger checksum保持SQL bytes SHA-256，descriptor列必须null。
- 0003 exact identity：version `3`、name `versioned_event_outbox`、唯一asset `rust/crates/kernel-sqlite/migrations/0003_versioned_event_outbox.sql`、transform三元组`shittim.kernel-sqlite.outbox-v1-to-versioned-v1` / `1` / `kernel_sqlite::migration::outbox_v1_to_versioned_v1`；phase set=`ledger_upgrade|replacement_schema|table_swap`。**transform 不迁移 v1 业务数据**：非空 pre-0003 Outbox 直接 `reinitialize-required`；空表仅做 shape 升级。
- 0004 exact identity：version `4`、name `root_task_create_v2`、唯一asset `rust/crates/kernel-sqlite/migrations/0004_root_task_create_v2.sql`、transform三元组`shittim.kernel-sqlite.ddl-only-v1` / `1` / `kernel_sqlite::migration::root_task_create_v2_ddl_only_v1`；phase set=`schema`（纯DDL，无row transform）。
- 0005 exact identity：version `5`、name `drop_v1_business_tables`、唯一asset `rust/crates/kernel-sqlite/migrations/0005_drop_v1_business_tables.sql`、transform三元组`shittim.kernel-sqlite.ddl-only-v1` / `1` / `kernel_sqlite::migration::drop_v1_business_tables_ddl_only_v1`；phase set=`schema`。在空表前提下 drop `content_origins`(+parent_refs)、`audit_records`、`task_create_idempotency`；非空拒绝。
- 0006 exact identity：version `6`、name `action_and_transition`、唯一asset `rust/crates/kernel-sqlite/migrations/0006_action_and_transition.sql`、transform三元组`shittim.kernel-sqlite.ddl-only-v1` / `1` / `kernel_sqlite::migration::action_and_transition_ddl_only_v1`；phase set=`schema`。创建 `actions` 与 `action_transition_intents`（canonical `record_json` + 投影列 + 双唯一键 + `committed_event_id`）。
- 0007 exact identity：version `7`、name `policy_and_permission_decision`、唯一asset `rust/crates/kernel-sqlite/migrations/0007_policy_and_permission_decision.sql`、transform三元组`shittim.kernel-sqlite.ddl-only-v1` / `1` / `kernel_sqlite::migration::policy_and_permission_decision_ddl_only_v1`；phase set=`schema`。创建 `policy_set_metadata`（bootstrap revision 0）、`policy_rules`、`permission_decisions`。
- 0010 exact identity：version `10`、name `action_lease_stop_fence`、唯一 asset `rust/crates/kernel-sqlite/migrations/0010_action_lease_stop_fence.sql`，phase set=`schema|guards`。创建 `action_leases`、`action_resource_locks`、`stop_fence`；schema 与 guards 之间以 Rust 校验既有 Action，发现内联 Lease、`leased|in_flight` 或非零 execution generation 即 `reinitialize-required`；目标对象名碰撞同样拒绝而不接管。
- 0010 将 ActionRequestV2 `lease.max_uses` 收敛为 Schema integer const `1` 并同步生成类型。SQL 只验证 Stop Actor 结构；未来 Stop owner 必须用 Rust Schema/typed decode/RFC 8785 JCS 在写入与 readback 两端证明 canonical bytes。
- ledger shape严格识别：descriptor两列必须同时存在或同时不存在；半shape、半填row、未知format/identity、hash drift均`migration_drift`。数据库version高于binary优先`database_schema_too_new`。
- 每个pending migration先`BEGIN IMMEDIATE`，锁后重新验证ledger，再执行DDL/transform/ledger insert；任一步失败整体rollback。rollback失败不会被忽略。
- **open 后** `reject_legacy_v1_business_data`：若仍存在 `outbox.schema_version=1` 行，或 `content_origins` / `audit_records` / `task_create_idempotency` 非空（表尚在时），返回稳定 `StoreErrorCode::StoredDataInvalid`，message 含 `reinitialize-required:` 前缀。禁止自动清库/隐式升级。

0001–0004 asset bytes 与 descriptor identity 保持稳定。没有自动备份API或down migration；操作员只能外部备份后删除/重建数据库文件。

## 写事务与savepoint poison

```rust
store.with_write_transaction(|transaction| {
    transaction.create_root_task_v2(command)
})?;
```

- public业务写统一使用`BEGIN IMMEDIATE`；closure成功后只有`COMMIT`成功才返回业务结果。
- 每个成功业务 closure 在真实 `COMMIT` 前经过统一 Lease/Stop relation closure：无 Lease/Lock/Fence/leased/in_flight 执行事实时走索引友好快路径；存在事实时严格核验 Action 内联 Lease↔`action_leases`逐字段一致、`resource_refs`↔锁表双向精确集合、Stop Fence 下副作用 Lease 已收敛及外键闭包。半 stage、半 delete、Fence 半收敛或锁集合缺失/多余全部回滚。
- panic/error回滚；outer rollback失败将store标记unhealthy，后续fail closed。
- Outbox append 使用唯一 transaction-bound savepoint helper。
- savepoint operation失败会`ROLLBACK TO`并`RELEASE`；release失败也尝试cleanup。
- rollback/release cleanup失败会poison outer transaction。即使caller吞掉局部错误并返回`Ok`，outer transaction仍不能commit。
- `WriteTransaction`不暴露任意SQL、commit或crate-private bypass constructor。

## Versioned unified Outbox read API + producer-internal append（v2-only）

公开读取类型由`kernel-sqlite/src/outbox.rs`拥有并从crate root re-export：

```rust
pub enum StoredEventEnvelope {
    ActiveV2(TypedEventEnvelopeV2),
}

pub struct OutboxRecord {
    pub envelope: StoredEventEnvelope,
    pub delivered_at: Option<DateTime<Utc>>,
}

pub enum EventAggregateId { Task(Uuid), Action(Uuid), ApprovalChain(Uuid), StopFenceGlobal }
```

Outbox append 不是公开业务入口：待写事件类型与 transaction-bound append 方法均为 crate-private，只能由 root、Action、Approval 等拥有业务状态变化的 producer 使用。crate 外调用方不能构造待写事件或绕过 owner producer 直接追加 Event。

`PendingLegacyEventV1` / `append_legacy_event_v1` / `StoredEventEnvelope::LegacyV1` 已删除且无 alias。producer 内部 append 内核：

1. 分配前exact prevalidation；
2. savepoint内对`(aggregate_type, aggregate_id)`分配连续sequence；
3. 单表insert并由AUTOINCREMENT分配position；
4. 从最终规范列调用唯一stored decoder readback。

active caller不能提供event type或aggregate type：store对`EventEnvelopeV2Payload`穷举匹配，并从`EVENT_ACTIVE_BINDINGS`派生mapping；`EventAggregateId` variant及UUID必须与payload中的task/action/approval chain ID canonical相等，Stop Fence只能是global。mismatch在sequence/position分配前失败。

## 存储shape与严格decoder

post-0003一张`outbox`表保存：position、event/type/envelope version、aggregate/sequence/time、canonical `causation_json`、correlation/dedup、canonical `payload_json`、delivery metadata。

DB CHECK 仍允许 `schema_version IN (1,2)`（0003 asset 字节稳定）；**production decoder 与 open 守卫只接受 v2**：

- stored decoder 对 `schema_version != 2` 一律 `stored_data_invalid`；
- open 对 `schema_version=1` 行返回 `reinitialize-required`；
- task/action/approval_chain/stop_fence-global mapping；
- event ID、dedup key、aggregate sequence唯一；
- payload/causation JSON valid且root object。

单一stored decoder用于`read_after`、`read_undelivered`与`mark_delivered`写前校验。它执行：

- payload/causation parse + RFC 8785 canonical byte equality；
- exact row `schema_version=2` 选择 v2 Envelope Schema 和 typed decoder；
- type/aggregate/payload relation及payload aggregate ID相等；
- timestamp与delivery timestamp解析。

stored corruption统一返回`stored_data_invalid`；caller invalid仍返回`contract_invalid`/`serialization_failed`。分页中任一row损坏使整页失败，不返回partial page。`mark_delivered`先完整decode目标row再conditional update，不能用mark隐藏损坏。

## Cursor与delivery

- `OutboxPosition`: `i64 > 0`。
- `OutboxCursor`: `i64 >= 0`，只接受ASCII十进制；为retained兼容继续接受前导零，输出普通十进制。
- `PageLimit`: `1..=500`。
- `read_after`: `position > cursor`，升序，包含delivered历史。
- `read_undelivered`: 同一position流中的未投递记录。
- `latest_position`: 空表返回`None`。
- `mark_delivered`: `Marked | AlreadyMarked | NotFound`；第一次时间不可覆盖，只有outer commit后返回。

crate没有Publisher loop、claim lease、retention、删除或订阅者ack状态。retained KCP `EventPollResponse v1`仍只能承载Envelope v1；遇到v2不得跳过、降级或推进cursor，未来必须独立升级response/binding/handler。

## Active root TaskCreate v2 repository

`WriteTransaction::create_root_task_v2`是active root-only v2 write path（IC §5.5 / §6.16；ADR-0009）：

```rust
pub struct RootTaskCreateV2Command {
    pub envelope: RootTaskCreateV2EnvelopeFacts, // actor, entry_point, request_id, context, idempotency_key
    pub request: TaskCreateRequestV2,
    pub allocation: RootTaskCreateAllocationV2,  // 七 UUID + correlation/dedup
    pub accepted_at: DateTime<Utc>,              // 调用方注入的第一次时钟读取；repository 不读时钟
}

pub enum CreateRootTaskV2Result {
    Created { task: TaskSpec, creation_provenance_ref: String },
    Replayed { task: TaskSpec, creation_provenance_ref: String },
}
```

流程：`kernel-task-creation` normalize → receipt/idempotency projection + hash → allocation validate → 统一 savepoint 内按固定顺序写入业务事实 `ContentOriginV2`（+ parent_refs）、TaskScope v1（+ source_refs）、TaskSpec v1（`parent_task_id=null`）、`TaskCreationProvenanceV1(root_command_v2)`、`AuditRecordV2(task.creation_recorded)` → 再写 `root_task_create_idempotency_v2` → 最后 `append_active_event_v2(task.created)` → 与 Created 等价的全闭包 canonical readback（Task/Origin/Scope/Provenance↔task 列交叉/Audit/Outbox Event/idempotency 映射）。同 scope 四元组 + 同 hash 重放执行同一闭包读回并返回已存 Task（不产生新 Event）；异 hash 返回 `idempotency_conflict`。任一失败整体回滚不占号。

0004 表：`content_origins_v2`（+ parent_refs）、`task_creation_provenances`、`audit_records_v2`、`root_task_create_idempotency_v2`；canonical `record_json` 为事实源，生成列仅投影。Task/Scope 继续复用 retained v1 **shape** 表。0005 删除 dead v1 业务表 `content_origins` / `audit_records` / `task_create_idempotency`。公开只读：`get_task` / `get_task_scope` / `get_content_origin_v2` / `get_audit_v2` / `get_task_creation_provenance`。legacy `get_content_origin` 已从公开 API 移除（crate 私有只读，0005 后恒 `None`）。

## Action current-snapshot + ActionTransitionIntent（切片4a）

```rust
// Action closed subset implemented in this slice (public):
WriteTransaction::insert_pending_action(InsertPendingActionCommand) -> ActionRequestV2
SqliteStore::get_action(id) -> Option<ActionRequestV2>
// ActionTransitionIntent durable/recovery surface:
WriteTransaction::insert_intent(ActionTransitionIntentV1) -> InsertIntentResult
SqliteStore::get_intent(transition_id) -> Option<ActionTransitionIntentV1>
SqliteStore::get_for_action_revision(...) -> Option<ActionTransitionIntentV1>
SqliteStore::reconcile_intent(transition_id) -> ReconcileIntentResult // prepared|committed|corrupt
// Lease/Stop Fence owners implemented:
WriteTransaction::acquire_lease(AcquireLeaseCommand) -> AcquireLeaseResult
WriteTransaction::begin_dispatch(BeginDispatchCommand) -> BeginDispatchResult
WriteTransaction::release_or_expire_lease(ReleaseOrExpireLeaseCommand) -> ReleaseOrExpireLeaseResult
WriteTransaction::activate_stop_fence(ActivateStopFenceCommand) -> ActivateStopFenceResult
SqliteStore::get_action_lease(id, read_at: DateTime<Utc>) -> Option<ActionLease>
SqliteStore::get_stop_fence() -> Option<StopFenceSnapshot>

// Production status edges are owned by named business orchestrators. The mechanical
// mark_committed_inside/mark_*_committed_with_event bridge is crate-internal; the raw
// mark_committed_with_event wrapper exists only under cfg(test).
```

- pending insert：`status=pending`、`revision=1`、`permission_decision_ref`/`approval_chain_id`/`result`/`lease` 为 null；owning Task 必须存在；canonical JCS readback。
- Action 状态变更没有通用 repository 写入口；命名 owner 先验证自身权威证据，再通过 crate-internal `mark_binding_policy_committed_with_event` 或 `mark_approval_committed_with_event` 进入 `mark_committed_inside`。机械层只投影已经验证的 typed fact，不接受 crate 外 caller 自选 PD/Approval 引用；裸 `mark_committed_with_event` 仅供 `cfg(test)` 的机械闭包测试。
- **需 effects 的边由命名 owner 消费**：`leased` / `in_flight` 退出边的 typed `LeaseReleaseEffect` 由 `release_or_expire_lease`（六条退出边）、Stop owner（`activate_stop_fence` 收敛）与 Approval 撤销（invalidation 的 `leased → cancelled`）经 `LeaseCommitProjection::Clear` 提交——先删 Lease 行（级联删锁）再 CAS 离开 lease-bearing 状态；其它路径仍拒绝，禁止静默半提交。
- `insert_intent`：`transition_id` 与业务六元组双唯一键；同事实重放返回原 intent；非法边 fail closed。
- 机械 commit 在同一 savepoint 内经 `domain-task::apply_action_transition` 做完整领域 evidence 门，再 CAS Action + `append_active_event_v2(action.state_changed)` + 回写 `committed_event_id`；失败不占 sequence/position。同 event id 重放幂等，只验 intent↔event 链路，不要求 Action head 仍停在该 revision。
- `reconcile_intent`：只观察 stored 关系，返回 `prepared|committed|corrupt`，不补造 event 或更换 transition id。Committed 只对比 intent↔outbox event 快照字段；后续合法推进不得映射 Corrupt。
- 已实现 Policy binding owner、Approval resolution owner、**Action Lease 全链 owner**（`acquire_lease` / `get_action_lease` / `begin_dispatch` / `release_or_expire_lease` / `activate_stop_fence` / `get_stop_fence`）与 migration 0010 的 Lease/Stop **持久化基座**。**（阶段史：本段写于切片4a；child materializer 已随切片5 落地，见上文「Child Action materializer（切片5）」节）** recovery orchestrator 仍**未实现**；Stop 动态 allocation 已按 causation 必须别名到持久 intent 收敛。

## PolicyRuleV2 + PermissionDecisionV2 + 评估编排（切片4b）

```rust
// PolicyRule closed write/read
WriteTransaction::append_policy_rule_revision(PolicyRuleV2) -> PolicyRuleMutationResult
SqliteStore::get_policy_rule_revision(rule_id, revision) -> Option<PolicyRuleV2>
SqliteStore::get_current_policy_rule(rule_id) -> Option<PolicyRuleV2>
SqliteStore::list_current_policy_rules() -> Vec<PolicyRuleV2>
SqliteStore::get_policy_set_revision() -> i64 // bootstrap 0 = empty set

// PermissionDecision closed set (IC §6.10.6 subset; no update/delete)
// pub(crate): sole legitimate write path is evaluate_action_permission (bidirectional Action↔PD binding)
WriteTransaction::append_permission_decision(PermissionDecisionV2) -> PermissionDecisionV2  // pub(crate)
// decision_revision: 0 placeholder or exact next; repository allocates max(action)+1
SqliteStore::get_permission_decision(id) -> Option<PermissionDecisionV2>
SqliteStore::get_current_permission_decision_for_action(action_id) -> Option<PermissionDecisionV2>
SqliteStore::list_permission_decisions_for_action(action_id) -> Vec<PermissionDecisionV2>
SqliteStore::validate_current_permission_decision_for_action(action_id)
    -> Option<PermissionDecisionV2> // Action.permission_decision_ref ↔ PD current

// Evaluation orchestration (single savepoint; no Approval creation)
WriteTransaction::evaluate_action_permission(EvaluateActionPermissionCommand)
    -> EvaluateActionPermissionResult // PD + Action CAS + permission.evaluated Audit

// Approval creation orchestration: evaluation + initial operation Approval in ONE savepoint.
// Caller supplies only non-derived request facts; the repository derives confirmation mode and
// the complete operation subject from Task/Action/newly-current PD. Chain collision is rejected
// before rules, matching, rate-limit consumption, or PD insertion.
WriteTransaction::evaluate_action_permission_and_create_approval(
    EvaluateActionPermissionCommand,
    approval_chain_id,
    CreateOperationApprovalRequestCommand,
    ApprovalEventAllocationV1,
    audit_record_id,
) -> (EvaluateActionPermissionResult, ApprovalMutationResult)

// Approval resolution orchestration: resolve the chain AND commit pending -> approved in ONE
// savepoint. The Action edge is driven only by a proof re-derived from the committed resolution
// (current chain head + approved + Action/PD/chain agreement); the caller supplies no Approval refs.
WriteTransaction::resolve_approval_and_commit_action(ResolveApprovalAndCommitActionCommand)
    -> (ApprovalMutationResult, ActionRequestV2, OutboxRecord)
```

- **PolicySet**：空初始 revision=0 是权威空状态；每次成功 rule mutation 同事务 +1。
- **PolicyRule**：append-only revision 历史；current head = MAX(revision) per id；disable 写新 revision `enabled=false`；物理 delete 禁止。
- **PermissionDecision**：immutable append；`decision_revision` 连续；canonical JCS readback；断号/id 冲突 fail closed。
- **评估编排不变量**：pending Action + expected revision CAS；TaskScope containment始终强制（合同不变量，非caller策略）；每个顶层 `evaluate_action_permission` / `evaluate_action_permission_and_create_approval` 命令只创建一个 transaction-bound typed UUID collector，先登记 PD/audit/transition/event 等 command allocations，再在 matcher/rate-limit 或任何持久化写入前登记该路径实际读取的 persisted Action/Task/TaskScope/current PD/PolicyRule；复合 Approval 创建继续用同一 collector 登记 chain/request/Approval event/audit，嵌套阶段不得另建集合。新 PD allocation 与命令前 current PD reference 是不同生命周期用途，互相复用必须 `ContractInvalid`。enabled `PolicyRuleV2` heads直接进入domain-policy matcher，无v2→v1转换、Generic probe、remote rule-ID side table或mode/decision adapter；五种`ConfirmationModeV1`都生成对应`PermissionDecisionV2Decision`，`remote_signature`正常参与specificity/priority/winner-only rate-limit/fallback；`kernel-authorization`真实重算material/observation指纹；append PD + Audit `permission.evaluated`；allow/deny状态边经 intent + crate-internal Policy owner bridge 发`action.state_changed`，confirmation为metadata CAS不发Action状态事件；失败整体回滚（含rate-limit、PD、Audit、Outbox sequence）。
- **Approval 授权不变量**：operation Approval subject 由仓储基于真实 Task/Action/当前 PD 投影派生，caller 无法注入派生字段；Action 已拥有链时普通重评失败关闭且零写入；`pending→approved` 要求当前 PD 为 Allow，否则必须提供由仓储从持久事实派生的 typed Approval proof（记录为当前链头、decision=approved、Action 绑定同链与同 PD、PD 为 RequireConfirmation 且 requirement 指向同链）；caller 提供的 `approval_resolution_ref` 一律拒绝。
- **`remote_signature` 真实验签已落地**：resolution append 同一事务内经 `kernel-authorization` 的 Ed25519 RFC8032 pure-mode 端口（RFC 8785 JCS preimage 重建）验签，失败 `signature_invalid` 整体回滚；usable-authorization 校验不再拒绝 remote 决议，acquire 执行授权对 `RequireRemoteSignature` 走 usable resolution 校验。
- **Approval 解析证据、UUID 与审计**：`approval.state_changed.confirmation_mode` 从链的权威原始 request 回读，缺失/损坏返回 `stored_data_invalid`；denied resolution Audit 固定 `outcome=blocked`、reason `approval_resolved_denied` 并保留 operation PD/policy context。每个顶层 `resolve` / `resolve_approval_and_commit_action` / `invalidate_and_optionally_replace` 使用唯一 transaction-bound typed collector，外层 Action transition/event allocation 与内层 request/head/Challenge/Task/PD/evidence/credential persisted facts 必须在任何 consume/append/CAS 前进入同一用途闭集；相同 UUID 只允许同一语义事实 mirror，跨用途碰撞返回 `ContractInvalid` 且零写入。
- **Identity / Challenge 原子闭包**：Challenge 消费强制 `now >= expires_at` 过期 CAS；公开的可独立提交 `consume_challenge` 已删除，system/remote resolution 通过 crate 内 `consume_challenge_with_binding` 把 consume 与 Approval record/head/Event/Audit 放在同一 savepoint；过期返回 typed `ChallengeExpired`，只提交 Challenge 终态与 identity Audit。local/system 证据校验 resolver actor/entry、时间窗、challenge/request/chain/task/subject/material 绑定；credential 生命周期与 local/system 证据插入均写 identity Audit。
- **（阶段史：本段写于切片4b；child materializer 已随切片5 落地）** 本阶段仍未实现：child materializer、recovery 闭集、Provider 真实远程验签；Policy binding 与 Lease/Stop 全链 owner 已实现。`invalidate_and_optionally_replace` 已同事务撤销链上 leased Action 的 Lease 并驱动 `leased→cancelled`（approved 不动、in_flight 不打断），`resolve` / `resolve_approval_and_commit_action` 已能驱动 `pending→approved`。
- **`acquire_lease` 责任边界**：仅在 Stop Fence 未激活时执行；从 Action 的 canonical `resource_refs` 派生 Resource Lock 集合；在同一 savepoint 内顺序插入 `action_leases` / `action_resource_locks`、写 `approved → leased` intent（`expected_execution_generation=G`，`resulting_execution_generation=G+1`）、CAS Action（revision +1、execution_generation +1、内联 lease 镜像）、追加 `action.state_changed`，失败整体回滚。重名资源冲突返回 `action_not_executable` 并回滚，不占 sequence。
- **`get_action_lease` 严格只读**：在 leased / in_flight 状态验证 Action 内联 lease、action_leases 行、Resource Lock 集合与 canonical `resource_refs` 完全一致、max_uses=1、lease generation 与 Action.execution_generation 一致、acquisition intent 与 event 一对一闭包；Stop Fence 激活时返回 `stop_fence_active`；lease 过期或数据破损返回 `stored_data_invalid`。
- **需 effects 的其它边仍 fail closed**：`leased → approved|cancelled|unknown_side_effect` 与 `in_flight` 终态要求 typed `LeaseReleaseEffect`，由命名 owner（`release_or_expire_lease` / Stop owner / Approval 撤销）经 `LeaseCommitProjection::Clear` 消费，其它路径仍拒绝，禁止静默半提交。

## Child Action materializer（切片5）

`WriteTransaction::materialize_child_task(command)` 原子完成一个 `kernel.task/task.child.create` S1 Action：在单一 savepoint 内写入完成态 Action、子 Task/TaskScope/ContentOrigin/Provenance、VerificationResult、创建 Audit 与两条 Outbox 事件（子 `task.created` sequence=0 causation=父 Action；`action.state_changed` causation=完成 transition），并执行严格 readback。

- **幂等四分支**：同 hash → 重放（`Replayed`，零新写入）；异 hash → `ChildMaterializationConflict`（分 proposal/material 维度）；跨 Action 同 execution key → `IdempotencyConflict`；mapping 缺失/不完整 → `StoredDataInvalid`。
- **双命脉校验**：proposal == `Action.structured_arguments`（`ContractInvalid`）；物化 delta 的指纹 == 评估时 `permission.evaluated` 审计记录的 `child_task_delta_hash`（`StoredDataInvalid`）。
- **readback 严格性**：fresh/replay/reconcile 共用同一校验函数；replay 从存储权威确定性重建 TaskScope/ContentOrigin/VerificationResult 后字节级全等比较，篡改必 `StoredDataInvalid`/`Corrupt`。
- **闭包钉**：mapping 行钉 transition_id、task_created_event_id、action_state_changed_event_id、scope_id、origin_id 五个闭包 FK（DEFERRED），replay 经 `outbox::read_record_by_event_id` 精确读。
- **approval 消费**：声明消费 approved resolution 时校验 usable/chain-head/requirement，并全链投影 `approval_resolution_ref`；credential/challenge refs 真实注入 external snapshot；rollback_capability 按 `rollback_policy` 投影。
- **UUID 闭集**：10 个 allocation UUID 在写入前与全部持久化表（含 0011 新表）碰撞预检，`ConstraintViolation` 零写入。

查询面：`get_child_materialization_by_action` / `get_child_materialization_by_child_task`（`ChildMaterializationView`，全量闭包校验）；`reconcile_child_materialization(action_id)`（`Committed` / `Absent` / `Corrupt`，Safe Recovery 面）。

## Transaction-bound rate limit

`WriteTransaction::rate_limit_port()`保持原合同：preview只读，winner-only consume在同一transaction重新计数并写入；Store本身不实现可独立commit的RateLimitPort。评估编排在同一事务内注入该 port。

## 错误

稳定`StoreErrorCode`包括配置/open、SQLite busy/full/corrupt、migration failed/drift/too-new、constraint、serialization、contract invalid、stored data invalid、cursor、not found与internal store error。**旧库 reinitialize-required 使用既有闭集 `StoredDataInvalid`**（`as_str() = "stored_data_invalid"`），message 稳定前缀 `reinitialize-required:`，不泄漏 SQL/payload。错误消息不包含SQL、参数、payload或密钥。

## 明确未实现

root TaskCreate v2 repository + migration 0010 Lease/Stop 持久化基座 + **Action Lease/Stop Fence 全链 owner**（acquire/dispatch/release/stop 激活与只读）+ **切片5 Child Action materializer**（migration 0011 原子物化 + 严格 readback + 四方法闭集，child `task.created` 与 `action.state_changed` 两事件 producer 已随 bundle 接入）已落地。child-completion与recovery写方法、Provider真实远程验签接入（Ed25519 纯库已落地，Provider 接入未做）、Publisher、versioned KCP poll、server/agentd、retention/claim lease仍未实现。Publisher/poll不在`V2InitialBuildActive`；§13.7完整闭合仍需 `task.list` / `event.subscribe` / `event.poll` handler 与 recovery orchestrator。
