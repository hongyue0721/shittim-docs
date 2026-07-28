# Shittim 实现矩阵

> 本矩阵只汇总状态，不取代 `specs/` 中的唯一事实源。

| 领域 | 规范状态 | Schema | 实现 | 自动化测试 | 备注 |
|---|---|---|---|---|---|
| Task/Action状态机 | active合同：root v2/child Action与Approval v2；`V2InitialBuildActive`（ADR-0009） | 首批TaskCreate/Child proposal/allocation、ActionRequestV2、ActionTransitionIntentV1、Approval核心五Schema及身份/证据八Schema/generated root types已落地 | root TaskCreate v2 repository、Action current-snapshot/transition/`action.state_changed`、PolicyRule/PD评估编排（PD 原子绑定至 migration 0009）、Approval/Identity/`approval.state_changed`均已落地；`domain-task` Lease release effect 与 migration 0010 Lease/Resource Lock/Stop Fence 持久化基座已落地并独立终验 GO，但 acquire/dispatch/release/Stop owner 尚未实现，`approved→leased→in_flight→completed` 仍走不通；Action 状态事件权威已收敛：无公开裸 CAS 入口，`cas_transition_for_intent` 为私有机械 primitive，`append_active_event_v2` 与 `PendingActiveEventV2` 已 crate-private，每个成功状态边恰好一条 `action.state_changed` | Schema/typed conformance + root v2 + Action CAS/intent + PD/PolicyRule/五模式评估 + Approval/Identity tests | 不从v1迁数据；旧库reinitialize-required |
| Child Task authority | ADR-0006 accepted（§7 migration/provenance被ADR-0009 supersede）；fixture/task-creation owner合同已闭合 | ChildTaskProposal/NormalizedChild/Allocation与ChildTaskDeltaProjection Schema/generated types已落地 | `kernel-task-creation`负责proposal/allocation；`kernel-authorization`负责delta/material/observation/subject projection；Approval/Identity基础已落地，active materializer未实现 | Schema conformance、pure crate hash/normalization/negative tests、official task fixtures及child-delta/subject official projection fixtures覆盖 | 无legacy direct-child provenance/migration |
| Approval/身份/失效 | ADR-0007 accepted；旧三Policy合同退役见ADR-0010 | ApprovalRecordV2、PermissionDecisionV2、PolicyRuleV2、SubjectProjectionV1、ApprovalEventAllocationV1与身份/证据八Schema已落地；被替代旧三合同已直接退役 | **安全闭环接近完成，整体仍待最终验收**：operation Approval 初始请求与评估/PD/Action/Approval 同一事务；subject 与 usable resolution proof 均由仓储从真实 current facts 派生；每个顶层 Approval/Policy 复合命令只创建一个 transaction-bound typed UUID collector，写前覆盖 command allocations 与该路径实际读取、验证或消费的 persisted Task/Scope/Action/PD/PolicyRule/request/challenge/evidence/credential，嵌套阶段共享该 collector；事件 mode 从原始 request 回读；denied 审计 blocked+PD/policy context；local/system 证据完整绑定；system/remote Challenge 消费与 resolution/head/Event/Audit 同事务，过期只提交 ChallengeExpired+identity Audit。**未闭合**：invalidation/replacement 的 Action/Lease 关联撤销、remote_signature 真实验签 | 真实 Task/Action/current PD/subject projection/material fingerprint fixtures；typed code + 完整零写入；跨层 UUID collision、新/current PD 生命周期、confirmation-mode、denied-audit、错误归属过期 Challenge、local actor mismatch、system/remote offset 与 Remote fail-closed rollback 专测 | 不得称 4c 完成；原 11 项 High 剩 2 项；`remote_signature` 以 `ApprovalRequired` 失败关闭直至真实验签落地 |
| Recovery/Verification | v1合同已消歧；v2 producer引用规则已定义 | Candidate/Attempt/Verification Schema与生成类型均为v1 | v1验证摘要与 retry_original 合法性 | v1 completed/failed/unknown/retry 测试 | 其它恢复候选只做枚举层接受，不代表授权或执行 |
| Policy matcher | Policy v2终态（ADR-0010）；typed NotMatched/Error与Default Allow闭合 | PolicyRuleV2、PermissionDecisionV2、ApprovalRecordV2及四projection/身份证据Schema已落地；旧三合同退役 | `domain-policy::evaluate_policy`直接消费`PolicyRuleV2`并输出`PermissionDecisionV2Decision`；五种mode一等支持；`kernel-sqlite`直接传v2 heads并持久化PD；无转换/probe/side table/adapter | URI/specificity/五模式/remote_signature优先级与winner-only rate-limit/fallback/rollback/typed error tests | Provider真实验签不改变matcher语义 |
| ContentOrigin/Actor/EntryPoint | v1已实现；active ContentOrigin v2 carrier合同已定义 | InputContentOriginV1、InputTaskScopeV1与stored ContentOriginV2已落地 | root v2 写路径持久化 ContentOriginV2；v1 ContentOrigin repository write 已删除（0005 drop 死表）；旧库 reinitialize-required | Input边界 + stored v2 exact wire/typed/JCS + root v2 origin readback | Actor/EntryPoint仍复用retained v1；v2 child carrier=action由未来producer证明 |
| PermissionDecision | active v2完整projection/lease合同；旧合同按ADR-0010退役 | PermissionDecisionV2 Schema/生成类型已落地 | `domain-policy`生成v2 decision draft；`kernel-sqlite`仅由评估编排内部 stage→bind并公开get/current/validate；migration 0009 `action_permission_decision_heads` 为每 Action 当前决定的唯一权威（CAS/绑定守卫/禁删触发器）；确认路径保持 pending，放行/拒绝在同一次状态 CAS 内投影 PD ref，一次评估只推进一次 revision | Default Allow/五mode/key params + sqlite PD连续性/双向一致/head CAS/未绑定暂存拒绝提交/并发首绑定/评估矩阵 tests | Approval subject 已强制绑定 Action 当前 PD；lease仍属后续切片 |
| AuditRecord | v1 legacy + active v2义务已定义 | AuditRecordV2与Schema-validated AuditAllocationV2已落地；v1 Schema/fixture retained | root task.create v2 写入 AuditRecordV2；评估编排写入 `permission.evaluated` 并校验 policy_context↔PD；v1 Store/write/producer 已删除（0005 drop 死表）；旧库 reinitialize-required | exact v2 wire + root v2 creation_recorded + permission.evaluated 跨对象一致 tests | rollback 权威投影 / Provider·ModelCall 等其它 producer 仍未实现 |
| Event/SQLite Outbox | ADR-0008 accepted（§7/§8 legacy production API被ADR-0009 supersede）；v2八Schema与统一Outbox shape已闭合 | Event v2八Schema + retained v1 已进manifest/generated；catalog/typed decode已落地 | migration 0003–0010、descriptor v1、**v2-only** append/read、严格stored decoder、delivery gate与savepoint poison已实现；root `task.created` 与 Action `action.state_changed`（causation=`action_transition`）有 owner producer；`append_legacy_event_v1`/`LegacyV1`已删；旧库 reinitialize-required；Publisher未实现 | descriptor/ledger/空表升级、reinitialize-required、v2-only事件、root/action causation/correlation/readback、intent commit 回滚不占号、stored corruption与poison tests已覆盖 | retained poll v1不能返回v2；Publisher/poll不在V2InitialBuildActive |
| KCP Envelope / Value preflight | active合同使用Command/Query Envelope V2结构门 + MethodVersionBinding业务门；IC §13.5/§13.7为V2InitialBuildActive口径 | 首批12项source/entries/generated types已落地；production bindings=§13.5八方法集（切片3a）；retained v1 envelopes bytes未改 | schema-tool library + production V2 authority + bindings已生效；**kernel-kcp method-aware preflight 已消费 bindings（切片3b）** | business-v2 schema conformance、production gate/selector、runtime method-aware 矩阵（create v2 Active / create v1 unsupported / 其余七方法 v1 Active / 优先级）已覆盖 | active task.create只接受2；五方法缺handler，禁止server |
| KCP首批八方法 | Catalog保留；task.create active语义升v2 root-only | v1 8组Schema保留；TaskCreate Request/Response v2与Envelope V2已生成；production bindings=§13.5八方法集 | **active create v2 + ping/get 三方法 runtime**；v1 create production 入口与 sqlite legacy write 已删；五方法 KnownCatalogMethodNotImplemented | method-aware preflight/dispatcher/handler + sqlite v2 integration tests | 五方法缺handler；§13.7 待 Lease/Stop Fence、child materializer、真实远程验签与 handler 闭合 |
| KCP typed application handler / dispatcher | §5.10/§5.11 三步 API、注册集合与无损路由；task.create variant=v2 | 复用 Envelope V2 结构门 + active create/ping/get response Schema | private-state `TypedCatalogRequest`/`RegisteredRequest`、`TaskCreateCommandRequestV2`、borrowing `TypedDispatcher`、v2 handlers/ports、`SqliteTaskBackend`→`create_root_task_v2`、生产 clock/ID | handler 七UUID/root-only/幂等/deadline/intent、dispatcher Created intent、adapter store error 闭集、sqlite active event/audit/provenance 绑定 | 只能库级不可连接；五方法正式 handler 前禁止 server |
| 首批active事件 | 五类合同与ADR-0008/0009权威闭合 | 八Schema + retained三payload + Envelope v1/v2 已落地 | v2-only Outbox append API已实现；root `task.created`、`action.state_changed` 与 `approval.state_changed` 均有 owner producer；child producer与Publisher未实现 | v2-only五类active、action/approval causation、aggregate mismatch及corrupt relation/reinitialize-required覆盖 | storage API不等于 producer；当前业务 producer 只缺 child，Publisher/poll 不在本里程碑 |
| KCP 本地传输 | ADR accepted；受 typed-only 阶段门约束 | 不适用 | 未开始 | 未开始 | Unix Socket / Windows Named Pipe；本批不拍 path/frame 新事实、不允许可连接 server |
| Schema生成链 | ADR-0002/0010 accepted；manifest v2/walker/transaction、KCP/Event authority与bindings生成已实现 | production=`80 = 38 retained + 42 component-native`；production bindings=§13.5八方法集 | schema-tool restricted profile与生成链已落地；旧三Policy source/manifest/ledger/generated入口已退役 | 生成链/fixtures/Event claimant/binding gate + exact 80/38/42 + 旧ID消失/active v2保留 | 历史切片曾为83/41；当前硬门为80/38 |
| Rust workspace | ADR accepted；`kernel-task-creation`与`kernel-authorization`已加入workspace | 不适用 | kernel-contracts、schema-tool、domain-task、domain-policy、kernel-sqlite、kernel-kcp、kernel-task-creation、kernel-authorization | fmt/clippy/workspace test + pure crate conformance + official projection fixtures/harness/oracle | URI复用domain-policy唯一实现，全部事实caller typed注入；两crate均不读repo/不写存储 |
| TypeScript workspace | ADR accepted | 尚无 TS 生成物 | 仅根零依赖基座（`package.json` / `pnpm-workspace.yaml` / lockfile / `check:toolchain` / `update-file-manifest` / `sync-docs-repository`）；无 `ts/*` 包 | `pnpm run check:toolchain`；`pnpm run test:file-manifest` / `check:file-manifest`；`pnpm run test:docs-repository` / `check:docs-repository` / `sync:docs-repository`；统一门 `PATH`+`./scripts/check-schema.sh`（先 Node 硬门） | Node exact 24.18.0、pnpm exact 11.3.0；入口 `~/.local/share/pnpm/node`；Corepack 不可用；无 deps/SDK/client；无跨平台 npm `check:all`；`FILE_MANIFEST` 只列 Git Markdown source set（路径严格 UTF-8 fail closed）；文档镜像同步工具已 library/CLI implemented，不进入统一门默认步骤 |
| Desktop client | 方向已定义 | 未开始 | 未开始 | 未开始 | 将使用 Tauri/React/AntD，蓝白配色 |
| Extension SDK Base | contract-only；统一 SDK 的唯一规范边界已定义 | 未开始 | 未开始：没有 library、composition、public API 或 SDK 包 | 未开始 | **基础产品 Core 阻塞项**；不得把规范、根 Node 工作区或未来 `desktop-client` 宣传为 SDK 实现 |
| Optional Computer Use Profile | contract-only；桌面专属事实源为 `COMPUTER_USE.md` | 未开始：没有专用 Schema | 未开始：没有 crate、SDK composition、Provider 或 real-platform 能力 | 未开始：没有 Provider/真机测试 | 不阻塞 Core 完成；`desktop-client` 不等同于 Computer Use；启用后仍投影 Task/Policy/Scope/Lease/Stop Fence/Audit |
| Provider/平台能力 | 仅接口边界 | 未开始 | 未开始 | 未开始 | 不伪造支持；Computer Use Provider 属于 optional Profile，不代表 Extension SDK Base 已完成 |

## 状态含义

- **ProfileClaim maturity**：`contract-only | schema/SDK | composition | provider contract | real-platform`；按精确 claim id 独立记录。
- **distribution assertion**：与 maturity 正交的布尔对外声明事实；不是成熟度，不能自动升级能力。
- **library**：可复用实现已存在；不自动表示已完成运行时组合或对外发布。
- **composition**：实现已接入其宿主并可按契约协作；不自动表示已公开或已在真实平台验证。
- **public / SDK**：稳定对外入口或可安装 SDK 包已交付；不自动表示所有 Profile 或平台均已支持。
- **real-platform**：目标真实平台/Provider 的集成验证已完成；模拟、Schema 或纯领域测试不能替代。
- **纯领域实现**：只计算规则和意图，不拥有持久化或外部副作用。
- **类型与运行时校验**：有 Schema/生成类型，不代表业务状态所有者已实现。
- **未开始**：没有对应实现或真实能力。

## 相关入口

- [进度](PROGRESS.md)
- [双仓库与持续文档维护](REPOSITORY_MAINTENANCE.md)
- [API 文档](api/README.md)
- [domain-task API](api/domain-task.md)
- [domain-policy API](api/domain-policy.md)
- [kernel-sqlite API](api/kernel-sqlite.md)
- [KCP Value preflight 与注册式 dispatcher](api/kcp-preflight-dispatcher.md)
- [kernel-kcp typed handler](api/kernel-kcp.md)
- [Task repository 创建契约](api/task-repository-contract.md)
- [AuditRecord版本合同](api/audit-record.md)
- [Event Catalog](api/event-catalog.md)
- [Error Catalog](api/error-catalog.md)
- [Approval v2合同](api/approval-contract.md)
- [Schema 生成](api/schema-generation.md)
- [SDK 文档](sdk/extension-sdk.md)
- [ADR 索引](../adr/README.md)
