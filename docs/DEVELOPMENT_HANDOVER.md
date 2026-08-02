# 开发交接手册

> 本文档面向在新设备上继续 Shittim 项目的开发者。clone 后按此手册准备环境、遵循流程、认领下一步任务。

## 1. 当前状态快照

- **主仓库**：<https://github.com/hongyue0721/shittim>（`master`）
- **文档镜像仓库**：<https://github.com/hongyue0721/shittim-docs>
- **本轮代码验收基线**：`e53c9ef`（已推送 `origin/master`）；继续开发时以已推送的 `origin/master` 为准，并要求工作树干净。本地另有已提交待推送切片：`domain-task` Approval v2 resolution 领域表达（`222e32f` 起 4 个功能域提交）与 Lease/Stop Fence 并发切片（`begin_dispatch` / `release_or_expire_lease` 本地已实现、未提交）。
- **下一实现任务**：`task.list` / `event.subscribe` / `event.poll` 正式 handler 与 §13.7 完全闭合（`stop.activate` / `stop.status` 已随切片6 落地）。切片 5（migration 0011 child materializer：原子物化 bundle + 五钉闭包 FK + 严格 readback，独立审查三轮闭合 0C/0H/0M）与 Lease/Stop Fence 全链、4c 清零（Approval invalidation 同事务撤销 Lease、真实远程验签）均已落地，不得重复实现。
- **同步状态**：主仓 push 与文档镜像 receipt 由 `scripts/sync-docs-repository.mjs` 校验；不得在文档中维护会随一次 push 立即失真的“领先提交数”。
- **domain-task 领域层**：Approval v2 resolution 领域表达已落地（`approval_resolution_ref` 仅限 `pending→approved` 消费且要求 `permission_decision_ref` 非空，deny/confirm 携带 fail closed；CORE §11.3 已明确 confirm 非状态边）；kernel-sqlite 侧对接领域 `approval_resolution_ref`（当前用自己的 intent 字段投影 payload）为后续挂账项。
- **里程碑 `V2InitialBuildActive`（ADR-0009）** 已完成或部分完成切片：
  0 规范手术；1a root v2 持久对象 Schema×4；1b Action/child Schema×5 + `kernel-authorization` 纯库；1c-i 授权核心五 Schema；1c-ii 身份/挑战/证据八 Schema；2 root TaskCreate v2 仓库/migration 0004；3a production MethodVersionBindings 八方法集；3b KCP 切 active v2 删 v1 路径；3c sqlite v1 写路径删除/v2-only Outbox/旧库 reinitialize-required 拒绝；4a Action 仓库+`action.state_changed`/migration 0006；4b PolicyRule+PermissionDecision 仓库+评估编排/migration 0007；4c Approval/Identity 仓库与安全闭环全部完成（11/11 High，migration 0008；Approval 撤销 Lease 与真实远程验签已随 c11d3be 落地）。
- **测试基线（主会话实测）**：`kernel-sqlite` 155；全工作区测试绿；`cargo clippy --workspace --all-targets -- -D warnings` 绿；`./scripts/check-schema.sh` 全量门绿。后续改动后以实际测试输出更新，不得预写猜测数量。
- **Schema 数量**：当前`schemas/manifest.json`精确`80 = 38 retained + 42 component-native`；历史1c-ii快照曾为83/41。ADR-0010已直接退役三项未投产旧Policy合同。
- **事实单一来源**：`docs/IMPLEMENTATION_MATRIX.md` 与 `docs/PROGRESS.md`，ADR/API 文档只保留状态徽章与锚点链接。

## 2. 新设备环境准备

1. 安装 Rust toolchain（`rustup`），确保 `cargo`、`rustc` 在 PATH。
2. 安装 Node 与 pnpm：
   - 推荐已存在 `Node 24.18.0` + `pnpm 11.3.0`；Node runtime 路径加入 `export PATH="$HOME/.local/share/pnpm/bin:$PATH"`。
   - 若 Node 缺失，使用 pnpm 用户级 runtime 安装：`pnpm env use --global 24.18.0`。
   - 校验：`node scripts/check-node-toolchain.mjs`（读取 `package.json` 的 `engines` 与 `packageManager`，要求精确版本）。
3. 设置大/临时产物目录到任意可写位置（推荐复用 gitignore 的 `rust/target` 与 `$HOME/.cache`）：

   ```bash
   export TMPDIR="${TMPDIR:-$HOME/.cache/shittim-build-tmp}"
   export CARGO_TARGET_DIR="${CARGO_TARGET_DIR:-$PWD/rust/target}"
   mkdir -p "$TMPDIR" "$CARGO_TARGET_DIR"
   ```

4. 首次验证命令（clone 后执行）：

   ```bash
   git rev-parse HEAD            # clone时以当前远端HEAD为准
   git status --short            # 新clone应为空；接续dirty任务时必须保留既有diff
   node scripts/check-node-toolchain.mjs
   cat schemas/manifest.json | python3 -c "import json,sys; print(len(json.load(sys.stdin)['schemas']))"
   # 当前应输出 80
   ```

5. 验收门前的特殊动作：
   - `git add -A`：因为 `./scripts/check-schema.sh` 的 generated-tree drift 检查对 index 做 diff，未 stage 的文档/生成物会被误判。
   - 文档变更后必须执行 `node scripts/update-file-manifest.mjs --write`，刷新 `FILE_MANIFEST.md`。

## 3. 标准工作流程

每个切片按以下顺序闭环，详情见 `docs/REPOSITORY_MAINTENANCE.md`：

1. **文档先行**：先更新规范/contract/ADR/API 文档，再写实现。
2. **实现**：按 contract 编码，优先复用现有仓库与模式。
3. **测试**：聚焦测试 + 全工作区 `cargo test`。
4. **独立验收**：必须 `0 Critical / 0 High / 0 Medium` 才 GO。
5. **提交**：中文、按功能域的提交；`./scripts/check-schema.sh` 全绿后再提交。
6. **推送**：验证远端 SHA 与本地已验收 SHA 一致。
7. **同步文档镜像**：`pnpm run sync:docs-repository` 并带 `--check` 验证镜像文件集合、内容与 `FILE_MANIFEST.md` 与主仓完全一致。

## 4. 下一步任务一：Lease/Stop Fence 持久化（ADR-0011）

这是切片 5 的硬前置。语义由 [`adr/0011-lease生命周期与stop-fence原子语义.md`](../adr/0011-lease生命周期与stop-fence原子语义.md) 拍板，实施依据为 [`docs/design/lease-stop-fence-blueprint.md`](design/lease-stop-fence-blueprint.md) v2。单元 1 持久化基座与领域 Lease release effect 已完成，接续任务不得重写 migration 0010 或把表存在误报为 owner 已完成。

### 4.1 实施单元与提交顺序

1. **单元 1（已完成，`18b03f1`）**：migration 0010 三表、descriptor/关系校验、CAS/不可变守卫、`max_uses=1` 生成链与统一业务 COMMIT 前关系闭包；独立 Gemini GO（0C/0H/0M/1L）。
2. **单元 2 前置合同（已完成）**：`ActionTransitionIntentV1.execution_generation` 已拆分为 `expected_execution_generation` / `resulting_execution_generation` 支持 acquire 的 `G→G+1`；Stop causation 是持久 intent 的同一事实 alias，不是动态 allocator 的独立 UUID。
3. **单元 2 owner（已完成）**：`acquire_lease` / `get_action_lease`（`approved → leased` + 严格只读）已按 ADR-0011 §7 协议落地。
4. **单元 3（已完成）**：`begin_dispatch` / `release_or_expire_lease` 已落地，消费 `domain-task` 的封闭 `LeaseReleaseReason` / `LeaseReleaseEffect`（`LeaseCommitProjection::Clear` 先删 Lease 级联删锁再 CAS 离开 lease-bearing 状态）。
5. **单元 4（已完成）**：`activate_stop_fence` / `get_stop_fence` 已落地；v1 原子范围 + transaction-bound allocation source + canonical Actor 快照。
6. **单元 5（已完成）**：4c 清零——Approval invalidation/replacement 同事务按 ADR-0011 §3 分流 Action（leased 撤销 Lease 驱动 `leased→cancelled`，approved 不动、in_flight 不打断）。

并行线：Ed25519 `RemoteSignatureVerifier`（ADR-0011 §10），在 4c 最终验收前合流。

每单元独立验收 `0 Critical / 0 High / 0 Medium` 后才提交；`max_uses` 的 Schema 收紧（`const 1`）随单元 1 同批过生成链门禁。

## 5. 切片 5 child materializer —— 已完成

> **状态（2026-08-02）**：切片 5 已落地并通过独立审查（三轮 NO-GO 修复闭合，终评 GO 0C/0H/0M）。实现：`WriteTransaction::materialize_child_task`（migration 0011 原子物化 bundle：完成态 Action + 子 Task/TaskScope/ContentOrigin/Provenance + VerificationResult/Audit + `task.created`(sequence=0, causation=Action)/`action.state_changed` 两事件）；proposal==structured_arguments 与 delta==评估指纹双命脉校验；mapping 五钉闭包 FK（transition/两 event/scope/origin）；严格 readback（fresh/replay/reconcile 同一校验函数 + 确定性重建字节比对）；approval resolution 消费全链；幂等四分支。repository 闭集：`materialize_child_task` / `get_child_materialization_by_action` / `get_child_materialization_by_child_task` / `reconcile_child_materialization`。**以下原文为历史任务定义，供追溯，不再作为待办。**

### 5.1 目标

- **子 Task 唯一创建路径**：由 parent 发起 `kernel.task/task.child.create` Action（S1 kernel.task），经 Policy 评估后，由 Kernel 原子物化。
- root 直接创建只能建 root；v2 已强制 root-only，child 不再通过 KCP/TaskCreate 直接写入。

### 5.2 已有资产

- Schema 与生成类型：`ChildTaskProposalV1`、`NormalizedChildTaskProposalV1`、`ChildTaskMaterializationAllocationV1`（`schemas/`、`kernel-contracts` 生成类型）。
- `kernel-task-creation` 纯库：child 规范化、hash、10 UUID allocation 校验与官方 fixtures：
  - `schemas/fixtures/task/child_task_proposal_normalized_hash.v1.json`
  - `schemas/fixtures/task/task_creation_allocations.v1.json`
- `kernel-authorization` 投影：`project_child_task_delta`（`rust/crates/kernel-authorization/src/child_delta.rs`）、`project_material_authorization`（`material.rs`）、`project_observation_evidence`（`observation.rs`）。
- 已闭合的持久化基础设施：Action 仓库（4a/migration 0006）、PolicyRule+PermissionDecision 仓库+评估编排（4b/migration 0007）、Approval 三 CAS 方法+Identity 仓库（4c/migration 0008）。

### 5.3 待实现

1. `kernel-sqlite` 新增 child materialization 方法；如需要新表，按 migration 0004-0009 的 descriptor 模式写 `migrations/0011_child_materialization.sql`（0009 已是 Action-PD heads，0010 分配给 Lease/Stop Fence，ADR-0011）。
2. **原子性**：同一 `BEGIN IMMEDIATE` 事务内完成：
   - 创建 Action 标记为完成；
   - 写入子 `Task`、`TaskScope`、`ContentOrigin`、`Provenance`；
   - 写入 `VerificationResult`、`AuditRecord`、Outbox 事件。
3. 子 `Task` 的 sequence 0 事件为 `task.created`，其 `causation` 必须指向父 Action，不得伪造 command 或 event carrier。
4. 所有 UUID allocation 由上层注入，并经 `kernel-task-creation` 的 typed fixture/validator 校验；拒绝自由 UUID bag。
5. 严格 readback：事务提交后返回物化的子 Task 全量视图，供调用方验证。readback 必须至少覆盖：
   - `task` 表的 `task_id`、`parent_task_id`、`goal` 摘要；
   - `task_scope` 与 `content_origin` 的关联完整性；
   - `provenance` 的 sequence 0 与 `causation_ref` 指向父 Action；
   - `audit_record` 的 carrier 为 action 且 receipt hash 覆盖 `NormalizedChildTaskProposalV1` 的 JCS bytes；
   - `outbox` 的 `task.created` 事件已生成且 event_id 与 allocation 注入的预期一致。

6. 禁止引入新的 v1 路径或允许直接写 child 的 API；任何新增 public 方法必须有对应的 contract 测试。

### 5.4 合同锚点

- `specs/IMPLEMENTATION_CONTRACTS.md`：§5.3（child proposal 与 TaskCreate 字段同构）、§13.6（切片 1b/2/4a-4c 的约束与仓库边界）、§13.7（`V2InitialBuildActive` 谓词清单）。
- `specs/CORE_ARCHITECTURE.md` §9.2（child task 父子关系与权威入口）。
- `specs/CONFORMANCE.md` 中关于 child proposal 的字段约束、allocation 互异校验与 Schema-valid/domain-invalid 边界（第 3 章与第 5 章）。
- `adr/0006-child-task权威与taskcreate-v2迁移.md`（`task.child.create` 是 Kernel-local Action operation、atomic materialization、allocation 结构）。
- `adr/0009-v2从零构建并取消v1数据迁移.md`（v2-only 基线，取消 v1 迁移）。

### 5.5 验收门槛

- 新增测试全绿；`kernel-task-creation` 的 allocation/validator 测试覆盖边界；`kernel-sqlite` 新增 integration 测试覆盖原子事务与 readback。
- `./scripts/check-schema.sh` 全量门绿。
- 独立验收 `0 Critical/High/Medium` 后才提交。
- 提交前：`git add -A && ./scripts/check-schema.sh`，确保 generated tree 与 `FILE_MANIFEST.md` 都已 stage 并通过漂移检查。

## 6. 切片 6 最终清理 + §13.7 闭合 —— 部分完成

> **状态（2026-08-02）**：`stop.activate` / `stop.status` 已随切片6 落地并通过独立审查（修复 M1/M3/L1 后代码侧闭合，统一门全绿）。`task.list` / `event.subscribe` / `event.poll` 三方法仍无 handler；§13.7 完整闭合待这三个 handler。**以下原文为历史任务定义，供追溯。**

1. 逐条核对 `specs/IMPLEMENTATION_CONTRACTS.md` §13.7 的谓词；未闭合的项必须显式记录，不得隐藏。
2. 只有在 §13.7 全部闭合后，才解锁以下三方法 handler：
   - `task.list`
   - `event.subscribe`
   - `event.poll`
3. 这些 handler 各自有前置合同，§13.7 纪律要求：谓词未闭合不得启动 server。

## 7. 再后续路线图

仅概述，五方法 handler 后才能进入：

- `task.list`：游标/排序合同。
- `event.subscribe` / `event.poll`：订阅关系与 Outbox 游标合同。
- `stop.activate` / `stop.status`：stop fence 持久化合同已由 ADR-0011 闭合（v1 原子范围、动态 allocation、canonical Actor 快照）；实现按蓝图 v2 单元 4。
- Publisher loop。
- 多语言 SDK。
- 桌面端。

## 8. 验收债务（必须先补）

切片 4b 与 4c 因子代理通道故障由主会话亲自实现并自验提交。当前状态：

- **4b/Policy v2**：已经独立复核并完成根因重构——`PolicyRuleV2` 直接匹配、五种 confirmation mode、remote_signature 正常参与排序、强制 TaskScope 包含、UUID 用途互异、PermissionDecision 原子绑定（migration 0009）。旧三合同已按 ADR-0010 退役。
- **4c 复核结果（原提交 `a72a757`）：NO-GO，11 项 High；现已闭合 9 项，剩 2 项**。已闭合：operation Approval subject 由仓储基于当前绑定 PD 派生且单事务创建（`ef6967b`）；Challenge 过期 CAS/审计与 credential/evidence identity 审计（`6b30e40`）；真实 confirmation mode、denied 审计 `blocked`+PD/policy context、local/system 证据完整绑定、Challenge 消费与 resolution 原子事务（`8ec55ea`）；每个顶层 Approval/Policy 复合命令以唯一 transaction-bound typed collector，在任何写入前闭合 command allocations 与该命令实际读取、验证或消费的 persisted Task/Scope/Action/PD/PolicyRule/request/challenge/evidence/credential UUID 用途（`de5a8da`）。跨层 Challenge=ActionEvent、Task=ActionTransition、新 PD=Task/current PD、错误归属过期 Challenge、RFC3339 offset 与 Remote 无 verifier 全回滚均有回归。剩余：Approval invalidation/replacement 的 Lease 关联撤销（依赖 Lease 持久化）与真实 remote_signature 验签（当前保持 `ApprovalRequired` 失败关闭）。**（历史记录：两项均已随 c11d3be 闭合，4c 11/11 High 完成，本条不再代表当前状态）**
- **终验说明**：独立 Terra 已对 `de5a8da` 给出正式 **GO（0 Critical / 0 High / 0 Medium / 0 Low）**，逐入口复核唯一 transaction-bound typed UUID collector 的创建、借用、用途登记与 first-write 顺序，并独立运行 `kernel-sqlite` 144/144 与 clippy 全绿；主会话另完成 workspace fmt/clippy/tests 与 `check-schema.sh` 全量门禁。

**路线依赖警告**：4c 中「resolve/invalidate 必须撤销 Action Lease」在 Lease 持久化不存在时无法真正关闭。因此顺序为：4c 非 Lease 修复 → Lease/Stop Fence → 4c Lease 关联复核 → child materializer → §13.7/handlers。不得在未清零 4c 的基础上叠加物化逻辑。

## 9. 已知问题

- `schema-tool` 测试 `artifact_transaction lock_conformance real_cross_process_holder_crash_releases_fd_lock` 在并行高负载下偶发时序 flake，单跑必过。与业务逻辑无关，待修复。

## 10. 子代理使用政策

- `inherit_context` 必须为 `false`。
- prompt 必须自包含：工作区、目标、范围、约束、步骤、验收标准、输出格式。
- 子代理报告返回后，必须对照实际工作区核验（曾出现假阳性与假阴性）。
- 推荐模型分工以 `docs/REPOSITORY_MAINTENANCE.md` 与主会话政策为准：快速探索用轻量模型，合同/推理用强模型，实现继承父会话或指定 `proxy/grok-4.5` 等。
- 给子代理的 `prompt` 必须显式禁止：
  - 未读仓库就臆想接口；
  - 用兜底默认值掩盖 missing fact；
  - 修改超出 prompt 范围的文件；
  - 未实际跑测试就报告“完成”。
