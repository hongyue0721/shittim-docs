# rust-workspace-Medium — Rust workspace 全局规则

> 难度：**Medium**（涉及 Rust 代码，需理解分层与门禁）。进入本文件夹即表示任务落在 `rust/` 工作区（不含 `Difficult/` 三个 crate 内部）。

## 目标

让 Rust 侧保持「分层清晰、纯 crate 无 IO、生成物禁手改、门禁全绿」。完成标准 = 改动通过聚焦测试 + 全 workspace 测试 + fmt/clippy + 统一门。

## 开工前必读

1. 根 `AGENT.md`（§3 编码规则、§4 完成条件）；
2. 本文件夹对应 crate 的 `AGENT.md`（`Medium/domain-task-Medium` 等）；
3. 涉及的 `specs/` 合同节与 `docs/IMPLEMENTATION_MATRIX.md` / `docs/PROGRESS.md` 状态。

## 工作区结构（依赖方向，只允许向下）

```text
kernel-contracts            生成类型 / 校验 / JCS / 时间规范化（最底层，无业务）
    ↑
domain-task                 纯 Task/Action 状态机（只依赖 kernel-contracts）
domain-policy               纯 Policy matcher / URI / Scope 包含
kernel-task-creation        纯 root/child 创建规范化与 allocation 校验
kernel-authorization        纯授权投影 / JCS / SHA-256
    ↑
kernel-sqlite               文件持久化（依赖以上全部纯 crate，不复制其语义）
    ↑
kernel-kcp                  KCP Value preflight / dispatcher / handler（经 ports 调 kernel-sqlite）

schema-tool                 生成链工具（独立，不被任何运行时 crate 依赖）
```

禁止反向或跨层依赖；新 crate / 新 workspace 依赖必须先有 ADR。

## 硬性规则

1. **生成物禁手改**：`kernel-contracts/src/generated/` 由 `schema-tool` 生成；改类型只能改 `schemas/source` + 走生成链（ADR-0002）。禁止手写平行 Rust 类型。
2. **纯 crate 禁 IO**：`domain-task` / `domain-policy` / `kernel-task-creation` / `kernel-authorization` 不读存储、不分配 ID、不取系统时间、不依赖 SQLite/KCP/Tokio；事实一律由调用方 typed 注入。
3. **外部输入错误走 `Result`**：生产路径禁止 panic / unwrap / expect 处理外部输入；内部不变量违例也映射为结构化错误（fail closed）。
4. **`#![deny(missing_docs)]`** 是既有 crate 惯例，新 public API 必须带文档注释。
5. **当前无 Tokio**：workspace 尚无 async runtime；引入属于 ADR 级变更，不得顺手加入。
6. **状态语义只在纯 crate 解释**：repository/KCP 层不得自行发明状态迁移、错误码或业务默认值；缺事实时 fail closed，禁止兜底伪造。
7. **工具链固定**：`rust-toolchain.toml`（Rust 1.97.0，edition 2021）。
8. **构建目录约定**：编译/测试前设置 `export TMPDIR="${TMPDIR:-$HOME/.cache/shittim-build-tmp}" CARGO_TARGET_DIR="${CARGO_TARGET_DIR:-$PWD/rust/target}"`（任意可写目录，禁止写死单一 host 路径）；库/测试代码本身用 `TMPDIR` / `tempfile` 标准抽象，**禁止**硬编码任何 host path。

## 歧义与缺失处理

- 合同之间冲突、合同与代码冲突、规范未覆盖：**停止并在报告中说明冲突点与候选解释，不得自行拍板**。已知未裁决偏差记录在各部分 `AGENT.md` 的「已知未裁决偏差」节，那些条目不是行为范本，未经维护者在 `specs/` 拍板前不得「修复」也不得仿效。

## 验收命令（任何 crate 改动后）

```bash
export PATH="$HOME/.local/share/pnpm/bin:$PATH"
export TMPDIR="${TMPDIR:-$HOME/.cache/shittim-build-tmp}"
export CARGO_TARGET_DIR="${CARGO_TARGET_DIR:-$PWD/rust/target}"
cargo test --manifest-path rust/Cargo.toml -p <改动crate>
cargo test --manifest-path rust/Cargo.toml --workspace
cargo clippy --manifest-path rust/Cargo.toml --workspace --all-targets -- -D warnings
cargo fmt --manifest-path rust/Cargo.toml --all -- --check
git add -A && ./scripts/check-schema.sh   # 统一门；先 stage 再做 generated-tree 漂移检查
```

## 完成判定

- 聚焦 + 全 workspace 测试绿、clippy/fmt 绿、统一门全绿；
- 文档与 MATRIX/PROGRESS 同切片更新（crate 职责、公开 API 闭集、实现状态变化时）；
- 提交/推送/镜像同步按 `Easy/commit-and-push-Easy/AGENT.md` 完成。
