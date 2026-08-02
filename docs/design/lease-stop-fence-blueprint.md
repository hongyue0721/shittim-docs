# Lease / Stop Fence 持久化实施蓝图（阶段 B，v2）

> 状态：**implemented**。`domain-task` Lease release effect、generation/intent 合同拆分与 `acquire_lease` / `get_action_lease` / `begin_dispatch` / `release_or_expire_lease` / `activate_stop_fence` / `get_stop_fence` 及 Approval invalidation 同事务撤销 Lease 全部落地并独立测试，`approved → leased → in_flight → completed` 端到端可走通；本文件作为历史实施依据保留。
> v2 说明：按 [ADR-0011](../../adr/0011-lease生命周期与stop-fence原子语义.md) 修订。v1 草稿存在九处与合同冲突的设计（Lease 只在 `leased` 存在、双源未闭合、触发器循环、单列 FK、`max_uses` 收紧无依据、Fence generation 递增、只存 actor id、Stop 副作用降格、动态 allocation 无来源），已全部修正；语义以 ADR-0011 为唯一权威。

## 1. 为什么这是硬前置

`approved → leased → in_flight → completed` 是副作用 Action 的唯一合法执行链。当前 `domain-task`
已完整定义该链的边与 typed effects，migration 0010 已建立 Lease/Resource Lock/Stop Fence 关系、
Schema/descriptor/guards、统一 COMMIT 前关系闭包与回滚证明；但 `kernel-sqlite` 仍没有任何
命名 Lease/Stop owner，`reject_unhandled_action_effects` 仍对 release effect 失败关闭。结果：

- Action 无法离开 `approved`，切片 5 child materializer 无法完成 Action；
- 切片 4c 中「resolve/invalidate 必须撤销受影响 Lease」无法真正关闭；
- §13.7 谓词 4 无法为真。

## 2. 已有契约资产（不需新建）

| 资产 | 位置 | 说明 |
|---|---|---|
| `ActionRequestV2.lease` | 生成类型 `ActionRequestV2Lease` | **Lease 是 Action 的内联字段**，非独立 Schema：`{holder, generation, expires_at, max_uses}` |
| `execution_generation` | `ActionRequestV2` 顶层字段 | 每次 Lease/执行尝试单调增加；不扩大 Action 业务唯一性 |
| `LeaseReleaseEffect` | `domain-task/src/action.rs` | 纯领域效果：`invalidate_lease` + `release_all_resource_locks`；`leased | in_flight` 的六条退出边均已产生，并以封闭 `LeaseReleaseReason` 表达原因；SQLite owner 尚未消费 |
| `StopFenceActivatedPayload` | `schemas/source/event/stop_fence_activated_payload.v1.json` | 已存在；`{generation, reason, activated_by_actor_id, activated_from_entry_point, activated_at}` |
| `EventAggregateId::StopFenceGlobal` | `kernel-sqlite/src/outbox.rs` | 单例全局聚合已就位 |
| 错误码 | 错误目录 | `lease_not_found` / `lease_expired` / `lease_holder_mismatch` / `action_not_executable` / `stop_fence_active` / `fence_generation_mismatch` / `approval_invalidated` 均已定义 |

**结论：阶段 B 不新增业务 Schema。** 只需 migration 0010 + 仓储编排 + `max_uses` Schema 收紧
（ADR-0011 §2，`const 1`，随 0010 实现提交过生成链门禁）。Resource Lock 是 Kernel 内部并发
治理事实而非跨进程契约对象，按内部表实现（§3.2）。

## 3. migration 0010 表设计（已实现）

`18b03f1` 已按 migration 0009 的 descriptor v1 风格落地：schema 阶段建表、Rust 关系校验、guards 阶段建触发器；既有 Action 中出现 lease、`leased|in_flight` 或非零 `execution_generation` 即 `reinitialize-required`。统一 `with_write_transaction` 成功出口在有执行事实时校验 Action 内联 Lease、Lease 行、资源锁精确集合、Stop 收敛与外键闭包；半 stage、半 delete、Fence 半收敛均整体回滚。SQL 对 Stop Actor 只验证结构，RFC 8785 canonical bytes 仍由未来 Stop owner 在写入和 readback 时验证。

### 3.1 `action_leases`（每 Action 至多一个活跃 Lease）

```sql
CREATE TABLE action_leases (
    action_id TEXT PRIMARY KEY,
    holder TEXT NOT NULL,
    generation INTEGER NOT NULL CHECK(generation >= 1),
    expires_at TEXT NOT NULL,
    max_uses INTEGER NOT NULL CHECK(max_uses = 1),   -- ADR-0011 §2：v1 固定 1
    acquired_at_action_revision INTEGER NOT NULL CHECK(acquired_at_action_revision >= 1),
    UNIQUE(action_id, generation),
    FOREIGN KEY(action_id) REFERENCES actions(id)
);
```

不变量与触发器：

- **生命周期跨 `leased | in_flight`**（ADR-0011 §1）：`actions.status` 离开这两个状态时
  必须无残留 Lease 行；`actions` UPDATE trigger 强制。
- **双源一致**（ADR-0011 §8）：Lease 行与 `actions.record_json` 内联 lease 逐字段相等，
  且 `generation == actions.execution_generation`；不一致 `stored_data_invalid`，不补写。
- **CAS 释放**：释放/过期必须匹配 `(action_id, holder, generation)`，否则
  `lease_holder_mismatch`。
- **禁止裸 UPDATE 改 holder/generation**：Lease 不可就地改主；重取必须递增
  `execution_generation` 后重新插入。

### 3.2 `action_resource_locks`（Lease 持有的资源锁）

```sql
CREATE TABLE action_resource_locks (
    resource_ref TEXT PRIMARY KEY,      -- 规范化 URI；全局互斥
    action_id TEXT NOT NULL,
    generation INTEGER NOT NULL,
    FOREIGN KEY(action_id, generation)
        REFERENCES action_leases(action_id, generation) ON DELETE CASCADE
);
```

- **复合 FK**（ADR-0011 §7）：锁行 generation 与父 Lease 强绑定，杜绝单列 FK 下的
  generation 漂移。
- `resource_ref` 为主键即表达**逻辑资源全局互斥**；冲突插入映射为 `action_not_executable`
  （附冲突 resource）。
- `ON DELETE CASCADE` 保证「释放 Lease 的同一原子变更中释放其全部 Resource Lock」。
- 资源 URI 必须复用 `domain_policy::normalize_uri`，与 TaskScope/PD 指纹同一规范化。

### 3.3 `stop_fence`（单例，含完整 Actor 快照）

```sql
CREATE TABLE stop_fence (
    id INTEGER PRIMARY KEY CHECK(id = 1),
    generation INTEGER NOT NULL CHECK(generation >= 1),
    reason TEXT NOT NULL,
    activated_by_actor_json TEXT NOT NULL,   -- canonical Actor JSON（ADR-0011 §6）
    activated_by_actor_id TEXT NOT NULL,     -- 镜像投影
    activated_from_entry_point TEXT NOT NULL,
    origin_ref TEXT,                          -- nullable；非空必须存在对应 ContentOrigin
    activated_at TEXT NOT NULL
);
```

- `CHECK(id = 1)` 表达单例；**行不存在 = 未激活**。
- 首版激活插入 `generation = 1`；**重复激活不 UPDATE、不递增**，repository canonical
  readback 返回原事实（ADR-0011 §4）。未来「清除后再次激活」合同出现前 generation 恒为 1。
- `stop.status` 响应的完整 `Actor` 从 `activated_by_actor_json` 构造；事件 payload 只投影
  `activated_by_actor_id`（Schema 已定）。

## 4. 仓储 API 设计（已实现）

编码这些 owner 前必须先闭合两个合同冲突：

1. 当前 `ActionTransitionIntentV1.execution_generation` 同时被 fresh commit 当作 pre-CAS 与 post-CAS generation；普通边相等，但 acquire 必须 `G→G+1`。必须通过明确的 typed generation projection 或合同字段重构表达，禁止在校验器里加 acquire 特例绕过。
2. Stop transaction-bound allocator 不独立分配 `causation_ref`；Action event causation 必须由同事务持久化的 ActionTransitionIntent 派生，UUID collector 只把两处登记为同一事实 alias。

其余 owner 全部经 crate-internal 机械 commit 进入唯一状态事件权威；裸 CAS 不暴露。

| 入口 | 职责 | 事务闭包 |
|---|---|---|
| `acquire_lease` | `approved → leased` | 读 Stop Fence（激活则 `stop_fence_active`）→ 按 ADR-0011 §7 协议插 Lease（generation+1）→ 插全部 Resource Lock（冲突失败关闭）→ Action CAS（revision/generation/status/内联 lease 一体）→ intent + `action.state_changed` |
| `begin_dispatch` | `leased → in_flight` | CAS holder + generation + exact `expires_at`（`now >= expires_at` 即过期）→ 事务内**再次读取** Fence generation → 状态边 CAS + 事件；Fence 变化返回 `fence_generation_mismatch` |
| `release_or_expire_lease` | `leased → approved`（仅 `lease_expired`）/ `leased → cancelled`（dispatch 未开始）/ `leased → unknown_side_effect`（dispatch 不确定） | CAS `(action_id, holder, generation)` → 删 Lease（级联删锁）→ 状态边 CAS + 事件 |
| `activate_stop_fence` | 激活围栏并收敛受影响 Action | ADR-0011 §4/§5：插单例 + `stop_fence.activated` → transaction-bound allocation source 为每个受影响 Action 造 ID → `leased→cancelled`、`in_flight→unknown_side_effect` 各一条 intent + 事件，原子删 Lease/锁 |
| `get_stop_fence` / `get_action_lease` | 严格只读 | canonical 闭包校验；破损 → `stored_data_invalid` |

**关键不变量**：

- Lease 消费必须 CAS `holder + generation + expires_at`（IC §6.10.6）；
- Stop Fence generation 必须在执行事务内**再次读取**，不能用事务外快照；
- `domain-task` 已为 `in_flight → completed | failed | unknown_side_effect` 补齐
  `LeaseReleaseEffect`，并以封闭 `LeaseReleaseReason` 防止持久层解析字符串；SQLite 在单元 3 只消费 typed effect，不复制状态图；
- `reject_unhandled_action_effects` 随之收敛：Lease owner 可消费的 typed effect 不再
  一刀切 fail closed，但仍不开放通用裸迁移入口。

## 5. Approval invalidation 的 Action 侧闭包（单元 5，已实现）

按 ADR-0011 §3，`invalidate_and_optionally_replace` 同事务：

- `approved` Action：不改状态，执行边界以 `approval_invalidated` 重门控；
- `leased` Action：撤销 Lease/锁，`leased → cancelled`（dispatch 未开始由 Kernel 可证）；
- `in_flight` Action：不打断，授权在 dispatch 时已消费；
- 禁止新增 `approved → pending` / `leased → pending` 边，禁止伪装 `lease_expired`。

## 6. 对切片 5 的解锁

child materializer 依赖 `approved → leased → in_flight → completed` 全链与 Stop Fence 读取。
mapping 表使用 migration **0011**（0009 已是 Action-PD heads，0010 分配给 Lease/Stop）。
child creation Audit 的 actor/entry_point 取 `ChildTaskMaterializationCommand` 的 typed
execution context（ADR-0011 §9），禁止继承父 Task actor。

## 7. 验收标准

- `approved → leased → in_flight → completed` 端到端可走通，每条边恰好一条
  `action.state_changed`；completion 原子删除 Lease 与锁。
- Lease 过期、holder 错配、generation 陈旧分别返回对应稳定错误码，且零写入。
- 资源锁冲突失败关闭；释放 Lease 后同一资源可被另一 Action 取得。
- 双源一致性：Action 内联 lease、Lease 行、锁行 generation 任一篡改均
  `stored_data_invalid`。
- Stop Fence：激活后新 `acquire_lease`/`begin_dispatch` 一律 `stop_fence_active`；
  重复激活幂等返回原 generation/Actor/时间且无第二事件；受影响 N 个 Action 恰好
  N 条 Action 事件 + 1 条 Fence 事件；任一失败整批回滚。
- 所有失败路径断言 Lease、锁、Action、Intent、Outbox、aggregate sequence 全量回滚。
