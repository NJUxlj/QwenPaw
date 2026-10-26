# AgentRunner 架构剖析

文档版本：QwenPaw v2.2.1  
对照源码：`src/qwenpaw/`（包版本 `__version__` = `2.2.2b1`）  
日期：2026-09-15

> **命名说明**：1.x 的 `AgentRunner` / `Runner.stream_query()` **类已删除**。
> 本文件名按产品要求保留，正文描述的是 2.x 接替层。
> 旧 `CoPawAgent(ToolGuardMixin, ReActAgent)`、路径 `src/copaw/`、
> agentscope-runtime 引擎描述均已过时。

---

## 1. 概述：AgentRunner 已移除，2.x 对应层

QwenPaw 2.x 不再有独立的 Agent 执行引擎类 `AgentRunner`。一次用户请求的
生命周期拆到四层：

| 历史职责（1.x AgentRunner） | 2.x 接替者 | 源码 |
|---|---|---|
| 按 agent 持有会话、频道、记忆、MCP | `Workspace` + `ServiceManager` | `app/workspace/workspace.py` |
| 多 agent 懒加载 / 热重载 | `WorkspaceRegistry`（IS-A `MultiAgentManager`） | `app/workspace_registry.py` |
| `Runner.stream_query(request)` | `Workspace.stream_query` → `Runtime.run` | `runtime/runtime.py` |
| 组装模型 / 工具 / 提示词 / Agent | `AgentBuilder.build` | `runtime/builder.py` |
| ReAct 循环与回复流 | `QwenPawAgent` + `AgentExecutor` | `agents/react_agent.py`、`runtime/executor.py` |
| 会话 JSON 读写 | `SafeJSONSession` + `SessionLoadHook` / `SessionSaveHook` | `app/chats/session.py`、`hooks/session/` |
| 斜杠命令 | `SlashCommandRegistry`（适配 `CommandHandler` 等） | `runtime/builtin_commands.py` |
| Tool Guard | `PolicyGuardedTool` + `ResourceGovernor`（非 Mixin） | `governance/`、`runtime/tool_guard.py`（已弃用） |
| 第三方后端（非 qwenpaw） | `HarnessRuntime` | `harnesses/runtime.py` |

### 核心技术栈（当前）

- **AgentScope 2.0.7**：基类是 `agentscope.agent.Agent`（带 `ReActConfig`），
  **不是** 1.x 的 `ReActAgent`
- **ReMe**（`reme-ai`）：默认长期记忆后端，注册名 `remelight`
- **Pydantic v2**：配置与 `AgentState`
- **Governance / Sandbox**：策略评估 + 平台沙箱，替代 `ToolGuardMixin._acting`

`QwenPawAgent` 仍通过 `agents/__init__.py` 的 `__getattr__` 懒加载，避免 CLI
导入时拉起 AgentScope / 工具。

---

## 2. 请求路径

### 2.1 启动挂载

`app/_app.py` 的 `lifespan()`：

1. 创建 `AppServiceManager`（跨 workspace 的 approval / tool coordinator）
2. 创建 `WorkspaceRegistry(app_services=...)`，写入
   `app.state.workspace_registry`
3. `WorkspaceBootstrapFactory.build_bootstrap_kwargs()` 发现内置工具、
   hooks、斜杠命令、prompt contributors、modes
4. 把 kwargs 写入 `workspace_registry._bootstrap_kwargs`
5. **兼容别名**：`app.state.multi_agent_manager = workspace_registry`

之后 `WorkspaceRegistry._create_workspace()` 会 `Workspace(...)` 再立刻
`bootstrap_plugins(**kwargs)`，把内置能力注入该 workspace 的
`WorkspacePlugins`。

### 2.2 入站

频道由 `create_channel_service` 构造，关键绑定：

```python
ChannelManager.from_config(process=ws.stream_query, ...)
```

因此：

```
Channel SDK 回调
  → BaseChannel.enqueue
  → ChannelManager / UnifiedQueueManager
  → consume_one / _consume_one_request
      → build_agent_request_from_native() → AgentRequest
      → self._process(request)   # 即 Workspace.stream_query
```

Console 是内置频道之一，走同一 `process=` 绑定。Cron、Heartbeat、邮箱
监控、ACP server、PawApp `ctx.chat()` 也会直接
`async for event in workspace.stream_query(req)`。

### 2.3 `Workspace.stream_query`

```python
async def stream_query(self, request):
    backend = load_agent_config(self.agent_id).backend
    if backend != "qwenpaw":
        # Codex / Qoder 等第三方 harness
        async for item in self.harness_runtime.stream(...):
            yield item
        return
    rt = Runtime(workspace=self, app_services=self._app_services)
    async for item in rt.run(request):
        yield item
```

默认 `backend == "qwenpaw"` 时进入 `Runtime`。其余后端由
`HarnessRuntime` 翻译成同一套 SSE envelope。

### 2.4 `Runtime.run`：八阶段 + 三个固定步骤

`runtime/phases.py` 固定八个钩子相位。斜杠分发与
`AgentBuilder.build` **不是 hook**，而是夹在相位之间的固定步骤：

```
PRE_DISPATCH
  → SlashCommandRegistry.dispatch()     # 命中则 SKIP_AGENT
POST_DISPATCH
PRE_AGENT_BUILD                         # SessionLoadHook 读 JSON
  → AgentBuilder.build(ctx)             # 构造 QwenPawAgent
  → mode.on_turn_start(ctx)
POST_AGENT_BUILD
PRE_EXECUTE                             # BootstrapHook 等
  → Envelope.emit_response_created
  → AgentExecutor.run(input_msgs)       # agent.reply_stream
POST_RESPONSE                           # SessionSaveHook 写 JSON
ON_ERROR / FINALLY
```

`AgentExecutor` 调用 `agent.reply_stream(inputs=msgs)`，用 heartbeat
包装空闲等待（审批阻塞时发 keep-alive），再由 `Envelope.translate_event`
转成与旧 `Runner.stream_query` 相同形态的 SSE 对象。

---

## 3. QwenPawAgent：MRO、初始化与注入

### 3.1 真实 MRO

```python
class QwenPawAgent(CodingModeMixin, Agent):
```

```
QwenPawAgent
    ├── CodingModeMixin     # Coding Mode：项目目录 / LSP·AST 工具收集
    └── agentscope.agent.Agent
            └── （AgentScope 2.0 ReAct 循环，由 ReActConfig 驱动）
```

**已删除、切勿再写：**

- `CoPawAgent`
- `ToolGuardMixin`
- 继承 `ReActAgent` / `AgentBase`
- 在 `__init__` 内部 `_create_toolkit` / `_register_hooks`

AgentScope 内置权限引擎被显式绕过：

```python
self.state.permission_context.mode = PermissionMode.BYPASS
```

工具安全改走 `PolicyGuardedTool.check_permissions`。

### 3.2 构造全部委托 `AgentBuilder`

`QwenPawAgent.__init__` **不**创建 toolkit / model / prompt。签名全部是
外部注入：

```python
QwenPawAgent(
    name=...,
    model=...,
    system_prompt=...,
    toolkit=...,
    react_config=ReActConfig(max_iters=effective_max),
    middlewares=...,
    agent_config=...,
    workspace_dir=...,
    request_context=...,
    offloader=...,
    context_config=...,
    context_manager=scroll.context_manager or None,
    effective_skills=...,
    governor=...,
)
```

`AgentBuilder.build(ctx)` 顺序（摘自 `runtime/builder.py`）：

1. `load_agent_config`；校验 `active_model`
2. `ensure_skills_initialized` + `resolve_effective_skills(workspace, channel)`
3. `plugins.active_mode_names(ctx)` 得到本轮 mode
4. `_init_governor` → `ResourceGovernor.start()`
5. `local_workspace.set_governor(governor)`，后续 `list_tools` 全部包一层
   `PolicyGuardedTool`
6. 额外工具：Coding Mode（`lsp` / `ast_search`）、Scroll recall、Driver/MCP
7. `create_model_and_formatter`（`agents/model_factory.py`）
8. `_build_offloader`；可选 `_build_scroll_components`（仅
   `light_context_config.strategy == "scroll"`）
9. `build_toolkit`：`QwenPawLocalWorkspace.list_tools()` + memory tools +
   AgentScope `Toolkit(tools=..., skills_or_loaders=...)`
10. `PromptManager` 拼系统提示
11. `_build_middlewares`
12. 实例化 `QwenPawAgent`
13. 若 `SessionLoadHook` 已填 `ctx.session_state`，调用
    `agent.load_state_dict(...)`

### 3.3 工具如何注入

内置函数带 `@tool_descriptor`，在 `agents/tools/__init__.py` 的副作用
import 时进入全局表。`discover_builtin_tool_funcs()` 返回该表；
`Workspace.bootstrap_plugins` 把它们登记进 **本 workspace** 的
`ToolRegistry`。

每请求：

```
QwenPawLocalWorkspace.list_tools(
    agent_config, active_modes, active_skills, enabled_features, ...)
  → ToolRegistry.filter(...)          # 配置 enabled=false → denied
  → PolicyGuardedTool(func, governor, request_context)
```

#### 内置工具（`agents/tools/`，对照当前 `__init__.py`）

| 工具 | 文件 | 说明 |
|------|------|------|
| `execute_shell_command` | `shell.py` | Shell；ToolCoordinator 默认超时 60s |
| `read_file` / `write_file` / `edit_file` / `append_file` | `file_io.py` | 文件读写；`append_file` 为 2.x 新增 |
| `grep_search` / `glob_search` | `file_search.py` | 文本 / glob |
| `browser` | `browser.py` 或 `deprecated_browser/browser_control.py` | 由 `browser.experimental` 二选一；**不再叫 `browser_use`** |
| `desktop_screenshot` | `desktop_screenshot.py` | 桌面截图 |
| `view_image` / `view_video` | `view_media.py` | 媒体查看（`view_video` 新增） |
| `send_file_to_user` | `send_file.py` | 发文件给用户 |
| `web_search` / `web_fetch` | `web_search.py` | 联网搜索 / 抓取 |
| `get_current_time` / `set_user_timezone` | `get_current_time.py` | 时间 / 时区 |
| `get_token_usage` | `get_token_usage.py` | Token 用量 |
| `list_agents` / `chat_with_agent` / `submit_to_agent` / `check_agent_task` / `spawn_subagent` | `agent_management.py` | 多 agent |
| `delegate_external_agent` | `delegate_external_agent.py` | 外部 agent |
| `ast_search` | `ast_tool.py` | AST 搜索 |
| `run_tool_batch` | `run_tool_batch.py` | 批量工具 |
| `activate_f1_exploration_mode` | `mail_f1_tool.py` | 邮箱 F1 探索 |
| `migration_compat_*` | `migration_compatibility.py` | 迁移兼容（自授权 opt-in） |

**已从内置表移除或改名：**

- `browser_use` → `browser`
- `create_memory_search_tool`：改为 `BaseMemoryManager.list_memory_tools()`
  返回的 `memory_search`（由 builder 追加，再包 `PolicyGuardedTool`）
- `execute_python_code` / `view_text_file` / `write_text_file`：明确不注册
  （`tools/__init__.py` 注释）

Coding Mode 额外：`make_lsp_tool()` 生成的单一 `lsp` 工具（非
`@tool_descriptor` 全局表）。Scroll 额外：`recall_history`；沙箱可用时还有
`recall_history_python`。

### 3.4 技能如何注入

技能系统入口是 `agents/skill_system/`，**不再**用
`agents/skills_manager.py` 的三层 `builtin / customized / active` 目录。

| 层 | 路径 | 说明 |
|---|---|---|
| 内置源码 | `src/qwenpaw/agents/skills/<name>-{zh,en}/SKILL.md` | 按语言分目录 |
| 共享池 | `<WORKING_DIR>/skill_pool/` | `get_skill_pool_dir()` |
| 本 workspace | `<workspace_dir>/skills/`（旧名 `skill/` 会改名） | `get_workspace_skills_dir` |
| 清单 | `<workspace_dir>/skill.json` | 启用集 |

`AgentBuilder` 调用 `resolve_effective_skills(workspace, channel)`，再：

- 把 skill 目录交给 AgentScope `Toolkit(..., skills_or_loaders=...)`
- `QwenPawAgent._register_skills` 写入 `toolkit._qp_skills`，供
  `/<skill_name>` fallback 使用

当前内置 skill 名（去语言后缀）：`browser`、`channel_message`、
`chat_with_agent`、`cron`、`dingtalk_channel`、`docx`、`file_reader`、
`guidance`、`mailbox`、`make-skill`、`make_plan`、
`multi_agent_collaboration`、`pdf`、`pptx`、`xlsx`、`QA_source_index`。

旧文档中的 `himalaya`、`news`、`browser_visible` **源码树中已不存在**。

未匹配任何斜杠命令时，`get_skill_fallback_handler()` 把
`/<skill>` 的 SKILL.md 正文注入用户消息，而不是当系统命令短路。

### 3.5 记忆如何注入

Workspace 服务 `memory_manager`（priority 20，optional）：

- 配置键：`agent.json` → `running.memory_manager_backend`，默认
  `"remelight"`
- 工厂：`create_memory_manager_backend(id, MemoryBackendContext)`
- 核心后端：`ReMeLightMemoryManager`（`@memory_registry.register("remelight")`）
- 关闭：`NoopMemoryManager`（`"none"`）
- 配置后端不可用时 **运行时** 回退到 `remelight`，不改写配置

Builder 侧：

- `memory_manager.list_memory_tools()` 并入 toolkit
- `memory_manager.build_middlewares()` 并入 middleware 洋葱（系统提示注入、
  自动召回、回复后调度）

**已删除：** `MemoryManager(ReMeLight)` 直接继承、`MemoryCompactionHook`
在 `pre_reasoning` 里 `compact_memory`。短期上下文压缩见 3.6。

### 3.6 Loop / Scroll / 治理如何注入

**Loop（推理循环）**

- AgentScope `Agent` + `ReActConfig(max_iters=...)`，上限由
  `modes.default.resolve_max_iterations` 与请求上下文决定
- `QwenPawAgent._reasoning` 覆盖：待处理 Loop Gate、媒体剥离与重试、
  上下文溢出一次恢复、每轮 `run_stop_handlers`
- Stop handlers 来自 `PluginRegistry.get_stop_handlers(agent_id)`
- 内置 gate 目录：`loop/catalog.py`（`IterationGate`、`DoomLoopGate`、
  `TokenBudgetGate`、`TimeoutGate`、`ToolCallBudgetGate`、
  `QualitativeRubricGate`、`CompletionRubricGate` 等）
- `StopAction.INTERRUPT_AND_CONTINUE` 会向 context 追加一条带
  `LOOP_CONTINUATION_MESSAGE_TAG` 的 user 消息，外层继续迭代

**Scroll（可选上下文策略）**

- 仅当 `light_context_config.strategy == "scroll"`
- `ScrollContextManager` 注入 `QwenPawAgent._context_manager`
- write-through 到 workspace 内 `history.db`；超阈值驱逐中间段，留下
  continuation summary + eviction index
- `compress_context` 走 AgentScope middleware 链，真正实现在
  `_compress_context_impl` → `context_manager.compress`
- 未启用 scroll 时用 AgentScope 原生压缩（可被
  `context_compact_config.enabled` 关掉）

**治理**

见第 5 节。Governor 实例挂在 `agent._governor`，`close()` 时 `gov.stop()`。

### 3.7 中间件（AgentScope `MiddlewareBase`）

与 Runtime 的 8-phase `LifecycleHook` **正交**。Builder 组装顺序（外→内）：

1. `ToolResultPruningMiddleware` — 工具结果截断
2. `ToolCoordinatorMiddleware` — 工具调用生命周期 / 后台 hint
3. 记忆后端 `build_middlewares()`（`MemoryMiddleware` 等）
4. 可选 `LangfuseToolSpanMiddleware`
5. 插件 `get_middleware_factories()`
6. `VisualCompressionMiddleware` — 请求边界视觉压缩

`QwenPawAgent` 不再 `register_instance_hook("pre_reasoning", ...)`。

### 3.8 系统提示

`PromptManager` + `runtime/prompt_contributors.py` 的
`_ALL_CONTRIBUTORS`：

1. `ProtectedExecutionContractContributor`
2. `WorkspacePromptFilesContributor` — 默认仍读 `AGENTS.md` / `SOUL.md` /
   `PROFILE.md`（`agent.json` 的 `system_prompt_files` 可改；空列表表示关闭）
3. `MultimodalHintContributor`
4. `DirectoryContextContributor`
5. `CodingModeContributor`
6. `ScrollContextContributor`
7. `DriverPolicyHintContributor`
8. `EnvContextContributor`
9. `PreloadedSkillsContributor`

`agents/prompt.py` 仍提供文件读取与 bootstrap 文案，但 **主路径不再是**
单独的 `build_system_prompt_from_working_dir()` 拼进 `CoPawAgent`。

---

## 4. 会话 / 状态如何持久化

`SafeJSONSession` **仍然存在**（`app/chats/session.py`），作为 Workspace
的 `session` 服务（priority 10）：

```
<workspace_dir>/sessions/<channel>/<sanitized_user>_<sanitized_session>.json
```

Windows 非法文件名字符被替换为 `--`。写入走 `write_json_atomic_async`
（先 `.tmp` 再 replace）。

### 4.1 正常路径

| Hook | Phase | 行为 |
|------|-------|------|
| `SessionLoadHook` | `PRE_AGENT_BUILD` | `session.load_session_state` → `ctx.session_state`；builder 调 `agent.load_state_dict` |
| `SessionSaveHook` | `POST_RESPONSE` | `agent.state_dict()` + `ctx.mode_state` → `session.save_session_state` |

ACP 临时请求（`request_context` 里 ephemeral 标记）跳过读写。

### 4.2 Agent 快照格式

`QwenPawAgent.state_dict()`：

```python
{"state": AgentState.model_dump(mode="json"), "scroll": cm.to_dict()}  # scroll 可选
```

`load_state_dict` 识别：

- **2.0**：`{"state": ...}`，并 `_sanitize_loaded_context()` 去掉孤儿
  `tool_result`
- **1.x**：`{"memory": {"content": [...], "_compressed_summary": "..."}}`，
  经 `parse_legacy_memory_state` 迁到 `AgentState`

取消路径：`Runtime._try_save_on_cancel` 把 Envelope 里未完成的文本 /
dangling tool call 补进 context，再 `asyncio.shield` 写盘。

Scroll 的 `history.db` 是另一份耐久历史，与 session JSON 互补，不是替代。

---

## 5. 安全接入点（Tool Guard / Governance / Sandbox）

### 5.1 不再走 Mixin `_acting`

1.x：`ToolGuardMixin._acting` 拦截 `ReActAgent._acting`。  
2.x：每个工具在进入 toolkit 前包成 `PolicyGuardedTool`
（`governance/tool_adapter.py`）。`runtime/tool_guard.GuardedFunctionTool`
仅作兼容，注释标明 **deprecated**。

`PolicyGuardedTool` 两层：

1. `check_permissions` — 预执行决策（ALLOW / DENY / ASK）
2. `__call__` — 实际执行，含沙箱违规重试

审批级别（`security/tool_guard/execution_level.py`）：

| 级别 | 行为 |
|------|------|
| `strict` | 几乎全部工具先审批 |
| `smart` | 低风险自动放行，中高危审批 |
| `auto` | 兼容旧 guarded 列表 |
| `off` | 关闭防护（开发） |

解析顺序：`request_context["approval_level"]` → `agent.json.approval_level`。
`execution_level=off` 也可写在 `<WORKING_DIR>/.qwenpaw/policy.yaml`。

### 5.2 ResourceGovernor

`AgentBuilder._init_governor` 按 **本请求实际项目目录** 构造
`ResourceGovernor`：

- `assert_policy(tool_call)` → `GovernanceDecision`
- 审计日志
- `compile_sandbox_config()` → `SandboxConfig`
- 用户批准后 `add_rule(...)`

Governor 初始化失败时 **fail-closed**（治理层禁用，工具侧保守拒绝 /
降级），并打 error 日志。无 governor 且未开 `QWENPAW_ALLOW_UNSANDBOXED_RECALL`
时，Scroll 会回退到 native 上下文，避免驱逐后无法召回。

### 5.3 Sandbox

`qwenpaw/sandbox/`：按平台 `detect_platform_mode` / `probe_sandbox_support`。

| 模式 | 平台 |
|------|------|
| `SEATBELT` | macOS `sandbox-exec` |
| `BUBBLEWRAP` / `LANDLOCK` | Linux |
| `WINDOWS` | AppContainer / Elevated / Unelevated |
| `NONE` | 无隔离 |

生命周期是 **按工具调用** 创建销毁，不是按会话长驻。

### 5.4 斜杠命令里的审批

`/approval`、`/approve`、`/deny` 由 control handlers 处理；
`AppServiceManager.approval_coordinator` 经 `request_context` 注入工具包装。

---

## 6. 斜杠命令与生命周期钩子（对照旧表）

### 6.1 分发点

`Runtime.run` 在 PRE_DISPATCH 之后、建 Agent 之前调用
`workspace.plugins.slash_command_registry.dispatch(text, ctx)`。
命中则直接 `envelope.from_msg`，**不再**进入 `QwenPawAgent.reply`。

`agents/command_handler.py` 的 `CommandHandler` /
`ConversationCommandHandlerMixin` **仍在**，但是被
`runtime/builtin_commands.py` 的 adapter 调用，而不是 Agent 内部
`is_command()`。

### 6.2 对话命令（`SYSTEM_COMMANDS`）

| 命令 | 说明 |
|------|------|
| `/compact` | 压缩当前上下文（可带指令） |
| `/new` | 新对话 |
| `/clear` | 清空上下文 |
| `/history` | 查看历史 |
| `/compact_str` | 显示压缩摘要 |
| `/auto_memory_status` | 自动记忆状态（2.x 新增） |
| `/message` | 发消息 |
| `/dump_history` / `/load_history` | 导出 / 导入历史 |
| `/proactive` | 主动记忆 |
| `/plan` | 仅无参数时为状态命令；`/plan <desc>` 放行给 Agent |
| `/system_prompt` | 查看系统提示 |
| `/reme` | ReMe 动作（受限白名单） |

**已删除：** `/await_summary`。

### 6.3 控制命令

`/stop`、`/model`、`/skills`、`/approval`、`/approve`、`/deny`、
`/checkpoint`。

### 6.4 Daemon 命令

`/daemon status|restart|reload-config|version|logs`，短名 `/restart`、
`/status` 等为别名。`/daemon restart` 走 `MultiAgentManager` 热重载，
不退出进程。

### 6.5 内置 LifecycleHook（`WorkspaceBootstrapFactory`）

| Hook | 典型 Phase |
|------|------------|
| `CronContextHook` / `CronMemoryIsolateHook` / `CronMemoryRestoreHook` | cron 隔离 |
| `SessionLoadHook` / `SessionSaveHook` | 会话 |
| `BootstrapHook` | `PRE_EXECUTE`：首轮注入 `BOOTSTRAP.md` |
| `SkillEnvHook` / `SkillEnvCleanupHook` | skill 环境变量 |
| `ContextVarsSetupHook` / `AgentContextVarsSetupHook` | ContextVar |
| `MediaProcessHook` | 入站媒体 |
| `ErrorNormalizeHook` / `CancelCleanupHook` | 错误 / 取消 |
| `CheckpointQueryGateHook` / `CheckpointAutoSnapshotHook` | checkpoint |
| `MailF1CleanupHook` | 邮箱 F1 |
| 可选 `LangfuseTraceHook` / `LangfuseTraceCleanupHook` | 可观测 |

**已删除：** `MemoryCompactionHook`（pre_reasoning）。上下文压缩改在
`QwenPawAgent.compress_context` / Scroll。

---

## 7. Workspace 服务清单

`Workspace._register_services`（数字越小越先启动）：

| Priority | 服务 | 说明 |
|----------|------|------|
| 5 | `local_workspace` | `QwenPawLocalWorkspace`，`list_tools` 路由 |
| 10 | `session` | `SafeJSONSession(save_dir=.../sessions)` |
| 20 | `memory_manager` | 可热重用；optional |
| 20 | `driver_manager` | Driver / MCP；optional |
| 20 | `chat_manager` | 会话列表 `chats.json` |
| 30 | `channel_manager` | `start_all` / `stop_all` |
| 40 | `cron_manager` | `jobs.json` |
| 45 | `mail_monitor` | 条件启动 |
| 50 | `agent_config_watcher` | `agent.json` 热更新 |
| 51 | `driver_config_watcher` | Driver card 热更新 |

`memory_manager` 与 `chat_manager` 可在 reload 时 `set_reusable_components`
复用。

内置 AgentMode：`DefaultMode`、`CodingMode`、`MissionMode`、`GoalMode`，
另可 `load_custom_loop_modes`。

---

## 8. 调用链 ASCII 图

### 8.1 端到端（qwenpaw 后端）

```
Channel / Console / Cron / ACP / PawApp
        │
        ▼
WorkspaceRegistry.get_agent(agent_id)     # IS-A MultiAgentManager
        │
        ▼
Workspace.stream_query(AgentRequest)
        │
        ├── backend != "qwenpaw" ──▶ HarnessRuntime.stream (codex/qoder)
        │
        ▼
Runtime.run
        │
        ├── PRE_DISPATCH hooks
        ├── SlashCommandRegistry.dispatch ──命中──▶ Envelope.from_msg ──▶ SSE
        ├── PRE_AGENT_BUILD: SessionLoadHook
        ├── AgentBuilder.build
        │       ├── ResourceGovernor
        │       ├── QwenPawLocalWorkspace.list_tools → PolicyGuardedTool
        │       ├── model_factory.create_model_and_formatter
        │       ├── PromptManager
        │       ├── middlewares + optional ScrollContextManager
        │       └── QwenPawAgent(...) + load_state_dict
        ├── PRE_EXECUTE: BootstrapHook / SkillEnv / Media ...
        ├── AgentExecutor: QwenPawAgent.reply_stream
        │       └── AgentScope Agent ReAct
        │               ├── compress_context (native 或 Scroll)
        │               ├── _reasoning（媒体 / overflow / Loop gates）
        │               └── _execute_tool_call
        │                       └── PolicyGuardedTool
        │                               ├── check_permissions (+ Approval)
        │                               └── sandbox execute
        ├── POST_RESPONSE: SessionSaveHook → SafeJSONSession JSON
        └── FINALLY: agent.close()（governor.stop / scroll.purge / offloader）
```

### 8.2 Agent 构造（替代旧 `__init__` 七步）

```
AgentBuilder.build(ctx)
    1. load_agent_config + resolve_effective_skills + active modes
    2. ResourceGovernor.start
    3. extra tools (coding / scroll / drivers)
    4. model + formatter
    5. offloader + optional ScrollComponents
    6. Toolkit ← LocalWorkspace.list_tools + memory_search + skills
    7. system prompt ← PromptManager
    8. middlewares
    9. QwenPawAgent(...)
   10. load_state_dict(ctx.session_state)
```

### 8.3 工具执行（替代旧 ToolGuardMixin._acting）

```
模型产出 tool_call
    → QwenPawAgent._execute_tool_call   # 先 schema 强制类型 (#6839)
    → PolicyGuardedTool.check_permissions
            ├── DENY  → ToolResult + 禁止重试说明
            ├── ASK   → ApprovalService，阻塞至用户 /approve|/deny
            └── ALLOW → __call__
                    └── ResourceGovernor 编译的 SandboxConfig
                            └── 平台 sandbox 或直接执行
```

---

## 9. 模型工厂（仍存在，职责缩小）

`agents/model_factory.py` 的 `create_model_and_formatter` 仍按 provider
选 ChatModel 与 Formatter（OpenAI / Anthropic / Gemini / DashScope 等）。
Agent 不再在自己的 `__init__` 里调用它；由 `AgentBuilder.build_model`
调用。Formatter 上还有 QwenPaw 扩展属性（强制剥离 media/audio、省略
thinking id），供 `_reasoning` 的被动重试使用。

---

## 10. 配置要点（agent.json，勿与 1.x 全局 `config.json.agents` 混淆）

每 workspace 一份 `agent.json`（`load_agent_config(agent_id)`）：

- `backend`：`qwenpaw` 或 harness id
- `active_model`：provider_id + model
- `running.memory_manager_backend`：`remelight` / `none` / 插件
- `running.light_context_config.strategy`：`native` 或 `scroll`
- `tools.builtin_tools.<name>.enabled` / `timeout_seconds`
- `approval_level`
- `coding_mode.enabled` / `project_dir`
- `system_prompt_files`
- `heartbeat.enabled`

内置工具默认启用表由 `@tool_descriptor` 元数据生成，不再手写整表到
旧 `CoPawAgent._create_toolkit`。

---

## 11. 设计模式对照

| 1.x | 2.x |
|-----|-----|
| Mixin 拦截 `_acting` | 工具包装 `PolicyGuardedTool` |
| Agent 内部建 toolkit | `AgentBuilder` 每请求组装 |
| `pre_reasoning` instance hook | Runtime `LifecycleHook` + AgentScope middleware |
| Agent 内 `is_command` | `SlashCommandRegistry` 先于 Agent |
| 单例 Runner 管 session | Workspace 服务 `SafeJSONSession` + 相位 hook |
| `ReActAgent.max_iters` | `ReActConfig(max_iters)` + Loop StopHandler |
| 三层 skills 目录 | skill pool + workspace `skills/` + `skill.json` |

懒加载未改：`from qwenpaw.agents import QwenPawAgent` 仍走 `__getattr__`。

---

## 12. 过时 API 速查（读旧文档时直接丢弃）

| 过时符号 / 路径 | 现状 |
|-----------------|------|
| `AgentRunner` / `app/runner/runner.py` | 删除；用 `Workspace` + `Runtime` |
| `CoPawAgent` | 改为 `QwenPawAgent` |
| `src/copaw/...` | 改为 `src/qwenpaw/...` |
| `ToolGuardMixin` | 删除 |
| `class X(ReActAgent)` | `QwenPawAgent(CodingModeMixin, Agent)` |
| `_create_toolkit` / `_register_hooks` 在 Agent 内 | `AgentBuilder` + `bootstrap_plugins` |
| `MemoryCompactionHook` | 删除 |
| `skills_manager` 三层目录 | `skill_system` |
| `toolkit.register_agent_skill` | AgentScope `Toolkit(skills_or_loaders=)` + `_qp_skills` |
| `browser_use` | `browser` |
| `/await_summary` | 删除 |
| Agent 内 `reply()` 先 `is_command` | Runtime 斜杠分发 |
| `register_instance_hook("pre_reasoning")` | 8-phase hooks + middleware |
| `GuardedFunctionTool` 作为主路径 | deprecated；用 `PolicyGuardedTool` |
| agentscope-runtime 作为执行内核 | 自研 `Runtime` + AgentScope 2.0 `Agent` |
