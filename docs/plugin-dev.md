# 插件开发指南

> 本文档描述如何为易游插件管理器（eyou）开发插件，涵盖 manifest.json 规范、内部 JS 插件、外部进程插件（Node/Python）、PluginContext API、前端 API、菜单/快捷键配置与完整示例。

---

## 1. 插件目录结构

```
plugins/{pluginId}/
├── manifest.json        ← 必填。元信息、UI 页面、菜单、快捷键
├── index.cjs            ← 内部插件入口（CommonJS，可选，外部插件不需要）
├── README.md            ← 可选。插件文档（上传到商店后展示在详情页）
└── dist/                ← 可选。前端 UI 构建产物（Vite 构建后输出）
    ├── index.html
    └── assets/
```

- `{pluginId}` 采用反向域名格式，如 `com.lx.packet_hijacker`；以字母/数字开头，仅含字母/数字/点/下划线/短横线，长度 ≤128，禁止 `..`（安装/加载侧会校验，防路径穿越）
- `manifest.json` **必填**（缺失则加载失败）；`index.cjs` 提供 `lifecycle` / `commands`
- `ui` / `menu` / `shortcuts` 等配置项**只能在 `manifest.json` 中声明**；入口文件（`index.cjs`）只实现代码逻辑（`lifecycle` / `commands`），不声明配置
- 推荐入口文件使用 `index.cjs` 扩展名（CommonJS），避免目录下存在 `package.json` 且 `"type": "module"` 时被当作 ESM 加载
- 外部插件 SDK（`eyou_sdk.js` / `eyou_sdk.py`）由客户端「开发工具」→「创建插件」创建项目时**自动生成**，无需手动获取
- 运行时插件目录为 `userData/plugins/`（Windows：`%APPDATA%\eyou\plugins\`，开发与打包一致）；开发阶段也可通过「开发工具」注册任意外部目录作为开发插件

> 插件打包由客户端「开发工具」的打包功能自动完成（无需手动打 zip）。

---

## 2. manifest.json 规范

### 2.1 完整示例

```json
{
  "id": "com.lx.my_plugin",
  "name": "我的插件",
  "version": "1.0.0",
  "type": "plugin",
  "description": "插件描述",
  "author": "lx",
  "tags": ["示例", "工具"],
  "entry": "index.cjs",
  "priority": 20,

  "ui": {
    "pages": [
      {
        "id": "home",
        "title": "我的插件窗口",
        "entry": "dist/index.html",
        "window": { "width": 900, "height": 650, "resizable": true, "alwaysOnTop": false },
        "menu": [
          {
            "id": "view",
            "label": "视图",
            "submenu": [
              { "id": "reload", "label": "刷新", "command": "reload" },
              { "id": "devtools", "label": "开发者工具", "command": "toggleDevTools" }
            ]
          }
        ]
      }
    ]
  },

  "menu": [
    { "id": "root", "label": "我的插件", "command": "openPage" }
  ],

  "shortcuts": [
    { "id": "f5-refresh", "key": "F5", "command": "reload", "scope": "window" },
    { "id": "f12-devtools", "key": "F12", "command": "toggleDevTools", "scope": "window" }
  ]
}
```

### 2.2 字段说明

#### 必填字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 全局唯一标识，如 `com.author.name`。以字母/数字开头，仅含字母/数字/点/下划线/短横线，长度 ≤128，禁止 `..` |
| `name` | string | 显示名称 |
| `version` | string | 语义化版本号 |

#### 可选字段

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `type` | `"plugin"` | `"plugin"` | 插件类型（仅支持插件） |
| `description` | string | `""` | 描述文本（不超过 50 字） |
| `author` | string | `""` | 作者名 |
| `tags` | string[] | `[]` | 标签列表 |
| `entry` | string | `"index.js"` | 内部插件 JS 入口文件名；外部插件不需要 |
| `command` | string | — | **外部插件**：完整命令行，如 `"python main.py"` / `"node main.js"`。存在则跳过 JS 入口加载 |
| `hideWindow` | bool | 开发隐藏/打包显示 | 外部插件进程是否隐藏控制台窗口；默认：开发环境隐藏、打包环境显示 |
| `priority` | int | `20` | 菜单排序优先级（越大越靠前） |
| `dependencies` | string[] | `[]` | 依赖的其他插件 ID 列表 |

#### ui.pages

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 页面标识（`ctx.ui.openPage(pageId)` 的参数） |
| `title` | string | 窗口标题 |
| `entry` | string | 页面入口：本地路径（`"dist/index.html"`）或 URL（`"http://localhost:5173/"`）。宿主只负责把该 HTML 文件加载进插件窗口，UI 如何开发完全自由（见下方说明） |
| `window.width` | int? | 窗口宽度 |
| `window.height` | int? | 窗口高度 |
| `window.resizable` | bool? | 是否可调整大小 |
| `window.alwaysOnTop` | bool? | 是否置顶 |
| `window.icon` | string? | 窗口图标（相对插件目录） |
| `menu` | array? | 窗口级菜单 |

> **UI 开发方式自由**：`entry` 指向的 HTML 文件可以是任意前端工具（Vite 等）的构建产物、直接手写的静态页，也可以是远程 URL（如本地 dev server），宿主不关心 UI 如何开发。UI 内通过预加载脚本暴露的 `window.$xxx` API 与宿主通信（见 §6）。
>
> 补充说明：
> - 窗口标题取 `window.title`，未指定时用页面 `title`
> - 同一插件的同一 `pageId` 重复调用 `ctx.ui.openPage` 会聚焦已有窗口，**不会新建**
> - `pageId` 未在 manifest 的 `ui.pages` 声明时，`openPage` 会抛错
> - 窗口启动参数含 `--pluginId` / `--pluginDir`，UI 侧 `$storage` / `$log` 据此定位插件上下文

#### menu（插件级与窗口级通用）

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 唯一标识 |
| `label` | string | 显示文本 |
| `enabled` | bool? | 是否可用 |
| `visible` | bool? | 是否可见 |
| `type` | `"normal"` \| `"separator"` \| `"checkbox"` \| `"radio"`? | 菜单项类型 |
| `command` | string? | 点击时触发的命令名 |
| `payload` | any? | 传给命令的额外参数 |
| `submenu` | array? | 子菜单 |

#### shortcuts

快捷键绑定到命令。宿主根据 `scope` 自动选择绑定方式：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 标识 |
| `key` | string | 按键（如 `"F5"`, `"R"`） |
| `modifiers` | array? | 修饰键（`["shift","control","alt","meta"]`） |
| `command` | string | 触发的命令名 |
| `payload` | any? | 额外参数 |
| `scope` | `"window"` \| `"global"` | 默认 `"window"`：绑定到插件窗口（不同窗口同键不冲突）；`"global"`：系统全局快捷键（先到先得） |

---

## 3. 内部 JS 插件（index.cjs）

### 3.1 入口文件

```js
module.exports = {
  lifecycle: {
    onLoad:    async (ctx) => { ctx.log.info("已加载"); },
    onEnable:  async (ctx) => { ctx.log.info("已启用"); },
    onDisable: async (ctx) => { ctx.log.info("已禁用"); },
    onUnload:  async (ctx) => { ctx.log.info("已卸载"); },
    onOpen:    async (ctx) => { await ctx.ui.openPage("home"); },
    onDaily:   async (ctx) => {
      // 每日任务逻辑（可选实现）
      ctx.log.info("执行每日任务");
    },
  },

  commands: {
    openPage: async (ctx) => ctx.ui.openPage("home"),
    myCommand: async (ctx, payload) => {
      ctx.log.info("收到命令 myCommand", payload);
      return { ok: true };
    },
  },
};
```

### 3.2 生命周期

| 钩子 | 触发时机 | 参数 | 说明 |
|------|---------|------|------|
| `onLoad(ctx)` | 插件被加载（启动时或手动加载） | ctx | 只执行一次，此时插件尚未启用 |
| `onEnable(ctx)` | 插件被启用 | ctx | 每次启用都执行 |
| `onDisable(ctx)` | 插件被禁用 | ctx | 每次禁用都执行 |
| `onUnload(ctx)` | 插件被卸载 | ctx | 只执行一次（退出或卸载时） |
| `onOpen(ctx)` | 用户点击"打开"按钮 | ctx | 打开 UI 的逻辑；未定义则不显示"打开"按钮 |
| `onDaily(ctx)` | 日常任务执行 | ctx | 可选；未定义则不显示在"日常任务"可选列表 |

执行顺序：`onLoad` → `onEnable` → … → `onDisable` → `onUnload`（退出时）。

### 3.3 命令处理器

命令函数签名：`(ctx, payload) => any`。命令通过菜单点击、快捷键、宿主/其他插件调用触发。

**内置命令**（由宿主提供，无需在入口中定义）：

| 命令 | 宿主行为 |
|------|---------|
| `reload` | `ctx.ui.reload(payload.windowId)` |
| `toggleDevTools` | `ctx.ui.toggleDevTools(payload.windowId)` |

> `payload.windowId` 由宿主**自动注入**，无需手动传参：
> - 窗口级快捷键（`scope: "window"`）触发时注入 `{ windowId, key, modifiers }`
> - 窗口级菜单（`ui.pages[].menu`）点击时注入 `{ windowId }`（与声明的 `payload` 合并）
> - 全局快捷键（`scope: "global"`）与插件级菜单（`menu`）**不注入** `windowId`

插件若自行定义了同名命令，以插件实现为准。

---

## 4. 外部进程插件（Node / Python）

外部插件是独立进程（Node/Python/任意可执行），通过 manifest 的 `command` 声明启动命令，宿主负责拉起进程并通过本地 TCP 网关与它通信。

### 4.1 manifest 示例

```json
{
  "id": "com.lx.my_script",
  "name": "外部脚本插件",
  "version": "1.0.0",
  "type": "plugin",
  "command": "python main.py",
  "hideWindow": true
}
```

- `command` 是完整命令行，支持引号包裹含空格的路径（如 `"C:\\Python\\python.exe" main.py`）
- `hideWindow`（默认：开发环境隐藏、打包环境显示）：`false` 时打包环境（宿主是无控制台的 GUI 进程）下 Windows 会为插件新建一个黑色控制台窗口；开发环境则挂到宿主控制台不弹新窗口。插件 stdout/stderr 始终走管道进入宿主日志中心，控制台窗口内不显示日志
- 启动参数除 §4.2 的 `--port` / `--id` / `--token` / `--dir` 外，`hideWindow: false` 时追加 `--console=1`

### 4.2 SDK

宿主 spawn 进程时传入网关连接参数：`--port` / `--id` / `--token` / `--dir`。进程连接网关后完成 hello → auth（一次性 token）→ 即可使用完整游戏 API。

- **Node**：`eyou_sdk.js` + `eyou_sdk.d.ts`
- **Python**：`eyou_sdk.py`

SDK 由客户端「开发工具」→「创建插件」创建外部插件项目时**自动生成**到项目目录，无需手动获取。

外部插件的生命周期钩子（onEnable / onDisable / onOpen / onDaily 等）由宿主通过 `system.lifecycle` 命令下发，SDK 自动映射为回调；插件在 hello 握手时向宿主声明已实现的阶段（UI 据此显示"打开"按钮）。

Node SDK 基本用法：

```js
const sdk = require("./eyou_sdk");
const plugin = sdk.createPlugin();

plugin.onEnable(async () => {
  await plugin.game.whenReady();
  const r = await plugin.game.req("game.status");
  plugin.log.info("status", r.value);
});

plugin.run();
```

`plugin.game` 与内部插件的 `ctx.game` 为同一套 API（whenReady / on / off / req / streamOpen）。

---

## 5. PluginContext API

`ctx` 是宿主提供给插件的受控能力集合：

```ts
type PluginContext = {
  host:    { isDev: boolean };
  plugin:  { id: string; dir: string };
  log:     { info(...a: any[]): void; warn(...a: any[]): void; error(...a: any[]): void };
  game:    GameAPI;
  packet:  { packPacket(p: Packet): string; unpackPacket(hex: string): Packet };
  ui:      UiAPI;
  menu:    { refresh(): void };
  storage: { get(key: string): Promise<any>; set(key: string, value: any): Promise<void> };
};
```

### 5.1 host / plugin

```ts
ctx.host.isDev;      // 宿主是否开发模式
ctx.plugin.id;       // 插件 ID
ctx.plugin.dir;      // 插件目录绝对路径
```

### 5.2 log

```ts
ctx.log.info(...args);
ctx.log.warn(...args);
ctx.log.error(...args);
```

日志自动带 `[plugin:{id}]` 前缀并路由到客户端日志中心（「日志」页可查看）。

### 5.3 game（游戏通信，EyouLink v1）

```ts
type GameAPI = {
  whenReady(timeout?): Promise<boolean>;   // 等待连接就绪（hello+auth 完成）；默认无限等待，传 timeout 超时返回 false
  onClosed(cb): () => void;                // 连接从 ready 掉线（重连前间隙）
  onReady(cb): () => void;                 // 每次就绪（含重连）触发，返回取消函数
  readonly status: "idle" | "connecting" | "ready" | "closed";
  on(topic, cb): () => void;               // 注册事件订阅者（跨插件多订阅者；game.* 由宿主转发 DLL 推送），返回退订函数
  off(topic, cb?): void;                   // 退订
  req(topic, data?, timeout?): Promise<Result>;  // 发布：本地有订阅者→调用全部并返回第一个返回值；否则走 DLL 命令
  streamOpen(filter, onData?): Promise<PacketStream>; // 打开监听/劫持流
};

type Result<T = any> = { ok: boolean; value: T | null; error: { code: string; message: string } | null };
```

- 连接状态机：`idle` → `connecting` → `ready`（hello + auth 通过）→ `closed`
- `req` 默认超时 30 秒；`whenReady` 默认无限等待（直到鉴权成功）
- 多订阅者：`req` 有本地订阅者时**并发调用全部**，返回第一个成功订阅者的返回值（哪怕 undefined）；全部失败则返回 `ok=false`（`INTERNAL`）

#### 订阅事件（DLL 推送）

| topic | data | 说明 |
|-------|------|------|
| `game.packet.recv` | `{ packet, cmdId, userId, length }` | 收到游戏封包 |
| `game.packet.sent` | `{ packet, cmdId, userId, length }` | 发出游戏封包 |
| `game.login` | `{ userId }` | 游戏登录成功 |
| `game.logout` | `{ userId }` | 游戏登出 |

#### 命令（req）

| topic | data | 说明 |
|-------|------|------|
| `game.status` | `{}` | 查询游戏状态 |
| `game.packet.send` | `{ packet }` | 发送封包（hex） |
| `game.speed.set` | `{ factor }` | 设置倍速 |
| `game.refresh` | `{}` | 重置游戏状态 |

#### 跨插件多订阅者（on / req）

`topic` 只是**事件名**，与插件无关。任何插件都能 `on` 同名事件，发布时**全部被调用**（多订阅者模式）：

```js
// 插件 A：注册事件处理器（暴露"接口"）
const unsub = ctx.game.on("someEvent", async (payload) => {
  // ...处理并返回结果
  return { code: 0, data: payload };
});

// 插件 B：发布并获取第一个订阅者的返回值（哪怕 undefined）
const res = await ctx.game.req("someEvent", { hello: "world" });
// res.value = 第一个订阅者的返回值（未定义则返回 undefined，ok 仍为 true）
```

- `on(topic, handler)`：注册订阅者；`game.*` 前缀是 DLL 事件（宿主自动订阅 DLL 并转发推送）。
- `req(topic, data)`：先查本地订阅者——有则并发调用全部，返回**第一个订阅者的返回值（哪怕 undefined）**；无订阅者则回退到 DLL 命令（`game.*`）。
- 不 `await` 的 `req` 即"发送不管"（fire-and-forget）。
- `game.*` 为 DLL 保留命名空间，插件事件请使用非 `game.*` 的事件名（建议 `业务.事件`，如 `command.request`）。
- **无权限 caps**：auth 通过即可访问插件全部能力，不存在能力限制。

#### command 插件示例（请求-响应接口）

内置 `command` 插件（`com.matchman33.command`）通过 `on` 暴露"向游戏发送指定 cmdId 并同步等待响应"的接口：

```js
// 其他插件调用：
const res = await ctx.game.req("command.request", {
  packet: "0000003F310000B8EC36605D75000001000000...",  // 完整请求封包（hex）
  options: { timeout: 5000 },  // 可选，默认 5000
});
// res.value = { cmdId, length, userId, result, body, raw }
//   result 非 0 = 游戏侧错误，仍正常返回由调用方判断
//   响应匹配：游戏固定返回与请求同号的响应 cmdId
```

#### 监听 / 劫持流（streamOpen）

```ts
type StreamFilter = {
  mode: "listen" | "hijack";
  cmds: number[] | "all";       // "all" 仅允许 listen
  direction: "send" | "recv" | "all";
  timeout?: number;              // 劫持等待响应的超时（ms）
};

type WatchPacket = { cmdId: number; length: number; userId?: number; direction: "send" | "recv"; packet: string } & { seq: number };

type PacketStream = {
  streamId: string;
  filter: StreamFilter;
  onData(cb: (pkt: WatchPacket) => void): void;
  ack(seq: number, action: HijackAction): void;  // 劫持模式必须回执
  close(): Promise<Result>;
};

type HijackAction = { action: "pass" } | { action: "drop" } | { action: "modify"; packet: string };
```

**监听封包**：

```js
const stream = await ctx.game.streamOpen({ mode: "listen", cmds: [1001, 2002], direction: "recv" });
stream.onData((pkt) => {
  ctx.log.info(`收到 cmdId=${pkt.cmdId}`, pkt.packet);
});
// 不再需要时：await stream.close();
```

**劫持封包**（必须调用 `stream.ack(seq, action)` 回执）：

```js
const stream = await ctx.game.streamOpen({ mode: "hijack", cmds: [1001], direction: "recv", timeout: 10000 });
stream.onData((pkt) => {
  if (pkt.cmdId === 1001) stream.ack(pkt.seq, { action: "drop" });   // 放行 / 丢弃
  else stream.ack(pkt.seq, { action: "modify", packet: "<修改后的完整封包 hex>" });
});
```

### 5.4 packet（封包工具，纯函数）

```ts
const p: Packet = { length, version, cmdId, userId, result, body };
const hex = ctx.packet.packPacket(p);       // Packet → hex
const parsed = ctx.packet.unpackPacket(hex); // hex → Packet
```

### 5.5 ui（窗口管理）

```ts
// 打开 UI 页面（必须先在 manifest 的 ui.pages 声明）
const { windowId } = await ctx.ui.openPage("home", { width: 800, height: 600 });
ctx.ui.closeWindow(windowId);
ctx.ui.reload(windowId);
ctx.ui.toggleDevTools(windowId);
ctx.ui.setAlwaysOnTop(windowId, true);
ctx.ui.setMenu(windowId, nodes);          // 动态设置窗口菜单
ctx.ui.getWindow(windowId);               // 获取 SeerWindow 实例（Electron API 级）

// 行为说明：
// - 同一插件的同一 pageId 重复 openPage 会聚焦已有窗口，不新建
// - 窗口标题取 window.title，未指定时用页面 title
// - pageId 未在 manifest 声明时 openPage 抛错
// - 窗口启动参数含 --pluginId / --pluginDir，UI 侧 $storage / $log 据此定位插件上下文
```

### 5.6 menu

```ts
ctx.menu.refresh();   // 强制重建应用菜单（启用/禁用后宿主自动处理，一般无需手动调用）
```

### 5.7 storage

持久化 key-value 存储，串行化写入保证数据安全。

```ts
const val = await ctx.storage.get("myKey");
await ctx.storage.set("myKey", { foo: 42 });
```

- 存储位置：`plugins/.storage/{pluginId}.json`
- 并发 `set()` 自动排队，避免 read-modify-write 竞态

---

## 6. 前端 API（window.$xxx）

插件 UI 页面（`ctx.ui.openPage` 打开的窗口）通过预加载脚本暴露的 API 与宿主通信。

### 6.1 $plugin

```ts
// 已安装列表（含 onOpen/onDaily 能力标记）
const list = await window.$plugin.getInstalled();
// [{ id, name, version, type, description, author, tags, enabled, dir, hasUi, hasOnOpen, hasOnDaily }]

// 安装 / 管理
await window.$plugin.installFromUrl("http://...");   // 云端 URL 下载安装
await window.$plugin.installFromFile();                // 本地 zip（弹窗选择）
await window.$plugin.enable(id);
await window.$plugin.disable(id);
await window.$plugin.open(id);
await window.$plugin.uninstall(id);
await window.$plugin.reload(id);

// 内置插件
const { plugins } = await window.$plugin.getBuiltinPlugins();  // [{ id, name, version, description, available, installed }]
await window.$plugin.installBuiltinAll({ enable: true });      // 一键安装全部未安装的内置插件
await window.$plugin.installBuiltin(id);                       // 安装单个内置插件

// 日常任务
const tasks = await window.$plugin.getDailyTasks();
await window.$plugin.saveDailyTasks([...]);
await window.$plugin.runDaily(id);   // 手动触发插件 onDaily

// 开发工具：创建 / 打包
await window.$plugin.createPlugin({ type: "node" });  // 创建插件项目（弹窗选择目录）
const { zipPath } = await window.$plugin.packPlugin(); // 打包当前选中项目为 zip

// 开发插件注册 / 重载（开发阶段用）
await window.$plugin.addDevPlugin();      // 选择外部目录注册为开发插件
await window.$plugin.reloadDevPlugin(id); // 强制热重载
```

### 6.2 $game

```ts
// 命令 / 状态
const res = await window.$game.req("game.status", {}, 3000);   // 统一 Result { ok, value, error }
const status = await window.$game.status();
await window.$game.whenReady(5000);

// 订阅事件（返回退订函数）
const unsub = window.$game.on("game.packet.recv", (data) => { ... });
window.$game.off("game.packet.recv");

// 连接就绪 / 掉线
window.$game.onReady(() => { ... });
window.$game.onClosed(() => { ... });

// 监听 / 劫持流
const stream = await window.$game.streamOpen({ mode: "listen", cmds: [1001], direction: "recv" });
stream.onData((pkt) => { ... });

// 封包工具（纯函数）
window.$game.packPacket(params);
window.$game.unpackPacket(hex);
```

> 提示：React 中把 `$game` 存到 `useRef`，避免闭包捕获旧值。
>
> `whenReady()` 默认无限等待（直到游戏连接就绪），传 timeout 超时返回 false。
> `streamOpen` 打开失败时（`ok=false`）会直接 throw Error。

### 6.3 $storage

插件上下文存储的渲染侧等价物（与 `ctx.storage` 同一存储位置）：

```ts
const val = await window.$storage.get("key");
await window.$storage.set("key", value);
```

### 6.4 $log

```ts
const entries = await window.$log.getRecent();
const unsub = window.$log.subscribe((entry) => { ... });
window.$log.info("message");
window.$log.warn("message");
window.$log.error("message");
window.$log.clear();
```

### 6.5 $settings

```ts
const { path, exists } = window.$settings.getDefaultPath();
const selected = await window.$settings.selectPath();
await window.$settings.launchGame();
await window.$settings.injectMod();
await window.$settings.restoreMod();
const { injected } = window.$settings.checkModStatus();
```

---

## 7. 完整示例

### 7.1 最小插件（无 UI，只有命令）

**manifest.json**：
```json
{
  "id": "com.lx.hello",
  "name": "Hello",
  "version": "1.0.0",
  "type": "plugin",
  "entry": "index.cjs"
}
```

**index.cjs**：
```js
module.exports = {
  lifecycle: {
    onLoad: async (ctx) => { ctx.log.info("Hello 已加载"); },
  },
  commands: {
    sayHello: async (ctx) => {
      ctx.log.info("Hello World!");
      return { message: "Hello World!" };
    },
  },
};
```

### 7.2 带 UI 的插件（Vite 前端）

> UI 开发方式自由：`entry` 指向的 HTML 可以是 Vite 等工具的构建产物（本示例）、直接手写的静态页，或远程 URL（如本地 dev server），宿主只负责把该文件加载进插件窗口。

**manifest.json**（ui 入口指向构建产物）：
```json
{
  "id": "com.lx.hello_ui",
  "name": "Hello UI",
  "version": "1.0.0",
  "entry": "index.cjs",
  "ui": {
    "pages": [{ "id": "home", "title": "Hello", "entry": "dist/index.html", "window": { "width": 600, "height": 400 } }]
  },
  "menu": [{ "id": "root", "label": "Hello UI", "command": "openPage" }],
  "shortcuts": [{ "id": "f5-refresh", "key": "F5", "command": "reload", "scope": "window" }]
}
```

**index.cjs**：
```js
module.exports = {
  lifecycle: {
    onOpen: async (ctx) => { await ctx.ui.openPage("home"); },
  },
  commands: {
    openPage: async (ctx) => ctx.ui.openPage("home"),
  },
};
```

**UI 内使用前端 API**（`dist/index.html` 的脚本里）：
```js
// 读取插件存储
const v = await window.$storage.get("myKey");
// 订阅游戏事件
window.$game.on("game.packet.recv", (data) => { /* ... */ });
// 发日志
window.$log.info("Hello from UI!");
```

### 7.3 外部 Node 插件

**manifest.json**：
```json
{
  "id": "com.lx.node_worker",
  "name": "Node 工作进程",
  "version": "1.0.0",
  "type": "plugin",
  "command": "node main.js",
  "hideWindow": true
}
```

**main.js**（配合 `eyou_sdk.js`）：
```js
const sdk = require("./eyou_sdk");
const plugin = sdk.createPlugin();

plugin.onEnable(async () => {
  await plugin.game.whenReady();
  const r = await plugin.game.req("game.status");
  plugin.log.info("当前状态", r.value);
});

plugin.run();
```

### 7.4 封包劫持（streamOpen + ack）

```js
module.exports = {
  lifecycle: {
    onEnable: async (ctx) => {
      // 劫持 cmdId 1001 的收包，10 秒内未回执自动放行
      const stream = await ctx.game.streamOpen({ mode: "hijack", cmds: [1001], direction: "recv", timeout: 10000 });
      stream.onData((pkt) => {
        if (pkt.cmdId === 1001) stream.ack(pkt.seq, { action: "drop" });
        else stream.ack(pkt.seq, { action: "pass" });
      });
    },
    onDisable: async () => { /* 宿主会自动关闭插件打开的流 */ },
  },
};
```

---

## 8. 最佳实践

1. **配置只在 manifest.json**：ui/menu/shortcuts 等配置项只能在 manifest.json 中声明，入口文件只写运行时逻辑（lifecycle / commands）
2. **`ctx.ui.openPage("home")` 统一入口**：插件打开窗口的唯一方式
3. **`ctx.log` 记日志**：不要用 `console.log`，否则不会路由到日志中心
4. **`ctx.storage` 持久化**：串行化安全写入，自动 JSON 序列化；UI 里用 `window.$storage`
5. **快捷键用顶层 `shortcuts[]`**：F5 刷新 / F12 DevTools 用内置命令 `reload` / `toggleDevTools`，无需自行实现；窗口级快捷键 / 窗口级菜单命令会自动注入 `windowId`
6. **劫持必须 `ack`**：`streamOpen` 劫持模式下，每条封包都要调用 `stream.ack(seq, action)`，否则 DLL 超时自动放行
7. **外部插件用 SDK**：Node/Python 插件创建项目时开发工具自动生成 `eyou_sdk`，宿主已处理网关连接与鉴权
8. **入口用 `.cjs`**：避免目录内 `package.json` 的 `"type": "module"` 把 CommonJS 入口误当 ESM
9. **类型提示**：入口函数用 JSDoc 标注 `@param {PluginContext}`，配合开发工具自动生成的类型声明获得代码提示
10. **脚手架**：客户端「开发工具」→「创建插件」可生成插件骨架（内部 JS / 外部 Python / 外部 Node）与外部插件 SDK
11. **热重载**：在「本地插件」面板点「重载」即可热加载，无需重启客户端
12. **插件 id 命名**：以字母/数字开头，仅含字母/数字/点/下划线/短横线，长度 ≤128，禁止 `..`
13. **打包交给开发工具**：插件打包由客户端「开发工具」自动完成，无需手动打 zip
