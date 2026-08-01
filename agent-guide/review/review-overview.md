# review-overview — 审查总流程与通用纪律

> 本文件定义一次审查从接单到出报告的完整流程。适用所有审查类型（code / contract / docs / release）。

## 1. 接单（准备）

1. 明确审查对象：commit 范围（`git log --oneline -n N`）、改动文件清单（`git diff --stat HEAD~N`）、涉及部分（对应 `agent-guide/*/AGENT.md`）；
2. 读根 `AGENT.md`（宪法）、受影响部分的 `AGENT.md`、对应 `specs/` 相关节；
3. 用 `docs/IMPLEMENTATION_MATRIX.md` / `docs/PROGRESS.md` 建立「当前应有状态」基线；
4. 列审查清单：按本文件 §4 逐项 + 对应类型文件（code/contract/docs/release）的清单。

## 2. 执行

- 逐文件、逐符号、逐断言核对；**只读**；
- 每项结论记：证据（文件:行号）+ 严重度 + 说明；
- 能交叉验证的必须交叉验证（如：claim「acquire_lease 已实现」→ 找到函数签名 + 调用链 + 测试名；claim「文档说完成」→ 与 MATRIX/PROGRESS 比对）。

## 3. 验证手段（只读、不运行破坏性命令）

```bash
git status --short                      # 工作区状态
git log --oneline -n 10                 # 提交范围
git diff --stat HEAD~N                  # 改动规模
# 读代码：ffgrep / read 目标符号与调用链
# 读测试：只读测试源码与测试名，不运行（除非审查范围明确允许且环境允许）
```

## 4. 通用审查清单（每项都要过）

| # | 检查项 | 怎么查 |
|---|---|---|
| 1 | 状态唯一 owner 是否成立 | 每个状态/事实只有一个写入口；repository 层无公开裸 CAS |
| 2 | fail closed 是否成立 | 缺失事实/非法输入 → 结构化错误，不是默认值/静默成功 |
| 3 | 是否存在「看起来成功」的假数据 | 写入路径是否有 canonical readback；回滚是否彻底 |
| 4 | 事件与状态是否严格 1:1 | 成功状态边是否恰好一条事件；有无「改状态不写事件」平行路径 |
| 5 | 时间戳精度是否统一 | 持久化边界是否 UTC 秒级；有无静默截断 |
| 6 | 错误是否闭集 | 错误码是否来自 IC §5.7；有无硬编码新 code |
| 7 | 幂等/重放语义是否自洽 | request_id 是否误入幂等投影；重放是否与合同一致 |
| 8 | 纯 crate 是否保持纯 | 无 IO/存储/时间/ID；事实是否 typed 注入 |
| 9 | 生成物是否禁手改 | generated/ 无手改痕迹；Schema 变更走生成链 |
| 10 | 文档是否同步 | MATRIX/PROGRESS 与代码一致；api 徽章无过时陈述 |

## 5. 报告格式（必须）

```text
# 审查报告：<对象>（commit <sha> 起）
## 结论：GO / NO-GO
## 严重度汇总：C=0 H=0 M=0 L=0
## Critical（如有）：<证据+影响+建议>
## High：...
## Medium：...
## Low：...
## 与基线不一致的文档声明（声明 vs 实现事实）
## 已运行命令
## Critical Files
## 给维护者的开放问题
```

## 6. 纪律红线

- 不修改、不提交、不推送、不安装、不联网；
- 不因为「开发方说完成了」就降低标准；不因为「看起来没问题」就跳过证据；
- 工具调用为 0 或秒级完成且无实质输出的审查，一律视为无效，必须重派；
- 结论必须可复核：每个 High+ 项都有文件:行号证据。
