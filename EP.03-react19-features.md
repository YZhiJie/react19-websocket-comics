# EP.03 React 19 上新

> 对应正文：[docs/03-react19-features.md](../docs/03-react19-features.md) · Demo：[src/demos/react19/](../src/demos/react19/)（actions / optimistic / context / metadata / ref / transition）

## 剧情回顾

- 像素酱被 pending props 层层透传折磨：按钮想知道加载态，得一路传下去；
- 码叔揭秘 Actions：`<form action={fn}>` + `useFormStatus` 隔空取状态；
- 标签君实证 `useOptimistic` 乐观上屏与 React 19 自动批处理；
- 结案：Context 本身即 Provider、metadata 内建、ref 回调返回清理函数。

## 知识点拆解

### 第 1 格 · 困境：props 层层传

异步交互的样板代码三件套：手动 `isPending` state、手动 try/catch、手动把 pending 传给深层按钮。中间层组件被无关 props 污染。

### 第 2 格 · 揭秘：Actions 登场

把异步函数交给框架：`<form action={submitAction}>` 配 `useActionState(fn, initial)` 拿 `state/pending`；深层按钮用 `useFormStatus()` 隔空读取所属表单的提交状态，成功后表单自动 reset。聊天室的发送框就是 action 函数（见 [src/demos/chat/composer.tsx](../src/demos/chat/composer.tsx)）。

### 第 3 格 · 实证：乐观与批处理

`useOptimistic(state, reducer)` 在 transition 内先渲染乐观值、结束后自动回滚或转正。React 19 中 setTimeout/WS 回调里的多次 setState 也全部自动批处理。注意：聊天室消息是长生命周期对象，用 reducer 显式建模 `pending/confirmed/failed`（EP.04），短 transition 场景才适合 useOptimistic。

### 第 4 格 · 结案：DX 新姿势

`<Context value={…}>` 直接当 Provider 用；组件里写 `<title>` 自动提升到 head；`ref` 可作为普通 prop 传递，ref 回调可以返回清理函数替代旧的卸载逻辑。

## 坑点清单

- `.map()` 里调 useMemo/useRef 违反 Hooks 规则——Hooks 不能在循环/条件里调用；
- useOptimistic 只适合「提交型」短 transition，别拿来管理会话级状态；
- ref 回调返回的清理函数在卸载与 ref 变更时执行，别忘了 return。

## 自测 3 题（含答案）

<details><summary><b>1. useFormStatus 和 useActionState 的分工是什么？</b></summary>

useActionState 属于「状态拥有方」：包装 action、返回 state/pending；useFormStatus 属于「深层消费者」：在 form 内部组件里读取所属表单的 pending/data，免 props 透传。

</details>

<details><summary><b>2. 为什么聊天室不用 useOptimistic 管理消息列表？</b></summary>

消息是长生命周期会话状态，要经历 pending→confirmed/failed、离线排队、回执替换等复杂转换；useOptimistic 只覆盖「短 transition 乐观显示」，用 reducer 显式建模才可测试、可恢复。

</details>

<details><summary><b>3. React 19 里 ref 回调的新能力是什么？</b></summary>

回调可以返回一个清理函数，React 会在元素卸载或 ref 变更时调用它，取代以前「ref 变更前先调旧回调传 null」的隐式行为，清理逻辑更显式。

</details>
