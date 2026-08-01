# dual-repo-sync-Difficult — 双仓发布闭环与同步工具

> 难度：**Difficult**（远端/身份/历史/推送门禁，破坏不可逆，必须强模型 + 只读审查）。进入本文件夹即表示任务属于「主仓发布、文档镜像同步、或维护同步工具」。

## 目标

让每次发布闭环可审计、可追溯、不可伪造：主仓是唯一权威，镜像永远与主仓同一事实。完成标准 = 两仓远端 SHA 核对、镜像 `--check` 通过、任何失败按结构化错误码修根因。

## 开工前必读

1. 根 `AGENT.md`（§3 双仓库发布闭环、§4 完成条件）；
2. `docs/REPOSITORY_MAINTENANCE.md` **全文**（§5 失败处理、§6 同步工具合同：§6.1 远端路径、§6.2 CLI、§6.3 闭集与对象导出、§6.4 历史与推送门禁、§6.5 结构化错误码、§6.6 配置事实）；
3. `Easy/commit-and-push-Easy/AGENT.md`（发布顺序的流程版）。

## 固定合同事实（`PRODUCTION_CONTRACT`）

| 角色 | 值 |
|---|---|
| 主仓远端 | `https://github.com/hongyue0721/shittim.git` |
| 文档仓远端 | `https://github.com/hongyue0721/shittim-docs.git` |
| 分支 | 两仓均 `master` |
| 身份 | 名称 `小岳`、邮箱 `2933634892@qq.com`（author 与 committer 都须匹配） |
| 文档 checkout | 主仓同级 `shittim-docs`（可用 `SHITTIM_SYNC_DOCS_ROOT` 覆盖） |

修改任一事实前**先**更新 `docs/REPOSITORY_MAINTENANCE.md` §6.6 与工具内 `PRODUCTION_CONTRACT`，再执行下一次同步。

## 硬性规则

1. **同步前置**：主仓必须 clean、在 `master`、HEAD 实时等于远端（否则 `source_dirty` / `source_not_pushed` 失败）。
2. **镜像闭集**：只含主仓已推送 commit 中全部 tracked `*.md` + `LICENSE` + 固定文档仓 `.gitignore`；拒绝 symlink/gitlink/非法路径/非 100644/100755 mode；不得包含 Rust/TS 源码、`schemas/`、`scripts/`、依赖、凭据。
3. **历史门禁**：文档仓 first-parent 线性、禁止 merge；每个 commit 身份必须匹配；marker SHA 必须属于来源 first-parent 索引且严格递增；每个历史 commit 的 tree 必须精确等于来源 SHA 闭集 + `.gitignore`；`FILE_MANIFEST.md` 必须严格匹配。
4. **推送门禁**：push 前复验远端 tip；待推 commit 的 parent 必须精确等于 expected remote parent；禁止 force / force-with-lease；push unknown 保留 `refs/heads/docs-sync-staging`，下一轮以 `recover_staging` 严格恢复同一 commit，绝不创建平行 commit。
5. **失败处理**：`source_identity` / `docs_history` / `docs_tree` / `docs_staging_invalid` / `docs_remote_diverged` 等结构化错误码任一出现，停止并修根因，不得绕过（完整码表见 `REPOSITORY_MAINTENANCE.md` §6.5）。
6. **环境净化**：所有 Git 子进程清除 `GIT_DIR`/`GIT_WORK_TREE`/`GIT_INDEX_FILE`/`GIT_CONFIG*` 等可重定向变量，保留认证传输变量；不使用 `gh`、不猜认证。
7. **checkout 状态机**：文档 checkout 由 `scripts/docs-checkout-transaction.mjs` 原子 journal 管理；journal 边界中断后下轮必须以同一目标重跑收敛，逐字节保护用户文件，禁止覆盖。

## 验收命令

```bash
export PATH="$HOME/.local/share/pnpm/bin:$PATH"
pnpm run check:docs-repository    # 只读验收（临时 bare 仓审计，不动本地 checkout）
pnpm run sync:docs-repository     # 主仓已推送后执行
pnpm run test:docs-repository     # 工具自身测试（本地 bare remotes）
```

## 完成判定

- 主仓远端 SHA == 本地验收 SHA；
- 镜像已同步，`--check` 通过（文件集合/内容/`FILE_MANIFEST.md` 一致）；
- 两仓工作区干净；报告附两仓 SHA；
- 提交/推送/镜像身份全部为 `小岳 <2933634892@qq.com>`。
