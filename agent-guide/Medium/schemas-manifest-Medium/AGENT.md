# schemas-manifest-Medium — Schema 单一事实源与 manifest

> 难度：**Medium**（改 Schema 影响全链生成物，需谨慎）。进入本文件夹即表示任务落在 `schemas/`（source / manifest / fixtures）。

## 目标

让 `schemas/` 保持「唯一人工源」地位：source + manifest 一致、fixtures 是官方 oracle、generated 只由生成链产出。完成标准 = 变更走完整生成链、数量硬门、conformance 锚点同步。

## 开工前必读

1. 根 `AGENT.md`（§3 Schema 编译铁律）；`Difficult/schema-tool-Difficult/AGENT.md`；
2. `adr/0002`（生成与兼容策略）、`adr/0005`（Profile 边界）、`adr/0009`（v2 fresh baseline）、`adr/0010`（旧合同退役）、`specs/IMPLEMENTATION_CONTRACTS.md` §13。

## 边界与职责

- `source/`：全部 production JSON Schema（Draft 2020-12）人工源，按组件组织；精确数量与 retained/component-native 分布以 `schemas/manifest.json`、MATRIX/PROGRESS 为准。
- `manifest.json`：Schema 台账（组件归属、retained/component-native 语义、MethodVersionBindings、Event 绑定），entry `$id` 命名空间权威。
- `fixtures/`：官方对账 fixtures（normalization/hash/projection/allocation oracle），版本化命名。
- `examples/`：示例，不是 conformance oracle。

## 硬性规则

1. **Schema 是唯一人工源**：类型/协议变化只改本目录 + manifest，然后走 `schema-tool` 生成链；禁止手改 `rust/crates/kernel-contracts/src/generated/`（ADR-0002）。
2. 每个 source 文件必须有 manifest 条目；`$id` 必须落在对应组件的 `id_base` 命名空间；禁止孤儿 Schema。
3. **retained vs component-native**：retained 是跨版本保留的历史合同（只用于验证/兼容判定），component-native 是当前组件私有；改动 retained 语义等同合同变更，需要 ADR/spec 依据。
4. **Core 不预埋 Profile 私有对象**（ADR-0005）：Computer Use 等可选 Profile 不得在 Core 侧出现；Profile claim 未落地前连 Profile 私有 Schema 也不创建。
5. fixtures 是官方 oracle：投影/哈希/normalization 输出变化必须同步更新对应 fixture 与 `specs/CONFORMANCE.md` 锚点；fixture 字段语义的唯一来源是对应 `specs/` 与纯 crate，不得反向从 fixture 猜合同。
6. 每个持久对象/协议消息带 `schema_version`；新版本对象走新 Schema，不做隐式升级或读后补写（ADR-0009）。
7. 已退役合同（ADR-0010 三项旧 Policy 合同）不得复活；负向测试中的历史 ID 字符串不代表可用入口。
8. **数量硬门**：Schema 总数与 retained/component-native 分布以 manifest + ADR + MATRIX/PROGRESS 为准；数量变化必须伴随 ADR/spec 变更，不得悄悄加减。

## 变更流程（不可跳步）

1. 改 `source/` / `manifest.json`；
2. `git add -A` 后 `./scripts/check-schema.sh`（generate×2 + check + 全量测试 + 漂移检查）；
3. generated diff 一并提交；
4. 更新 `specs/CONFORMANCE.md` 锚点、MATRIX「Schema生成链」行与受影响 `docs/api/*` 徽章。

## 歧义处理

manifest 与 source 不一致、fragment 不唯一、`$ref` 解析歧义：停止上报，以 `schema-tool` 报错为准修 source，不得绕过校验或放宽 graph 规则。

## 验收命令

```bash
export PATH="$HOME/.local/share/pnpm/bin:$PATH"
export TMPDIR="${TMPDIR:-$HOME/.cache/shittim-build-tmp}"
export CARGO_TARGET_DIR="${CARGO_TARGET_DIR:-$PWD/rust/target}"
git add -A && ./scripts/check-schema.sh
```

## 完成判定

- 生成×2 + check 无 diff；全量门禁绿；generated diff 已提交；
- conformance 锚点与文档同步完成；提交/推送/镜像按 `Easy/commit-and-push-Easy/AGENT.md`。
