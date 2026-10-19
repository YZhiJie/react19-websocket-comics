# EP.06 高频不卡屏

> 对应正文：[docs/05-advanced-best-practices.md](../docs/05-advanced-best-practices.md) · 看板：[src/demos/dashboard/dashboard-page.tsx](../src/demos/dashboard/dashboard-page.tsx) · 协议：[src/lib/protocol.ts](../src/lib/protocol.ts) · 测试：[tests/telemetry-codec.test.ts](../tests/telemetry-codec.test.ts)

## 剧情回顾

- 标签君的看板被 50Hz 推送打爆：每帧 setState，React 每秒渲染 50 次；
- 码叔揭秘 rAF 合批：帧落 ref 零渲染，rAF 整批取空再 flush；
- 像素酱实证 memo 分层与消息窗口化；
- 结案：ErrorBoundary 四层防线兜住渲染期异常。

## 知识点拆解

### 第 1 格 · 困境：帧帧 setState

onmessage 里直接 setState，网络帧率 50Hz ≠ 屏幕刷新率 60Hz——一半渲染是白算的，输入延迟飙升。推多少不重要，用户看到多少才重要。

### 第 2 格 · 揭秘：rAF 合批

onmessage 只做 `ref.push()`（零渲染）；requestAnimationFrame 循环里把缓冲整批 dispatch。rAF 与 vsync 对齐：一个刷新周期最多渲染一次，标签页隐藏时自动停摆（setInterval 后台照样空转，不是正确节拍器）。

### 第 3 格 · 实证：memo 分层

Page → 4 个 Slot（useMemo 切片）→ MetricCard（memo）→ Sparkline。消息列表「存 1000 渲染 60」：全量留存供翻阅，窗口外不进 DOM，行级 memo 隔离无关更新。回调经 ref 转发保持稳定引用，memo 才不白开；memo 有比较成本，用 Profiler 验证收益。

### 第 4 格 · 结案：兜底防线

页面级 ErrorBoundary 兜住**渲染期**异常，resetKey 让切页自动恢复；React 19 的 `onUncaughtError` 全局上报兜底。边界接不住 onmessage/事件回调里的错误——那类错误用 try/catch 就地消化。

## 坑点清单

- onmessage 里直接 setState：高频通道直通渲染，主线程被淹没；
- setInterval 当节拍器：频率拍脑袋、后台空转，rAF 才与屏幕同频；
- onmessage 抛错想靠 ErrorBoundary 接住——边界只管渲染期错误。

## 自测 3 题（含答案）

<details><summary><b>1. 为什么 rAF 合批优于 setInterval 节流？</b></summary>

rAF 与浏览器 vsync 对齐，一个刷新周期最多 flush 一次，渲染永远发生在「屏幕即将绘制」的时机；标签页隐藏时自动停摆省资源。setInterval 频率靠猜、后台照跑，容易过度渲染或白白空转。

</details>

<details><summary><b>2. 「存 1000 渲染 60」怎么实现？</b></summary>

状态层全量留存最近 1000 条（供对账与翻阅），渲染层只切出最近 60 行进 DOM；配合行级 memo（行组件用 memo 包裹、回调稳定引用），新消息只重渲染受影响的行。

</details>

<details><summary><b>3. ErrorBoundary 接不住哪些错误？分别怎么兜？</b></summary>

事件回调、onmessage、异步任务里的错误：用 try/catch 就地消化；渲染期/生命周期错误：ErrorBoundary 局部兜底 + resetKey 恢复；漏网的：React 19 onUncaughtError/onCaughtError 全局上报。

</details>
