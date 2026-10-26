# ChannelManager 详解

文档版本：QwenPaw v2.2.1  
对照源码：`src/qwenpaw/`（包版本 `__version__` = `2.2.2b1`）  
日期：2026-09-15

本文对照当前源码说明 `ChannelManager` 与 **UnifiedQueueManager** 的职责划分。队列、worker、session 合并已从「每频道一条队列 + 固定 4 worker」迁到按 `(channel_id, session_id, priority_level)` 按需建队。

相关源码：

| 文件 | 职责 |
|---|---|
| `src/qwenpaw/app/channels/manager.py` | 频道生命周期、入队路由、批次合并、热替换 |
| `src/qwenpaw/app/channels/unified_queue_manager.py` | 按 QueueKey 建队、按需 consumer、空闲回收 |
| `src/qwenpaw/app/channels/command_registry.py` | 按查询文本判定优先级 |
| `src/qwenpaw/app/channels/registry.py` | 内置 + 插件频道类注册表 |
| `src/qwenpaw/app/channels/base.py` | `BaseChannel`：enqueue 回调、consume、debounce |
| `src/qwenpaw/app/workspace/service_factories.py` | `ChannelManager.from_config` 的生产入口 |
| `src/qwenpaw/app/workspace/workspace.py` | `start_all` / `stop_all` 挂到 ServiceManager |

---

## 1. 总览：状态在哪

`ChannelManager` 仍是频道的所有者：创建、启动、停止、替换频道实例，并把平台回调接到统一队列。  
**队列、consumer 任务、空闲清理不再挂在 ChannelManager 上**，全部在 `UnifiedQueueManager`。

```
状态仍在 ChannelManager
  channels, _lock, _loop
  _command_registry
  _queue_manager          # 指向 UnifiedQueueManager，不自管队列
  _workspace
  _restart_locks
  _enqueue_tasks, _start_tasks

状态在 UnifiedQueueManager
  _queues: QueueKey → QueueState
  _lock, _cleanup_task, _running
  每条队列的 asyncio.Queue + consumer Task
  processed_count / last_activity

状态在 BaseChannel（debounce，不在 Manager）
  _no_text_debounce, _pending_content_by_session
  _debounce_seconds, _debounce_pending, _debounce_timers
  _enqueue 回调
```

### 已移除、不要对着旧文档找代码

| 旧字段 / 常量 | 现状 |
|---|---|
| `self._queues: Dict[str, asyncio.Queue]` | **已移除**。迁至 `UnifiedQueueManager._queues`，key 不再是 `channel_id`，而是 `QueueKey` |
| `self._consumer_tasks` | **已移除**。consumer 存在 `QueueState.consumer_task` |
| `self._in_progress` | **已移除**。同一 QueueKey 只有一个 consumer，无需「处理中」集合 |
| `self._pending` | **已移除**。处理中到达的消息直接进入该 QueueKey 的 `asyncio.Queue` |
| `self._key_locks` | **已移除**。串行由「一 key 一 consumer」保证，不再按 session 加锁 drain |
| `_CONSUMER_WORKERS_PER_CHANNEL = 4` | **已移除**。没有固定 worker 池；按需为每个 QueueKey 创建 **1** 个 consumer |
| `ChannelManager._enqueue_one` 的 in_progress/pending 分支 | **已移除**。现职责是分类优先级并 `await _queue_manager.enqueue(...)` |
| `ChannelManager._consume_channel_loop` | **已移除**。消费循环是 `ChannelManager._consume_queue`，由 UnifiedQueueManager 启动 |

`_CHANNEL_QUEUE_MAXSIZE = 1000` **仍在** `manager.py`，但含义变了：是 **每个 QueueKey** 的 `asyncio.Queue(maxsize=1000)`，不是每个频道一条共享队列。

---

## 2. 端到端数据流

```
平台 SDK 回调（可能在非 asyncio 线程）
        │
        │  ch._enqueue(payload)   # set_enqueue 注入的闭包
        ▼
ChannelManager.enqueue(channel_id, payload)     # 线程安全
        │  loop.call_soon_threadsafe(_enqueue_one)
        ▼
ChannelManager._enqueue_one                     # 必须在事件循环线程
        │  query  = ch._extract_query_from_payload(payload)
        │  prio   = CommandRegistry.get_priority_level(query)
        │  sid    = _extract_session_id(ch, payload)
        │  create_task(_enqueue_with_timeout(...))   # 超时 30s
        ▼
UnifiedQueueManager.enqueue(channel_id, session_id, priority, payload)
        │  QueueKey = (channel_id, session_id, priority_level)
        │  按需 _get_or_create_queue：Queue(maxsize=1000) + 1 consumer
        │  await queue.put(payload)                  # 超时 30s
        ▼
ChannelManager._consume_queue(queue, channel_id, session_id, priority)
        │  payload = await queue.get()
        │  drain：get_nowait 抽空当前队列 → batch
        │  _process_batch(ch, batch)  → merge → consume
        ▼
BaseChannel._consume_one_request 或 consume_one
        │  no-text debounce / ACL / TaskTracker / process
        ▼
channel.send_event / send
```

同一会话、同一优先级：一条队列、一个 consumer，严格串行。  
同一会话、不同优先级（例如 `/stop` = 0，普通消息 = 20）：**两条队列、两个 consumer，可并行**。  
不同会话：彼此独立。

---

## 3. ChannelManager 成员变量

```python
class ChannelManager:
    def __init__(self, channels: List[BaseChannel]):
        self.channels = channels
        self._lock = asyncio.Lock()
        self._loop: Optional[asyncio.AbstractEventLoop] = None

        # New unified queue system
        self._command_registry = CommandRegistry()
        self._queue_manager: UnifiedQueueManager | None = None
        self._workspace = None

        self._restart_locks: dict[str, asyncio.Lock] = {}
        self._enqueue_tasks: set[asyncio.Task] = set()
        self._start_tasks: set[asyncio.Task] = set()
```

### 3.1 仍由 Manager 持有

**`channels: List[BaseChannel]`**  
当前进程里真正跑着的频道实例。`from_config` 只放入「注册表有类 + 环境允许 + `enabled=True`」的频道。`replace_channel` 会按 `ch.channel` 原地替换或 append。

**`_lock: asyncio.Lock`**  
保护 `self.channels` 的读快照 / 写替换。`get_channel`、`start_all` 快照、`stop_all` 快照、`replace_channel` 的 swap+stop 都用它。锁 **不** 保护 UnifiedQueueManager 内部字典。

**`_loop: Optional[asyncio.AbstractEventLoop]`**  
`start_all()` 里 `asyncio.get_running_loop()`。`enqueue()` 依赖它做 `call_soon_threadsafe`。未 `start_all` 时入队会 warning 并丢弃。

**`_command_registry: CommandRegistry`**  
Manager 自己持有一份，用于入队时算 `priority_level`；`set_workspace` 时同样注入每个频道，供 `_consume_one_request` 识别控制命令（绕过 TaskTracker）。

**`_queue_manager: UnifiedQueueManager | None`**  
`start_all` 创建，`stop_all` 里 `stop_all()` 后置 `None`。未初始化时 `_enqueue_one` 直接 return。

**`_workspace`**  
`set_workspace(ws)` 写入。`restart_channel` 需要它拿 `agent_id` 读最新 `agent.json`；频道侧用它访问 TaskTracker / ChatManager。

**`_restart_locks: dict[str, asyncio.Lock]`**  
按频道名加锁，避免并发 `restart_channel` 造出两份 clone、丢掉其中一份。

**`_enqueue_tasks` / `_start_tasks`**  
跟踪 `asyncio.create_task` 出来的入队协程和频道 `start()` 任务，供 `stop_all` 取消。`add_done_callback(...discard)` 做集合清理。

### 3.2 工厂与注册

生产路径不是 `_app.py` 直接 `ChannelManager.from_config`，而是：

```
_app.py lifespan
  → workspace_registry.start_all_configured_agents()
    → Workspace.start()
      → ServiceManager.start_all()
        → create_channel_service()          # service_factories.py
            ChannelManager.from_config(
                process=ws.stream_query,
                config=temp_config,         # 该 agent 的 channels + 全局 show_tool_details
                on_last_dispatch=...,
                workspace_dir=ws.workspace_dir,
            )
            cm.set_workspace(ws)
        → ChannelManager.start_all()        # ServiceDescriptor.start_method
```

`from_config` 要点（以源码为准）：

1. `get_available_channels()`：注册表全量，再被 `QWENPAW_ENABLED_CHANNELS` / `QWENPAW_DISABLED_CHANNELS` 过滤。
2. `get_channel_registry()`：内置 `_BUILTIN_SPECS` + 插件 `PluginRegistry.get_registered_channels()`；插件 key 与内置冲突则跳过插件。
3. 配置里该 key 缺失则跳过；`enabled` 为假则跳过。
4. `inspect.signature(ch_cls.from_config)` 过滤 kwargs（`process` / `config` / `on_reply_sent` / `display_config` / `no_text_debounce` / `workspace_dir`）。构造失败只 warning，不影响其它频道。
5. 内置 key 见 `registry.py` 的 `_BUILTIN_SPECS`：`imessage`, `discord`, `dingtalk`, `feishu`, `qq`, `telegram`, `mattermost`, `mqtt`, `console`, `matrix`, `slack`, `voice`, `sip`, `wecom`, `xiaoyi`, `yuanbao`, `wechat`, `onebot`。其中 `console` 是 required，加载失败会抛错。

`from_env` 仍存在（按环境变量构造），当前仓库内 **没有调用点**；运行时走 `from_config`。

`uses_manager_queue`（`BaseChannel` 默认 `True`）：为真才 `set_enqueue`。`voice`、`sip` 为 `False`（长连接自己跑循环），仍会 `start()` / `stop()`，但不进统一队列。

---

## 4. UnifiedQueueManager 成员变量

```python
QueueKey = Tuple[str, str, int]  # (channel_id, session_id, priority_level)

@dataclass
class QueueState:
    queue: asyncio.Queue
    consumer_task: asyncio.Task[None]
    created_at: float
    last_activity: float
    processed_count: int = 0

class UnifiedQueueManager:
    def __init__(
        self,
        consumer_fn: ConsumerFn,
        queue_maxsize: int = 1000,
        idle_timeout: float = 600.0,   # 10 minutes
        cleanup_interval: float = 60.0,  # 1 minute
    ):
        self._consumer_fn = consumer_fn
        self._queue_maxsize = queue_maxsize
        self._idle_timeout = idle_timeout
        self._cleanup_interval = cleanup_interval
        self._queues: Dict[QueueKey, QueueState] = {}
        self._lock = asyncio.Lock()
        self._cleanup_task: asyncio.Task[None] | None = None
        self._running = False
```

`ChannelManager.start_all` 的构造调用是：

```python
self._queue_manager = UnifiedQueueManager(
    consumer_fn=self._consume_queue,
    queue_maxsize=_CHANNEL_QUEUE_MAXSIZE,  # 1000
)
```

未改 `idle_timeout` / `cleanup_interval`，因此默认 **空闲 600s 且队列为空则回收**，清理循环 **每 60s** 跑一次。

| 成员 | 含义 |
|---|---|
| `_consumer_fn` | 即 `ChannelManager._consume_queue` |
| `_queue_maxsize` | 每条 QueueKey 队列容量，满则 `put` 阻塞，最多等 30s |
| `_queues` | 活着的队列；consumer 退出或 idle cleanup 会 `pop` |
| `_lock` | 保护 `_queues` 字典本身，不是 payload 处理锁 |
| `_cleanup_task` | 后台 `_cleanup_idle_queues` |
| `_running` | `start_cleanup_loop` 置 True，`stop_all` 置 False |

`QueueState.last_activity` 在 enqueue 成功路径、以及 `increment_processed` 后更新。idle 判定：`queue.empty()` **且** `now - last_activity > idle_timeout`。

---

## 5. 优先级：CommandRegistry

入队前 Manager 抽 query，再问 registry：

```python
query = ch._extract_query_from_payload(payload)
priority_level = self._command_registry.get_priority_level(query)
```

默认档位（数字越小越优先）：

| level | 名称 | 默认注册 |
|---|---|---|
| 0 | critical | `/stop` |
| 10 | high | `/daemon ...` 及短别名 `/status` `/restart` `/logs` `/approve` `/deny` `/approval` 等 |
| 20 | normal | **缺省**（普通聊天、`/compact` `/new` 等未注册命令） |
| 30 | low | 预留，默认未挂命令 |

匹配规则：query 必须以 `/` 开头；按 **最长前缀** 匹配，其后必须是结尾或空白，因此 `/stopx`、`/statusx` 落到 20。

`/stop` 处理里会 `channel_manager.clear_queue(channel_id, session_id, 20)`，只清 **normal(20)** 队列，不清 priority 0 自己那条。

---

## 6. enqueue：线程安全路径

### 6.1 频道侧回调

`start_all` / `replace_channel` 对 `uses_manager_queue=True` 的频道：

```python
def _make_enqueue_cb(self, channel_id: str) -> Callable[[Any], None]:
    def cb(payload: Any) -> None:
        self.enqueue(channel_id, payload)
    return cb

ch.set_enqueue(self._make_enqueue_cb(ch.channel))
```

平台线程（钉钉 Stream 等）调用 `self._enqueue(payload)` 或自己再包一层 `call_soon_threadsafe`。钉钉 handler 已对回调做了一次 `call_soon_threadsafe`；`ChannelManager.enqueue` 还会再做一次，最终 `_enqueue_one` 仍在主 loop 上执行。

### 6.2 `enqueue`（可从任意线程调用）

```python
def enqueue(self, channel_id: str, payload: Any) -> None:
    if self._loop is None:
        logger.warning("enqueue: loop not set ...")
        return
    self._loop.call_soon_threadsafe(
        self._enqueue_one,
        channel_id,
        payload,
    )
```

这是线程安全边界：同步 SDK 不得直接 `await` 队列。必须先 `start_all()`，否则 `_loop` 为空。

### 6.3 `_enqueue_one`（事件循环线程，同步函数）

不再检查 `_in_progress` / `_pending`。步骤：

1. `_queue_manager is None` → warning，丢弃。
2. 按 `c.channel == channel_id` 找频道；找不到 → warning，丢弃（旧实现这里会 `put_nowait` 到频道队列；新实现没有按频道的共享队列，无法投递）。
3. `query` / `priority_level` / `session_id`。
4. `asyncio.create_task(_enqueue_with_timeout(...))` 记入 `_enqueue_tasks`。

`session_id` 提取：payload 已有 `session_id` 则用之，否则 `ch.get_debounce_key(payload)`（与频道 session 隔离同一套逻辑）。

### 6.4 `_enqueue_with_timeout` + `UnifiedQueueManager.enqueue`

Manager 外包 `asyncio.wait_for(..., timeout=30.0)`。  
`TimeoutError` 在 Manager 里是 **`pass`（无日志）**；`CancelledError` 再抛出；其它异常 `logger.exception`。

UnifiedQueueManager 内部：

1. `_get_or_create_queue`：没有则 `asyncio.Queue(maxsize=1000)` + `create_task(_run_consumer)`，名字形如 `consumer_{channel}_{session[:20]}_{priority}`。
2. 更新 `last_activity`。
3. `await asyncio.wait_for(state.queue.put(payload), timeout=30.0)`；超时打 warning（含 `qsize`）再 **raise**，由外层 wait_for 吃掉。

两层 30s：外层包整个 enqueue（含建队），内层只包 `put`。队列满时 `put` 阻塞直到有空位或超时。

---

## 7. worker / drain / session 合并

### 7.1 谁在跑循环

`UnifiedQueueManager._run_consumer` 只做：调用 `consumer_fn` → 退出时从 `_queues` pop 该 key。真正的 `while True` 在 `ChannelManager._consume_queue`。

**没有「每频道 4 个 worker」。** 并发度 = 当前活着的 QueueKey 数量。同一 key 永远只有一个 consumer。

### 7.2 `_consume_queue` drain

```python
payload = await queue.get()
ch = await self.get_channel(channel_id)   # 每次重取，配合 replace_channel
# 找不到则 0.5s × 3 次重试；仍无则 logger.error 并 return（该 consumer 结束）

batch = [payload]
while True:
    try:
        batch.append(queue.get_nowait())
    except asyncio.QueueEmpty:
        break

await _process_batch(ch, batch)
await self._queue_manager.increment_processed(..., count=len(batch))
```

drain 不再按 debounce key 过滤：这条队列里的消息 **已经** 是同一 `(channel, session, priority)`。drain 只合并「已经排队、尚未 get」的连发（如图片紧跟文本）。处理期间新入队的消息留在队列里，本轮 `_process_batch` 结束后下一轮再取——这替代了旧的 `_pending` 回灌。

`CancelledError`：break 退出。其它异常：打日志，**继续循环**（单条失败不杀 consumer）。

### 7.3 `_process_batch`

```python
async def _process_batch(ch: BaseChannel, batch: List[Any]) -> None:
    if len(batch) > 1 and ch._is_native_payload(batch[0]):
        merged = ch.merge_native_items(batch)
        await ch._consume_one_request(merged)
    elif len(batch) > 1:
        merged = ch.merge_requests(batch)
        if merged is not None:
            await ch._consume_one_request(merged)
        else:
            await ch.consume_one(batch[0])
    elif ch._is_native_payload(batch[0]):
        await ch._consume_one_request(batch[0])
    else:
        await ch.consume_one(batch[0])
```

`_is_native_payload`：`dict` 且含 `content_parts`。  
native 合并走 `merge_native_items`（默认拼 `content_parts`、合并部分 meta）；非 native 走 `merge_requests`（拼 `input[0].content`）。钉钉有额外 merge 日志。

注意：native 路径（含单条）走 `_consume_one_request`，**绕过** `consume_one`。因此 `BaseChannel.consume_one` 里 `_debounce_seconds > 0` 的时间窗合并，对 Manager 投递的 native payload **不会触发**。时间窗 debounce 只在有人直接调用 `consume_one` 且 payload 为 native 时生效。当前 `dingtalk` 把 `_debounce_seconds` 设为 `0.0`。

无文本防抖在 `_consume_one_request` → `_debounce_payload` → `_apply_no_text_debounce`，责任在 **频道**，不在 Manager。

---

## 8. start / stop

### 8.1 `start_all`

```
1. self._loop = get_running_loop()
2. 构造 UnifiedQueueManager(consumer_fn=_consume_queue, maxsize=1000)
3. start_cleanup_loop()          # 后台 idle 回收
4. 锁内 snapshot = list(channels)
5. uses_manager_queue 的频道 set_enqueue(...)
6. 每个频道 create_task(_start_channel(g))  # fire-and-forget，不 await start()
```

与旧实现差异：

- **不再** 按频道创建 `asyncio.Queue`，也 **不再** 启动 `channel_consumer_{ch}_{0..3}`。
- 频道 `start()` 改为后台任务（避免 WebSocket 握手堵住整个 Workspace 启动）。失败只记 exception 日志。
- 第一条消息到来时才建 Queue + consumer。

### 8.2 `stop_all`

顺序（源码原样）：

1. 取消 `_start_tasks`，`wait` 最多 3s。
2. 取消 `_enqueue_tasks`，`wait` 最多 2s；超时 warning。
3. `await _queue_manager.stop_all()`：停 cleanup；cancel 全部 consumer，wait 最多 5s；`_queues.clear()`；然后 Manager 把 `_queue_manager = None`。
4. 频道 `set_enqueue(None)`。
5. **逆序** `gather` 各频道 `stop()`。

---

## 9. 热重载与单频道重启

ChannelManager **没有** 旧式「改完 config 在 Manager 内重建 `_queues`」的逻辑。现有两条路径：

### 9.1 单频道重启（Manager 原地替换）

HTTP `POST .../channels/{channel_name}/restart` → `restart_channel`：

1. 取 `_restart_locks[channel_name]`。
2. 必须已 `set_workspace`（否则 `RuntimeError`）。
3. `load_agent_config(agent_id)` 读该频道最新配置。
4. `channel_instance.clone(config)` → `from_config` 新实例，再 `set_workspace`。
5. `replace_channel(new_channel)`：
   - 先 `set_enqueue` + `await new.start()`（在 `_lock` 外，启动失败会 `stop` 新实例再 raise）。
   - 锁内按名字 swap 或 append，再 `stop` 旧实例。

`replace_channel` **不** 重建 UnifiedQueue。注释写明 queue/consumer 仍按需创建。`_consume_queue` 每轮 `get_channel`，swap 后下一轮用新实例。swap 窗口频道暂时找不到时，最多重试 1.5s，失败则该 QueueKey 的 consumer 退出。

### 9.2 Agent 级热重载（重建整个 Workspace，含新的 ChannelManager）

- 前端/API 改配置后 `schedule_agent_reload` → `MultiAgentManager.reload_agent`。
- `AgentConfigWatcher` 轮询该 agent 的 `agent.json`（默认 2s）。`channels` 或 `heartbeat` hash 变化时同样走 `reload_agent`。

这是 **换 Workspace**，不是在现有 `ChannelManager` 上改 `_queues`。旧 Manager 走 `stop_all`，新 Manager 再 `from_config` + `start_all`。

---

## 10. 与 BaseChannel 的协作

| 方向 | 约定 |
|---|---|
| Manager → Channel | `set_enqueue(cb)` / `None`；`start()` / `stop()`；`clone`；`health_check`；`send_event` / `send_content_parts` |
| Channel → Manager | 收到平台消息后调用 `_enqueue(payload)`（线程安全由 Manager.enqueue 保证） |
| Manager 消费 | native：`_consume_one_request`；非 native 单条：`consume_one` |
| Channel 必须提供 | `channel` 名、`from_config`、`get_debounce_key` / `resolve_session_id`、`_extract_query_from_payload`、`merge_native_items` / `merge_requests` |
| debounce 边界 | **session 隔离 + 队列 drain 合并**：Manager/UnifiedQueue；**无文本缓冲、时间窗**：Channel |

`get_debounce_key` 默认：dict 用 `session_id` 或 `resolve_session_id(sender_id, meta)`（默认 `"{channel}:{sender_id}"`）。钉钉等可 override。

`send_event` / `send_text`：Manager 按频道名查找，补 `session_id` / `user_id` / `bot_prefix` 后交给频道。定时任务纯文本走 `send_text`。

`set_workspace(workspace)`：Manager 保存引用，并对每个频道 `ch.set_workspace(workspace, self._command_registry)`。

---

## 11. 处理过程示例（对照旧 pending 模型）

钉钉同一 `session_id` 连续三条普通消息（priority=20），另有一条 `/stop`（priority=0）：

```
msg1 "hello"     → 创建 QueueKey=(dingtalk, sid, 20)，consumer 开始处理
msg2 "world"     → 若 msg1 仍在 _process_batch：msg2 进同一队列，本轮 drain
                   若已开始处理且队列已空：msg2 等下一轮，不再进 Manager._pending
msg3 /stop       → QueueKey=(dingtalk, sid, 0) 另一条队列，可与 20 并行
                   （/stop 随后 clear_queue(..., 20) 丢掉未处理的普通消息）
```

旧模型：msg1 处理中时 msg2 进 `_pending`，finally 再 merge 放回频道队列。  
新模型：msg2 始终进同一 `asyncio.Queue`；drain 能合并的当场合并，合并窗口之外的下轮处理。

---

## 12. 其它 API（Manager 薄封装）

| 方法 | 行为 |
|---|---|
| `get_channel(name)` | 锁内线性查找 |
| `get_channel_health(name)` | 调 `health_check()`，异常则返回 `unhealthy` dict；频道不存在 `KeyError` |
| `clear_queue(ch, sid, prio)` | 转 `_queue_manager.clear_queue`，`get_nowait` 抽空；manager 未启动返回 0 |
| `get_metrics` | **只在 UnifiedQueueManager**；ChannelManager **没有** 转发 |

---

## 13. 相对旧实现的关键变化

1. **队列维度**：从「每频道 1 条 Queue、固定 4 worker」变为「每 `(channel, session, priority)` 1 条 Queue、1 个按需 consumer」。
2. **maxsize=1000**：仍是 `asyncio.Queue` 上限，但是 **每个 QueueKey**，不是每个频道共享 1000。
3. **session 串行**：不再靠 `_key_locks` + `_in_progress`；同一 QueueKey 天然单消费者。
4. **`_pending` 回灌取消**：处理中到达的消息直接 `queue.put`；drain 只合并当前积压。
5. **优先级队列**：`/stop` 等控制命令可与普通会话并行，并可 `clear_queue` 丢掉普通积压。
6. **debounce 下沉**：无文本 / 时间窗在 `BaseChannel`；Manager 只做优先级路由 + 同 QueueKey 批次 merge。
7. **启动**：不再预创建频道队列和 4N 个 consumer；`start()` 后台化；第一条消息才建队。
8. **空闲回收**：空队列且 600s 无活动则 cancel consumer（每 60s 扫描）。
9. **热重载**：单频道 `replace_channel` 不拆队列；配置级变更走 Workspace/`reload_agent` 整实例替换。
10. **入队失败语义**：频道找不到或 queue_manager 未就绪则丢消息并打日志，不再写入「频道级」队列。
)
