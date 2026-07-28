# Shittim 实现进度

> 状态日期：2026-07-28（`V2InitialBuildActive`：Policy v2、Action/PermissionDecision、Approval/Identity 与顶层 UUID 用途闭集已落地；Action Lease 领域释放效果和 migration 0010 Lease/Resource Lock/Stop Fence 持久化基座已完成并独立终验 GO。切片 4c 原 11 项 High 已闭合 9 项，剩 Lease 关联撤销与真实远程验签 2 项；Lease/Stop 命名 owner、child materializer 与 §13.7 五方法 handler 仍未实现。）
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
| 4c | Approval / Identity repository 与安全闭环 | **部分完成（9/11 High）** |
| 5 | child Action materializer | 未开始 |
| 6 | §13.7 谓词闭合（依赖 4–5 + 其余 active producers） | 未开始 |

**已实现（代码/Schema 事实）**

- 规范与工程基线：Freedom-first / Kernel Owns Reality 合同、Apache-2.0、双仓同步 library/CLI、Node/pnpm 零依赖根基座（exact Node 24.18.0 / pnpm 11.3.0）、统一门 `scripts/check-schema.sh`。
- Schema/Rust 契约：Rust workspace、Draft 2020-12 + manifest v2（production=`80 = 38 retained + 42 component-native`）、`schema-tool` 单 root transaction / target-scoped IR / TaggedUnion / string enum `ALL` / string-array const / RFC 8785；production `METHOD_VERSION_BINDINGS` 为 IC §13.5 八方法集。ADR-0010已直接退役未投产旧PolicyRule/PermissionDecision/ApprovalRecord三合同，无migration/compat/旧validation入口；`V1`后缀本身不等于legacy。
- 纯领域：`domain-task`（Task/Action 状态图、revision/plan_version）、`domain-policy`（直接消费`PolicyRuleV2`、URI/glob/Default Allow/五种confirmation mode/winner-only rate-limit，Stop Fence/Recovery独立Blocked）。
- 持久化：`kernel-sqlite` migration 0001–0010；AuditRecord **v2**；strict Task/TaskScope/ContentOrigin(v2)读；版本化 **v2-only** Outbox + 严格stored decoder + savepoint poison；active root TaskCreate v2 repository；Action current-snapshot/transition/`action.state_changed`；PolicyRuleV2/PermissionDecisionV2/评估编排；Approval current-head CAS/`approval.state_changed`/Identity repositories；migration 0010 已建立 `action_leases` / `action_resource_locks` / `stop_fence`、CAS/不可变守卫、旧执行事实 reinitialize-required 校验与统一事务提交前关系闭包；legacy TaskCreate/Audit/Outbox v1 write已删。
- migration 0009 `action_permission_decision_heads`：每个 Action 当前 PermissionDecision 的唯一持久权威，配 CAS/绑定守卫/禁删触发器；评估经受事务生命周期约束的 Staged→Bound 协议绑定，确认路径保持 pending，放行与拒绝在同一次状态 CAS 内投影 `permission_decision_ref`，一次评估只推进一次 Action revision。
- KCP 库级：`kernel-kcp` method-aware Value preflight、三方法 registration/dispatcher/handler（`system.ping` / **active root `task.create` v2** / `task.get`）与 SQLite adapter→`create_root_task_v2`；不可连接，无 bytes/frame/server。
- ADR-0006 首批：12 business-v2 Schema + `kernel-task-creation` pure library + official fixtures/harness + schema-tool strict pointer CLI。
- ADR-0008 前两段：Event v2 八 Schema、`EventTypeBinding`/active·legacy catalog、typed EventEnvelope v1/v2、migration 0003 descriptor v1 与统一 Outbox shape（切片3c 起 production v2-only）。
- V2InitialBuildActive切片1a–1c-ii：root持久对象、Action/child授权、授权核心、身份/挑战/证据Schema与pure crate已落地；该阶段历史快照为manifest=83，ADR-0010退役三项旧Policy合同后当前为80。
- V2InitialBuildActive切片2：migration 0004（`content_origins_v2`、`task_creation_provenances`、`audit_records_v2`、`root_task_create_idempotency_v2` + tasks/scope FK 重建以允许 v2 origin）；`WriteTransaction::create_root_task_v2` 单事务写 Origin/Scope/Task/Provenance/Audit/idempotency/Event；全闭包 canonical readback（Created/Replayed 共用）；幂等重放/冲突；回滚不占号；与 v1 表互不污染。
- V2InitialBuildActive切片3a：production `method_version_bindings` 精确八方法（`task.create` active=[2]/legacy=[1]，其余 active=[1]）；`validate_production_manifest_stage` 要求 Envelope-derived 完整集 + IC §13.5 lifecycle；generated `METHOD_VERSION_BINDINGS` 非空且 `select_request_version` 可用。
- V2InitialBuildActive切片3b：kernel-kcp preflight 按 (family, method, payload.schema_version) 调 `select_request_version`；V2 Envelope 结构验证 + active payload Schema；`task.create` v2 Accepted/Registered，v1 → `unsupported_schema_version`；handler 七 UUID（含 CreationProvenance）+ root-only 检查 + `TaskCreateResponseV2`；adapter 映射 `create_root_task_v2`；删除 kcp 侧 v1 create handler/adapter/ports 路径。
- V2InitialBuildActive切片3c：删除 `create_task`/`TaskCreateCommand`/`prepare_legacy_v1_create`、AuditRecord v1 write、`append_legacy_event_v1`/`PendingLegacyEventV1`/`StoredEventEnvelope::LegacyV1`；Outbox decoder 对 schema_version=1 → `stored_data_invalid`；`SqliteStore::open` 后 `reject_legacy_v1_business_data`；migration 0003 transform 对非空 legacy Outbox 直接 reinitialize-required；migration 0005 在空表前提下 drop dead v1 表。
- V2InitialBuildActive切片4a：migration 0006（`actions` + `action_transition_intents`）；`insert_pending_action` / `get_action`（公开）；intent durable/recovery surface + crate-internal mechanical commit 同事务 CAS+`action.state_changed`（causation=`action_transition`）+ reconcile 三态；状态事件由命名业务 owner 进入唯一机械权威，无公开裸状态迁移入口；需 lease effects 的边 fail closed；domain-task 边合法性与 evidence 门；sequence/position 失败不占号。
- V2InitialBuildActive切片4b：migration 0007（`policy_set_metadata` bootstrap revision 0、`policy_rules`、`permission_decisions`）；PolicyRule append-only revision + global set counter；PD immutable append（连续 decision_revision）+ Action ref 双向校验；`evaluate_action_permission` 单事务 matcher→指纹→PD→`permission.evaluated` Audit→Action CAS（allow/deny/require_* deferred，无 Approval 创建）；rate-limit 同事务消费与回滚；material/observation 双指纹真实重算。
- V2InitialBuildActive切片4c：migration 0008（`approval_records`、`approval_chain_heads`、`identity_credentials`、`identity_challenges`、`identity_evidence`）；三种 Approval head mutation 由命名 owner 独占：初始 operation request 只经 `evaluate_action_permission_and_create_approval` 与 Policy evaluation/PD/Action binding 同一 savepoint 创建，无 generic 独立 `append_request`；`resolve` 处理 expected-head/mode evidence，`resolve_approval_and_commit_action` 可在同一 savepoint 重新派生 usable proof 并驱动 `pending→approved`；`invalidate_and_optionally_replace` 原子推进 replacement。每成功 head 变化恰好一条 `approval.state_changed` + 对应 `approval.requested|resolved|invalidated` Audit；CAS 冲突/replay 不产 Event；Identity credential register/rotate/revoke、challenge issue/consume/expire（终态不可逆、expire 只写 `identity.challenge_expired` Audit 不发 Approval event）、local/system evidence immutable。

**未实现（不得宣称完成）**

- **Lease / Stop Fence 仅完成持久化基座，命名 owner 尚未实现**：`domain-task` 已闭合 Lease 生命周期的 typed effect；migration 0010 已建立 Action Lease、Resource Lock、Stop Fence 三表及 descriptor、关系守卫、`max_uses=1` 生成合同与统一 COMMIT 前关系闭包，半 stage/delete/Fence、锁集合缺失/多余与 REPLACE 绕过均失败关闭。但仍无 `acquire_lease` / `begin_dispatch` / `release_or_expire_lease` / `activate_stop_fence` / 严格读取 owner，SQLite 仍不会消费 typed Lease effect，所以 `approved → leased → in_flight → completed` 当前仍走不通。单元 2 开码前必须先闭合 `ActionTransitionIntentV1.execution_generation` 无法同时表达 acquire 前后 generation 的合同冲突，以及 Stop 动态 allocation 中 causation 必须别名到持久 intent 而非独立分配的问题。
- **切片 4c 安全闭环接近完成，仍未最终验收**：原 11 项 High 已闭合 9 项。除既有 operation Approval 当前 PD 绑定、Challenge 过期 CAS/审计和 identity 审计外，本轮又闭合 5 项：每个顶层 Approval/Policy 复合命令用唯一 transaction-bound typed collector，在任何业务写入前覆盖 command allocations 与该路径实际读取、验证或消费的 persisted Task/Scope/Action/PD/PolicyRule/request/challenge/evidence/credential UUID 用途，嵌套 prepare/apply 禁止另建 collector；事件 payload 从权威原始 request 回读真实 confirmation mode；denied resolution 审计为 `outcome=blocked`、稳定 reason code 且保留 operation PD/policy context；local/system 证据校验 actor/entry/time/challenge/request/chain/task/subject/material 绑定；system/remote Challenge 消费与 Approval resolution/head/Event/Audit 通过 `consume_challenge_with_binding` 同一事务提交，过期返回 typed `ChallengeExpired` 而不写 resolution。
- **仍未闭合的 4c High（2 项）**：单独调用 `resolve` / `invalidate_and_optionally_replace` 仍未完整更新 Action 关联并撤销受影响 Lease（撤销 Lease 必须等 Lease 持久化；`resolve_approval_and_commit_action` 已能基于重新派生 proof 同事务驱动 `pending→approved`）；`remote_signature` 尚无真实密码学验签。
- **`remote_signature` 如实失败关闭**：远程 Challenge 的过期/消费与 resolution 已有原子事务闭包，但无密码学验签时，远程决议一律不得作为批准 Action 的授权（`ApprovalRequired`）；待可信 `RemoteSignatureVerifier`（首个实现为 Ed25519 RFC8032）落地后再开放。
- child Action materializer；Action闭集其余写方法（lease acquire/dispatch/release、child completion、recovery list；policy binding 已实现）；真实远程验签 Provider 边界。
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
- [~] 切片 4c：Approval/Identity repository（原 11 项 High 已闭合 9 项；剩 Lease 关联撤销与真实远程验签 2 项，不得称完成）
- [x] 收尾：Approval 当前绑定 PD 门 / Action 唯一状态事件权威 / post-Outbox 全量回滚证明
- [~] Lease + Stop Fence：领域 Lease release effect 与 migration 0010 持久化基座已完成；generation/intent 合同、acquire/dispatch/release/Stop owners 与 Approval Lease 撤销待完成
- [ ] 切片 5：child materializer
- [ ] 切片 6：§13.7 谓词闭合（child/Action/PD/Approval + 其余 producers）
- [ ] active Event business producer：child（root已在切片2接入，action已在4a接入，approval已在4c接入）
- [ ] 五方法正式 handler；可连接 KCP server/client；`agentd`
- [ ] 其它 Command 幂等与乐观锁；Task list cursor ADR 拍板后实现
- [ ] Publisher + versioned KCP poll（**后续里程碑**，不在 V2InitialBuildActive）
- [ ] Extension SDK Base → `schema/SDK`；TS 包 / SDK client / `agent-runtime`
- [ ] optional Computer Use Profile；桌面客户端；Provider/Memory/Initiative/Broker
- [ ] CONFORMANCE 全量 BASE + 声明 Profile 套件

## 当前阻塞

- kcp runtime与sqlite repository已切到active create v2 / Outbox v2-only；Action/`action.state_changed`（4a）、PD/PolicyRule/评估编排（4b）、Approval/Identity/`approval.state_changed`（4c 结构层）已落地；五方法无handler，禁止启动server。
- §13.7 谓词 4 当前为 **false**：migration 0010 的表和 COMMIT 前关系闭包已就位，但 repository owner 尚未组成可执行链。剩余执行前置是 Lease/Stop owners、child materializer与真实远程验签。
- legacy v1 repository 的 Delegation 正向路径未实现（非 null 固定 not found）；active v2 repository 同样在 Delegation authority 未落地前 fail-closed 返回 `delegation_not_found`。
- Task list cursor 编码须先 ADR/API 拍板。
- Audit：`permission.evaluated` 已在评估编排同事务校验 policy_context 与 PD 字段相等；仍缺 rollback 权威投影、Provider/ModelCall 一致性与其它业务 producer。
- Extension SDK Base / Computer Use 仍为 `contract-only`；不得把规范或根 Node 工作区冒充 SDK 实现。
- 真实 Provider/Channel/Privilege Broker 需要后续真实环境；当前无伪造支持。
- 默认 PATH 可能不是 Node 24.18.0；须显式 `~/.local/share/pnpm`。

## 下一步

1. 在实现 `acquire_lease` 前先修订并验证 generation/intent 合同：现有 `ActionTransitionIntentV1.execution_generation` 同时被当作 pre-CAS 与 post-CAS generation，而 acquire 必须 `G→G+1`；不得用例外条件绕过。同期收敛 Stop 动态 allocation：causation 必须由持久 intent 派生，只作为同一事实 alias，不另分配 UUID。
2. 在合同闭合后实现 `acquire_lease` / `get_action_lease`，再实现 `begin_dispatch` / `release_or_expire_lease` 消费 typed effect，随后实现 Stop owner 与 Approval invalidation 同事务撤销 Lease（4c 清零）。
3. 引入可信 `RemoteSignatureVerifier` 边界（首个实现为 Ed25519 RFC8032 pure mode，纯 crypto 放 `kernel-authorization`），解除 `remote_signature` 的失败关闭并完成 4c 最终验收。
4. 切片5：实现child materializer并接入child `task.created` producer（mapping 表用 migration 0011；child creation Audit 的 actor/entry_point 取 typed execution context）。
5. 切片6：在切片5完成后闭合§13.7全部谓词。
6. **之后**再做 Publisher、versioned KCP poll、剩余 Catalog handlers、可连接 server，以及 Extension SDK Base 与 TypeScript/client。

## 最近验证

当前代码基线（含 Approval/Identity 与顶层 UUID 用途闭集）验证命令：

```text
export PATH="$HOME/.local/share/pnpm:$PATH"
export TMPDIR=/mnt/data/shittim-build-tmp
export CARGO_TARGET_DIR=/mnt/data/shittim-cargo-target
cargo test --manifest-path rust/Cargo.toml -p kernel-kcp
cargo test --manifest-path rust/Cargo.toml --workspace
cargo clippy --manifest-path rust/Cargo.toml --workspace --all-targets -- -D warnings
cargo fmt --manifest-path rust/Cargo.toml --all -- --check
./scripts/check-schema.sh
node scripts/update-file-manifest.mjs --check
git diff --check
```
