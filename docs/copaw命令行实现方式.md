# QwenPaw 命令行实现方式详解

> QwenPaw v2.2.1 · 2026-09-15  
> 对照源码：`src/qwenpaw/cli/main.py` 及 `lazy_subcommands` 引用的模块。  
> 命令示例统一使用 `qwenpaw`（`copaw` 仍是同一入口的兼容别名）。

## 1. 概述

QwenPaw CLI 是基于 **Click 8.x**（`pyproject.toml` 约束 `click>=8.1,<9`）的分组命令行应用。与 1.x 不同，顶层组不再 `cli.add_command()` 急切导入全部子命令，而是使用自定义 **`LazyGroup` + `lazy_subcommands`**：只有真正执行到某个子命令时才 `__import__` 对应模块。无子命令时默认进入终端 TUI。

### 核心技术栈

- **Click**：命令定义、全局选项、Group 嵌套
- **httpx**：与 FastAPI 后端通信（经 `runtime_api.api_client`）
- **questionary**：交互式向导（经 `cli/utils.py` 封装）
- **rich**：初始化向导等富文本面板
- **uvicorn**：`qwenpaw app` / Hub 启动 HTTP 服务
- **Textual**：`qwenpaw tui` 终端聊天界面

### 安装入口

`pyproject.toml` 的 `[project.scripts]`：

| 脚本名 | 目标 |
|--------|------|
| `qwenpaw` | `qwenpaw.cli.main:cli` |
| `copaw` | 同一对象（兼容旧命令名） |

也可 `python -m qwenpaw`（`src/qwenpaw/__main__.py` 调用同一个 `cli`）。

---

## 2. 核心入口：`main.py`

### 2.1 顶层 `cli`：`LazyGroup`

```python
@click.group(
    cls=LazyGroup,
    invoke_without_command=True,
    context_settings={"help_option_names": ["-h", "--help"]},
    lazy_subcommands={...},
)
@click.version_option(version=__version__, prog_name="QwenPaw")
@click.option("--host", default=None, help="API Host")
@click.option("--port", default=None, type=int, help="API Port")
@click.pass_context
def cli(ctx, host, port) -> None:
    """QwenPaw CLI."""
```

职责：

1. **全局 `--host` / `--port`**：写入 `ctx.obj`，供依赖 HTTP 的子命令拼 base URL。
2. **地址回退**：
   - 命令行 `--host` / `--port`
   - `read_runtime_api()`（Hub 托管运行时：`QWENPAW_RUNTIME_API_URL` + token）
   - `read_last_api()`（进程内缓存，否则 `config.json` 的 `last_api`）
   - 最终默认 `127.0.0.1:8088`
3. **无子命令时进 TUI**：`invoke_without_command=True`，且 `ctx.invoked_subcommand is None` 时 `from .tui.launch import run_tui`。`--help` 由 Click 先处理，不会误开 TUI。
4. **`qwenpaw --version`**：`prog_name="QwenPaw"`。

### 2.2 无命令时的路径语义

`LazyGroup.parse_args` 会检查第一个位置参数：若它**不是**已注册命令名，且 `_looks_like_project_path()` 为真（`.` / `..`、含 `/` `\\`、或本地存在的目录），则当作 TUI 项目目录塞进 `ctx.meta["tui_project"]`。

因此：

| 调用 | 行为 |
|------|------|
| `qwenpaw` | 打开 TUI（当前工作目录作为项目） |
| `qwenpaw .` / `qwenpaw /path/to/proj` | 打开 TUI，ACP 进程 cwd 绑到该目录 |
| `qwenpaw tui --agent NAME` | 显式 TUI，可选 agent / resume / project |
| `qwenpaw app` | 启动子命令，不进 TUI |

### 2.3 `LazyGroup` 如何加载命令

`lazy_subcommands` 映射：`命令名 → (模块路径, 属性名, 计时标签)`。

`get_command()` 流程：

1. 先查 Click 已注册（急切）命令
2. 再按表 `__import__(module_path)` + `getattr(attr_name)`
3. `_record(label, elapsed)` 记导入耗时
4. `self.add_command(cmd, cmd_name)` 缓存，下次不再导入
5. 导入失败打 error 日志并返回 `None`

`list_commands()` 合并急切命令名与 lazy 表的 key，因此 `qwenpaw --help` **不会**导入 `app_cmd` / Textual / AgentScope。

别名：`channel`→`channels`、`chat`→`chats`、`agent`→`agents`，指向同一 Group。

### 2.4 计时导入与惰性加载的目的

`main.py` 启动时**只急切导入**：

- `qwenpaw.config.utils.read_last_api`
- `qwenpaw.__version__`

其余子命令全部 lazy。`_init_timings` 可在 `qwenpaw app --log-level debug` 时由 `app_cmd.log_init_timings()` 再打一遍。

目的：**启动性能**。`qwenpaw --help`、`qwenpaw tui`、`qwenpaw env list` 不必拉起 uvicorn、FastAPI、Playwright、AgentScope 或 Textual。TUI 内部仍延迟 `import textual` / transport，注释写明是为了让 `--help` 和其他子命令保持快。

---

## 3. 全部子命令一览

下表仅含 `main.py` `lazy_subcommands` 中的名字。`cli/mission_cmd.py` 等文件存在但**未注册**，不作为公开命令。

| 命令 | 模块路径 | 导出对象 | 一句话职责 |
|------|----------|----------|------------|
| `acp` | `qwenpaw.cli.acp_cmd` | `acp_cmd` | 以 ACP agent 身份在 stdio 上跑 Agent |
| `app` | `qwenpaw.cli.app_cmd` | `app_cmd` | Uvicorn 启动 FastAPI（`qwenpaw.app._app:app`） |
| `hub` | `qwenpaw.cli.hub_cmd` | `hub_cmd` | 多用户 Hub 控制面；或 `--init-admin` 后退出 |
| `channels` / `channel` | `qwenpaw.cli.channels_cmd` | `channels_group` | 按 agent 列出/交互配置频道；`send` 走 HTTP |
| `daemon` | `qwenpaw.cli.daemon_cmd` | `daemon_group` | 本地 daemon 状态/日志/版本（不经 HTTP） |
| `chats` / `chat` | `qwenpaw.cli.chats_cmd` | `chats_group` | 会话 CRUD，HTTP `/chats` |
| `clean` | `qwenpaw.cli.clean_cmd` | `clean_cmd` | 清空工作目录内容（保留遥测标记） |
| `cron` | `qwenpaw.cli.cron_cmd` | `cron_group` | 定时任务 CRUD / pause / resume / run，HTTP `/cron` |
| `env` | `qwenpaw.cli.env_cmd` | `env_group` | 本地环境变量 list/set/delete |
| `init` | `qwenpaw.cli.init_cmd` | `init_cmd` | 交互（或 `--defaults`）初始化工作区 |
| `models` | `qwenpaw.cli.providers_cmd` | `models_group` | LLM 提供商、激活模型、本地下载 |
| `skills` | `qwenpaw.cli.skills_cmd` | `skills_group` | 工作区 / skill pool 技能管理 |
| `tui` | `qwenpaw.cli.tui.launch` | `tui_cmd` | 终端聊天 UI（Textual + ACP 子进程） |
| `uninstall` | `qwenpaw.cli.uninstall_cmd` | `uninstall_cmd` | 移除 venv/bin 与 shell PATH；`--purge` 删数据 |
| `desktop` | `qwenpaw.cli.desktop_cmd` | `desktop_cmd` | 子进程起 app，原生 webview 打开 Console |
| `update` | `qwenpaw.cli.update_cmd` | `update_cmd` | 查 PyPI 并升级当前 Python 环境中的包 |
| `shutdown` | `qwenpaw.cli.shutdown_cmd` | `shutdown_cmd` | 按端口杀后端进程（及 Vite / desktop 包装进程） |
| `auth` | `qwenpaw.cli.auth_cmd` | `auth_group` | Web 认证：重置已注册用户密码 |
| `agents` / `agent` | `qwenpaw.cli.agents_cmd` | `agents_group` | 多 agent 列表/创建/删除/互聊 |
| `plugin` | `qwenpaw.cli.plugin_commands` | `plugin` | 插件安装/列表/校验；运行中走热加载 API |
| `task` | `qwenpaw.cli.task_cmd` | `task_cmd` | 无 Web 服务的 headless 单次 ReAct 任务 |
| `doctor` | `qwenpaw.cli.doctor_cmd` | `doctor_cmd` | 只读体检；子命令 `fix` 做保守修复 |
| `auto` | `qwenpaw.cli.auto` | `auto_group` | 从 `@api_action(..., methods 含 "cli")` 生成 HTTP 客户端命令 |

---

## 4. 核心命令实现要点

### 4.1 `qwenpaw app` — FastAPI 进程

**文件**：`app_cmd.py`

主要选项：`--host`（默认 `127.0.0.1`）、`--port`（默认 `8088`）、`--reload`、`--log-level`、`--hide-access-paths`（默认隐藏 console SSE 路径）、`--workers`（**已废弃**，始终 1 worker）。

实现要点：

- `configure_server_process()`：`write_last_api(probe_host_for_bind_host(host), port)`，设 `LOG_LEVEL_ENV`，reload 时设 `QWENPAW_RELOAD_MODE=1`，安装 uvicorn access 过滤器。
- 非 loopback 且未开认证时打印安全警告。
- `uvicorn.run("qwenpaw.app._app:app", ..., workers=1, timeout_graceful_shutdown=5, ws_max_size=NM_MAX_INBOUND_BYTES)`。

这是**本地启动后端**，不是 HTTP 客户端。

### 4.2 `qwenpaw init` — 工作区向导

**文件**：`init_cmd.py`

选项：`--force`、`--defaults`（无交互）、`--accept-security`（与 `--defaults` 一起跳过安全确认，供脚本/Docker）。

流程（本地文件，不依赖已运行的 app）：

1. Rich 安全警告；必须确认（或 `--defaults --accept-security`）
2. 可选匿名遥测
3. `ensure_default_agent_exists` / 技能池迁移 / 内置 QA agent / `ensure_skill_pool_initialized`
4. heartbeat、语言、音频/转写、频道（可选）、写 `config.json`
5. `ProviderManager` 配置 LLM（无激活模型时不可跳过）
6. 技能：`--defaults` 同步 pool 且只启用新增项；交互可选 all/none/custom
7. 可选环境变量
8. 按语言拷贝 MD 模板
9. 写 `HEARTBEAT.md`

### 4.3 `qwenpaw tui` — 终端聊天

**文件**：`tui/launch.py`

选项：`--agent`、`--resume SESSION_ID`；位置参数 `project`（必须是已存在目录）。

实现要点：

- `run_tui()` 延迟导入 Textual 与 `PawApp`。
- 通过 `AcpTransport` 拉起 **当前解释器** 的 ACP 子进程：  
  `sys.executable -m qwenpaw acp --local-diagnostics`  
  不依赖 PATH 上的 `qwenpaw`。
- 项目目录会作为 ACP cwd，并带上 Coding Mode overlay 元数据。
- 退出时打印 `qwenpaw tui --resume <id>` 提示。

裸 `qwenpaw` 走同一 `run_tui(project=ctx.meta.get("tui_project"))`。

### 4.4 `qwenpaw agents` — 多 Agent

**文件**：`agents_cmd.py`

| 子命令 | 后端 |
|--------|------|
| `list` | HTTP（`agent_tools.list_agents_data`） |
| `create` | **本地**：写 `config.json` profiles + `agent.json` + workspace |
| `delete` | HTTP `DELETE /agents/{id}`，可选再删本地 workspace |
| `chat` | HTTP `/api/console/chat`（stream/final、后台 task） |

`create` 不要求 app 已启动；`list` / `delete` / `chat` 需要可达 API。`chat` 支持 `--background` / `--task-id`。

### 4.5 `qwenpaw channels` — 频道

**文件**：`channels_cmd.py`

| 子命令 | 后端 |
|--------|------|
| `list` | 本地 `load_agent_config(agent_id)` |
| `config` | 本地 questionary 向导，`save_agent_config` |
| `send` | HTTP `POST /messages/send`，头 `X-Agent-Id` |

内置交互配置器（再按 `get_available_channels()` 过滤）：iMessage、Discord、Telegram、DingTalk、Feishu、WeChat (iLink)、QQ、Console、Twilio Voice。插件频道可走 `get_configurator()` 或默认 `enabled` + `bot_prefix`。显示名表里还有 Yuanbao、Slack 等，实际出现与否取决于已安装频道/插件。

**已移除** 1.x 的 `channels install <key>` 模板生成。

1.x 把频道写在全局 `config.json`；2.x 写到 **agent 级** `agent.json`。

### 4.6 `qwenpaw cron` — 定时任务

**文件**：`cron_cmd.py`

全程 HTTP（`cli/http.py` 的 `client()`，路径带 `/api` 前缀），请求头 `X-Agent-Id`（默认 `default`）。

| 子命令 | 方法 | 路径 |
|--------|------|------|
| `list` | GET | `/cron/jobs` |
| `get` | GET | `/cron/jobs/{id}` |
| `state` | GET | `/cron/jobs/{id}/state` |
| `create` | POST | `/cron/jobs` |
| `update` | PUT | `/cron/jobs/{id}` |
| `delete` | DELETE | `/cron/jobs/{id}` |
| `pause` / `resume` / `run` | POST | `.../pause` 等 |

相对 1.x 增加 `update`。`create` 仍支持 cron 表达式、interval、一次性时间，以及 `text` / `agent` 任务类型与 channel dispatch。

### 4.7 `qwenpaw skills` — 技能

**文件**：`skills_cmd.py`

主要走本地 `SkillService` / `SkillPoolService`（工作区 `skill.json` + 共享 pool），不是 FastAPI CRUD。

| 子命令 | 作用 |
|--------|------|
| `list` | 工作区或 `--pool` |
| `config` | 交互启用/禁用 |
| `enable` / `disable` | 按名切换 |
| `info` | 详情 |
| `install` | URL 安装到 pool 或 `--agent-id` 工作区（异步 Hub client） |
| `uninstall` | 从工作区或 pool 移除 |
| `test` | 校验 SKILL.md / 依赖；命令冲突检查可打 HTTP |

### 4.8 `qwenpaw models` — 提供商

**文件**：`providers_cmd.py`（命令组名仍是 `models`）

本地 `ProviderManager`，写提供商存储，不经 HTTP。

子命令：`list`、`config`、`config-key`、`set-llm`、`add-provider`、`remove-provider`、`add-model`、`remove-model`、`download`（HuggingFace / ModelScope 整仓）、`local`、`remove-local`。

### 4.9 `qwenpaw plugin` — 插件

**文件**：`plugin_commands.py`

子命令：`install`、`list`、`info`、`uninstall`、`validate`。

`install` / `uninstall`：**app 在跑**时优先 `read_runtime_api() or read_last_api()` + `api_client` 调 `/api/plugins/install|upload|DELETE plugins/{id}` 热加载；否则拷贝到插件目录，下次启动再生效。`list` / `info` / `validate` 读本地 `plugin.json`。

### 4.10 `qwenpaw doctor` — 体检

**文件**：`doctor_cmd.py`（Group，`invoke_without_command=True`）

裸 `qwenpaw doctor` 跑只读检查：工作目录、`config.json`、agent workspace、MCP、技能布局、Console 静态资源、Web 认证、HTTP `/api/healthz`、`/api/doctor/runtime`、激活 LLM ping。`--deep` 额外做频道可达性与本地 llama.cpp 备注。

`qwenpaw doctor fix`：带备份的保守修复（`--dry-run` / `-y` / `--non-interactive` / `--only`）。扩展点：`pyproject` 的 `qwenpaw.doctor` entry-points（仍兼容 `copaw.doctor`）。

### 4.11 `qwenpaw hub` — 多用户控制面

**文件**：`hub_cmd.py`

- `--init-admin USERNAME`：本地初始化首个管理员后退出，不能与 bind 选项混用。
- 否则 `configure_server_process` 后 `run_hub_app(...)`。缺 `docker` 时提示安装 `qwenpaw[hub]`。
- 非 loopback 默认拒绝，需 `--force-public`。

### 4.12 `qwenpaw desktop` — 桌面窗口

**文件**：`desktop_cmd.py`

`get_stable_port` 占端口 → `subprocess.Popen([sys.executable, "-m", "qwenpaw", "app", ...])` → TCP 就绪后 `webview.create_window("QwenPaw Desktop", url)`。关窗后 terminate 子进程。存储目录 `WORKING_DIR/webview_data`。

### 4.13 `qwenpaw acp` — Agent Client Protocol

**文件**：`acp_cmd.py`

选项：`--agent`、`--workspace`、`--debug`、`--local-diagnostics`（给 TUI 等本地客户端暴露原始错误）、`--runtime-provider openai-env`（从 `OPENAI_*` 环境变量建临时提供商）。

`asyncio.run(run_qwenpaw_agent(...))`。Windows 上在进事件循环前尝试 `import numpy`，避免 ACP 运行时首次导入卡在 loader lock。这是 **stdio 本地 Agent**，不是 HTTP 服务。

### 4.14 `qwenpaw daemon` — 本地守护信息

**文件**：`daemon_cmd.py`

调用 `qwenpaw.runtime.commands.daemon`（与聊天里 `/daemon` 同源），**不访问 HTTP**。CLI 没有 `MultiAgentManager` 句柄，`restart` 只打印说明。

子命令：`status`、`restart`、`reload-config`、`version`（均可 `--agent-id`）、`logs`（`-n` 行数，读 `WORKING_DIR/qwenpaw.log`）。

### 4.15 其余已注册命令（摘要）

**`chats`**（`chats_cmd.py`）：HTTP `/chats` 的 list/get/create/update/delete，带 `X-Agent-Id`。

**`env`**（`env_cmd.py`）：本地 `load_envs` / `set_env_var` / `delete_env_var`。

**`auth`**（`auth_cmd.py`）：仅 `reset-password`；改本地 `auth.json` 并轮换 JWT secret。需 `QWENPAW_AUTH_ENABLED=true`。

**`task`**（`task_cmd.py`）：headless。`AgentBuilder` 在当前进程跑一轮 ReAct，写 `result.json`。`--skills-dir` 用临时 overlay workspace。不启动 uvicorn。

**`auto`**（`auto.py`）：第二层 LazyGroup。当前 `_MANAGER_CLASSES` 只有 `CronManager`。扫描 `_api_actions` 且 `methods` 含 `"cli"` 的项，生成 HTTP 客户端子命令（先 `HEAD /version` 探活）。与手写 `cron` 组并存，路径来自 `ApiActionSpec.http_path`（如 `/crons/jobs`）。

**`clean`**：删 `WORKING_DIR` 子项，保留遥测 marker；`--dry-run` / `--yes`。

**`update`**：httpx 查 PyPI；若服务在跑则需先 shutdown（非 `--yes` 时可确认强制关）。

**`shutdown`**：按端口找 LISTEN PID，并清 Vite / desktop / Windows wrapper；**进程信号，不是 HTTP shutdown API**。

**`uninstall`**：删 `WORKING_DIR/venv` 与 `bin`，清 shell PATH；`--purge` 删整个工作目录。

---

## 5. 与 FastAPI 后端的通信

### 5.1 三层地址解析

```
命令 --base-url
    → 全局 --host/--port（cli() 写入 ctx.obj）
        → read_runtime_api()     # Hub 运行时环境变量
            → read_last_api()    # 内存缓存或 config.json last_api
                → 127.0.0.1:8088
```

`app` / `hub` 启动时 `write_last_api()`，供其它 CLI 进程连接。

### 5.2 `cli/http.py`

```python
def client(base_url: str) -> httpx.Client:
    base = base_url.rstrip("/")
    if not base.endswith("/api"):
        base = f"{base}/api"
    return api_client(base)
```

所有相对路径自动挂在 `/api` 下。`resolve_base_url(ctx, base_url)` 实现上面的优先级（不含 runtime 文件；runtime 已在顶层 `cli()` 填进 `ctx.obj`）。

### 5.3 `utils/runtime_api.py`

- `read_runtime_api()`：仅当 `QWENPAW_RUNTIME_ID`、内部 token、`QWENPAW_RUNTIME_API_URL` 均为合法 loopback `http` URL 时返回 `(host, port)`。
- `api_client()`：httpx.Client，loopback 绕过环境代理；同源请求自动加 `X-QwenPaw-Runtime-Token`。

`plugin_commands`、`doctor_cmd`、`update_cmd` 等可直接用 `api_client`，不一定走 `cli/http.client`。

### 5.4 谁走 HTTP、谁纯本地

| 方式 | 命令 |
|------|------|
| HTTP 客户端 | `chats`、`cron`、`auto`、`channels send`、`agents list/delete/chat`、运行中的 `plugin install/uninstall`、`doctor` 的 health/runtime、`skills test` 的命令冲突探测、`update` 探活 |
| 本地进程 / 文件 | `app`、`hub`、`init`、`tui`/`acp`、`task`、`env`、`models`、`skills`（除上述探测）、`channels list/config`、`daemon`、`auth`、`clean`、`uninstall`、`desktop`（本地起子进程）、`shutdown`（杀进程） |

多数 HTTP 命令还带 `--agent-id` 或 `X-Agent-Id`，因为 2.x 是多 agent。

---

## 6. 公共工具模块

### 6.1 `cli/utils.py`

封装 questionary：`prompt_confirm`、`prompt_path`、`prompt_choice`、`prompt_select`、`prompt_checkbox`。业务命令不直接 import questionary。

### 6.2 `cli/process_utils.py`

进程表、Windows 快照、判断是否 QwenPaw wrapper。`shutdown_cmd` / `update_cmd` 使用。

### 6.3 计时

`main._record` / `log_init_timings`：测量 **import** 成本，不是业务耗时。

---

## 7. 架构图

```
                         用户
                           │
        qwenpaw [--host/--port] [subcommand | project-path]
                           │
                           ▼
              ┌────────────────────────────┐
              │  main.py: LazyGroup cli()  │
              │  急切: version + last_api  │
              │  无子命令 → run_tui()      │
              └────────────┬───────────────┘
                           │ get_command → __import__
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
   启动类进程           HTTP 客户端           本地文件/Agent
   app / hub            chats / cron         init / env / models
   desktop→app          agents chat/list     skills / channels config
   tui → acp stdio      plugin 热加载         task (headless ReAct)
                        doctor health        daemon / auth / plugin 离线
                           │
                           ▼
                    FastAPI (uvicorn)
                    /api/...
```

**`chats list` 调用链：**

```
$ qwenpaw --port 8088 chats list --agent-id default
        │
        ▼
cli(): ctx.obj = {host, port}     # 可能来自 runtime / last_api
        │
        ▼
LazyGroup 导入 chats_cmd
        │
        ▼
resolve_base_url → http://127.0.0.1:8088
client() → httpx base http://127.0.0.1:8088/api
GET /chats   Header: X-Agent-Id
```

---

## 8. 关键设计模式

1. **LazyGroup**：子命令模块按需导入；help 与无关命令启动成本接近「Click + 两个轻量 import」。
2. **`invoke_without_command`**：默认产品入口是 TUI，而不是打印 help。
3. **Context 传递**：`ctx.obj["host"|"port"]` + 各命令 `--base-url`。
4. **HTTP 统一前缀与运行时 Token**：`cli/http.client` + `runtime_api.api_client`。
5. **双模命令**：`plugin`（热加载 API vs 拷文件）、`agents`（create 本地 vs list HTTP）。
6. **第二层懒加载**：`auto` 的 `_LazyAutoGroup` 在第一次 `list_commands` 才导入 Manager 类。
7. **TUI 与 ACP 解耦**：UI 是 Textual；推理在 ACP 子进程，便于同一套 Agent 给 IDE/其它 ACP 客户端复用。

---

## 9. 相对 1.x CLI 文档的命令变更

### 新增

- `tui`（以及裸 `qwenpaw` / 路径参数进 TUI）
- `acp`
- `hub`
- `agents`（别名 `agent`）
- `plugin`
- `doctor`（含 `doctor fix`）
- `auth`
- `task`
- `auto`
- 别名：`channel`、`chat`
- 已有组上的新子命令：`cron update`、`channels send`、`skills` 的 enable/disable/info/install/uninstall/test、`models` 的 download/local 等

### 重命名 / 入口变化

- 产品命令从 `copaw` 改为 `qwenpaw`（`copaw` 仍指向同一 `cli`）
- `prog_name`：CoPaw → QwenPaw
- 顶层注册：`cli.add_command` 急切导入 → `LazyGroup.lazy_subcommands`
- `app` 的 ASGI 字符串：`copaw.app._app:app` → `qwenpaw.app._app:app`
- `desktop` 子进程：`copaw app` → `python -m qwenpaw app`
- 频道/技能配置从全局 config 转为 **按 agent 工作区**
- `daemon` 实现从 `app.runner.daemon_commands` 迁到 `runtime.commands.daemon`

### 删除（相对旧文档描述）

- `channels install <key>`（自定义频道模板脚手架）不再作为 CLI 子命令
- `app --workers` 不再生效（保留选项但忽略，始终 1 worker）
- 无命令时不再「只显示 Group help 后退出」——改为启动 TUI
)
