# ADR-0010：Policy v2 终态与未投产旧合同直接退役

- 状态：accepted
- 日期：2026-07-21
- 实现状态徽章：**implemented** — Policy matcher 已直接消费 active v2 合同；三项未投产旧合同已从 source、manifest、retained ledger、generated artifacts 与验证入口退役。当前进度见 [`docs/IMPLEMENTATION_MATRIX.md`](../docs/IMPLEMENTATION_MATRIX.md) 与 [`docs/PROGRESS.md`](../docs/PROGRESS.md)。

## 背景

Shittim 尚未向真实用户交付可连接 runtime，也没有 production Policy 数据。ADR-0009 已将存储定义为 v2 fresh baseline，并明确不承担未投产 v1 业务数据迁移。此前保留的 `PolicyRule`、`PermissionDecision`、`ApprovalRecord` 三项旧合同已分别被 active `PolicyRuleV2`、`PermissionDecisionV2`、`ApprovalRecordV2` 完整替代；production Schema source 中也没有其它对象通过 `$ref` 依赖它们。

继续保留这三项会产生两套错误事实：matcher 需要 v2→v1 转换、`remote_signature` 需要 rule-ID side table 或 Generic probe、decision/mode 需要 adapter，生成目录仍公开已经没有生命周期的旧类型。它们不是兼容资产，而是未投产设计遗留。

同时，项目存在大量首次正式合同以 `V1` 命名，例如 `ConfirmationModeV1`、`VerificationResultV1`、`ActionTransitionIntentV1` 与多项 projection/allocation。版本后缀只标识 wire 版本，不表示 legacy 生命周期；生命周期必须逐合同、逐引用方判断。

## 决策

### 1. Policy matcher 以 v2 为唯一生产合同

- `domain-policy::evaluate_policy` 直接接收 `PolicyRuleV2`。
- `PermissionDecisionDraft.decision` 直接使用 `PermissionDecisionV2Decision`。
- `ConfirmationModeV1` 五值 `generic | local | system_authentication | remote_signature | plan_revision` 全部是一等 matcher 输入。
- `remote_signature` 正常产生 `RequireRemoteSignature`，并与其它确认模式一样参与 applicability、priority、specificity、winner-only rate-limit 与 fallback。
- 删除 v2→v1 转换、Generic probe、remote rule-ID side table、mode/decision adapter，以及依赖错误消息识别普通未匹配的路径。
- Approval/Identity repository 或 Provider 真实验签的可执行性属于后续事实门；它们不得反向改变 matcher 对合法 PolicyRule 的确定性求值。

### 2. 三项未投产旧合同直接退役

精确删除：

- `schemas/source/policy/policy_rule.v1.json`
- `schemas/source/policy/permission_decision.v1.json`
- `schemas/source/policy/approval_record.v1.json`

退役同时传播到 production manifest entries、policy component `retained_ids`、`schemas/fixtures/manifest/retained_ids.v1.json`、generated Rust types/catalog/validators/bindings，以及只验证这三项的 examples/imports/tests/samples。不得保留 legacy-validation、legacy-read、migration、adapter 或隐藏 validator 入口。

这是 fresh baseline 的直接删除：没有 production 数据、没有已发布 SDK consumer、没有迁移输入，也没有兼容窗口。发现旧开发代码引用这些类型时应编译失败并迁到 active v2，不提供 alias。

### 3. 精确保留边界

本 ADR 不按版本后缀批量删除。下列合同继续保留并按各自引用方生命周期 active：

- `ConfirmationModeV1`
- `VerificationResultV1`
- `ActionTransitionIntentV1`
- `ActionRequest` v1 与 active `ActionRequestV2`
- KCP/Event v1 retained 合同及其它仍被 active 合同引用的 v1 对象

未来任何退役都必须同时证明：已有 active replacement 完整覆盖、没有 production `$ref` consumer、没有已投产数据/SDK 兼容义务，并通过单独决策明确边界。

### 4. 当前 Schema 基线

退役后 production manifest 精确为：

```text
80 = 38 retained + 42 component-native
```

component-native 数量不变。retained ledger 从历史迁移时的 41 项缩减为当前 38 项；ledger 文件名中的 `v1` 是 fixture 格式/基线版本，不表示其中每个合同都是 legacy。

历史实施章节可保留当时的 41/83 计数，但必须明确标注为当时快照，不能再把它们写成当前 production 事实或当前硬门。

## 兼容与迁移

无 migration、无 compatibility adapter、无双写、无 alias、无旧 Schema validation。原因是项目处于未投产 fresh baseline；为不存在的 consumer 维护兼容层会制造第二套 production 语义，并使 `remote_signature` 再次降格。

## 验收

1. `domain-policy` public matcher 仅接受 `PolicyRuleV2`，draft decision 为 `PermissionDecisionV2Decision`；五种 confirmation mode 均有 matcher 与 SQLite 编排测试。
2. `remote_signature` 参与正常排序、specificity、winner-only rate-limit 和 exhausted fallback，不依赖 side table/probe。
3. 三项旧 source、manifest/ledger/generated/validation/example 入口全部消失；全仓搜索旧三个 schema ID/source path 无命中。
4. production manifest 精确 80，retained 精确 38，component-native 精确 42；保留合同边界有回归断言。
5. schema-tool 的通用 Rust projection 测试迁到 active `PolicyRuleV2#/$defs/actor_entry_point` 或等价 test-only synthetic probe，不因退役降低覆盖。
6. source→generated tree、Schema gate、四个受影响 crate 的 test/clippy、fmt 与 `git diff --check` 全部通过。
