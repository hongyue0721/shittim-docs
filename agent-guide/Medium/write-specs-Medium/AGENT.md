# write-specs-Medium — 怎么做规范

> 难度：**Medium**（编辑 `specs/` 与合同事实，需理解领域）。进入本文件夹即表示任务属于「编写/修订规范、给规范加验收锚点、处理规范与实现的冲突」。

## 目标

让 `specs/` 保持「唯一事实源」地位：每个字段/枚举/状态机/错误码只定义一次，可被实现精确消费，可被测试自动化验收。完成标准 = 规范先行、锚点齐备、无第二套事实、无过时完成度陈述。

## 开工前必读

1. 根 `AGENT.md`（§1 全局不变量、§3 编码规则、§4 完成条件）；
2. `docs/REPOSITORY_MAINTENANCE.md`（§2 持续更新义务、§2.1 状态单一来源）；
3. 受影响领域对应的 `specs/*.md` 全文（见下表和本文件夹对应部分的 `AGENT.md`）；
4. `schemas/` 相关 Schema（规范与 Schema 必须同事实）。

## `specs/` 领域地图（唯一事实源）

| 文件 | 领域 |
|---|---|
| `CORE_ARCHITECTURE.md` | 运行时、Task/Action、事件、恢复、Stop Fence |
| `SECURITY_PRIVILEGE.md` | Freedom-first Policy、入口、来源、特权与停止 |
| `IDENTITY_MEMORY_INITIATIVE.md` | 身份、记忆、探索、主动性、可生长对象 |
| `EXTENSION_SDK.md` | Extension 生命周期、边界、安装、调用、声明性风险 |
| `MODEL_RUNTIME.md` | Pi、模型路由、Context Pack、云调用记录 |
| `COMPUTER_USE.md` | 可选 Computer Use Profile 桌面事实源 |
| `IMPLEMENTATION_CONTRACTS.md`（IC） | 版本化对象、KCP、编码契约（§5.7 Error Catalog、§13.5 方法集、§13.6 切片记录、§13.7 谓词） |
| `CONFORMANCE.md` | 必须自动化的验收锚点 |
| `REFERENCES.md` | 直接依赖 / 可选 Provider / 仅参考项目边界 |

## 硬性规则（怎么做规范）

1. **规范先行**：改行为先改对应 spec（必要时配 ADR），再写实现；实现不得反向改变规范（README 权威性顺序、根 `AGENT.md`）。
2. **一个事实只定义一次**：字段、枚举、状态机、错误码只能在对应 spec 出现一次；README、`docs/api`、ADR、AGENTS.md 只链接不复制。
3. **锚点稳定**：章节编号与标题是外部链接目标（docs、ADR、AGENTS.md 大量引用 `§x.y` 与 `#anchor`）；禁止大规模重排/重编号。已知历史顺序问题（如 IC `§13.7` 物理位置在 `§13.6.12` 与 `§13.6.13` 之间）保持原样，只可在文字中说明，不得搬家。
4. **CONFORMANCE 同步**：新增/变更不变量必须在 `CONFORMANCE.md` 登记可自动化锚点；没有锚点的合同视为不可验收。改实现必须同步锚点，改锚点必须同步实现。
5. **状态不外溢**：完成度、切片进度只写 `docs/IMPLEMENTATION_MATRIX.md` / `docs/PROGRESS.md`；specs 内的切片小节（IC §13.6.x）是历史交付记录，改其中的「已实现/未实现」陈述必须与 MATRIX 一致。
6. **future direction 纪律**：未形成正式 Schema/实现/测试的方向只能标 future direction，不得写成已提供能力（根 `AGENT.md` §4）。Core MUST / claim-conditional Profile MUST / future direction 三类义务要区分。
7. **Error Catalog**（IC §5.7）是 KCP 错误唯一事实源：新增错误码先改这里，再改实现；实现硬编码的 code/message 与 catalog 冲突时以 catalog 为准收敛。
8. **Schema 联动**：规范涉及对象形状时，同步检查 `schemas/`（见 `Medium/schemas-manifest-Medium/AGENT.md`）：Schema 是唯一人工源，生成的 Rust 类型不可手改。
9. **版本/兼容语义**：v1 是历史验证路径，active 合同是 v2；旧合同按 ADR-0009/0010 直接退役，不写 adapter、不写 migration、不写「读后补写」。

## 已知未裁决偏差（写规范时注意，不得擅自定性为已修复）

- root 重放 request_id 与 IC §5.3.1「request ID 不参与幂等投影」的张力；
- Kernel 时钟（纳秒）与持久化秒级精度门冲突；
- `invalid_scope_pattern` 的 `input_kind`/`index` 在跨层映射中丢失；
- success criteria 的 `trim()` 与「精确多重集合」语义冲突。

以上都是**开放问题**：写规范时必须用「待拍板」语气或直接修订拍板（修订需说明理由），不得写成已修复。

## 验收命令

```bash
export PATH="$HOME/.local/share/pnpm/bin:$PATH"
git add -A
./scripts/check-schema.sh            # 规范改动通常伴随 Schema 联动，跑全量门
node scripts/update-file-manifest.mjs --check
```

## 完成判定

- 规范先行完成，锚点（CONFORMANCE）与实现/测试同步；
- 无第二套事实；状态陈述与 MATRIX/PROGRESS 一致；
- 门禁全绿；提交/推送/镜像同步按 `Easy/commit-and-push-Easy/AGENT.md` 完成；
- **spec 之间冲突（含同一文件两节冲突）是最优先级问题：停止实现类工作，列出冲突点由维护者拍板，编码 Agent 不得自行选择一边实现。**
