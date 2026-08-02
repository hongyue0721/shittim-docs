# kernel-authorization-Medium — 纯授权投影与哈希

> 难度：**Medium**（纯逻辑，投影是 Approval/Child 的事实来源）。进入本文件夹即表示任务落在 `rust/crates/kernel-authorization`。

## 目标

让授权投影（Material / Observation / Subject / Child Delta）的构造、JCS 与 SHA-256 保持合同精确。完成标准 = 投影输出与 official fixtures 逐字节对账，纯函数无 IO。

## 开工前必读

1. 根 `AGENT.md`；`Medium/rust-workspace-Medium/AGENT.md`；
2. `specs/IMPLEMENTATION_CONTRACTS.md` §5.3.1、§6.10；`adr/0006`（Child Task 权威与 delta）、`adr/0007`（Approval subject / usable proof）。

## 边界与职责

**只拥有** typed facts → 投影的构造、验证、RFC 8785 canonicalization 与 SHA-256：
- `project_material_authorization`（Material Authorization 投影与指纹）；
- `project_observation_evidence`（Observation Evidence 投影与指纹）；
- `project_subject_projection`（Approval Subject 投影）；
- `project_child_task_delta`（Child Task Delta 投影，child materializer 的授权事实来源）。

**不拥有**：repository 读取、ID 分配、存储写入、Policy 匹配（不得替代 `domain-policy`）。所有权威事实由调用方以强类型注入；本 crate 不自己取数。

## 硬性规则

1. 投影输入必须是 typed facts 结构体（如 `MaterialAuthorizationFactsV1`），禁止接受 `serde_json::Value` 自由袋。
2. 同一投影的 JCS bytes 与哈希是跨层对账事实：算法、字段集、字段顺序的任何变化都是合同变更，必须先改 `specs/` 与 fixtures。
3. 纯函数：无 IO、无存储、无时间/ID 分配。
4. `RemoteSignatureVerifier` 的纯密码学部分（Ed25519 RFC8032 pure mode）已落地本 crate（`remote_verifier.rs`，随 Lease/Stop Fence 切片 `c11d3be` 进入基线）。**verify 边界与失败语义的任何变更都是合同变更**：必须先拍板 ADR/spec，再同步 fixtures 与 conformance 锚点，不得直接改验签行为。

## 测试资产

- 官方投影 fixtures：`schemas/fixtures/`（child-delta / subject 等 official projection fixtures）。投影输出必须与 fixtures 逐字节对账；改投影必须同步改 fixtures 与 conformance 锚点。

## 验收命令

```bash
export TMPDIR="${TMPDIR:-$HOME/.cache/shittim-build-tmp}"
export CARGO_TARGET_DIR="${CARGO_TARGET_DIR:-$PWD/rust/target}"
cargo test --manifest-path rust/Cargo.toml -p kernel-authorization
```

## 完成判定

- 聚焦测试 + fixtures 对账绿；全 workspace 门禁绿；
- 文档同步完成；提交/推送/镜像按 `Easy/commit-and-push-Easy/AGENT.md`。
