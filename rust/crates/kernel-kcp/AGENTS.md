# kernel-kcp/ AGENTS.md — 规范指针

> 本目录规范正文已集中到 `agent-guide/`。进入本目录工作前，先读总入口与本目录规范，两者都完整阅读后再动手。

- 总入口：[agent-guide/AGENT.md](agent-guide/AGENT.md)（权威顺序、难度分级、开工/完成硬门、通用禁止事项）
- 本目录规范：[Difficult/kernel-kcp-Difficult/AGENT.md](agent-guide/Difficult/kernel-kcp-Difficult/AGENT.md)

**铁律摘要**：Value 层库级，不碰 bytes/socket；v1 create → unsupported_schema_version / InternalContractViolation fail closed；错误码以 IC §5.7 为唯一来源；五方法缺 handler 前禁止 server。完成度开工前查 MATRIX/PROGRESS。

**状态与完成度**：一律以 docs/IMPLEMENTATION_MATRIX.md 与 docs/PROGRESS.md 为唯一来源；本指针不复述状态，避免漂移。
