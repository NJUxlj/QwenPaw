# QwenPaw 项目架构文档

> 版本：v2.2.1（源码级剖析） | 日期：2026-09-15
>
> 本文面向开发者，落到真实模块 / 类 / 路径。官网 `website/public/docs/architecture.zh.md` 讲的是相对稳定的 Agent OS 概念；本文以仓库源码为准。

---

## 目录

1. [项目概述](#1-项目概述)
2. [技术栈](#2-技术栈)
3. [整体架构](#3-整体架构)
4. [目录结构](#4-目录结构)
5. [核心模块详解](#5-核心模块详解)
   - 5.1 [Agent OS：工作区与注册表](#51-agent-os工作区与注册表)
   - 5.2 [Runtime 与 QwenPawAgent](#52-runtime-与-qwenpawagent)
   - 5.3 [频道系统（Channels）](#53-频道系统channels)
   - 5.4 [Skills 系统](#54-skills-系统)
   - 5.5 [定时任务系统（Crons）](#55-定时任务系统crons)
   - 5.6 [Drivers（MCP 等外部能力）](#56-driversmcp-等外部能力)
   - 5.7 [配置管理](#57-配置管理)
   - 5.8 [LLM 提供商系统（Providers）](#58-llm-提供商系统providers)
   - 5.9 [本地模型支持](#59-本地模型支持)
   - 5.10 [记忆、Scroll 与上下文](#510-记忆scroll-与上下文)
   - 5.11 [Loop Engineering 与 Agent Mode](#511-loop-engineering-与-agent-mode)
   - 5.12 [治理、沙箱与安全](#512-治理沙箱与安全)
   - 5.13 [插件与 PawApp](#513-插件与-pawapp)
   - 5.14 [Hub、Harness 与 ACP](#514-hubharness-与-acp)
   - 5.15 [Browser-use 与 Computer-use](#515-browser-use-与-computer-use)
6. [API 接口设计](#6-api-接口设计)
7. [数据流](#7-数据流)
8. [数据存储方案](#8-数据存储方案)
9. [前端架构](#9-前端架构)
10. [CLI 系统](#10-cli-系统)
11. [部署方案](#11-部署方案)
12. [扩展机制](#12-扩展机制)
13. [1.x → 2.x 映射](#13-1x--2x-映射)

---

## 1. 项目概述

QwenPaw 是一个**本地优先的个人 AI 助手 / Agent OS**。一次安装可托管多个互相隔离的智能体；每个智能体对应一个 **Workspace**（磁盘目录 + 一组运行时服务）。请求经 **Runtime** 编排，在治理与沙箱之下把模型、工具、记忆、Skills 和 Drivers 串起来。内核是进程内的 **AgentScope 2.0**（智能体循环、会话、事件流、工具层），QwenPaw 是其上的操作系统层。

入口包括 IM 频道、Web Console、终端 TUI 和 CLI。数据默认落在用户本机文件系统（`WORKING_DIR`，通常为 `~/.qwenpaw`；若存在遗留 `~/.copaw` 则继续使用该目录）。

**核心特性：**

| 特性 | 说明 |
|------|------|
| 多 Agent 工作区 | `WorkspaceRegistry` 懒加载多个 `Workspace`；文件、记忆、频道、定时任务按智能体隔离 |
| 多频道入口 | 钉钉、飞书、QQ、Discord、Telegram、Slack、微信、企业微信、iMessage、Console 等（见 §5.3） |
| Skills | `SKILL.md` 指令注入；全局 `skill_pool/` + 每工作区 `skills/` |
| Drivers | 协议中立的外部能力层；当前落地的协议是 **MCP**（凭据加密 + 调用时策略门控） |
| Loop Engineering | Coding / Mission / Goal / 自定义循环模板 + 可组合 StopGate |
| Scroll 上下文 | 每轮持久化；滚出窗口的轮次建索引，按需 `recall_history`，不以摘要替换原文 |
| ReMe Light | `ReMeLightMemoryManager`（ReMe 0.4）把对话沉淀为可编辑 Markdown 知识库 |
| 插件 / PawApp | `PluginLoader` 扩展工具、频道、记忆后端、前端、HTTP；PawApp 是带 `meta.pawapp` 的插件 |
| TUI | 无子命令时 `qwenpaw` 打开 Textual 全屏对话，与 Console / 频道共享同一智能体 |
| Hub | `qwenpaw hub` 多用户运行时控制面（本地 / Docker 供给） |
| 安全 | Tool Guard、File Guard、Skill Scanner、OS 沙箱（macOS seatbelt / Linux bwrap·landlock / Windows AppContainer） |

---

## 2. 技术栈

版本以 `pyproject.toml` 与 `console/package.json` 为准。

### 后端

| 技术 | 版本/说明 |
|------|-----------|
| Python | `>=3.11,<3.14` |
| FastAPI | Web / REST；应用入口 `src/qwenpaw/app/_app.py` |
| AgentScope | `agentscope[model-ollama]==2.0.7.post1`，进程内 ReAct / Agent / Toolkit |
| Uvicorn | `>=0.40.0`，ASGI |
| APScheduler | `>=3.11.2,<4`，Cron |
| Pydantic | v2，`Config` / Agent 配置 / API 模型 |
| ReMe-AI | `reme-ai==0.4.1.11`（另有 `reme-auto-fin`、`reme-daily-paper`） |
| MCP SDK | `mcp>=1.28,<2.0` |
| Playwright | `>=1.49.0`，浏览器自动化 |
| Textual | `>=8.2.8,<8.2.9`，捆绑 TUI |
| Transformers | `>=4.30.0`，Tokenizer / Embedding |
| python-dotenv | `>=1.0.0` |
| OpenAI SDK | `>=2.0.0,<=2.33.0` |
| agent-client-protocol | `>=0.9.0,<0.11.0`，ACP |
| python-lsp-server | Coding Mode 的 Python LSP 回退 |
| ast-grep-cli | Coding Mode `ast_search` |

**已移除：** agentscope-runtime（1.x 的 `AgentApp` / `Runner` 基类）。会话与流式查询由 `Workspace.stream_query()` → `Runtime.run()` 完成。

### 前端（Console）

| 技术 | 版本/说明 |
|------|-----------|
| React | ^18 |
| TypeScript | ~5.8.3 |
| Vite | 6.4.1 |
| Ant Design | 5.29.3 |
| antd-style | ^3.7.1 |
| React Router | ^7.13.0 |
| i18next | ^25.8.4 |
| ahooks | ^3.9.6 |
| zustand | ^5.0.3 |
| @agentscope-ai/chat | `1.1.73-beta.*` |
| @agentscope-ai/design | 1.0.29 |
| @agentscope-ai/icons | 1.0.67 |
| lucide-react | ^0.562.0 |
| monaco-editor | 0.55.1 |
| @tauri-apps/api | ^2.10.1（桌面端） |

### 频道集成 SDK

| 频道 | 依赖 |
|------|------|
| 钉钉 | `dingtalk-stream>=0.24.3`，`alibabacloud-dingtalk` 等 |
| 飞书 | `lark-oapi>=1.5.3` |
| Discord | `discord-py>=2.3` |
| Telegram | `python-telegram-bot>=20.0` |
| Slack | `slack-bolt>=1.22.0` |
| 企业微信 | `wecom-aibot-python-sdk==1.0.2` |
| Matrix | `matrix-nio>=0.25.0` |
| MQTT | `paho-mqtt>=2.0.0` |
| Voice / Twilio | `twilio>=9.10.2` |
| SIP | 可选 extra `qwenpaw[sip]`（`pyVoIP` 等） |
| QQ / OneBot / 微信 / 元宝 / 小艺 | 仓库内实现（WebSocket / HTTP / 协议编解码） |
| iMessage | macOS 原生（SQLite 读取 + AppleScript 发送） |

### 可选 extra

| extra | 用途 |
|-------|------|
| `local` | `huggingface_hub`，本地模型目录 |
| `hub` | `docker>=7.1,<8`，Hub Docker 供给 |
| `whisper` | 语音转写 |
| `codex` / `qoder` | 第三方 Harness |
| `qwenpaw-data` | QwenPaw Data 组件 |
| `sip` / `sip-livekit` | SIP / LiveKit 语音 |
| `full` | 上述组合 |

本地推理后端：`src/qwenpaw/local_models/llamacpp.py` 的 `LlamaCppBackend`；Ollama / LM Studio 走 Provider 目录（`PROVIDER_OLLAMA`、`PROVIDER_LMSTUDIO`）。

---

## 3. 整体架构

2.x 分层：**入口 → Runtime 调度 → Workspace（资源 / 治理 / 沙箱）→ Drivers → AgentScope 2.0**。

```
┌──────────────────────────────────────────────────────────────────────────┐
│ 入口                                                                      │
│  频道(IM)  │  Web Console  │  TUI(Textual)  │  CLI / desktop / hub       │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────────────┐
│ FastAPI 进程（src/qwenpaw/app/_app.py lifespan）                           │
│                                                                          │
│  AppServiceManager          WorkspaceRegistry (IS-A MultiAgentManager)   │
│  ├─ ToolCoordinator         ├─ 懒加载 / 并行启动 / 热重载                  │
│  ├─ ApprovalCoordinator     └─ 每个 agent_id → Workspace                 │
│  └─ task_tracker                                                         │
│                                                                          │
│  PluginLoader  │  ProviderManager  │  LocalModelManager  │  BackupMgr    │
│  Browser kernel idle watchdog                                            │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │ 路由到目标工作区
┌───────────────────────────────▼──────────────────────────────────────────┐
│ Workspace（每智能体一份隔离空间）                                          │
│                                                                          │
│  资源                          治理                         沙箱          │
│  ├─ 记忆 (ReMe Light / none)   ResourceGovernor             create_sandbox│
│  ├─ Scroll history.db          allow / deny / ask / sandbox  per-tool-call│
│  ├─ Skills / md_files          Tool Guard / Skill Scanner                │
│  ├─ ChannelManager             加密凭据 credentials.yaml                  │
│  ├─ CronManager                                                              │
│  ├─ ChatManager + SafeJSONSession                                            │
│  └─ WorkspacePlugins (tools/hooks/commands/modes)                            │
│                                                                          │
│  stream_query() → Runtime.run()  或  HarnessRuntime (codex/qoder)        │
└──────────────┬─────────────────────────────────────────┬─────────────────┘
               │                                         │
┌──────────────▼──────────────┐            ┌─────────────▼─────────────────┐
│ Drivers（DriverManager）     │            │ Runtime 8 阶段 + AgentBuilder  │
│  MCPDriverHandler（落地）    │            │ QwenPawAgent (AgentScope Agent)│
│  DriverCard YAML + 策略门控  │            │ Loop gates / Scroll / Toolkit  │
│  ACP/A2A：见 §5.6 / §5.14    │            └────────────────────────────────┘
└─────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────────────┐
│ 基建 · AgentScope 2.0（库，进程内）                                        │
│  Agent 循环 · Toolkit · 消息/状态序列化 · 流式事件                          │
└──────────────────────────────────────────────────────────────────────────┘
```

依赖方向（不可逆）：`cli → app → agents → config / providers / constant`。`agents/__init__.py` 用 `__getattr__` 懒加载 `QwenPawAgent`，避免 CLI 拉起 AgentScope / Playwright。

---

## 4. 目录结构

```
CoPaw/   # 仓库根；PyPI 包名 qwenpaw
├── src/qwenpaw/                 # Python 核心
│   ├── __init__.py
│   ├── __version__.py           # 包版本（当前源码为 2.2.2b1）
│   ├── __main__.py              # python -m qwenpaw
│   ├── constant.py              # WORKING_DIR、心跳、路径常量
│   ├── agents/                  # 智能体实现
│   │   ├── react_agent.py       # QwenPawAgent（Agent + CodingModeMixin）
│   │   ├── model_factory.py
│   │   ├── prompt.py            # 工作区 Markdown 提示词辅助
│   │   ├── skill_system/        # SkillPoolService / SkillService
│   │   ├── skills/              # 内置 SKILL.md（按语言后缀 -zh/-en）
│   │   ├── tools/               # @tool_descriptor 内置工具
│   │   ├── memory/              # ReMeLightMemoryManager 等
│   │   ├── context/scroll/      # ScrollContextManager
│   │   ├── acp/                 # ACP 客户端/服务端（懒加载）
│   │   ├── md_files/            # AGENTS.md / SOUL.md 等模板
│   │   └── utils/
│   ├── app/                     # FastAPI 应用
│   │   ├── _app.py              # lifespan + 静态 Console
│   │   ├── workspace_registry.py
│   │   ├── multi_agent_manager.py
│   │   ├── workspace/           # Workspace、ServiceManager、bootstrap
│   │   ├── app_services/        # 跨工作区协调器
│   │   ├── channels/            # BaseChannel + 各频道实现
│   │   ├── crons/               # CronManager / CronExecutor / heartbeat
│   │   ├── chats/               # ChatManager、SafeJSONSession
│   │   ├── routers/             # REST
│   │   ├── mcp/                 # MCP API schema / config_service（运行时走 Drivers）
│   │   ├── computer_use/        # 桌面 Computer-use 宿主能力
│   │   ├── approvals/
│   │   └── migration.py         # 1.x 工作区 / Skills 迁移
│   ├── runtime/                 # Runtime、AgentBuilder、8 阶段钩子
│   ├── drivers/                 # DriverManager、MCP handler、凭据
│   ├── loop/                    # StopGate 目录与编译器
│   ├── modes/                   # coding / mission / goal / custom_loop / default
│   ├── governance/              # ResourceGovernor、PolicyGuardedTool
│   ├── sandbox/                 # 各 OS 沙箱实现
│   ├── security/                # tool_guard、skill_scanner、secret_store
│   ├── plugins/                 # PluginLoader / PluginRegistry
│   ├── pawapp/                  # PawApp SDK（薄封装插件系统）
│   ├── hub/                     # 多用户 Hub 控制面
│   ├── harnesses/               # Codex / Qoder 外部智能体适配
│   ├── hooks/                   # Runtime 生命周期钩子实现
│   ├── providers/               # ProviderManager + 内置目录
│   ├── local_models/            # LlamaCppBackend、下载管理
│   ├── config/                  # Config / AgentProfileConfig / watcher
│   ├── browser/                 # 统一 Browser-use 运行时（experimental）
│   ├── cli/                     # click LazyGroup + TUI
│   ├── envs/                    # 环境变量持久化
│   ├── backup/                  # 备份 / 恢复
│   ├── market/                  # 插件市场
│   ├── portability/             # PawPort 导入
│   ├── checkpoints/
│   ├── token_usage/
│   ├── tokenizer/
│   └── utils/
├── console/                     # React 控制台（构建产物不提交 git）
├── website/public/docs/         # 用户文档站点
├── deploy/                      # Dockerfile、supervisord、entrypoint
├── packages/qwenpawmail-mcp/    # 邮件 MCP 附属包
├── pyproject.toml
└── docs/architecture.md         # 本文
```

---

## 5. 核心模块详解

### 5.1 Agent OS：工作区与注册表

**工作区是隔离的基本单位。** 一次安装多个智能体，每个对应 `config.agents.profiles[agent_id].workspace_dir`（默认 `{WORKING_DIR}/workspaces/<id>`）。首次被请求时才创建并 `start()`。

#### WorkspaceRegistry

`src/qwenpaw/app/workspace_registry.py`：

```python
class WorkspaceRegistry(MultiAgentManager):
    def _create_workspace(self, agent_id, workspace_dir) -> Workspace:
        workspace = Workspace(...)
        workspace.bootstrap_plugins(**self._bootstrap_kwargs)
        workspace.set_app_services(self.app_services)
        return workspace
```

`lifespan` 把同一实例同时挂到 `app.state.workspace_registry` 与 `app.state.multi_agent_manager`（注释写明 IS-A，兼容旧路由）。

#### MultiAgentManager

`src/qwenpaw/app/multi_agent_manager.py`：懒加载、`get_agent()` 去重、自定义智能体并行启动（信号量 `CUSTOM_AGENT_STARTUP_CONCURRENCY`，默认 5）、`reload_agent()` 零停机替换（配置世代计数防止过期快照写回）。

启动时 `start_all_configured_agents()`：先起核心智能体（含 `default`），再并行其余；内置 QA 档案 id 为 `QwenPaw_QA_Agent_0.2`。

#### Workspace

`src/qwenpaw/app/workspace/workspace.py`。每个实例通过 `ServiceManager` 登记服务：

| 服务 | 工厂 / 类 | 职责 |
|------|-----------|------|
| session | `SafeJSONSession`（`app/chats/session.py`） | 会话状态落盘 |
| memory_manager | `create_memory_manager_backend` | ReMe Light 或 `none` |
| driver_manager | `create_driver_service` | MCP Drivers |
| chat_manager | `create_chat_service` | 聊天列表 `chats.json` |
| channel_manager | `create_channel_service` | 频道队列与收发 |
| cron_manager | `CronManager` | 每工作区一份 jobs |
| mail_monitor | `create_mail_monitor_service` | 邮件推送 |
| agent_config_watcher | `create_agent_config_watcher` | `agent.json` 热重载 |

`Workspace.stream_query(request)`：

- `backend == "qwenpaw"`（默认）→ `Runtime(workspace, app_services).run(request)`
- 其它 backend（`codex` / `qoder`）→ `HarnessRuntime.stream(...)`

#### AppServiceManager

`src/qwenpaw/app/app_services/`。**仅**跨工作区三件套：`task_tracker`、`ToolCoordinator`（HITL 工具协调）、`ApprovalCoordinator`。工具注册表、钩子、斜杠命令、Mode 一律在 `Workspace.plugins`（`WorkspacePlugins`），禁止放进 AppServiceManager。

#### 启动顺序（`_app.py` lifespan）

快路径（目标尽快听 HTTP）：

1. 清理备份恢复残留；环境自动注册；代理配置检查
2. 遗留工作区 / Skills 迁移；确保 `default` 与 QA Agent 存在
3. PawPort 中断事务恢复；Scroll 历史回填 `sync_all_scroll_agents`
4. `ProviderManager` / `LocalModelManager`
5. `AppServiceManager.start()` + `WorkspaceRegistry`
6. `@api_action` HTTP / 斜杠命令收集；HITL 工具命令
7. `WorkspaceBootstrapFactory.build_bootstrap_kwargs()` 注入内置工具 / 命令 / 钩子 / Mode
8. `BackupManager`、`TokenUsageManager`、Browser kernel watchdog
9. `yield`（后台任务继续加载智能体与插件）

后台：`PluginLoader` **phase 1**（`channel` + `memory`）→ `start_all_configured_agents` → Provider 模型同步 → **phase 2** 其余插件 → 插件启动钩子 → Approval 绑定默认频道 → Skill Pool 自动化。

停止（`finally`）：取消后台启动 → PawPort 导入任务 → Backup → Browser → 插件 shutdown 钩子 → 本地模型服务 → `AppServiceManager.stop()` → `MultiAgentManager.stop_all()` → TokenUsage / Hub HTTP 客户端 → Windows 沙箱残留清理。

---

### 5.2 Runtime 与 QwenPawAgent

#### Runtime 八阶段

`src/qwenpaw/runtime/runtime.py`、`phases.py`。一次 `AgentRequest` 对应一次 `run()`，产出与旧 `Runner.stream_query` 兼容的 SSE 信封。

固定步骤夹在阶段之间：**斜杠命令分发**、**AgentBuilder.build**、**AgentExecutor.run**。

| Phase | 含义 |
|-------|------|
| `PRE_DISPATCH` | 规范化之后、斜杠分发之前 |
| `POST_DISPATCH` | 未命中斜杠命令 |
| `PRE_AGENT_BUILD` | 组 Agent 前（session.load 等） |
| `POST_AGENT_BUILD` | Agent 已构造，注入 Mode 上下文 |
| `PRE_EXECUTE` | bootstrap / 提示词刷新 |
| `POST_RESPONSE` | session.save、cron 回写 |
| `ON_ERROR` | 异常规范化、取消信封 |
| `FINALLY` | 幂等清理（关 MCP 作用域、复位 ContextVars） |

这些是 **Runtime 编排钩子**（`runtime/hooks.py` 的 `HookRegistry`），与 AgentScope `middleware`（包一层 `reply`）正交。

实现目录：`src/qwenpaw/hooks/`（`BootstrapHook`、`MediaProcessHook`、`CronContextHook`、`ErrorNormalizeHook`、session / skill_env / langfuse 等）。旧的 `agents/hooks/memory_compaction.py` 已不存在，长上下文由 Scroll 承担。

#### AgentBuilder

`src/qwenpaw/runtime/builder.py`。**每个请求**组装 `QwenPawAgent`：工具来自 `QwenPawLocalWorkspace.list_tools()`，系统提示来自 `PromptManager`，模型来自 `model_factory`。Agent **不再**在 `__init__` 内部自己建 Toolkit。

#### QwenPawAgent

`src/qwenpaw/agents/react_agent.py`：

```python
class QwenPawAgent(CodingModeMixin, Agent):
    def __init__(self, *, name, model, system_prompt, toolkit,
                 react_config, middlewares, agent_config,
                 workspace_dir=None, request_context=None,
                 offloader=None, context_config=None,
                 context_manager=None, effective_skills=None,
                 governor=None):
        ...
```

继承 AgentScope 2.0 的 `Agent`（不是 1.x 的 `ReActAgent` 子类直接堆工具）。`context_manager` 可注入 `ScrollContextManager`。工具权限走 `PolicyGuardedTool` / `governor`。

懒加载仍在 `agents/__init__.py`：`from qwenpaw.agents import QwenPawAgent` 才导入 `react_agent`。

#### 系统提示词

`runtime/prompt_contributors.py` 默认文件顺序：`AGENTS.md` → `SOUL.md` → `PROFILE.md`。优先读工作区，回退 `agents/md_files/<lang>/`。`PromptManager` 按 contributor `priority` 拼接。

#### 内置工具（`agents/tools/`）

通过 `@tool_descriptor` 在 import 时登记；`discover_builtin_tool_funcs()` 收集。`tools/__init__.py` 显式 import 以触发装饰器：

| 工具 | 文件 |
|------|------|
| `read_file` / `write_file` / `edit_file` / `append_file` | `file_io.py` |
| `grep_search` / `glob_search` | `file_search.py` |
| `execute_shell_command` | `shell.py` |
| `send_file_to_user` | `send_file.py` |
| `web_search` / `web_fetch` | `web_search.py` |
| `desktop_screenshot` | `desktop_screenshot.py` |
| `view_image` / `view_video` | `view_media.py` |
| `get_current_time` / `set_user_timezone` | `get_current_time.py` |
| `get_token_usage` | `get_token_usage.py` |
| `list_agents` / `chat_with_agent` / `submit_to_agent` / `check_agent_task` / `spawn_subagent` | `agent_management.py` |
| `delegate_external_agent` | `delegate_external_agent.py` |
| `ast_search` | `ast_tool.py` |
| `run_tool_batch` | `run_tool_batch.py` |
| `activate_f1_exploration_mode` | `mail_f1_tool.py` |
| `browser` | `browser.py`（`browser.experimental=true`）或 `deprecated_browser/browser_control.py` |
| `migration_compat_*` | `migration_compatibility.py`（PawPort） |

Coding Mode 额外：`lsp`（`lsp_tool.make_lsp_tool`）、`ast_search` 由 `modes/coding/mixin.py` 的 `collect_coding_tools()` 注入。Scroll 的 `recall_history` 在上下文策略里注册，治理类型为 `internal`。

#### 斜杠命令

`runtime/slash_command_registry.py` + `runtime/builtin_commands.py`；插件与 `@api_action` 可追加。命中则跳过 Agent。旧 1.x 的 `CommandHandler`（`/compact` 等）已拆到 Runtime 命令注册表与 Scroll 策略。

---

### 5.3 频道系统（Channels）

模式未变：**注册表 + 队列 + 消费**，但队列实现换成按会话的按需消费者。

#### BaseChannel

`src/qwenpaw/app/channels/base.py`：`channel`、`uses_manager_queue`、`from_config()`、`build_agent_request_from_native()`、`send()` / `send_event()`、`start()` / `stop()`、防抖合并。

#### 注册表

`src/qwenpaw/app/channels/registry.py` 的 `_BUILTIN_SPECS`（**console 为必加载**；其它依赖失败只打 debug，不拖垮 CLI）：

| key | 类 |
|-----|-----|
| `console` | `ConsoleChannel` |
| `dingtalk` | `DingTalkChannel` |
| `feishu` | `FeishuChannel` |
| `qq` | `QQChannel` |
| `discord` | `DiscordChannel` |
| `imessage` | `IMessageChannel` |
| `telegram` | `TelegramChannel` |
| `mattermost` | `MattermostChannel` |
| `mqtt` | `MQTTChannel` |
| `matrix` | `MatrixChannel` |
| `slack` | `SlackChannel` |
| `voice` | `VoiceChannel` |
| `sip` | `SIPChannel` |
| `wecom` | `WecomChannel` |
| `xiaoyi` | `XiaoYiChannel` |
| `yuanbao` | `YuanbaoChannel` |
| `wechat` | `WeChatChannel` |
| `onebot` | `OneBotChannel` |

`get_channel_registry()` = 内置 + `PluginRegistry.get_registered_channels()`。**不再扫描 `custom_channels/` 目录。** 配置侧 `ChannelConfig`（`config/config.py`）字段与上表一致；遗留 `weixin` 键会迁到 `wechat`。

频道过滤：`QWENPAW_ENABLED_CHANNELS` / `QWENPAW_DISABLED_CHANNELS`（Docker 默认禁用 `imessage`）。

#### ChannelManager 与队列

`manager.py` + `unified_queue_manager.py`：

- 每个 `QueueKey` 队列 `maxsize=1000`（不是每频道共享一条）
- `QueueKey = (channel_id, session_id, priority_level)`
- **按需创建 consumer Task**，无固定「每频道 4 Worker」池
- 同 key 串行；不同 session / 优先级可并行
- 空闲队列超时回收（默认 600s）
- `enqueue` 线程安全；`replace_channel` 热替换

处理路径：`enqueue` → 防抖合并 → `build_agent_request_from_native` → `Workspace.stream_query` → `send_event` / `MessageRenderer`。

`session_id` 由频道自定义。默认形态接近 `{channel}:{user}`，但飞书实际是短后缀：群 `{app_id后4}_{chat_id后8}`，单聊 `open_id` 后 8 位。`feishu:chat_id:<id>` / `feishu:open_id:<id>` 是发送地址 `to_handle`，不是 session_id。详情见 `docs/FeishuChannel详解.md`。

---

### 5.4 Skills 系统

Skills 仍是 **YAML front matter + Markdown 指令**，不是 Python 插件。实现在 `agents/skill_system/`，不再使用 1.x 的 `customized_skills/` + `active_skills/` 三层（启动时 `migrate_legacy_skills_to_skill_pool`）。

#### 存储

| 层 | 路径 | 角色 |
|----|------|------|
| 内置 | `src/qwenpaw/agents/skills/<name>-<lang>/SKILL.md` | 随包分发 |
| 全局池 | `{WORKING_DIR}/skill_pool/` + `skill.json` | `SkillPoolService`；可配置额外只读 `skill_paths` |
| 工作区 | `{workspace}/skills/` + `{workspace}/skill.json` | Agent 实际启用集；`SkillService` |

`get_workspace_skills_dir()` 会把遗留目录名 `skill/` 重命名为 `skills/`。

Front matter 命名空间兼容 `qwenpaw` / `openclaw` / `clawdbot`。安装前走 `security/skill_scanner`。

#### 内置 Skills（源码目录，成对 `-zh`/`-en`）

`browser`、`cron`、`dingtalk_channel`、`docx`、`xlsx`、`pptx`、`pdf`、`file_reader`、`mailbox`、`channel_message`、`chat_with_agent`、`multi_agent_collaboration`、`make-skill`、`make_plan`、`guidance`、`QA_source_index`。

1.x 的 `himalaya` / `news` / `browser_visible` 等已不在内置目录。

Hub 安装：`agents/skill_system/hub.py`（lifespan 关闭时 `aclose_hub_client`）。

启动后 `run_pool_automation_pipeline()` 做池更新与工作区自动同步。

---

### 5.5 定时任务系统（Crons）

仍在 `src/qwenpaw/app/crons/`：**每个 Workspace 一份 `CronManager`**（不再挂在全局 AgentRunner 上）。

```
CronManager
├── APScheduler
├── CronExecutor  → workspace.stream_query() 或 ChannelManager.send_text()
├── HeartbeatTask → 读 HEARTBEAT.md，默认间隔 6h
└── JsonJobRepository → 工作区 jobs.json
```

`CronJobSpec.task_type`：`"text"` | `"agent"`。心跳目标：`main` / `last` / `inbox`（常量 `HEARTBEAT_TARGET_*`）。超时默认 300s，上限 3600s。

---

### 5.6 Drivers（MCP 等外部能力）

`src/qwenpaw/drivers/`。`DriverManager` 在生命周期边界上协议中立：登记 `DriverHandler` 类型、持久化 `DriverCard`、加密凭据、`ApprovalGate`、按次调用策略。

**当前注册的协议只有 `mcp`**（`create_driver_service` → `MCPDriverHandler`）。源码注释写明 ACP/A2A 可作为后续协议扩展能力模型；**仓库里没有 A2A `DriverHandler` 实现**。ACP 走独立子系统（§5.14），不是 Driver 协议插件。

遗留 MCP JSON 配置由 `drivers/adapters/mcp_legacy_config.py` 迁入 DriverCard 目录（`{workspace}/drivers/`），旧 `MCPClientManager` 运行时路径已去掉。`app/mcp/` 只保留 API schema 与 `config_service`。`MCPConfigWatcher` 对应现实现为 `app/driver_config_watcher.py`（按工作区）。

邮件 MCP：`app/mail/driver_config.py` 同步托管 `qwenpawmail.yaml` 卡片。

调用：Agent 工具层通过 `drivers/adapters/agentscope_tool.py` 把 Driver capability 暴露给 Toolkit；策略拒绝会返回 `driver_policy_denied`（系统提示里要求下一轮按最新策略重试）。

---

### 5.7 配置管理

#### 全局 `Config`（`config.json`）

`src/qwenpaw/config/config.py` 根模型字段包括：`channels`、`mcp`、`tools`、`last_api`、`agents`、`last_dispatch`、`security`、`acp`、`browser`、`show_tool_details`、`user_timezone`、`plugins`、`skill_paths`。

`AgentsConfig`：`active_agent`、`profiles`（id → `workspace_dir` 等）。

#### 每智能体 `agent.json`

`AgentProfileConfig`：运行参数、记忆后端、`LoopConfig`、`CodingModeConfig`、模型槽位 / LLM 路由、邮件、backend（`qwenpaw` | harness id）等。`load_agent_config(agent_id)` 带缓存与指纹；写盘仍走原子替换。

热重载：`AgentConfigWatcher`（工作区服务）+ 全局 `config/watcher.py`。频道变更由该工作区 `ChannelManager.replace_channel`。

敏感值：`SECRET_DIR`（默认 `{WORKING_DIR}.secret`）+ keyring（`QWENPAW_KEYRING_ACCOUNT`）。

---

### 5.8 LLM 提供商系统（Providers）

`ProviderManager`（单例）+ `providers/provider_catalog.py` 的 `BUILTIN_PROVIDERS`，包括但不限于：

QwenPaw Local、Ollama、LM Studio、OpenRouter、GitHub Models、ModelScope、DashScope、阿里云 Coding/Token Plan（国内/国际）、OpenCode、Kilo、OpenAI / OpenAI Response、Azure OpenAI、Anthropic、Gemini、DeepSeek、Kimi（国内/国际/Coding Plan）、MiniMax、智谱、SiliconFlow、Volcengine、小米 MiMo 等。

插件可通过 `PluginRegistry` 注册额外 Provider。启动时同步目录与本地模型 resume。支持 fallback 链（`fallback_chat_model.py`）与限流（`rate_limiter.py`）。

控制台 / CLI `qwenpaw models` 走 `/api/models`。

---

### 5.9 本地模型支持

`src/qwenpaw/local_models/`：`LocalModelManager`、`LlamaCppBackend`、`download_manager.py`。模型文件目录 `DEFAULT_LOCAL_PROVIDER_DIR`（`WORKING_DIR/local_models`）。Ollama / LM Studio 作为远程兼容 Provider，不是 llama.cpp 进程内后端。

下载任务仍由内存 store 跟踪（pending → downloading → completed / failed / cancelled）。API：`/api/local-models`。

---

### 5.10 记忆、Scroll 与上下文

#### 长期记忆后端

`memory_registry`（`agents/memory/base_memory_manager.py`，对外再导出 `qwenpaw.memory`）：

| id | 类 | 说明 |
|----|-----|------|
| `remelight` | `ReMeLightMemoryManager` | 默认；委托 ReMe 0.4 application/job |
| `none` | `NoopMemoryManager` | 关闭长期记忆 |

工作区配置的后端不可用时回退 `remelight`（`workspace.py` 的 `_MEMORY_BACKEND_FALLBACK`）。插件类型 `memory` 必须在 Workspace 启动前注册（lifespan phase 1）。

主动记忆：`agents/memory/proactive/`（懒导出，避免与 `react_agent` 循环导入）。

工作区 Markdown：`AgentMdManager`；人设文件仍在工作区根 / `md_files`。

#### Scroll（短上下文策略）

`src/qwenpaw/agents/context/scroll/`：

- `ScrollContextManager`：write-through 每一轮到耐久 `conversation_history`；超 token 阈值时保留尾部与当前轮，中间折叠进 `EvictionIndex`；continuation summary **只是状态缓存，不替代原文**
- `history.py`：`HistoryStore`（SQLite）
- `recall_tool.py`：`recall_history`（参数化 SQL）；`recall_history_python` 为沙箱内逃逸口
- `sync.py`：启动时把旧 `sessions/*.json` 迁入 `history.db`

1.x `CoPawInMemoryMemory` + `MemoryCompactionHook` 摘要压缩路径已由 Scroll 取代。

另有 `agents/context/visual_compression/`（视觉压缩管线，打包字体 atlas）。

---

### 5.11 Loop Engineering 与 Agent Mode

#### StopGate

`src/qwenpaw/loop/`：`StopHandler` + 门控。`catalog.py` 供用户可编辑循环模式选用：

`IterationGate`、`DoomLoopGate`（重复检测）、`CompletionRubricGate` / `QualitativeRubricGate`、`TokenBudgetGate`、`TimeoutGate`、`ToolCallBudgetGate`、`BudgetGate`、`FileLoopGate` 等。

`LoopConfig`（写在 `agent.json`）：`iteration` / `doom_loop` / `rubric` / `goal` / `mission` / `custom_modes`（最多 20，斜杠命令唯一）。

#### AgentMode

`src/qwenpaw/modes/`：每个 Mode 打包命令、工具、钩子、提示词片段。

| Mode | 包 | 行为 |
|------|-----|------|
| `coding` | `modes/coding/` | LSP/AST、项目目录注入、`CodingModeMixin` |
| `mission` | `modes/mission/` | `/mission`，故事迭代 + MissionGate |
| `goal` | `modes/goal/` | `/goal`，直到 rubric 满足 |
| `custom_loop` | `modes/custom_loop/` | 用户从内置门组合 |
| `default` | `modes/default/` | 默认聊天循环 |

`WorkspacePlugins.register_mode()` 拒绝重名。`Runtime._start_modes` 在 POST_AGENT_BUILD 附近激活。

---

### 5.12 治理、沙箱与安全

#### Governance

`src/qwenpaw/governance/`：`ResourceGovernor`、`GovernanceAction`（`allow` / `deny` / `ask` / `sandbox_fallback`）。双层规则：`builtin_rules`（不可被 Agent 改）+ `user_rules`。评估：ToolRegistry 类型 → builtin → user → 全局回退。批准时可泛化规则（高危命令保持精确匹配）。工具包装：`PolicyGuardedTool`。

#### Sandbox

`src/qwenpaw/sandbox/`：**每次工具调用创建、用完销毁**。

| 模式 | 实现 |
|------|------|
| `SEATBELT` | `macos_sandbox.py`（sandbox-exec） |
| `BUBBLEWRAP` | `bubblewrap_sandbox.py`（Linux 首选） |
| `LANDLOCK` | Linux 5.13+ 回退 |
| `WINDOWS` | AppContainer / Elevated / Unelevated |
| `NONE` | 直跑 |

进程退出时 Windows 路径会 `shutdown_all_sandboxes()`。

#### Security 包

- `security/tool_guard/`：shell 危险命令、文件守护、规避检测
- `security/skill_scanner/`：SKILL.md 签名规则（注入、外泄、供应链等）
- `security/secret_store.py`：密钥存储

配置：`SecurityConfig`、`ToolGuardConfig`、`FileGuardConfig`、`SkillScannerConfig`。

---

### 5.13 插件与 PawApp

#### Plugin 类型（`plugins/architecture.py`）

`tool` / `provider` / `hook` / `command` / `channel` / `memory` / `frontend` / `app` / `general`。

目录：`{WORKING_DIR}/plugins/`。`PluginLoader.load_all_plugins(types=[...])` 分阶段。PawApp 也装进同一目录，manifest 带 `meta.pawapp`，只在 App Center 展示。

#### PawApp SDK

`src/qwenpaw/pawapp/`：`PawApp` 包一层 `PluginApi`；`PawAppContext`（chat / storage / tools / ui）；`get_ctx` FastAPI 依赖；托管 Agent 档案与 ManagedService。

市场：`app/routers/market.py`、`src/qwenpaw/market/`。

---

### 5.14 Hub、Harness 与 ACP

#### Hub

`src/qwenpaw/hub/` + `cli/hub_cmd.py`：`qwenpaw hub` 独立控制面。`HubService` 编排 `RuntimeRegistry` / `RuntimeProvisioner`（本地进程或 Docker extra）。含 OAuth 路由、WebSocket 代理、Windows 反向隧道与进程隔离。这是**多用户运行时托管**，与 Skills Hub 不是同一概念。

#### Harness

`src/qwenpaw/harnesses/`：工作区 `backend != "qwenpaw"` 时 `Workspace.stream_query` 转给 `HarnessRuntime`。`PROVIDER_CATALOG` 含 **codex**、**qoder**（缺依赖时 `MissingDependencyAdapter`）。事件映射到统一 `HarnessEvent`。

#### ACP

`src/qwenpaw/agents/acp/`：`ACPService` / `QwenPawACPAgent` 懒加载。CLI `qwenpaw acp`；TUI 可走 ACP transport（`cli/tui/transport/acp.py`）。`delegate_external_agent` 走 ACP 适配。全局 `Config.acp`。前端路由 `/acp`。

---

### 5.15 Browser-use 与 Computer-use

#### Browser-use

配置 `BrowserConfig`（`config.browser`）。`browser.experimental=true`（默认走实验轨）加载 `agents/tools/browser.py` + `src/qwenpaw/browser/`：subprocess 执行平面、`execution/kernel.py` 工作池、Playwright / Chrome CDP control_link、审批与治理错误码。lifespan 挂 idle watchdog，`discard_idle_workers`。`experimental=false` 回退 `deprecated_browser/browser_control.py`。

#### Computer-use

`src/qwenpaw/app/computer_use/`：`HostRuntimeProvider` 对接桌面宿主（**仅 win32 / darwin** 有原生 helper）。命名管道 / 控制口协议版本 `COMPUTER_USE_PROTOCOL_VERSION = 2`。请求钩子写入 `computer_use_turn_id`。无 helper 时 `RuntimeStatus.available` 为假，不假装全平台桌面控制。

---

## 6. API 接口设计

REST 挂在 `/api`。`app/routers/__init__.py` 汇总全局路由；`create_agent_scoped_router()` 把同一套能力挂到 `/api/agents/{agentId}/...`。另有 `AgentContextMiddleware`、`AuthMiddleware`、`RuntimeBoundaryMiddleware`。

`DOCS_ENABLED`（`QWENPAW_OPENAPI_DOCS`）为真时才开 `/docs`。

### 根路径

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/` | Console SPA `index.html`（无构建则提示 npm build） |
| GET | `/api/version` | `{"version": ...}` |
| GET | `/api/healthz` | 健康检查（`routers/healthz.py`） |

### 路由前缀（实现文件）

| 前缀 | 模块 |
|------|------|
| `/agents` | `routers/agents.py` |
| `/agents/{agentId}` | `routers/agent_scoped.py` |
| `/config` | `routers/config.py`（频道等） |
| `/models` | `routers/providers.py` |
| `/local-models` | `routers/local_models.py` |
| `/skills` | `routers/skills.py` + `skills_stream.py` |
| `/mcp` | `routers/mcp.py` + `mcp_oauth.py` |
| `/tools` | `routers/tools.py` |
| `/workspace` | `routers/workspace.py` |
| `/workspace/git` | `routers/git.py` |
| `/workspace/checkpoints` | `routers/checkpoints.py` |
| `/envs` | `routers/envs.py` |
| `/cron` | `app/crons/api.py` |
| `/chats` | `app/chats/api.py` |
| `/console` | `routers/console.py` |
| `/messages` | `routers/messages.py` |
| `/files` | `routers/files.py` |
| `/settings` | `routers/settings.py` |
| `/plugins` / `/frontend_plugin` | 插件 |
| `/pawapps` | `routers/pawapps.py` |
| `/market` | `routers/market.py` |
| `/backups` | `routers/backup.py` |
| `/auth` | `routers/auth.py` |
| `/approval` | `routers/approval.py` |
| `/tool-calls` | `routers/tool_calls.py` |
| `/coding-mode` | `routers/coding_mode.py` |
| `/loops` | `routers/loops.py` |
| `/harnesses` | `routers/harnesses.py` |
| `/token-usage` / `/agent-stats` | 用量与统计 |
| `/fork` | `routers/fork.py` |
| `/access-control` / `/mail-access-control` | 访问控制 |
| `/provider_oauth` | 提供商 OAuth |
| voice | `routers/voice.py`（无统一 `/api` 前缀约定，见 `_app.py`） |
| browser chrome WS | `_app.py` 挂载的 control_link 路由 |

聊天流式仍走 Console / chats 相关接口，内部调用 `Workspace.stream_query`。具体 verb 以各 router 源码为准，避免与 1.x 路径表一一臆造。

---

## 7. 数据流

### 7.1 用户消息

```
外部频道 / Console / TUI / CLI
    → Channel.enqueue(native_payload)
    → UnifiedQueueManager（按 session 串行、防抖合并）
    → build_agent_request_from_native
    → Workspace.stream_query
         ├─ backend=qwenpaw:
         │     Runtime 8 阶段 → 斜杠命令?
         │     AgentBuilder → QwenPawAgent
         │     ReAct：LLM ↔ Toolkit（治理/沙箱/Drivers/Skills）
         │     Scroll on_save / compress
         └─ backend=codex|qoder: HarnessRuntime
    → Channel.send_event + MessageRenderer
```

### 7.2 定时任务

```
APScheduler → CronExecutor
  text  → ChannelManager.send_text
  agent → workspace.stream_query → send_event
心跳   → 读 HEARTBEAT.md → stream_query → main/last/inbox
```

### 7.3 配置热重载

```
config.json / agent.json / DriverCard
  → ConfigWatcher / AgentConfigWatcher / driver_config_watcher
  → ChannelManager.replace_channel
  → DriverManager.build_drivers
  → MultiAgentManager.reload_agent（世代守卫）
```

### 7.4 Scroll

```
每轮写入 history.db
超窗 → EvictionIndex + continuation summary（缓存）
模型调用 recall_history 回放原文
启动 sync_all_scroll_agents 回填旧 sessions/*.json
```

---

## 8. 数据存储方案

仍是**文件系统优先**（Hub 控制面另有自己的 `hub/database.py`，仅 Hub 进程使用）。

```
~/.qwenpaw/          # 或 QWENPAW_WORKING_DIR；若 ~/.copaw 存在则用遗留目录
├── config.json
├── providers.json   # ProviderManager 持久化
├── envs.json
├── .env
├── skill_pool/      # 全局 Skills + skill.json
├── plugins/
├── media/
├── local_models/ / models/
├── workspaces/
│   └── <agent_id>/
│       ├── agent.json
│       ├── skills/ + skill.json
│       ├── md_files/ 或根目录 AGENTS.md 等
│       ├── memory/              # ReMe Markdown
│       ├── drivers/             # DriverCard YAML
│       ├── credentials.yaml
│       ├── jobs.json / chats.json
│       ├── sessions/*.json      # SafeJSONSession
│       ├── history.db           # Scroll（路径以 HistoryStore 为准）
│       └── coding_projects/
├── token_usage.json
└── HEARTBEAT.md                 # 也可按工作区覆盖，见 get_heartbeat_query_path
```

密钥：`{WORKING_DIR}.secret`（`QWENPAW_SECRET_DIR`）。备份：`QWENPAW_BACKUP_DIR`。

JSON 写入：**先 `.tmp` 再 `Path.replace()`**。YAML 治理规则同样原子写（`write_yaml_atomic`）。

---

## 9. 前端架构

`console/`：React 18 + Vite 6 + Ant Design 5 + `@agentscope-ai/design`。路由与侧栏改为 **registry 数据驱动**（`layouts/registry/builtinRoutes.tsx`、`builtinMenu.ts`），插件可 `QwenPaw.route.add` / `menu.add`。

### 内置路由（节选）

| 路径 | 页面 |
|------|------|
| `/` → `/chat` | Chat |
| `/files` | 通用文件工作区 |
| `/channels` `/sessions` `/inbox` `/cron-jobs` `/heartbeat` | Control |
| `/skills` `/skill-pool` `/tools` `/mcp` `/acp` `/checkpoints` `/agent-config` | Agent |
| `/agents` `/models` `/environments` `/offload-policy` `/security` `/token-usage` `/agent-stats` `/voice-transcription` `/debug` `/backups` | Settings |
| `/market` `/imports` `/apps/:appId` | 市场 / PawPort / App Center |
| `/settings/*` | SettingsCenter |

组件模式仍为 `pages/<Feature>/index.tsx` + `use*.ts` + `components/`。API 在 `console/src/api/modules/`。用户可见文案走 i18n（`locales/en.json`、`zh.json`）。状态：ahooks + zustand。

生产：FastAPI 托管 `src/qwenpaw/console/`（或 `QWENPAW_CONSOLE_STATIC_DIR`）。开发：Vite 代理 `/api` 到 8088。`index.html` 强制 no-cache。

桌面：Tauri 2（`@tauri-apps/*`，`qwenpaw desktop`）。

---

## 10. CLI 系统

入口：`qwenpaw = qwenpaw.cli.main:cli`（兼容脚本名 `copaw`）。`LazyGroup` 懒加载子命令。

**无子命令**（或第一个位置参数像项目路径）→ TUI（`cli/tui/launch.py`）。默认 API `127.0.0.1:8088`（`read_runtime_api` / `read_last_api`）。

`lazy_subcommands`（`cli/main.py`）：

| 命令 | 模块 |
|------|------|
| `app` | `app_cmd.py` 启动 FastAPI |
| `init` | `init_cmd.py` |
| `tui` | `tui/launch.py` |
| `hub` | `hub_cmd.py` |
| `acp` | `acp_cmd.py` |
| `channels` / `channel` | `channels_cmd.py` |
| `chats` / `chat` | `chats_cmd.py` |
| `cron` | `cron_cmd.py` |
| `models` | `providers_cmd.py` |
| `skills` | `skills_cmd.py` |
| `env` | `env_cmd.py` |
| `agents` / `agent` | `agents_cmd.py` |
| `plugin` | `plugin_commands.py` |
| `daemon` | `daemon_cmd.py` |
| `desktop` | `desktop_cmd.py` |
| `task` | `task_cmd.py` |
| `doctor` | `doctor_cmd.py` |
| `auth` | `auth_cmd.py` |
| `clean` / `uninstall` / `update` / `shutdown` | 对应 `*_cmd.py` |
| `auto` | `@api_action` 生成的 HTTP 客户端子命令 |

`cli/mission_cmd.py` 存在，供 Mission 相关调用；主分组以 `lazy_subcommands` 为准。HTTP 模式命令依赖已启动的 `qwenpaw app`（`cli/http.py`）。

---

## 11. 部署方案

### Docker（`deploy/`）

多阶段：`console-builder`（`npm ci && npm run build`）→ runtime（Node 基础镜像 + Python venv + Chromium + Xvfb/XFCE + supervisor）。

- `QWENPAW_WORKING_DIR=/app/working`
- `QWENPAW_SECRET_DIR=/app/working.secret`
- `QWENPAW_BACKUP_DIR=/app/working.backups`
- `QWENPAW_DISABLED_CHANNELS` 默认 `imessage`
- `QWENPAW_PORT` 默认 8088
- `PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH=/usr/bin/chromium`
- `QWENPAW_RUNNING_IN_CONTAINER=1`

supervisord：`dbus`、`xvfb`（`:1` 1280x800x24）、`xfce4`、`app` → `qwenpaw app --host 0.0.0.0 --port ${QWENPAW_PORT}`。

### 本地

```bash
pip install -e ".[dev]"
qwenpaw init --defaults
qwenpaw app --host 0.0.0.0 --port 8088
# 另一终端
cd console && npm ci && npm run dev
```

### PyPI

```bash
pip install qwenpaw
qwenpaw init --defaults
qwenpaw app
```

---

## 12. 扩展机制

| 扩展点 | 做法 |
|--------|------|
| 频道 | 实现 `BaseChannel`，以 plugin type `channel` 注册（不要再丢 `custom_channels/`） |
| Skills | 池或工作区 `SKILL.md`；API / Console / Hub |
| 工具 | `@tool_descriptor` + 在 `tools/__init__.py` import；或 plugin type `tool` |
| LLM | `/api/models` 自定义 OpenAI 兼容，或 plugin type `provider` |
| 记忆后端 | 实现 `BaseMemoryManager`，`memory_registry.register`，plugin type `memory` |
| 钩子 / 斜杠命令 / Mode | `WorkspacePlugins` 或 plugin `hook` / `command` |
| MCP | DriverCard + Console MCP 页（底层 `DriverManager`） |
| PawApp | `meta.pawapp` 插件 + `PawApp` SDK |
| 前端路由/菜单 | `QwenPaw.route.add` / `menu.add` |
| Harness | `harnesses/` 适配器 + agent `backend` |
| Doctor | entry point 组 `qwenpaw.doctor`（仍加载遗留 `copaw.doctor`） |

工作区迁移：`/api/workspace` 下载上传、`BackupManager`、PawPort（`portability/`）。

---

## 13. 1.x → 2.x 映射

| 1.x（约 2026-03 CoPaw） | 2.x QwenPaw |
|-------------------------|-------------|
| `src/copaw`、`CoPawAgent` | `src/qwenpaw`、`QwenPawAgent` |
| agentscope-runtime `AgentRunner` / `AgentApp` | `Workspace.stream_query` + `Runtime`；无 AgentApp |
| 全局单 Agent + `~/.copaw/md_files` | 多 `Workspace`，默认 `~/.qwenpaw/workspaces/<id>` |
| `MCPClientManager` | `DriverManager` + `MCPDriverHandler` |
| `customized_skills` / `active_skills` | `skill_pool` + 工作区 `skills/` |
| `MemoryCompactionHook` + `CoPawInMemoryMemory` | `ScrollContextManager` + `history.db` |
| `MemoryManager`(ReMe 0.3) | `ReMeLightMemoryManager`(ReMe 0.4) |
| 每频道固定 4 Worker | `UnifiedQueueManager` 按 session 按需 consumer |
| `custom_channels/` 扫描 | 插件 `channel` 类型 |
| 无 TUI / Hub / PawApp / Loop / Drivers | 均已在对应源码目录落地 |

---

## 附录：全局常量（`constant.py`）

| 常量 | 环境变量 / 来源 | 默认 |
|------|-----------------|------|
| `WORKING_DIR` | `QWENPAW_WORKING_DIR` | 显式路径 → 否则若 `~/.copaw` 存在则用之 → 否则 `~/.qwenpaw` |
| `SECRET_DIR` | `QWENPAW_SECRET_DIR` | `{WORKING_DIR}.secret` |
| `CONFIG_FILE` | `QWENPAW_CONFIG_FILE` | `config.json` |
| `JOBS_FILE` / `CHATS_FILE` | `QWENPAW_*` | `jobs.json` / `chats.json` |
| `HEARTBEAT_FILE` | `QWENPAW_HEARTBEAT_FILE` | `HEARTBEAT.md` |
| `HEARTBEAT_DEFAULT_EVERY` | — | `"6h"` |
| `HEARTBEAT_DEFAULT_TARGET` | — | `"main"` |
| `PLUGINS_DIR` | — | `{WORKING_DIR}/plugins` |
| `DEFAULT_LOCAL_PROVIDER_DIR` | — | `{WORKING_DIR}/local_models` |
| `CODING_PROJECT_SUBDIR` | — | `coding_projects` |
| `BUILTIN_QA_AGENT_ID` | — | `QwenPaw_QA_Agent_0.2` |
| `LOG_LEVEL_ENV` | `QWENPAW_LOG_LEVEL` | `info` |
| `PROJECT_NAME` | — | `QwenPaw` |
| `CUSTOM_AGENT_STARTUP_CONCURRENCY` | `QWENPAW_CUSTOM_AGENT_STARTUP_CONCURRENCY` | `5` |

---

*本文档根据 QwenPaw 2.x 源码整理。概念图可对照官网 `website/public/docs/architecture.zh.md`；类名与路径以本仓库为准。*
