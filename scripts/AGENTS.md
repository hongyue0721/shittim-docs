# scripts/ AGENTS.md — 规范指针

> 本目录规范正文已集中到 `agent-guide/`。进入本目录工作前，先读总入口与本目录规范，两者都完整阅读后再动手。

- 总入口：[agent-guide/AGENT.md](agent-guide/AGENT.md)（权威顺序、难度分级、开工/完成硬门、通用禁止事项）
- 本目录规范：[Medium/scripts-toolchain-Medium/AGENT.md](agent-guide/Medium/scripts-toolchain-Medium/AGENT.md)

**铁律摘要**：零依赖 Node 标准库；精确 Node 24.18.0/pnpm 11.3.0（入口 ~/.local/share/pnpm/bin）；锁 O_EXCL 不自动清 stale；统一门失败修根因禁止绕过；双仓同步见 Difficult/dual-repo-sync-Difficult。

**状态与完成度**：一律以 docs/IMPLEMENTATION_MATRIX.md 与 docs/PROGRESS.md 为唯一来源；本指针不复述状态，避免漂移。
