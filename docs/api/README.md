# Shittim API 文档

## 当前状态

已有manifest v2、Rust生成类型与校验/哈希API、纯领域状态机、SQLite基座，以及不可连接的method-aware KCP preflight/dispatcher/handlers（active create v2 + ping/get + stop.activate/stop.status）。当前manifest共有`80 = 38 retained + 42 component-native`个Schema；ADR-0010已将未投产且被active v2完整替代的旧PolicyRule/PermissionDecision/ApprovalRecord三合同从source/manifest/ledger/generated/validation入口直接退役，无migration/compat。production MethodVersionBindings为IC §13.5八方法集。`domain-policy`直接消费PolicyRuleV2并一等支持五种confirmation mode；`kernel-authorization`负责四投影与真实远程验签。SQLite migration 0001–0011、v2-only Outbox、root/child/action/approval/stop_fence active producers、Action/PolicyRule/PD/Approval/Identity repositories、Lease/Stop Fence全链owner与child materializer已实现（含Approval撤销Lease与真实远程验签）。Provider真实远程验签接入、Publisher、versioned KCP poll、`task.list`/`event.subscribe`/`event.poll` handler仍未实现；里程碑为`V2InitialBuildActive`。当前没有可连接`agentd`、稳定网络endpoint或TypeScript客户端包。

本目录是中文导航，不是新的事实源。字段、状态机、错误和兼容规则以 `specs/` 及 `schemas/source` 为准。Core API 不暴露预埋的 Computer Use 方法；未来 Profile 的方法必须在正式 Schema、Catalog 和 Extension SDK Base 组合契约确立后再出现。`desktop-client` 是桌面客户端，不是 Computer Use。

## 文档

- [双仓库与持续文档维护](../REPOSITORY_MAINTENANCE.md)（主仓权威、文档主动更新、纯文档镜像同步与发布门禁）

- [Schema 生成与契约类型](schema-generation.md)
- [domain-task 内部 Rust API](domain-task.md)（非 KCP 外部 API）
- [domain-policy 内部 Rust API](domain-policy.md)（非 KCP 外部 API）
- [kernel-sqlite 内部 Rust API](kernel-sqlite.md)（migration 0001–0011；0010 为 Lease/Lock/Fence 持久化基座、0011 为 child materializer 基座；`acquire_lease` / `get_action_lease` / `begin_dispatch` / `release_or_expire_lease` / `activate_stop_fence` / `get_stop_fence` 全链 owner 与 `materialize_child_task` 已落地；非 KCP 外部 API）
- [KCP Value preflight 与注册式 dispatcher](kcp-preflight-dispatcher.md)（已实现、不可连接、非 SDK）
- [kernel-kcp typed application handler](kernel-kcp.md)（`system.ping`、`task.create`、`task.get`、`stop.activate`、`stop.status`；不可连接、非 SDK）
- [Task创建、Child materialization与repository硬合同](task-repository-contract.md)（root/Action/PD/Approval基础已落地；child materializer 已完成；无v1数据迁移）
- [Approval v2与PermissionDecision授权合同](approval-contract.md)（Schema/repositories已实现；invalidation 撤销 Lease 与远程 Ed25519 真实验签已落地）
- [AuditRecord版本合同](audit-record.md)（v2 Schema、root/permission/approval/identity producers已部分实现）
- [Kernel Control Protocol](kernel-control-protocol.md)（method-aware active生命周期合同；Schema/root types与切片3b method-aware runtime已落地，仍为不可连接库级路径）
- 首批正式事件索引：[Event Catalog](event-catalog.md)（Event v2 Schema/catalog/typed decode、v2-only Outbox 与 root/child/action/approval/stop_fence owner producers 已实现；Publisher、versioned poll 未实现）
- 稳定错误索引：[Error Catalog](error-catalog.md)（method lifecycle、v2业务/身份/CAS错误）

Core API 不预留 `snapshot`、`user_takeover` 或其他 Computer Use 专用方法；这些能力若未来实现，应通过 Optional Profile 的正式契约接入，而不是扩张 Core API。

## 权威来源

- KCP、对象和 Schema：[`../../specs/IMPLEMENTATION_CONTRACTS.md`](../../specs/IMPLEMENTATION_CONTRACTS.md)
- Event/Outbox / Task·Action 状态机：[`../../specs/CORE_ARCHITECTURE.md`](../../specs/CORE_ARCHITECTURE.md)
- Policy 与错误安全语义：[`../../specs/SECURITY_PRIVILEGE.md`](../../specs/SECURITY_PRIVILEGE.md)
- 自动化锚点：[`../../specs/CONFORMANCE.md`](../../specs/CONFORMANCE.md)

## 版本原则

KCP Envelope 使用 `protocol_version`；payload、Event payload 和持久对象使用 `schema_version`。第一版 KCP protocol 为 `1.0`。正式 Schema 使用 JSON Schema 2020-12，并通过 RFC 8785 canonical JSON 支撑稳定哈希与幂等等价比较。

`domain-task`只产出领域转换结果与事件意图；`domain-policy`直接消费`PolicyRuleV2`并产出非持久v2 decision draft/canonical input，以及纯TaskScope containment。`kernel-sqlite`拥有migration 0001–0011；0010 已建立 Lease/Resource Lock/Stop Fence 持久化基座与提交前关系闭包，Lease/Stop 命名 owner（acquire/dispatch/release/stop 激活与只读）已全部实现，0011 建立 child materializer 基座；`kernel-kcp`是不可连接的method-aware Value preflight、registration/dispatcher、五个handler与SQLite adapter；`V2InitialBuildActive`全谓词闭合待 `task.list`/`event.subscribe`/`event.poll`，server与SDK/client仍未完成。Task list/update仍未实现。

这些当前实现和计数均属于 Core；没有 Computer Use 预埋 API、Schema 或实现状态。
