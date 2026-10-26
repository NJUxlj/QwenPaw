# QwenPaw 核心功能剖析

> 产品：**QwenPaw v2.2.1**（对照日期 **2026-09-15**）
> 历史名称：1.x 时代称为 **CoPaw**，2.0 起更名为 QwenPaw。本文按当前 `src/qwenpaw/` 源码描述 2.x Agent OS，不再沿用 1.x 的 `AgentRunner` / `CoPawAgent` / `src/copaw/` 路径。
> 源码树快照：`src/qwenpaw/__version__.py` 当前为 `2.2.2b1`；包名 `qwenpaw`（见 `pyproject.toml`）。

本文按模块说明 QwenPaw 2.x 仍在源码中落地的主功能：概述、调用链、核心代码路径。路径均来自本仓库，不以 README 营销句代替实现。

数据根目录 **`WORKING_DIR`**（`src/qwenpaw/constant.py`）：优先 `QWENPAW_WORKING_DIR`；否则若本机存在 `~/.copaw` 则沿用该遗留目录；否则为 `~/.qwenpaw`。每个智能体工作区默认在 `{WORKING_DIR}/workspaces/<agent_id>/`。

---

## 0. 整体架构（Agent OS）

QwenPaw 2.x 不再是「全局一个 AgentRunner」。每个智能体是一个独立 **Workspace**（资源 + 治理 + 沙箱 + 驱动），由 **WorkspaceRegistry** 管理。请求处理交给 **Runtime**（8 阶段编排），智能体由 **AgentBuilder** 按请求组装。

```
用户交互
  频道 / Console / TUI(ACP) / Cron / Mail / PawApp
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  ChannelManager（每工作区一份；process = Workspace.stream_query）│
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  WorkspaceRegistry  (IS-A MultiAgentManager)                │
│  - 懒加载 / 热重载 / 并行启动多个 Workspace                   │
│  - FastAPI 上同时挂在 app.state.workspace_registry           │
│    与 app.state.multi_agent_manager（兼容旧路由）            │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Workspace.stream_query()                                   │
│  - backend != "qwenpaw" → HarnessRuntime（Codex / Qoder）    │
│  - backend == "qwenpaw" → Runtime.run()                     │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Runtime（8 阶段）                                           │
│  PRE_DISPATCH → slash 分发 → PRE_AGENT_BUILD                 │
│  → AgentBuilder.build() → QwenPawAgent                       │
│  → AgentExecutor（reply_stream + heartbeat）                 │
└─────────────────────────────────────────────────────────────┘
    │
    ├── Toolkit / PolicyGuardedTool / Sandbox
    ├── Skills（skill_pool + workspace/skills）
    ├── ScrollContextManager + ReMeLightMemoryManager
    └── DriverManager（MCP 等协议）
```

**启动（`src/qwenpaw/app/_app.py` `lifespan()`）：**

1. 遗留迁移（默认智能体、QA 智能体、skill pool、Scroll 历史回填）
2. `ProviderManager` / `LocalModelManager`
3. `AppServiceManager` + `WorkspaceRegistry`
4. `WorkspaceBootstrapFactory` 注入内置工具 / Mode / Hook / 斜杠命令
5. 后台：先加载 `channel` / `memory` 插件，再 `start_all_configured_agents()`
6. 浏览器 runtime watchdog、TokenUsage、Backup 等进程级服务

---

## 1. 工作区（Workspace / Agent OS）

### 1.1 功能概述

每个 Workspace 是一套独立运行时，拥有自己的频道、记忆、驱动、定时任务、插件注册表。`WorkspaceRegistry` 在创建后立刻 `bootstrap_plugins()`，把内置工具、Loop Mode、Hook、斜杠命令注入该工作区。

工作区服务由 `ServiceManager` 按优先级注册：LocalWorkspace → Session → Memory / Driver / Chat → ChannelManager → Cron / MailMonitor 等。

### 1.2 调用链分析

```
WorkspaceRegistry._create_workspace(agent_id, workspace_dir)
    │
    ├── Workspace(agent_id, workspace_dir)
    │     ├── WorkspacePlugins（tool / hook / command / prompt / mode）
    │     ├── QwenPawLocalWorkspace（list_tools → ToolRegistry）
    │     └── ServiceManager._register_services()
    │
    ├── bootstrap_plugins(**factory_kwargs)
    │     ├── 内置工具描述符 → tool_registry
    │     ├── CodingMode / MissionMode / GoalMode / DefaultMode
    │     ├── load_custom_loop_modes()
    │     └── 斜杠命令 + LifecycleHook
    │
    └── set_app_services(AppServiceManager)

请求：
Workspace.stream_query(request)
    ├── agent.json.backend != "qwenpaw"
    │     └── HarnessRuntime.stream(backend=codex|qoder, ...)
    └── Runtime(workspace, app_services).run(request)
```

### 1.3 核心代码路径

| 组件 | 文件路径 |
|------|----------|
| WorkspaceRegistry | `src/qwenpaw/app/workspace_registry.py` |
| MultiAgentManager | `src/qwenpaw/app/multi_agent_manager.py` |
| Workspace | `src/qwenpaw/app/workspace/workspace.py` |
| 服务工厂 | `src/qwenpaw/app/workspace/service_factories.py` |
| Bootstrap | `src/qwenpaw/app/workspace/bootstrap_factory.py` |
| QwenPawLocalWorkspace | `src/qwenpaw/app/workspace/local_workspace.py` |
| WorkspacePlugins | `src/qwenpaw/app/workspace/workspace_plugins.py` |
| 会话 | `src/qwenpaw/app/chats/session.py`（`SafeJSONSession`） |
| 检查点 | `src/qwenpaw/checkpoints/` |
| 第三方后端 | `src/qwenpaw/harnesses/`（Codex / Qoder） |

---

## 2. 智能体循环与 Loop Engineering

### 2.1 功能概述

**QwenPawAgent**（`agentscope.agent.Agent` + `CodingModeMixin`）不再在构造函数里自行搭 Toolkit / Prompt / 记忆。这些由 **AgentBuilder** 按请求组装后注入。

**Runtime** 固定 8 个生命周期相位（`runtime/phases.py`），斜杠命令分发与 `AgentBuilder.build` 夹在相位之间。执行阶段由 **AgentExecutor** 驱动 `agent.reply_stream()`。

**Loop Engineering** 是可组合的停止门（StopGate）：

- 内置 Mode：`default`（默认 ReAct）、`coding`、`mission`、`goal`
- 用户可编辑自定义 Loop：`CustomLoopModeConfig` → `compile_loop_mode()` → `DeclarativeLoopMode`
- 门目录：iteration、doom_loop、token_budget、timeout、tool_call_budget、qualitative_rubric、completion_rubric

Agent 绕过 AgentScope 自带权限引擎（`PermissionMode.BYPASS`），工具权限走 QwenPaw 的 `PolicyGuardedTool.check_permissions`。

### 2.2 调用链分析

```
Runtime.run(request)
    ├── [phase] PRE_DISPATCH
    ├── SlashCommandRegistry.dispatch()
    ├── [phase] POST_DISPATCH / PRE_AGENT_BUILD
    ├── AgentBuilder.build(ctx)
    │     ├── local_workspace.list_tools(...)
    │     ├── PromptManager 拼系统提示
    │     ├── create_model_and_formatter / RoutingChatModel
    │     └── QwenPawAgent(model, toolkit, middlewares, governor, ...)
    ├── [phase] POST_AGENT_BUILD / PRE_EXECUTE
    ├── AgentExecutor → agent.reply_stream()
    │     ├── ReAct 推理 / 工具调用
    │     ├── StopHandler + StopGate（Loop 门）
    │     └── ScrollContextManager.compress / on_save
    └── [phase] POST_RESPONSE / ON_ERROR / FINALLY
```

**QwenPawAgent 初始化（依赖外注）：**

```python
QwenPawAgent.__init__()
    ├── _register_skills(toolkit, effective_skills)  # 只登记路径到 toolkit._qp_skills
    ├── super().__init__(model, system_prompt, toolkit, middlewares, ...)
    ├── PermissionMode.BYPASS
    └── _register_tool_call_hooks()
```

### 2.3 核心代码路径

| 组件 | 文件路径 |
|------|----------|
| QwenPawAgent | `src/qwenpaw/agents/react_agent.py` |
| Agent 懒加载 | `src/qwenpaw/agents/__init__.py`（`__getattr__`） |
| Runtime | `src/qwenpaw/runtime/runtime.py` |
| AgentBuilder | `src/qwenpaw/runtime/builder.py` |
| AgentExecutor | `src/qwenpaw/runtime/executor.py` |
| 相位枚举 | `src/qwenpaw/runtime/phases.py` |
| Loop 门 | `src/qwenpaw/loop/`、`src/qwenpaw/loop/gates/` |
| 门目录 / 编译 | `src/qwenpaw/loop/catalog.py`、`src/qwenpaw/loop/compiler.py` |
| DefaultMode | `src/qwenpaw/modes/default/mode.py` |
| CodingMode | `src/qwenpaw/modes/coding/` |
| MissionMode | `src/qwenpaw/modes/mission/` |
| GoalMode | `src/qwenpaw/modes/goal/` |
| 自定义 Loop | `src/qwenpaw/modes/custom_loop/` |
| 模型工厂 / 路由 | `src/qwenpaw/agents/model_factory.py`、`routing_chat_model.py` |
| 内置工具 | `src/qwenpaw/agents/tools/` |

内置工具（`agents/tools/__init__.py` 通过 `@tool_descriptor` 收集）包括：`read_file` / `write_file` / `edit_file` / `append_file`、`grep_search` / `glob_search`、`execute_shell_command`、`send_file_to_user`、`web_search` / `web_fetch`、`desktop_screenshot`、`view_image` / `view_video`、时间与 token、多智能体（`list_agents` / `chat_with_agent` / `spawn_subagent` 等）、`ast_search`、`run_tool_batch`、邮件 F1、以及实验性统一浏览器或旧版 browser 实现。

---

## 3. Channel 系统（多频道消息）

### 3.1 功能概述

频道对接即时通讯与语音入口，统一收成 `AgentRequest`，再交给本工作区的 `Workspace.stream_query`。`ChannelManager.from_config(process=ws.stream_query, ...)` 在 `service_factories.create_channel_service` 中绑定。

**内置频道**（`registry.py` `_BUILTIN_SPECS`，加载失败则跳过；`console` 为必需）：

钉钉、飞书、Discord、QQ、Telegram、企业微信、微信、iMessage、Matrix、Mattermost、Slack、MQTT、Voice、SIP、小艺、元宝、OneBot、Console。

插件类型 `channel` 可在启动阶段注册额外频道。

### 3.2 调用链分析

```
外部消息（Webhook / WebSocket / 轮询）
    │
    ▼
具体 Channel（DingTalkChannel, FeishuChannel, ...）
    ├── 解析原生 payload → AgentRequest
    └── enqueue() → ChannelManager 队列
    │
    ▼
ChannelManager
    ├── UnifiedQueueManager（每频道队列，默认 maxsize=1000）
    ├── 合并同一会话的连续消息
    └── _process_batch() → consume_one / _consume_one_request
    │
    ▼
BaseChannel._process(request)
    └── process handler = Workspace.stream_query
    │
    ▼
流式 Event → send_event() / send_text() → 渠道 API
```

### 3.3 核心代码路径

| 组件 | 文件路径 |
|------|----------|
| ChannelManager | `src/qwenpaw/app/channels/manager.py` |
| BaseChannel | `src/qwenpaw/app/channels/base.py` |
| 注册表 | `src/qwenpaw/app/channels/registry.py` |
| 统一队列 | `src/qwenpaw/app/channels/unified_queue_manager.py` |
| 频道接入绑定 | `src/qwenpaw/app/workspace/service_factories.py`（`create_channel_service`） |
| 事件 schema | `src/qwenpaw/schemas.py` |

---

## 4. Skill 系统

### 4.1 功能概述

Skills 仍是 **YAML front matter + Markdown 指令**（`SKILL.md`），不是 Python 插件。2.x 存储从 1.x 的 `active_skills/` + `customized_skills/` 改为：

- **全局 Skill Pool**：`{WORKING_DIR}/skill_pool/`（`SkillPoolService`）
- **工作区技能**：`{workspace_dir}/skills/`（旧目录名 `skill` 会尝试改名为 `skills`）
- **内置技能**：`src/qwenpaw/agents/skills/<name>-{en\|zh}/`

启用集合由 `resolve_effective_skills()` 决定。`QwenPawAgent._register_skills()` 只把有效技能目录挂到 `toolkit._qp_skills`，供斜杠命令与提示注入；真正同步 / 扫描 / 安装走 `SkillService` / `SkillPoolService`。启用或安装前会跑 **SkillScanner**。

内置技能（中英各一份目录）：browser、cron、pdf / docx / xlsx / pptx、file_reader、mailbox、guidance、make-skill、make_plan、channel_message、dingtalk_channel、chat_with_agent、multi_agent_collaboration、QA_source_index。

### 4.2 调用链分析

```
启动 migrate_legacy_skills_to_skill_pool()
    └── ensure_skill_pool_initialized() / reconcile_pool_manifest()

AgentBuilder / QwenPawAgent
    ├── resolve_effective_skills(workspace)
    └── _register_skills() → toolkit._qp_skills[name] = {dir: ...}

API / CLI
    ├── SkillPoolService（池级创建、同步、自动更新）
    └── SkillService（工作区启用 / 禁用 / 列表）
            └── scan_skill_directory()  # SkillScanner
```

### 4.3 核心代码路径

| 组件 | 文件路径 |
|------|----------|
| 技能系统包 | `src/qwenpaw/agents/skill_system/` |
| 注册 / 同步 | `src/qwenpaw/agents/skill_system/registry.py` |
| 存储与清单 | `src/qwenpaw/agents/skill_system/store.py` |
| 工作区 API | `src/qwenpaw/agents/skill_system/workspace_service.py`（`SkillService`） |
| Pool API | `src/qwenpaw/agents/skill_system/pool_service.py` |
| HTTP | `src/qwenpaw/app/routers/skills.py` |
| 内置技能 | `src/qwenpaw/agents/skills/` |
| 扫描器 | `src/qwenpaw/security/skill_scanner/` |

---

## 5. 记忆：Scroll 上下文 + ReMe

### 5.1 功能概述

2.x 把「对话窗口」和「长期知识」拆开：

**Scroll**（`agents/context/scroll/`）是上下文策略，不是摘要压缩替代历史：

- 每一轮 write-through 写入持久 `conversation_history`（SQLite）
- 超过 token 阈值时保留尾部，中间轮次进入 **EvictionIndex**，可按需 **recall**
- 续写摘要只是状态缓存，不是原始记录的替代
- 启动时 `_sync_scroll_history_on_startup()` 把旧 `sessions/*.json` 回填进 history.db

**ReMe** 长期记忆：类名仍为 **ReMeLightMemoryManager**（注册键 `remelight`），实现委托 ReMe 应用/任务框架（依赖 `reme-ai==0.4.1.11`）。也可选 `none`（`NoopMemoryManager`）或插件 memory 后端。配置后端不可用时回退 `remelight`。

另有主动记忆（proactive）、`AgentMdManager`（工作区 Markdown 人设/记忆文件）。

### 5.2 调用链分析

```
Workspace 启动
    └── create_memory_manager_backend(configured, MemoryBackendContext)
            ├── remelight → ReMeLightMemoryManager
            ├── none → NoopMemoryManager
            └── 插件 memory 类型（PluginLoader types=["memory"]）

每次请求
    QwenPawAgent
        ├── ScrollContextManager.on_save()     # 落盘本轮
        ├── ScrollContextManager.compress()    # 超阈值 → EvictionIndex
        └── recall_history / recall_history_python 工具按需回放

ReMe
    ReMeLightMemoryManager
        ├── 检索 / 写入个人知识（Markdown 记忆）
        ├── Embedding / Reranker
        └── 可选 inbox / cron 任务沉淀
```

### 5.3 核心代码路径

| 组件 | 文件路径 |
|------|----------|
| Scroll 管理器 | `src/qwenpaw/agents/context/scroll/manager.py` |
| 历史存储 | `src/qwenpaw/agents/context/scroll/history.py` |
| 逐出索引 | `src/qwenpaw/agents/context/scroll/eviction_index.py` |
| 回放工具 | `src/qwenpaw/agents/context/scroll/recall_tool.py` |
| 启动同步 | `src/qwenpaw/agents/context/scroll/sync.py` |
| 公共后端 API | `src/qwenpaw/memory/__init__.py` |
| ReMeLight | `src/qwenpaw/agents/memory/reme_light_memory_manager.py` |
| 基类 / 注册表 | `src/qwenpaw/agents/memory/base_memory_manager.py` |
| ReMe 配置 | `src/qwenpaw/agents/memory/reme_config.py` |
| 主动记忆 | `src/qwenpaw/agents/memory/proactive/` |

---

## 6. 治理、沙箱与 Tool Guard

### 6.1 功能概述

1.x 的 `ToolGuardMixin` 拦截 `_reasoning` / `_acting` 已不存在。2.x 分三层：

1. **ResourceGovernor**：策略求值 `ALLOW / DENY / ASK / SANDBOX_FALLBACK`，审计，编译沙箱配置
2. **PolicyGuardedTool**：包装每个工具；`check_permissions` → `assert_policy` + 审批
3. **Sandbox**：按次创建/销毁的本地隔离（macOS seatbelt、Linux bubblewrap/landlock、Windows AppContainer / elevated / unelevated）

**ToolGuardEngine** 仍作为参数扫描器存在（规则 / 文件路径 / shell 规避），由治理策略与 `runtime/tool_guard.py` 的 `GuardedFunctionTool` 在执行前调用，而不是 Mixin。

**File Guard**（`FilePathToolGuardian`）拦截敏感路径；**SkillScanner** 在技能入库前扫描。

审批级别：`OFF / AUTO / SMART / STRICT`（`ToolExecutionLevel`），会话覆盖优先于 `agent.json`。

### 6.2 调用链分析

```
工具调用
    │
    ▼
PolicyGuardedTool.check_permissions()
    ├── 解析 approval_level（session > agent.json）
    ├── ResourceGovernor.assert_policy(ToolCallSpec)
    │       ├── GovernancePolicy 规则
    │       ├── ToolGuard 规则 / File Guard 敏感路径
    │       └── GovernanceDecision (allow/deny/ask/sandbox_fallback)
    ├── ResourceGovernor.audit()
    └── ASK → ApprovalService（频道卡片 / Console / TUI）
    │
    ▼
PolicyGuardedTool.__call__()
    └── create_sandbox(SandboxConfig) 内执行（SANDBOX_FALLBACK 或策略指定）
```

### 6.3 核心代码路径

| 组件 | 文件路径 |
|------|----------|
| ResourceGovernor | `src/qwenpaw/governance/resource_governor.py` |
| 策略 / 动作 | `src/qwenpaw/governance/policy.py` |
| PolicyGuardedTool | `src/qwenpaw/governance/tool_adapter.py` |
| 工具注册表 | `src/qwenpaw/governance/tool_registry.py` |
| 沙箱 | `src/qwenpaw/sandbox/`（`create_sandbox`） |
| ToolGuardEngine | `src/qwenpaw/security/tool_guard/engine.py` |
| 规则 / 文件 / Shell | `src/qwenpaw/security/tool_guard/guardians/` |
| GuardedFunctionTool | `src/qwenpaw/runtime/tool_guard.py` |
| 审批 | `src/qwenpaw/app/approvals/` |
| SkillScanner | `src/qwenpaw/security/skill_scanner/` |

---

## 7. MCP / Drivers

### 7.1 功能概述

1.x 的 `MCPClientManager`（`app/mcp/manager.py`）已不作为运行时。外部能力收口到 **DriverManager**：协议中立的卡片（DriverCard）、加密凭据、按次策略门控。当前落地协议是 **MCP**（`MCPDriverHandler`），stdio / HTTP / streamable HTTP。

Console 的 MCP 页面走 `MCPConfigService`，把表单映射成 DriverCard（`app/mcp/config_service.py`）。旧 MCP JSON 会经 `migrate_legacy_mcp_if_needed` 迁入 Driver 存储。

### 7.2 调用链分析

```
Workspace 启动 create_driver_service()
    ├── migrate_legacy_mcp_if_needed()
    ├── DriverManager(cards_dir, credential_store, approval_gate)
    └── register_handler_type("mcp", MCPDriverHandler)

AgentBuilder 组 toolkit
    └── MCP 能力暴露为 Driver 工具（agentscope_tool 适配）

调用
    DriverManager.invoke(capability)
        ├── 解析凭据（CredentialStore / OAuth / env）
        ├── DriverPolicy（allow / ask / deny，按 channel/user/session）
        └── MCPDriverHandler → StdIOStatefulClient / HttpStatefulClient / HttpAutoClient
```

### 7.3 核心代码路径

| 组件 | 文件路径 |
|------|----------|
| DriverManager | `src/qwenpaw/drivers/manager.py` |
| MCP Handler | `src/qwenpaw/drivers/handlers/mcp.py` |
| 卡片 / 策略 | `src/qwenpaw/drivers/contracts.py`、`policy.py` |
| 凭据 | `src/qwenpaw/drivers/credentials/` |
| Console 映射 | `src/qwenpaw/app/mcp/config_service.py` |
| 工作区装配 | `src/qwenpaw/app/workspace/service_factories.py`（`create_driver_service`） |
| HTTP | `src/qwenpaw/app/routers/mcp.py`、`mcp_oauth.py` |

---

## 8. 定时任务（Cron）与 Heartbeat

### 8.1 功能概述

每个 Workspace 一份 `CronManager`（APScheduler AsyncIOScheduler）+ `JsonJobRepository`。任务类型可发文本或走 Agent：`CronExecutor` 调用 `workspace.stream_query` 再 `send_event`。另有内部 `_heartbeat` 任务读 `HEARTBEAT.md`，以及 `SERVICE_JOB_ID_PREFIX` 服务任务（如 ReMe / Mail 相关）。

### 8.2 调用链分析

```
CronManager
    ├── JsonJobRepository（jobs.json）
    ├── AsyncIOScheduler
    └── CronExecutor
            ├── task 文本：频道直接发送
            └── task Agent：workspace.stream_query(req) → send_event

Heartbeat
    run_heartbeat_once() → workspace.stream_query(HEARTBEAT.md 查询)
```

### 8.3 核心代码路径

| 组件 | 文件路径 |
|------|----------|
| CronManager | `src/qwenpaw/app/crons/manager.py` |
| CronExecutor | `src/qwenpaw/app/crons/executor.py` |
| Heartbeat | `src/qwenpaw/app/crons/heartbeat.py` |
| 模型 | `src/qwenpaw/app/crons/models.py` |
| 存储 | `src/qwenpaw/app/crons/repo/json_repo.py` |
| HTTP | `src/qwenpaw/app/crons/api.py` |
| CLI | `src/qwenpaw/cli/cron_cmd.py` |

---

## 9. CLI / TUI / Console / 桌面

### 9.1 功能概述

- **CLI**：`qwenpaw` Click 组（`cli/main.py`）。无子命令时打开 TUI。子命令懒加载：`app`、`hub`、`tui`、`init`、`daemon`、`agents`、`channels`、`cron`、`skills`、`models`、`plugin`、`desktop`、`acp`、`doctor` 等。
- **TUI**：Textual 全屏对话（`cli/tui/`）。通过拉起 `qwenpaw acp`（当前解释器）与工作区对话，和 Console / 频道共享同一智能体。
- **ACP**：`cli/acp_cmd.py` + `agents/acp/server.py`，stdio JSON-RPC，给 TUI / Zed / OpenCode 等客户端。ACP 走完整 Workspace 生命周期。
- **Console**：`console/`（Vite + React + Ant Design），由 `qwenpaw app`（FastAPI，默认 `127.0.0.1:8088`）提供 API 与静态页。
- **桌面**：`qwenpaw desktop` 用 pywebview 包一层；另有 `src/qwenpaw/tauri/` 桌面壳。

### 9.2 调用链分析

```
qwenpaw                  → run_tui() → 子进程 python -m qwenpaw acp
qwenpaw tui              → 同上（可 --agent / --resume）
qwenpaw app              → FastAPI lifespan → WorkspaceRegistry
qwenpaw acp              → ACP Agent server → workspace.stream_query
qwenpaw desktop          → 本地 webview 打开 Console
qwenpaw hub              → Hub 控制面（见第 11 节）
```

### 9.3 核心代码路径

| 组件 | 文件路径 |
|------|----------|
| CLI 入口 | `src/qwenpaw/cli/main.py` |
| 启动服务 | `src/qwenpaw/cli/app_cmd.py` |
| TUI | `src/qwenpaw/cli/tui/app.py`、`launch.py` |
| ACP 服务 | `src/qwenpaw/agents/acp/server.py` |
| FastAPI 应用 | `src/qwenpaw/app/_app.py` |
| 路由汇总 | `src/qwenpaw/app/routers/__init__.py` |
| 前端页面 | `console/src/pages/`（Chat、Agent、Control、Settings、Inbox、Market、Hub、Coding…） |
| 桌面 CLI | `src/qwenpaw/cli/desktop_cmd.py` |

---

## 10. 插件与 PawApp

### 10.1 功能概述

插件由 `PluginLoader` 发现 `{WORKING_DIR}` 插件目录与仓库 `plugins/` 捆绑包。类型见 `PluginType`：tool、provider、hook、command、channel、memory、frontend、app、general。

**PawApp** 是带前后端入口且 `meta.pawapp` 的 `app` 型插件。SDK（`src/qwenpaw/pawapp/`）包一层 `PluginApi`：`PawApp`、`PawAppContext.chat()` → `Workspace.stream_query`。应用中心 API：`app/routers/pawapps.py`。

仓库内捆绑示例：

| 插件 | 路径 | 类型 |
|------|------|------|
| QwenPaw Creator | `plugins/apps/qwenpaw-creator/` | app（PawApp） |
| QwenPaw Data | `plugins/apps/qwenpaw-data/` | app（PawApp） |
| Agent Kanban | `plugins/apps/agent-kanban/` | app（PawApp） |
| Computer Use | `plugins/bundle/computer-use/` | bundle / tool |
| Chrome | `plugins/bundle/chrome/` | bundle |
| CloudPaw | `plugins/bundle/cloudpaw/` | bundle |
| 记忆后端等 | `plugins/memory/`、`plugins/tool/`、`plugins/channel/` | 对应类型 |

### 10.2 调用链分析

```
lifespan 后台启动
    PluginLoader.load_all_plugins(types=["channel", "memory"])  # 智能体启动前
    WorkspaceRegistry.start_all_configured_agents()
    再加载其余插件类型

PawApp 请求
    /pawapps 列表（读 PluginRegistry 或扫描 plugin.json + meta.pawapp）
    ctx.chat() / ctx.chat_stream() → workspace.stream_query()
```

### 10.3 核心代码路径

| 组件 | 文件路径 |
|------|----------|
| PluginLoader | `src/qwenpaw/plugins/loader.py` |
| 清单 / 类型 | `src/qwenpaw/plugins/architecture.py` |
| PawApp SDK | `src/qwenpaw/pawapp/app.py`、`context.py` |
| PawApp 路由 | `src/qwenpaw/app/routers/pawapps.py` |
| 工作区插件表 | `src/qwenpaw/app/workspace/workspace_plugins.py` |
| CLI | `src/qwenpaw/cli/plugin_commands.py` |

---

## 11. Hub（多用户控制面）

### 11.1 功能概述

`qwenpaw hub` 启动独立 FastAPI 控制面（`hub/control_app.py`）：管理员、租户凭据、运行时注册表、本地进程或 Docker 供给（`LocalProcessRuntimeProvisioner` / `DockerRuntimeProvisioner`）、反向代理到各 QwenPaw runtime（含 WebSocket）。默认绑回环；公网需 `--force-public` 且已初始化管理员。

### 11.2 调用链分析

```
qwenpaw hub
    └── hub.control_app
            ├── HubAuthService / 凭据库
            ├── RuntimeRegistry + RuntimeService
            ├── Local / Docker Provisioner
            └── HTTP/WS 代理到各 runtime Console
```

### 11.3 核心代码路径

| 组件 | 文件路径 |
|------|----------|
| CLI | `src/qwenpaw/cli/hub_cmd.py` |
| 控制面 | `src/qwenpaw/hub/control_app.py` |
| 运行时服务 | `src/qwenpaw/hub/service.py`、`registry.py` |
| 供给 | `src/qwenpaw/hub/local_provisioner.py`、`docker_provisioner.py` |
| 鉴权 | `src/qwenpaw/hub/auth.py` |
| Console Hub 页 | `console/src/pages/Hub/` |

---

## 12. Mail

### 12.1 功能概述

智能体可配个人邮箱。`MailMonitorService` 在工作区线程里用 IMAP IDLE（失败则 NOOP 轮询）收信：确定性规则（已读/移动/inbox 通知）之后，按 `mail.push.mode` 决定是否 `workspace.stream_query` 唤醒 Agent。MCP 侧邮件工具在独立包 `packages/qwenpawmail-mcp`（`qwenpawmail-mcp`），监控代码通过该包做 IMAP 操作。内置 Skill `mailbox-{en,zh}` 描述 Agent 如何用邮箱。

### 12.2 调用链分析

```
create_mail_monitor_service()
    └── MailMonitorService（mail.push.mode != off 且有凭据时启动）
            ├── IMAP IDLE / 轮询
            ├── 规则：mark_read / move / notify
            ├── rules_then_agent | agent_all → workspace.stream_query
            └── inbox_store.append_event("new_email")
```

### 12.3 核心代码路径

| 组件 | 文件路径 |
|------|----------|
| MailMonitorService | `src/qwenpaw/app/mail/monitor.py` |
| 访问控制 | `src/qwenpaw/app/mail/mail_access_control.py` |
| 工作区装配 | `src/qwenpaw/app/workspace/service_factories.py` |
| MCP 邮件包 | `packages/qwenpawmail-mcp/` |
| 邮箱 Skill | `src/qwenpaw/agents/skills/mailbox-zh/`、`mailbox-en/` |
| HTTP ACL | `src/qwenpaw/app/routers/mail_access_control.py` |

---

## 13. Creator、Data、Computer-use、Browser

这些不是内核里的同名 Python 包「产品模块」，而是有明确源码入口的捆绑能力。

### 13.1 QwenPaw Creator

PawApp `plugins/apps/qwenpaw-creator/`（`plugin.json` version `1.2.0`，type `app`）。后端 `backend/main.py`，前端 `ui/`，入口页 `/apps/qwenpaw-creator`。蓝图工作台、生成/剪辑相关服务都在该插件树内，不在 `src/qwenpaw/` 内核。

### 13.2 QwenPaw Data

PawApp `plugins/apps/qwenpaw-data/`。后端 `backend/main.py` 使用 `qwenpaw.pawapp.PawApp`；另需可选运行时包（`qwenpaw-data-context` 等，见插件 `plugin.json` 的 `runtime_dependencies`）。入口 `/apps/qwenpaw-data`。

### 13.3 Computer-use

- 宿主能力：`src/qwenpaw/app/computer_use/runtime.py`（Windows / macOS 原生 helper，命名管道 / 控制口）
- 工具插件：`plugins/bundle/computer-use/plugin.py` 注册 `computer_use` 工具

### 13.4 Browser-use

内核 `src/qwenpaw/browser/`：`KernelRuntime` + 子进程 plane，lifespan 里 idle watchdog。Agent 工具在 `agents/tools/browser.py`（`browser.experimental` 为真时用统一实现，否则旧 `deprecated_browser`）。Chrome 扩展桥：`plugins/bundle/chrome/`。

---

## 14. Provider 与本地模型

### 14.1 功能概述

`ProviderManager` 仍是模型入口。内置目录见 `providers/provider_catalog.py` 的 `BUILTIN_PROVIDERS`，包括 **QwenPaw Local**（`qwenpaw-local`，无 API Key）、Ollama、LM Studio、OpenRouter、GitHub Models、ModelScope、DashScope / 阿里云 CodingPlan & TokenPlan、OpenCode、Kilo、OpenAI / OpenAI Response / Azure、Anthropic、Gemini、DeepSeek、Kimi、MiniMax、智谱、SiliconFlow、火山、MiMo 等。自定义提供商与插件 `provider` 类型可扩展。

本地权重由 `LocalModelManager`（llama.cpp 封装，`src/qwenpaw/local_models/`）管理。`RoutingChatModel` 按 `AgentsLLMRoutingConfig` 在 local / cloud 槽之间选择。

### 14.2 调用链分析

```
AgentBuilder / create_model_and_formatter()
    ├── ProviderManager.get_instance()
    ├── create_chat_model(provider_id, model)
    └── 可选 RoutingChatModel / FallbackChatModel
```

### 14.3 核心代码路径

| 组件 | 文件路径 |
|------|----------|
| ProviderManager | `src/qwenpaw/providers/provider_manager.py` |
| 内置目录 | `src/qwenpaw/providers/provider_catalog.py` |
| 本地模型 | `src/qwenpaw/local_models/manager.py` |
| 路由模型 | `src/qwenpaw/agents/routing_chat_model.py` |
| 环境变量 | `src/qwenpaw/envs/registry.py` |

---

## 15. 总结

QwenPaw 2.x 的主轴是 **每智能体一个 Workspace**，请求统一进入 `Workspace.stream_query` → `Runtime` → `QwenPawAgent`，而不是 1.x 全局 `AgentRunner`。

仍存在且应对照源码阅读的主功能：

1. **工作区 / Agent OS**：资源、服务、插件注册表、检查点、Harness 后端
2. **智能体循环与 Loop**：Runtime 8 阶段、Coding / Mission / Goal / 自定义门
3. **频道**：每工作区 ChannelManager，process 绑 `stream_query`
4. **Skills**：全局 pool + 工作区 `skills/` + SkillScanner
5. **记忆**：Scroll 持久历史与按需回放 + ReMe 个人知识库
6. **治理 / 沙箱 / Tool Guard / File Guard**
7. **Drivers / MCP**
8. **Cron + Heartbeat**
9. **CLI / TUI / ACP / Console / 桌面**
10. **插件 / PawApp**（含 Creator、Data、Kanban）
11. **Hub、Mail、Computer-use、Browser**

1.x 文档中的 **AgentRunner、CoPawAgent、ToolGuardMixin、MCPClientManager、SkillsManager、active_skills/customized_skills 三目录同步、以摘要压缩为主的 MemoryManager** 已从主路径移除或更名，详见各节路径表。
