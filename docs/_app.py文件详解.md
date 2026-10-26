# `_app.py` 文件详解

> 基于 **QwenPaw v2.2.1** 源码（当前树 `__version__` 为 `2.2.2b1`）
>
> 更新日期: **2026-09-15**
>
> 真实路径: `src/qwenpaw/app/_app.py`
>
> 角色: QwenPaw 的**应用入口**——创建 FastAPI 实例、编排进程级生命周期、挂载 API 路由、托管 Console 静态文件。频道 / Cron / MCP（Driver）等**不再**在本文件直接 `start()`，而是交给每个 Agent 的 `Workspace`。

---

## 目录

1. [文件在架构中的位置](#1-文件在架构中的位置)
2. [模块级初始化（第 1–79 行）](#2-模块级初始化第-179-行)
3. [生命周期管理 `lifespan()`（第 133–722 行）](#3-生命周期管理-lifespan第-133722-行)
4. [FastAPI 实例创建与中间件（第 725–749 行）](#4-fastapi-实例创建与中间件第-725749-行)
5. [前端静态文件服务（第 752–971 行）](#5-前端静态文件服务第-752971-行)
6. [API 路由挂载](#6-api-路由挂载)
7. [组件交互关系图](#7-组件交互关系图)
8. [关键设计模式](#8-关键设计模式)
9. [启动/关闭时序](#9-启动关闭时序)
10. [常见修改场景](#10-常见修改场景)

---

## 1. 文件在架构中的位置

```
QwenPaw 分层架构（2.x）:

┌──────────────────────────────────────────────────────────┐
│  CLI Layer (qwenpaw app → uvicorn "qwenpaw.app._app:app") │
├──────────────────────────────────────────────────────────┤
│  App Layer ← _app.py 就在这一层                           │
│  ┌──────────────┬──────────────┬───────────────────────┐ │
│  │ AppService   │ Workspace    │ Provider / LocalModel │ │
│  │ Manager      │ Registry     │ Backup / Plugins      │ │
│  │ (跨 WS 协调) │ (多 Agent)   │ Browser kernel        │ │
│  └──────┬───────┴──────┬───────┴───────────┬───────────┘ │
│         │              │                   │             │
│         │     每个 Workspace 内部:          │             │
│         │     ChannelManager / CronManager │             │
│         │     DriverManager(MCP) / Chat    │             │
│         │     Memory / Watchers            │             │
├─────────┼──────────────┼───────────────────┼─────────────┤
│  Agent / Runtime Layer │                   │             │
│  Workspace.stream_query → Runtime.run()    │             │
│  （AgentScope 2.0，不再经过 AgentRunner）  │             │
├────────────────────────┼───────────────────┼─────────────┤
│  Foundation Layer      │                   │             │
│  Config / Providers / Constant / Envs / Auth             │
└──────────────────────────────────────────────────────────┘
```

`_app.py` 是 App Layer 的进程级编排文件。它不包含频道协议或 Agent 推理逻辑，只负责：

- **进程级组件**：遥测、遗留迁移、PawPort 恢复、Provider/本地模型、`AppServiceManager`、`WorkspaceRegistry`、备份、浏览器运行时、插件加载器
- **定义启动/关闭顺序**（快路径 + 后台重任务）
- **将组件注入 `app.state`** 供路由使用
- **挂载 API 路由、中间件和静态文件**

CLI 通过 `uvicorn.run("qwenpaw.app._app:app", ...)` 拉起本模块（见 `src/qwenpaw/cli/app_cmd.py`）。

---

## 2. 模块级初始化（第 1–79 行）

```python
# 第 66 行：设置日志
logger = setup_logger(os.environ.get(LOG_LEVEL_ENV, "info"))

# 第 70–75 行：补齐静态资源 MIME（尤其 Windows 缺 .js/.mjs）
mimetypes.init()
mimetypes.add_type("application/javascript", ".js")
# ... .mjs / .css / .wasm / .svg

# 第 79 行：加载持久化环境变量
load_envs_into_environ()
```

**源码中未见**：模块顶层不再创建 `AgentRunner()`，也不再创建 `agentscope-runtime` 的 `AgentApp(...)`。请求处理入口是各 `Workspace.stream_query()`。

### 逐行解析

#### `setup_logger()` — 日志初始化

```python
logger = setup_logger(os.environ.get(LOG_LEVEL_ENV, "info"))
```

- `LOG_LEVEL_ENV` 在 `constant.py` 中定义为 **`QWENPAW_LOG_LEVEL`**，默认 `"info"`
- 在模块导入时执行，确保 `uvicorn --reload` 的子进程使用相同日志级别
- `setup_logger()` 来自 `qwenpaw.utils.logging`

`src/qwenpaw/constant.py` 里环境变量一律是 `QWENPAW_*`。`EnvVarLoader.get_str/get_bool/...` 只读传入的完整变量名，**源码中未见 `COPAW_*` 回退读取**。遗留兼容只出现在数据目录：若 `QWENPAW_WORKING_DIR` 未设置且 `~/.copaw` 已存在，则 `WORKING_DIR` 固定使用该目录，否则默认 `~/.qwenpaw`。

#### MIME 类型补齐

Windows 上 Python 的 `mimetypes` 可能缺少 `.js` / `.mjs` 映射，Console 的 Vite 产物会被当成错误类型。模块导入时强制注册：

| 扩展名 | MIME |
|--------|------|
| `.js` / `.mjs` | `application/javascript` |
| `.css` | `text/css` |
| `.wasm` | `application/wasm` |
| `.svg` | `image/svg+xml` |

#### `load_envs_into_environ()` — 环境变量预加载

```python
load_envs_into_environ()
```

- 从 **`SECRET_DIR/envs.json`** 读取（默认 `{WORKING_DIR}.secret/envs.json`，可用 `QWENPAW_SECRET_DIR` 覆盖）
- 若该文件尚不存在，会尝试从包内或 `WORKING_DIR/envs.json` 迁一次（见 `envs/store.py` 的 `_LEGACY_ENVS_JSON_CANDIDATES`）
- 注入 `os.environ`；内部保留键不会注入
- **必须在 `lifespan()` 之前执行**，后续 `load_config()`、认证、频道 SDK 都可能读这些变量

#### 导入期辅助函数（第 82–130 行）

| 函数 | 作用 |
|------|------|
| `_sync_scroll_history_on_startup()` | 把旧 `sessions/*.json` 回填进各 scroll agent 的 `history.db`；失败只打日志 |
| `_browser_idle_watchdog()` | 周期性回收空闲浏览器 worker / session / wire spill |
| `_start_browser_runtime()` / `_stop_browser_runtime()` | 挂到 `app.state` 并在关闭时 cancel watchdog、丢弃全部 worker、停止 Chromium 下载 |

这些函数由 `lifespan()` 调用，不是模块导入时执行。

---

## 3. 生命周期管理 `lifespan()`（第 133–722 行）

这是整个文件最核心的部分。`lifespan()` 是 FastAPI 的 [Lifespan 协议](https://fastapi.tiangolo.com/advanced/events/)。

2.x 把启动拆成两段：

1. **快路径（目标 < 100ms）**：同步清理、迁移、进程级管理器创建；打出 `Server ready in ...s` 后即可接受 HTTP
2. **后台任务 `_background_startup()`**：插件、各 Agent Workspace、Provider 同步等重活。需要 Agent 的 API 通过 `WorkspaceRegistry.get_agent()` 等待该 Agent 就绪（`/api/healthz` 在核心 Agent 未就绪时返回 503）

### 3.1 完整流程概览

```
┌─── 快路径（yield 之前，同步/短 await） ─────────────────────────┐
│                                                                  │
│  0. add_project_file_handler(LOG_FILE_PATH)                      │
│  1. cleanup_startup_restore_artifacts()  ← 失败则 raise，阻止启动 │
│  2. auto_register_from_env() / check_proxy_config_sanity()       │
│  3. telemetry（未 opt-out 且本版本未采集过才上传）                 │
│  4. 遗留迁移:                                                    │
│       migrate_legacy_workspace_to_default_agent()                │
│       ensure_default_agent_exists()                              │
│       migrate_legacy_skills_to_skill_pool()                      │
│       ensure_qa_agent_exists()                                   │
│  5. recover_import_transactions()  ← PawPort 中断事务恢复         │
│  6. _sync_scroll_history_on_startup()                            │
│  7. ProviderManager.get_instance() / LocalModelManager.get_instance()
│  8. AppServiceManager.start() + WorkspaceRegistry                │
│       ├─ @api_action HTTP 路由 + slash specs                     │
│       ├─ HITL tool command specs                                 │
│       ├─ WorkspaceBootstrapFactory → registry._bootstrap_kwargs  │
│       └─ 预热 descriptor 缓存                                    │
│  9. BackupManager()                                              │
│ 10. TokenUsageManager.start(flush_interval=10)                   │
│ 11. 注入 app.state.*                                             │
│ 12. 浏览器 kernel + idle watchdog + prime_bridge_token()         │
│ 13. 打日志 "Server ready"，然后 create_task(_background_startup)  │
│                                                                  │
├─── 后台任务（与 yield 并行） ─────────────────────────────────────┤
│                                                                  │
│  A. PluginLoader 阶段 1：types=["channel","memory"]               │
│  B. workspace_registry.start_all_configured_agents()             │
│       ├─ 核心阶段并发: default + 内置 QA                         │
│       │    成功后 startup_ready.set()  → /api/healthz 变 200     │
│       └─ 自定义 Agent：QWENPAW_CUSTOM_AGENT_STARTUP_CONCURRENCY   │
│  C. Provider 本地模型 resume + 模型/目录同步（create_task）        │
│  D. PluginLoader 阶段 2：其余插件                                 │
│  E. 插件 Provider / 控制命令 / startup hooks                      │
│  F. ApprovalService 绑定 default 的 channel_manager              │
│  G. Skill Pool automation pipeline                               │
│                                                                  │
├─── yield（应用运行中） ───────────────────────────────────────────┤
│                                                                  │
├─── 关闭阶段（finally 块） ────────────────────────────────────────┤
│                                                                  │
│  1. 取消尚未完成的 _background_startup                            │
│  2. PORTABILITY_IMPORT_JOBS.shutdown()                           │
│  3. backup_manager.shutdown()                                    │
│  4. _stop_browser_runtime + shutdown_browser_runtime()           │
│  5. 插件 shutdown hooks                                          │
│  6. LocalModelManager.shutdown_server()                          │
│  7. AppServiceManager.stop()                                     │
│  8. multi_agent_manager.stop_all()  ← 停掉所有 Workspace          │
│       （内部停 Channel / Cron / Driver / Memory / Watcher）       │
│  9. PORTABILITY_IMPORT_JOBS.drain()                              │
│ 10. TokenUsageManager.stop() ∥ skills hub HTTP client close      │
│ 11. Windows: shutdown_all_sandboxes()                            │
│ 12. startup_display.stop()                                       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**旧文档中的「AgentRunner.start → MCP → ChannelManager → Cron → ChatManager → ConfigWatcher」已不适用于进程级 lifespan。** 这些组件改由每个 `Workspace._register_services()` 按 priority 启动（见 3.3）。

### 3.2 快路径逐段解析

#### 备份恢复残留清理（第 145–155 行）

```python
cleanup_startup_restore_artifacts()
```

扫描 `WORKING_DIR/skill_pool` 以及各 Agent `workspace_dir`，清理中断恢复留下的 `.restore_tmp` / `.restore_old`。这是快路径里**少数会阻断启动**的步骤：失败会 `raise RuntimeError`。

#### 认证引导与代理配置检查（第 157–158 行）

```python
auto_register_from_env()
check_proxy_config_sanity()
```

- `QWENPAW_AUTH_ENABLED` 为真，且尚未注册用户，同时设置了 `QWENPAW_AUTH_USERNAME` / `QWENPAW_AUTH_PASSWORD` 时，自动创建管理员（便于 Docker）
- 若 `security.allow_no_auth_hosts` 含非 loopback 地址但 `trusted_proxies` 为空，打警告

#### Telemetry（第 160–175 行）

```python
if not is_telemetry_opted_out(WORKING_DIR) and not has_telemetry_been_collected(WORKING_DIR):
    collect_and_upload_telemetry(WORKING_DIR)
```

- 进程级关闭：`QWENPAW_TELEMETRY_DISABLED=true`
- 或工作目录遥测标记文件里 `opted_out=true`
- 异常全部吞掉，只 `logger.debug`

#### 遗留配置迁移（第 177–181 行）

均来自 `src/qwenpaw/app/migration.py`：

| 函数 | 做什么 |
|------|--------|
| `migrate_legacy_workspace_to_default_agent()` | 单 Agent 时代的 `sessions/`、`chats.json`、`jobs.json` 等迁到 `workspaces/default/` |
| `ensure_default_agent_exists()` | 保证 default 档案与工作区 JSON 存在 |
| `migrate_legacy_skills_to_skill_pool()` | `active_skills/` + `customized_skills/` → 工作区 `skills/` |
| `ensure_qa_agent_exists()` | 创建内置 QA（`BUILTIN_QA_AGENT_ID`，当前为 `QwenPaw_QA_Agent_0.2`）；首次创建时会禁用遗留 `CoPaw_QA_Agent_0.1beta1` |

这些函数内部吞异常，不阻断启动。

#### PawPort 事务恢复（第 183–199 行）

```python
recovered_transactions = await recover_import_transactions(get_agent_dirs())
```

扫描各工作区 `.qwenpaw/imports/transactions/*.json`。状态为 `applying` 的计划会被打回 `ready`，避免半写入资源。失败只记日志，`recovered_transactions` 置空列表。

#### Scroll 历史回填（第 201–210 行）

```python
await _sync_scroll_history_on_startup()
```

一次性把旧会话导入 scroll 的 `history.db`。注释写明：失败则该 Agent 没有旧聊天导入，新回合仍会正常记录。

#### Provider / 本地模型单例（第 212–216 行）

```python
provider_manager = await asyncio.to_thread(ProviderManager.get_instance)
local_model_manager = await asyncio.to_thread(LocalModelManager.get_instance)
```

扫描并可能迁移已持久化的模型配置。真正的「本地模型 resume / 远程目录同步」在后台任务里启动。

#### AppServiceManager + WorkspaceRegistry（第 218–329 行）

```python
app_services = AppServiceManager()
await app_services.start()
workspace_registry = WorkspaceRegistry(app_services=app_services)
```

`AppServiceManager`（`src/qwenpaw/app/app_services/app_service_manager.py`）是**唯一的跨工作区容器**，合同冻结为三个字段：

| 字段 | 职责 |
|------|------|
| `task_tracker` | 流式运行可观测性 |
| `tool_coordinator` | HITL 工具调用协调 |
| `approval_coordinator` | HITL 审批协调 |

`WorkspaceRegistry`（`src/qwenpaw/app/workspace_registry.py`）**继承** `MultiAgentManager`：懒加载、热重载、并行启动都在父类；子类只覆盖 `_create_workspace()`，创建后注入 `bootstrap_plugins(**kwargs)` 和 `set_app_services()`。

同一段 try 里还会：

1. 用 `ManagerRegistry` 把默认工作区的 `CronManager` 注册给 `@api_action`，扫描并 `register_http_routes(app, ...)`（路由是启动时动态挂上的）
2. 收集 HITL slash command specs
3. `WorkspaceBootstrapFactory.build_bootstrap_kwargs()` 填内置 tools / commands / hooks / modes，写入 `workspace_registry._bootstrap_kwargs`
4. 线程里预热 `DEFAULT_REGISTRY`、`_default_builtin_tools()`、`get_default_user_rules()`

整段被宽泛 `except` 包住：**基础设施初始化失败只 debug 日志**，后续 `workspace_registry` 可能仍是 `None`。

#### Backup / TokenUsage / app.state（第 331–357 行）

```python
backup_manager = BackupManager()          # 只创建，作业由 API 触发
token_usage_manager.start(flush_interval=10)

app.state.multi_agent_manager = workspace_registry  # 向后兼容别名
app.state.provider_manager = provider_manager
app.state.local_model_manager = local_model_manager
app.state.backup_manager = backup_manager
app.state.plugin_loader = None            # 后台任务填入
app.state.plugin_registry = None
app.state.get_agent_by_id = _get_agent_by_id
app.state.startup_ready = asyncio.Event()
```

注释写明：`WorkspaceRegistry IS-A MultiAgentManager`，路由 / `agent_context` 仍读 `app.state.multi_agent_manager`。

#### 浏览器运行时（第 361–374 行）

```python
browser_config = load_config(get_config_path()).browser
_start_browser_runtime(app, get_default_kernel_manager(), max(0.1, browser_config.idle_ttl_seconds))
prime_bridge_token()
```

Chrome Native Messaging 桥的 token 预生成失败只 warning。

### 3.3 后台任务：Agent 真正启动的地方

`_background_startup()`（第 391–593 行）在 `asyncio.create_task` 中运行，与 `yield` 并行。

#### 插件两阶段加载

PawApps 装进 plugins 目录，走同一套 `plugin.json`（`meta.pawapp`），只在 App Center 展示。

- **阶段 1**：`load_all_plugins(..., types=["channel", "memory"])` —— 必须在 Agent 启动前注册
- **阶段 2**：其余插件；已加载的 channel 插件会被 `load_plugin` 跳过

#### `start_all_configured_agents()`

见 `src/qwenpaw/app/multi_agent_manager.py`：

- 只启动 `config.agents.profiles` 里 `enabled=True` 的档案
- **核心阶段**并发：`default` 与 `BUILTIN_QA_AGENT_ID`
- `default` 成功后调用 `on_core_ready` → `app.state.startup_ready.set()`
- `default` 失败则**跳过全部自定义 Agent**
- 自定义 Agent 并发度：`QWENPAW_CUSTOM_AGENT_STARTUP_CONCURRENCY`（默认 5，最小 1）

每个 Workspace 的 `start()`（`workspace.py`）顺序是：加载 `agent.json` → weixin→wechat 数据迁移 → `ServiceManager.start_all()`。

进程里**不再**有全局 `MCPClientManager` / `ConfigWatcher`。每个 Workspace 注册的服务（priority 升序）为：

| priority | 服务名 | 说明 |
|----------|--------|------|
| 5 | `local_workspace` | 工具路由用的本地工作区 |
| 10 | `session` | `SafeJSONSession`，目录为该 Agent 的 `sessions/`（替代旧 Runner.init） |
| 20 | `memory_manager` | 可选；可热重载复用 |
| 20 | `driver_manager` | **外部能力运行时**；MCP 作为第一种 Driver，遗留 MCP 配置迁入 DriverCard |
| 20 | `chat_manager` | `chats.json` 在该 Agent 工作区 |
| 30 | `channel_manager` | `ChannelManager.from_config(process=ws.stream_query, ...)` 再 `start_all()` |
| 40 | `cron_manager` | `jobs.json` 在该 Agent 工作区；依赖该 WS 的 `channel_manager` |
| 45 | `mail_monitor` | 条件启动（qwenpaw backend + mail.push 非 off） |
| 50 | `agent_config_watcher` | 有 channel 或 cron 才创建；热重载走 `MultiAgentManager` |
| 51 | `driver_config_watcher` | 监视工作区 `drivers/` 下的手工 DriverCard 编辑 |

频道白名单/黑名单环境变量（`config/utils.py`，非 `_app.py`）：

- `QWENPAW_ENABLED_CHANNELS`（优先）
- `QWENPAW_DISABLED_CHANNELS`

#### 后台收尾

- `provider_manager.start_local_model_resume(local_model_manager)`
- `asyncio.create_task(sync_startup_provider_models(...))`、`sync_remote_catalogs()`
- 插件 Provider 注册进 `ProviderManager`
- 插件控制命令写入 `CommandRegistry`
- 插件 `startup_hooks`
- `get_approval_service().set_channel_manager(default_agent.channel_manager)`
- `run_pool_automation_pipeline()` + inbox 通知

后台任务顶层 `except` 只记 error，**不会把已 listen 的 HTTP 打下来**。

### 3.4 关闭阶段（第 597–722 行）

与旧文档不同：没有「先停 ConfigWatcher 再停全局 ChannelManager」。进程级清理顺序见 3.1。每个 `Workspace.stop()` 再由 `ServiceManager.stop_all()` 逆序停该 Agent 的 Cron / Channel / Driver 等。

容错：多数 `stop()` 独立 `try/except`；浏览器 watchdog 用 `suppress(CancelledError)`；Windows 沙箱清理在线程里跑（`takeown` / `icacls` / `net user` 可能阻塞）。

---

## 4. FastAPI 实例创建与中间件（第 725–749 行）

```python
app = FastAPI(
    lifespan=lifespan,
    docs_url="/docs" if DOCS_ENABLED else None,
    redoc_url="/redoc" if DOCS_ENABLED else None,
    openapi_url="/openapi.json" if DOCS_ENABLED else None,
)
register_exception_handlers(app)

app.add_middleware(AgentContextMiddleware)
app.add_middleware(AuthMiddleware)
app.add_middleware(RuntimeBoundaryMiddleware)

if CORS_ORIGINS:
    app.add_middleware(CORSMiddleware, ...)
```

- `DOCS_ENABLED`：`QWENPAW_OPENAPI_DOCS`（`EnvVarLoader.get_bool`，默认 `false`）
- 异常处理：`RequestValidationError` → 422（非有限浮点可序列化）；`AgentConfigConflictError` → 409

### 中间件（Starlette：后 `add` 的更靠外）

请求进入顺序（若启用 CORS）：

```
CORSMiddleware
  → RuntimeBoundaryMiddleware
    → AuthMiddleware
      → AgentContextMiddleware
        → 路由
```

| 中间件 | 文件 | 行为 |
|--------|------|------|
| `CORSMiddleware` | FastAPI | 仅当 `QWENPAW_CORS_ORIGINS` 非空；逗号分隔 origin；`expose_headers=["Content-Disposition"]` |
| `RuntimeBoundaryMiddleware` | `app/auth.py` | 若设置了 `QWENPAW_RUNTIME_INTERNAL_TOKEN`，HTTP/WS 必须带 `x-qwenpaw-runtime-token`；否则 401 / WS 4401 |
| `AuthMiddleware` | `app/auth.py` | `QWENPAW_AUTH_ENABLED` 且已注册用户后，保护 `/api/*`。公开路径见下。Token：`Authorization: Bearer`，或 upgrade/query `token` |
| `AgentContextMiddleware` | `routers/agent_scoped.py` | 从 `/api/agents/{agentId}/...` 或 `X-Agent-Id` 写入 contextvar；可选 `X-Root-Session-Id` |

Auth 公开路径（`auth.py` 中 `_PUBLIC_PATHS` / `_PUBLIC_PREFIXES`）：

- `/api/auth/login`、`/status`、`/register`
- `/api/desktop/shutdown`、`/api/version`
- `/api/settings/language`、`/api/settings/upload-limit`
- `/api/frontend_plugin` 及其前缀（只读）
- `/assets/`、`/logo.png`、`/qwenpaw-symbol.svg`
- 非 `/api/` 前缀（静态/SPA）
- `OPTIONS`
- `security.allow_no_auth_hosts` 命中且（loopback 时）直连 peer 也是 loopback

**源码中未见** 旧文档的 `COPAW_OPENAPI_DOCS` / `COPAW_CORS_ORIGINS`。

---

## 5. 前端静态文件服务（第 752–971 行）

### 5.1 静态目录解析 `_resolve_console_static_dir()`（第 755–784 行）

```python
_CONSOLE_STATIC_ENV = "QWENPAW_CONSOLE_STATIC_DIR"
```

优先级：

| 顺序 | 来源 |
|------|------|
| 1 | 环境变量 `QWENPAW_CONSOLE_STATIC_DIR`（`EnvVarLoader.get_str`） |
| 2 | 包内 `src/qwenpaw/console/`（含 `index.html`） |
| 3 | 仓库 `console/dist`（相对包目录上两级） |
| 4 | CWD 下 `console/dist` 或 `console_dist` |
| 兜底 | CWD `console/dist`（可能不存在，会 warning） |

`index.html` 带 **禁止缓存** 头（`Cache-Control: no-cache, no-store, must-revalidate` + `Pragma` / `Expires`），避免桌面 WebView 拿着过期入口去请求旧哈希资源。`/assets` 下带内容哈希的文件仍可长期缓存。

### 5.2 根路径与 SPA

```python
@app.get("/")
def read_root():
    # 有 index.html → FileResponse（no-cache）
    # 否则 JSON 提示在 console/ 执行 npm ci && npm run build
```

静态资源与 SPA fallback **必须在所有 API 路由之后注册**：

1. `mount("/assets", StaticFiles(...))`（目录存在时）
2. `/console`、`/console/`、`/console/{full_path:path}` → 同一份 `index.html`
3. `GET /{full_path:path}` catch-all：
   - 显式拒绝 `docs` / `redoc` / `openapi.json` 以及 `api` / `api/*`（防漏匹配）
   - 拒绝 `..` 与绝对路径
   - 若 `console` 构建根下存在普通文件则直接 `FileResponse`（logo、favicon 等）
   - 否则回退 `index.html`

旧文档里独立的 `@app.get("/logo.png")` / `@app.get("/copaw-symbol.svg")` **已删除**；根目录静态文件由 catch-all 按真实文件提供。Auth 白名单仍列出 `/logo.png` 与 `/qwenpaw-symbol.svg`。

---

## 6. API 路由挂载

### 6.1 `_app.py` 直接注册的端点

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/` | Console `index.html` 或提示 JSON |
| GET | `/api/version` | `{"version": __version__}`，公开 |
| GET | `/api/doctor/runtime` | Python 可执行文件与环境摘要 |
| POST | `/api/desktop/shutdown` | 仅桌面 sidecar：`QWENPAW_DESKTOP_APP=1` 且 HMAC 比对 `QWENPAW_DESKTOP_SHUTDOWN_TOKEN`；否则 **404**（刻意伪装） |

### 6.2 聚合路由 `api_router`（`routers/__init__.py` 的 `router`）

```python
app.include_router(api_router, prefix="/api")
```

子路由（前缀均相对 `/api`）。下表只列 `APIRouter(prefix=...)` 或文件内明确路径，**不编造未读到的 handler 清单**：

| 模块 | 挂载前缀 | 功能（据 router 标签/文件职责） |
|------|----------|--------------------------------|
| `agents` | `/agents` | Agent 档案 CRUD |
| `config` | `/config` | 全局配置 |
| `console` | `/console` | Console 聊天/任务（替代旧 `/api/agent/process`） |
| `fork` | `/fork` | 会话 fork |
| `crons.api` | `/cron` | 定时任务 |
| `local_models` | `/local-models` | 本地模型 |
| `mcp_oauth` | `/mcp` | MCP OAuth |
| `mcp` | `/mcp` | MCP / Driver 相关 API |
| `messages` | `/messages` | 消息 |
| `providers` | `/models` | LLM 提供商/模型（注意前缀是 **`/models`**） |
| `chats.api` | `/chats` | 聊天元数据 |
| `market` | `/market` | 技能市场 |
| `skills` | `/skills` | 技能管理 |
| `skills_stream` | （无 prefix） | `POST /skills/ai/optimize/stream` |
| `tools` | `/tools` | 工具列表 |
| `workspace` | `/workspace` | 工作区文件 |
| `envs` | `/envs` | 环境变量 CRUD |
| `token_usage` | `/token-usage` | Token 用量 |
| `agent_stats` | `/agent-stats` | Agent 统计 |
| `auth` | `/auth` | 登录/注册/状态 |
| `files` | `/files` | 文件 |
| `settings` | `/settings` | 设置 |
| `plugins` | `/plugins` | 插件（写操作需认证） |
| `frontend_plugin` | `/frontend_plugin` | 前端插件只读 |
| `backup` | `/backups` | 备份 |
| `git` | `/workspace/git` | Git |
| `project_directory` | `/workspace/project-directory` | 编码项目目录 |
| `access_control` | `/access-control` | 频道访问控制 |
| `mail_access_control` | `/mail-access-control` | 邮件访问控制 |
| `provider_oauth` | `/providers` | 提供商 OAuth |
| `pawapps` | `/pawapps` | PawApp |
| `harnesses` | `/harnesses` | 第三方 harness |
| `checkpoints` | `/workspace/checkpoints` | 检查点 |

### 6.3 `_app.py` 额外 `include_router`（仍在 `/api` 下，除非注明）

| 路由对象 | 前缀 | 已知路径 |
|----------|------|----------|
| Chrome NM WebSocket | `/api` + `/ws` | `WS /api/ws/chrome` |
| Chrome 观测 | `/api` + `/browser/chrome` | `/status`、`/self-test`、`/traces` |
| `healthz_router` | `/api` | `GET /api/healthz`（核心 Agent 未就绪 → 503） |
| `tool_calls_router` | `/api` + `/tool-calls` | HITL 工具调用 |
| `approval_router` | `/api` + `/approval` | 审批 approve/deny 等 |
| `coding_mode_router` | `/api` + `/coding-mode` | Coding Mode |
| `loops_router` | `/api` + `/loops` | Loops |
| `create_agent_scoped_router()` | `/api` + `/agents/{agentId}` | 见下 |
| `voice_router` | **应用根**（无 `/api`） | `POST /voice/incoming`、`WS /voice/ws`、`POST /voice/status-callback` |

`register_builtin_control_links()` 在挂载 Chrome 路由之后调用，把内置 browser control link 登记进运行时（不是 HTTP 路由表本身）。

### 6.4 Agent 作用域路由

`create_agent_scoped_router()`（`routers/agent_scoped.py`）把下列子路由再挂一份到 `/api/agents/{agentId}/...`：

`agent-status`、`chats`、`config`、`cron`、`mcp`（含 oauth）、`skills`、`tools`、`workspace`、`console`、`plugins`、`checkpoints`、`portability/imports`

请求里的 `{agentId}` 由 `AgentContextMiddleware` 注入，下游用 `get_agent_for_request()` 解析到对应 `Workspace`。

### 6.5 已移除的 AgentScope Runtime 标准 API

**源码中未见** `AgentApp`、`/api/agent/process`、`/api/agent/sessions`。Console 流式对话走 `console` 路由 + `Workspace.stream_query()` → `Runtime.run()`。

### 6.6 启动期动态路由

`@api_action` 扫描到的 HTTP 方法会在 lifespan 快路径里 `app.add_api_route(...)`。具体 path 取决于各 `ManagerBase.endpoint_prefix` 与 action 的 `http_path`，**不要假设固定列表**。

---

## 7. 组件交互关系图

```
                         ┌─────────────────────────────┐
                         │     FastAPI (app)            │
                         │  中间件: CORS → RuntimeToken  │
                         │           → Auth → AgentCtx  │
                         │  app.state.* ───────────────  │
                         └──────────────┬──────────────┘
                                        │
         ┌──────────────────────────────┼──────────────────────────┐
         │                              │                          │
 ┌───────▼────────┐          ┌──────────▼──────────┐    ┌──────────▼─────────┐
 │  /api/* 路由    │          │  Voice / WS Chrome   │    │  Static + SPA      │
 │  + agent-scoped │          │  /voice/*  /api/ws   │    │  /, /assets, /console
 └───────┬────────┘          └──────────┬──────────┘    └────────────────────┘
         │                              │
         │    get_agent_for_request()   │
         └──────────────┬───────────────┘
                        │
              ┌─────────▼──────────┐
              │ WorkspaceRegistry  │  IS-A MultiAgentManager
              │ (app.state.multi_  │
              │  agent_manager)    │
              └─────────┬──────────┘
                        │ 每 Agent 一个 Workspace
          ┌─────────────┼──────────────────┐
          │             │                  │
   ┌──────▼──────┐ ┌────▼─────┐    ┌──────▼──────┐
   │ default     │ │ QA Agent │    │ custom ...  │
   │ Workspace   │ │          │    │             │
   └──────┬──────┘ └──────────┘    └─────────────┘
          │
          │ stream_query() → Runtime.run()
          │
    ┌─────┴──────────────────────────────────────────┐
    │ ServiceManager                                 │
    │  session / memory / driver(MCP) / chat         │
    │  channel_manager.start_all()                   │
    │  cron_manager.start()                          │
    │  mail_monitor / agent_config_watcher           │
    │  driver_config_watcher                         │
    └─────┬──────────────────────────────────────────┘
          │
    ┌─────▼──────┐   process = ws.stream_query
    │ 钉钉/飞书/  │
    │ Discord/QQ │
    │ ...        │
    └────────────┘

  进程级（非每 Agent）:
  AppServiceManager (task/tool/approval)
  ProviderManager / LocalModelManager
  BackupManager / TokenUsageManager
  PluginLoader（后台填入 app.state）
  Browser kernel + idle watchdog
```

---

## 8. 关键设计模式

### 8.1 依赖注入（app.state）+ 按请求解析 Agent

进程级单例进 `app.state`；**按 Agent 的状态不再挂在全局 runner 上**。路由通过 `get_agent_for_request(request)`（`app/agent_context.py`）拿到 `Workspace`，再读 `workspace.channel_manager` / `cron_manager` 等。

优先级：显式参数 → `request.state.agent_id` → `X-Agent-Id` → `config.agents.active_agent`。

### 8.2 两阶段启动（快路径 + 后台）

HTTP 先 listen，核心 Agent 在后台起来。`/api/healthz` 用 `startup_ready` Event 区分 `starting`（503）与 `ok`（200）。需要 Agent 的请求在 `get_agent()` 上等待懒加载完成。

### 8.3 Workspace 即运行时边界

旧「一个进程一个 AgentRunner」变成「一个进程多个 Workspace」。Channel / Cron / MCP Driver / Chat / Memory 的生命周期跟 Agent 走，热重载是重建或替换该 Workspace，而不是进程级 `ConfigWatcher.replace_channel()`。

### 8.4 跨工作区合同（AppServiceManager）

HITL 与任务追踪必须跨会话、跨 Agent 可见，因此单独放进进程级三字段容器。工作区私有的 ToolRegistry / Prompt / SlashCommand **禁止**塞进 `AppServiceManager`。

### 8.5 桥接（`process=ws.stream_query`）

`ChannelManager.from_config` 仍接收 `process` 回调，但回调是 **该 Workspace 的** `stream_query`，不是全局 `runner.stream_query`。频道层仍然只依赖「给请求、收流」协议。

### 8.6 容错隔离

- 遥测、PawPort 恢复、scroll 回填、插件、Skill Pool、Approval 绑定：失败不阻断 listen
- 备份残留清理失败：**阻断启动**（避免读到半恢复目录）
- 后台 Agent 启动失败：打日志并继续其他 Agent；default 失败则跳过自定义 Agent
- 关闭阶段逐步 `try/except`，Windows 沙箱清理失败只记 error

### 8.7 观察者下沉到 Workspace

| 旧（进程级） | 新（每 Agent） |
|--------------|----------------|
| `ConfigWatcher` 轮询根 `config.json` 的 channels | `AgentConfigWatcher` 看该工作区配置，经 `MultiAgentManager` 重载 |
| `MCPConfigWatcher` + `MCPClientManager` | `DriverManager` + `DriverConfigWatcher`（`workspace/drivers/`） |

根 `config.json` 的 channels 热更新路径在 `_app.py` **源码中未见**直接启动的进程级 watcher。

---

## 9. 启动/关闭时序

### 启动时序

```
时间轴 ─────────────────────────────────────────────────────────────▶

[导入]
     setup_logger / mimetypes / load_envs_into_environ
                 │
[T0 快路径]
     恢复残留清理 → 认证引导 → telemetry → 四步 migration
     → PawPort recover → scroll 回填
     → ProviderManager / LocalModelManager
     → AppServiceManager.start + WorkspaceRegistry
     → @api_action 动态路由 / bootstrap kwargs / 缓存预热
     → BackupManager / TokenUsageManager.start
     → app.state 注入 / browser watchdog
     → 日志 "Server ready in Xs (agents loading in background)"
                 │
[T1] uvicorn 已可接受 HTTP（healthz 仍可能 503）
     asyncio.create_task(_background_startup)
                 │
[T2 后台]
     插件 phase1 (channel+memory)
     start_all_configured_agents
        ├── 并发 start default + QA
        │      Workspace.start → ServiceManager:
        │         session → memory/driver/chat
        │         → channel.start_all → cron.start
        │         → mail / watchers
        ├── startup_ready.set()     ← healthz 变 200
        └── 限并发自定义 Agent
     插件 phase2 + hooks + Skill Pool
                 │
[T3] yield ── 应用运行中 ──────────────────────────────────────────▶
```

### 关闭时序

```
[S0] cancel _background_startup（若未完成）
[S1] PORTABILITY_IMPORT_JOBS.shutdown()
[S2] BackupManager.shutdown()
[S3] 停止 browser watchdog / 丢弃 workers / 停 Chromium 下载
     + agents.tools.shutdown_browser_runtime()
[S4] 插件 shutdown hooks
[S5] LocalModelManager.shutdown_server()（失败则 sync 兜底）
[S6] AppServiceManager.stop()  （tool_coordinator.shutdown 等）
[S7] MultiAgentManager.stop_all()
     └── 每个 Workspace.stop() → ServiceManager.stop_all()
           （Cron / Channel / Driver / Memory / Session ...）
[S8] PORTABILITY_IMPORT_JOBS.drain()
[S9] TokenUsageManager.stop ∥ aclose_hub_client   （gather 并行）
[S10] win32: shutdown_all_sandboxes()
[S11] startup_display.stop()
```

---

## 10. 常见修改场景

### 场景 1: 添加新的进程级生命周期组件

1. 判断它属于**快路径**（listen 前必须有）还是**后台**（可等 Agent）
2. 在 `lifespan()` 对应段落初始化，写入 `app.state`
3. 在 `finally` 里关闭（`try/except`）
4. 若每个 Agent 一份，应做成 `Workspace._register_services()` 的 `ServiceDescriptor`，而不是写进 `_app.py`

### 场景 2: 添加新的 API 路由组

1. 创建 `src/qwenpaw/app/routers/new_feature.py`
2. 在 `routers/__init__.py` 的 `router.include_router(...)` 注册（随 `prefix="/api"` 挂上）
3. 需要按 Agent 隔离时，再在 `create_agent_scoped_router()` 里 `include_router`
4. 通过 `get_agent_for_request(request)` 或 `request.app.state.xxx` 取组件
5. 若路由必须在 SPA catch-all 之前，确认 `_app.py` 里 `include_router` 发生在静态挂载之前

### 场景 3: 修改 Channel / MCP 热重载

- 频道：`src/qwenpaw/app/agent_config_watcher.py` + `Workspace` 重建路径
- MCP/Driver：`DriverConfigWatcher` + `DriverManager`（`workspace/drivers/`）
- `_app.py` 只负责启动 `WorkspaceRegistry`，不直接 `replace_channel`

### 场景 4: 调整关闭顺序

原则仍是「依赖方后关」：

- 先停导入作业和备份，避免它们还在写工作区
- 再停浏览器与插件 hook
- 再停 `AppServiceManager`（HITL 协调器）
- 再 `stop_all()` Workspace（里面才是 Channel/Cron/Driver）
- Token 用量与 hub client 无互相依赖，可并行

### 场景 5: 调试启动问题

- **healthz 一直 503**：看后台日志里 default Agent 是否 `start()` 失败；`startup_ready` 只在 default 成功时 set
- **频道不通**：确认该 Agent 工作区已 start、`QWENPAW_ENABLED_CHANNELS` / `DISABLED` 过滤、以及 `Workspace` 内 `channel_manager.start_all` 日志（不再查进程级 ChannelManager）
- **MCP 连不上**：查该工作区 `drivers/` 与 `DriverManager`，不是全局 `MCPClientManager`
- **静态文件 404**：`npm ci && npm run build`（在 `console/`），或设 `QWENPAW_CONSOLE_STATIC_DIR`
- **API 文档没有**：`QWENPAW_OPENAPI_DOCS=true`
- **401**：`QWENPAW_AUTH_ENABLED`、是否已注册、是否漏带 Bearer；桌面内部调用还要 `x-qwenpaw-runtime-token`
- **CORS**：开发时设 `QWENPAW_CORS_ORIGINS`（如 `http://localhost:5173`）；未设置则不加 CORS 中间件
- **日志级别**：`QWENPAW_LOG_LEVEL`
- **工作目录找错**：检查是否误用了遗留 `~/.copaw`；可用 `QWENPAW_WORKING_DIR` 强制指定

### 场景 6: 环境变量速查（与本文件相关）

均以 `constant.py` / `_app.py` / `auth.py` / `tauri/env.py` 为准。**源码中未见 `COPAW_*` 同义读取。**

| 变量 | 作用 |
|------|------|
| `QWENPAW_WORKING_DIR` | 覆盖数据根；未设且存在 `~/.copaw` 则用遗留目录 |
| `QWENPAW_SECRET_DIR` | 密钥目录（`auth.json`、`envs.json`），默认 `{WORKING_DIR}.secret` |
| `QWENPAW_LOG_LEVEL` | 日志级别 |
| `QWENPAW_OPENAPI_DOCS` | 暴露 `/docs` `/redoc` `/openapi.json` |
| `QWENPAW_CORS_ORIGINS` | CORS 白名单 |
| `QWENPAW_CONSOLE_STATIC_DIR` | Console 静态根 |
| `QWENPAW_AUTH_ENABLED` / `_USERNAME` / `_PASSWORD` | 认证开关与自动注册 |
| `QWENPAW_RUNTIME_INTERNAL_TOKEN` | 运行时边界 token |
| `QWENPAW_TELEMETRY_DISABLED` | 关闭遥测 |
| `QWENPAW_CUSTOM_AGENT_STARTUP_CONCURRENCY` | 自定义 Agent 启动并发 |
| `QWENPAW_ENABLED_CHANNELS` / `QWENPAW_DISABLED_CHANNELS` | 频道过滤（`config/utils.py`） |
| `QWENPAW_DESKTOP_APP` / `QWENPAW_DESKTOP_SHUTDOWN_TOKEN` | 桌面 sidecar 优雅退出 |
| `QWENPAW_BACKUP_DIR` | 备份目录，默认 `{WORKING_DIR}.backups` |

---

## 附录：`app.state` 在 lifespan 中赋值的字段

| 属性 | 何时赋值 | 用途 |
|------|----------|------|
| `app_services` | 快路径成功时 | `AppServiceManager` |
| `workspace_registry` | 同上 | 与 `multi_agent_manager` 同一对象 |
| `manager_registry` | `@api_action` 成功时 | 动态 HTTP / slash |
| `multi_agent_manager` | 快路径末尾（即使 registry 为 None 也会赋） | 路由兼容名 |
| `provider_manager` | 快路径 | 模型提供商 |
| `local_model_manager` | 快路径 | 本地模型进程 |
| `backup_manager` | 快路径 | 备份作业 |
| `plugin_loader` / `plugin_registry` | 先 `None`，后台填入 | 插件 |
| `get_agent_by_id` | 快路径 | 按 id 取 Workspace |
| `startup_ready` / `startup_time` | 快路径 | healthz |
| `browser_kernel` / `browser_watchdog` | 快路径 | 浏览器 worker |
| `uvicorn_server` | **源码中未见在 `_app.py` 赋值**；桌面关闭接口会读它 | Tauri sidecar 停 uvicorn |
