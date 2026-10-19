# EP.05 断线复活术

> 对应正文：[docs/05-advanced-best-practices.md](../docs/05-advanced-best-practices.md) · Hook：[src/hooks/use-reconnecting-websocket.ts](../src/hooks/use-reconnecting-websocket.ts) · 退避：[src/lib/backoff.ts](../src/lib/backoff.ts) · 服务端：[server/main.ts](../server/main.ts)

## 剧情回顾

- 像素酱遭遇假绿灯：进电梯再出来，连接灯还绿着，消息却发不出去；
- 码叔揭秘两层心跳：协议层 ping 服务端回收死链，应用层 ping 客户端探活；
- 标签君实证 full-jitter 指数退避，把重连时刻打散；
- 结案：会话恢复靠重放、对账与幂等三件套。

## 知识点拆解

### 第 1 格 · 困境：假绿灯

TCP 半开链路两端无感；NAT/代理 60~120 秒回收空闲连接；`readyState === OPEN` 但 `send` 石沉大海；服务重启时全员同一秒挤进重连。所以「连上了」不等于「可运维」，要自己验证「还活着」。

### 第 2 格 · 揭秘：两层心跳

服务端每 30s `ws.ping()` 巡检，两轮无 pong 就 `terminate()` 回收；客户端发应用层 `{type:'ping'}`，看门狗 45s 无**任何入站**即 `close(4000)`。关键设计：看门狗只负责 close，重连决策统一交给 onclose，避免两条路径互相 scheduleReconnect 打架。浏览器无法发协议层 ping，应用层心跳是唯一选择。

### 第 3 格 · 实证：抖动退避

`delay = random(0, min(2^n × 1s, 30s))`：0~1s → 0~2s → 0~4s → … 封顶 30s。full jitter 让断线用户的重连时刻均匀分布，服务器重启也能被温柔分散冲击。open 成功后 attempt 归零；`online` 事件可跳过等待立即重连。重连成功后重放 join，等 history 下发完成对账。

### 第 4 格 · 结案：恢复三前提

重放（join 重放会话语义）+ 对账（history/presence 整体替换）+ 幂等（clientId 去重）。三个竞态坑都要防：看门狗不自己重连、迟到的 close 直接忽略（`wsRef.current !== ws`）、手动关闭与自动重连严格互斥（close fallback 定时器兜住 close 事件不来的半开场景）。

## 坑点清单

- 看门狗回调里不要自己 scheduleReconnect——与 onclose 路径重复调度会雪上加霜；
- 迟到的 close 事件：连接指针不匹配就直接忽略；
- 手动 close 后又触发自动重连，是「用户说走、程序硬留」的经典竞态。

## 自测 3 题（含答案）

<details><summary><b>1. 为什么客户端不能用协议层 ping？看门狗为什么监听「任何入站」而不是 pong？</b></summary>

浏览器 WebSocket API 没有暴露 ping/pong 控制帧。看门狗监听任何入站数据（消息/pong 都算喂狗），比精确等 pong 更宽松也更接近真实——只要链路活着就不误杀。

</details>

<details><summary><b>2. full jitter 相比固定间隔重连好在哪？</b></summary>

固定间隔会让所有掉线用户同步重试（服务重启=全员同秒冲击）；full jitter 把每次重连时刻在 [0, min(2^n×base, cap)] 内均匀随机，退避仍指数增长但冲击被天然分散。

</details>

<details><summary><b>3. 看门狗触发后为什么要先 close(4000) 再等 onclose 重连？</b></summary>

单一决策点原则：close(4000) 触发标准 close 流程，重连只写在 onclose 一处。若看门狗直接重连，与 onclose 的重连形成双路径竞态（如迟到 close 又触发一次），连接数会失控。

</details>
