# kernel-sqlite/ AGENTS.md — 规范指针

> 本目录规范正文已集中到 `agent-guide/`。进入本目录工作前，先读总入口与本目录规范，两者都完整阅读后再动手。

- 总入口：[agent-guide/AGENT.md](agent-guide/AGENT.md)（权威顺序、难度分级、开工/完成硬门、通用禁止事项）
- 本目录规范：[Difficult/kernel-sqlite-Difficult/AGENT.md](agent-guide/Difficult/kernel-sqlite-Difficult/AGENT.md)

**铁律摘要**：文件型唯一；migration descriptor 精确稳定；BEGIN IMMEDIATE + savepoint poison；事件写入 crate-private 仅 owner producer；时间戳 UTC 秒级；旧库 reinitialize-required；Delegation fail closed。完成度开工前查 MATRIX/PROGRESS。

**状态与完成度**：一律以 docs/IMPLEMENTATION_MATRIX.md 与 docs/PROGRESS.md 为唯一来源；本指针不复述状态，避免漂移。
