# kernel-kcp-Difficult — KCP Value preflight / dispatcher / handler

> 难度：**Difficult**（协议/错误闭集/版本语义，跨层一致性核心，必须强模型 + 全量门禁 + 独立审查）。进入本文件夹即表示任务落在 `rust/crates/kernel-kcp`。

## 目标

让 KCP 保持「库级、不可连接、fail closed」的诚实边界：Value preflight 精确消费 bindings、错误闭集与 catalog 一致、未实现方法诚实返回 `KnownCatalogMethodNotImplemented`。完成标准 = handler/错误映射改动通过全量门禁 + `review/` 审查。

## 开工前必读

1. 根 `AGENT.md`；`Medium/rust-workspace-Medium/AGENT.md`；
2. `specs/IMPLEMENTATION_CONTRACTS.md` §5（§5.7 Error Catalog、§5.10/§5.11 handler/dispatcher）、§13.5（八方法集）、§13.7（谓词）；`adr/0003`、`0006`、`0009`；
3. **当前状态不复述在本文件**：每个方法当前是否已注册/可调用，开工前查 MATRIX/PROGRESS 与 `dispatcher.rs`；Schema/Binding 存在不等于 handler 存在。

## 边界与职责

**只拥有 KCP 的 Value 层**：`serde_json::Value` → method-aware preflight（消费 generated `METHOD_VERSION_BINDINGS` / `select_request_version`）→ typed decode → `narrow_to_registered` → 正式 handler 与 backend ports。

首批方法合同固定为 `system.ping`、`task.create`、`task.get`、`task.list`、`event.subscribe`、`event.poll`、`stop.activate`、`stop.status`（§13.5）。

**刻意不提供（必须作为显式后续层，不得在本 crate 顺手塞入）**：bytes/UTF-8 parser、frame codec、Unix Domain Socket / Windows Named Pipe、transport server、client、`agentd` composition root、Publisher。所需 handler + §13.7 谓词未闭合前，**禁止**启动可连接 server。

## 硬性规则

1. **Value 边界**：preflight 只接受已解析的 `serde_json::Value`；本 crate 永远不碰 bytes/socket。
2. **版本语义**：`task.create` active 只接受 v2（root-only）；v1 create → `unsupported_schema_version`；经 active preflight 后仍出现的 typed v1 create 是内部合同违例 → `InternalContractViolation` fail closed，禁止伪装成 wire 错误或其他方法。
3. **响应纪律**：`request_id` 逐字节复制；success 与 error 互斥；deadline 在入口与完成各读一次 clock；post-commit notification 只是 intent，不代表事件已发布。
4. **错误闭集**：对外 error code/message/retryable 必须以 IC §5.7 v2 Error Catalog 为唯一事实源；禁止发明新 code、禁止把内部错误细节泄进 message。backend 错误映射保持封闭穷举（`ports::BackendError` → handlers），新 backend 错误必须先入合同。
5. **后端语义不复制**：创建/查询业务规则在 `kernel-sqlite` 与纯 crate；adapter 只做端口映射，不发明业务默认值。Delegation 未落地前保持 `delegation_not_found` fail closed。
6. **生产 ID/clock 实现**（`RandomKernelIdGenerator` / `SystemKernelClock`）只分配，不拥有业务关系；UUID 用途闭集见 `ports::UuidPurpose` / `OpaqueIdPurpose`。

## 已知未裁决偏差（审阅发现，非行为范本）

- **时钟精度**：`SystemKernelClock` 返回含纳秒的系统时间，而 `kernel-sqlite` 写路径要求秒级精度——生产 `task.create` 可能因此被拒。拍板前不得在本 crate 静默截断，也不得在 sqlite 侧放宽。
- **Error Catalog 未生成化**：当前 handlers/测试存在硬编码 code/message/details，与 §5.7「catalog 唯一来源」存在已知漂移；**新增错误必须先入 catalog 合同**，不得继续扩散硬编码。
- **`invalid_scope_pattern` details**：合同要求 `input_kind`/`index`，当前 backend 映射丢失这两个字段；拍板前不得在 handler 侧自行编造 details。
- **Delegation 错误闭集**：`delegation_inactive` / `delegation_not_authorized` 尚无 backend 表达；不得用 `delegation_not_found` 冒充已实现。

## 验收命令

```bash
export TMPDIR="${TMPDIR:-$HOME/.cache/shittim-build-tmp}"
export CARGO_TARGET_DIR="${CARGO_TARGET_DIR:-$PWD/rust/target}"
cargo test --manifest-path rust/Cargo.toml -p kernel-kcp
```

handler/错误映射变更后必须：全 workspace 测试 + `scripts/check-schema.sh` + 更新 `docs/api/kernel-kcp.md`、`docs/api/kcp-preflight-dispatcher.md`、MATRIX/PROGRESS，且必须过 `review/` 独立审查。

## 完成判定

- 聚焦 + 全 workspace 测试绿；clippy/fmt 绿；统一门全绿；
- `review/` 审查通过；
- 提交/推送/镜像按 `Easy/commit-and-push-Easy/AGENT.md` 完成。
