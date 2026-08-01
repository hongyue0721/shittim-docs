# domain-policy-Medium — 纯 Freedom-first Policy matcher

> 难度：**Medium**（纯领域逻辑，但 Policy 是安全核心）。进入本文件夹即表示任务落在 `rust/crates/domain-policy`。

## 目标

让 Freedom-first 匹配、Default Allow、五种 confirmation mode、TaskScope 包含与 URI 规范化保持合同精确。完成标准 = matcher 改动有 conformance 测试覆盖，无 IO/持久化依赖。

## 开工前必读

1. 根 `AGENT.md`；`Medium/rust-workspace-Medium/AGENT.md`；
2. `specs/SECURITY_PRIVILEGE.md`（Freedom-first / Default Allow / confirmation modes）、`specs/IMPLEMENTATION_CONTRACTS.md` §6.10 起（PolicyRule v2 / PermissionDecision v2）、`adr/0010`（旧合同退役）。

## 边界与职责

**只拥有**：
- `evaluate_policy`：直接消费 generated `PolicyRuleV2`，输出 `PermissionDecisionV2` decision draft（ADR-0010 终态，无 adapter/转换层）；
- 确定性排序：specificity → priority → effect → revision → ID；
- 五种 confirmation mode 一等支持，`remote_signature` 是正常 Policy 结果而非 matcher error；
- winner-only rate limit 预览/消费接口（`RateLimitPort` 由持久层实现）；
- URI / URI pattern 规范化（**全 workspace 唯一实现**）与 TaskScope containment（`resource_refs_within_task_scope`）；
- `KernelInvariantBlock` typed 输入（Stop Fence / Recovery 阻断是独立状态，不是普通 rule deny）。

**不拥有**：PD/PolicyRule 持久化、UUID/时间/revision 分配、SQLite、KCP、Stop Fence 事实。

## 硬性规则

1. **Default Allow 只在零命中且无错误时成立**；任何 `PolicyError`（含非法 URI pattern、非法输入）都不得落入 allow。
2. exclude 优先于 include；`resource_refs_within_task_scope` 必须**先完整校验全部输入**再返回布尔——早退会掩盖后续非法 URI。
3. `ResourceContainmentError` 必须保留 `input_kind` / `index` 等定位信息；本 crate 禁止压缩错误信息（上层丢失是已知未裁决偏差，见 `Difficult/kernel-kcp-Difficult/AGENT.md`，不得在本层跟进）。
4. Kernel invariant（Stop Fence / Recovery）只能经 `KernelInvariantState` typed 输入表达，禁止编码为普通 rule 或隐式 deny。
5. matcher 只接受 active v2 合同类型；已退役旧合同不得复活（ADR-0010）。
6. 纯函数：无 IO、无存储、无时间/ID 分配。

## 验收命令

```bash
export TMPDIR="${TMPDIR:-$HOME/.cache/shittim-build-tmp}"
export CARGO_TARGET_DIR="${CARGO_TARGET_DIR:-$PWD/rust/target}"
cargo test --manifest-path rust/Cargo.toml -p domain-policy
```

改动排序/mode/URI 语义后同步全 workspace 测试、`scripts/check-schema.sh`，并检查 `docs/api/domain-policy.md` 与 `specs/CONFORMANCE.md` 锚点。

## 完成判定

- 聚焦测试（含 `tests/policy_conformance.rs`、`tests/resource_scope_containment.rs`）绿；门禁全绿；
- 文档同步完成；提交/推送/镜像按 `Easy/commit-and-push-Easy/AGENT.md`。
