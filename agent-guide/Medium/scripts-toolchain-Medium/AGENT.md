# scripts-toolchain-Medium — 零依赖 Node 工具链与统一门

> 难度：**Medium**（维护工具/门禁脚本，需理解契约）。进入本文件夹即表示任务落在 `scripts/`（不含双仓同步，那是 `Difficult/dual-repo-sync-Difficult`）。

## 目标

让仓库工具链保持零依赖、版本精确、门禁可信。完成标准 = 脚本改动通过自身测试 + 统一门，不破坏 `PRODUCTION_CONTRACT`。

## 开工前必读

1. 根 `AGENT.md`；`docs/REPOSITORY_MAINTENANCE.md`（§5 失败门禁、§6 同步工具合同概要）；
2. `adr/0001`（工作区与工具链决策）。

## 边界与职责

| 文件 | 职责 |
|---|---|
| `check-node-toolchain.mjs` | 精确 Node 24.18.0 / pnpm 11.3.0 硬门（读 `package.json` engines/packageManager） |
| `check-schema.sh` | 仓库统一门：Node 硬门 → schema generate×2/check → fmt/clippy/workspace test → generated 漂移 → FILE_MANIFEST |
| `update-file-manifest.mjs` | 从 Git Markdown source set 生成/校验 `FILE_MANIFEST.md`（`--check` / `--write` / `--self-test`） |
| `sync-docs-repository.mjs` 等 | 双仓同步工具（见 `Difficult/dual-repo-sync-Difficult/AGENT.md`，不在本部分） |

## 硬性规则

1. **零依赖**：只准用 Node 标准库；新增 npm 依赖属 ADR 级变更。
2. **工具链事实**：Node 精确 24.18.0、pnpm 精确 11.3.0，入口 `~/.local/share/pnpm/bin`；`engine-strict=true` 只是提示，不得当作硬门；Corepack 不可用。
3. **临时目录合同**：仅工具测试使用由 `TMPDIR` 派生的 `shittim-docs-sync-tests` 与 `shittim-file-manifest-tests`（mode 0700/0600，逐 fixture 清理），不硬编码 host 路径；`TMPDIR` 必须显式设置为 `/tmp` 之外的任意可写目录（如 `$HOME/.cache/shittim-build-tmp`），未设置或指向 `/tmp` 时测试 fail closed；不得把该约束推广成全 workspace 禁令，也不得让工具读写其它 host 路径。
4. **锁纪律**：`O_EXCL` 独占锁不自动清 stale；撞锁停止上报（双仓同步锁见 `Difficult/dual-repo-sync-Difficult/AGENT.md`）。
5. **统一门纪律**：`check-schema.sh` 任一步失败都修根因，禁止绕过单步提交；跑之前先 `git add -A`（generated 漂移检查基于 index/worktree diff）。
6. **失败处理**：统一门/工具失败时按结构化错误码定位根因，禁止用「跳过/重试碰运气」掩盖（`REPOSITORY_MAINTENANCE.md` §5）。

## 验收命令

```bash
export PATH="$HOME/.local/share/pnpm/bin:$PATH"
pnpm run test:file-manifest && pnpm run test:docs-repository
node scripts/check-node-toolchain.mjs
```

## 完成判定

- 工具自身测试绿；统一门全绿；
- 文档同步完成；提交/推送/镜像按 `Easy/commit-and-push-Easy/AGENT.md`。
