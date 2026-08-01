# commit-and-push-Easy — commit 与 push 到哪

> 难度：**Easy**（纯流程，便宜快模型可接）。进入本文件夹即表示任务属于「把已验收的改动提交并推送到远端，或同步文档镜像」。

## 目标

让每一次变更以可审计的方式进入主仓与文档镜像：提交信息中文且按功能域、身份固定、推送有验证、镜像与主仓同事实。完成标准 = 两仓远端 SHA 均已核对、工作区干净、可以报告完成。

## 开工前必读

1. 根 `AGENT.md`（§3 双仓库发布闭环、§4 完成条件）；
2. `docs/REPOSITORY_MAINTENANCE.md`（§3 每个切片的发布顺序、§5 失败处理与禁止事项、§6 同步工具合同）；
3. 涉及的门禁定义：`scripts/check-schema.sh`（统一门）。

## commit 与 push 到哪（固定合同事实）

| 角色 | 远端 | 分支 | 说明 |
|---|---|---|---|
| **主仓**（唯一权威源） | `https://github.com/hongyue0721/shittim.git` | `master` | 源码、Schema、测试、脚本、文档都只在这里演化 |
| **文档镜像** | `https://github.com/hongyue0721/shittim-docs.git` | `master` | 纯文档只读镜像，由 `scripts/sync-docs-repository.mjs` 同步，**禁止**手动修改/独立演化 |
| **身份** | — | — | author/committer 名称 `小岳`、邮箱 `2933634892@qq.com`（local config 与每个 commit 都要一致） |

## 发布顺序（每个切片严格按此闭环）

1. 主仓完成规范、实现、测试与文档更新；
2. `node scripts/update-file-manifest.mjs --write` 刷新 `FILE_MANIFEST.md`；
3. `git add -A`（统一门的 generated-tree 漂移检查对 index 做 diff，未 stage 会被误判）；
4. 执行 `./scripts/check-schema.sh` 全量门（Node 硬门 → schema generate×2/check → fmt/clippy/workspace test → generated 漂移 → FILE_MANIFEST）；
5. 独立验收通过（Difficult 任务须过 `review/` 审查）后提交：
   - 提交信息**中文、按功能域**（例：`文档: 新增 agent-guide 规范库并同步 Lease 实现状态`）；
   - 一个功能域一个提交，不混入无关改动；
6. `git push origin master`；
7. **验证远端 SHA == 本地已验收 SHA**（`git ls-remote origin master` 与 `git rev-parse HEAD` 对比）；
8. 从已推送主仓 HEAD 同步文档镜像：`pnpm run sync:docs-repository`，随后 `pnpm run check:docs-repository` 验证镜像文件集合/内容/`FILE_MANIFEST.md` 与主仓一致；
9. 两仓远端验证完成且工作区干净，才可报告任务完成。

## 硬性规则与禁止事项

- **身份不符不得提交/推送**：author/committer 的 name/email 任一不等于合同值，先修 `git config user.name` / `user.email` 再提交。
- **禁止强推**：不得 `--force` / `--force-with-lease` 覆盖未知远端变化；确需重写历史必须先确认协作状态、保存旧 SHA、带明确 lease。
- **禁止跳过门禁提交**：统一门失败（含 Node 版本不对、generated 漂移、clippy 报错、测试红）不得提交。
- **禁止从未提交/未推送工作区制作镜像**：镜像只能来自已推送主仓 commit。
- **镜像不得包含实现**：Rust/TS 源码、`schemas/`、`scripts/`、依赖、凭据一律不进镜像；镜像只含 tracked `*.md` + `LICENSE` + 固定 `.gitignore`。
- **禁止提交身份/仓库地址之外的事实**：远端、分支、身份发生变化时，先更新 `docs/REPOSITORY_MAINTENANCE.md` §6.6 与同步工具内 `PRODUCTION_CONTRACT`，再执行下一次同步。
- **同步工具的错误码**：`source_dirty` / `source_not_pushed` / `source_identity` / `docs_history` / `docs_tree` / `docs_push_unknown` 等任一出现，停止并修根因，不得绕过（完整错误码表见 `docs/REPOSITORY_MAINTENANCE.md` §6.5）。

## 验收命令

```bash
export PATH="$HOME/.local/share/pnpm/bin:$PATH"
git config user.name && git config user.email            # 必须 小岳 / 2933634892@qq.com
git rev-parse HEAD
git ls-remote origin master                              # 推送后与 HEAD 对比
pnpm run sync:docs-repository                            # 主仓推送后同步镜像
pnpm run check:docs-repository                           # 镜像验收
```

## 完成判定

- 主仓远端 SHA == 本地验收 SHA；
- 文档镜像已同步且 `--check` 通过；
- 工作区干净；
- 报告里附上两仓 SHA 与门禁结果。
