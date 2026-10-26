# FeishuChannel 详解

> QwenPaw v2.2.1 · 2026-09-15
>
> 源码路径：`src/qwenpaw/app/channels/feishu/`
> 注册名：`registry._BUILTIN_SPECS["feishu"] = (".feishu", "FeishuChannel")`
> 配置模型：`src/qwenpaw/config/config.py` → `FeishuConfig`（挂在全局配置字段 `feishu`）

## 一、架构概述

`FeishuChannel` 是 QwenPaw 连接飞书（中国区 Feishu）/ Lark（国际区）的渠道适配器：用 **lark-oapi WebSocket 长连接**收事件（无需公网 IP），用 **Open API**（SDK `lark.Client`，tenant_access_token 由 SDK `TokenManager` 管理）发消息。HTTP 辅助请求走 **httpx.AsyncClient**（`_http_client`），不再使用 aiohttp。

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Feishu / Lark Platform                            │
│   ┌──────────────┐                           ┌──────────────────┐   │
│   │  User/Group  │                           │  Open API        │   │
│   │  Messages    │                           │  domain:         │   │
│   └──────┬───────┘                           │  FEISHU_DOMAIN / │   │
│          │ WebSocket                          │  LARK_DOMAIN /   │   │
│          │ (long connection)                  │  custom http(s)  │   │
│          ▼                                    └────────┬─────────┘  │
│   ┌─────────────────────────────────┐                  │            │
│   │  lark.ws.Client                 │         ┌────────▼─────────┐  │
│   │  (_run_ws_forever, 独立线程     │         │  FeishuChannel   │  │
│   │   + 独立 _ws_loop)              │         │  _client (SDK)   │  │
│   └─────────────┬───────────────────┘         │  _ws_client      │  │
│                 │ P2ImMessageReceiveV1        │  _http_client    │  │
│                 │ P2CardActionTrigger         │   (httpx)        │  │
│                 ▼                             └────────┬────────┘  │
│   ┌─────────────────────────────────┐                   │            │
│   │  _on_message_sync /             │───────────────────┘            │
│   │  _card_handler.handle_card_     │                                │
│   │    action  (WS 线程同步回调)    │                                │
│   └─────────────┬───────────────────┘                                │
│                 │ asyncio.run_coroutine_threadsafe                   │
│                 │ （消息）/ 卡片内直接 _enqueue（审批命令）            │
│                 ▼                                                     │
│   ┌─────────────────────────────────┐                                 │
│   │  _on_message（主事件循环）       │                                 │
│   │  1. message_id 去重             │                                 │
│   │  2. 解析 sender/chat/thread    │                                 │
│   │  3. SDK 下载媒体到 _media_dir  │                                 │
│   │  4. 构建 native dict            │                                 │
│   │  5. self._enqueue(native)       │                                 │
│   │     → ChannelManager.enqueue    │                                 │
│   └─────────────────────────────────┘                                 │
│                                                                     │
│   ChannelManager 队列 → BaseChannel.consume_one                      │
│     → build_agent_request_from_native → _process                     │
│     → on_event_message_completed → send_content_parts                │
│                                                                     │
│   ┌─────────────────────────────────┐                                 │
│   │  回复发送 (send_content_parts)  │                                 │
│   │  1. _get_receive_for_send       │                                 │
│   │  2. 话题 → _reply_in_thread     │                                 │
│   │  3. 文本 → _send_text           │                                 │
│   │     (post md / interactive 表)  │                                 │
│   │  4. 图片 → _upload_image + send │                                 │
│   │  5. 文件 → _upload_file + send  │                                 │
│   └─────────────────────────────────┘                                 │
└─────────────────────────────────────────────────────────────────────┘
```

包内文件：

| 文件 | 职责 |
|------|------|
| `channel.py` | `FeishuChannel`：WS 收消息、Open API 发送、流式卡片、路由 |
| `constants.py` | 去重/缓存/WS/流式等常量 |
| `utils.py` | session 短 ID、post/卡片文本提取、markdown 表格转 interactive |
| `cards/dispatcher.py` | 运行时卡片分发器 `FeishuCardHandler`（`channel.py` 实际导入） |
| `cards/context.py` | 卡片共用的 meta / body / session_ctx 提取 |
| `cards/tool_guard.py` | tool-guard 审批卡：构建、解析、render、handle |
| `card_handler.py` / `card_templates.py` | 同功能的旧版并列实现；`channel.py` 不导入，单测仍引用 |
| `__init__.py` | 导出 `FeishuChannel` |

## 二、核心设计思想

1. **线程安全的消息接收**：`lark.ws.Client` 跑在独立线程 `_ws_thread`，该线程自建 `_ws_loop`。IM 消息经 `_on_message_sync` 用 `asyncio.run_coroutine_threadsafe` 投到主循环的 `_on_message`。卡片点击回调必须同步返回 `P2CardActionTriggerResponse`，审批命令在 WS 线程直接调 `_enqueue`（`ChannelManager.enqueue` 被设计为线程安全）。
2. **Token 交给 SDK**：不再自管 `_tenant_access_token` 缓存。发消息、传文件走 `lark.Client` 异步 API；仅 `_fetch_bot_open_id` 通过 `TokenManager.get_self_tenant_token(self._client._config)` 取 token 调 `/open-apis/bot/v3/info`。
3. **会话路由解耦**：接收时把 `(receive_id_type, receive_id)` 按 `session_id`（及 open_id）写入 `_receive_id_store` 并落盘；发送时用 `to_handle` / meta 反查，避免把完整 receive_id 塞进消息体。
4. **媒体本地化**：入站图片/文件经 SDK `im.v1.message_resource.aget` 写到 `_media_dir`；出站再 upload 后按 `image` / `file` / `audio` 发出。
5. **message_id 去重 + 过期重试丢弃**：`OrderedDict` 缓存，上限 `FEISHU_PROCESSED_IDS_MAX`；WS 回调里用 `_clock_offset` 校正时钟，超过 `FEISHU_STALE_MSG_THRESHOLD_MS` 的事件当作飞书重试直接丢弃。
6. **区域与自定义网关**：`domain` 为 `"feishu"` / `"lark"` 或完整 `http(s)` base URL，经 `_sdk_domain()` 同时传给 HTTP Client 与 WS Client。
7. **话题与卡片**：群话题用 `thread_id` + `im.v1.message.areply(reply_in_thread=True)`；交互卡片走 `cards/` 分发器；可选 CardKit 流式打字卡（`streaming_enabled`）。

## 三、成员变量详解

### 配置类变量（`__init__` / `FeishuConfig`）

| 变量 | 类型 | 说明 |
|------|------|------|
| `channel` | `str` | 类属性，固定 `"feishu"` |
| `enabled` | `bool` | 渠道是否启用（`FeishuConfig.enabled`，继承 `BaseChannelConfig`） |
| `app_id` | `str` | 飞书/Lark 应用 ID |
| `app_secret` | `str` | 应用密钥 |
| `bot_prefix` | `str` | 发出文本前缀（继承自 `BaseChannelConfig`） |
| `encrypt_key` | `str` | 事件订阅加密密钥，交给 `EventDispatcherHandler.builder` |
| `verification_token` | `str` | 事件订阅 Verification Token |
| `domain` | `str` | `"feishu"`（默认）/ `"lark"` / `http(s)://...` 自定义网关 |
| `media_dir`（配置） | `Optional[str]` | 配置项；实例上对应 `_media_dir` |
| `streaming_enabled` | `bool` | 是否启用 CardKit 流式卡片（基类字段，Feishu 覆盖钩子） |
| `share_session_in_group` | `bool` | `True` 时群内把 `feishu_sender_id` 设为 `"group"`，成员共享 user 侧会话 |
| `_media_dir` | `Path` | 媒体目录：优先 `workspace_dir/media`，否则配置路径，再否则 `DEFAULT_MEDIA_DIR` |
| `_workspace_dir` | `Path \| None` | Agent 工作区，用于媒体目录和 `feishu_receive_ids.json` |
| `_STREAM_DELTA_MIN_INTERVAL_S` | `float` | 类属性，等于 `FEISHU_STREAM_MIN_INTERVAL_S`（0.15），限制流式更新频率 |

`from_config` 还会传入基类策略字段：`dm_policy` / `group_policy` / `allow_from` / `deny_message` / `require_mention` / `no_text_debounce` / `access_control_dm` / `access_control_group` / `display_config`。基类注释写明 `dm_policy`/`allow_from` 为遗留字段，真正拦截在 `BaseChannel._access_control_gate`。

### 客户端与网络

| 变量 | 类型 | 说明 |
|------|------|------|
| `_client` | `Any` | `lark.Client`，异步发消息、上传、反应、拉消息/资源 |
| `_ws_client` | `Any` | `lark.ws.Client`，长连接收事件 |
| `_ws_thread` | `Optional[threading.Thread]` | WS 后台线程，target=`_run_ws_forever` |
| `_loop` | `Optional[asyncio.AbstractEventLoop]` | 主事件循环（`start()` 里 `get_running_loop()`） |
| `_ws_loop` | `Optional[asyncio.AbstractEventLoop]` | WS 线程内自建的 event loop |
| `_closed` | `bool` | 关闭标志，阻止继续投递消息 |
| `_stop_event` | `threading.Event` | 停止/重连等待信号 |
| `_http_client` | `httpx.AsyncClient \| None` | bot info、按 URL 拉字节；**不是 aiohttp** |
| `_clock_offset` | `int` | `server_time - local_time`（毫秒），来自 bot info 响应 `Date` |
| `_last_ws_recv_time` | `float` | 最近一次 WS 收到数据（含 pong）的时间，用于静默断线检测 |
| `_card_handler` | `FeishuCardHandler` | `cards.dispatcher.FeishuCardHandler(self)` |

### 身份与缓存

| 变量 | 类型 | 说明 |
|------|------|------|
| `_bot_open_id` | `Optional[str]` | 机器人 open_id，用于 @ 检测和忽略自身消息 |
| `_nickname_cache` | `Dict[str, str]` | open_id → 通讯录姓名 |
| `_nickname_cache_lock` | `asyncio.Lock` | 昵称缓存锁 |

### 会话路由

| 变量 | 类型 | 说明 |
|------|------|------|
| `_receive_id_store` | `Dict[str, Tuple[str, str]]` | key → `(receive_id_type, receive_id)` |
| `_receive_id_lock` | `asyncio.Lock` | 路由存储锁 |

### 消息去重

| 变量 | 类型 | 说明 |
|------|------|------|
| `_processed_message_ids` | `OrderedDict[str, None]` | 已处理 `message_id`，超限 `popitem(last=False)` |

### 继承自 BaseChannel（与飞书交界相关）

| 变量 | 类型 | 说明 |
|------|------|------|
| `_process` | `ProcessHandler` | Agent 处理函数，yield Event |
| `_enqueue` | `EnqueueCallback` | 由 `ChannelManager.set_enqueue(_make_enqueue_cb("feishu"))` 注入 |
| `uses_manager_queue` | `bool` | 基类默认 `True`，Feishu **未覆盖**，因此走 Manager 队列 |
| `require_mention` | `bool` | 群聊必须 @；`_on_message` 调基类 `_check_group_mention` |
| `_renderer` | `MessageRenderer` | 完成消息 → `OutgoingContentPart` |
| `streaming_enabled` | `bool` | 为 True 时基类还会打流式钩子 |

**已删除、源码中不存在的旧字段：** `_http`、`_tenant_access_token`、`_tenant_access_token_expire_at`、`_token_lock`。

## 四、工作机制详解

### 4.1 与 BaseChannel / ChannelManager 的交界

```
FeishuChannel.start()
    ChannelManager.start_all 里对 uses_manager_queue=True 的渠道
        ch.set_enqueue(manager._make_enqueue_cb("feishu"))
        后台 task: await ch.start()

收消息（本频道）
    _on_message / 卡片 _enqueue_approval_command
        → self._enqueue(native)          # 唯一入队点
        → ChannelManager.enqueue("feishu", payload)
        → UnifiedQueueManager
        → BaseChannel.consume_one(payload)
            防抖 merge_native_items（Feishu 覆盖）
            _access_control_gate
            _payload_to_request
                → FeishuChannel.build_agent_request_from_native
            _before_consume_process      # Feishu 覆盖：存 receive_id、预创建流式卡
            _run_process_loop            # Feishu 未覆盖，用基类
                on_event_message_completed  # Feishu 覆盖：先试卡片
                    未命中卡片 → super → send_message_content
                        → FeishuChannel.send_content_parts   # 出站主路径
                _on_process_completed    # Feishu 覆盖：DONE 反应
                _on_reply_sent(*get_on_reply_sent_args)

主动发送（cron 等）
    FeishuChannel.send(to_handle, text, meta)
        → _get_receive_for_send → _send_text
```

`to_handle` 由 `get_to_handle_from_request` 生成：优先 `feishu:sw:{session_id}`。

### 4.2 消息接收流程

```
Feishu Server (WebSocket)
    │
    ▼
[_run_ws_forever 独立线程 + _ws_loop]
    指数退避重连：FEISHU_WS_INITIAL_RETRY_DELAY → ×BACKOFF → 上限 MAX
    健康检查：_conn is None 或静默 > FEISHU_WS_RECV_TIMEOUT → 停 loop 重连
    EventDispatcherHandler:
      p2.im.message.receive_v1  → _on_message_sync
      p2.card.action.trigger    → _card_handler.handle_card_action
      reaction created/deleted  → no-op（占位注册）
    │
    ▼
_on_message_sync（WS 线程）
    ├─► _closed 则丢弃
    ├─► header.app_id ≠ self.app_id 则丢弃（多实例串台防护）
    ├─► create_time 过旧（> FEISHU_STALE_MSG_THRESHOLD_MS，含 _clock_offset）丢弃
    └─► run_coroutine_threadsafe(_on_message, self._loop)
            │
            ▼
        _on_message（主循环）
            ├─► _processed_message_ids 去重
            ├─► sender_type==bot 且 open_id==_bot_open_id → 忽略
            ├─► 昵称：事件字段，否则 _get_user_name_by_open_id
            ├─► sender_display_string → "nick#后4位"
            ├─► @_all 或 mentions 中匹配 _bot_open_id → bot_mentioned
            ├─► _parse_message_content（text/post/image/file/media/audio/interactive）
            ├─► parent_id → _process_quoted_message（拉被引用消息）
            ├─► 组装 meta（含 feishu_thread_id / receive_id）
            ├─► _check_group_mention（基类；require_mention 时看 bot_mentioned）
            ├─► _add_reaction(message_id, "Typing")
            ├─► resolve_session_id；话题则 user_id=thread:{短id}
            ├─► share_session_in_group 且非话题：feishu_sender_id="group"
            └─► self._enqueue(native)   # → ChannelManager
```

`_on_message` **不再**调用 `_check_allowlist`（该方法本频道也不存在）。白名单/ACL 在 Manager 出队后的 `BaseChannel._access_control_gate` 执行。

### 4.3 回复发送流程

Feishu **不重写** `_run_process_loop`。流程在基类循环里，出站由本频道覆盖的钩子完成：

```
BaseChannel._run_process_loop
    async for event in self._process(request):
        obj=="content"  → on_event_content（基类；工具输出可 send_content_parts）
        obj=="message" && Completed
            → FeishuChannel.on_event_message_completed
                  ├─► _card_handler.try_send_card_for_event
                  │     命中 metadata.message_type（如 tool_guard_approval）
                  │     → cards.tool_guard.render → _send_message(..., "interactive")
                  └─► 未命中 → super().on_event_message_completed
                        → send_message_content
                          → _message_to_content_parts
                          → send_content_parts
        obj=="response" → on_event_response
    出错 → _on_consume_error → send_content_parts
    成功 → FeishuChannel._on_process_completed
              _add_reaction(_last_sent_message_id, "DONE")
           on_reply_sent(channel, user_id, session_id)

send_content_parts
    ├─► _get_receive_for_send
    ├─► 若 meta.feishu_thread_id：thread_msg_id = feishu_message_id
    │     文本只用 post + _reply_in_thread（话题不支持 interactive / 流式卡）
    ├─► 普通文本 → _send_text
    │     含 GFM 表 → build_interactive_content_chunks → msg_type=interactive
    │     否则 post（md）
    ├─► IMAGE → _send_image（可 thread）
    └─► FILE/VIDEO/AUDIO → _send_file（opus/ogg 用 msg_type=audio）
```

流式（`streaming_enabled=True` 且非话题、非卡片动作）：

- `_before_consume_process` 预创建 CardKit 卡，挂到 `request._precreated_card`
- `on_streaming_start` / `on_streaming_delta` / `on_streaming_end` 更新/关闭卡
- 创建失败则 `on_streaming_end` 回退 `send_content_parts` 纯文本

### 4.4 proactive send（主动发送）

外部（如 cron）调用 `send(to_handle, text, meta)`：

```
send(to_handle, text, meta)
    ├─► enabled 检查
    ▼
_get_receive_for_send(to_handle, meta)
    ├─► 优先 meta.feishu_receive_id + feishu_receive_id_type
    ├─► _route_from_handle(to_handle)
    │       feishu:sw:{session_id}     → _load_receive_id
    │          miss 时：旧格式 feishu:open_id:... ；或 nick#后4 后缀匹配 store
    │       feishu:open_id:{open_id}   → open_id
    │       feishu:chat_id:{chat_id}   → chat_id
    │       oc_... / ou_...            → 推断类型
    └─► 再试 _load_receive_id(to_handle)
    ▼
_send_text(receive_id_type, receive_id, bot_prefix + text)
```

### 4.5 Token 管理

当前实现 **没有** `_get_tenant_access_token()`，也没有内存里的 expire_at 双检。

```
lark.Client（start 时 builder.app_id/app_secret/domain）
    └── 各 im/cardkit/contact API 由 SDK 内部刷新 tenant_access_token

_fetch_bot_open_id（显式取 token）
    TokenManager.get_self_tenant_token(self._client._config)
    GET {_sdk_domain()}/open-apis/bot/v3/info
    Authorization: Bearer {token}
    同时用响应 Date 计算 _clock_offset
```

`constants.FEISHU_TOKEN_REFRESH_BEFORE_SECONDS = 60` **仍定义，但 `channel.py` 未引用**。刷新策略以 lark-oapi 为准。

### 4.6 区域 / 自定义网关

`_sdk_domain()`：

- `domain` 以 `http` 开头 → 原样作为 base URL（测试 mock / 私有网关）
- `"lark"` → `lark.LARK_DOMAIN`
- 其它（含默认 `"feishu"`）→ `lark.FEISHU_DOMAIN`

`start()` 把该值传给 `lark.Client.builder().domain(...)`；`_run_ws_forever` 里 `lark.ws.Client(..., domain=self._sdk_domain())`。非法 `domain` 在 Pydantic `_check_domain` 会校验失败；`__init__` 对非 feishu/lark/http 回退 `"feishu"`。

## 五、方法调用关系图

### 初始化与生命周期

```
from_env / from_config
    └── __init__  （含 FeishuCardHandler(self)）

start()
    ├── _load_receive_id_store_from_disk()
    ├── 校验 lark-oapi、app_id、app_secret
    ├── _loop = get_running_loop()
    ├── _http_client = httpx.AsyncClient(timeout=30)
    ├── _client = lark.Client.builder()...domain(_sdk_domain())
    ├── Thread(_run_ws_forever).start()
    └── _fetch_bot_open_id()

_run_ws_forever
    └── 循环：建 _ws_loop → ws.Client → _connect + _ping_loop + health monitor
            失败/静默超时 → disconnect、close loop → 退避 wait → 再连

stop()
    ├── _closed=True, _stop_event.set()
    ├── _ws_loop.call_soon_threadsafe(stop)
    ├── _ws_thread.join(timeout=5)
    └── _http_client.aclose()

health_check()
    └── disabled / SDK 未初始化 / WS 线程未存活 → unhealthy
```

### 消息处理

```
_on_message_sync (WS 线程)
    └── run_coroutine_threadsafe(_on_message, _loop)

_on_message
    ├── 去重 / 忽略自身 bot
    ├── _get_user_name_by_open_id  → contact.v3.user.get
    ├── _parse_message_content
    │       └── _download_image_resource / _download_file_resource
    │               └── im.v1.message_resource.aget
    ├── _process_quoted_message
    │       └── _fetch_quoted_message_content → im.v1.message.aget
    ├── _check_group_mention
    ├── _add_reaction("Typing")  → im.v1.message_reaction.acreate
    └── _enqueue(native)

卡片点击 (WS 线程同步)
    FeishuCardHandler.handle_card_action
        └── tool_guard.handle → _enqueue_approval_command → _enqueue

send_content_parts
    ├── _get_receive_for_send
    │       ├── _route_from_handle
    │       └── _load_receive_id
    ├── _send_text / _reply_in_thread
    │       ├── _build_post_content + normalize_feishu_md
    │       ├── build_interactive_content_chunks
    │       └── _send_message  → im.v1.message.acreate
    ├── _send_image
    │       ├── _part_to_image_bytes → _fetch_bytes_from_url
    │       ├── _upload_image → im.v1.image.acreate
    │       └── _send_message 或 _reply_in_thread
    └── _send_file
            ├── _part_to_file_path_or_url
            ├── _upload_file → im.v1.file.acreate
            └── _send_message 或 _reply_in_thread
```

## 六、Session ID 与路由机制

### 实际 session_id（`resolve_session_id`）

源码 **不是** 把 `feishu:chat_id:<id>` 当作 `AgentRequest.session_id`。短 ID 长度由 `FEISHU_SESSION_ID_SUFFIX_LEN`（8）决定：

| 场景 | `resolve_session_id` 返回值 |
|------|-----------------------------|
| 群聊（`feishu_chat_type=="group"` 且有 chat_id） | `{app_id 后 4 位}_{chat_id 后 8 位}`（区分同群多 bot） |
| 单聊且有 sender_id | `open_id` 后 8 位 |
| 否则有 chat_id | `chat_id` 后 8 位 |
| 兜底 | `feishu:{sender_id}` |

`build_agent_request_from_native` **优先使用 payload 已有 `session_id`**（合并 native 时不能用 `sender_display` 如 `949#1d1a` 重算，否则短 ID 对不上 store）。

`user_id` 优先 `meta.feishu_sender_id`（真实 open_id）。话题消息会改写：

- `native["user_id"] = "thread:{thread_id 后 8 位}"`
- `meta["feishu_sender_id"]` 同上，使同话题成员共享 user 侧上下文

`share_session_in_group=True` 且无 `thread_id` 时，只把 `feishu_sender_id` 设为 `"group"`（群的 `session_id` 本来就按 chat_id 共享）。

类文档字符串里写的 `feishu:chat_id:<chat_id>` / `feishu:open_id:<open_id>` 是 **发送地址 `to_handle` 形态**，不是 `resolve_session_id` 的返回值。

### to_handle 格式（`_route_from_handle` / `to_handle_from_target`）

| to_handle 格式 | 含义 | 路由结果 |
|---------------|------|---------|
| `feishu:sw:{session_id}` | 短 session（cron/回复主路径） | `session_key`，再查 `_receive_id_store` |
| `feishu:open_id:{open_id}` | 明确用户 | `receive_id_type=open_id` |
| `feishu:chat_id:{chat_id}` | 明确群 | `receive_id_type=chat_id` |
| `ou_...` | open_id 形态 | `open_id` |
| `oc_...` | chat_id 形态 | `chat_id` |
| 其它 | 默认当 open_id | `open_id` |

`to_handle_from_target(user_id, session_id)`：有 session_id → `feishu:sw:{session_id}`，否则 `feishu:open_id:{user_id}`。

### receive_id 持久化

`_before_consume_process` 调用 `_save_receive_id(session_id, receive_id, receive_id_type)`：

- 以 `session_id` 为 key 存 `(receive_id_type, receive_id)`
- `open_id` 且与 session_id 不同时再按 open_id 存一份（cron 用 open_id 查）
- 写入 `_receive_id_store_path()`：优先 `{workspace_dir}/feishu_receive_ids.json`，否则 `get_config_path().parent / "feishu_receive_ids.json"`
- 磁盘格式 `[receive_id_type, receive_id]`；加载时兼容旧文件 `[receive_id, receive_id_type]`

群聊 receive：`chat_id` + `chat_id`；单聊：`open_id` + sender open_id。

## 七、消息类型处理矩阵

入站由 `_parse_message_content` 统一处理（主消息与引用消息共用）。`_MSG_TYPE_LABEL` 用于错误提示和 `[quoted ...]` 前缀。

| msg_type | 文本提取 | 媒体下载 | 出站 |
|----------|---------|---------|------|
| `text` | `extract_json_key(..., "text")`；去掉 bot mention `key` | 无 | post md；含表格则 interactive |
| `post` | `extract_post_text` | img → image_key；media → file_key | 同上 + 分开发图/文件 |
| `image` | 无（失败时 `[image: ...]`） | image_key/file_key | `msg_type=image` |
| `file` | 无 | file_key | `msg_type=file` |
| `media` | 无（label=`video`） | file_key，hint `video.mp4` | `file` |
| `audio` | 无 | file_key，hint `audio.opus` | 入站 `AudioContent`；出站 opus/ogg → `audio` |
| `interactive` | `extract_interactive_text` | 无 | 审批卡等走 card handler |
| 其它 | 调用方加 `[{msg_type}]` | 无 | — |

出站文本有 GFM 表时走 `utils.build_interactive_content_chunks`（每卡最多 `_MAX_TABLES_PER_CARD=5` 张表）。**话题回复强制 post，不走 interactive 分块。**

群聊 @：内容含 `@_all`，或 `mentions[].id.open_id == _bot_open_id`，则 `meta["bot_mentioned"]=True`。`require_mention=True` 时基类 `_check_group_mention` 要求 `bot_mentioned` 或 `has_bot_command`。

## 八、关键常量

均定义于 `src/qwenpaw/app/channels/feishu/constants.py`，除非另注。

| 常量 | 值 | 说明 |
|------|---|------|
| `FEISHU_TOKEN_REFRESH_BEFORE_SECONDS` | 60 | **未在 channel.py 使用**；历史/SDK 侧参考 |
| `FEISHU_FILE_MAX_BYTES` | 30×1024×1024 | 上传文件大小上限 |
| `FEISHU_PROCESSED_IDS_MAX` | 1000 | message_id 去重缓存上限 |
| `FEISHU_NICKNAME_CACHE_MAX` | 500 | 昵称缓存上限 |
| `FEISHU_SESSION_ID_SUFFIX_LEN` | 8 | `short_session_id_from_full_id` 截取长度 |
| `FEISHU_USER_NAME_FETCH_TIMEOUT` | 2 | **channel.py 未引用**；卡片 handle 里姓名查询 `timeout=2` 为字面量 |
| `FEISHU_STALE_MSG_THRESHOLD_MS` | 20_000 | 丢弃过期 WS 重试 |
| `FEISHU_WS_INITIAL_RETRY_DELAY` | 1.0 | 重连初始等待（秒） |
| `FEISHU_WS_MAX_RETRY_DELAY` | 60.0 | 重连等待上限 |
| `FEISHU_WS_BACKOFF_FACTOR` | 2 | 未真正连上时延迟倍增 |
| `FEISHU_WS_RECV_TIMEOUT` | 240 | 无数据（含 pong）则强制重连；SDK ping ~120s |
| `FEISHU_STREAM_MIN_INTERVAL_S` | 0.15 | 流式 CardKit 更新最小间隔 |
| `FEISHU_STREAM_ELEMENT_ID` | `"streaming_content"` | 流式卡 markdown 元素 ID |
| `_MAX_TABLES_PER_CARD` | 5 | 在 `utils.py`，非 constants |

## 九、配置模型与环境变量

`FeishuConfig(BaseChannelConfig)` 字段：`app_id`、`app_secret`、`encrypt_key`、`verification_token`、`media_dir`、`domain`（默认 `"feishu"`）、`streaming_enabled`、`share_session_in_group`，以及基类的 `enabled`、`bot_prefix`、展示开关、`dm_policy`、`group_policy`、`allow_from`、`deny_message`、`require_mention`、`no_text_debounce`、`access_control_dm`/`access_control_group`、`dm_disabled`/`group_disabled`。

`from_env` 仍可用：`FEISHU_CHANNEL_ENABLED`、`FEISHU_APP_ID`、`FEISHU_APP_SECRET`、`FEISHU_BOT_PREFIX`、`FEISHU_ENCRYPT_KEY`、`FEISHU_VERIFICATION_TOKEN`、`FEISHU_MEDIA_DIR`、`FEISHU_DM_POLICY`、`FEISHU_GROUP_POLICY`、`FEISHU_ALLOW_FROM`、`FEISHU_DENY_MESSAGE`、`FEISHU_REQUIRE_MENTION`、`FEISHU_DOMAIN`、`FEISHU_STREAMING_ENABLED`、`FEISHU_SHARE_SESSION_IN_GROUP`。生产路径以 `from_config` + `config.json` 的 `feishu` 段为主。

## 十、交互卡片（2.x）

`channel.py` 导入 `from .cards import FeishuCardHandler`。

lark-oapi 每个 dispatcher **只能注册一个** `p2.card.action.trigger`，因此出站 `metadata.message_type` → `render`、入站按钮 `value.type` → `handle` 都集中在 `FeishuCardHandler`。

当前内置一种：`cards/tool_guard.py`

| 元数据 | 值 |
|--------|-----|
| `NAME` / `MESSAGE_TYPE` / `ACTION_TYPE` | `"tool_guard_approval"` |

- 出站：`on_event_message_completed` → `try_send_card_for_event` → `render` → `_get_receive_for_send` + `_send_message(..., "interactive", card_json)`
- 入站：按钮 value 带 `session_ctx`；`handle` 向队列注入 `/approval {approve\|deny} {request_id}`（`meta.from_card_action=True`），并同步返回 toast + 替换后的卡片 JSON
- 跨实例：`handle_card_action` 比较 `header.app_id` 与 `channel.app_id`

包根目录的 `card_handler.py` / `card_templates.py` 是同主题旧实现（`TOOL_GUARD_ACTION_TYPE` 等），**运行时频道不走这两文件**。

## 十一、继承关系

```
BaseChannel                          # uses_manager_queue=True
    └── FeishuChannel
            覆盖 from_config / from_env
            覆盖 build_agent_request_from_native
            覆盖 merge_native_items
            覆盖 to_handle_from_target
            覆盖 get_to_handle_from_request
            覆盖 get_on_reply_sent_args
            覆盖 send_content_parts
            覆盖 send / start / stop
            覆盖 resolve_session_id
            覆盖 _before_consume_process
            覆盖 on_event_message_completed
            覆盖 on_streaming_start / delta / end
            覆盖 _on_process_completed
            覆盖 health_check
            未覆盖 _run_process_loop（使用基类）
```

## 十二、注意事项

1. **双 event loop**：主循环处理 `_on_message` 与所有 SDK 异步 API；WS 线程另有 `_ws_loop`。导入时用 `_EventLoopProxy` 把 `lark_oapi.ws.client.loop` 指到「当前线程正在跑的 loop」，避免多 Feishu 实例互相覆盖模块级 loop。
2. **pkg_resources 兼容**：`channel.py` 顶部在导入 lark-oapi 期间为 setuptools≥82 补 `declare_namespace` shim，导入结束后还原。
3. **路径安全**：下载文件名用 `Path(filename_hint).name`；image_key 过滤为 alnum/`-_.`。
4. **去重窗口**：超过 `FEISHU_PROCESSED_IDS_MAX`（1000）删除最旧条目；另有 20s 过期重试过滤。
5. **DONE / Typing 反应**：入站加 `Typing`；本轮 `_on_process_completed` 对 `_last_sent_message_id` 加 `DONE`。
6. **话题限制**：流式卡与 interactive 表格卡在 thread 中不使用，回复走 `ReplyMessageRequest.reply_in_thread=True`。
7. **卡片按钮**：需在飞书开放平台订阅 `card.action.trigger`；未配置时按钮无响应（卡上有文档链到 channels#feishu-callback-config）。
8. **依赖**：`start()` 时若未安装 `lark-oapi` 抛 `ChannelError`，提示 `pip install lark-oapi`。
