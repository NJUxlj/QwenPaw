# QwenPaw 记忆模块结构与调用链

- **版本**：QwenPaw v2.2.1
- **日期**：2026-09-15
- **对照源码**：`src/qwenpaw/agents/memory/`、`src/qwenpaw/agents/context/`、`src/qwenpaw/runtime/builder.py`、`src/qwenpaw/agents/react_agent.py`、`src/qwenpaw/config/config.py`

本文档描述 **2.x 三层记忆栈**：工作上下文（AgentState 窗口）+ Scroll 逐字历史（`history.db`）+ 可插拔长期知识库（默认 ReMeLight）。旧文档中的 `src/copaw/agents/memory/`、`CoPawInMemoryMemory`、`MemoryManager(ReMeFb)`、`COMPRESSED` mark 属于 **1.x 机制**，源码中已删除。

---

## 一、三层记忆（2.x）

| 层 | 职责 | 持久化 | 谁驱动 |
|---|---|---|---|
| **工作上下文** | 当前推理窗口：`agent.state.context` | `workspaces/<agent_id>/sessions/{session_id}.json` 里的 `AgentState` | AgentScope + `QwenPawAgent` |
| **Scroll 逐字历史** | 窗口外的完整对话仍可按 `seq` 召回；压缩时驱逐中间段，不丢原文 | `{workspace_dir}/history.db`（默认文件名） | `ScrollContextManager`（`light_context_config.strategy == "scroll"`，**默认**） |
| **长期知识库** | 从对话提炼事实/日记/digest，语义检索 | 后端自有目录或远程服务 | `BaseMemoryManager` 实现（默认 `remelight`） |

职责边界（源码注释，`reme_config.py`）：

- **Scroll** `recall_history(op="search")`：搜 **原始对话日志**（`conversation_history`）。
- **ReMe** `search`：只索引 **提炼后的 Markdown**（`daily_dir` + `digest_dir`），避免两套索引重复。

`strategy == "native"` 时不接线 Scroll：压缩走 AgentScope 原生 `compress_context()`，被驱逐的消息可由 `QwenPawOffloader.offload_context()` 追加到 `dialog/{YYYY-MM-DD}.jsonl`。Scroll 接线失败（依赖缺失、无 governor 且未开 unsandboxed）会 **降级到 native**，完整历史留在窗口内。

---

## 二、模块树

```
src/qwenpaw/agents/memory/
├── __init__.py                      # 导出核心类；proactive 符号 __getattr__ 懒加载
├── base_memory_manager.py           # BaseMemoryManager + MemoryBackendRegistry
├── reme_light_memory_manager.py     # 内置后端 remelight（嵌入 ReMe 0.4.1.11）
├── dummy.py                         # 内置后端 none（NoopMemoryManager）
├── action_provider.py               # MemoryActionProvider 协议（list_actions / run_action）
├── agent_md_manager.py              # Console 读写工作区 / memory / digest 的 .md
├── prompts.py                       # ReMe 系统提示里的长期记忆指引
├── reme_config.py                   # 嵌入式 ReMe Application 配置（jobs/components）
├── reme_embedding.py                # Embedding 测试 / 应用 / reindex / undo
├── reme_reranker.py                 # memory_search 可选 rerank
├── reme_inbox.py                    # auto_memory / auto_dream / daily_paper / auto_fin 收件箱
├── embedding_model.py               # AgentScope Embedding 构造与连通性测试
└── proactive/                       # 会话空闲后主动找用户（与 ReMe job "proactive" 不同）
    ├── __init__.py
    ├── proactive_types.py
    ├── proactive_trigger.py         # enable/disable + 30s 轮询
    ├── proactive_responder.py       # generate_proactive_response()
    ├── proactive_prompts.py
    └── proactive_utils.py

src/qwenpaw/agents/context/
├── base.py                          # ContextManager Protocol（on_save / compress / recover）
├── __init__.py                      # build_scroll_components()
├── types.py
├── overflow_recovery.py
├── visual_compression/              # 请求级图文压缩（非记忆栈核心）
└── scroll/
    ├── manager.py                   # ScrollContextManager
    ├── history.py                   # HistoryStore（SQLite conversation_history）
    ├── eviction_index.py            # EvictionIndex（窗口内「已驱逐地图」）
    ├── continuation_summary.py      # ContinuationSummary（状态缓存，不是原文替代）
    ├── recall_tool.py               # recall_history（expand / search / recall_tool / days_between）
    ├── repl.py                      # recall_history_python（沙箱 Python REPL）
    ├── memoryspace.py               # 只读 ATTACH hist 的 SQLite 工作面
    ├── serialize.py                 # Msg → LogEntry
    ├── prompt.py                    # Scroll 系统提示（headline / 召回纪律）
    ├── sync.py                      # 历史会话回填 history.db
    └── _as_internals.py             # 调用 AgentScope 内部 split / prepare_model_input

src/qwenpaw/memory/__init__.py       # 插件应从此包导入合约（不要从 agents 内部包）

plugins/memory/                      # 仓库内插件源码，不是 core 内置后端
├── adbpg/                           # backend_id = "adbpg"
└── powercontext/                    # backend_id = "powercontext"
```

已删除（1.x）：`copaw_memory.py` / `CoPawInMemoryMemory`、`memory_manager.py` / `MemoryManager(ReMeFb)`、`MemoryCompactionHook`、独立工具文件 `agents/tools/memory_search.py`。

---

## 三、核心类职责

| 类 | 位置 | 职责 |
|---|---|---|
| `BaseMemoryManager` | `memory/base_memory_manager.py` | 长期记忆抽象：`start` / `close` / `memory_search` / `auto_memory`；自动检索、后台 `submit_auto_memory` 队列、`build_middlewares()` → `MemoryMiddleware` |
| `MemoryBackendRegistry` | 同上，全局 `memory_registry` | `@register`（core）或 `register_backend`（插件）；`create()` 与 unload 互斥 |
| `ReMeLightMemoryManager` | `reme_light_memory_manager.py` | 内置默认后端 `remelight`：嵌入 `reme.ReMe`，跑 search / auto_memory / auto_dream 等 job |
| `NoopMemoryManager` | `dummy.py` | 后端 `none`：`enabled=False`，无工具、无中间件、interval=0 |
| `MemoryActionProvider` | `action_provider.py` | `list_actions()` / `run_action()`；ReMe 与 Console `/reme`、reindex API 共用 |
| `AgentMdManager` | `agent_md_manager.py` | Console 路由读写工作区根目录 `*.md`，以及 `daily_dir` / `digest_dir` 下的记忆 Markdown |
| `MemoryMiddleware` | `agents/middlewares.py` | 注入记忆指引、每轮 `auto_memory_search`、回复后按 interval 提交 `auto_memory`、压缩后 flush |
| `ContextManager` | `context/base.py` | 协议：`on_save` / `compress` / `recover_from_context_overflow` |
| `ScrollContextManager` | `context/scroll/manager.py` | 写穿 `history.db`、阈值驱逐、EvictionIndex、continuation summary、工具结果 fold |
| `HistoryStore` | `context/scroll/history.py` | SQLite WAL：`conversation_history` + 可选 FTS5 |
| `EvictionIndex` | `context/scroll/eviction_index.py` | 窗口内分层地图（每层最多 10 block），叶子是带 `⟦headline⟧` 的 turn + seq 区间 |
| `QwenPawAgent` | `react_agent.py` | `_save_to_context` → `on_save`；`_compress_context_impl` → `compress`；session 里存 `scroll` 书签 |
| `AgentBuilder` | `runtime/builder.py` | 接线 Scroll 组件、注册 recall 工具、从 Workspace 取 `memory_manager`、组装 middleware |
| `Workspace` | `app/workspace/workspace.py` | 按 `memory_manager_backend` 构造并 `start()` 长期记忆服务 |

---

## 四、长期记忆后端

### 4.1 选择方式

每个 Agent 的 `agent.json` → `running.memory_manager_backend`（默认 `"remelight"`）。插件私有配置放在 `running.memory_backend_configs[<backend_id>]`，由插件自己的 Pydantic schema 校验；core **不**把 ADBPG / PowerContext 字段写进 `ReMeLightMemoryConfig`。

构造入口（`Workspace._register_services`）：

```
Workspace.start
  └── ServiceManager 注册 name="memory_manager"
        ├── get_memory_manager_backend(_effective_memory_backend_id)
        └── create_memory_manager_backend(backend_id, MemoryBackendContext)
              └── memory_registry.create() → factory(context=...)
                    └── await manager.start()
```

`MemoryBackendContext` 字段：`agent_id`、`working_dir`（该 Agent 工作区）、`host_working_dir`（`WORKING_DIR`）、`backend_config`、`language`、`token_estimate_divisor`。

若所选插件未注册：`Workspace._effective_memory_backend_id()` **运行时回落到 `remelight`**（打 warning，**不改写**用户配置）。`remelight` 本身不可用则抛 `MemoryBackendUnavailableError`（fatal）。记忆服务 `optional=True`：ReMe 导入失败时 Workspace 仍可启动，只是没有长期记忆。

### 4.2 内置 core

| backend_id | 类 | 说明 |
|---|---|---|
| `remelight` | `ReMeLightMemoryManager` | `@memory_registry.register("remelight")`。固定依赖 `reme-ai==0.4.1.11`。工作区作 ReMe vault。 |
| `none` | `NoopMemoryManager` | 关闭长期记忆：无 `memory_search`、无 `MemoryMiddleware`、`auto_memory` 丢弃。 |

ReMeLight 把 QwenPaw 当前模型注入 ReMe 的 `as_llm` 组件（`_update_qwenpaw_model`）。无激活模型时仍可启动（BM25 reindex 等不依赖 LLM 的 job）。

ReMe 监视目录只有 `daily_dir` 与 `digest_dir`（默认 `memory/`、`digest/`），后缀 `md`。对话原文检索归 Scroll。

主要 job（`reme_config.build_reme_app_config`）：`search`、`auto_memory`、`auto_dream`、`daily_paper`、`auto_fin`、`proactive`（兴趣主题，不是空闲主动聊）、`reindex`、`graph_snapshot`、`status`，以及 vault 读写类 job。QwenPaw 关掉插件自带的 `daily_paper_cron` / `auto_fin_cron`（`enable_serve=False`），改由 `CronManager._register_memory_jobs()` 调 `list_cron_jobs()`。

`memory_search`：跑 ReMe `search`；若 `reranker_config.enabled`，按 `candidate_multiplier` 多取再 rerank 截断。

`auto_memory`：消息 `model_dump` 后交给 ReMe job `auto_memory`；`session_id` 经 `_to_reme_session_id()` SHA-256 哈希（`qpsid_sha256_...`），避免当文件名时的大小写/Unicode 碰撞。空 `session_id` 直接跳过。

### 4.3 插件后端（非 core）

仓库 `plugins/memory/` 提供两个可安装插件，**不是** QwenPaw 内置实现：

| 插件 id | backend_id | 入口 |
|---|---|---|
| `memory-adbpg` | `adbpg` | `plugins/memory/adbpg/plugin.py` → `ADBPGMemoryPlugin.register()` |
| `memory-powercontext` | `powercontext` | `plugins/memory/powercontext/plugin.py` → `PowerContextMemoryPlugin.register()` |

加载方式：

1. 安装 type=`memory` 的插件（`plugin.json` 的 `meta.memory_backends`）。
2. 插件 `register(api)` 调用 `PluginApi.register_memory_backend(...)`。
3. 该方法写入 `qwenpaw.memory.memory_registry.register_backend(plugin_id=..., backend_id=..., factory=..., config_schema=..., metadata=...)`。
4. Agent 把 `memory_manager_backend` 设为 `adbpg` 或 `powercontext`，配置进 `memory_backend_configs`。
5. 卸载插件前 `MemoryBackendRegistry.begin_owner_unload()`：若仍有该 owner 的 live instance / 构造中 / 选中中的 agent，拒绝卸载。

插件应 `from qwenpaw.memory import BaseMemoryManager, MemoryBackendContext, ...`，不要依赖 `qwenpaw.agents.memory` 内部路径。

---

## 五、人格文件与 AgentMdManager

仍由 **memory 包内的 `AgentMdManager`** 管 Markdown 文件的 **Console CRUD**（`app/routers/workspace.py` 每次请求 `AgentMdManager(workspace_dir, agent_id=...)`），**不是**对话窗口本身。

| 位置 | 谁读进系统提示 | 谁写入 |
|---|---|---|
| `AGENTS.md` / `SOUL.md` / `PROFILE.md`（工作区根） | `WorkspacePromptFilesContributor`（`runtime/prompt_contributors.py`）；`AGENTS.md` 会剥 `<!-- heartbeat -->` / `<!-- memory -->` 段 | 用户或 Agent 文件工具；Console 经 `AgentMdManager.read/write_working_md` |
| `MEMORY.md`（工作区根） | **不**自动注入系统提示。ReMe 指引把它描述为「核心长期记忆」，Agent 用 `read_file` / `write_file` 维护 | 同上；也在 checkpoint 的 `CHECKPOINT_STATE_FILES` 里 |
| `{daily_dir}/`、`{digest_dir}/` | 不注入全文；靠 `memory_search` + `read_file` 渐进展开 | ReMe `auto_memory` / `auto_dream` 等 job；Console 经 `list/read/write_memory_md` |

`AgentMdManager` 的 `memory_dir` / `digest_dir` 来自 `reme_light_memory_config.daily_dir` / `digest_dir`（无 `agent_id` 时退回 `memory` / `digest`）。路径会做 `..` 拒绝与 `resolve().relative_to()` 防逃逸。

长期记忆 **指引文本** 由 `ReMeLightMemoryManager.get_memory_prompt()` → `build_memory_guidance_prompt()` 生成，运行时经 `MemoryMiddleware.on_system_prompt()` 拼到系统提示。`WorkspacePromptFilesContributor` 也会尝试 `extras["memory_manager"]`，但当前 `AgentBuilder.build_prompt()` **没有**把 `memory_manager` 放进 extras，因此 AGENTS.md 路径只剥旧 `<!-- memory -->` 段，真正的 ReMe 指引以 Middleware 为准。

Scroll 另有 `ScrollContextContributor` → `build_scroll_system_prompt()`（headline 协议、`recall_history` 用法），与 ReMe 指引独立。

---

## 六、1.x COMPRESSED / CoPawInMemoryMemory（已删除）

1.x 用 `CoPawInMemoryMemory.get_memory(exclude_mark=COMPRESSED, prepend_summary=True)` 把旧消息打标、摘要塞回窗口。

2.x：

- 工作窗口是 **`AgentState.context`**，没有 mark 系统。
- 超阈值时 **Scroll 把中间段驱逐进 `history.db` + EvictionIndex**，原文可 `recall_history` 取回。摘要只是 `ContinuationSummary` 状态缓存，「失败则保留上一份有效摘要，原文仍在 DB」。
- `parse_legacy_memory_state()`（`app/chats/utils.py`）与 `QwenPawAgent.load_state_dict()` 仍能读 1.x session 的 `{"memory": {"content": [[msg, marks], ...], "_compressed_summary": "..."}}`，**丢掉 marks**（`HINT` / `COMPRESSED` 在新 schema 不可达），消息迁到 `AgentState.context`。

结论：**1.x 的 COMPRESSED mark / CoPawInMemoryMemory 已删除，2.x 由 Scroll（默认）或 native AgentScope compact 替代。**

---

## 七、整体架构

```
Workspace
  ├── memory_manager: BaseMemoryManager      # remelight | none | 插件
  ├── session: SafeJSONSession               # AgentState + scroll 书签
  └── CronManager._register_memory_jobs()    # list_cron_jobs()

Runtime / AgentBuilder.build_agent()
  ├── build_scroll_components()              # strategy=scroll 时
  │     HistoryStore + ScrollContextManager
  │     + recall_history + recall_history_python
  ├── memory_manager.list_memory_tools()     # 通常 [memory_search]
  ├── memory_manager.build_middlewares()     # [MemoryMiddleware]
  └── QwenPawAgent(context_manager=ScrollContextManager | None)
```

---

## 八、调用链（真实函数名）

### 8.1 初始化

```
Workspace._register_services
  └── create_memory_manager_backend(backend, MemoryBackendContext)
        └── ReMeLightMemoryManager.__init__
              └── _initialize_reme()          # reme.ReMe(**get_reme_app_config(...))
        └── await ReMeLightMemoryManager.start()
              ├── _update_qwenpaw_model()     # update_component("as_llm", ...)
              └── _reme.start()

AgentBuilder.build_agent
  ├── _build_scroll_components → build_scroll_components()
  │     ├── HistoryStore(workspace_dir / scroll_config.db_filename)
  │     ├── ScrollContextManager(...)
  │     ├── make_recall_history(...)
  │     └── make_recall_history_python(...)
  ├── _append_scroll_recall_tools()           # 始终挂 recall_history；REPL 视沙箱而定
  ├── _get_memory_manager(ctx) → workspace.memory_manager
  ├── toolkit += memory_manager.list_memory_tools()
  ├── _build_middlewares += memory_manager.build_middlewares()
  └── QwenPawAgent(..., context_manager=scroll.context_manager)
        └── load_state_dict(session)          # 恢复 context + scroll.to_dict()
```

### 8.2 用户一轮对话：写入 Scroll、检索 ReMe、compact/evict

```
用户消息
  └── Runtime / QwenPawAgent.reply
        │
        ├── MemoryMiddleware.on_system_prompt
        │     └── memory_manager.get_memory_prompt()
        │
        ├── 每步推理前 QwenPawAgent.compress_context
        │     └── middleware on_compress_context
        │           MemoryMiddleware.on_compress_context
        │             1. snapshot pending turns（压缩可能丢掉窗口内原文）
        │             2. next_handler → QwenPawAgent._compress_context_impl
        │                  └── ScrollContextManager.compress   # 或 super() native
        │             3. 若确实压缩：_flush_auto_memory(trigger="compact")
        │
        ├── MemoryMiddleware.on_model_call
        │     └── 该 user turn 尚未检索过：
        │           memory_manager.auto_memory_search(query_msg, session_id=...)
        │             ├── _build_query()           # 最近一条 user 文本，最多 50 字符
        │             └── memory_search(...)       # ReMe job "search"（或插件实现）
        │           命中则合成 AssistantMsg（假 memory_search 工具回合）
        │           注入本轮 model input（不污染后续 turn 缓存）
        │
        ├── 模型输出块写入窗口
        │     QwenPawAgent._save_to_context
        │       ├── AgentScope 追加到 state.context
        │       └── ScrollContextManager.on_save → _persist_new
        │             HistoryStore.append / update_entry
        │             （跳过 synthetic 占位；去掉 auto_memory_search 合成块）
        │
        └── MemoryMiddleware.on_reply（整轮结束后）
              ├── 记录 user turn_marker 到 pending
              └── 若 len(pending) >= get_auto_memory_interval()
                    _flush_auto_memory(trigger="periodic")
                      └── submit_auto_memory(messages, session_id=...)
                            后台 _auto_memory_worker → auto_memory()
                              └── ReMe job "auto_memory" 写 daily note
```

`auto_memory_search` 默认 **关**（`auto_memory_search_config.enabled=False`）。打开后每轮用户问题会先检索长期库。cron / heartbeat / `portability_adaptation` 请求 `_is_automation_request` 为真，跳过检索与写入。

### 8.3 Scroll `compress` 压力管道（`ScrollContextManager.compress`）

```
1 persist      _persist_guarded_async → 整窗写穿 history.db
               写失败则不驱逐（避免 seq 指向不存在的行）
2 trigger      count_tokens；tokens <= trigger_ratio * context_size → 返回
               should_compress()：恰好等于阈值仍保留 live
3 pre-fold     自动压力下批量折叠已完成 turn 的大工具结果（保护最近 5 条、
               活跃 turn、≤200 字符）；手动 /compact 跳过此步
4 split        AgentScope split_for_compression；保住活跃 user turn 整段
5 summarize    _update_continuation_summary（失败保留旧摘要）
6 add_eviction EvictionIndex.add_eviction；_rebuild_context = [占位 UserMsg] + tail
7 live-fold    仍超压：折叠其余已完成工具结果为 recall stub
8 think-fold   省略已被模型成功读过的活跃 ThinkingBlock（只改请求，不改 DB）
9 active-fold  仍超 hard_limit：折叠已 acknowledge 的活跃工具结果
溢出           recover_from_context_overflow：trigger_ratio≈1e-6 再 compress 一次
```

Fold stub 示例：`[scroll folded] ... recall_history(op="recall_tool", tool_call_id=...)`。

`QwenPawAgent.close()`：`purge_old(history_retention_days)`（默认 30，0=永不删）然后 `HistoryStore.close()`。

### 8.4 Agent 主动召回逐字历史

```
模型调用 recall_history(op, ...)     # recall_tool.py::make_recall_history
  ├── expand(lo, hi)                 # MemorySpace.expand
  ├── search(query, k, 日期/session 过滤)
  ├── recall_tool(tool_call_id)
  └── days_between(...)
可选 recall_history_python           # 沙箱 REPL；需 governor 或
                                     # QWENPAW_ALLOW_UNSANDBOXED_RECALL + allow_unsandboxed
```

`recall_history` / `recall_history_python` 自己的 turn **写入 DB 但不进 FTS**（`HistoryStore._RECALL_TOOL_NAMES`），避免搜索命中自己的查询文本。

### 8.5 系统命令（`CommandHandler`）

| 命令 | 函数 | 行为 |
|---|---|---|
| `/compact [hint]` | `_process_compact` | 强制 `compress`（`trigger_ratio=1e-6`）；Scroll 驱逐进 DB；`submit_auto_memory(trigger="compact")`。`manual_compact_memory_by_handler()` 避免 Middleware 再 flush 一次 |
| `/new` | `_process_new` | `submit_auto_memory(trigger="new")`；`_persist_and_clear`；清空 continuation / auto-memory turn state |
| `/clear` | `_process_clear` | 可选 offload 后 `state.context.clear()`；**不**抽长期记忆 |
| `/compact_str` | `_process_compact_str` | Scroll：`ContinuationSummary.render()`；native：AgentScope summary |
| `/history` | `_process_history` | 窗口 token 统计 |
| `/auto_memory_status` | `_process_auto_memory_status` | `list_auto_memory_tasks()` |
| `/reme <action>` | `_process_reme` | `MemoryActionProvider.run_action`；聊天面只开放 `status/search/proactive/auto_memory/auto_dream/daily_paper/auto_fin` |
| `/proactive [minutes\|on\|off]` | `_process_proactive` | 见下一节 |

已无 `/await_summary`、`/compact` 不再调用 `memory_manager.compact_memory()`。

`/new`、`/clear` 在 `offloader` 存在时会 `offload_context`。Scroll 下 `offload_dialog` **默认 False**：`history.db` 已是全文；打开后驱逐时额外写 `dialog/*.jsonl`。

### 8.6 Proactive（空闲主动聊）

与 ReMe job `proactive`（暴露 dream 兴趣主题）不是同一条路径。

```
/proactive [idle_minutes]
  └── enable_proactive_for_session(session_id, idle_minutes, workspace)
        写入 proactive_configs / proactive_workspaces
        asyncio.create_task(_run_trigger_loop)
          └── proactive_trigger_loop          # 每 30s
                ├── is_agent_busy(task_tracker)
                ├── get_last_message_ts()
                └── _should_trigger_proactive  # 空闲 >= idle_minutes
                      └── _handle_proactive_trigger
                            └── generate_proactive_response(workspace)
                                  ├── build_proactive_memory_context
                                  │     近会话消息 + 可选截屏
                                  ├── _extract_tasks_from_memory
                                  └── _execute_query / _generate_final_message
```

`/proactive off` → `disable_proactive_for_session`（cancel task、清 dict）。冷却：上次尝试 60s 内不重复；上一条已是 `[PROACTIVE]` 且用户未回则跳过。

---

## 九、配置字段

路径：`AgentProfileConfig.running`（`config/config.py`）。

### 长期记忆

| 字段 | 默认 | 含义 |
|---|---|---|
| `memory_manager_backend` | `"remelight"` | `remelight` / `none` / 已注册插件 id |
| `memory_backend_configs` | `{}` | 插件不透明配置，key 为 backend_id |
| `reme_light_memory_config.*` | 见下 | 仅 ReMeLight |

`ReMeLightMemoryConfig` 要点：

| 字段 | 默认 |
|---|---|
| `daily_dir` / `digest_dir` | `"memory"` / `"digest"` |
| `metadata_dir` / `session_dir` / `mem_session_dir` | `"mem_metadata"` / `"mem_session"` / `"mem_agent"` |
| `auto_memory_interval` | `5`（每 N 个 user turn；`None` 或 `<=0` 关闭周期写入） |
| `auto_memory_search_config.enabled` | `False` |
| `auto_memory_search_config.max_results` | `2` |
| `memory_search_enabled` | `True` |
| `embedding_model_config` | `backend=openai`，`model_name=""` 则关闭向量、仅 BM25 |
| `reranker_config.enabled` | `False` |
| `dream_cron_enabled` / `dream_cron` | `True` / `"0 23 * * *"` |
| `daily_paper_cron_enabled` / `auto_fin_cron_enabled` | `False` |

Embedding：`openai` / `dashscope` / `dashscope_multimodal` / `gemini` / `ollama`。OpenAI 兼容与 Gemini 需要 `api_key`；`ollama` 可只配 `base_url`（作 host）。

1.x 环境变量 `COPAW_MEMORY_COMPACT_*`、`MEMORY_STORE_BACKEND=chroma|local`、`EMBEDDING_API_KEY` **不再**驱动 2.x 记忆栈。压缩阈值在 `light_context_config`；Embedding 在 `reme_light_memory_config.embedding_model_config`。

### Scroll / 工作上下文

| 字段 | 默认 |
|---|---|
| `light_context_config.strategy` | `"scroll"`（另可选 `"native"`） |
| `context_compact_config.enabled` | `True` |
| `context_compact_config.compact_threshold_ratio` | `0.8` |
| `context_compact_config.reserve_threshold_ratio` | `0.1` |
| `scroll_config.db_filename` | `"history.db"` |
| `scroll_config.history_retention_days` | `30`（`0` = 永不 purge） |
| `scroll_config.repl_timeout_s` | `300` |
| `scroll_config.allow_unsandboxed` | `False`（还需环境变量 `QWENPAW_ALLOW_UNSANDBOXED_RECALL`） |
| `scroll_config.offload_dialog` | `False` |
| `token_count_estimate_divisor` | `4`（自动检索合成消息的 token 估算） |

`AgentBuilder._build_context_config` 把它映射为 AgentScope `ContextConfig(trigger_ratio, reserve_ratio, ...)`。

---

## 十、磁盘布局

工作区根：`WORKING_DIR`（`QWENPAW_WORKING_DIR` → 若存在则沿用 `~/.copaw` → 否则 `~/.qwenpaw`）。每个 Agent：`{WORKING_DIR}/workspaces/<agent_id>/`。

```
workspaces/<agent_id>/
├── AGENTS.md, SOUL.md, PROFILE.md, MEMORY.md   # 人格 / 手写长期记忆
├── history.db                                  # Scroll 逐字历史（+ -wal/-shm）
├── sessions/{session_id}.json                  # AgentState + state_dict["scroll"]
├── memory/                                     # ReMe daily_dir（日记、session 笔记）
│   └── YYYY-MM-DD.md 及 YYYY-MM-DD/{topic}.md
├── digest/                                     # ReMe digest_dir（dream 抽象节点）
├── mem_metadata/  mem_session/  mem_agent/     # ReMe 内部状态
├── dialog/YYYY-MM-DD.jsonl                     # 仅 native compact 或 offload_dialog=true
└── .scroll/                                    # recall REPL scratch
```

`HistoryStore` 表 `conversation_history`：`seq` PK、`session_id`、`agent_id`、`kind`、`content`、`headline`、`dedup_key` 等。同一 workspace 的 DB 跨会话共享，行用 `session_id` + `agent_id` 区分。

---

## 十一、设计要点

1. **三层分离**：窗口要短、原文要全、知识要提炼。Scroll 不替代 ReMe，ReMe 不索引对话 JSONL。
2. **写穿先于驱逐**：`compress` 第一步持久化；磁盘失败则保持 live。
3. **摘要不可当作真相**：`ContinuationSummary` 带 `covered_seq`；purge 后失效会从上下文拿掉。
4. **后端可插拔**：core 只有 `remelight` 与 `none`；ADBPG / PowerContext 走插件 registry。
5. **合成检索回合可剥除**：`BaseMemoryManager.message_without_auto_memory_search` 避免假工具回合进入 auto_memory 与 history.db。
6. **Session 升级**：1.x `memory.content` + `_compressed_summary` 在 `load_state_dict` 当场迁到 `AgentState`。

---

## 附录：1.x → 2.x 记忆栈映射

| 1.x | 2.x |
|---|---|
| `src/copaw/agents/memory/` | `src/qwenpaw/agents/memory/` + `src/qwenpaw/agents/context/scroll/` |
| `CoPawInMemoryMemory`（窗口 + COMPRESSED mark + `_compressed_summary`） | `AgentState.context`；压缩改由 Scroll / native；1.x session 仅兼容加载 |
| `MemoryCompactionHook` + `memory_manager.compact_memory()` | `QwenPawAgent._compress_context_impl` → `ScrollContextManager.compress`（或 AgentScope native） |
| 窗口内摘要顶替旧消息 | 原文进 `history.db`；窗口放 `EvictionIndex` + 可选 `ContinuationSummary` |
| `MemoryManager(ReMeFb)` 单例 | `Workspace.memory_manager`：`ReMeLightMemoryManager` 或插件 / `none` |
| chroma / local `MEMORY_STORE_BACKEND` | ReMe `LocalFileStore`（BM25 + 可选 embedding）；或插件远程库 |
| 独立 `memory_search` 工具模块 | `BaseMemoryManager.memory_search` 经 `list_memory_tools()` 注册 |
| `AgentMdManager` + `AGENT_MD_MANAGER` 全局单例 | 仍是 `AgentMdManager`，按请求绑定 `workspace_dir` + `agent_id` |
| `AGENTS.md` 等人格文件 | 仍在工作区根；系统提示改由 `prompt_contributors.WorkspacePromptFilesContributor` 加载 |
| `/compact` 调 ReMe 摘要 | `/compact` 调 Scroll/native 压缩；并可 `submit_auto_memory(trigger="compact")` |
| `/new` `add_async_summary_task` | `/new` `submit_auto_memory(trigger="new")` |
| `/await_summary`、`/compact_str` 读 COMPRESSED 摘要 | 无 await；`/compact_str` 读 Scroll continuation 或 native summary |
| `COPAW_MEMORY_COMPACT_RATIO` 等 env | `light_context_config.context_compact_config` |
| 钉钉 sessionWebhook 写在该文档 | 与记忆栈无关，已不在本模块描述 |
)
