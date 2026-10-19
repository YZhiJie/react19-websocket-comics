# EP.02 React 里的连接

> 对应正文：[docs/02-websocket-basics.md](../docs/02-websocket-basics.md) · Hook：[src/hooks/use-reconnecting-websocket.ts](../src/hooks/use-reconnecting-websocket.ts) · 基础版：[src/demos/ws-basics/use-websocket-basic.ts](../src/demos/ws-basics/use-websocket-basic.ts)

## 剧情回顾

- 标签君把 `new WebSocket` 写进组件体，每次渲染多一条连接；
- 码叔给出四条铁律：ref 存实例、ref 转回调、connect 幂等、cleanup 必关；
- 像素酱在 StrictMode 下看到两张 101，慌了；
- 结案：双挂载是开发环境演练，两条 101 正常、消息处理两遍才是 bug。

## 知识点拆解

### 第 1 格 · 困境：越连越多

组件体每次渲染都执行，`new WebSocket` 属于命令式外部资源，放进渲染流程必然泄漏。必须放进 `useEffect`（或事件回调）。

### 第 2 格 · 揭秘：四条铁律

1. **ref 存实例**：`wsRef.current = new WebSocket(url)`，跨渲染持有同一连接；
2. **ref 转回调**：`onMessageRef.current = onMessage`，监听器永远调 `onMessageRef.current`，避免闭包捕获旧 state；
3. **connect 幂等**：入口先判 `if (wsRef.current) return`；
4. **cleanup 必关**：effect 返回清理函数 `ws.close()`。

### 第 3 格 · 实证：双挂载之谜

StrictMode 故意「挂载 → 清理 → 再挂载」来验证副作用可重入。时序：connect（ws#1）→ cleanup（close ws#1）→ connect（ws#2）。开发环境出现两张 101 是正常演练，生产不会双调用；若消息被处理两遍，说明监听注册了两遍。

### 第 4 格 · 结案：生命周期

连接与组件同生共死。`onclose` 里必须判断 `wsRef.current !== ws` 才清理状态，防止迟到的旧连接事件误杀新连接；cleanup 缺失会导致卸载后仍 setState。

## 坑点清单

- onclose 收到旧连接的事件：不匹配 `wsRef.current` 就直接忽略；
- cleanup 缺失：组件卸载后连接仍在，回调还在 setState；
- 事件监听注册两遍（StrictMode 下重新赋值 onmessage 除外），消息就会处理两遍。

## 自测 3 题（含答案）

<details><summary><b>1. 为什么 onMessage 要通过 ref 转发而不是直接注册 props 回调？</b></summary>

props 回调每轮渲染都可能变，而事件监听只在连接建立时注册一次；经 ref 转发后监听器永远调到最新闭包，既保持注册稳定又不捕获旧 state。

</details>

<details><summary><b>2. StrictMode 下出现两条 101 是 bug 吗？</b></summary>

不是。它是「挂载→清理→再挂载」的演练，第一条连接会被 cleanup 正确关闭。只有当同一条消息被处理两遍时才说明清理/注册逻辑有误。

</details>

<details><summary><b>3. onclose 里为什么要比较 wsRef.current !== ws？</b></summary>

旧连接的 close 事件可能迟到（半开链路、被看门狗判死后）。若不比较就执行清理/重连，会误伤已经建立的新连接，造成状态机错乱。

</details>
