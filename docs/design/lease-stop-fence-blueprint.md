# Lease / Stop Fence 持久化实施蓝图（阶段 B）

> 状态：设计已定，未实施。本文件是实施依据，落地后应随实现同步修订或退役。
> 前置：阶段 A（切片 4c 剩余 5 项不依赖 Lease 的 High）已闭合。

## 1. 为什么这是硬前置

`approved → leased → in_flight → completed` 是副作用 Action 的唯一合法执行链。当前
`domain-task` 已完整定义该链的边与 evidence（含 `LeaseReleaseEffect`），但 `kernel-sqlite`
没有任何 Lease/Resource Lock/Stop Fence 持久化，因此 `reject_unhandled_action_effects`
对任何 `release_lease_and_locks` 效果一律失败关闭。结果：

- Action 无法离开 `approved`，切片 5 child materializer 无法完成 Action；
- 切片 4c 中「resolve/invalidate 必须撤销受影响 Lease」无法真正关闭；
- §13.7 谓词 4 无法为真。

## 2. 已有契约资产（不需新建）

| 资产 | 位置 | 说明 |
|---|---|---|
| `ActionRequestV2.lease` | 生成类型 `ActionRequestV2Lease` | **Lease 是 Action 的内联字段**，非独立 Schema：`{holder, generation, expires_at, max_uses}` |
| `execution_generation` | `ActionRequestV2` 顶层字段 | 每次 Lease/执行尝试单调增加；不扩大 Action 业务唯一性 |
| `LeaseReleaseEffect` | `domain-task/src/action.rs` | 纯领域效果：`invalidate_lease` + `release_all_resource_locks` |
| `StopFenceActivatedPayload` | `schemas/source/event/stop_fence_activated_payload.v1.json` | 已存在；`{generation, reason, activated_by_actor_id, activated_from_entry_point, activated_at}` |
| `EventAggregateId::StopFenceGlobal` | `kernel-sqlite/src/outbox.rs` | 单例全局聚合已就位 |
| 错误码 | 错误目录 | `lease_not_found` / `lease_expired` / `lease_holder_mismatch` / `action_not_executable` / `stop_fence_active` 已定义 |

**结论：阶段 B 不新增业务 Schema。** 只需 migration 0010 + 仓储编排。Resource Lock 尚无
Schema，但它是 Kernel 内部并发治理事实而非跨进程契约对象，按内部表实现即可（见 §3.2）。

## 3. migration 0010 表设计

沿用 migration 0009 的 descriptor v1 风格：schema 阶段建表、Rust 关系校验/回填、guards 阶段建触发器。

### 3.1 `action_leases`（每 Action 至多一个活跃 Lease）

```sql
CREATE TABLE action_leases (
    action_id TEXT PRIMARY KEY,
    holder TEXT NOT NULL,
    generation INTEGER NOT NULL CHECK(generation >= 1),
    expires_at TEXT NOT NULL,
    max_uses INTEGER NOT NULL CHECK(max_uses >= 1),
    acquired_at_action_revision INTEGER NOT NULL CHECK(acquired_at_action_revision >= 1),
    FOREIGN KEY(action_id) REFERENCES actions(id)
);
```

不变量与触发器：

- **generation 与 Action 一致**：`generation` 必须等于 `actions.execution_generation`；
  插入触发器校验，防止用陈旧 generation 取得 Lease。
- **只在 leased 状态存在**：`actions.status` 离开 `leased` 时必须无残留 Lease 行；
  用 `actions` UPDATE 触发器强制"status 不是 leased 则该 action 无 lease 行"。
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
    FOREIGN KEY(action_id) REFERENCES action_leases(action_id) ON DELETE CASCADE
);
```

- `resource_ref` 为主键即表达**逻辑资源全局互斥**；冲突插入 → `constraint_violation`
  映射为 `action_not_executable`（附冲突 resource）。
- `ON DELETE CASCADE` 保证 §2 要求的"释放 Lease 的同一原子变更中释放其全部 Resource Lock"，
  不依赖调用方记得删。
- 资源 URI 必须复用 `domain_policy::normalize_uri`，与 TaskScope/PD 指纹同一规范化，
  禁止平行语义。

### 3.3 `stop_fence`（单例）

```sql
CREATE TABLE stop_fence (
    id INTEGER PRIMARY KEY CHECK(id = 1),
    generation INTEGER NOT NULL CHECK(generation >= 1),
    reason TEXT NOT NULL,
    activated_by_actor_id TEXT NOT NULL,
    activated_from_entry_point TEXT NOT NULL,
    activated_at TEXT NOT NULL
);
```

- `CHECK(id = 1)` 表达单例；**行不存在 = 未激活**，不用布尔列表达"已激活但 generation 无意义"。
- generation 单调递增；重复 `stop.activate` 返回当前同一 generation 且**不产生第二个事件**
  （IC §680 幂等 scope）。
- 首批不提供清除方法（IC §704），不得预留私有开关。

## 4. 仓储 API 设计

全部为 owner 编排器，`cas_transition_for_intent` 仍是唯一私有机械 CAS，
`action.state_changed` 仍是唯一状态事件权威。

| 入口 | 职责 | 事务闭包 |
|---|---|---|
| `acquire_lease` | `approved → leased` | 读 Stop Fence（激活则 `stop_fence_active`）→ 递增 `execution_generation` → 插入 Lease → 插入全部 Resource Lock（冲突失败关闭）→ 状态边 CAS + 事件 |
| `release_or_expire_lease` | `leased → approved`（过期）/ `leased → cancelled`（未派发） | CAS `(action_id, holder, generation)` → 删除 Lease（级联删锁）→ 状态边 CAS + 事件 |
| `begin_dispatch` | `leased → in_flight` | 复核 Lease 有效（未过期、holder 匹配、generation 匹配）+ Stop Fence 未激活 → 状态边 CAS + 事件 |
| `activate_stop_fence` | 激活围栏 | 插入/递增单例 → 撤销所有活跃副作用 Action 的 Lease 与锁 → 每个受影响 Action 一条状态边事件 + 一条 `stop_fence.activated` |
| `get_stop_fence` / `get_action_lease` | 严格只读 | 破损数据 → `stored_data_invalid` |

**关键不变量**：Lease 消费必须 CAS `holder + generation + expires_at`（IC §1594）；
Stop Fence generation 必须在执行事务内**再次读取**，不能用事务外快照。

## 5. 对切片 4c 与切片 5 的解锁

- 4c 最后 2 项：`resolve` / `invalidate_and_optionally_replace` 可在同一事务内撤销受影响
  Lease 与锁，完成 IC §1440 要求的"撤销受影响 Permission/Privilege/Action Lease"。
- 切片 5：child materializer 依赖 `leased → in_flight → completed`，其中 completed 必须与
  child facts 同事务（IC §6.5.1）。`complete_child_materialization` 在阶段 B 之后才有合法前驱状态。

## 6. 验收标准

- `approved → leased → in_flight → completed` 端到端可走通，每条边恰好一条 `action.state_changed`。
- Lease 过期、holder 错配、generation 陈旧分别返回对应稳定错误码，且零写入。
- 资源锁冲突失败关闭；释放 Lease 后同一资源可被另一 Action 取得。
- Stop Fence 激活后新 `acquire_lease` 与 `begin_dispatch` 一律 `stop_fence_active`；
  重复激活幂等且不产生第二个事件。
- 所有失败路径断言 Lease、锁、Action、Intent、Outbox、aggregate sequence 全量回滚。
