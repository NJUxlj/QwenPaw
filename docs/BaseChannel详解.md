# BaseChannel 详解

> QwenPaw v2.2.1 · 2026-09-15  
> 源码：`src/qwenpaw/app/channels/base.py`（约 2369 行）  
> 协作：`schema.py`、`renderer.py`、`access_control.py`、`manager.py`  
> 队列实现细节见 `docs/ChannelManager详解.md`，本文只写 BaseChannel 侧的契约与边界。

`BaseChannel` 是所有渠道的抽象基类。它定义「如何把原生消息变成 `AgentRequest`、如何消费、如何把 Event 发回用户」。  
**队列、优先级、按 session 串行的消费者循环不再属于 BaseChannel**，由 `ChannelManager` + `UnifiedQueueManager` 拥有。Channel 只通过 `set_enqueue()` 拿到入队回调，通过 `consume_one()` / `_consume_one_request()` 定义如何消费。

核对子类：飞书 `FeishuChannel`（`src/qwenpaw/app/channels/feishu/channel.py`）实现了 `from_env` / `from_config` / `build_agent_request_from_native` / `start` / `stop` / `send`，并覆盖 `resolve_session_id`、`merge_native_items`、`get_to_handle_from_request`、`_before_consume_process` 等 Hook。钉钉把 `_debounce_seconds` 固定为 `0.0`，依赖 Manager 的 drain+merge。

---

## 核心属性

### 类属性

| 属性 | 类型 / 默认值 | 说明 |
|------|----------------|------|
| `channel` | `ChannelType`（`str`） | 渠道标识，必须由子类赋值。内置集合见 `schema.BUILTIN_CHANNEL_TYPES` |
| `uses_manager_queue` | `bool = True` | `True` 时 Manager 会 `set_enqueue()` 并走统一队列。`voice` / `sip` 设为 `False`（长连接自管循环） |
| `streaming_enabled` | `bool = False` | 为 `True` 时，在 completed-message 路径之外额外派发 `on_streaming_start` / `on_streaming_delta` / `on_streaming_end` |
| `_STREAMABLE_TYPES` | `{"reasoning", "message"}` | 可走流式 Hook 的 event.type |
| `_STREAM_DELTA_MIN_INTERVAL_S` | `float = 0.0` | 流式 delta 最小刷新间隔（秒）。`0` 表示不限速。飞书等子类会覆盖 |
| `_STREAM_FLUSH_TIMEOUT_S` | `float = 5.0` | 等待上一次 flush task 的超时 |

### 实例属性（`__init__`）

| 属性 | 类型 | 说明 |
|------|------|------|
| `_process` | `ProcessHandler` | `AgentRequest → AsyncIterator[Event]`，通常是 `workspace.stream_query` |
| `_on_reply_sent` | `OnReplySent` | 用户回复发送成功后的回调：`(channel, user_id, session_id)` |
| `_display_config` | `ChannelDisplayConfig` | 工具/思考展示开关（见下表） |
| `_no_text_debounce` | `bool` | 无文本消息是否缓存等待文本。默认 `True` |
| `_render_style` | `RenderStyle` | 渲染风格；`internal_tools` 来自 `config.tools.builtin_tools` 中 `display_to_user=False` 的工具名 |
| `_renderer` | `MessageRenderer` | `Message → List[OutgoingContentPart]` |
| `_enqueue` | `EnqueueCallback` | 入队回调，由 `ChannelManager.set_enqueue()` 注入；未启动时为 `None` |
| `_workspace` | Workspace / `None` | `set_workspace()` 注入；提供 `chat_manager`、`task_tracker` |
| `_command_registry` | `CommandRegistry` | 与 workspace 一起注入；用于识别 `/stop` 等控制命令 |
| `_pending_content_by_session` | `Dict[str, List[Any]]` | 无文本内容缓存，key = session_id（`get_debounce_key`） |
| `_debounce_seconds` | `float = 0.0` | 时间防抖窗口。`> 0` 时 `consume_one()` 才缓冲。钉钉显式设为 `0.0` |
| `_debounce_pending` | `Dict[str, List[Any]]` | 时间防抖缓冲区 |
| `_debounce_timers` | `Dict[str, asyncio.Task[None]]` | 防抖 flush 定时任务 |
| `_language` | `str = "zh"` | ACL 文案语言。Workspace 启动时由 `service_factories` 写成 agent 语言 |
| `_http` | `Optional[Any] = None` | 子类可用的 HTTP 会话占位 |

类型别名（模块级）：

- `EnqueueCallback = Optional[Callable[[Any], None]]`
- `OnReplySent = Optional[Callable[[str, str, str], Awaitable[None]]]`
- `ProcessHandler = Callable[[Any], AsyncIterator[Event]]`
- `OutgoingContentPart = Union[TextContent, ImageContent, VideoContent, AudioContent, FileContent, RefusalContent]`

---

## 策略配置默认值

全部抄自 `BaseChannel.__init__` 与 `ChannelDisplayConfig` / `config.BaseChannelConfig`。

### BaseChannel 构造参数

| 参数 | 默认值 | 当前是否生效 |
|------|--------|----------------|
| `dm_policy` | `"open"` | **遗留**。存下来做兼容，**不参与过滤** |
| `group_policy` | `"open"` | **遗留**。同上 |
| `allow_from` | `set()`（`allow_from or []`） | **遗留**。白名单已迁到 `access_control.json` |
| `deny_message` | `""` | **遗留**。ACL 拒绝文案改走 `_acl_msg()` i18n，不再读这个字段 |
| `require_mention` | `False` | **仍生效**。`_check_group_mention()`；子类在入队前调用 |
| `no_text_debounce` | `True` | **仍生效**。`_apply_no_text_debounce()` |
| `streaming_enabled` | `False` | **仍生效** |
| `access_control_dm` | `False` | **现行 ACL**：私信是否走白/黑名单门 |
| `access_control_group` | `False` | **现行 ACL**：群聊是否走白/黑名单门 |
| `display_config` | `ChannelDisplayConfig()` | 工具/思考展示 |

配置层 `BaseChannelConfig` 另有 `dm_disabled=False` / `group_disabled=False`（频道级静音）。**BaseChannel 本身不读这两个字段**；Slack / Matrix 等子类自行实现。

旧配置迁移（`config._migrate_access_control_fields`）：

- `dm_policy == "allowlist"` → `access_control_dm = True`
- `group_policy == "allowlist"` → `access_control_group = True`
- `allow_from` 列表 → `AccessControlStore.import_allow_from()` 写入白名单

### ChannelDisplayConfig（替代旧 `show_tool_details` / `filter_tool_messages` / `filter_thinking`）

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `show_tool_details` | `True` | 是否展开工具参数/结果；`False` 时预览为 `"..."` |
| `show_thinking` | `True` | 是否渲染 `REASONING` / thinking 块 |
| `show_tool_calls` | `True` | 是否渲染工具调用 |
| `show_tool_results` | `True` | 是否渲染工具结果 |
| `tool_call_max_length` | `200` | 调用参数预览长度；`0` = 不截断 |
| `tool_result_max_length` | `500` | 结果预览长度；`0` = 不截断 |

`from_config(config, show_tool_details=True)`：`show_tool_details` 来自**全局** `Config.show_tool_details`（默认 `True`），其余字段读频道配置。

`RenderStyle` 默认：`supports_markdown=True`、`supports_code_fence=True`、`use_emoji=True`。

---

## 必须实现的接口

基类用 `raise NotImplementedError`（不是 `@abstractmethod`）。飞书/钉钉均实现下列方法。

| 方法 | 说明 |
|------|------|
| `from_env(process, on_reply_sent=None)` | 从环境变量构造 |
| `from_config(process, config, on_reply_sent=None, display_config=None, no_text_debounce=True)` | 从配置构造。Manager 还会按签名可选传入 `workspace_dir` |
| `start()` | 启动连接（WebSocket / 轮询等） |
| `stop()` | 停止连接 |
| `send(to_handle, text, meta=None)` | 发送一条文本 |
| `build_agent_request_from_native(native_payload)` | 原生 payload → `AgentRequest`；约定把 `request.channel_meta = meta` |

`clone(config)` 默认调用 `from_config(process=self._process, config=..., on_reply_sent=..., display_config=...)`，热重载/重启频道时用。

建议子类同时覆盖（非强制）：

| 方法 | 默认行为 | 飞书示例 |
|------|----------|----------|
| `resolve_session_id(sender_id, channel_meta=None)` | `f"{self.channel}:{sender_id}"` | 群聊用 chat_id 短后缀，私聊用 open_id 短后缀 |
| `get_to_handle_from_request(request)` | `request.user_id` | `feishu:sw:{session_id}` |
| `get_on_reply_sent_args(request, to_handle)` | `(to_handle, session_id)` | `(user_id, session_id)` |
| `merge_native_items(items)` | 拼接 `content_parts`，合并部分 meta | 用最后一条的 sender/session/meta |
| `send_media(...)` / `send_content_parts(...)` | 文本合并 + URL 占位 | 真实发图/发文件 |

`schema.ChannelMessageConverter` 协议对应 `build_agent_request_from_native` + `send_response`。  
`schema.ChannelAddress`（`kind` / `id` / `extra`）是统一路由结构，BaseChannel 默认仍用字符串 `to_handle`。

---

## 与 ChannelManager 的边界

**Manager 拥有：**

- `UnifiedQueueManager`：key = `(channel_id, session_id, priority_level)`，每 key 一条 `asyncio.Queue`（`maxsize=1000`）+ 按需创建的 consumer
- 线程安全入队：`enqueue(channel_id, payload)` → `loop.call_soon_threadsafe(_enqueue_one)`
- 优先级：用 `ch._extract_query_from_payload(payload)` + `CommandRegistry.get_priority_level()`（默认 20；`/stop` 为 0）
- session 键：`ch.get_debounce_key(payload)`（与时间防抖同一套）
- 入队超时 30s；空闲队列 600s 后清理
- `start_all()`：创建 QueueManager、对 `uses_manager_queue=True` 的频道 `set_enqueue()`、后台 `start()`
- `set_workspace(workspace)`：给每个 channel 注入 workspace + 同一个 `CommandRegistry`

**BaseChannel 仍拥有：**

- 原生消息解析与 `build_agent_request_from_native`
- `consume_one` 时间防抖（仅当走 `consume_one` 且 `_debounce_seconds > 0`）
- `_debounce_payload` / `_apply_no_text_debounce` 无文本合并
- `_access_control_gate` ACL
- `_process` 循环、渲染、`send` / `send_media`
- TaskTracker 路径（`_consume_with_tracker`）
- 流式 Hook、工具展示过滤

**入队方向：** 子类收到平台消息后调用 `self._enqueue(native_dict)`，不要自己 `put` 队列。飞书在 `_on_message` 里组好 native dict（含 `acl_sender_id`、`content_parts`、`meta`）再 `_enqueue`。

**消费方向（Manager `_process_batch`）：**

```
len>1 且 native dict  → merge_native_items → _consume_one_request
len>1 且 AgentRequest → merge_requests    → _consume_one_request
单条 native dict     → _consume_one_request          # 跳过 consume_one
单条非 native        → consume_one
```

因此：**主路径上 native payload 不会经过 `consume_one()` 的时间防抖**。同 session 连发由 QueueManager 串行 + drain 合并消化。`consume_one()` 的 `_debounce_seconds` 只对「直接调用 consume_one」或非 native 单条 payload 生效。钉钉注释写明：时间防抖关闭，交给 Manager drain。

旧文档里的「每频道 4 个 worker」「`_in_progress` / `_pending` session 合并」已不在 BaseChannel，也不在 Channel 队列上；同 QueueKey 严格串行，不同 session / 不同优先级可并行。

---

## 工作链路详解

### 链路 1: 消息入队 (Enqueue)

```
平台事件（WS/轮询）
    │
    ▼
子类解析 → native dict
    │  群聊 mention? _check_group_mention() 不通过则直接 return
    ▼
self._enqueue(payload)          # ChannelManager._make_enqueue_cb
    │
    ▼
ChannelManager.enqueue()        # 线程安全
    │
    ▼
_enqueue_one()
    ├─ query  = ch._extract_query_from_payload(payload)
    ├─ prio   = CommandRegistry.get_priority_level(query)   # 默认 20
    └─ sid    = ch.get_debounce_key(payload)
    │
    ▼
UnifiedQueueManager.enqueue(channel, sid, prio, payload)   # timeout=30s
```

Native dict 约定字段（飞书/钉钉实际写入）：

```
channel_id, sender_id, acl_sender_id, content_parts, meta
可选: session_id, user_id, session_webhook
```

`meta` 常用键：`is_group`、`bot_mentioned`、`has_bot_command`、`user_name`、`session_webhook`、各频道路由键。

---

### 链路 2: 消费循环 (Consume Loop)

```
UnifiedQueueManager consumer（按 QueueKey 按需创建）
    │
    ▼
queue.get()
    │
    ▼
drain get_nowait() 直到 QueueEmpty     # 同 key 连发合并
    │
    ▼
_process_batch(ch, batch)
    │
    ├─ native 多条 → merge_native_items → _consume_one_request
    ├─ request 多条 → merge_requests   → _consume_one_request
    ├─ native 单条 → _consume_one_request
    └─ 其他单条  → consume_one
```

不再有固定 worker 池，也没有跨 worker 的 `_drain_same_key` 锁表。

---

### 链路 3: consume_one（时间防抖）

```
consume_one(payload)
      │
      ├─ _debounce_seconds > 0 且 _is_native_payload（dict 且含 content_parts）
      │     1. get_debounce_key(payload)
      │     2. 已有缓冲则 _on_debounce_buffer_append()
      │     3. append 到 _debounce_pending[key]
      │     4. 取消旧 timer，sleep(_debounce_seconds) 后 flush
      │     5. merge_native_items → _consume_one_request
      │
      └─ 否则 _consume_one_request(payload)
```

`get_debounce_key`：dict 优先 `payload["session_id"]`，否则 `resolve_session_id(sender_id, meta)`；对象则用 `payload.session_id`。

---

### 链路 4: `_consume_one_request`（消费核心）

```
_consume_one_request(payload)
      │
      ├─ 1. _debounce_payload()          # 无文本防抖，写回 payload
      │      False → return（已缓存，等文本）
      │
      ├─ 2. _access_control_gate()       # ACL；True = 已拒绝并回消息
      │      True → return
      │
      ├─ 3. 已注入 workspace + CommandRegistry，且不是控制命令
      │      → _payload_to_request → _consume_with_tracker → return
      │         （/stop、/status 等 is_control_command 为 True 则跳过 tracker）
      │
      └─ 4. 无 workspace / 控制命令：走本地 _run_process_loop
             _payload_to_request()
             从 payload 挂 channel_meta（含 session_webhook）
             get_to_handle_from_request()
             _before_consume_process()
             构建 send_meta（payload.meta + session_webhook + bot_prefix）
             _run_process_loop()
```

#### (a) `_payload_to_request`

- 已有 `session_id` + `input` 属性 → 当作 `AgentRequest` 原样返回
- 否则 `build_agent_request_from_native(payload)`
- `payload is None` → `ValueError`

子类实现 native 转换时通常：解析 `content_parts` → `build_agent_request_from_user_content(...)` → `setattr(request, "channel_meta", meta)`。  
`AgentRequest` schema **没有** `channel_meta` 字段，靠动态属性挂载，供 `_before_consume_process` / 发送路由使用。

`build_agent_request_from_user_content`：空 `content_parts` 时填一个空格 `TextContent`；组装 `Message(type=MESSAGE, role=USER)` + `AgentRequest(session_id, user_id=sender_id, input, channel)`。

#### (b) 无文本防抖（先于转换，写回 payload）

```python
should_process, merged = self._apply_no_text_debounce(session_id, content_parts)
# True 时写回 payload["content_parts"] 或 input[0].content（优先 model_copy）
```

与旧文档不同：合并发生在 **payload 上**，再 `_payload_to_request`，而不是先转 request 再改 `input[0].content`。

#### (c) channel_meta / send_meta（仅非 tracker 路径在此处构建）

```python
if isinstance(payload, dict):
    meta_from_payload = dict(payload.get("meta") or {})
    if payload.get("session_webhook"):
        meta_from_payload["session_webhook"] = payload["session_webhook"]
    setattr(request, "channel_meta", meta_from_payload)
```

`send_meta` 同样从 payload 复制；若 `bot_prefix` / `_bot_prefix` 非空且 meta 中没有，则注入。  
tracker 路径改在 `_stream_with_tracker` 里用同一套规则组 `send_meta`。

---

### 链路 5: 流程执行

两条执行循环，事件处理规则接近。

**A. `_run_process_loop`**（无 workspace，或控制命令）

```
_run_process_loop(request, to_handle, send_meta)
      │
      ▼
async for event in self._process(request):
      │
      ├─ object=="content"
      │     on_event_content() 返回 True → continue
      ├─ object=="message" AND status==Completed
      │     on_event_message_completed() → 默认 send_message_content()
      │     _send_model_fallback_notice()   # console 频道跳过
      └─ object=="response"
            last_response = event
            on_event_response()
      │
      ▼
无错误 → _on_process_completed() + _commit_turn_usage(emit_sse=False)
有错误 → _on_consume_error()
最后  → _on_reply_sent(channel, *get_on_reply_sent_args())
finally → _finish_response_cycle(session_id)  # 浏览器 kernel 清理
```

**B. `_consume_with_tracker` → `_stream_with_tracker`**（常规用户消息）

```
get_or_create_chat(session_id, user_id, channel, name=_extract_chat_name)
touch_chat()
task_tracker.attach_or_start(chat.id, payload, _stream_with_tracker, ...)
      │
      ├─ is_new → 从 queue 抽 SSE（给 Console / Web）
      └─ 已有任务 → 打 warning 并忽略
                    （正常不应发生：UnifiedQueueManager 已按 session 串行）
```

`_stream_with_tracker` 额外：

- `request.channel_instance = self`
- `streaming_enabled` 时 `_dispatch_streaming_event()`；若已由流式路径消化 completed message，则跳过 `on_event_message_completed`
- 把 Event 序列化成 SSE `data: ...\n\n`（会剥 scroll headline）
- 成功时 `_commit_turn_usage(emit_sse=True)` 再 yield `turn_usage`
- 取消 / 异常时 `_on_consume_error`（文案 `"Internal error"`）

控制命令（`CommandRegistry.is_control_command`，query 以 `/` 开头且匹配已注册前缀，如 `/stop`、`/status`、`/approval`）**绕过 TaskTracker**，直接 `_run_process_loop`，以便立刻响应。

---

### 链路 6: 消息发送 (Send Path)

```
on_event_message_completed()          # 默认
    │
    ▼
send_message_content(to_handle, message, meta)
    │
    ▼
_renderer.message_to_parts(message) → List[OutgoingContentPart]
    │  空列表则 skip send
    ▼
send_content_parts(to_handle, parts, meta)
    │
    ├─ 拼接 TEXT / REFUSAL
    ├─ 可选 bot_prefix
    ├─ media 追加为 [Image: url] / [Video:] / [File:] / [Audio]
    ├─ body 非空 → send(to_handle, body, meta)
    └─ 每个 media → send_media()   # 默认 no-op
```

`send_response(to_handle, response, meta)`：从 `AgentResponse.output` **倒序**找最后一条 `type==MESSAGE`，抽 TEXT/REFUSAL，再 `send()`。

`send_event(user_id=, session_id=, event=, meta=)`：仅当 `object==message` 且 Completed，经 `to_handle_from_target()` 后走 `send_message_content`（cron / runner 主动推）。

`send_approval_notification(...)`：构造 `metadata.message_type=tool_guard_approval` 的 mock Event，走 `on_event_message_completed`（卡片频道画审批卡，其它频道变纯文本）。

---

### 链路 7: 错误处理 (`_on_consume_error`)

```
异常 或 last_response.error
      │
      ▼
_on_consume_error(request, to_handle, err_text)
      │
      ▼
send_content_parts(to_handle, [TextContent(text=err_text)], request.channel_meta)
```

文案因路径而异：

- `_run_process_loop` 响应错误：`"Error: {err_msg}"`
- `_run_process_loop` 异常：`"An error occurred while processing your request."`
- `_stream_with_tracker` 异常：`"Internal error"`

---

## 消息合并策略

### 1. `merge_native_items`

默认：拼接全部 `content_parts`；meta 后写覆盖这些键：`reply_future`、`reply_loop`、`incoming_message`、`conversation_id`、`message_id`。返回：

```
channel_id, sender_id, acl_sender_id, content_parts, meta
```

飞书覆盖为：parts 全拼，**sender/session/meta 取最后一条**。

### 2. `merge_requests`

拼接各 request `input[0].content`，保留第一个 request 的 session/meta；优先 `model_copy`。

### 3. `_apply_no_text_debounce`

| 条件 | 行为 |
|------|------|
| `_no_text_debounce is False` | 弹出 pending，立刻处理 `pending + 当前` |
| 无 TEXT/REFUSAL 文本，但有 `AUDIO` | **立即处理**（语音视为完整输入） |
| 无文本且无音频 | 写入 `_pending_content_by_session[session_id]`，返回 `(False, [])` |
| 有文本 | 弹出 pending，`(True, pending + 当前)` |

「有文本」：`ContentType.TEXT` 且 `text.strip()`，或 `REFUSAL` 且 `refusal.strip()`。

---

## 权限：DM / Group / Allowlist / Mention

### Mention（入队前，子类调用）

```python
def _check_group_mention(self, is_group: bool, meta: dict) -> bool:
    if not is_group or not self.require_mention:
        return True
    return bool(meta.get("bot_mentioned") or meta.get("has_bot_command"))
```

`require_mention=False`（默认）时群聊不过滤。飞书/Discord/Telegram/OneBot 等在组好 `meta` 后、`_enqueue` 前调用；不通过则静默丢弃。**不在 `consume_one` 里做。**

### ACL 门（消费时，`_access_control_gate`）

`access_control_enabled` = `access_control_dm or access_control_group`。两者默认 `False`，门关闭。

开启后：

1. sender：dict 用 `acl_sender_id or sender_id`；对象用 `acl_sender_id or user_id`（共享 session 时必须用真实发送者）
2. 无 sender_id → 放行
3. `meta.is_group` 与 `access_control_group` / `access_control_dm` 不匹配 → 放行（该聊天类型未开 ACL）
4. `AccessControlStore`（`<workspace>/access_control.json`）按 **channel 名** 查：
   - 白名单 → 放行
   - 黑名单 → 发 `_acl_msg("blocked")`，返回 True（拦截）
   - 都不是 → `add_pending(channel, sender_id, first_message, username)`，发 `_acl_msg("pending", sender_id=...)`，拦截

拒绝消息走 `send_content_parts`。i18n 语言取 `self._language`，不在 `{zh, en, ja, ru, pt-BR, id}` 时：`zh*` → zh，否则 en。

**`dm_policy` / `group_policy` / `allow_from` / `deny_message` 不再过滤。** 旧 allowlist 应已迁移进 JSON 白名单。

---

## 工具消息过滤

旧参数 `filter_tool_messages`、`filter_thinking` **已删除**。现行全部由 `ChannelDisplayConfig` + `MessageRenderer.message_to_parts`：

| 配置 | 效果 |
|------|------|
| `show_thinking=False` | `MessageType.REASONING` 返回空列表；thinking 块不渲染；流式 reasoning 也被吞掉 |
| `show_tool_calls=False` | 不渲染 FUNCTION/PLUGIN/MCP `_CALL` |
| `show_tool_results=False` | 不渲染 `*_CALL_OUTPUT` 文本；`internal_tools`（如 view_image）的媒体也对用户隐藏 |
| `show_tool_details=False` | 参数/结果预览为 `"..."` |
| 长度字段 | 超过则截断加 `...`；`0` 不截断 |

`on_event_content`：`type==DATA` 且 `status==InProgress` 且 `show_tool_results` 时，把流式工具输出格式化成 `⌛️ **{tool_name}**:` 立刻 `send_content_parts`。`show_tool_results=False` 则整段跳过。

渲染内容类型：`TextContent`、`ImageContent`、`VideoContent`、`AudioContent`、`FileContent`、`RefusalContent`。文本会 `strip_headline`（去掉 scroll 注释 `⟦ … ⟧`）。

---

## 可覆盖的 Hook

| 方法 | 时机 |
|------|------|
| `_before_consume_process(request)` | `_process` 前。飞书在此保存 `receive_id`，并可预创建流式卡片 |
| `on_event_content(...)` | content 事件。返回 True 表示已处理 |
| `on_event_message_completed(...)` | 消息完成。默认 `send_message_content`。钉钉可改成合并后再发 |
| `on_event_response(...)` | response 事件。默认 no-op |
| `_on_process_completed(...)` | 全部事件成功结束。飞书可加 DONE 表情 |
| `_on_consume_error(...)` | 错误。默认把 err_text 当文本发出 |
| `_on_debounce_buffer_append(...)` | 时间防抖追加。可用来解开上一条 `reply_future` |
| `on_streaming_start/delta/end(...)` | `streaming_enabled` 时 reasoning/message 流式 |
| `send_media(...)` | 单条媒体。默认 no-op |
| `refresh_webhook_or_token()` | 可选刷新。默认 no-op |
| `to_handle_from_target(user_id=, session_id=)` | cron/主动推的 handle。默认 `user_id` |
| `health_check()` | 默认 `{"channel", "status": "healthy", "detail": "Channel is loaded and running."}` |
| `doctor_connectivity_notes(agent_id, config, timeout=)` | `copaw doctor --deep` 额外探测。默认 `[]` |
| `_on_turn_usage_ready(turn, ctx)` | 每轮 token 用量就绪。Console 可打状态行 |

---

## 时序图

```
用户消息
    │
    ▼
子类 _check_group_mention（群聊+require_mention）
    │
    ▼
self._enqueue(native) ──► ChannelManager.enqueue()
    │
    ▼
UnifiedQueueManager
  key=(channel, session_id, priority)
    │
    ▼
_consume_queue: get + drain
    │
    ▼
_process_batch()
    │
    ├─ native → _consume_one_request()     # 跳过 consume_one 时间防抖
    └─ 其他单条 → consume_one()
                      │
                      └─ _debounce_seconds>0? 缓冲 : _consume_one_request
                              │
                    _debounce_payload()          # 无文本 / 音频例外
                              │
                    _access_control_gate()
                              │
              ┌───────────────┴────────────────┐
              │ workspace 且非控制命令          │ 否则
              ▼                                ▼
     _consume_with_tracker              _run_process_loop
     _stream_with_tracker                     │
              │                                │
              └───────────────┬────────────────┘
                              │
                    _before_consume_process
                    _process(request) Event 流
                              │
              ┌───────────────┼────────────────┐
              ▼               ▼                ▼
     streaming hooks   on_event_message   on_event_content
     (optional)        _completed         (工具流式预览)
                              │
                              ▼
                    send_message_content()
                              │
                              ▼
                    _renderer.message_to_parts()
                              │
                              ▼
                    send_content_parts()
                              │
                              ▼
                    send() / send_media()
                              │
                              ▼
                    _on_reply_sent()
                    _finish_response_cycle()
```

---

## 配置示例

```python
from qwenpaw.app.channels.renderer import ChannelDisplayConfig

channel = SomeChannel(
    process=stream_query,
    on_reply_sent=on_reply_sent,
    display_config=ChannelDisplayConfig(
        show_tool_details=True,
        show_thinking=True,
        show_tool_calls=True,
        show_tool_results=True,
        tool_call_max_length=200,
        tool_result_max_length=500,
    ),
    no_text_debounce=True,
    streaming_enabled=False,
    require_mention=False,
    access_control_dm=False,
    access_control_group=False,
    # 以下为遗留字段，不再做访问控制：
    dm_policy="open",
    group_policy="open",
    allow_from=["user1", "user2"],
    deny_message="Unauthorized",
)
```

Manager 侧：`ChannelManager.from_config(process, config, on_last_dispatch=..., workspace_dir=...)` 会按 `from_config` 签名注入 `display_config` / `no_text_debounce` / `workspace_dir`，然后 `set_workspace()` + 写入 `_language`。

---

## 关键设计

1. **队列所有权在 Manager**：Channel 只 `set_enqueue` + 定义 `consume_one` / `_consume_one_request`。`uses_manager_queue=False` 的频道（voice/sip）完全自管。
2. **UnifiedQueueManager 按 (channel, session, priority) 串行**；同 key drain 合并。不再每频道固定 4 worker。
3. **主路径 native 消息跳过 `consume_one` 时间防抖**；无文本合并仍在 `_consume_one_request` 内。
4. **双循环**：有 workspace 的用户消息走 TaskTracker + SSE；控制命令和无 workspace 走 `_run_process_loop`。
5. **ACL 与 mention 分层**：mention 在子类入队前；白/黑名单在消费门。`dm_policy`/`allow_from` 只兼容存储。
6. **展示与发送解耦**：`MessageRenderer` + `ChannelDisplayConfig` 决定用户看见什么；`send` / `send_media` 决定怎么发到平台。
)
