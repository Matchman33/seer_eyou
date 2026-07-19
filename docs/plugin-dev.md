# 🧩 插件开发指南

> 本文档描述如何为易游插件管理器开发插件。涵盖 manifest.json 规范、index.js 规范、PluginContext API 以及完整示例。

---

## 1. 插件目录结构

```
plugins/{pluginId}/
├── manifest.json        ← 必填。元信息、UI 页面、菜单、快捷键
├── index.js             ← 必填。生命周期、命令处理器
├── icon.png             ← 可选。插件窗口图标
├── README.md            ← 可选。插件文档
└── ui/
    └── index.html       ← 可选。内嵌 UI 页面
```

- `{pluginId}` 采用反向域名格式，如 `com.lx.packet_hijacker`
- `index.js` 是 CommonJS 模块（`module.exports = { ... }`）
- `manifest.json` 必填，不存在则插件加载失败

---

## 2. manifest.json 规范

### 完整示例

```json
{
  "id": "com.lx.my_plugin",
  "name": "我的插件",
  "version": "1.0.0",
  "type": "plugin",
  "description": "插件描述",
  "author": "lx",
  "tags": ["示例", "工具"],
  "onOpen": "openPage",
  "entry": "index.js",
  "minHostVersion": "2.0",
  "priority": 0,
  "dependencies": ["com.lx.command"],

  "ui": {
    "pages": [
      {
        "id": "home",
        "title": "我的插件窗口",
        "entry": "ui/index.html",
        "window": {
          "width": 900,
          "height": 650,
          "resizable": true,
          "alwaysOnTop": false,
          "icon": "icon.png"
        }
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

### 必填字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 全局唯一标识，如 `com.author.name` |
| `name` | string | 显示名称 |
| `version` | string | 语义化版本号 |
| `type` | `"magic"` \| `"plugin"` | 插件类型 |

### 窗口配置 (ui.pages[].window)

| 字段 | 类型 | 说明 |
|------|------|------|
| `width` | number | 窗口宽度 |
| `height` | number | 窗口高度 |
| `resizable` | boolean | 是否可调整大小 |
| `alwaysOnTop` | boolean | 是否置顶 |
| `title` | string | 窗口标题（覆盖 page.title） |
| `icon` | string | 窗口图标路径（相对插件目录） |

---

## 3. index.js 规范

### 完整示例

```js
module.exports = {
  lifecycle: {
    onLoad:    async (ctx) => { ctx.log.info("已加载"); },
    onEnable:  async (ctx) => { ctx.log.info("已启用"); },
    onDisable: async (ctx) => { ctx.log.info("已禁用"); },
    onUnload:  async (ctx) => { ctx.log.info("已卸载"); },
    onOpen:    async (ctx) => { await ctx.ui.openPage("home"); },
    onDaily:   async (ctx) => { ctx.log.info("执行每日任务"); },
  },

  commands: {
    openPage: async (ctx) => ctx.ui.openPage("home"),
    reload: async (ctx, payload) => {
      if (payload?.windowId) ctx.ui.reload(payload.windowId);
    },
    toggleDevTools: async (ctx, payload) => {
      if (payload?.windowId) ctx.ui.toggleDevTools(payload.windowId);
    },
  },
};
```

### 生命周期钩子

| 钩子 | 触发时机 | 说明 |
|------|---------|------|
| `onLoad(ctx)` | 插件被加载 | 只执行一次 |
| `onEnable(ctx)` | 插件被启用 | 每次启用都执行 |
| `onDisable(ctx)` | 插件被禁用 | 每次禁用都执行 |
| `onUnload(ctx)` | 插件被卸载 | 只执行一次 |
| `onOpen(ctx)` | 用户点击"打开" | 对应 UI 打开逻辑 |
| `onDaily(ctx)` | 日常任务执行 | 可选实现 |

---

## 4. PluginContext API

插件通过 `ctx` 参数访问宿主能力：

```ts
interface PluginContext {
  plugin: { id: string; dir: string };
  log: { info(...args); warn(...args); error(...args); };
  
  ui: {
    openPage(pageId, options?) → Promise<{ windowId: string }>;
    closeWindow(windowId) → Promise<void>;
    reload(windowId);
    toggleDevTools(windowId);
    setAlwaysOnTop(windowId, flag);
  };

  game: {
    getClient() → GameClient;
    packPacket(params) → string;
    unpackPacket(packet) → Packet;
  };

  storage: {
    get(key) → Promise<any>;
    set(key, value) → Promise<void>;
  };

  menu: { refresh() };
}
```

---

## 5. 打包与分发

插件打包为 `.zip` 文件，根目录包含 `manifest.json`：

```
my-plugin-v1.0.0.zip
├── manifest.json
├── index.js
├── icon.png
└── ui/
    └── index.html
```

用户可在客户端内通过「本地文件安装」或「云端市场」导入。
