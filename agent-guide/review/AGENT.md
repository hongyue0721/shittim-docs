# review/ AGENT.md — 审查入口

> **进入本文件夹后，先完整读本文件。** 审查是独立于开发的质检环节：任何 Difficult 任务与所有发布切片，在合并/推送前必须按本文件夹定义的方式审查。审查者的任务不是「帮开发说话」，而是**找出开发没发现的问题**。

## 审查为什么独立

Shittim 最大的历史风险是「文档看起来完成了但代码没有」、以及「模型把静态存在（表/Schema/文件）冒充业务闭环」。开发模型天然会为自己的产出辩护；所以审查必须由**不同的模型/会话**执行：`inherit_context: false`、只读、以实测为准。

## 何时审查

| 场景 | 必须审查 |
|---|---|
| Difficult 任务（kernel-sqlite / kernel-kcp / schema-tool / dual-repo-sync） | 是 |
| 涉及 Schema / 合同 / 状态 owner / 发布门的任何改动 | 是 |
| Easy/Medium 纯文档或纯领域改动 | 抽查（推荐但非强制） |
| 每次发布（push 前） | 是（发布门审查） |

## 审查类型（本文件夹各文件）

1. [`review-overview.md`](review-overview.md)：审查总流程、通用纪律、验收等级（0 Critical / 0 High / 0 Medium）、报告格式；
2. [`review-code.md`](review-code.md)：代码审查清单（状态 owner、CAS 私有、事件 producer、fail closed、时间精度、错误闭集、无 panic）；
3. [`review-contract.md`](review-contract.md)：合同一致性审查（spec ↔ Schema ↔ generated ↔ 实现 ↔ fixtures 逐层对账）；
4. [`review-docs.md`](review-docs.md)：文档同步审查（MATRIX/PROGRESS 唯一状态源、徽章、数字实测、已知漂移高危点）；
5. [`review-release.md`](review-release.md)：发布门审查（统一门、generated 漂移、FILE_MANIFEST、双仓 SHA、身份、镜像闭集）。

## 审查者纪律（全部类型通用）

- `inherit_context: false`；prompt 必须自包含（工作区、审查对象、范围、约束、输出格式）；
- **只读**：不修改、不创建、不提交、不推送、不安装依赖、不联网；
- **实测为准**：声称「测试存在」必须读到测试源码与测试名；声称「实现存在」必须读到代码符号与调用链；未运行测试就不得说「测试通过」；
- **不猜**：接口、业务、状态一律以源码/spec/MATRIX 为证，找不到证据就写「未发现证据」；
- **交叉验证**：文档声明 vs 实现事实分开标注；同一结论至少一个证据路径；
- 发现「表/Schema/文件存在」≠「业务闭环」时，必须指出缺的 owner/调用链/事件/验证；
- 报告格式：中文、分点、附绝对路径与行号、按严重度（Critical/High/Medium/Low）分级、最后给 Critical Files 与结论（GO / NO-GO）。

## 严重度定义

- **Critical**：数据/事实一致性被破坏、安全边界失效、幂等/重放/时钟合同冲突、发布后不可逆损害；
- **High**：合同与实现漂移、文档宣称完成但功能缺失、错误闭集不完整、跨层信息丢失；
- **Medium**：状态表述过时、缺少测试锚点、可维护性/清晰度问题；
- **Low**：命名/注释/风格。

## 完成判定

审查结论必须显式写 **GO**（0 Critical / 0 High / 0 Medium）或 **NO-GO**（列出全部阻断项与证据）。NO-GO 时开发方必须修根因后重新审查，不得由开发方自行翻案。
