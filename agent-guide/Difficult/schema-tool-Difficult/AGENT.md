# schema-tool-Difficult — Schema 生成链工具

> 难度：**Difficult**（生成链/artifact transaction/编译铁律，破坏面大，必须强模型 + 全量门禁 + 独立审查）。进入本文件夹即表示任务落在 `rust/crates/schema-tool`。

## 目标

让生成链保持「幂等、原子、可恢复、与合同一致」：`schemas/` → generated artifacts 的单一可信通道。完成标准 = generate×2 + check 无 diff、artifact transaction 语义不被破坏、数量硬门不破。

## 开工前必读

1. 根 `AGENT.md`（§3 Schema 编译铁律）；`adr/0002`、`adr/0010`；`specs/IMPLEMENTATION_CONTRACTS.md` §13；
2. `Medium/schemas-manifest-Medium/AGENT.md`（Schema 源侧规则）。

## 边界与职责

`schemas/` → generated artifacts 的唯一生成链：
- manifest v2 解析与身份（`manifest` / `manifest_identity`）、SchemaRegistry、production stage 校验（`validate_production_manifest_stage`）；
- `$ref` 解析（`Url::join` + percent-decode → RFC6901）、canonical fragment 唯一性、target contract graph、RustProjection（SCC `Option<Box<T>>` 处理 direct 递归）；
- MethodVersionBinding / Event catalog 从 Envelope 派生（authority 不手维护）；
- artifact 单 root transaction（`artifact_transaction`）：计划 → 原子落盘 → 崩溃恢复；
- CLI 是 library-first API 的薄适配层；integration tests 可检查 graph/projection 事实，不只能靠刮 generated 源码字符串。

## 硬性规则（Schema 编译铁律，源自根 `AGENT.md` §3）

1. 中立 graph 的 `ContractTypeId`（`$id` + 严格 JSON Pointer）与语言侧 `RustDeclarationId` 分离；禁止把 language name / logical_title 写进中立 IR。
2. `manifest.id_base` 是 entry `$id` 权威 URL path 命名空间（canonical absolute http(s) + trailing `/`）。
3. renderer 按 use-site lineage 投影多个 declaration；`RustProjection` 对同一 `ContractTypeId` 只 project 一次。
4. `ArtifactPlan::try_new` 必须 path/root component-safe；外部输入错误走 `Result`，生产路径禁止 panic。
5. **generate 幂等**：统一门连续跑两次 `generate` 再 `check`；任何第二次运行产生 diff 都是 bug。
6. **production 硬门**：精确 Schema 总数与 retained/component-native 分布以 `schemas/manifest.json`、ADR-0010 与 MATRIX/PROGRESS 为准；数量变化必须伴随 ADR/spec 变更，不得悄悄加减。
7. 无网络访问；生成只读 `schemas/` 与 manifest，写只经 artifact transaction。
8. **锁纪律**：`.schema-tool-generate.lock` 是持久 regular file 上的 advisory FD lock（`File::try_lock`）：文件存在或 owner 文本过时不等于锁被占用；只有 `WouldBlock` 才表示真实 contention。禁止按时间/文本猜 stale 并删除；若路径是目录或 symlink，停止并上报。

## 已知注意事项

- 统一门的 generated-tree 漂移检查是 `git diff`（index↔worktree 语义）：**跑 `scripts/check-schema.sh` 前必须 `git add -A`**，否则已 stage 的生成物变化会被误判（`docs/DEVELOPMENT_HANDOVER.md` §2.5）。
- 测试 `artifact_transaction lock_conformance real_cross_process_holder_crash_releases_fd_lock` 并行高负载下偶发时序 flake，单跑必过（`docs/DEVELOPMENT_HANDOVER.md` §9）；属已知问题，不得用删除测试的方式「修复」。

## 验收命令

```bash
export PATH="$HOME/.local/share/pnpm/bin:$PATH"
export TMPDIR="${TMPDIR:-$HOME/.cache/shittim-build-tmp}"
export CARGO_TARGET_DIR="${CARGO_TARGET_DIR:-$PWD/rust/target}"
git add -A
./scripts/check-schema.sh   # Node 硬门 → generate×2 → check → fmt → clippy → workspace test → 漂移 → FILE_MANIFEST
```

生成链行为变更后同步更新 `docs/api/schema-generation.md`、MATRIX「Schema生成链」行与 `specs/CONFORMANCE.md` 锚点，且必须过 `review/` 独立审查。

## 完成判定

- generate×2 + check 无 diff；全量门禁绿；无 artifact transaction 语义回退；
- `review/` 审查通过；
- 提交/推送/镜像按 `Easy/commit-and-push-Easy/AGENT.md` 完成。
