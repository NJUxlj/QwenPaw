# QwenPaw 前端代码学习指南

> 面向：会一点 HTML / CSS / JS、刚接触 React 的同学  
> 产品：QwenPaw v2.2.x（当前树为 `2.2.2b1`）  
> 前端包名：`qwenpaw-console`（见 `console/package.json`）  
> 日期：2026-09-15  
> 原则：跟着仓库里的真实文件走，不要背目录清单。

---

## 0. 怎么用这份指南

这份文档像一门短课：每一课告诉你「打开哪个文件、会看到什么、下一步跳到哪」。读完你应该能自己从入口跟到聊天页，再扩到设置 / 频道 / OS Shell。

建议本地同时开两个进程：

1. **后端**（FastAPI，默认端口 **8088**）

   ```bash
   qwenpaw app
   ```

   默认绑定 `127.0.0.1:8088`（见 `src/qwenpaw/cli/app_cmd.py`）。浏览器直接打开 `http://127.0.0.1:8088/` 时，后端会托管已经构建好的静态 Console。

2. **前端开发服务器**（Vite，端口 **5173**）

   ```bash
   cd console
   npm install
   npm run dev
   ```

   `npm run dev` 实际是 `vite --host`。`console/vite.config.ts` 里写死了：

   - 开发服务器：`host: "0.0.0.0"`，`port: 5173`
   - 把 `/api` 代理到 `http://localhost:8088`

开发时用浏览器打开 `http://localhost:5173/`。F12 打开开发者工具，Network 面板里所有 `/api/...` 请求都会被 Vite 转到后端。

如果你只想看「已经打包好的界面」，先 `cd console && npm run build`，再 `qwenpaw app`，打开 `http://127.0.0.1:8088/`。构建产物不进 git，运行时由 `src/qwenpaw/app/_app.py` 找静态目录并托管。

读代码时请用编辑器的「转到定义」（Ctrl/Cmd + 点击）。这份指南只带你认路，不会把每个文件讲一遍。

---

## 1. 前端是什么、和后端怎么分工

**Console** 是一个 **SPA**（Single-Page Application，单页应用）：浏览器只加载一次 HTML，之后点侧边栏、切页面，都是前端自己用路由换组件，而不是整页刷新。

后端是 Python 的 FastAPI。两边这样分工：

| 谁 | 负责什么 |
|---|---|
| Console（`console/`） | 画出界面、记住你选了哪个 Agent、把你的点击变成 HTTP 请求 |
| FastAPI（`src/qwenpaw/app/`） | 鉴权、会话、频道、模型配置、真正跑 Agent |
| 静态托管 | 生产环境由 `_app.py` 把 `index.html` + `/assets` 发给浏览器 |

请求怎么走：

```
浏览器 Console
    │  fetch("/api/chats") 或 POST /api/console/chat
    ▼
Vite 开发代理（5173 → 8088）  或  同源（后端直接托管静态文件）
    ▼
FastAPI 路由（/api/...）
```

`console/src/api/config.ts` 的 `getApiUrl()` 会给路径加上 `/api` 前缀。`VITE_API_BASE_URL` 为空时表示**同源**（不写死主机名），所以开发靠 Vite 代理，生产靠后端自己提供同一端口。

静态文件谁托管（`src/qwenpaw/app/_app.py`）：

1. 环境变量 `QWENPAW_CONSOLE_STATIC_DIR`
2. 安装包里的 `src/qwenpaw/console/`（带 `index.html`）
3. 仓库里的 `console/dist/`
4. 找不到就警告，并回落到当前目录的 `console/dist`

`GET /` 返回 `index.html`；`/assets` 挂静态资源；其它前端路径走 SPA fallback（再吐一遍 `index.html`，让 React Router 在浏览器里匹配）。`/api` 永远优先于这段静态逻辑。

仓库里还有两块「旁路前端」，主线仍是 Console：

- 桌面壳：`console/src-tauri/`（Tauri 2），把同一套 Console 塞进原生窗口
- 插件 UI：`plugins/*/frontend` 或 `plugins/*/ui`（例如 `plugins/bundle/chrome/frontend`、`plugins/apps/qwenpaw-creator/ui`），运行时由 Console 动态加载

---

## 2. 你需要的最小前端知识

读 QwenPaw 前端，不需要先成为 React 专家。看到下面这些词，按这句话理解即可。

**组件（Component）**  
一个函数，返回「看起来像 HTML」的界面。例如 `function Sidebar() { return <div>...</div>; }`。文件名经常是 `index.tsx`。

**JSX**  
在 JS/TS 里写 `<Button>保存</Button>` 这种标签。它最终会被编译成 `React.createElement(...)`。`console/` 里几乎每个 `.tsx` 都是 JSX。

**props**  
父组件传给子组件的参数，类似函数参数。`<Sidebar selectedKey="core.chat" />` 里的 `selectedKey` 就是 prop。

**state**  
组件自己记住的数据。点按钮后界面会变，通常就是 state 变了。React 的 `useState`、以及后面的 zustand store，都是在管 state。

**Hook**  
名字以 `use` 开头的函数，只能写在组件（或其它 Hook）里面。例如 `useTranslation()` 拿文案，`useNavigate()` 跳转页面，`useChannels()` 拉频道列表。QwenPaw 页面的习惯是：`index.tsx` 负责画，`useXxx.ts` 负责数据和请求。

**路由（Route）**  
URL 路径 ↔ 页面组件的对照表。`/chat` 显示聊天页，`/channels` 显示频道页。库是 `react-router-dom`。

**TypeScript**  
给变量、函数参数加上类型。读代码时看到 `string`、`ChatSpec` 看不懂，先当注释；真正拦你的是「这个函数从哪调到哪」，不是类型细节。

**Ant Design（antd）**  
现成的 UI 组件库：按钮、表格、弹窗、布局。QwenPaw 还包了一层 `@agentscope-ai/design`（主题 / Form 等）和 `@agentscope-ai/icons`（图标）。你很少从零写按钮，而是 `<Button>`、`<Modal>`。

**SSE / 流式**  
Server-Sent Events：服务器用一条 HTTP 连接持续往浏览器推文字。聊天回复不是等整段结束才出现，而是 token 一块块过来。Chat 页的 `customFetch` 请求 `stream: true`，再用 `responseParser` 解析每一块 JSON。

---

## 3. 技术栈一览

以 `console/package.json` 为准（不要背成「大概是某某版本」）：

| 用途 | 包 | 版本（当前树） |
|---|---|---|
| UI 框架 | `react` / `react-dom` | `^18` |
| 路由 | `react-router-dom` | `^7.13.0` |
| 组件库 | `antd` | `5.29.3` |
| 设计主题 | `@agentscope-ai/design` | `1.0.29` |
| 聊天 SDK | `@agentscope-ai/chat` | `1.1.73-beta.1787638407498` |
| 全局状态 | `zustand` | `^5.0.3` |
| 国际化 | `i18next` / `react-i18next` | `^25.8.4` / `^16.5.4` |
| 构建 | `vite` | `6.4.1` |
| 语言 | `typescript` | `~5.8.3` |
| 格式化 | `prettier` | `3.0.0` |
| 检查 | `eslint` | `^9.25.0` |
| 桌面 | `@tauri-apps/api` / `@tauri-apps/cli` | `^2.10.1` |

其它你会在 Chat / Coding 里撞上的库：`monaco-editor`（代码编辑器）、`react-markdown`（Markdown）、`ahooks`、`motion`（动画）、`lucide-react`（图标）。

脚本（同文件 `scripts` 字段）：

- `npm run dev` — 开发
- `npm run build` — 生产构建（还会校验 Monaco CSS、预压缩、检查初始包体积）
- `npm run format` / `npm run lint` — 格式化与 ESLint
- `npm test` — Vitest

---

## 4. 仓库文件地图

先认 `console/` 顶层，再认 `console/src/`。每个目录一句话，细节留给后面的课。

### `console/` 顶层

| 路径 | 职责 |
|---|---|
| `index.html` | 唯一 HTML。里面有 `<div id="root">` 和启动画面，脚本指向 `/src/main.tsx` |
| `package.json` | 包名 `qwenpaw-console`、依赖、npm 脚本 |
| `vite.config.ts` | 开发端口 5173、`/api` 代理、路径别名 `@` → `src/` |
| `eslint.config.js` | ESLint 9 配置 |
| `tsconfig.json` | TypeScript 工程入口（引用 `tsconfig.app.json`） |
| `public/` | 不会被打包改名的静态资源（如 logo） |
| `src/` | 几乎全部前端源码 |
| `src-tauri/` | Tauri 2 桌面壳（Rust + `tauri.conf.json`） |
| `tauri.html` | 桌面开发入口 HTML（`tauri.conf.json` 的 `devUrl` 指向它） |

### `console/src/` 各目录

| 目录 | 一句话 |
|---|---|
| `main.tsx` / `App.tsx` | 启动与根组件 |
| `api/` | 跟后端说话：`modules/` 一个功能一份封装，`types/` 类型，`request.ts` 通用 fetch |
| `auth/` | 启动时检查要不要登录（`gate.ts`） |
| `layouts/` | 外壳：侧边栏、顶栏、主布局、菜单/路由注册 |
| `pages/` | 每个功能一页 |
| `components/` | 多页复用的零件（Agent 选择器、ToolCards、PageHeader…） |
| `stores/` | zustand 全局状态 |
| `contexts/` | React Context（主题、审批、桌面更新） |
| `plugins/` | 插件 SDK、菜单/路由/插槽注册表、动态加载 |
| `locales/` | 文案 JSON |
| `os/` | 桌面 OS Shell（窗口、Dock、壁纸） |
| `tauri/` | 桌面运行时：等后端就绪、关窗口提示 |
| `features/files-workspace/` | 会话文件抽屉 / 工作区 |
| `hooks/` | 跨页小钩子（移动端判断、消息提示等） |
| `utils/` | 纯工具函数（路由 basename、session 路径、懒加载） |
| `styles/` | 全局 CSS token / 布局 |
| `i18n.ts` | i18next 初始化 |

### 页面模式（请记住这个形状）

多数功能页长这样：

```
pages/<Feature>/
  index.tsx          ← 页面：布局、按钮、把 hook 的数据画出来
  useXxx.ts          ← Hook：请求 API、本地 state
  components/        ← 只给这一页用的卡片、抽屉、弹窗
  index.module.less  ← 样式
```

例子：

- 频道：`pages/Control/Channels/index.tsx` + `useChannels.ts` + `components/`
- 模型：`pages/Settings/Models/index.tsx` + `useProviders.ts` + `components/`
- Agent 配置：`pages/Agent/Config/index.tsx` + `useAgentConfig.tsx` + `components/`

Chat 页是例外：它很厚，核心 UI 来自 `@agentscope-ai/chat`，QwenPaw 在外面接 API、工具卡片和侧栏。

---

## 5. 第一课：启动链路（必读）

目标：从浏览器打开页面，一路跟到侧边栏出现。

```
index.html
    │  <script type="module" src="/src/main.tsx">
    ▼
main.tsx
    │  1. import "./i18n"                  加载多语言
    │  2. import builtinMenu / builtinRoutes  副作用：自注册菜单和路由
    │  3. installHostExternals()           把 React/antd 挂到 window，给插件用
    │  4. createRoot(#root).render(<App />)
    ▼
App.tsx
    │  ThemeProvider → BackendModeRouter（问后端 /api/auth/status）
    │       → PluginProvider → AppInner
    ▼
AppInner
    │  路径是 /os* ？──是──► DesktopOS（没有外层 BrowserRouter）
    │                └──否──► BrowserRouter
    │                         /login
    │                         /hub/admin
    │                         /*  → AuthGuard → MainLayout
    ▼
MainLayout
    │  useRoutes() 取出注册表里的全部路由
    │  左边 Sidebar，上边 Header，中间 <Routes> 渲染当前页
    ▼
Sidebar
         读 useMenuItems()，画「新任务」、Inbox、快捷入口、会话列表
```

### 5.1 打开 `console/index.html`

你会看到：

- `<div id="root">` 里有一段「Loading Console」启动画面
- 底部：`<script type="module" src="/src/main.tsx"></script>`

Vite 从这里把 TypeScript 编译后送进浏览器。`#root` 就是 React 的挂载点。

### 5.2 打开 `console/src/main.tsx`

按顺序发生四件重要的事：

1. `import "./i18n"` — 初始化 i18next，读 `locales/*.json`
2. **副作用导入** `layouts/registry/builtinMenu` 和 `builtinRoutes.tsx`  
   这两个文件在被 import 的瞬间，就把内置菜单/路由塞进全局注册表。注释写得很直白：这样 Sidebar / MainLayout **第一次渲染就能看到它们**。
3. `installHostExternals()` — 把 React、antd 放到 `window.QwenPaw.host`，插件不必再打包一份
4. `createRoot(...).render(<App />)` — 真正开始画界面

另外它还拦了「把文件拖到窗口空白处导致浏览器直接打开文件」——桌面 WebView 需要这个。

### 5.3 打开 `console/src/App.tsx`

先看最底部的 `App`：只包了一层 `ThemeProvider`，然后是 `BackendModeRouter`。

`BackendModeRouter` 会调用 `auth/gate.ts` 的 `resolveBackendInfo()`（内部是 `authApi.getStatus()` → `GET /api/auth/status`），用来区分普通模式和 Hub 模式。失败则显示 `tauri/BackendLoadingPage`。

成功后进入 `AppInner`：

- 配好 antd 的中/英/日/俄/印尼 locale 和亮暗主题（`@agentscope-ai/design` 的 `bailianTheme`）
- 若 URL 是 `/os` 或 `/os/...`（`utils/navigationMode.ts` 的 `isOsPath`），渲染 `os/DesktopOS.tsx`，**外面没有** `BrowserRouter`（每个窗口自己有 `MemoryRouter`）
- 否则用 `BrowserRouter`，只声明三条「外壳路由」：
  - `/login` → 登录页
  - `/hub/admin` → Hub 管理页
  - `/*` → `MainLayout`（其它所有业务页都在这里面）

`AuthGuard` 会再调 `resolveAuthGate`：后端没开鉴权就放行；开了鉴权则看 `localStorage` 里的 `qwenpaw_auth_token`，必要时跳 `/login`。

### 5.4 打开 `console/src/layouts/registry/builtinRoutes.tsx`

这是**真正的页面表**。每一项有 `id`、`path`、懒加载的 `component`。文件末尾：

```ts
routeRegistry.addBuiltin(BUILTIN_ROUTES);
```

重要的几条（完整列表以该文件为准）：

| id | path | 页面 |
|---|---|---|
| `core.root` | `/` | 立刻重定向到 `/chat` |
| `core.chat` | `/chat/*` | `pages/Chat` |
| `core.settings-center` | `/settings/*` | `pages/SettingsCenter` |
| `core.channels` | `/channels` | `pages/Control/Channels` |
| `core.models` | `/models` | `pages/Settings/Models` |
| `core.agent-config` | `/agent-config` | `pages/Agent/Config` |
| `core.inbox` | `/inbox` | `pages/Inbox` |
| `core.files` | `/files` | `pages/Files` |

页面用 `lazyImportWithRetry` 按需加载（见 `utils/lazyWithRetry.ts`），所以第一次点某个菜单会转一下圈。

### 5.5 打开 `console/src/layouts/registry/builtinMenu.ts`

这是**侧边栏数据**，不是路由。末尾：

```ts
menuRegistry.addBuiltin(BUILTIN_MENU);
```

注意注释：**`core.chat` 不在这份菜单里**。聊天入口是侧边栏顶部的「新任务」按钮 + 会话列表，不是 antd Menu 的一项。菜单分成两桶：

- `primary.agentScoped`：Inbox、扩展、Control 组（频道/会话/定时任务/心跳）、Agent 组（文件/技能/工具/MCP/ACP/配置…）
- `primary.settings`：Agents、Models、环境变量、安全、备份…

每一项的 `route` 字段是路由 **id**（如 `"core.channels"`），不是 URL。Sidebar 会用 `adapter.tsx` 的 `routeIdToPath` 把 id 换成 `/channels`。

### 5.6 打开 `console/src/layouts/MainLayout/index.tsx`

你会看到经典的 antd `Layout`：

1. `useRoutes()` — 从注册表拿「内置 + 插件」全部路由
2. `pickSelectedKey(currentPath, routes)` — 用 react-router 的 `matchRoutes` 算出当前高亮的菜单 id
3. 设置中心页（`/settings/*`）会**藏掉 Sidebar**，只留 Header
4. 中间 `<Routes>` 循环 `renderableRoutes`，把每个 `r.Component` 挂上去
5. 还有两个一直挂着的后台组件：`ConsolePollService`、`AgentStatusPollingController`

### 5.7 打开 `console/src/layouts/Sidebar.tsx`

这是最长的布局文件之一，先抓三件事：

- `useMenuItems("primary.agentScoped")` / `useMenuItems("primary.settings")` 拿菜单
- `useAgentStore()` 拿当前选中的 Agent（每个请求都会带 `X-Agent-Id`）
- 「新任务」按钮调用 `handleNewChat`：已在 `/chat` 上就发自定义事件 `qwenpaw:sidebar-new-chat`；否则 `navigate("/chat")`
- 点某条会话：`buildChatPath(id)` → `navigate("/chat/<id>")`（`utils/sessionRoute.ts`）
- 「更多设置」：`navigate("/settings/general")`

到这里，启动链路结束。浏览器地址栏如果是 `/`，会被 `DefaultRedirect` 送到 `/chat`，中间区域开始加载 Chat 页。

---

## 6. 第二课：一次「打开聊天页」

Chat 目录文件很多。**先抓住发送 / 接收主路径**，周边面板（文件抽屉、模型选择器、后台任务）知道它们挂在哪就行。

### 6.1 从侧边栏走进 `/chat`

真实交互不是点一个叫「Chat」的菜单项（`builtinMenu.ts` 故意没放 `core.chat`），而是：

1. 点 **「新任务」**（`Sidebar.tsx` 的 `handleNewChat`）→ URL 变成 `/chat`
2. 或点 **会话列表**（`SidebarSessionList`）→ URL 变成 `/chat/<会话id>`

`MainLayout` 用 `pickSelectedKey` 匹配到 `core.chat`（path 是 `/chat/*`），懒加载 `pages/Chat/index.tsx` 里的 `ChatPage`。

### 6.2 打开 `console/src/pages/Chat/index.tsx`

文件很长（四千多行），但骨架很短：

1. 从 URL 取出会话 id：`getSessionIdFromPath(location.pathname)`（`/chat/xxx` → `xxx`）
2. 组装一份巨大的 `options` 对象，交给聊天 SDK
3. 渲染：

```tsx
<AgentScopeRuntimeWebUI
  ref={chatRef}
  key={refreshKey}
  options={options}
/>
```

**`AgentScopeRuntimeWebUI` 来自 npm 包 `@agentscope-ai/chat`**，不是本仓库自己画的气泡列表。输入框、消息列表、流式打字，大部分是这个 SDK 做的。QwenPaw 负责：接后端、改卡片、接插件、接文件抽屉。

同文件附近还会挂：

- `ChatSessionInitializer` — 让 URL 和 SDK 内部「当前会话」双向同步
- `FilesDrawer` — 右侧文件工作区（`features/files-workspace/`）
- 模型未配置时的 Modal（点「去配置」会 `navigate("/models")`）

### 6.3 URL ↔ 会话：`ChatSessionInitializer`

打开 `pages/Chat/components/ChatSessionInitializer/index.tsx`。

它做两件桥接：

- URL 里的 id → 调用 SDK 的 `setCurrentSessionId`
- SDK 里的会话列表 → `sessionListStore.syncFromLibrary`，这样 **Sidebar 在 Chat 组件树外面也能显示会话**

侧边栏「新任务」在已位于聊天页时，靠 `window` 事件 `qwenpaw:sidebar-new-chat` 通知这里去建会话。

### 6.4 会话列表怎么从后端来：`sessionApi`

打开 `pages/Chat/sessionApi/index.ts`。

这是一个实现了 SDK 接口 `IAgentScopeRuntimeWebUISessionAPI` 的对象，被塞进 `options.session.api`。它把 SDK 的「列出会话 / 读消息」翻译成我们的 REST：

- 列会话 → `api.listChats()` → `GET /api/chats`
- 读一个会话的历史 → `GET /api/chats/{id}`
- 默认 `userId = "default"`，`channel = "console"`

后端返回的是扁平 `role` + `content` 消息；`sessionApi` 会转换成 SDK 要的 **card** 结构（用户卡片、助手卡片、工具结果卡片）。

### 6.5 发送消息（主路径）

在 `ChatPage` 里搜索 `customFetch`。它被放进 `options.api.fetch`。

用户点发送后，SDK 调用 `customFetch`。关键步骤：

1. 检查当前 Agent 有没有激活的 LLM（`providerApi.getActiveModels`）；没有就弹出「去配置模型」
2. 组请求体：`input`（用户消息）、`session_id` / `user_id` / `channel`、`stream: true`
3. 带上鉴权头和 `X-Agent-Id`
4. `POST` 到 `getApiUrl("/console/chat")`，也就是 **`/api/console/chat`**
5. 用 `wrapChatResponseUsageStream` 包住 `Response`，一边给 SDK 读流，一边更新 token 用量

同一套 URL 在断线重连时也会再 POST 一次，body 带 `reconnect: true`（`options.api.reconnect`）。

### 6.6 流式收消息

`options.api.responseParser` 接收 SDK 拆出来的每一块 JSON 字符串：

- 普通增量：解析后交给 SDK 追加到当前助手气泡
- `type === "turn_usage"`：用量事件，不渲染成气泡
- `type === "rate_limited"`：提示限流
- 流结束时如果 `output` 为空，会补一条错误文案

所以：**流式不是 EventSource 对象，而是 fetch + ReadableStream + 逐块 JSON。**

停止生成走 `options.api.cancel` → `chatApi.stopChat(chatId)`。

### 6.7 消息怎么渲染：气泡 + ToolCards + Markdown

三层：

1. **SDK 默认气泡**  
   `pages/Chat/HostBubbles.tsx` 包装了 `@agentscope-ai/chat` 内部的 Request/Response Card，并用 vendor 的 `Bubble`、`Markdown`。助手回复的 Markdown 主要是 SDK 在画。

2. **本仓库的 Markdown 小品**  
   `components/Markdown/InlineMarkdown.tsx` 用于标题、loop 提示等小文本，不是主气泡。

3. **工具卡片 ToolCards**  
   Agent 调用 `read_file`、`execute_shell_command` 等工具时，不应该只显示一坨 JSON。  
   - 卡片实现：`components/Chat/ToolCards/cards/`（`ReadFileCard`、`ShellCard`、`GenericToolCard`…）  
   - 启动时 `registerBuiltinCards()`（`PluginContext` 初始化里调用）把它们注册进 `pluginSystem`  
   - Chat 页：`customToolRenderConfig: withGenericFallback(mergedToolRenderers)`  
   - 不认识的工具名会落到 `GenericToolCard`

`withGenericFallback` 在 `components/Chat/ToolCards/adapters/v1Adapter.tsx`：SDK 的 props 形状和本仓库卡片不完全一样，适配器负责翻译。

### 6.8 周边面板（知道挂点即可）

先不要逐个读完 `pages/Chat/` 下每个文件。主路径通了之后，再按需打开：

| 你在界面上看到 | 对应文件 |
|---|---|
| 顶栏标题 / 模型切换 | `components/ChatHeaderTitle`、`ModelSelector/` |
| 输入框附件、语音 | `RichFileReferenceInput.tsx`、`WhisperSpeechButton` |
| 右侧文件 | `features/files-workspace/FilesDrawer.tsx` |
| 后台跑着的工具 | `components/BackgroundTaskPanel.tsx` + `stores/backgroundTasksStore.ts` |
| 排队的消息 | `components/MessageQueuePanel.tsx` + `stores/messageQueueStore.ts` |

---

## 7. 第三课：前端怎么调后端

### 7.1 三层封装

```
页面 Hook（useChannels / useProviders / useAgentConfig）
    │
    ▼
api/modules/<功能>.ts   （chatApi、channelApi、providerApi…）
    │
    ▼
api/request.ts 的 request()
    │  getApiUrl(path) + buildAuthHeaders()
    ▼
浏览器 fetch  →  /api/...
```

总入口 `api/index.ts` 把各模块摊平成一个大 `api` 对象，所以页面里既能 `import api from "@/api"` 写 `api.listChannels()`，也能 `import { channelApi } from "@/api/modules/channel"`。

### 7.2 Base URL

打开 `api/config.ts`：

```ts
export function getApiUrl(path: string): string {
  const base = VITE_API_BASE_URL || "";
  const apiPrefix = "/api";
  const normalizedPath = path.startsWith("/") ? path : `/${path}`;
  return `${base}${apiPrefix}${normalizedPath}`;
}
```

`VITE_API_BASE_URL` 在 `vite.config.ts` 里用 `define` 注入，默认空字符串。开发时请求是 `/api/chats`，Vite 代理到 `localhost:8088`。桌面端 `getApiPort()` 会尝试从 base URL 或当前窗口端口解析，最后回落到 **8088**。

### 7.3 鉴权 token

同一文件：

- 登录成功后 `setAuthToken` 写入 `localStorage` 键 `qwenpaw_auth_token`
- `getApiToken()` 先读这个键，没有再用构建时的 `TOKEN` 常量

`api/authHeaders.ts` 的 `buildAuthHeaders()` 会附加：

- `Authorization: Bearer <token>`（有 token 时）
- `X-Agent-Id: <当前选中的 agent>`（从 `sessionStorage` / `localStorage` 的 `qwenpaw-agent-storage` 读，这是 `agentStore` 的 persist 数据）

登录页走 `api/modules/auth.ts`：`POST /api/auth/login`，再 `setAuthToken`。启动门卫是 `auth/gate.ts`。

### 7.4 错误处理

`api/request.ts`：

- 默认超时 30 秒，可 `retries`
- **401**：清 token，若当前不是登录页就跳到 `/login`
- 非 2xx：从 JSON 的 `detail` / `message` / `error` 拼错误信息再 `throw`
- 204 返回 `undefined`；非 JSON 当文本返回

页面 Hook 一般 `try/catch` 后用 `useAppMessage()` 弹 antd message。

聊天上传、SSE 这些不能走通用 `request()`（要 `FormData` 或原始 `Response`），所以 `chatApi.uploadFile`、Chat 的 `customFetch` 自己 `fetch`。

### 7.5 跟一个具体例子：频道页

打开 `api/modules/channel.ts`，例如：

```ts
listChannels: () => request<ChannelConfig>("/config/channels"),
updateChannelConfig: (channelName, body) =>
  request(`/config/channels/${encodeURIComponent(channelName)}`, {
    method: "PUT",
    body: JSON.stringify(body),
  }),
```

然后打开 `pages/Control/Channels/useChannels.ts`：

- `useEffect` 里 `api.listChannels()` + `api.listChannelTypes()`
- `selectedAgent` 一变就重新拉（因为请求头带了别的 Agent）
- 把结果放进 Hook 自己的 `useState`，**没有**放进 zustand

再打开 `pages/Control/Channels/index.tsx`：

- 调用 `useChannels()`
- 点卡片打开 `ChannelDrawer`
- 保存时 `api.updateChannelConfig(activeKey, proposedConfig)`

这就是「API 模块 → 页面 Hook → 页面组件」的标准链。Chat 比较特殊，中间多了 `@agentscope-ai/chat` 和 `sessionApi`。

---

## 8. 第四课：状态从哪来

不要把 `stores/` 当成百科去背。先分清三种容器：

| 容器 | 何时用 | 例子 |
|---|---|---|
| **zustand store** | 很多互不嵌套的组件要读同一份数据 | 当前 Agent、会话列表、Coding Mode |
| **页面 Hook 的 useState** | 数据只服务这一页 | `useChannels` 里的 `channels` |
| **React Context** | 真正的「主题 / 树形提供值」 | 亮暗色、审批弹窗 |

### 8.1 当前 Agent：`stores/agentStore.ts`

几乎每个请求都依赖它。`persist` 到 `qwenpaw-agent-storage`。`selectedAgent` 默认 `"default"`。Sidebar 的 `AgentSelector` 调用 `setSelectedAgent`。`authHeaders.ts` 读的就是这份存储，所以切 Agent 后新请求会带新的 `X-Agent-Id`。

### 8.2 会话列表：`stores/sessionListStore.ts`

注释已经说明动机：SDK 的会话列表活在 Chat 组件树里，Sidebar 在树外面。`ChatSessionInitializer` 写入，`SidebarSessionList` 读出。CRUD 之后调用 `syncSessionsGlobal()` 两边一起更新。

### 8.3 主题：`contexts/ThemeContext.tsx`（不是 zustand）

`ThemeMode`：`light` / `dark` / `system`，存在 `localStorage` 键 `qwenpaw-theme`。`App.tsx` 用 `useTheme().isDark` 切换 antd 算法。给 `<html>` 加 `dark-mode` class。

### 8.4 后台工具任务：`stores/backgroundTasksStore.ts`

Chat 里工具在后台跑时，面板要显示 live output。状态是 `running | done | cancelled`。和「当前正在流式输出的那条气泡」是两条线。

### 8.5 Coding Mode：`stores/codingModeStore.ts`

按 `agentId` 记住是否打开 Coding Mode。**以后端 `agent.json` 为准**，内存里不 persist；`MainLayout` 调用 `useSyncCodingMode()` 每次换 Agent 都向后端再灌一次。相关 UI 在 `pages/Coding/` 和 Chat 的布局切换。

其它 store（`messageQueueStore`、`filesSurfaceStore`、`sidebarStore`、`loopStore`…）等你改到对应功能再读。`sidebarStore` 记住侧边栏「钉住哪些快捷入口」。

---

## 9. 第五课：设置 / Agent / 频道页

### 9.1 一个设置页通常长什么样

1. `index.tsx`：`PageHeader` + 卡片/表格 + 抽屉/Modal
2. `useXxx.ts`：进入页面 `useEffect` 拉数据；保存函数调用 `api.xxx`
3. `components/`：卡片、表单字段
4. 文案全部 `t("...")`
5. 当前 Agent 来自 `useAgentStore()`，换 Agent 会触发 Hook 重拉

设置有**两扇门**：

- 侧边栏「更多设置」→ `/settings/general` → `pages/SettingsCenter/index.tsx`（分组目录，再嵌具体页）
- 直接 URL / 旧书签：`/channels`、`/models`、`/agent-config`（`builtinRoutes.tsx` 里的独立路由）

Settings Center 的分组定义就在它的 `index.tsx` 顶部 `SETTINGS_GROUPS`：General、Navigation，以及把 `core.channels`、`core.skills` 等 **routeId** 嵌进去。所以你在设置中心点「频道」，最后渲染的仍是 `pages/Control/Channels` 那个组件。

### 9.2 带读一整页：频道（代码最清晰）

按这个顺序打开，不要跳：

**① `layouts/registry/builtinRoutes.tsx`**  
找到 `{ id: "core.channels", path: "/channels", component: ChannelsPage }`。

**② `layouts/registry/builtinMenu.ts`**  
`core.channels` 挂在 `core.control-group` 下，`route: "core.channels"`。Sidebar 点它会 `navigate("/channels")`。

**③ `pages/Control/Channels/index.tsx`**  
`ChannelsPage`：`useChannels()` 拿列表；本地再 `useState` 管抽屉开关、筛选（全部 / 内置 / 自定义）。

**④ `pages/Control/Channels/useChannels.ts`**  
`api.listChannels()`、`api.listChannelTypes()`、`api.listChannelSchemas()`。`orderedKeys` 把内置频道排在前面（console、钉钉、飞书…）。

**⑤ `api/modules/channel.ts`**  
REST 路径一览：`GET /api/config/channels`、`PUT /api/config/channels/{name}`、冲突检查、二维码登录等。

**⑥ `pages/Control/Channels/components/ChannelCard.tsx` + `ChannelDrawer.tsx`**  
卡片展示开关状态；抽屉里是表单。保存回 `index.tsx` 的 `handleSubmit` → `api.updateChannelConfig`。

走完这一圈，Models（`useProviders` + `GET /api/models`）和 Agent Config（`useAgentConfig` + `GET/PUT /api/workspace/running-config`）就是同一套路，只是表单更长。

Agent Config 多一个细节：`pages/Agent/Config/index.tsx` 用 Tabs（`?tab=` 查询参数），`useAgentConfig.tsx` 一次拉 running-config、语言、时区。

---

## 10. 第六课：OS Shell 与桌面

### 10.1 浏览器里的 OS Shell（`console/src/os/`）

`App.tsx` 发现路径是 `/os` 时，不进 `MainLayout`，而是全屏渲染 `os/DesktopOS.tsx`。

你可以把它理解成「把已有页面塞进可拖动的窗口」，而不是另一套业务实现。`DesktopOS.tsx` 文件头注释写了：每个窗口渲染的仍是 `routeRegistry` 里那个 Component。

关键零件：

| 文件 | 作用 |
|---|---|
| `os/DesktopOS.tsx` | 桌面、壁纸、图标、启动 Splash |
| `os/Dock.tsx` / `osDockStore.ts` | 底栏 |
| `os/WindowFrame.tsx` / `osWindowStore.ts` | 窗口几何、焦点、空间（Spaces） |
| `os/WindowRouter.tsx` | **每个窗口一个 `MemoryRouter`**，页面里的 `useNavigate` 仍然能用 |
| `os/osApps.ts` | 把 `core.chat`、`core.files` 等 route id 映射成桌面图标 |
| `os/Launcher.tsx` / `AppStore.tsx` / `SettingsApp.tsx` | 启动台、应用商店、系统设置窗 |

从经典布局切过去：Sidebar 用户菜单里的「桌面模式」调用 `getOsRootHref()` → 跳到 `/os`（若站点挂在 `/console` basename 下会带前缀）。

### 10.2 Tauri 桌面 vs 浏览器 Console

源码里能直接看到的差别：

| | 浏览器 `npm run dev` | Tauri 桌面 |
|---|---|---|
| 入口 HTML | `index.html` → `main.tsx` | 开发时 `tauri.html`；`src-tauri/tauri.conf.json` 的 `devUrl` 是 `http://localhost:5173/tauri.html` |
| 等后端 | 你自己先起 `qwenpaw app` | `tauri/bootstrap.tsx` 渲染 `BackendReadyGate`，轮询后端就绪后 **整页跳到后端托管的 Console URL** |
| 关窗口 | 关标签页即可 | `CloseWindowPrompt`：问最小化到托盘还是退出 |
| 右键 | 浏览器默认菜单 | `App.tsx` 在 `isTauri()` 时 `preventDefault` 掉 contextmenu |
| 外链 | 新标签打开 | 拦截 `<a target="_blank">`，改走系统浏览器（`interceptBlankLinkClicks`） |
| 构建 | `vite build` → 静态目录 | `npm run build:tauri-bootstrap`，产物 `dist-tauri`，Rust 侧还会带上 python/node 运行时资源 |

`src-tauri/tauri.conf.json` 产品名是 **QwenPaw Desktop**，窗口默认 1280×800。Rust 代码在 `src-tauri/src/`（托盘、更新、computer-use 辅助进程），前端同学先不用下钻。

判断是否桌面：`@tauri-apps/api/core` 的 `isTauri()`。

---

## 11. 第七课：插件与 i18n

### 11.1 插件如何挂菜单 / 路由

启动时 `PluginProvider`（`plugins/PluginContext.tsx`）会：

1. `installHostSdk()` — 把 `window.QwenPaw.menu / route / slot / chat` 补齐
2. `registerBuiltinCards()` — 内置 ToolCards
3. `loadAllPlugins()`（`plugins/usePluginLoader.ts`）  
   `GET /api/frontend_plugin`，对每个有 `frontend_entry` 的插件，再拉 ` /api/frontend_plugin/{id}/files/{entry}`，当成 JS 模块 `import()`

插件脚本调用例如：

- `QwenPaw.menu.add(pluginId, item)` → 和内置菜单进**同一个** `menuRegistry`
- `QwenPaw.route.add(pluginId, route)` → 和内置路由进**同一个** `routeRegistry`

所以 Sidebar / MainLayout **不用写 if (plugin)**：`useMenuItems()` / `useRoutes()` 已经是合并后的快照。插槽用 `<Slot name="sider.bottom" />` 这类名字，插件 `QwenPaw.slot.fill(...)`。

`hostExternals.ts` 还把 React、antd 放到 `window.QwenPaw.host`，避免插件再打一份 React（否则会出现两个 React，Hook 会坏）。

仓库里独立的插件前端工程（主线以外，点到即可）：

- `plugins/bundle/*/frontend` 或 `plugins/bundle/*/ui`
- `plugins/apps/*/ui`（如 qwenpaw-creator）
- `plugins/memory/*/frontend`

### 11.2 为什么改文案必须改 `en.json` 和 `zh.json`

打开 `console/src/i18n.ts`：它把多种语言 JSON 注册进 i18next。当前树包括 `en`、`zh`、`ja`、`ru`、`pt-BR`、`id`、`vi`。回退语言是 **`en`**。

组件里应写：

```tsx
const { t } = useTranslation();
<Button>{t("common.save")}</Button>
```

`t("common.save")` 会去当前语言的 JSON 里找 `common.save`。只改 `zh.json`、不改 `en.json`：英文用户会看到 key 或英文 fallback 不一致；只改页面里的汉字、不走 `t()`：切语言不会变。

约定（项目规则）：**用户可见字符串用 `t("key")`，至少同步 `console/src/locales/en.json` 和 `zh.json`。** 新增 key 时两个文件都加，结构保持相同。

语言偏好：优先 `localStorage.language`；没有则 `App.tsx` 会调 `languageApi.getLanguage()` 问后端。

---

## 12. 推荐自学顺序（清单）

### Day 1 — 启动链路 + Chat

打开这些文件（按顺序）：

1. `console/index.html`
2. `console/src/main.tsx`
3. `console/src/App.tsx`
4. `console/src/layouts/registry/builtinRoutes.tsx`
5. `console/src/layouts/registry/builtinMenu.ts`
6. `console/src/layouts/MainLayout/index.tsx`
7. `console/src/layouts/Sidebar.tsx`
8. `console/src/pages/Chat/index.tsx`（只搜 `customFetch`、`AgentScopeRuntimeWebUI`、`options`）
9. `console/src/pages/Chat/sessionApi/index.ts`（只看 `listChats` / `getSession` 怎么调 API）
10. `console/src/pages/Chat/components/ChatSessionInitializer/index.tsx`

自己验证：`npm run dev` + `qwenpaw app`，从「新任务」发一句话，在 Network 里看到 `POST /api/console/chat`。

### Day 2 — API + 一个 Settings 页

1. `console/src/api/config.ts`、`request.ts`、`authHeaders.ts`
2. `console/src/api/modules/channel.ts`
3. `console/src/pages/Control/Channels/useChannels.ts`
4. `console/src/pages/Control/Channels/index.tsx`
5. `console/src/pages/SettingsCenter/index.tsx`（理解 `/settings/*` 和独立路由的关系）
6. 选读：`pages/Settings/Models/useProviders.ts` 或 `pages/Agent/Config/useAgentConfig.tsx`

自己验证：打开 `/channels`，改一项配置，Network 里出现 `PUT /api/config/channels/...`。

### Day 3 — OS / 插件 / 文案

1. `console/src/utils/navigationMode.ts`（`isOsPath`、`getOsRootHref`）
2. `console/src/os/DesktopOS.tsx`、`WindowRouter.tsx`、`osApps.ts`
3. `console/src/plugins/PluginContext.tsx`、`usePluginLoader.ts`、`registry/sdk.ts`
4. `console/src/i18n.ts`、`locales/en.json`、`locales/zh.json`
5. 桌面差异：`console/src/tauri/bootstrap.tsx`、`BackendReadyGate.tsx`、`console/src-tauri/tauri.conf.json`

自己验证：浏览器打开 `/os`，点 Dock 里的 Chat，确认仍是同一套 Chat 页，只是在窗口里。

---

## 13. 改代码时的约定

- **格式化**：`cd console && npm run format`（Prettier **3.0.0**）。仓库根的 pre-commit prettier **排除了** `console/`，所以 Console 的格式以这个 npm 脚本为准。
- **ESLint**：`npm run lint`，配置在 `console/eslint.config.js`（ESLint 9 + `typescript-eslint` + react-hooks）。`dist` 被 ignore。
- **i18n**：用户能看见的字不要写死在 JSX 里，用 `t("key")`，同时改 `en.json` 和 `zh.json`（其它语言有余力再补）。
- **不要提交构建产物**：`.gitignore` 含 `dist/`、`src/qwenpaw/console/`。Docker/CI 会在打包时 `npm run build`。
- **路径别名**：`@/` = `console/src/`（`vite.config.ts` 的 `resolve.alias`）。
- **行宽 / 引号**：前端跟 Prettier；不要把 Python 的 79 列硬套到 TSX。
- **页面结构**：新功能优先 `pages/<Feature>/index.tsx` + `use*.ts` + `components/`，API 放 `api/modules/`，类型放 `api/types/`。
- **新菜单 / 新路由**：加到 `builtinMenu.ts` / `builtinRoutes.tsx`（或插件 SDK），不要在 `MainLayout` 里手写一长串 `<Route>`。

---

## 14. 常见迷路点

**找不到路由**  
别只搜 `App.tsx`。外壳只有 `/login`、`/hub/admin`、`/*`。业务 path 在 `layouts/registry/builtinRoutes.tsx`，运行时在 `plugins/registry/store.ts` 的 `routeRegistry`。插件路由也在同一张表。

**找不到接口**  
别在页面里搜 URL 字符串。去 `console/src/api/modules/`。Chat 发送例外：直接写在 `pages/Chat/index.tsx` 的 `customFetch` 里，路径是 `/console/chat`。

**文案写死了**  
会出现「切到英文还是中文」或漏翻。必须 `t()` + 双语 JSON。`i18n.ts` 的 fallback 是英文。

**Chat 组件其实部分来自 `@agentscope-ai/chat`**  
你在 `pages/Chat/index.tsx` 里看不到气泡 DOM，因为那是 `AgentScopeRuntimeWebUI`。改气泡外观先看 `HostBubbles.tsx` 和 `customToolRenderConfig`；改发送协议看 `customFetch`；改历史列表看 `sessionApi`。测试里这个包被 alias 到 `src/test/chat-mock.ts`，所以单测里你看不到真实 SDK。

**侧边栏没有「Chat」菜单**  
设计如此。入口是「新任务」和会话列表。`builtinMenu.ts` 顶部注释专门说明了 `core.chat` 的 carve-out。

**设置页怎么有的在 `/channels` 有的在 `/settings/...`**  
两套入口，同一组件。Settings Center 用 `routeId` 把独立页面嵌进去。

**请求 401 突然跳登录**  
`request.ts` 的全局行为。先看 `localStorage.qwenpaw_auth_token` 和后端是否开了鉴权。

**换了 Agent 数据不对**  
检查 `X-Agent-Id`。`agentStore` 和 `authHeaders.ts` 必须指向同一个 id。

**开发环境 API 全失败**  
确认 `qwenpaw app` 已在 8088 监听，且 Vite 的 proxy target 仍是 `http://localhost:8088`。

**生产打开 `/` 提示 console 不可用**  
没构建或静态目录不对。按 `_app.py` 提示在 `console/` 执行 `npm ci && npm run build` 后重启。

---

## 附录：一张总图

```
用户浏览器
    │
    ├─ 开发: Vite :5173  ──proxy /api──► FastAPI :8088
    └─ 生产: FastAPI 同时托管 index.html + /api
                │
                ▼
         AuthGuard / PluginProvider
                │
        ┌───────┴────────┐
        │ /os            │ 其它路径
        ▼                ▼
   DesktopOS        MainLayout
   (窗口+Dock)      (Sidebar+Header)
        │                │
        └──── 同一套 routeRegistry ────┘
                         │
              /chat/* → ChatPage
                         │
              AgentScopeRuntimeWebUI (@agentscope-ai/chat)
                         │
              customFetch POST /api/console/chat  (SSE JSON 流)
                         │
              sessionApi  GET /api/chats
              ToolCards   工具调用可视化
```

读到这里，你已经有一条能自己走的路：入口 → 布局 → Chat 主路径 → API 习惯 → 再复制到任意设置页。遇到具体文件对不上时，以仓库源码为准，这份指南只负责指路。
