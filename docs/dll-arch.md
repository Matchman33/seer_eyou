# DLL 端架构文档

> seer_eyou_hook — 游戏进程注入 DLL，基于 Microsoft Detours 实现 API Hook，通过 TCP 3000 端口对外暴露 JSON 帧协议接口。

---

## 1. 技术栈

| 组件 | 说明 |
|------|------|
| **语言** | C++17 |
| **Hook 库** | Microsoft Detours |
| **JSON** | nlohmann/json |
| **通信** | TCP Socket (端口 3000，IpcHub) |
| **构建** | Visual Studio 2026 |

---

## 2. Hook 列表

DLL 注入后 Hook 以下 Windows API：

| 函数 | 用途 |
|------|------|
| `send` | 拦截游戏发出包，修改后转发 |
| `recvfrom` | 拦截游戏接收包，支持劫持 |
| `timeGetTime` | 倍速控制 |
| `GetTickCount` | 倍速控制 |
| `GetTickCount64` | 倍速控制 |
| `QueryPerformanceCounter` | 倍速控制 |
| `Sleep` | 倍速控制 |
| `QueryPerformanceFrequency` | 倍速控制 |
| `GetSystemTimeAsFileTime` | 倍速控制 |
| `NtQuerySystemTime` | 倍速控制 |

---

## 3. 协议规范

### 3.1 传输层

TCP 连接，JSON 帧协议。帧格式：**换行分隔**（`\\n`-delimited），每条 JSON 以 `\\n` 结尾。

`parse_packet_local` 按 `\\n` 分割帧，`last_pck` 缓存不完整行（处理 TCP 拆包）。

### 3.2 消息结构

```json
{
  "type": "emit | return | on | off | handshake",
  "eventName": "事件名",
  "data": { ... },          // 载荷
  "id": "客户端请求标识"      // return 消息回传时携带
}
```

### 3.3 消息类型

| type | 方向 | 说明 |
|------|------|------|
| `handshake` | DLL → Client | 连接建立后发送协议版本 |
| `emit` | Client → DLL | 发送命令，可带 id 等待 return 响应 |
| `return` | DLL → Client | emit 的响应，id 与请求匹配 |
| `on` | Client → DLL | 注册事件订阅 |
| `off` | Client → DLL | 取消事件订阅 |

### 3.4 协议版本握手

连接建立后：
```
DLL → Client: {"type":"handshake","eventName":"_protocol_version","data":{"version":1}}
Client → DLL: {"type":"emit","eventName":"_protocol_version","data":{"version":1}}
```


---

## 3.5 IpcHub 线程安全改进

### 锁外 send

`emit()` / `call_client()` / `return_packet()` 均在锁内收集目标 socket，锁外执行 `send()`。避免 TCP 背压时持锁阻塞整个 IpcHub。

### internal handler 锁外调用

`handle_packet` 在锁内查找 handler，复制 `std::function` 后在锁外调用。消除 handler 内部触发 `gHub.emit()` 时递归请求 `mtx` 的死锁风险。

### safe_send 分片重试

所有 `send()` 调用封装为 `safe_send()`：循环发送直至全部字节发出，遇 `SOCKET_ERROR` 记录并返回。

### 断开清理

`on_close()` 增加了 `wait_for_return` 遍历清理，防止连接断开后残留孤儿请求。

---

## 3.6 调试日志

配置文件 `seer_hook_config.json`（与游戏 exe 同目录）：

```json
{ "debug_log": false, "port": 3000 }
```

设为 `true` 后 DLL 写入 `seer_hook_debug.log`（同目录），每行记录：
- `RECV` — 收到客户端包（type + event + id）
- `SEND return` — 回复客户端
- `SEND broadcast` — 广播给其他客户端
- `CONNECT` — 新连接
- `INTERCEPT matched/waiting/response/timeout` — 劫持链路追踪

文件超过 5MB 自动截断保留末尾 1MB。生产环境关 `false`。

`port` 控制 DLL TCP 服务端监听端口，默认 `3000`。端口非法或缺失时回退到 `3000`。
---

## 4. DLL IPC 命令参考

所有命令通过 TCP 发送。格式：`{"type":"emit","eventName":"命令名","data":{...}}`

### 4.1 game.refresh

重置所有游戏状态（登录态、结果链、用户 ID 等）。

```
请求: {"type":"emit","eventName":"game.refresh","data":{}}
响应: {"type":"return","data":{"success":true}}
```

### 4.2 game.status

查询当前游戏状态。

```
请求: {"type":"emit","eventName":"game.status","data":{}}
响应: {"type":"return","data":{"success":true,"data":{"userId":12345,"isLogin":true,"speedFactor":1.0}}}
```

### 4.3 game.packet.send

发送封包到游戏服务器。DLL 自动替换 userId、重新计算 result，客户端只需提供封包 hex。

```
请求: {"type":"emit","eventName":"game.packet.send","data":{"packet":"0011000002..."}}
响应: {"type":"return","data":{"success":true}}

```

### 4.4 game.speed.set

设置游戏倍速。

```
请求: {"type":"emit","eventName":"game.speed.set","data":{"factor":2.0}}
响应: {"type":"return","data":{"success":true,"data":{"factor":2.0}}}
```

### 4.5 watch 流式监听（替代旧 intercept）

新版客户端通过 `client.watch()` 统一处理收包和发包的监听/劫持。DLL 端对应 `watch.open` / `watch.packet` / `watch.close` 协议事件。

> **已废弃**：`intercept.start` / `intercept.stop` / `intercept.response` / `sentIntercept.start` / `sentIntercept.response` 命令已不再使用。

### 4.9 sentIntercept.stop

停止发包劫持，清除 cmdId 映射和响应记录。

```
请求: {"type":"emit","eventName":"sentIntercept.stop","data":{}}
响应: {"type":"return","data":{"success":true}}
```

### 4.10 sentIntercept.response

对发包劫持事件做出响应。与 `intercept.response` 完全平行。

```
请求: {"type":"emit","eventName":"sentIntercept.response","data":{"id":"req_12345","action":"modify","packet":"..."}}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `id` | string | 劫持事件返回的 id |
| `action` | `"pass"` \| `"modify"` \| `"drop"` | pass=放行, modify=修改, drop=丢弃 |
| `packet` | string? | action=modify 时必填，修改后的封包 hex |

**drop 行为**：`return len`（游戏认为发送成功，实际未调用 `OriginalSend`）
**modify 行为**：替换 cipher → 重新 ParsePacket → 继续 CalculateResult → OriginalSend
**超时行为**：auto-pass（正常流程继续，封包正常发出）

> **SendPacketToGame 不受发包劫持影响**：客户端通过 `game.packet.send` 发送的封包在 `HookedSend` 开头被格式检查跳过（前两字节不满足 `0x00 0x00`），直接走 `return OriginalSend` 原路发出。

### 4.11 watch stream

`watch` 是新的封包流协议，使用 `type: "stream"` 创建有生命周期的数据流。它不会替代旧的 `game.packet.recv` / `game.packet.sent` 全量广播；旧广播仍保留，`watch` 用于按 `cmdId + direction` 精准推送。

注册：

```json
{"type":"stream","eventName":"watch.open","id":"watch_xxx","data":{"mode":"listen","cmds":[1001],"direction":"recv","timeout":3000}}
```

全量监听：

```json
{"type":"stream","eventName":"watch.open","id":"watch_xxx","data":{"mode":"listen","cmds":"all","direction":"all"}}
```

关闭：

```json
{"type":"stream","eventName":"watch.close","id":"watch_xxx","data":{}}
```

注册/关闭响应均为 `type: "return"`，`id` 与请求中的 stream id 相同。

监听模式规则：

- `mode: "listen"` 允许多个客户端监听同一个 `cmdId + direction`。
- `cmds: "all"` 表示全量监听；可与 `direction: "send" | "recv" | "all"` 组合。

劫持模式规则：

- `mode: "hijack"` 下同一个 `cmdId + direction` 只能有一个 stream。
- `direction: "all"` 会同时占用 send 和 recv 两个方向。
- `cmds: "all"` 在 hijack 模式下明确禁止。
- 劫持事件通过 `watch.packet` 推送，客户端通过 `type: "return"`, `eventName: "watch.packet"`, `id: requestId` 响应 `pass | modify | drop`。

---

## 5. DLL 推送事件

DLL 主动推送到客户端的 JSON 消息，客户端通过 `on(eventName, callback)` 订阅。

### 5.1 game.login

用户登录成功。
### 5.6 game.packet.sent.intercepted

**发包劫持事件**。当 sentIntercept 启用且封包 cmdId 匹配时触发。与 `game.packet.recv.intercepted` 完全平行。

```
{
  "type": "emit",
  "eventName": "game.packet.sent.intercepted",
  "data": {
    "id": "123456789_s_1001",   // 劫持请求唯一 ID（带 _s_ 前缀区分收/发）
    "cmdId": 1001,
    "length": 40,
    "packet": "0011000002..."    // 封包 hex
  }
}
```


```
{"type":"emit","eventName":"game.login","data":{"userId":12345}}
```


### 5.3 game.packet.recv

收到游戏封包（已解析）。

```
{
  "type": "emit",
  "eventName": "game.packet.recv",
  "data": {
    "packet": "0011000002...",  // 完整封包 hex
    "cmdId": 1001,
    "userId": 12345,
    "length": 40
  }
}
```

### 5.4 game.packet.sent

已发出的封包（HookedSend 处理后发出）。

```
{ "type": "emit", "eventName": "game.packet.sent", "data": { ...同上... } }
```

### 5.5 game.packet.recv.intercepted

**劫持事件**。当 intercept 启用且封包 cmdId 匹配时触发。客户端必须在 timeout 时间内通过 `intercept.response` 回复。

```
{
  "type": "emit",
  "eventName": "game.packet.recv.intercepted",
  "data": {
    "id": "123456789_1001",    // 劫持请求唯一 ID
    "cmdId": 1001,
    "length": 40,
    "packet": "0011000002..."  // 封包 hex
  }
}
```

---

## 6. 封包格式

游戏封包结构（封包应用层，非 JSON 帧协议）：

```
 字节 0-3: length (4 bytes, Big-Endian)
 字节 4:   version (1 byte)
 字节 5-8: cmdId (4 bytes, Big-Endian)
 字节 9-12: userId (4 bytes, Big-Endian)
 字节 13-16: result (4 bytes, Big-Endian)
 字节 17+: body (length - 17 bytes)
```

- 最小有效封包：17 字节
- cmdId > 1000 时，result 参与链式计算（Algorithm::MSerial）
- cmdId = 1001 为登录事件

---


## 7. 封包劫持实现细节（v2：per-connection + 收发双劫持）

### 7.1 架构概览

v2 将劫持从**全局广播式**重构为**per-connection 反查式**：

- 每个客户端连接的劫持状态独立（`gRecvIntercept[conn]` / `gSentIntercept[conn]`）
- cmdId → socket 反向映射（`gCmdIdToSocket` / `gSentCmdIdToSocket`），一个 cmdId 只对应一个连接
- 劫持事件**直接 `safe_send` 到目标连接**，不再通过 `gHub.emit` 广播
- 收包劫持 + 发包劫持两套平行结构

### 7.2 核心数据结构

```cpp
// 单个连接的劫持状态
struct InterceptState {
    bool        enabled = false;
    std::set<int> cmdIds;
    DWORD       timeoutMs = 3000;
    std::mutex  mtx;
    std::condition_variable cv;
    std::map<std::string, InterceptResponse> responses;
};

struct InterceptResponse {
    std::string action;                        // "pass" | "modify" | "drop"
    std::vector<BYTE> modifiedPacket;          // action=modify 时的新封包
};

// 收包劫持
static std::map<int, SOCKET> gCmdIdToSocket;         // cmdId → 发起劫持的 socket
static std::mutex gCmdIdToSocketMtx;
static std::map<SOCKET, InterceptState> gRecvIntercept;  // per-connection 状态
static std::mutex gRecvInterceptMtx;

// 发包劫持（平行结构）
static std::map<int, SOCKET> gSentCmdIdToSocket;
static std::mutex gSentCmdIdToSocketMtx;
static std::map<SOCKET, InterceptState> gSentIntercept;
static std::mutex gSentInterceptMtx;

// handler 调用上下文（IpcHub 在调用内部处理器前设置）
thread_local SOCKET tls_handler_conn = INVALID_SOCKET;
```

### 7.3 注册流程（intercept.start / sentIntercept.start）

```
客户端发送 intercept.start { cmds: [1001, 2002], timeout: 5000 }
  │
  ├─ IpcHub::handle_packet 设置 tls_handler_conn = conn
  ├─ handler 通过 tls_handler_conn 获取当前连接的 socket
  ├─ 查重：遍历 cmds，若已有其他连接注册 → 返回 Err("cmdId xxx already registered")
  ├─ 清理：删除本连接之前的所有 cmdId 映射
  ├─ 注册：gCmdIdToSocket[1001] = conn, gCmdIdToSocket[2002] = conn
  ├─ 初始化 gRecvIntercept[conn] 状态（enabled, cmdIds, timeoutMs）
  └─ 返回 Ok()
```

同一个连接重复调用 `intercept.start` 会覆盖之前的注册（先清后写）。不同连接注册相同 cmdId 会返回错误。

### 7.4 拦截流程（HookedRecvFrom）

```
HookedRecvFrom 收到数据:
  1. OriginalRecvFrom → 解析封包 → 提取 cmdId
  2. gCmdIdToSocket.find(cmdId) → 获取 targetConn
     ├─ 未找到 → 跳过劫持，正常 emit "game.packet.recv"
     └─ 找到 → 查 gRecvIntercept[targetConn]
  3. 若 state.enabled && cmdId 匹配：
     a. 构造 intercepted JSON → safe_send(targetConn, ...)
        （直接发给注册劫持的连接，不广播）
     b. unique_lock<mutex> + cv.wait_for(timeoutMs) 等待客户端响应
  4. 客户端响应到达（intercept.response handler）：
     └─ 遍历 gRecvIntercept，找到包含该 reqId 的 state
        → 写入 responses[id] → cv.notify_one()
  5. cv.wait_for 返回后：
     ┌─ pass   → 正常 emit "game.packet.recv"，继续
     ├─ modify → RecvBuf 中替换封包 → anyHijack = true
     ├─ drop   → explicitDrop = true，goto drop_packet
     └─ 超时    → emit "game.packet.recv" + goto drop_packet
```

### 7.5 发包劫持流程（HookedSend）

```
HookedSend 被游戏调用:
  1. 格式检查（buf[0]!=0x00 || buf[1]!=0x00 → 非封包数据 → 直接 OriginalSend 跳过）
  2. ParsePacket → 提取 cmdId
  3. gSentCmdIdToSocket.find(cmdId) → 获取 targetConn
     ├─ 未找到 → 跳过劫持
     └─ 找到 → 查 gSentIntercept[targetConn]
  4. 拦截逻辑：
     ┌─ pass   → 继续正常 CalculateResult → OriginalSend
     ├─ modify → 用修改后的 cipher 替换 → 继续 CalculateResult → OriginalSend
     ├─ drop   → return len（游戏认为发送成功，实际未发送）
     └─ 超时    → auto-pass（继续正常流程）
```

> **SendPacketToGame 不受劫持影响**：客户端通过 `game.packet.send` 发送的封包经 `SendPacketToGame` → `OriginalSend(gSocket, ...)` 进入 `HookedSend`，但其封包前两字节不满足 `0x00 0x00` 格式检查，直接在 `HookedSend` 开头被 `return OriginalSend` 跳过拦截逻辑。

### 7.6 线程安全设计

| 改进点 | 说明 |
|-------|------|
| handler 锁外调用 | `handle_packet` 在锁内查找 handler 并复制 `std::function`，锁外调用。消除 handler 内部 `gHub.emit()` 递归请求 `mtx` 的死锁 |
| safe_send 锁外发送 | `emit()` / `call_client()` / `return_packet()` 均在锁内收集 socket，锁外 `send()`。避免 TCP 背压持锁阻塞整个 IpcHub |
| cv.wait_for 不持 state.mtx | 仅用 `std::lock_guard` 检查状态后释放，再用 `std::unique_lock` 等待。避免同一 mutex 上 lock_guard + cv.wait_for 死锁 |
| 断连自动清理 | `IpcHub::on_close` → `cleanup_recv_intercept(conn)` + `cleanup_sent_intercept(conn)`，清理 cmdId 映射和 per-connection 状态 |
| cmdId 互斥 | `intercept.start` 检查重复注册，不同连接不能劫持同一 cmdId |

### 7.7 与 v1 的关键差异

| 方面 | v1（全局） | v2（per-connection） |
|------|----------|---------------------|
| 状态存储 | 单个全局 `gIntercept` | `gRecvIntercept[conn]` + `gSentIntercept[conn]` |
| cmdId→连接 | 无反向映射 | `gCmdIdToSocket` / `gSentCmdIdToSocket` |
| 事件发送 | `gHub.emit` 广播给所有订阅者 | `safe_send` 直接发给注册连接 |
| 发包劫持 | 不支持 | 支持（`HookedSend` 内 parallel 实现） |
| 多连接支持 | 不支持（全局状态互相覆盖） | 独立状态，cmdId 互斥 |
| 断连清理 | 无 | `on_close` → `cleanup_*_intercept` |

---

## 8. 构建与部署

1. 用 Visual Studio 2026 打开 `seer_eyou_hook.sln`
2. 选择 Release | x64 配置
3. 生成 → 输出 `seer_eyou_hook.dll`
4. 放入 `seer_eyou_client/dll/` 目录
5. 客户端通过「设置」页面 → 「注入 Mod」替换游戏目录的 `baselib.dll`
