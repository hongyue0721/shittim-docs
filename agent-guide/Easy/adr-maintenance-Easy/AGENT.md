# adr-maintenance-Easy — ADR 维护

> 难度：**Easy**（纯文档/决策记录维护，不碰实现）。进入本文件夹即表示任务属于「新建/更新/取代 ADR，或维护 `adr/README.md` 索引」。

## 目标

让架构决策有记录、有状态、可追溯，且**不形成第二套事实源**。完成标准 = 每篇 ADR 文首一枚状态徽章 + 指向 MATRIX/PROGRESS 的链接，索引与状态同步，无冲突事实。

## 开工前必读

1. 根 `AGENT.md`（§3「影响新常驻进程、核心协议、状态所有者、Core 边界或特权 Action 时写 ADR」）；
2. `adr/README.md`（状态语义与索引）；
3. 对应 `specs/*.md`（ADR 不得与之冲突）。

## 边界与职责

**拥有**：`adr/0001-0011` 及未来编号、`adr/README.md` 索引、ADR 状态徽章。

**不拥有**：字段级合同（归 `specs/`）、实现完成度（归 `docs/IMPLEMENTATION_MATRIX.md` / `docs/PROGRESS.md`）、代码。

## 硬性规则

1. **状态生命周期**：`proposed`（讨论中）→ `accepted`（已接受，可能未实现）→ `superseded`（被后续 ADR 替代）。**accepted ≠ 代码已完成。**
2. **徽章规则**：每篇 ADR 文首只允许一枚实现状态徽章（`designed` / `partial` / `implemented`）+ 指向 MATRIX/PROGRESS 的链接；**禁止**在 ADR 维护字段级或切片级完成清单。
3. **徽章必须与 MATRIX 同步**：实现落地后立即升徽章。已知实例：ADR-0011（Lease/Stop Fence）在 migration 0010 与 `acquire_lease` / `get_action_lease` 落地后应从 `designed` 升为 `partial`；再实现 `begin_dispatch` / `release_or_expire_lease` / Stop owner 后升 `implemented`（以 MATRIX/PROGRESS 实际状态为准）。
4. **何时必须写 ADR**：影响新常驻进程、核心协议、状态所有者、Core 边界、特权 Action 类别、技术栈/新 crate/新 workspace 依赖、Schema 数量基线变化。
5. **supersede 不改写历史**：被替代的 ADR 保留原文，在 `adr/README.md` 索引与相关 ADR 文首标注「部分被 ADR-XXXX supersede（具体节）」。
6. **编号与命名**：新 ADR 取下一空闲编号，中文文件名与既有风格一致（`NNNN-短横线标题.md`），并登记 `adr/README.md` 索引。
7. **不得引入第二套事实**：ADR 与根 `AGENT.md` 或 `specs/` 冲突时，ADR 让位，先修 ADR。
8. ADR 只拍板「决策与理由」，字段级定义归 `specs/`。

## 验收命令

```bash
export PATH="$HOME/.local/share/pnpm/bin:$PATH"
node scripts/update-file-manifest.mjs --write   # 新增/修改 md 后必做
node scripts/update-file-manifest.mjs --check
```

## 完成判定

- 新 ADR / 状态变更已登记索引；
- 徽章与 MATRIX/PROGRESS 一致；
- 与 `specs/`、根 `AGENT.md` 无冲突事实；
- 文档镜像同步按 `Easy/commit-and-push-Easy/AGENT.md` 完成。
