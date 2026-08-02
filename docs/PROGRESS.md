# Shittim 实现进度

> 状态日期：2026-08-02（`V2InitialBuildActive`：Policy v2、Action/PermissionDecision、Approval/Identity 与顶层 UUID 用途闭集已落地；Action Lease/Stop Fence 全链已落地——`acquire_lease` / `get_action_lease` / `begin_dispatch` / `release_or_expire_lease` / `activate_stop_fence` / `get_stop_fence` 全部实现并闭环 `approved→leased→in_flight→completed`；migration 0010 Lease/Resource Lock/Stop Fence 持久化基座已落地并独立终验 GO；切片 4c 安全闭环最终验收通过（11/11 High：Approval 撤销 Lease 与真实远程验签均已落地）。**child materializer（切片 5，migration 0011）与 §13.7 五方法 handler（切片 6：`stop.activate` / `stop.status` 已落地，`task.list` / `event.subscribe` / `event.poll` 仍未实现）已落地并通过独立审查与统一门**。门禁稳定性：并发首次打开同一 SQLite 新文件的偶发 `SqliteBusy` 失败已修复——`SqliteStore::open` 对 `SqliteBusy` 在 3×`busy_timeout` 窗口整体重试（`config::retry_on_busy` 辅助，`config_tests` 单测覆盖瞬态重试/窗口耗尽/非 busy 直通，open 层另有路径 marker 隔离的确定性注入测试验证重试与窗口耗尽）。
>
> **新设备接力开发请先读 [`DEVELOPMENT_HANDOVER.md`](DEVELOPMENT_HANDOVER.md)**（环境准备、标准流程、切片 5/6 精确任务、验收债务与已知问题）。

域状态表唯一来源：[`IMPLEMENTATION_MATRIX.md`](IMPLEMENTATION_MATRIX.md)。本文只保留当前切片事实、未完成 backlog 与下一步；逐切片编年史由 git log 与 ADR 承载。

## 当前里程碑：V2InitialBuildActive

**目标（ADR-0009 / IC §13.7）**

- 没有用户，不从 v1 迁移数据；当前存储格式就是 v2 fresh baseline。
- production bindings + v2 dispatcher + v2 repository 作为**初始交付**，不是旧部署 cutover。
- 旧开发数据库必须拒绝启动并返回稳定 `reinitialize-required`；禁止自动清库 / 隐式升级 / 读后补写。
- v1 runtime 写路径删除（handler/adapter/repository write、legacy direct-child、`append_legacy_event_v1`、`StoredEventEnvelope::LegacyV1` production variant），而非加版本分支。
- `TaskCreationProvenance` 只保留 `root_command_v2` / `child_action_v2`。
- **不在本里程碑**：Publisher、versioned KCP `event.poll`；retained poll v1 不得返回 EventEnvelope v2。

**完成定义**

1. root `task.create` v2 唯一 active 写路径（repository + handler + method-aware preflight + production bindings）；
2. child 唯一经 `task.child.create` Action materializer；
3. Action / PermissionDecision / Approval v2 真实持久化闭环；
4. 有业务 owner 的 active Event producer：root/child `task.created`、`action.state_changed`、`approval.state_changed`；
5. production MethodVersionBindings：`task.create` active=[2]、v1 仅 `unsupported_schema_version`，其余七方法 active=[1]；
6. 不得伪造无 owner 的 producer；v1 写路径已删；旧库 reinitialize-required。

**七切片计划**

| 切片 | 内容 | 状态 |
|---|---|---|
| 0 | 规范文档落锤：ADR-0009 + IC/CORE/CONFORMANCE/PROGRESS/MATRIX/API 对齐 | **已完成** |
| 1a | root创建路径持久对象Schema：ContentOriginV2、AuditRecordV2、TaskCreationProvenanceV1、AuditAllocationV2 + manifest/generated/conformance | **已完成** |
| 1b | Action/child Schema：ActionTransitionIntentV1、ActionRequestV2、Delta/Material/Observation projection + `kernel-authorization` pure crate | **已完成** |
| 1c-i | 授权核心五Schema：PermissionDecisionV2、PolicyRuleV2、ApprovalRecordV2、SubjectProjectionV1、ApprovalEventAllocationV1 | **已完成** |
| 1c-ii | 身份/挑战/证据与远程签名八Schema | **已完成** |
| 2 | fresh SQLite 基线 + root repository | **已完成** |
| 3a | production MethodVersionBindings 基础层（manifest 八方法 + stage gate + generated catalog/selector） | **已完成** |
| 3b | method-aware KCP v2 preflight/dispatcher/handler 消费 bindings | **已完成** |
| 3c | 删除 v1 runtime 写路径 + Outbox v2-only + 旧库 reinitialize-required + migration 0005 | **已完成** |
| 4a | Action 持久化 + ActionTransitionIntent + `action.state_changed` producer | **已完成** |
| 4b | PolicyRule + PermissionDecision repositories + Action 评估编排（Approval 属 4c） | **已完成** |
| 4c | Approval / Identity repository 与安全闭环 | **已完成**（11/11 High 闭合：Approval Lease 撤销 + 真实远程验签落地） |
| 5 | child Action materializer | 未开始 |
| 6 | §13.7 谓词闭合（依赖 4–5 + 其余 active producers） | 未开始 |

**已实现（代码/Schema 事实）**

- 规范与工程基线：Freedom-first / Kernel Owns Reality 合同、Apache-2.0、双仓同步 library/CLI、Node/pnpm 零依赖根基座（exact Node 24.18.0 / pnpm 11.3.0）、统一门 `scripts/check-schema.sh`。
- Schema/Rust 契约：Rust workspace、Draft 2020-12 + manifest v2（production=`80 = 38 retained + 42 component-native`）、`schema-tool` 单 root transaction / target-scoped IR / TaggedUnion / string enum `ALL` / string-array const / RFC 8785；production `METHOD_VERSION_BINDINGS` 为 IC §13.5 八方法集。ADR-0010已直接退役未投产旧PolicyRule/PermissionDecision/ApprovalRecord三合同，无migration/compat/旧validation入口；`V1`后缀本身不等于legacy。
- 纯领域：`domain-task`（Task/Action 状态图、revision/plan_version、Approval v2 resolution 领域表达：`approval_resolution_ref` 仅限 `pending→approved` 消费且要求 `permission_decision_ref` 非空，deny/confirm 携带 fail closed）、`domain-policy`（直接消费`PolicyRuleV2`、URI/glob/Default Allow/五种confirmation mode/winner-only rate-limit，Stop Fence/Recovery独立Blocked）。
- 持久化：`kernel-sqlite` migration 0001–0010；AuditRecord **v2**；strict Task/TaskScope/ContentOrigin(v2)读；版本化 **v2-only** Outbox + 严格stored decoder + savepoint poison；active root TaskCreate v2 repository；Action current-snapshot/transition/`action.state_changed`；PolicyRuleV2/PermissionDecisionV2/评估编排；Approval current-head CAS/`approval.state_changed`/Identity repositories；migration 0010 已建立 `action_leases` / `action_resource_locks` / `stop_fence`、CAS/不可变守卫、旧执行事实 reinitialize-required 校验与统一事务提交前关系闭包；legacy TaskCreate/Audit/Outbox v1 write已删。
- migration 0009 `action_permission_decision_heads`：每个 Action 当前 PermissionDecision 的唯一持久权威，配 CAS/绑定守卫/禁删触发器；评估经受事务生命周期约束的 Staged→Bound 协议绑定，确认路径保持 pending，放行与拒绝在同一次状态 CAS 内投影 `permission_decision_ref`，一次评估只推进一次 Action revision。
- KCP 库级：`kernel-kcp` method-aware Value preflight、三方法 registration/dispatcher/handler（`system.ping` / **active root `task.create` v2** / `task.get`）与 SQLite adapter→`create_root_task_v2`；不可连接，无 bytes/frame/server。
- ADR-0006 首批：12 business-v2 Schema + `kernel-task-creation` pure library + official fixtures/harness + schema-tool strict pointer CLI。
- `kernel-task-creation` 合同缺口测试补齐（`91ec798`）：child 十 UUID duplicate-internal / duplicate-opaque 单测、`InvalidUuid` 与 `internal_shape` 防御路径直接构造/真实触发测试、child 版「普通字符串不 trim」proptest；独立合同审查（对照 IC §5.3/§5.3.1、ADR-0006、三份 official fixtures 对账）结论为**无合同违反**，投影确认未迁就 repository 重放引入 request ID；幂等重放偏差的决策输入见 `docs/design/task-create-idempotency-decision-input.md`（待维护者拍板）。
- ADR-0008 前两段：Event v2 八 Schema、`EventTypeBinding`/active·legacy catalog、typed EventEnvelope v1/v2、migration 0003 descriptor v1 与统一 Outbox shape（切片3c 起 production v2-only）。
- V2InitialBuildActive切片1a–1c-ii：root持久对象、Action/child授权、授权核心、身份/挑战/证据Schema与pure crate已落地；该阶段历史快照为manifest=83，ADR-0010退役三项旧Policy合同后当前为80。
- V2InitialBuildActive切片2：migration 0004（`content_origins_v2`、`task_creation_provenances`、`audit_records_v2`、`root_task_create_idempotency_v2` + tasks/scope FK 重建以允许 v2 origin）；`WriteTransaction::create_root_task_v2` 单事务写 Origin/Scope/Task/Provenance/Audit/idempotency/Event；全闭包 canonical readback（Created/Replayed 共用）；幂等重放/冲突；回滚不占号；与 v1 表互不污染。
- V2InitialBuildActive切片3a：production `method_version_bindings` 精确八方法（`task.create` active=[2]/legacy=[1]，其余 active=[1]）；`validate_production_manifest_stage` 要求 Envelope-derived 完整集 + IC §13.5 lifecycle；generated `METHOD_VERSION_BINDINGS` 非空且 `select_request_version` 可用。
- V2InitialBuildActive切片3b：kernel-kcp preflight 按 (family, method, payload.schema_version) 调 `select_request_version`；V2 Envelope 结构验证 + active payload Schema；`task.create` v2 Accepted/Registered，v1 → `unsupported_schema_version`；handler 七 UUID（含 CreationProvenance）+ root-only 检查 + `TaskCreateResponseV2`；adapter 映射 `create_root_task_v2`；删除 kcp 侧 v1 create handler/adapter/ports 路径。
- V2InitialBuildActive切片3c：删除 `create_task`/`TaskCreateCommand`/`prepare_legacy_v1_create`、AuditRecord v1 write、`append_legacy_event_v1`/`PendingLegacyEventV1`/`StoredEventEnvelope::LegacyV1`；Outbox decoder 对 schema_version=1 → `stored_data_invalid`；`SqliteStore::open` 后 `reject_legacy_v1_business_data`；migration 0003 transform 对非空 legacy Outbox 直接 reinitialize-required；migration 0005 在空表前提下 drop dead v1 表。
- V2InitialBuildActive切片4a：migration 0006（`actions` + `action_transition_intents`）；`insert_pending_action` / `get_action`（公开）；intent durable/recovery surface + crate-internal mechanical commit 同事务 CAS+`action.state_changed`（causation=`action_transition`）+ reconcile 三态；状态事件由命名业务 owner 进入唯一机械权威，无公开裸状态迁移入口；需 lease effects 的边 fail closed；domain-task 边合法性与 evidence 门；sequence/position 失败不占号。
- V2InitialBuildActive切片4b：migration 0007（`policy_set_metadata` bootstrap revision 0、`policy_rules`、`permission_decisions`）；PolicyRule append-only revision + global set counter；PD immutable append（连续 decision_revision）+ Action ref 双向校验；`evaluate_action_permission` 单事务 matcher→指纹→PD→`permission.evaluated` Audit→Action CAS（allow/deny/require_* deferred，无 Approval 创建）；rate-limit 同事务消费与回滚；material/observation 双指纹真实重算。
- V2InitialBuildActive切片4c：migration 0008（`approval_records`、`approval_chain_heads`、`identity_credentials`、`identity_challenges`、`identity_evidence`）；三种 Approval head mutation 由命名 owner 独占：初始 operation request 只经 `evaluate_action_permission_and_create_approval` 与 Policy evaluation/PD/Action binding 同一 savepoint 创建，无 generic 独立 `append_request`；`resolve` 处理 expected-head/mode evidence，`resolve_approval_and_commit_action` 可在同一 savepoint 重新派生 usable proof 并驱动 `pending→approved`；`invalidate_and_optionally_replace` 原子推进 replacement。每成功 head 变化恰好一条 `approval.state_changed` + 对应 `approval.requested|resolved|invalidated` Audit；CAS 冲突/replay 不产 Event；Identity credential register/rotate/revoke、challenge issue/consume/expire（终态不可逆、expire 只写 `identity.challenge_expired` Audit 不发 Approval event）、local/system evidence immutable。`validate_usable_approval_resolution`（IC §6.10.1）在函数本体读取 Stop Fence：Fence 激活时 usable-resolution 校验与 acquire 执行授权一律 `stop_fence_active`，resolve 不再驱动 `pending→approved`（CORE §19.2：Fence 期间 pending 保持 pending，只由 Stop owner 收敛状态）。

**未实现（不得宣称完成）**

- **Lease / Stop Fence 全链已闭合**：`domain-task` 已闭合 Lease 生命周期的 typed effect（三条 leased 退出边与三条 in_flight 退出边均产 `LeaseReleaseEffect`）；migration 0010 已建立 Action Lease、Resource Lock、Stop Fence 三表及 descriptor、关系守卫、`max_uses=1` 生成合同与统一 COMMIT 前关系闭包；`ActionTransitionIntentV1` 已拆分为 `expected_execution_generation` / `resulting_execution_generation` 以支持 `approved→leased` 的 `G→G+1`；`WriteTransaction::acquire_lease`（Stop Fence 未激活时方可获取，执行授权从真实 current Task/Action/PD/Approval/资源事实派生）、`SqliteStore::get_action_lease`（严格读取验证双源一致、intent/event 一对一闭包）、`begin_dispatch`（leased→in_flight，Lease 行保留、双源一致）、`release_or_expire_lease`（先删 Lease 级联删锁再 CAS 离开 lease-bearing 状态）均已落地，SQLite 经 `LeaseCommitProjection::Clear` 消费 typed `LeaseReleaseEffect`（reason 与边匹配验证），`approved → leased → in_flight → completed` 端到端可走通；Stop owner `activate_stop_fence` / `get_stop_fence` 已落地：单一 `BEGIN IMMEDIATE` 内插入 `stop_fence` 单例（generation=1）+ 唯一 `stop_fence.activated`（global aggregate），受影响副作用 Action 分流 `leased→cancelled` 与 `in_flight→unknown_side_effect`（各自原子删 Lease/锁 + 独立 intent + 恰好一条 `action.state_changed`），transition/event ID 由 transaction-bound 纯内存 typed allocation source 事务内产出并登记进命令唯一 UUID purpose collector（ADR-0011 §5 例外），重复激活幂等 readback 零写入；Stop Actor 写入前与 readback 均做 Schema/typed decode/canonical JCS bytes 校验。
- **切片 4c 安全闭环最终验收通过（11/11 High）**：除既有 operation Approval 当前 PD 绑定、Challenge 过期 CAS/审计和 identity 审计外，已闭合全部 11 项：每个顶层 Approval/Policy 复合命令用唯一 transaction-bound typed collector，在任何业务写入前覆盖 command allocations 与该路径实际读取、验证或消费的 persisted Task/Scope/Action/PD/PolicyRule/request/challenge/evidence/credential UUID 用途，嵌套 prepare/apply 禁止另建 collector；事件 payload 从权威原始 request 回读真实 confirmation mode；denied resolution 审计为 `outcome=blocked`、稳定 reason code 且保留 operation PD/policy context；local/system 证据校验 actor/entry/time/challenge/request/chain/task/subject/material 绑定；system/remote Challenge 消费与 Approval resolution/head/Event/Audit 通过 `consume_challenge_with_binding` 同一事务提交，过期返回 typed `ChallengeExpired` 而不写 resolution；**Approval invalidation 同事务撤销链上 leased Action 的 Lease 并驱动 `leased→cancelled`**（approved 不改状态、in_flight 不打断，撤销 transition/event ID 经命令唯一 collector）；**`remote_signature` 真实密码学验签落地**（kernel-authorization `RemoteSignatureVerifier` 端口：RFC 8785 JCS preimage 重建 + Ed25519 RFC8032 pure mode，binding 通过后同事务验签，失败 `signature_invalid` 整体回滚零写入），远程决议可正常作为批准 Action 的授权，`remote_signature` 失败关闭已解除，acquire 执行授权对 `RequireRemoteSignature` 走 usable resolution 校验。
- child Action materializer；Action 闭集其余写方法（child completion、recovery list；policy binding、lease acquire/dispatch/release 已实现）；真实远程验签 Provider 边界。
- 其它 active business **producer**只缺child；approval producer已在4c落地。**Publisher** 与 versioned KCP **poll** 明确不在本里程碑。
- 其余 Command 幂等/乐观锁；**五方法**正式 handler；Unix Domain Socket / Windows Named Pipe KCP **server/client**；**agentd**。
- `ts/*` 包、SDK client、Pi `agent-runtime`；统一 **Extension SDK Base**；optional Computer Use Profile；Tauri 桌面客户端；Provider/Memory/Initiative/Broker。
- `specs/CONFORMANCE.md` 全部 BASE 与声明的 Profile/Platform 条件套件。

## 未完成 backlog

- [x] 切片 0：规范文档落锤（ADR-0009 等）
- [x] 切片 1a：root创建路径持久对象Schema/manifest/generated/conformance
- [x] 切片 1b：Action/child五Schema + `kernel-authorization` pure crate + projection official fixtures/harness/oracle
- [x] 切片 1c-i：授权核心五Schema + SubjectProjection pure API/fixture
- [x] 切片 1c-ii：Credential/Challenge/Evidence/Remote signature家族
- [x] 切片 2：fresh SQLite 基线 + root repository
- [x] 切片 3a：production MethodVersionBindings 基础层
- [x] 切片 3b：method-aware KCP v2 preflight/dispatcher/handler
- [x] 切片 3c：删除 v1 写路径 + Outbox v2-only + 旧库拒绝 + migration 0005
- [x] 切片 4a：Action 持久化 + ActionTransitionIntent + `action.state_changed` producer
- [x] 切片 4b：PolicyRule + PermissionDecision repositories + Action 评估编排
- [x] 切片 4c：Approval/Identity repository（11/11 High 全闭合：Approval Lease 撤销 + 真实远程验签）
- [x] 收尾：Approval 当前绑定 PD 门 / Action 唯一状态事件权威 / post-Outbox 全量回滚证明
- [x] Lease + Stop Fence：领域 release effect、generation/intent 合同、`acquire_lease` / `get_action_lease` / `begin_dispatch` / `release_or_expire_lease`、Stop owner（`activate_stop_fence` / `get_stop_fence`）与 Approval Lease 撤销全部完成
- [x] 切片 5：child materializer（migration 0011 原子物化 bundle + 严格 readback；独立审查三轮闭合 0C/0H/0M；`materialize_child_task` / `get_by_action` / `get_by_child_task` / `reconcile` 闭集）
- [x] 切片 6：§13.7 谓词闭合（`stop.activate` / `stop.status` handler 已落地并通过独立审查；`task.list` / `event.subscribe` / `event.poll` 仍缺 handler）
- [ ] active Event business producer：child `task.created`（切片 5 已接入）、`action.state_changed`（4a 已接入）、`approval.state_changed`（4c 已接入）其余 producer
- [ ] `task.list` / `event.subscribe` / `event.poll` 正式 handler；可连接 KCP server/client；`agentd`
- [ ] 其它 Command 幂等与乐观锁；Task list cursor ADR 拍板后实现
- [ ] Publisher + versioned KCP poll（**后续里程碑**，不在 V2InitialBuildActive）
- [ ] Extension SDK Base → `schema/SDK`；TS 包 / SDK client / `agent-runtime`
- [ ] optional Computer Use Profile；桌面客户端；Provider/Memory/Initiative/Broker
- [ ] CONFORMANCE 全量 BASE + 声明 Profile 套件

## 当前阻塞

- kcp runtime与sqlite repository已切到active create v2 / Outbox v2-only；Action/`action.state_changed`（4a）、PD/PolicyRule/评估编排（4b）、Approval/Identity/`approval.state_changed`（4c 结构层）已落地；切片 5（child materializer）与切片 6（`stop.activate` / `stop.status` handler）已落地；`task.list` / `event.subscribe` / `event.poll` 无handler，禁止启动server。
- §13.7 谓词：Lease 全链、真实远程验签、child materializer 与 `stop.activate` / `stop.status` 已落地；仍缺 `task.list` / `event.subscribe` / `event.poll` handler 与 recovery orchestrator，§13.7 完整闭合待后续切片。
- legacy v1 repository 的 Delegation 正向路径未实现（非 null 固定 not found）；active v2 repository 同样在 Delegation authority 未落地前 fail-closed 返回 `delegation_not_found`。
- Task list cursor 编码须先 ADR/API 拍板。
- Audit：`permission.evaluated` 已在评估编排同事务校验 policy_context 与 PD 字段相等；仍缺 rollback 权威投影、Provider/ModelCall 一致性与其它业务 producer。
- Extension SDK Base / Computer Use 仍为 `contract-only`；不得把规范或根 Node 工作区冒充 SDK 实现。
- 真实 Provider/Channel/Privilege Broker 需要后续真实环境；当前无伪造支持。
- 默认 PATH 可能不是 Node 24.18.0；须显式将 `~/.local/share/pnpm/bin` 放在 PATH 最前。

## 下一步

1. 在实现 `acquire_lease` 前先修订并验证 generation/intent 合同：现有 `ActionTransitionIntentV1.execution_generation` 同时被当作 pre-CAS 与 post-CAS generation，而 acquire 必须 `G→G+1`；不得用例外条件绕过。同期收敛 Stop 动态 allocation：causation 必须由持久 intent 派生，只作为同一事实 alias，不另分配 UUID。**（已随 commit `c2efc5d` 完成）**
2. 实现 `acquire_lease` / `get_action_lease`、`begin_dispatch` / `release_or_expire_lease`、Stop owner（`activate_stop_fence` / `get_stop_fence`）与 Approval invalidation 同事务撤销 Lease（4c 清零）。**（已随本轮全部落地）**
3. 引入可信 `RemoteSignatureVerifier` 边界（首个实现为 Ed25519 RFC8032 pure mode，纯 crypto 放 `kernel-authorization`），解除 `remote_signature` 的失败关闭并完成 4c 最终验收。**（已随本轮落地）**
4. 切片5：实现child materializer并接入child `task.created` producer（mapping 表用 migration 0011；child creation Audit 的 actor/entry_point 取 typed execution context）。**（已随本轮落地：独立审查三轮闭合 0C/0H/0M，统一门全绿）**
5. 切片6：在切片5完成后闭合§13.7全部谓词。**（`stop.activate` / `stop.status` handler 已落地并过审查；`task.list` / `event.subscribe` / `event.poll` 仍缺）**
6. **之后**再做 `task.list` / `event.subscribe` / `event.poll` handler、Publisher、versioned KCP poll、可连接 server，以及 Extension SDK Base 与 TypeScript/client。

## 最近验证

### 2026-08-02 review 修复闭合与发布状态

- 独立 review（`agent-guide/review/`，不同会话只读审查）对 Lease/Stop Fence 全链切片给出 NO-GO（3 项 Medium），已从根因修复并复评 **GO**（0 Critical / 0 High / 0 Medium）：
  - **M1**：`validate_usable_approval_resolution`（IC §6.10.1）在函数本体读取 Stop Fence，Fence 激活时 usable-resolution 校验与 acquire 执行授权一律 `stop_fence_active`，resolve 不再驱动 `pending→approved`（CORE §19.2：Fence 期间 pending 保持 pending）；
  - **M2**：Approval invalidation 撤销路径（`revoke_leased_action_for_invalidation`）删除 Lease 前复用 `load_lease_closure` 双源闭包校验（inline↔行逐字段、锁集合精确闭合、intent/event 闭包），与 Stop owner 路径对称，单行错 generation 篡改 fail closed 为 `stored_data_invalid` 而非静默收敛；
  - **M3/L2/L3**：`docs/api/kernel-sqlite.md` 签名类型名与 `docs/api/approval-contract.md` 残留矛盾句订正；
  - **L1**：kernel-kcp 错误码映射测试补全 8 个切片新增码 cases；
  - **L4**：`SqliteStore::open` 的 SqliteBusy 重试窗口增加确定性注入测试（路径 marker 隔离 + 互斥串行，瞬态重试成功/窗口耗尽原码暴露）。
- 修复提交位于分支 `lease-stop-fence-closure`（基于 master `d31259d`）：`741d8a9`（修复 M1/M2）、`3de8e73`（测试 L1/L4）、`777c4a1`（文档 L3/PROGRESS）。统一门 `./scripts/check-schema.sh` 复跑全绿。
- **发布状态（未完成）**：按 `agent-guide/Easy/commit-and-push-Easy/AGENT.md` 闭环，主仓 push + 远端 SHA 核对与文档镜像同步（`pnpm run sync:docs-repository` + `check:docs-repository`）需待分支合并到 master 并推送后进行；镜像只能来自已推送主仓 commit，禁止从未推送工作区制作。

当前代码基线（含 Approval/Identity 与顶层 UUID 用途闭集、Lease 全链 `acquire_lease` / `get_action_lease` / `begin_dispatch` / `release_or_expire_lease`、Stop owner 与真实远程验签）验证命令：

```text
export PATH="$HOME/.local/share/pnpm/bin:$PATH"
export TMPDIR="${TMPDIR:-$HOME/.cache/shittim-build-tmp}"
export CARGO_TARGET_DIR="${CARGO_TARGET_DIR:-$PWD/rust/target}"
cargo test --manifest-path rust/Cargo.toml -p kernel-kcp
cargo test --manifest-path rust/Cargo.toml --workspace
cargo clippy --manifest-path rust/Cargo.toml --workspace --all-targets -- -D warnings
cargo fmt --manifest-path rust/Cargo.toml --all -- --check
./scripts/check-schema.sh
node scripts/update-file-manifest.mjs --check
git diff --check
```
