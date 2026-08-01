# agent-guide/ AGENT.md — 规范库总入口

> **进入本文件夹后，先完整读本文件。** 本文件夹是 Shittim 仓库全部可执行规范的集中地：每个部分一个独立文件夹，文件夹内的 `AGENT.md` 就是该部分的构建/维护手册——进去打开即可照着干活。

## 1. 这是什么

Shittim 是一个合同密集、跨层一致性强、还处于内核建设期的个人 AI 系统仓库。把规范散在代码与文档里，模型进来只能靠猜。本文件夹把规范**集中、分级、写成可执行步骤**，目标是：

- 模型（尤其便宜/快的模型）进入对应部分文件夹，打开 `AGENT.md`，不需要发挥主观能动性就能正确干活；
- 任务按难度（Easy / Medium / Difficult）分派给不同能力的模型，用并发换时间而不翻车；
- 审查独立成 `review/`，把质量关从模型手里拿回来；
- commit/push、文档同步全部写死为流程，发布闭环可审计。

**使用方式**：按任务找到对应部分文件夹（见 §3 索引）→ 完整读该文件夹的 `AGENT.md` → 按「开工前必读 → 硬性规则 → 验收命令 → 完成判定」执行。总入口只负责导航与全局规则，不复制各部分的字段级合同。

## 2. 仓库权威顺序（冲突裁决）

1. 根 [`AGENT.md`](../AGENT.md)：全局不变量、可信边界、编码与发布硬门（宪法）；
2. 对应 `specs/*.md`：字段、枚举、状态机、错误码的唯一事实源；
3. `adr/*.md`：规范允许范围内的实施决策；
4. 本文件夹对应部分的 `AGENT.md`：边界职责、禁止事项与验收命令；
5. 实现状态只看 `docs/IMPLEMENTATION_MATRIX.md` 与 `docs/PROGRESS.md`。

**同级事实源冲突时：停止工作并上报，不得自行选择更顺手的一边。**

## 3. 目录索引（按难度分级）

| 难度 | 含义 | 适用模型 | 部分 |
|---|---|---|---|
| **Easy** | 纯流程/文档/元数据维护，不碰业务语义，风险低 | 便宜快模型 | [`Easy/docs-maintenance-Easy`](Easy/docs-maintenance-Easy/AGENT.md)（怎么维护文档）<br>[`Easy/commit-and-push-Easy`](Easy/commit-and-push-Easy/AGENT.md)（commit 与 push 到哪）<br>[`Easy/adr-maintenance-Easy`](Easy/adr-maintenance-Easy/AGENT.md)（ADR 维护） |
| **Medium** | 纯领域逻辑、单 crate、Schema 源与规范编辑，需读合同，有测试锚点 | 常规模型 | [`Medium/write-specs-Medium`](Medium/write-specs-Medium/AGENT.md)（怎么做规范）<br>[`Medium/rust-workspace-Medium`](Medium/rust-workspace-Medium/AGENT.md)<br>[`Medium/domain-task-Medium`](Medium/domain-task-Medium/AGENT.md)<br>[`Medium/domain-policy-Medium`](Medium/domain-policy-Medium/AGENT.md)<br>[`Medium/kernel-contracts-Medium`](Medium/kernel-contracts-Medium/AGENT.md)<br>[`Medium/kernel-task-creation-Medium`](Medium/kernel-task-creation-Medium/AGENT.md)<br>[`Medium/kernel-authorization-Medium`](Medium/kernel-authorization-Medium/AGENT.md)<br>[`Medium/schemas-manifest-Medium`](Medium/schemas-manifest-Medium/AGENT.md)<br>[`Medium/scripts-toolchain-Medium`](Medium/scripts-toolchain-Medium/AGENT.md) |
| **Difficult** | 跨层一致性、持久化/协议/生成链/发布闭环，必须强模型 + 全量门禁 | 强模型 | [`Difficult/kernel-sqlite-Difficult`](Difficult/kernel-sqlite-Difficult/AGENT.md)<br>[`Difficult/kernel-kcp-Difficult`](Difficult/kernel-kcp-Difficult/AGENT.md)<br>[`Difficult/schema-tool-Difficult`](Difficult/schema-tool-Difficult/AGENT.md)<br>[`Difficult/dual-repo-sync-Difficult`](Difficult/dual-repo-sync-Difficult/AGENT.md) |
| **审查** | 独立于开发的质检 | 与开发不同的模型（避免自我审查） | [`review/AGENT.md`](review/AGENT.md)（审查入口与全部审查方法） |

**难度判定**：Easy = 不碰业务语义；Medium = 纯领域逻辑 / 单 crate / 规范编辑；Difficult = 跨层一致性 + 发布闭环。任务低于模型能力是浪费，高于能力是翻车；分派错误由派工方负责。

## 4. 开工硬门（任何任务，无论难度）

1. 读根 [`AGENT.md`](../AGENT.md)（宪法）与受影响部分的 `AGENT.md`；
2. 读对应 `specs/*.md` 与 `adr/*.md` 相关节；
3. 用 `docs/IMPLEMENTATION_MATRIX.md` / `docs/PROGRESS.md` 确认**唯一状态所有者与当前完成度**——禁止猜接口、猜业务、用默认值掩盖 missing fact；
4. 检查 git 状态：工作树必须干净（或明确说明接续的 dirty diff）；
5. 确认 Node 入口：`export PATH="$HOME/.local/share/pnpm/bin:$PATH"`（精确 Node 24.18.0 / pnpm 11.3.0；默认系统 node 是 26.x，禁止使用）；
6. 确认构建目录：`export TMPDIR="${TMPDIR:-$HOME/.cache/shittim-build-tmp}" CARGO_TARGET_DIR="${CARGO_TARGET_DIR:-$PWD/rust/target}"`，先 `mkdir -p`——任意可写目录即可，禁止写死单一 host 路径；本机推荐复用 `rust/target`（已有缓存且被 gitignore）。

## 5. 怎么维护文档（详见 `Easy/docs-maintenance-Easy/AGENT.md`）

- **状态单一来源**：完成度只写 `docs/IMPLEMENTATION_MATRIX.md` 与 `docs/PROGRESS.md`；`docs/api/*` 与 ADR 只保留一枚状态徽章 + 锚点链接。
- 每个切片同批更新受影响文档；只改代码不同步文档 = 未完成。
- 数字必须来自实测（测试数量、Schema 数量），禁止预写猜测。
- 文档变更后刷新 `FILE_MANIFEST.md`：`node scripts/update-file-manifest.mjs --write`。

## 6. commit 与 push 到哪（详见 `Easy/commit-and-push-Easy/AGENT.md`）

- **主仓**：`https://github.com/hongyue0721/shittim.git`（`master`，唯一权威源）；
- **文档镜像**：`https://github.com/hongyue0721/shittim-docs.git`（`master`，只读镜像，由工具同步）；
- **身份固定**：`小岳 <2933634892@qq.com>`（local config 与每个 commit 的 author/committer 都要一致）；
- **顺序**：主仓验收 → `git add -A` → 统一门 → 中文按功能域提交 → push 主仓并验证远端 SHA → `pnpm run sync:docs-repository` 同步镜像 → 两仓验证完成才可报告完成；
- **禁止**：强推未知远端、跳过门禁提交、身份不符提交、镜像先于主仓演化。

## 7. 怎么做规范（详见 `Medium/write-specs-Medium/AGENT.md`）

- 规范先行：先改 `specs/`（与 ADR），再实现，再测试，再文档；
- 一个事实只定义一次，其它文档只链接不复制；
- 章节编号/锚点是外部链接目标，禁止重排重编号；
- 每个不变量必须在 `specs/CONFORMANCE.md` 有自动化锚点；
- 未形成 Schema/实现/测试的方向只能标 future direction，不得写成已提供能力。

## 8. 审查（详见 `review/AGENT.md`）

每个 Difficult 任务与所有发布切片，在合并/推送前必须经过 `review/` 定义的独立审查：代码审查、合同一致性审查、文档同步审查、发布门审查。审查者必须是与开发不同的模型/会话，`inherit_context: false`，只读，实测为准。

## 9. 完成硬门（未满足不得报告完成）

1. 聚焦测试 + 全 workspace 测试绿；`cargo clippy --workspace --all-targets -- -D warnings` 绿；`cargo fmt --all -- --check` 绿；
2. `git add -A` 后 `./scripts/check-schema.sh` 全绿（Node 硬门 → generate×2 → check → fmt/clippy/test → generated 漂移 → FILE_MANIFEST）；
3. 文档、`FILE_MANIFEST.md`、MATRIX/PROGRESS 同切片更新完成；
4. 主仓提交/推送成功且远端 SHA == 本地 SHA；
5. 文档镜像同步完成且 `--check` 通过；
6. 工作区干净。

## 10. 通用禁止事项（全部部分）

- 未读仓库就臆想接口；用兜底默认值掩盖 missing fact；把特例伪装成通用能力。
- 手改 `rust/crates/kernel-contracts/src/generated/`；手写平行类型；复制平行语义。
- 制造「看起来成功但关系不一致」的数据；错误延后到更难排查的位置。
- 在未跑实际测试的情况下报告「完成」；把静态存在（表/Schema/文件）冒充业务闭环。
- 子代理 `inherit_context: true`（默认必须 `false`）；把超长父会话塞给子代理。
