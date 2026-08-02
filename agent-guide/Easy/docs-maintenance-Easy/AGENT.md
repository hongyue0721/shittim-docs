# docs-maintenance-Easy — 怎么维护文档

> 难度：**Easy**（不碰业务语义，纯文档/元数据维护，便宜快模型可接）。进入本文件夹即表示任务属于「维护 `docs/`、`FILE_MANIFEST.md` 或任何 Markdown 状态陈述」。

## 目标

进入本文件夹后你要做的事：让仓库文档与代码/规范**永远同事实**，且不产生第二套事实源。完成标准 = 文档变更后 `docs/IMPLEMENTATION_MATRIX.md` / `docs/PROGRESS.md` 与代码一致、`FILE_MANIFEST.md` 已刷新、没有任何页面在描述旧事实。

## 开工前必读

1. 根 `AGENT.md`（宪法，尤其 §3「文档随实现持续更新」与 §4「完成条件」）；
2. 本仓库 `docs/REPOSITORY_MAINTENANCE.md`（§2 持续更新义务、§2.1 状态单一来源、§3 发布顺序、§4 镜像闭集、§6 同步工具）；
3. 若涉及 `docs/api/*`：对应部分的 `AGENT.md`（本目录不复制字段级合同）；
4. 若任务只是「读文档确认现状」：以 `docs/PROGRESS.md` 为当前事实快照，以 MATRIX 为域状态表。

## 边界与职责

**拥有**：`docs/` 全部页面、`FILE_MANIFEST.md`、README/PROJECT_OVERVIEW 中的状态陈述、文档镜像同步前置工作。

**不拥有**：`specs/` 字段定义（那是唯一事实源）、`adr/` 决策、Rust/Schema 代码。发现代码与文档冲突时，文档让位——但**不得自己改代码**，只记录并上报，或按对应部分手册处理。

## 硬性规则

1. **状态单一来源**：完成度只允许写在 `IMPLEMENTATION_MATRIX.md`（域状态表）与 `PROGRESS.md`（里程碑/backlog/阻塞）。`docs/api/*` 与 ADR 只允许一枚状态徽章（`implemented` / `partial` / `contract-only` 等）+ 指向 MATRIX/PROGRESS/specs 的锚点链接。**禁止**在 api 页/ADR 维护字段级复述（subject_hash、UUID 分配表、projection 形状、IR 细节等）。
2. **诚实分级**：`contract-only` / `schema-source-present` / `library-implemented` / `composition-reachable` / `publicly-usable` / `SDK-publishable` / `real-platform-verified` 必须明确区分；历史事实不得改写成当前能力。表存在 ≠ owner 实现；Schema 存在 ≠ 业务闭环。
3. **数字必须来自实测**：测试数量、Schema 数量与 retained/component-native 分布只写实际运行/当前 manifest 的结果；写不出就标注「未实测」，禁止预写猜测。
4. **不写速朽事实**：如「领先远端 N 个提交」这类一次 push 即失真的数据；验收基线只以「以已推送 origin/master 为准」方式表述。
5. **FILE_MANIFEST 纪律**：任何 Markdown 增删改后执行 `node scripts/update-file-manifest.mjs --write`；`FILE_MANIFEST.md` 禁止手改（由脚本生成）。
6. **HANDOVER 维护**：`docs/DEVELOPMENT_HANDOVER.md` 的「下一实现任务/当前状态快照」每次切片收尾必须重读重改；描述的下一步已完成就立即更新，不得留下引导重复实现的旧指令。
7. **锚点稳定**：不重排 `specs/` 章节编号、不移动被链接的标题；链接断了要修链接，不是搬家。
8. 文档变更属于仓库变更：同切片必须能通过 §验收命令，不能只改字不改门禁。

## 已知漂移高危点（修改文档前先核对）

- `kernel-sqlite` 的 Lease/Stop Fence：`acquire_lease` / `get_action_lease` / `begin_dispatch` / `release_or_expire_lease` / `activate_stop_fence` / `get_stop_fence` **全部已实现**（c11d3be）——任何写「Lease owner 未实现」的表述都要按这个事实改写。
- 切片 4c：11/11 High 已随 c11d3be 闭合（Approval invalidation 同事务撤销 Lease + 真实远程验签）——写「4c 未完成/剩 2 项」是错的。
- 三方法 handler（`task.list` / `event.subscribe` / `event.poll`）**无正式 handler**，KCP 不可连接 server（`stop.activate` / `stop.status` 已随切片6 落地）。
- Node 入口：`~/.local/share/pnpm/bin`（不是 `~/.local/share/pnpm/node`，也不是默认系统 node）。
- 已知未裁决偏差（root 重放 request_id、时钟精度、invalid_scope_pattern details 丢失、success criteria trim）不是「已修复事实」，文档不得写成已修复。

## 验收命令

```bash
export PATH="$HOME/.local/share/pnpm/bin:$PATH"
node scripts/update-file-manifest.mjs --write   # 刷新 FILE_MANIFEST（增删改 md 后必做）
node scripts/update-file-manifest.mjs --check   # 校验
git diff --check                                # 无空白错误
```

## 完成判定

- 所有被任务触及的状态陈述与 MATRIX/PROGRESS/代码事实一致；
- `FILE_MANIFEST.md` 已刷新且 `--check` 通过；
- 主仓提交/推送、文档镜像同步按 `Easy/commit-and-push-Easy/AGENT.md` 完成；
- 若发现「文档 vs 代码」实质冲突超出文档范围（要改代码/改 spec），**停止并上报**，不越界修复。
