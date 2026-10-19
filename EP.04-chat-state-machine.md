# EP.04 聊天室状态机

> 对应正文：[docs/04-realtime-app.md](../docs/04-realtime-app.md) · 状态层：[src/demos/chat/chat-reducer.ts](../src/demos/chat/chat-reducer.ts) · Provider：[src/demos/chat/chat-state.tsx](../src/demos/chat/chat-state.tsx) · 服务端：[server/main.ts](../server/main.ts)

## 剧情回顾

- 标签君被三个源头的事件流冲垮：连接生命周期、服务端推送、本地操作；
- 码叔揭秘 reducer：把「收到什么 → 怎么变」收敛成可单测的纯函数；
- 像素酱实证乐观消息与 clientId 回执替换；
- 结案：永不信任输入，状态以服务端对账为准。

## 知识点拆解

### 第 1 格 · 困境：状态打架

聊天域 9 类事件互相牵连：`open` 要补发挂起的 join、`history` 整体替换列表、`chat` 按 clientId 转正乐观消息、`presence` 全量同步名单。十几个 useState 各自为政必然打架。

### 第 2 格 · 揭秘：reducer 纯函数

`reducer(state, action)` 是纯函数，组件只管 `dispatch`；协议帧 → action 的翻译集中在 `onMessage`（见 chat-state.tsx），单帧 JSON 解析失败直接忽略，不影响连接。reducer 被 Vitest 直接单测（[tests/chat-reducer.test.ts](../tests/chat-reducer.test.ts)）。

### 第 3 格 · 实证：回执替换

本地消息先以 `pending(tmp-1)` 乐观上屏；服务端广播回执带 `clientId: 'tmp-1'`，客户端把该气泡**原位替换**为 `confirmed(id=7)`，不闪烁、不重排、不产生重复。发送失败的消息标记 `failed`，行内提供「重发」按钮（resend 用 useCallback 保持稳定引用，不破坏行级 memo）。

### 第 4 格 · 结案：永不信任

入参一律 trim + 长度钳制；房间成员用 Set 存储保证 O(1) 摘除、防幽灵在线；遥测定时器只在有订阅者时启动；未入房就发 chat 帧直接丢弃。服务端是单一事实源，客户端状态靠 history/presence 对账。

## 坑点清单

- 成员用数组存的幽灵在线问题——必须用 Set 摘除；
- 遥测定时器无条件启动，无订阅者时空转浪费；
- 未入房的 chat 帧不校验直接广播，会把垃圾写进历史。

## 自测 3 题（含答案）

<details><summary><b>1. 为什么乐观消息转正必须靠 clientId 而不是比对文本？</b></summary>

文本比对在重复消息、多人同名、并发编辑下不可靠；clientId 由发送方生成并随消息往返，是唯一能建立「本地乐观气泡 ↔ 服务端回执」对应关系的身份证，且服务端可用它做幂等去重。

</details>

<details><summary><b>2. 重连后为什么要重放 join 再靠 history 对账？</b></summary>

服务端连接是会话载体，断线即离开房间。重连后 open 回调重放 join 归位，随后服务端下发全量 history/presence，客户端整体替换状态——本地增量修补永远追不上服务端真相。

</details>

<details><summary><b>3. 为什么 resend 必须保持稳定引用？</b></summary>

resend 作为 prop 传入 memo 化的消息行，若每次渲染都生成新引用，行级 memo 全部失效，重发功能反而引发全列表重渲染。

</details>
