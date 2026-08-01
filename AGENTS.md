# AGENTS.md — 仓库规则发现入口

> 本文件供会自动发现 `AGENTS.md` 的编码工具读取。**进入仓库后先读根 [`AGENT.md`](AGENT.md)（项目宪法）与 [`agent-guide/AGENT.md`](agent-guide/AGENT.md)（规范库总入口）**；然后按任务进入 `agent-guide/` 下对应部分的文件夹，打开其中的 `AGENT.md` 即可照着构建/维护。

## 权威顺序

1. 根 `AGENT.md`：全局不变量、可信边界、编码与发布硬门；
2. 对应 `specs/*.md`：字段、枚举、状态机、错误码的唯一事实源；
3. `adr/*.md`：规范允许范围内的实施决策；
4. `agent-guide/` 下受影响部分的 `AGENT.md`：边界职责、禁止事项与验收命令（按难度分级：Easy / Medium / Difficult，另有 `review/` 审查）；
5. 实现状态只看 `docs/IMPLEMENTATION_MATRIX.md` 与 `docs/PROGRESS.md`。

冲突时按上述顺序裁决；若同级事实源互相冲突，停止工作并上报，不得自行选择更顺手的一边。

## 开工与完成硬门

- 开工前读根 `AGENT.md`、`agent-guide/AGENT.md` 与受影响目录对应部分的 `AGENT.md`，并确认唯一状态所有者；禁止猜接口、猜业务或用默认值掩盖 missing fact。
- 代码、Schema、状态、错误或兼容语义变化时，同切片更新测试、MATRIX/PROGRESS 与相关 API/ADR/SDK 文档；刷新 `FILE_MANIFEST.md`。
- 验收按 `agent-guide/Medium/rust-workspace-Medium/AGENT.md`、`agent-guide/Medium/scripts-toolchain-Medium/AGENT.md` 与 `docs/REPOSITORY_MAINTENANCE.md`；统一门失败不得提交。Difficult 任务与发布切片须过 `agent-guide/review/` 独立审查。
- 主仓提交/推送、远端 SHA 核对、文档镜像同步与双仓校验全部完成前，不得报告任务完成（见 `agent-guide/Easy/commit-and-push-Easy/AGENT.md` 与 `agent-guide/Difficult/dual-repo-sync-Difficult/AGENT.md`）。
- 提交身份固定为 `小岳 <2933634892@qq.com>`；禁止强推未知远端、禁止跳过人工/CI 发布门直接改写历史。
