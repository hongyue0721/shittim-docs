# domain-task/ AGENTS.md — 规范指针

> 本目录规范正文已集中到 `agent-guide/`。进入本目录工作前，先读总入口与本目录规范，两者都完整阅读后再动手。

- 总入口：[agent-guide/AGENT.md](agent-guide/AGENT.md)（权威顺序、难度分级、开工/完成硬门、通用禁止事项）
- 本目录规范：[Medium/domain-task-Medium/AGENT.md](agent-guide/Medium/domain-task-Medium/AGENT.md)

**铁律摘要**：纯状态机；字符串比较不做 trim/Unicode 改写（既有 trim 偏差未拍板，不得修不得仿效）；需要 Lease 效果的边只产出 typed effect。

**状态与完成度**：一律以 docs/IMPLEMENTATION_MATRIX.md 与 docs/PROGRESS.md 为唯一来源；本指针不复述状态，避免漂移。
