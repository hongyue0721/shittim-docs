# review-docs — 文档同步审查

> 审查文档是否与代码/规范同事实，是否符合状态单一来源纪律。

## 目标

确认：完成度只在 MATRIX/PROGRESS、api/ADR 只有徽章 + 锚点、数字来自实测、无第二套事实。

## 逐项清单

- [ ] **状态单一来源**：任何完成度陈述是否只在 `docs/IMPLEMENTATION_MATRIX.md` 与 `docs/PROGRESS.md`；api 页/ADR 是否有字段级复述（subject_hash、UUID 分配、projection 形状、IR 细节等）——有即违规；
- [ ] **徽章纪律**：api/ADR 是否一枚徽章 + 链接；徽章是否与 MATRIX 一致（如 ADR-0011 应为 `partial` 而非 `designed`）；
- [ ] **数字实测**：测试数量、Schema 数量是否有实测依据；有无预写猜测；
- [ ] **无速朽事实**：无「领先远端 N 提交」类表述；
- [ ] **HANDOVER**：`docs/DEVELOPMENT_HANDOVER.md` 的「下一实现任务」是否仍是旧任务（重复实现风险）；
- [ ] **FILE_MANIFEST**：`node scripts/update-file-manifest.mjs --check` 通过；增删 md 后是否已刷新；
- [ ] **锚点稳定**：specs 章节编号/标题未被重排；文档内链接未断；
- [ ] **诚实分级**：contract-only / library / composition / publicly-usable 等术语使用是否正确；「表存在/Schema 存在」是否被冒充为「已支持」。

## 已知漂移高危点清单（对照检查）

| 主题 | 正确事实 |
|---|---|
| Lease | acquire/get 已实现；dispatch/release/Stop owner 未实现 |
| 切片 4c | 9/11 High，剩 Lease 关联撤销 + 真实远程验签 |
| 五方法 handler | 未实现；KCP 不可连接 |
| Event 计数 | active 事件类型 5（task.created / action.state_changed / approval.state_changed / stop_fence.activated 等，以代码与文档口径为准）；「八 Schema」是资产数不是 active 类型数 |
| Node 入口 | `~/.local/share/pnpm/bin`（不是 `/node` 也不是系统 node） |
| 已知未裁决偏差 | 不得写成已修复 |

## 输出

按 `review-overview.md` §5 报告格式；每处漂移给「文档文件:行号 → 实际事实（证据文件:行号）」。
