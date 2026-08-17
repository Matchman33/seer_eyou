# 客户端架构文档

> seer_eyou_client — Electron 桌面客户端，易游插件管理器的主进程。管理插件生命周期、窗口、菜单，通过 TCP 3000 与 DLL 通信，通过日志总线统一收集日志。

---

## 1. 技术栈

| 层 | 技术 |
|----|------|
| 运行时 | Electron 28+ |
| 语言 | TypeScript 5.x |
| 模块系统 | CommonJS |
| 编译 | `tsc` → `dist/` |
| 打包 | electron-builder + NSIS |
| 自动更新 | electron-updater + GitHub Releases |

---

## 2. 项目结构

```
seer_eyou_client/
├── electron-builder.yml             ← electron-builder 打包配置
├── icon.png / icon_256x256.ico      ← 应用图标
├── index.ts                         ← 入口：启动 Electron、初始化插件、注册 IPC
├── tsconfig.json                    ← TypeScript 配置（outDir: dist）
├── package.json
├── src/
│   ├── preload.ts                   ← 预加载入口（聚合所有 API 模块）
│   ├── preload/                              ← 预加载子模块（7 个文件）
│   │   ├── shared.ts / game.ts / log.ts / plugin.ts / settings.ts / storage.ts / auth.ts
│   ├── ipc/
│   │   ├── gameClient.ts            ← TCP 客户端（连接 DLL，\\n 分隔帧协议）
│   │   ├── gameHandlers.ts           ← 游戏通信 IPC 桥接（渲染进程代理）
│   │   ├── pluginHandlers.ts        ← 插件安装/管理 IPC handler
│   │   └── settingsHandlers.ts      ← 设置/Mod IPC handler
│   ├── plugins/
│   │   ├── pluginManager.ts         ← 插件管理器（核心）
│   │   └── types.ts                 ← 类型定义（PluginContext、生命周期等）
│   └── utils/
│       ├── logBus.ts                ← 日志总线（环形缓冲 500 条）
│       ├── seerWindows.ts           ← BrowserWindow 封装
│       ├── packetUtils.ts           ← 封包解包/打包工具
│       └── safeUnzip.ts             ← zip 安全解压
├── plugins/                         ← 插件目录（每个插件一个子目录）
│   ├── .data/plugin_states.json     ← 插件禁用状态持久化
│   ├── .storage/{id}.json           ← 每个插件的 kv 存储
│   ├── com.lx.packet_interceptor/   ← 封包拦截器
│   ├── com.lx.log_viewer/           ← 日志查看器
│   ├── com.lx.command/              ← 内置命令封装
│   └── com.lx.packet_hijacker/      ← 封包劫持器
├── dll/                             ← DLL 文件（baselib.dll, gameServer.dll）
├── .data/                           ← 应用数据（daily_tasks.json 等）
└── release/                         ← 文档
```

---

## 3. 启动流程

```
app.requestSingleInstanceLock()
app.on("second-instance")
app.on("ready")
  → LogBus.getInstance()
  → PluginManager.getInstance("plugins")  // 单例
  → pluginManager.loadAll()               // 扫描 plugins/ 目录，加载所有插件
  → pluginManager.enableAll()             // 跳过持久化禁用列表中的插件
  → pluginManager.refreshMenu()           // 构建 Electron 应用菜单
  → new SeerWindow("http://localhost:5175/")  // 打开主窗口（plugin-manager-ui）
  → registerPluginHandlers()              // 注册所有 IPC handler
  → registerSettingsHandlers()            // 注册设置 IPC handler
  → registerGameHandlers()                // 注册游戏通信 IPC handler
```

---

## 4. 核心模块详解

### 4.1 PluginManager（`src/plugins/pluginManager.ts`）

单例管理器，是所有插件能力的入口。

#### 生命周期管理

| 方法 | 说明 |
|------|------|
| `loadAll()` | 扫描 `plugins/` 目录，对每个子目录调用 `loadOne()` |
| `loadOne(dir)` | 读取 `manifest.json` + `index.js`，创建 PluginContext，调用 `onLoad` |
| `enable(id)` | 调用 `onEnable`，从禁用列表移除，刷新菜单 |
| `disable(id)` | 清理游戏事件注册 → 调用 `onDisable` → 加入禁用列表 → 刷新菜单 |
| `openPlugin(id)` | 调用 `lifecycle.onOpen`（用户点击"打开"按钮或菜单项） |
| `runDailyPlugin(id)` | 调用 `lifecycle.onDaily`（日常任务执行） |
| `reloadPlugin(id)` | 热重载：禁用→关闭窗口→清除 require 缓存→重新加载→重新启用 |
| `enableAll()` | 加载持久化禁用列表，跳过已禁用的插件，其余全部启用 |
| `disableAll()` | 禁用所有插件 |
| `shutdown()` | 禁用→关闭窗口→卸载（app 退出前调用） |

#### loadPluginDefinition 流程

```
1. 读 manifest.json → 校验必填字段（id/name/version）
2. require(entryFile) → 获取 JS 模块
3. 拼装 PluginDefinition:
   - meta: 来自 manifest.json
   - lifecycle: 来自 JS 模块
   - commands: 来自 JS 模块
   - ui/menu/shortcuts: manifest.json，fallback JS 模块
4. manifest.json 不存在则加载失败（旧格式已弃用）
```

#### 菜单管理

| 方法 | 说明 |
|------|------|
| `buildMenuTemplate()` | 遍历所有启用插件，按优先级降序，转换 `definition.menu` 为 Electron menu 模板 |
| `refreshMenu()` | 构建模板 + `Menu.setApplicationMenu()` |
| `convertMenuNode()` | 递归转换 `PluginMenuNode` → `MenuItemConstructorOptions` |
| `dispatchCommand()` | 查找插件命令并执行（用于菜单项 click 事件） |

#### 窗口管理

| 方法 | 说明 |
|------|------|
| `ui.openPage(pageId)` | 打开插件 UI 页面。若已有同 pluginId+pageId 的窗口则聚焦置顶，不创建新窗口 |
| `ui.closeWindow(windowId)` | 关闭并销毁窗口 |
| `ui.reload(windowId)` | 刷新窗口 |
| `ui.toggleDevTools(windowId)` | 打开/关闭 DevTools |
| `ui.setAlwaysOnTop(windowId, flag)` | 设置窗口置顶 |
| `ui.getWindow(windowId)` | 获取 `SeerWindow` 实例 |
| `ui.setMenu(windowId, nodes)` | 设置窗口级菜单 |

窗口映射由 `pageWindows` Map（`pluginId:pageId → windowId`）维护，窗口关闭时自动清理。

#### 插件状态持久化

- 存储位置：`plugins/.data/plugin_states.json`（`{"disabled": ["id1","id2"]}`）
- `enable()` 时从 disabled 列表移除 → 写文件
- `disable()` 时加入 disabled 列表 → 写文件
- `enableAll()` 先读状态，跳过 disabled 中的插件

#### 安装/卸载

| 方法 | 说明 |
|------|------|
| `installPlugin(zipPath)` | 解压 zip → 读 manifest.json 获取 id → 覆盖写入 `plugins/{id}/` → loadOne → enable |
| `uninstallPlugin(id)` | disable → 关闭窗口 → 删除目录 |

---

### 4.2 GameClient（`src/ipc/gameClient.ts`）

TCP 客户端，连接 DLL 的 3000 端口，使用 `\\n` 分隔 JSON 帧协议。**懒连接** — 构造时不自动连接，需显式 `connect()` 或首次 `emit()` 触发。

#### 核心特性

- **`\\n` 分隔帧协议**：`LineFrameParser` 按 `\\n` 分割帧。出站消息自动追加 `\\n`
- **懒连接**：`getInstance()` 同步返回，TCP 连接需显式 `connect()` 触发。`on()` / `watch()` 只记录监听器不触发连接，`emit()` 自动连接并排队发送
- **自动重连**（固定 5 秒间隔，永不终止）。`disconnect()` 显式停止
- **状态事件**：`onWhenReady(cb)` / `onClosed(cb)` 连接状态变化回调，均返回取消注册函数
- 15 秒心跳保活
- 协议版本握手（连接建立后接收 DLL 的 `_protocol_version`）
- 30 秒请求超时（emit 带 id 时，超过 REPLY_TIMEOUT_MS 自动回调 error）
- 事件注册/注销机制（`on`/`off` → TCP `on`/`off` 消息）
- stream watch 机制（`watch` → TCP `stream/watch.open`，按 `cmdId + direction` 精准监听或劫持）
- 连接关闭时自动清理所有等待中的回调

#### API

```ts
class GameClient {
  // ---- 静态工厂 ----
  static getInstance(port?, ip?): GameClient;  // 同步，不等待连接

  // ---- 实例方法 ----
  readonly status: "idle" | "connecting" | "running" | "closed";
  protocolVersion: number | null;

  connect(): void;                            // 显式建立连接（开始重连）
  disconnect(): void;                         // 断开并停止重连
  destroy(): void;                            // 销毁实例（应用退出）

  on(eventName, callback): () => void;        // 注册事件（同事件支持多个回调，返回取消函数）
  off(eventName, callback?): void;            // 注销指定回调或该事件的全部回调
  emit(eventName, params?, callback?): void;  // 发送命令（未连接时自动 connect）
  watch(mode, cmdIds, direction, callback, options?): Promise<PacketWatcher>;

  whenReady(): Promise<void>;                 // 等待连接就绪（无超时）
  onWhenReady(cb): () => void;                // 连接就绪回调，返回取消函数
  onClosed(cb): () => void;                   // 连接关闭回调，返回取消函数
}
```

构造器为 `private`，外部只通过 `GameClient.getInstance()` 获取实例。重连成功时自动重新注册所有 `registerEvents` 中的事件并触发 `onWhenReady`。

---

### 4.3 LogBus（`src/utils/logBus.ts`）

主进程日志总线，环形缓冲 500 条，四种级别。

#### 获取实例

```ts
const logBus = LogBus.getInstance();
```

#### API

| 方法 | 说明 |
|------|------|
| `info(pluginId, message)` | 记录 info 级别日志 |
| `warn(pluginId, message)` | 记录 warn 级别日志 |
| `error(pluginId, message)` | 记录 error 级别日志 |
| `debug(pluginId, message)` | 记录 debug 级别日志 |
| `getRecent()` | 返回缓冲区副本（页面轮询用） |
| `clear()` | 清空缓冲区 |
| `subscribe(callback)` | 注册回调，返回取消函数 |

#### 日志写入来源

| 来源 | 方式 |
|------|------|
| 主进程插件 `ctx.log.info()` | `PluginManager.createContext` 中直接调用 LogBus |
| 渲染进程 `window.$log.info()` | `ipcRenderer.send("log:forward")` → 主进程 `LogBus.getInstance().info()` |
| GameClient | `LogBus.getInstance().info("GameClient", ...)` |
| index.ts 入口 | 同上，pluginId="Main" |

#### 订阅者通知

- `logBus.subscribe()` — 同进程注册（index.ts 中用此方式，广播给所有订阅了 `log:entry` IPC 的渲染进程）
- `ipcMain.handle("log:getRecent")` — 页面主动拉取
- `ipcMain.on("log:subscribe")` — 渲染进程订阅 IPC 推送

---

### 4.4 SeerWindow（`src/utils/seerWindows.ts`）

`BrowserWindow` 的轻量封装。

```ts
class SeerWindow {
  pluginId: string;  // 关联的插件 ID

  constructor(url, options?, additionalArguments?);
  get browserWindow(): BrowserWindow;
  get webContents(): WebContents;
  on(eventName, listener): this;
  reload(): void;
  close(): void;
  setAlwaysOnTop(flag): void;
}
```

关键配置：
- `contextIsolation: true`
- `sandbox: false`
- `preload: path.join(__dirname, "../preload.js")`（所有窗口共享同一个预加载脚本）
- 支持 `http://` 和本地文件加载

---

### 4.5 预加载脚本（`src/preload.ts`）

通过 `contextBridge.exposeInMainWorld` 暴露 6 个 API 到渲染进程：

> **TODO（待补充）**：此处声称暴露 6 个 API，下表仅列出 5 个（预加载子模块含 `auth.ts`，疑似第 6 个为 `$auth`），待核对补充。

| API | 暴露方式 | 用途 |
|-----|---------|------|
| `$game` | IPC 桥接 | DLL 通信、封包解包、插件存储（通过 gameHandlers 代理主进程单例） |
| `$plugin` | IPC → 主进程 | 插件安装/卸载/启用/禁用/内置插件/日常 |
| `$log` | IPC → 主进程 | 日志读写、订阅推送 |
| `$settings` | 部分直连 + IPC | 游戏路径/Mod 注入/启动游戏 |
| `$storage` | preload 直连 | 插件上下文存储（plugins/.storage/{id}.json） |

详见 [插件开发指南](./docs/plugin-dev.md#前端-apiwindowxxx)。

---

### 4.6 IPC Handler

#### gameHandlers.ts

`game:xxx` 通道的 handler，将渲染进程的 `$game` 调用代理到主进程 GameClient 单例。

| 通道 | 方向 | 说明 |
|------|------|------|
| `game:handshake` | invoke | 注册窗口，返回当前连接状态 |
| `game:connect` | invoke | 代理 `GameClient.connect()` |
| `game:disconnect` | invoke | 代理 `GameClient.disconnect()` |
| `game:status` | invoke | 查询当前连接状态 |
| `game:when-ready` | invoke | 代理 `GameClient.whenReady()` |
| `game:on` / `game:off` | invoke | 旧 IPC 桥接路径，当前 `$game` 已改为 preload 直连 GameClient |
| `game:emit` | invoke | 代理 `GameClient.emit()` |
| `game:on-when-ready` / `game:on-closed` | send | 订阅连接状态推送 |

事件监听机制：`GameClient` 对同一个 `eventName` 维护多个本地回调；向 DLL 只注册一次该事件，收到 DLL 推送后依次触发所有回调。插件禁用时应调用 `on()` 返回的取消函数，只移除自己的回调；最后一个本地回调移除后才向 DLL 发送 `off`。旧 IPC 桥接会为每个 `eventName` 在 GameClient 上只注册一次 forwarder。当前推荐路径是 preload 直接创建 GameClient，渲染进程拥有自己的 TCP 连接，可直接使用 `watch()`。

#### pluginHandlers.ts

所有 `plugin:xxx` 通道的 handler，负责转发插件的安装/管理请求。

| 通道 | 方向 | 说明 |
|------|------|------|
| `plugin:installFromUrl` | invoke | 下载远程 zip → 解压 → 根据 manifest.type 分流安装 |
| `plugin:installFromFile` | invoke | 打开文件对话框 → 选 zip → 安装 |
| `plugin:getInstalled` | invoke | 返回 `pluginManager.getInstalledPlugins()` |
| `plugin:enable/disable/open/uninstall` | invoke | 转调 pluginManager 对应方法 |
| `plugin:runDaily` | invoke | 转调 `pluginManager.runDailyPlugin()` |

**安装逻辑**：仅支持 `type=plugin` 的 zip，解压到 `plugins/{pluginId}/` 并加载（魔法已移除）。

#### settingsHandlers.ts

| 通道 | 说明 |
|------|------|
| `settings:selectPath` | 打开文件夹选择对话框 |
| `settings:launchGame` | `shell.openPath("Seer.exe")` |
| `settings:injectMod` | 复制 `dll/baselib.dll` + `dll/gameServer.dll` 到游戏目录，备份原始 baselib.dll |
| `settings:restoreMod` | 还原 `baselib_orig.dll`，删除 `gameServer.dll` |

---


## 5. 主窗口

主窗口 URL 根据运行环境自动切换：

| 环境 | URL | 说明 |
|------|-----|------|
| 开发 (
pm start) | http://localhost:5175/ | 连接 plugin-manager-ui 的 Vite dev server |
| 生产（打包后） | ile://resources/ui/index.html | 加载本地构建产物 |

打包时 plugin-manager-ui/dist/ 通过 extraResources 复制到 esources/ui/。

快捷键：
- F5 → 刷新主窗口
- F12 → 打开/关闭 DevTools

---

## 6. 配置常量

| 常量 | 值 | 说明 |
|------|----|------|
| DEFAULT_GAME_PATH | `D:\SeerLauncher\games\NewSeer` | 默认游戏目录 |
| GAME_EXE | `Seer.exe` | 游戏可执行文件 |
| DLL 源目录 | `seer_eyou_client/dll/` | 项目内 DLL 存放位置 |
| LogBus MAX_BUFFER | 500 | 日志环形缓冲大小 |
| GameClient 端口 | 从 `.data/setting.json` 的 `gamePath` 下读取 `seer_hook_config.json.port`，默认 3000 | DLL TCP 端口 |
| GameClient 重连间隔 | 5s | 固定间隔，永不终止 |
| GameClient REPLY_TIMEOUT | 30000ms | emit 请求超时 |

---

## 7. 数据流

```
                   ┌──────────────────┐
                   │  plugin-manager- │
                   │  ui (Vite :5175) │  ← 主窗口
                   └────────┬─────────┘
                            │ window.$xxx API
                   ┌────────┴─────────┐
                   │   preload.ts     │
                   │ contextBridge    │
                   └───┬─────────┬────┘
            direct TCP│   ipcRenderer
                ┌─────┴─────┐  ┌──────────────┐
                │ GameClient│  │ IPC Handlers │
                │ (TCP:port)│  │ plugin/settings
                └───────────┘  └──┬───────────┘
                     │            │
              ┌─────┴────────────┴──────────┐
              │       PluginManager          │
              │  (生命周期/命令/窗口/菜单)       │
              └─────────────────────────────┘
                            │
                    ┌───────┴────────┐
                    │  DLL (C++)     │
                    │  TCP Server    │
                    │  Hooked API    │
                    └────────────────┘
```
