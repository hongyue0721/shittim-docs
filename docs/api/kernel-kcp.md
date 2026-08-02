# kernel-kcp 不可连接库边界（active runtime）

`kernel-kcp` 是不可连接的 Rust 库级 KCP 边界。它消费 production `METHOD_VERSION_BINDINGS` / `select_request_version`：

- method-aware `serde_json::Value` preflight（V2 Envelope 结构门 + method/version 业务门）；
- 全八方法 catalog 可 Accepted（`task.create` 仅 active v2；v1 create → `unsupported_schema_version`）；
- 五方法 registration narrow：`system.ping`、root-only `task.create` v2、`task.get`、`stop.activate`、`stop.status`；
- `TypedDispatcher`；
- `system.ping`、`task.create` v2、`task.get` typed handlers + `stop.activate` / `stop.status` typed handlers；
- `SqliteTaskBackend` → `create_root_task_v2`；`SqliteStopBackend` → `activate_stop_fence` / `get_stop_fence`。

它不接受 bytes 或 transport frame，不提供 UTF-8/JSON parser、frame codec、Socket、Named Pipe、server 或 `agentd`。其余三个 Catalog 方法（`task.list` / `event.subscribe` / `event.poll`）没有 handler，仍阻塞 server 启动。§13.7 谓词随切片 5（child materializer）与切片 6（`stop.activate` / `stop.status`）进一步闭合；`kernel-sqlite` v1 write 路径已在切片3c 删除，本 crate 触达 `create_root_task_v2` / `materialize_child_task` / Lease/Stop owner / v2 Outbox。

## Active 合同状态

| 项 | 状态 |
|---|---|
| production bindings + `select_request_version` | 切片3a 激活；runtime 消费 |
| method-aware preflight | **已实现** |
| `task.create` v2 root-only handler | **已实现**（七 UUID + provenance） |
| v1 create production 入口 | **已删除**（preflight 仅 `unsupported_schema_version`） |
| `task.get` handler | **已实现**（retained v1 query） |
| `stop.activate` / `stop.status` handler | **已实现**（切片6；Stop Fence 单例 + 原子转换 + 幂等 readback） |
| `task.list` / `event.subscribe` / `event.poll` handler | **未实现**（KnownCatalogMethodNotImplemented；阻塞 server 启动） |
| SQLite adapter | `SqliteTaskBackend` 映射 `RootTaskCreateV2Command` / `create_root_task_v2`；`SqliteStopBackend` 映射 Stop operation / `activate_stop_fence` |
| child Task KCP 方法 | 不存在；由 `kernel-sqlite` 的 child materializer（切片5）承载，无 KCP 方法 |
| server / Publisher / poll | 未实现 |

不得生成合同禁止的泛型 `TypedKcpCommandEnvelopeV2` wrapper；`task.create` 使用内部 `TaskCreateCommandRequestV2`（已选择 method/version 的请求）。

## 分步入口

```rust
let preflight = kernel_kcp::preflight_value(value);
// Accepted 后：
let registration = kernel_kcp::narrow_to_registered(request);
// Registered 后：
let dispatcher = kernel_kcp::TypedDispatcher::new(clock, ids, task_backend, stop_backend);
let result = dispatcher.dispatch(request);
```

禁止一站式全 Catalog 执行入口。`TypedCatalogRequest` 与 `RegisteredRequest` 内部 variant 私有，分别只能由 preflight/narrow 构造。公共 `handle_*` 仍保留本地 `InputMethodMismatch`，用于防御绕过正常路径的直接错误调用。

详细 Value 分类、固定 wire error、结构化 contract failure 与 registration 集合见 [`kcp-preflight-dispatcher.md`](kcp-preflight-dispatcher.md)。

## 端口边界

handler/dispatcher 复用：

- `KernelClock`：返回已解析的 `DateTime<Utc>`；公开 `SystemKernelClock` 使用 OS 系统时钟，并以显式 epoch 前借位、整数溢出和 chrono 可表示范围检查把异常映射为 `ClockError`，不走隐式 panic 转换；
- `KernelIdGenerator`：active root v2 按 purpose 分配 **七个** UUID（`Task` / `TaskScope` / `ContentOrigin` / `KernelReceipt` / `CreationProvenance` / `AuditRecord` / `Event`）与两个 opaque ID（`Correlation` / `EventDedup`）；`stop.activate` 另分配 `StopFenceEvent` / `StopActionTransition` / `StopActionEvent` 三个 purpose 与两个 opaque ID。公开 `RandomKernelIdGenerator` 使用可失败的 OS 随机源；UUID v4 只是实现细节，不形成协议版本承诺；
- `TaskApplicationBackend`：只暴露 create/get 与闭集 `BackendError`（active create 无 parent-task 错误；`StoreErrorCode::ParentTaskNotFound` 已从 kernel-sqlite 删除）；
- `StopApplicationBackend`：暴露 `activate_stop_fence` / `get_stop_fence` 与闭集 `BackendError`。激活操作接受事务内 per-Action allocation source（ADR-0011 §5，纯内存分配；任一失败整体回滚并映射 `internal_error`，不泄漏协议细节）。

`TypedDispatcher` 借用这四个端口，并按 variant 只把所需能力传给对应 public handler。它不创建平行接口、不重复 deadline 或 Schema 检查，也不改写 `HandlerResult`。

Response Schema 门是内置生产事实：公共 API 不接收 validator，也不允许调用方替换或绕过。故障注入 seam 仅存在于 crate 私有测试代码。preflight wire error 与 handler error 共用 crate-private final response Schema 门。

`sqlite_adapter::SqliteTaskBackend` 将 `TaskCreateOperation` 转换为 `kernel_sqlite::RootTaskCreateV2Command`，并只通过 `SqliteStore::with_write_transaction` 调用 `create_root_task_v2`。Created 绑定 `committed_event_id` 与 `creation_provenance_ref`；handler 不依赖 SQL、transaction、normalize/hash 或 producer 细节。`sqlite_adapter::SqliteStopBackend` 将 `StopActivateOperation` 转换为 `ActivateStopFenceCommand` 并调用 `activate_stop_fence`；`OriginNotFound` 显式映射为业务错误，其余 store 错误码保持 Internal 闭集（不泄漏细节）。

## Deadline 与提交语义

preflight 不读取 clock、不比较 deadline。registered handler 的第一个可观察操作仍是 clock。deadline 解析为 UTC instant 后以 `now >= deadline` 判断。`task.create` 第一次读时钟同时是唯一 `accepted_at`；backend 返回后才进行第二次完成检查，不在 SQLite 事务内轮询或取消。commit 后超时仍保留已提交事实并返回 `deadline_exceeded`。`stop.activate` 同样在进入 backend 前检查 deadline；post-commit deadline 失败保留已提交的 fence 事实。

只有 `Created` 返回一个 `TaskCreatedCommitted { task_id, event_id }` notification intent。该 intent 在 post-commit deadline、clock failure、response Schema failure 或最终本地 `ContractFailure` 时仍保留；`Replayed` 不产生 intent。stop 方法无 intent 合同，返回空集合。dispatcher 原样透传这些结果。

## Root-only 与 response

`task.create` v2：

- Envelope `task_id` / `expected_revision` 必须为 null（handler 额外 fail-closed 为 `invalid_request`）；
- payload 无 `parent_task_id`（Schema unknown-field 拒绝）；
- 成功 payload 为 `TaskCreateResponseV2 { schema_version:2, task, creation_provenance_ref }`。

`stop.activate`：

- 成功 payload 为 `StopActivateResponse { schema_version:1, active, generation, activated_at, activated_by, reason }`；
- 重复激活幂等：同 generation、零新增写入（canonical readback）。

方法成功 payload 先用对应 response Schema 校验，再装入 generated `KcpResponseEnvelope` 并通过通用 response Schema 校验。成功 payload 失败折叠成固定 `internal_error`；最终 error envelope 也失败时返回本地 `HandlerContractFailure`，不发送未验证响应。

固定 handler error 的 message/details/retryable 以 IC §5.7 / §5.10 映射闭集为准；固定 preflight wire error 来自 §5.11.4。实现不拼接 storage/schema/serde message、SQL、路径或 payload。
