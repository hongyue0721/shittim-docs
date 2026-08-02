# Event Catalog

> 状态徽章：**partial**（Event v2 八 Schema / catalog / typed decode 与 SQLite migration 0003–0010 / 统一 Outbox shape **已实现**；V2InitialBuildActive切片1a又落地AuditRecordV2、AuditAllocationV2与TaskCreationProvenanceV1，明确Audit/Event正交；切片3c 起 production Outbox 为 **v2-only**（`append_legacy_event_v1` / `LegacyV1` 已删，旧库 reinitialize-required）；root `task.created`、`action.state_changed`（causation=`action_transition`）、`approval.state_changed` 与 `stop_fence.activated`（global aggregate）active producer 已接入；child producer、Publisher、versioned KCP poll **未实现**）

## 唯一事实源

| 主题 | 锚点 |
|---|---|
| 首批正式 Event Catalog / payload 真值表 | [`IMPLEMENTATION_CONTRACTS.md` §5.6](../../specs/IMPLEMENTATION_CONTRACTS.md#56-首批正式-event-catalog) |
| Action transition authority / `ActionTransitionIntentV1` 边界 | [`IMPLEMENTATION_CONTRACTS.md` §6.14](../../specs/IMPLEMENTATION_CONTRACTS.md#614-action-transition-authority) |
| CausationRef / EventEnvelope 版本 | [`IMPLEMENTATION_CONTRACTS.md` §6.15](../../specs/IMPLEMENTATION_CONTRACTS.md#615-causationref-与-eventenvelope版本) |
| Event v2 八 Schema 精确 ID/source/DAG | [`IMPLEMENTATION_CONTRACTS.md` §13.6.2](../../specs/IMPLEMENTATION_CONTRACTS.md#1362-event-v2-八schema实现合同schemacompilergenerated已落地) |
| 决策与 Outbox/poll 边界 | [ADR-0008](../../adr/0008-active-event-v2与版本化统一outbox.md)（部分被 [ADR-0009](../../adr/0009-v2从零构建并取消v1数据迁移.md) supersede） |
| Outbox 事务语义 | [`CORE_ARCHITECTURE.md` §17](../../specs/CORE_ARCHITECTURE.md#17-事务边界与-sqlite-outbox) |
| 域状态 | [`../IMPLEMENTATION_MATRIX.md`](../IMPLEMENTATION_MATRIX.md) · [`../PROGRESS.md`](../PROGRESS.md) |

## 范围

本页索引 active Event Catalog、CausationRef v2、typed envelope、版本化统一 **v2-only** Outbox 与 retained `event.poll` v1 不得返回 v2 的边界。不复述八 Schema 字段、claimant 谓词、binding 常量或已删除的 legacy append API 形状。AuditRecordV2与AuditAllocationV2不是Event或Outbox envelope；challenge expiry只写Audit且不发`approval.state_changed`。切片4a已在`kernel-sqlite`落地`ActionTransitionIntentV1` repository 与 Action owner producer：命名 Policy/Approval 编排器通过 crate-internal mechanical commit 同事务 CAS Action 并 append `action.state_changed`，causation 精确为该 intent 的 `ActionTransitionRefV1`；`reconcile_intent` 只返回 prepared|committed|corrupt，不补造。切片4c已落地 Approval 三种 head mutation 的命名 owner 与 `approval.state_changed` producer；当前业务 producer 只缺 child。

`ApprovalEventAllocationV1` 已由 Approval repository 三种 head mutation 的命名 owner 正式消费。每个顶层 Approval/Policy 复合命令以唯一 transaction-bound typed collector 在写前校验 event/audit/record/Action transition allocations 与该路径实际读取、验证或消费的 persisted Task/Scope/Action/PD/request/challenge/evidence/credential UUID 用途；外层与嵌套阶段不允许各自校验后留下跨层碰撞。其 `causation_ref` 复用 `CausationRefV2`，`changed_at` 是 record/Audit/Event 的唯一业务时间。

## 导航

- [kernel-sqlite API](kernel-sqlite.md)
- [Error Catalog](error-catalog.md)
- [Approval 合同](approval-contract.md)
- [API 索引](README.md)
