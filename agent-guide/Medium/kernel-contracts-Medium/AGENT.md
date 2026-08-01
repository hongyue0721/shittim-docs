# kernel-contracts-Medium — 生成合同类型与校验

> 难度：**Medium**（涉及生成类型与合同校验，需理解生成链）。进入本文件夹即表示任务落在 `rust/crates/kernel-contracts`。

## 目标

让合同类型保持「生成即事实」：generated 类型与 `schemas/source` 完全一致、JCS/SHA-256 可靠、时间戳规范化合同精确。完成标准 = 不手改 generated、类型变更走生成链、门禁全绿。

## 开工前必读

1. 根 `AGENT.md`；`Medium/rust-workspace-Medium/AGENT.md`；`Difficult/schema-tool-Difficult/AGENT.md`；
2. `adr/0002-schema生成与兼容策略.md`、`specs/IMPLEMENTATION_CONTRACTS.md` §13。

## 边界与职责

本 crate 是全部合同类型的唯一载体：
- `generated/`：`schema-tool` 从 `schemas/source` 生成的 Rust 类型、catalog（`METHOD_VERSION_BINDINGS`、`EVENT_ACTIVE_BINDINGS` 等）——**DO NOT EDIT BY HAND**；
- `validator`：JSON Schema 校验与 `decode_validated`（带 `DecodeStage` 分阶段错误分类）；
- `canonical`：RFC 8785 JCS（`canonical_json_bytes` / `sha256_canonical`）；
- `timestamp`：`canonicalize_rfc3339_seconds`——接受 offset 与全零小数秒，**拒绝非零小数秒**，输出 UTC 秒级 `...Z`。

## 硬性规则

1. **禁止手改 `generated/` 任何文件**；类型变更只能改 `schemas/source`（或 manifest/bindings），然后跑生成链。
2. **禁止手写平行类型**：任何层需要合同形状时，引用本 crate 的 generated 类型。
3. string enum 的 `ALL` 顺序 = Schema 声明顺序（nullable enum 过滤 `null`）；依赖该顺序的代码必须在注释中说明。
4. response envelope intentionally untyped 是合同决定，不是疏漏；客户端按原请求方法选择 response payload Schema，不得给 envelope 加 method 判别字段。
5. 时间合同：凡进入持久化/哈希的 Kernel-owned 时间戳必须秒级精度；调用方应经 `canonicalize_rfc3339_seconds` 语义对齐，禁止静默截断/四舍五入后声称 canonical。
6. 错误分类（`ContractFailureStage` / `DecodeStage`）是跨层对账事实，新增阶段属于合同变更。

## 歧义与缺失处理

- generated 类型与 `schemas/source` 不一致：一律以 source + 重新生成为准，不得反向改 source 迁就手改过的 generated。
- Schema 本身歧义：停止上报，不得在 Rust 侧发明解释。

## 验收命令

```bash
export PATH="$HOME/.local/share/pnpm/bin:$PATH"
export TMPDIR="${TMPDIR:-$HOME/.cache/shittim-build-tmp}"
export CARGO_TARGET_DIR="${CARGO_TARGET_DIR:-$PWD/rust/target}"
cargo run --manifest-path rust/Cargo.toml -p schema-tool -- --repo-root . generate
cargo run --manifest-path rust/Cargo.toml -p schema-tool -- --repo-root . check
cargo test --manifest-path rust/Cargo.toml -p kernel-contracts
```

## 完成判定

- 生成×2 + check 无 diff；聚焦测试绿；全 workspace 门禁绿；
- 文档同步完成；提交/推送/镜像按 `Easy/commit-and-push-Easy/AGENT.md`。
