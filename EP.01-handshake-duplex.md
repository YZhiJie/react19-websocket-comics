# EP.01 全双工登场

> 对应正文：[docs/02-websocket-basics.md](../docs/02-websocket-basics.md) · 服务端：[server/echo.ts](../server/echo.ts) · 前端：[src/demos/ws-basics/echo-demo.tsx](../src/demos/ws-basics/echo-demo.tsx)

## 剧情回顾

- 标签君被轮询折磨：每 2 秒 GET 一次，服务器收到大量空响应；
- 码叔揭秘握手：带 Upgrade 头借道 HTTP，101 应答后切换为帧协议；
- 像素酱盘点四事件与 readyState 状态机；
- 结案：帧只管传输，业务语义用统一的 JSON 信封协议。

## 知识点拆解

### 第 1 格 · 困境：轮询太累

| 方案 | 原理 | 代价 |
| ---- | ---- | ---- |
| 短轮询 | 定时 GET | 大量空响应、头部开销数百字节 |
| 长轮询 | 请求挂起，有数据再重连 | 频繁建连、易超时 |
| SSE | 单向推送 | 上行仍要走 HTTP |
| WebSocket | 全双工帧协议 | 帧头仅 2~14 字节 |

### 第 2 格 · 揭秘：HTTP 升级

握手就是一次特殊的 HTTP 请求：`Upgrade: websocket` + `Sec-WebSocket-Key`，服务端返回 `101 Switching Protocols` 与 `Sec-WebSocket-Accept`（key 经固定 GUID 做 SHA-1）。此后同一条 TCP 连接与 HTTP 无关。`ws://` 是明文、`wss://` 走 TLS——握手借道 HTTP 所以能复用 Cookie、鉴权头与反向代理，但 Accept 头不是加密也不是鉴权。

### 第 3 格 · 实证：四事件四态

`open / message / close / error` 四事件 + `readyState` 状态机（CONNECTING → OPEN → CLOSING → CLOSED）。关闭码里 `1000` 是正常关闭、`1006` 是异常断开（没有 close 帧）、`4000~4999` 留给应用自定义（本项目看门狗用 4000）。发送前判断 `ws.readyState === OPEN`，否则 `send` 直接抛异常。

### 第 4 格 · 结案：信封协议

帧协议不承载语义，业务消息统一用 JSON 信封：`{ type, ...payload }`，客户端与服务端按 `type` 分发。见 [src/lib/protocol.ts](../src/lib/protocol.ts)。

## 坑点清单

- https 页面连 `ws://` 会被混合内容策略拦截，上线一律 `wss://`；
- `1006` 表示异常断开（浏览器禁止代码里手造 1006，它只会被动出现）；
- CONNECTING 状态调用 `send` 直接抛异常，不是排队。

## 自测 3 题（含答案）

<details><summary><b>1. 为什么 WebSocket 握手要走一次 HTTP？</b></summary>

复用 HTTP 生态：端口（80/443）、反向代理、Cookie/鉴权头、防火墙策略都天然兼容；101 之后才切换为帧协议。

</details>

<details><summary><b>2. Sec-WebSocket-Accept 的作用是什么？</b></summary>

证明服务端理解 WebSocket 协议（对 key + 固定 GUID 做 SHA-1 Base64）。它既不加密也不鉴权，安全依赖 wss 与应用层鉴权。

</details>

<details><summary><b>3. 1000 和 1006 关闭码分别代表什么？</b></summary>

1000 = 正常关闭（双方约定完成）；1006 = 异常断开（连接死掉、没收到 close 帧），是生产环境最常见的掉线码。

</details>
