# kernel-authorization 内部 Rust API

> **非 KCP 外部 API**。本页描述 `rust/crates/kernel-authorization` 的纯授权投影与远程签名验签原语。
>
> 不读 repository、不分配 ID、不写存储、不取系统时间、不匹配 Policy（`domain-policy` 的唯一职责）。
> 全部权威事实由调用方以 typed facts 结构体注入；本 crate 不自己取数、不把 `serde_json::Value` 当自由袋接受。

## 定位

| 项 | 说明 |
|---|---|
| Crate | `kernel-authorization` |
| 权威语义 | `IMPLEMENTATION_CONTRACTS.md` §5.3.1（通用投影与哈希）、§6.10.1（SubjectProjection）、§6.10.4（Remote signature preimage）；ADR-0006（Child Task 权威与 delta）、ADR-0007（Approval subject）、ADR-0011 §10（RemoteSignatureVerifier 边界） |
| 依赖 | `kernel-contracts`（生成类型/JCS/SHA-256）、`domain-policy`（仅复用其类型事实）、serde/serde_json、thiserror、uuid、chrono |
| 禁止依赖 | SQLite、Tokio、KCP server、UI、Extension；禁止 IO/时间/ID 分配 |
| 输出合同 | 构造 → 再次 Schema 验证 → RFC 8785 JCS → UTF-8 bytes（无 BOM、无尾换行）→ SHA-256 → lowercase hex；JCS bytes 与哈希是跨层对账事实，字段集/顺序变化即合同变更 |

## 授权投影入口

```rust
use kernel_authorization::{
    project_material_authorization, project_observation_evidence,
    project_subject_projection, project_child_task_delta, CanonicalProjection,
    MaterialAuthorizationFactsV1, ObservationEvidenceFactsV1,
    SubjectProjectionFactsV1, ChildTaskDeltaFactsV1,
};
```

- `project_material_authorization(MaterialAuthorizationFactsV1)`：Material Authorization 投影与指纹（material fingerprint）。
- `project_observation_evidence(ObservationEvidenceFactsV1)`：Observation Evidence 投影与指纹（observed / not_applicable 两分支；evidence refs 按 set 投影排序去重，protected surface 顺序保留）。
- `project_subject_projection(SubjectProjectionFactsV1)`：Approval Subject 投影（operation / task_proposal / plan_revision 三分支，即 `subject_hash` 的唯一 preimage）。
- `project_child_task_delta(ChildTaskDeltaFactsV1)`：Child Task Delta 投影（child materializer 的授权事实来源，ADR-0006）。
- `material_policy_set_revision_for_projection(...)`：投影时使用的 policy set revision 事实辅助。

每个投影返回 `CanonicalProjection<T>`：`value`（Schema-validated 投影对象）、`jcs_utf8`（RFC 8785 规范化字节）、`sha256`（lowercase hex）。输入为 typed facts；校验失败返回 `AuthorizationProjectionError::InvalidFact { field, reason }`（fail closed，不产出哈希）。

## 远程签名验签原语（ADR-0011 §10）

```rust
use kernel_authorization::{
    verify_remote_approval_ed25519, remote_approval_signature_preimage_jcs,
    decode_ed25519_public_key_base64url_no_pad,
    decode_ed25519_signature_base64url_no_pad, verify_ed25519_pure,
    RemoteSignatureVerifierError,
};
```

- `remote_approval_signature_preimage_jcs(&RemoteApprovalSignaturePreimageV1)`：重建 preimage 的 RFC 8785 JCS UTF-8 bytes（再 Schema 验证 + lossless round-trip；无 BOM、无尾换行）。
- `verify_ed25519_pure(public_key, message, signature)`：Ed25519 RFC 8032 pure mode（`verify_strict`），message 即签名输入本身、不先 hash。
- `verify_remote_approval_ed25519(public_key_b64, &preimage, signature_b64)`：高层端口——base64url_no_pad 解码（公钥恰 32 bytes、签名恰 64 bytes）→ preimage JCS 重建 → pure mode 验签。
- `decode_ed25519_public_key_base64url_no_pad` / `decode_ed25519_signature_base64url_no_pad`：wire 解码原语，精确长度校验。
- 常量 `ED25519_PUBLIC_KEY_LEN = 32`、`ED25519_SIGNATURE_LEN = 64`。

失败语义（fail closed）：任何解码/长度/预像/验证失败都是 `RemoteSignatureVerifierError` 的闭集 variant（`InvalidInput` / `Contract` / `Json` / `VerificationFailed`），绝不发明授权成功。Challenge CAS、expiry、consume、resolution 与 head CAS 是调用方（SQLite owner）按 §6.10.3 同一事务的义务，本 crate 不承担。

## 测试资产

- 官方投影 fixtures：`schemas/fixtures/policy/{material_authorization_projection,observation_evidence_observed,observation_evidence_not_applicable,subject_projection,remote_approval_signature_preimage}.v1.json`、`schemas/fixtures/task/child_task_delta_projection.v1.json`。
- harness：`tests/official_authorization_projection_fixtures.rs` 逐字节对账 JCS/SHA-256 + tamper matrix（含 schema-valid 篡改 hash-bound 断言与 schema-invalid fail-closed 断言）。
- 密码学向量：RFC 8032 §7.1 官方向量 1–3 + 篡改/长度/解码负例（`src/remote_verifier.rs` 单元测试）。
