# JavaScript为什么是单线程语言

## 1. JavaScript的诞生背景
JavaScript 诞生于1995年，最初的设计目的是为网页添加动态交互效果。它的主要应用场景是**操作DOM**（文档对象模型），即动态修改网页内容、样式和结构。

## 2. 操作DOM的复杂性
DOM 是一个复杂的树形结构，多个线程同时操作同一个 DOM 元素会导致：
- **数据竞争**：线程 A 正在读取元素尺寸，线程 B 同时删除该元素，导致程序崩溃。
- **状态不一致**：多个线程同时修改 DOM，最终状态取决于线程调度顺序，难以预测。

## 3. 多线程方案的缺陷

如果采用多线程模型，必须引入**锁机制**（如互斥锁、读写锁）来保证线程安全：
- **增加复杂度**：开发者需要手动管理锁，容易引发死锁、优先级反转等问题。
- **性能损耗**：锁的获取和释放会带来额外的性能开销。
- **难以调试**：多线程并发问题通常难以复现和排查。

## 4. 单线程设计的优势
JavaScript 选择单线程模型，主要原因：
- **避免锁机制**：所有代码按顺序执行，无需考虑线程同步问题。
- **简化编程模型**：开发者只需关注业务逻辑，不必处理复杂的并发问题。
- **与DOM操作天然契合**：单线程保证了对DOM操作的原子性和顺序性。

## 5. 事件循环实现伪多线程
JavaScript 通过**事件循环**（Event Loop）机制实现非阻塞异步编程：
- **单线程处理任务**：主线程只负责执行 JavaScript 代码。
- **异步任务委托**：耗时操作（如网络请求、定时器）交给浏览器/Node.js 的其他线程处理。
- **回调队列**：异步任务完成后，其回调函数被放入任务队列，等待主线程空闲时执行。

这种设计既保留了单线程的简单性，又通过事件循环实现了类似多线程的并发效果，因此也被称为**伪多线程**或**并发模型**。

## 6. Web Worker：真正的多线程补充
浏览器提供了 **Web Worker** API，允许在后台线程中执行 JavaScript 代码：
- **独立线程运行**：Worker 线程在独立的上下文中执行，无法访问主线程的 DOM 对象。
- **计算密集型任务**：适合处理复杂计算、大数据处理等耗时操作，避免阻塞主线程。
- **消息传递通信**：Worker 线程通过 `postMessage` 与主线程进行数据传递，实现异步通信。
- **无法直接更新 UI**：由于 Worker 线程没有 DOM 访问权限，需要将计算结果通过消息传递给主线程，由主线程更新页面。

Web Worker 提供了真正的多线程能力，但受限于 DOM 访问，主要用于计算密集型任务，与主线程的单线程模型形成互补。


# JavaScript事件循环

## 1. 事件循环要解决的根本问题

JavaScript 是单线程语言，这意味着它一次只能做一件事。但如果所有操作都是同步的，那么网络请求、定时器、用户交互等耗时操作都会**阻塞主线程**，导致页面卡死。

事件循环（Event Loop）的解决方案是：**主线程只负责执行代码，将耗时操作委托给浏览器其他线程，等这些操作完成后，再把回调函数送回任务队列排队，等待主线程空闲时执行**。

## 2. 两条不可违背的铁则

事件循环的所有行为都建立在两条规则之上：

| 规则 | 说明 |
|------|------|
| **铁则一：执行栈为空才能取任务** | 事件循环只有检测到执行栈完全清空时，才会从任务队列中取出下一个任务压入执行栈 |
| **铁则二：微任务队列必须一次性清空** | 每执行完一个宏任务，必须把当前微任务队列中的所有任务（包括执行新微任务过程中产生的）全部执行完，才能进行下一步 |

这两个规则是一切异步顺序推演的基础。

## 3. 核心数据结构

### 3.1 执行栈（Call Stack）

#### 结构

执行栈是**后进先出（LIFO）**的栈结构，用于管理函数的执行上下文（Execution Context）。每次调用函数，JS 引擎创建一个新的执行上下文并**压入（push）**栈顶；函数执行完毕后，从栈顶**弹出（pop）**。

一个执行上下文包含三部分：
- **变量环境**（VariableEnvironment）：`let`/`const` 声明的变量
- **词法环境**（LexicalEnvironment）：`var` 声明的变量和函数声明
- **this 绑定**：当前函数 `this` 的指向

#### 工作机制

```javascript
function a() {
  console.log('a')
  b()
  console.log('a end')
}
function b() {
  console.log('b')
}
a()
```

执行栈变化过程：

```
时间 →
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
│  全局    │ │  a()    │ │  a()    │ │  a()    │ │  a()    │ │  a()    │ │  全局    │
│  上下文  │ │  上下文  │ │  上下文  │ │  上下文  │ │  上下文  │ │  上下文  │ │  上下文  │
│         │ │  全局    │ │  全局    │ │  全局    │ │  全局    │ │  全局    │ │         │
│         │ │  上下文  │ │  上下文  │ │  上下文  │ │  上下文  │ │  上下文  │ │         │
└─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘
   ①          ②          ③          ④          ⑤          ⑥          ⑦
```

| 步骤 | 操作 | 栈内容 | 输出 |
|------|------|--------|------|
| ① | 全局上下文入栈 | `[global]` | |
| ② | `a()` 调用，a 的上下文入栈 | `[global, a]` | |
| ③ | `a` 内 `console.log('a')` 执行并弹栈 | `[global, a]` | `a` |
| ④ | `a` 内 `b()` 调用，b 的上下文入栈 | `[global, a, b]` | |
| ⑤ | `b` 内 `console.log('b')` 执行并弹栈 | `[global, a, b]` | `b` |
| ⑥ | `b` 返回，b 的上下文弹栈 | `[global, a]` | |
| ⑦ | `a` 内 `console.log('a end')` 执行，`a` 返回弹栈 | `[global]` | `a end` |

#### 执行栈在事件循环中的角色

执行栈是事件循环的**闸门**。事件循环的核心逻辑可以简化为：

```
while (true) {
  if (执行栈为空 && 有任务在队列中等待) {
    从任务队列取出一个任务，压入执行栈
  }
}
```

这意味着：
- **异步回调要等到同步代码全部执行完后才能执行**——因为执行栈必须清空
- **递归/死循环会永久阻塞事件循环**——因为执行栈永远不会变空
- **执行栈溢出（Stack Overflow）**——递归过深超过栈的最大容量（约 10000 层）

```javascript
// 执行栈永远不会变空，事件循环永远无法取下一个任务
while (true) {}   // 同步代码块，直接卡死
setTimeout(() => console.log('永远不会执行'), 0)
```

### 3.2 任务队列

事件循环使用两个队列管理异步回调：

| 队列 | 入队方式 | 处理时机 |
|------|----------|----------|
| **宏任务队列（Task Queue）** | 由宿主环境（浏览器/Node.js）将回调推入 | 每次事件循环取**一个**执行 |
| **微任务队列（Micro Task Queue）** | 由 JS 引擎将回调推入 | 当前宏任务执行完后，**一次性清空所有** |

## 4. 任务类型详解

### 4.1 宏任务（Macro Task）

#### 定义

由宿主环境发起的异步任务。每次事件循环从宏任务队列中**取出一个**压入执行栈。

#### 常见宏任务

| 来源 | 示例 |
|------|------|
| 首次执行 | `<script>` 标签 — 整体同步代码作为**第一个宏任务** |
| 定时器 | `setTimeout`、`setInterval` |
| I/O 操作 | XHR / Fetch 回调、FileReader 完成回调 |
| DOM 事件 | `click`、`keydown`、`scroll` 等用户交互回调 |
| 浏览器渲染 | `requestAnimationFrame`（部分浏览器归为宏任务） |
| Node.js | `setImmediate`、`I/O 回调` |

#### 宏任务的优先级：任务源（Task Source）

HTML 规范定义多个**任务源**，不同任务源的宏任务进入**独立的队列**。浏览器按优先级从不同队列中取任务：

```
高优先级（浏览器优先调度）
  ├─ 用户交互队列：click、keydown、input、scroll、mousemove
  ├─ 渲染队列：requestAnimationFrame、UI 渲染
  └─ 网络队列：fetch / XHR 回调

低优先级
  └─ 定时器队列：setTimeout、setInterval
```

**为什么有优先级？** 用户交互的响应速度比定时器的触发更重要。如果所有宏任务都在一个队列里，一个耗时的 `setTimeout` 回调会阻塞点击事件的响应。多队列机制保证高优先级任务不会被低优先级任务阻塞。

### 4.2 微任务（Micro Task）

#### 定义

由 JS 引擎自身发起的异步任务。当前宏任务的同步代码执行完后、下一个宏任务开始前，**一次性清空整个微任务队列**。

#### 常见微任务

| 来源 | 示例 | 说明 |
|------|------|------|
| Promise | `.then()`、`.catch()`、`.finally()` | 最常用的微任务来源 |
| async/await | `await` 后面的代码 | 本质是 `Promise.then()` |
| MutationObserver | DOM 变化回调 | 监听 DOM 树变更 |
| `queueMicrotask()` | 显式调用 | 手动创建微任务 |
| Node.js | `process.nextTick` | **优先级高于 Promise.then** |

## 5. 浏览器多线程协作架构

事件循环的主循环在 **JS 引擎线程**运行，但宏任务的来源依赖浏览器其他线程的配合：

```
┌────────────────────────────────────────────────────────────┐
│                    浏览器进程空间                             │
│                                                              │
│  ┌──────────────────────┐  ┌─────────────────────────┐       │
│  │  JS 引擎线程（主线程） │  │  GUI 渲染线程            │       │
│  │  作用：               │  │  作用：                  │       │
│  │  • 执行执行栈中的代码   │  │  • 解析 HTML/CSS         │       │
│  │  • 运行事件循环主循环   │  │  • 计算样式、布局         │       │
│  │  • 管理执行栈入栈出栈   │  │  • 重绘（Paint）         │       │
│  │  • 清空微任务队列      │  │  • 回流（Reflow）         │       │
│  │                      │  │                        │       │
│  │  ⚡ 与渲染线程互斥     │  │  ⚡ 与 JS 引擎线程互斥     │       │
│  └────────┬─────────────┘  └─────────────────────────┘       │
│           │                                                    │
│           任务队列（共享数据结构）                                │
│           │                                                    │
│  ┌────────┴────────────────────────────────────────────┐      │
│  │  宏任务队列（多个优先级子队列）                       │      │
│  │  [交互队列]  [渲染队列]  [网络队列]  [定时器队列]     │      │
│  │  微任务队列                                          │      │
│  └────────┬────────────────────────────────────────────┘      │
│           │                                                    │
│  ┌────────┴─────────┐  ┌──────────┐  ┌───────────────────┐    │
│  │  定时器线程       │  │ 网络线程  │  │ 事件触发线程       │    │
│  │  职责：           │  │ 职责：    │  │ 职责：             │    │
│  │  • 计时 setTimeout│  │ • DNS解析 │  │ • 监听用户交互事件 │    │
│  │  • 到点将回调      │  │ • HTTP请求│  │ • 触发后将回调      │    │
│  │   推入定时器队列   │  │ • 响应后  │  │   推入交互队列     │    │
│  │                   │  │   推入    │  │                    │    │
│  │                   │  │   网络队列│  │                    │    │
│  └───────────────────┘  └──────────┘  └────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

**JS 引擎线程与 GUI 渲染线程互斥**的原因：如果 JS 在修改 DOM 的同时渲染线程在读取 DOM 进行布局，会导致数据竞争。所以浏览器规定：要么 JS 跑，要么 UI 渲染，同一时间只能有一个在工作。

## 6. 单次事件循环的精确流程

```
┌─ 第 1 步：检查执行栈 ─────────────────────────────────────┐
│  事件循环检查调用栈是否为空。                               │
│  → 如果栈不为空，等待（继续执行当前代码）。                  │
│  → 如果栈为空，进入第 2 步。                                │
├─ 第 2 步：取一个宏任务 ─────────────────────────────────────┤
│  从宏任务队列中取优先级最高的队列（交互 > 渲染 > 网络 > 定时器）│
│  的**第一个**任务，压入执行栈。                              │
│  （首次运行：任务就是整个 <script> 标签内的同步代码）         │
├─ 第 3 步：执行该宏任务 ──────────────────────────────────────┤
│  执行栈从上到下执行同步代码：                                │
│  → 遇到函数调用 → 压栈 → 执行 → 弹栈                        │
│  → 遇到 setTimeout/setInterval → 委托给定时器线程计时       │
│  → 遇到 fetch/XHR → 委托给网络线程                          │
│  → 遇到 Promise.then/queueMicrotask → 加入微任务队列        │
│  → 遇到 DOM 事件绑定 → 委托给事件触发线程                    │
│  同步代码全部执行完毕 → 执行栈为空                            │
├─ 第 4 步：清空微任务队列 ───────────────────────────────────┤
│  检查微任务队列：                                            │
│  → 从队列头部取一个微任务，压入执行栈执行                    │
│  → 执行过程中可能产生新的微任务，追加到队列尾部              │
│  → 重复直到微任务队列为空                                    │
├─ 第 5 步：[可选] UI 渲染 ─────────────────────────────────────┤
│  浏览器判断是否需要渲染（由帧率和 DOM 变更决定）：            │
│  → 需要渲染 → 暂停 JS 引擎线程，启用 GUI 渲染线程            │
│  → 不需要渲染 → 跳过                                         │
├─ 第 6 步：回到第 1 步 ──────────────────────────────────────┤
│  循环。                                                      │
└───────────────────────────────────────────────────────────────┘
```

**口诀**：一个宏任务 → 清空微任务 → 可选渲染 → 下一个宏任务

## 7. 若干示例的推演变式

以下示例逐步增加复杂度，展示如何用两条铁则推导出执行结果。

### 示例 1：基础

```javascript
console.log('1')
setTimeout(() => console.log('2'), 0)
console.log('3')

// 推导：
// ① 第一个宏任务（script）同步代码执行：
//    输出 '1'，遇到 setTimeout → 交给定时器线程，0ms 后回调入定时器队列
//    输出 '3'
// ② 同步代码执行完，微任务队列为空，宏任务队列有 [setTimeout 回调]
// ③ 取 setTimeout 回调入栈执行，输出 '2'
// 输出：1 → 3 → 2
```

### 示例 2：微任务

```javascript
console.log('1')
setTimeout(() => {
  console.log('2')
  Promise.resolve().then(() => console.log('3'))
}, 0)
Promise.resolve().then(() => console.log('4'))
console.log('5')

// 推导：
// ① 第一个宏任务（script）同步代码：
//    输出 '1'，setTimeout 交给定时器线程，Promise.then('4') 入微任务队列，输出 '5'
// ② 清空微任务队列 → 输出 '4'
// ③ 取下一个宏任务（setTimeout 回调）：
//    输出 '2'，Promise.then('3') 入微任务队列
// ④ 清空微任务队列 → 输出 '3'
// 输出：1 → 5 → 4 → 2 → 3
```

### 示例 3：微任务嵌套

```javascript
console.log('start')
Promise.resolve().then(() => {
  console.log('微1')
  Promise.resolve().then(() => {
    console.log('微2')
    Promise.resolve().then(() => console.log('微3'))
  })
})
setTimeout(() => console.log('宏'), 0)
console.log('end')

// 推导：
// ① 第一个宏任务同步代码：输出 'start'，Promise.then('微1') 入微任务队列，
//    setTimeout 给定时器线程，输出 'end'
// ② 清空微任务队列：
//    取 '微1' → 输出 '微1'，其内部 Promise.then('微2') 追加到队列尾部
//    取 '微2' → 输出 '微2'，其内部 Promise.then('微3') 追加到队列尾部
//    取 '微3' → 输出 '微3'
// ③ 取下一个宏任务：setTimeout 回调 → 输出 '宏'
// 输出：start → end → 微1 → 微2 → 微3 → 宏
```

### 示例 4：宏任务嵌套

```javascript
setTimeout(() => {
  console.log('宏1')
  setTimeout(() => console.log('宏2'), 0)
  Promise.resolve().then(() => console.log('微'))
}, 0)
setTimeout(() => console.log('宏3'), 0)

// 推导：
// ① 第一个宏任务（script）同步代码：
//    两个 setTimeout 都交给定时器线程，0ms 后依次入队列 [宏1回调, 宏3回调]
// ② 取第一个宏任务（宏1回调）：
//    输出 '宏1'，新 setTimeout 给定时器线程 → 宏2 入队列尾部 [宏3, 宏2]
//    Promise.then('微') 入微任务队列
// ③ 清空微任务队列 → 输出 '微'
// ④ 取下一个宏任务（宏3回调）→ 输出 '宏3'
// ⑤ 取下一个宏任务（宏2回调）→ 输出 '宏2'
// 输出：宏1 → 微 → 宏3 → 宏2
```
# JavaScript事件

## 1. 事件模型（DOM 事件标准）

DOM 事件模型经历了三个阶段：

### DOM0 级事件

直接给元素事件属性赋值，同一事件只能绑定**一个**处理函数，后绑定覆盖前者。

```javascript
btn.onclick = function() { console.log('clicked'); };
btn.onclick = null; // 解绑
```

### DOM2 级事件

通过 `addEventListener` / `removeEventListener` 绑定与解绑，同一事件可绑定**多个**处理函数，按注册顺序执行。支持控制触发阶段。

```javascript
btn.addEventListener('click', handler, useCapture);
btn.removeEventListener('click', handler, useCapture);
```

- 第三个参数 `useCapture`：`false`（**默认**）冒泡阶段触发，`true` 捕获阶段触发
- 也可传入配置对象：`{ once: true, capture: true, passive: true }`

### IE 事件模型（已淘汰）

`attachEvent` / `detachEvent`，仅有冒泡阶段，事件名需加 `on` 前缀。IE11+ 已全面支持 DOM2 标准。

---

## 2. 事件流（事件传递方式）

事件传播分三个阶段，从 `window` 出发，最终回到 `window`：

```
       捕获阶段 ↓                冒泡阶段 ↑
 window → document → html → body → ... → 目标元素 → body → html → document → window
                    1. 捕获       2. 目标阶段        3. 冒泡
```

1. **捕获阶段**：事件从 `window` 向下经层层节点传递到目标元素的父节点
2. **目标阶段**：事件到达目标元素，触发该元素上所有监听器（不考虑 capture 顺序）
3. **冒泡阶段**：事件从目标元素向上原路冒泡回 `window`

---

## 3. 特殊事件类型（不冒泡）

以下事件**不冒泡**，仅在目标元素自身触发，无法用于事件委托：

| 类型 | 事件 | 说明 |
|------|------|------|
| 焦点事件 | `focus`、`blur` | 聚焦/失焦，不冒泡。对应的冒泡版本：`focusin`、`focusout` |
| 页面事件 | `load`、`unload`、`scroll`(部分) | 资源加载完成、页面卸载 |
| 媒体事件 | `play`、`pause`、`ended` | 音视频播放状态 |
| 鼠标事件 | `mouseenter`、`mouseleave` | 进入/离开元素（不冒泡），对应冒泡版：`mouseover`、`mouseout` |
| 表单事件 | `submit`、`reset` | 表单提交/重置时在 `<form>` 上触发（不冒泡到 document 以上） |
| 输入事件 | `input`、`change` | 少数场景下不冒泡 |

### 常用事件类型速查

| 分类 | 事件名 | 说明 |
|------|--------|------|
| 鼠标 | `click` | 单击（左键） |
| | `dblclick` | 双击 |
| | `mousedown` / `mouseup` | 鼠标按下 / 松开 |
| | `mousemove` | 鼠标在元素上移动 |
| | `mouseover` / `mouseout` | 鼠标进入 / 离开（冒泡） |
| | `mouseenter` / `mouseleave` | 鼠标进入 / 离开（不冒泡） |
| | `contextmenu` | 右键菜单 |
| | `wheel` | 鼠标滚轮 |
| 键盘 | `keydown` / `keyup` | 按键按下 / 松开 |
| | `keypress` | 字符键按下（已废弃，建议用 keydown） |
| 焦点 | `focus` / `blur` | 获得 / 失去焦点（不冒泡） |
| | `focusin` / `focusout` | 获得 / 失去焦点（冒泡） |
| 表单 | `submit` | 表单提交 |
| | `reset` | 表单重置 |
| | `input` | 输入框值变化（实时） |
| | `change` | 输入框值变化（失焦时触发） |
| | `select` | 选中文本 |
| 页面 | `DOMContentLoaded` | DOM 树构建完成（比 load 早） |
| | `load` | 页面及所有资源加载完成 |
| | `resize` | 窗口大小变化 |
| | `scroll` | 滚动（多数冒泡，document 上不冒泡） |
| | `beforeunload` | 页面即将卸载（可弹确认框） |
| 拖拽 | `dragstart` / `dragend` | 拖拽开始 / 结束 |
| | `dragover` | 拖拽悬停在目标上 |
| | `drop` | 释放拖拽元素 |
| 触摸 | `touchstart` / `touchmove` / `touchend` | 移动端触摸 |
| 剪贴板 | `copy` / `cut` / `paste` | 复制 / 剪切 / 粘贴 |

---

## 4. 阻止事件传播与默认行为

- `event.stopPropagation()` — 阻止捕获/冒泡继续传播，当前元素上**其他**同类型监听器仍会触发
- `event.stopImmediatePropagation()` — 阻止传播，同时阻止当前元素上**未执行**的其他同类型监听器
- `event.preventDefault()` — 阻止浏览器默认行为（如 `<a>` 跳转、表单提交），**不阻止事件传播**

---

## 5. 事件委托（Event Delegation）

利用事件冒泡，将子元素的事件统一绑定到祖先元素上，通过 `event.target` 判断实际触发源。

### 获取目标 DOM 元素的方式

#### 1. tagName 判断（简单匹配标签名）

```javascript
ul.addEventListener('click', function(e) {
  if (e.target.tagName === 'LI') {
    console.log('点击了:', e.target.textContent);
  }
});
```

#### 2. matches() 判断（支持 CSS 选择器）

```javascript
ul.addEventListener('click', function(e) {
  if (e.target.matches('li.active')) {
    // 精确匹配 li 且有 .active 类
  }
});
```

#### 3. closest() 向上查找（推荐，处理嵌套子元素）

```html
<li><span>点我</span></li>
```

当点击 `<span>` 时，`e.target` 是 `span` 而非 `li`。使用 `closest` 向上查找到最近匹配的祖先：

```javascript
ul.addEventListener('click', function(e) {
  const target = e.target.closest('li');
  if (!target) return;          // 点击的不是 li 或 li 内部，直接忽略
  console.log('目标元素:', target);
});
```

对比总结：

| 方法 | 适用场景 | 是否处理嵌套子元素 |
|------|---------|------------------|
| `e.target.tagName` | 简单结构，无嵌套 | ❌ 不处理 |
| `e.target.matches(selector)` | CSS 选择器精确匹配 | ❌ 不处理 |
| `e.target.closest(selector)` | 通用场景，有嵌套 | ✅ 自动向上查找 |

### 事件委托的优势

- 减少监听器数量，提升性能
- 动态添加的子元素自动生效，无需重新绑定

---

## 6. Vue 与 React 中绑定多个事件

### Vue

```html
<!-- 不同事件直接并列 -->
<button @click="onClick" @dblclick="onDblclick">按钮</button>

<!-- 同一事件绑定多个回调 -->
<button @click="onClick1(); onClick2()">按钮</button>
```

### React

```jsx
// 不同事件
<button onClick={onClick} onDoubleClick={onDblclick}>按钮</button>

// 同一事件绑定多个回调
<button onClick={() => { onClick1(); onClick2(); }}>按钮</button>
```

本质都是在一个函数体内串行调用，事件机制本身不限制回调数量。
# Fetch 与 Ajax

## 1. 设计哲学与演进

Ajax 和 Fetch 的表面区别是 API 形式（回调 vs Promise），但深层差异在于**谁主导请求生命周期**。

### Ajax（XMLHttpRequest）

设计于 2000 年代初，当时 Web 还没有 Promise 标准。XHR 采用了**事件驱动模型**——浏览器定义了 XHR 的底层机制（请求、就绪、完成等状态），开发者通过注册回调函数来"响应"这些事件。

**核心模型**：一个 XHR 实例就是一个**状态机**。通过 `readyState` 追踪 5 个状态（0: 未初始化 → 1: 已建立连接 → 2: 已收到响应头 → 3: 下载中 → 4: 完成），每次状态变更触发 `readystatechange` 事件。开发者可以监听不同状态做不同处理。

```javascript
const xhr = new XMLHttpRequest()
console.log(xhr.readyState) // 0 — UNSENT

xhr.open('GET', '/api/data')  // 状态变为 1 — OPENED
xhr.onreadystatechange = function() {
  if (xhr.readyState === 4) {           // 请求完成
    if (xhr.status === 200) {           // HTTP 状态码
      console.log(xhr.responseText)     // 响应体文本
    }
  }
}
xhr.send()  // 状态变为 2 — HEADERS_RECEIVED → 3 — LOADING → 4 — DONE
```

### Fetch

设计于 2015 年左右，ES6 已将 Promise 纳入标准。Fetch 采用 **Promise + Stream** 模型——请求是异步操作，响应是**可流式读取的数据源**。它将一次 HTTP 请求拆解为两个独立的异步阶段：

1. **请求已发出**：`fetch()` 返回的 Promise 在收到**响应头**时 resolve（即使状态码是 404/500）
2. **响应体读取中**：通过 `response.body`（`ReadableStream`）流式读取数据，或通过 `.json()`/`.text()` 等便捷方法等待完整数据

```javascript
const response = await fetch('/api/data')
// ↑ 此时仅收到响应头，响应体还在传输中

const data = await response.json()
// ↑ 等待整个响应体下载并解析完成
```

这也解释了为什么 Fetch 默认不把 HTTP 错误码视为异常——服务端确实返回了响应（有状态码、有头信息），只是业务层面不成功而已。

## 2. API 语法差异对比

### 请求生命周期对比

| 阶段 | Ajax | Fetch |
|------|------|-------|
| 初始化实例 | `new XMLHttpRequest()` | `fetch(url, options)` — 函数调用即创建 |
| 配置请求 | `xhr.open(method, url)` + `xhr.setRequestHeader()` | `options` 对象统一传入 |
| 发送请求 | `xhr.send(body)` | `fetch()` 调用时自动发送 |
| 监听响应头 | `readystatechange` + 判断 `readyState >= 2` | `response` 对象 resolve 时已包含头 |
| 监听进度 | `onprogress` 事件 | `response.body` 的 `ReadableStream` |
| 读取响应体 | `xhr.responseText` / `xhr.response` | `response.text()` / `response.json()`（均返回 Promise） |

### 完整请求示例对比

**Ajax — 分步配置，事件驱动：**

```javascript
const xhr = new XMLHttpRequest()

// 1. 配置请求
xhr.open('POST', '/api/users')
xhr.setRequestHeader('Content-Type', 'application/json')
xhr.setRequestHeader('X-CSRF-Token', 'abc123')
xhr.responseType = 'json'          // 自动解析 JSON

// 2. 监听事件
xhr.onload = function() {
  if (xhr.status >= 200 && xhr.status < 300) {
    console.log('成功:', xhr.response)
  } else {
    console.error('HTTP 错误:', xhr.status)
  }
}
xhr.onerror = function() {
  console.error('网络错误')
}
xhr.ontimeout = function() {
  console.error('请求超时')
}

// 3. 发送
xhr.timeout = 5000
xhr.send(JSON.stringify({ name: 'Alice' }))
```

**Fetch — 统一配置，链式调用：**

```javascript
try {
  const response = await fetch('/api/users', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-CSRF-Token': 'abc123'
    },
    body: JSON.stringify({ name: 'Alice' }),
    signal: AbortSignal.timeout(5000)   // 超时
  })

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`)
  }

  const data = await response.json()
  console.log('成功:', data)
} catch (err) {
  if (err.name === 'AbortError') {
    console.error('请求超时或取消')
  } else {
    console.error('请求失败:', err)
  }
}
```

### 关键差异总结

| 方面 | Ajax | Fetch |
|------|------|-------|
| 语法风格 | 命令式（一步步告诉浏览器做什么） | 声明式（描述「要什么」，返回 Promise） |
| 配置方式 | 分散在多个属性/方法调用中 | 集中在 `options` 对象，一次传入 |
| 错误模型 | 网络错误和 HTTP 错误都进 `onerror` | HTTP 错误不 reject，需检查 `response.ok` |
| 响应解析 | `responseType` 属性控制自动解析 | 手动调用 `.json()/.text()` 等，返回 Promise |
| 超时控制 | `xhr.timeout` + `ontimeout` 事件 | `AbortSignal.timeout()` + `AbortError` |
| 取消请求 | `xhr.abort()` | `AbortController.abort()` |
| 上传进度 | `xhr.upload.onprogress` | 不支持（需用 XHR） |
| Cookie 发送 | 同域请求默认携带 | **默认不带**，需 `credentials: 'include'` |
| 请求生命周期管理 | 手动管理 `readyState` 状态机 | Promise 自动管理 |

## 3. 错误处理深度对比

两者的错误处理差异源于底层设计：

**Ajax 错误处理：**
- `onerror` 触发条件：**网络层失败**——DNS 解析失败、连接被拒绝、连接超时（非 `timeout` 属性设置的那种）
- HTTP 4xx/5xx 不会触发 `onerror`，而是触发 `onload`，需要通过 `xhr.status` 手动判断
- `ontimeout` 专门处理请求超时（由 `xhr.timeout` 属性设定）

```javascript
xhr.onerror = () => console.log('触发条件：断网、DNS 解析失败、连接拒绝')
xhr.onload = () => {
  // 无论 200 还是 500 都进这里
  if (xhr.status >= 400) {
    console.log('这是 HTTP 错误，不是网络错误')
  }
}
```

**Fetch 错误处理：**
- `fetch()` 返回的 Promise 仅在网络层面失败时 reject（断网、DNS 解析失败、连接拒绝）
- HTTP 4xx/5xx 视为"请求成功但响应不完美"，Promise 正常 resolve，`response.ok` 为 `false`
- 取消/超时通过 `AbortController` 触发 `AbortError`

```javascript
fetch('/api/data').then(response => {
  // 状态码 404，response.ok === false，但这里正常执行
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`)
  }
  return response.json()
}).catch(err => {
  // 这里可能因为：断网、response.json() 解析失败、手动 throw 的 HTTP 错误
  console.error(err)
})
```

**为什么 Fetch 这样设计？** 因为 Fetch 基于 Promise 链，统一的 `.catch()` 应该仅捕获**异常情况**。HTTP 4xx/5xx 是**预期内的业务状态**，应该由开发者自己在 `.then()` 中处理。如果 Fetch 把 404 也 reject，开发者就无法区分“请求没发出去”和“请求发了但服务端返回 404”这两种场景。

## 4. 请求/响应拦截器实现对比

### Ajax 的拦截器原理（以 Axios 为例）

Axios 本质上是一个 XHR 的 wrapper，拦截器通过**函数管道（middleware pipeline）**实现——请求拦截器数组和响应拦截器数组，每个拦截器依次传递 config 或 response：

```javascript
// Axios 内部简化逻辑
const requestInterceptors = []
const responseInterceptors = []

// 请求发出前执行拦截器链
let config = { url, method, headers, data }
for (const interceptor of requestInterceptors) {
  config = interceptor(config) // 可以修改 headers、添加 token 等
}

// 执行 XHR 请求
xhr.send(config.data)

// 响应到达后执行拦截器链
let response = { data, status, headers }
for (const interceptor of responseInterceptors) {
  response = interceptor(response) // 可以统一解析、统一错误处理
}
```

### Fetch 实现拦截器的方式

Fetch 原生没有拦截器，因为它是**浏览器原生 API**，不是库。实现拦截器有几种方案：

**方案 1：包装 `fetch` 函数（推荐——不侵入全局）**

```javascript
function createFetchWithInterceptors(baseUrl) {
  const requestInterceptors = []
  const responseInterceptors = []

  async function customFetch(url, options = {}) {
    let config = { ...options, url: baseUrl + url }

    // 请求拦截器链
    for (const interceptor of requestInterceptors) {
      config = await interceptor(config)
    }

    let response = await fetch(config.url, config)

    // 响应拦截器链
    for (const interceptor of responseInterceptors) {
      response = await interceptor(response)
    }

    return response
  }

  customFetch.request = (fn) => requestInterceptors.push(fn)
  customFetch.response = (fn) => responseInterceptors.push(fn)

  return customFetch
}

// 使用
const api = createFetchWithInterceptors('https://api.example.com')

api.request((config) => {
  config.headers = { ...config.headers, Authorization: 'Bearer ' + getToken() }
  return config
})

api.response(async (res) => {
  if (!res.ok) {
    const err = await res.json()
    throw new Error(err.message)
  }
  return res.json()
})

// 调用
const data = await api('/users')
```

**方案 2：`Proxy` 代理全局 fetch（激进——适合监控场景）**

```javascript
const originalFetch = window.fetch
window.fetch = new Proxy(originalFetch, {
  apply(target, thisArg, args) {
    console.log('[Fetch 监控]', args[0], args[1]?.method || 'GET')
    const start = performance.now()
    const result = Reflect.apply(target, thisArg, args)
    result.then(() => {
      console.log('[Fetch 监控] 耗时:', performance.now() - start)
    })
    return result
  }
})
```

## 5. 现代特性深度解析

### 5.1 流式响应（Response Streaming）

Fetch 原生支持流式读取，这是 XHR 做不到的。服务端可以一边生成数据一边发送，前端可以逐块处理，不需要等完整响应到达。

```javascript
const response = await fetch('/api/large-stream')

// response.body 是一个 ReadableStream
const reader = response.body.getReader()
const decoder = new TextDecoder()

while (true) {
  const { done, value } = await reader.read()
  if (done) break

  const chunk = decoder.decode(value, { stream: true })
  console.log('收到一块数据:', chunk)
  // 可以逐块渲染 UI，实现"打字机"效果
}
```

**适用场景：** AI 对话流式输出、大文件下载进度、实时日志推送。

### 5.2 请求取消（AbortController）

两者都能取消请求，但机制不同：

```javascript
// XHR 取消
const xhr = new XMLHttpRequest()
xhr.open('GET', '/api/data')
xhr.send()
xhr.abort() // 直接调用，简单粗暴

// Fetch 取消 — 同一 AbortController 可控制多个请求
const controller = new AbortController()
const signal = controller.signal

fetch('/api/user', { signal })
fetch('/api/posts', { signal })

// 5 秒后同时取消两个请求
setTimeout(() => controller.abort(), 5000)
```

**XHR 的 `abort()`** 是实例方法，一次只能取消一个请求。**Fetch 的 `AbortController`** 是信号量模式，一个信号可以关联任意多个请求，更适合竞态场景：

```javascript
// 竞态场景：用户反复切换 Tab，取消前一个未完成的请求
let currentController

function searchUsers(keyword) {
  currentController?.abort() // 取消上一次请求

  currentController = new AbortController()
  fetch(`/api/users?q=${keyword}`, {
    signal: currentController.signal
  }).then(res => res.json())
    .then(data => renderUsers(data))
    .catch(err => {
      if (err.name !== 'AbortError') {
        showError(err)
      }
    })
}
```

### 5.3 默认行为差异

| 行为 | Ajax | Fetch |
|------|------|-------|
| 跨域 Cookie | 同域请求默认携带 | **默认不携带**，需 `credentials: 'include'` |
| 跨域请求 | 支持 CORS | 同左，但 Fetch 对 CORS 限制更严格 |
| 重定向 | 默认跟随，可通过 `xhr.redirect` 控制 | 默认跟随，可通过 `redirect: 'manual'` 控制 |
| 缓存控制 | XHR 走 HTTP 缓存（除非加 header） | Fetch 在 `no-cors` 模式下会限制某些 header |

### 5.4 兼容性与 Polyfill

Ajax 兼容 IE5+，这是历史原因。Fetch 的主流兼容性如下：

- Chrome 42+, Firefox 39+, Safari 10+, Edge 14+
- **IE 全系不支持**（但 IE 已退役）
- 需要兼容低版本浏览器时，可用 `whatwg-fetch` polyfill（原理是用 XHR 实现 Fetch API）

```bash
npm install whatwg-fetch
```

```javascript
import 'whatwg-fetch'
// 不支持的浏览器会自动使用 XHR 模拟的 fetch
```

## 6. 与传统请求对比（为什么 Ajax/Fetch 能无刷新更新数据）

### 流程对比

**传统请求**（表单提交、`<a>` 跳转）：
1. 浏览器**卸载当前页面**，清空所有 JS/DOM 状态
2. 向服务端发送 HTTP 请求
3. 服务端返回**完整 HTML 文档**
4. 浏览器**重新解析、渲染整个页面**，页面闪烁

**Ajax / Fetch**：
1. 请求由浏览器**网络线程**异步发出，不阻塞 JS 主线程
2. 主线程继续执行其他 JS 代码，页面保持正常运行
3. 服务端返回**纯数据（JSON/XML）或 HTML 片段**
4. 回调进入任务队列，主线程空闲后执行回调
5. 通过 DOM API **局部修改**需要更新的节点，浏览器仅对差异部分做增量渲染

```
传统请求：
  点击提交 → 页面卸载 → 请求 → 服务端返回完整 HTML → 浏览器整页重渲染（闪烁）

Ajax/Fetch：
  JS 调用 → 网络线程异步发请求（不阻塞主线程）
         → 主线程继续运行，页面正常交互
         → 响应回调执行 → DOM 局部更新 → 仅增量渲染差异部分（无闪烁）
```

### 为什么能做到无刷新？

核心原因有两个：

1. **控制权在前端**：传统请求的渲染流程由浏览器导航机制驱动，开发者无法干预。Ajax/Fetch 把请求控制权交给了 JS，拿到数据后开发者可以**选择**更新哪些 DOM 节点，而不是整个页面替换。

2. **异步非阻塞**：Ajax/Fetch 的请求在**浏览器网络线程**中执行，不阻塞 JS 主线程。用户仍然可以点击按钮、滚动页面、输入文字——页面始终是"活的"。而传统导航会冻结页面直到新页面加载完成。

### 对比表格

| | 传统请求 | Ajax / Fetch |
|---|---|---|
| 页面是否卸载 | 是，浏览器卸载当前页面 | 否，页面始终存在 |
| 请求谁发出 | 浏览器导航行为（地址栏/表单） | JS 通过网络线程异步发出 |
| 阻塞主线程？ | 是，页面冻结直到新页面加载完成 | 否，异步回调，网络线程单独处理 |
| 响应是什么 | 完整 HTML 文档 | 纯数据或 HTML 片段 |
| 谁控制更新 | 浏览器整页替换 | 前端 JS 通过 DOM API 局部修改 |
| 用户体验 | 闪烁、中断操作、需重新加载全部资源 | 流畅、无中断、持续交互 |
| 资源开销 | 重下载 CSS/JS/图片等全部资源 | 仅传输必要数据，前端按需更新 |
| 服务端职责 | 拼接完整的 HTML 页面 | 仅提供数据接口或 HTML 片段 |
| 前后端耦合 | 强耦合，视图和逻辑混在服务端 | 松耦合，前后端通过接口约定通信 |


# 闭包

## 1. 闭包的定义

闭包是指一个函数**记住了它定义时所在作用域的变量**，即使那个作用域已经执行完毕。简单说：**闭包 = 函数 + 它记住的外部变量**。

```javascript
function outer() {
  let count = 0
  return function inner() {
    count++
    return count
  }
}
const counter = outer()
console.log(counter()) // 1
console.log(counter()) // 2
// outer 执行完了，但 inner 仍然持有 count 的引用
```

## 2. 闭包的应用场景

### 防抖（Debounce）

```javascript
function debounce(fn, delay) {
  let timer = null
  return function (...args) {
    clearTimeout(timer)
    timer = setTimeout(() => fn.apply(this, args), delay)
  }
}
```

### 节流（Throttle）

```javascript
function throttle(fn, interval) {
  let last = 0
  return function (...args) {
    const now = Date.now()
    if (now - last >= interval) {
      last = now
      fn.apply(this, args)
    }
  }
}
```

### 私有变量

```javascript
function createCounter() {
  let count = 0
  return {
    increment: () => ++count,
    decrement: () => --count,
    get: () => count
  }
}
```

### 函数柯里化

```javascript
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) return fn(...args)
    return (...next) => curried(...args, ...next)
  }
}
```

## 3. 闭包与内存泄漏

### 如何泄漏

闭包本身不泄漏，泄漏的是**被闭包持有且永远不会被释放的引用**。常见场景：

```javascript
// 泄漏场景：全局变量持有闭包，闭包持有大对象
let leak = null
function setup() {
  const bigData = new Array(1000000)
  leak = function() {
    console.log(bigData.length)
  }
}
```

### 如何定位

**Chrome DevTools Memory 面板：**
1. 录制堆快照（Heap Snapshot），搜索 `(closure)` 查看闭包详情
2. 对比两个快照（操作前后），看哪些对象未被回收
3. Performance 面板录制，勾选 Memory，观察内存曲线是否持续上升不回落

**关键排查对象：**
- 全局变量引用的闭包
- 未移除的事件监听器（`Elements` → `Event Listeners`）
- 未清理的定时器
- 闭包中引用的已移除 DOM 元素

### 如何防范与解决

| 做法 | 说明 |
|------|------|
| 避免将闭包赋值给**全局变量** | 全局引用不会被 GC 回收 |
| 及时清理事件监听 | `removeEventListener` 移除不再需要的监听 |
| 及时清理定时器 | `clearInterval` / `clearTimeout` |
| 不再需要时**置 null** | 切断闭包中的引用链 |
| 用 `WeakMap/WeakSet` 作缓存 | 键为弱引用，不影响 GC |
| 组件卸载时清理副作用 | React `useEffect` 返回清理函数 |

# 原型与原型链

## 1. 原型（Prototype）
每个对象都有一个**内部(**指针) `[[Prototype]]`（可通过 `__proto__` 或 `Object.getPrototypeOf()` 访问），它是一个**引用**，指向另一个对象。对于通过构造函数创建的实例，这个引用指向构造函数的 `prototype` 属性所引用的对象。

构造函数的 `prototype` 属性本身是一个对象，包含可被所有实例共享的属性和方法。

## 2. 原型链（Prototype Chain）
当访问对象属性时，如果对象自身没有，就沿着内部引用 `[[Prototype]]` 向上查找，直到找到属性或到达原型链末端（通常是 `Object.prototype`，其 `[[Prototype]]` 为 `null`）。这个查找路径就是原型链。

## 3. 特点

JavaScript 对象是通过引用来传递的，我们创建的每个新对象实体中并没有一份属于自己的原型副本。当我们修改原型时，与之相关的对象也会继承这一改变。

# new 操作符内部工作步骤

## 1. 创建空对象

创建一个新的空对象（`{}`）。

## 2. 设置原型
将新对象的内部 `[[Prototype]]` 引用指向构造函数的 `prototype` 对象。

## 3. 执行构造函数
以新对象作为 `this` 上下文执行构造函数，为新对象添加属性和方法。

## 4. 返回对象
如果构造函数显式返回一个对象，则使用该对象作为新实例；否则返回步骤1创建的新对象。

```javascript
// 实现一个 myNew 函数，模仿 JavaScript 中的 new 操作符的行为。
// this 在js中 是动态绑定的，但是new操作符创建出来的对象会强制this指向本身，于是我们需要手动改变this执行
// 所以我们 只能在 bind apply call 这三个方法中选择一个来改变this指向
// 1. bind方法会返回一个新的函数，而不是直接调用函数，所以不适合我们在这里使用
// 2. apply方法会接受一个参数数组，并作为参数传递给调用函数
function myNew(constructorFn, ...args) {
  // 创建一个新对象，并将其原型指向构造函数的 prototype 属性
  const newObj = {};
  // Object.setPrototypeOf() 方法设置一个指定的原型对象和属性到一个指定的对象上，返回修改后的对象。
  Object.setPrototypeOf(newObj, constructorFn);
  // 使用 apply 方法调用构造函数，并将新对象作为 this 传入，同时传递剩余的参数
  const res = constructorFn.apply(newObj, args);
  // 如果构造函数返回一个对象，则使用该对象作为结果；否则，使用新创建的对象作为结果
  return res instanceof Object ? res : newObj;
}
```

# js判断数据类型的四种方式

## 1. typeof

`typeof` 操作符返回一个表示数据类型的字符串。其底层实现是通过判断变量在内存中的**类型标签（Type Tag）**来区分基本类型。JavaScript 引擎内部使用低位二进制位存储类型信息：

- `000`：对象（Object）
- `1`：整数（Int）
- `010`：浮点数（Double）
- `100`：字符串（String）
- `110`：布尔值（Boolean）
- `-2^30`：undefined（即全 0，但特殊标记）

```javascript
typeof 42          // "number"
typeof 'hello'     // "string"
typeof true        // "boolean"
typeof undefined   // "undefined"
typeof Symbol()    // "symbol"
typeof 10n         // "bigint"
typeof function(){}  // "function"
```

**特殊情况与局限：**

- **无法精准判断引用类型**：`typeof` 对数组、对象、正则、Date 等引用类型统一返回 `"object"`，无法进一步区分。
- **`typeof null` 返回 `"object"`**：这是一个历史遗留 Bug。在 JS 底层存储中，`null` 的机器码全为 0，而对象类型标签也是 `000`，导致 `typeof null` 错误地返回 `"object"`。

```javascript
typeof null           // "object"  ← 历史遗留Bug
typeof []             // "object"
typeof {}             // "object"
typeof /regex/        // "object"
typeof new Date()     // "object"
```

## 2. instanceof

`instanceof` 操作符通过**原型链查找**来判断数据类型，检测左边对象的原型链上是否存在右边构造函数的 `prototype`。主要用于判断引用类型数据。

```javascript
[] instanceof Array           // true
{} instanceof Object          // true
new Date() instanceof Date    // true
function(){} instanceof Function  // true

// 原型链上的构造函数都会返回 true
[] instanceof Object          // true（Array 的原型链上有 Object.prototype）
```

**局限：**

- 不能判断基本类型（如 `123 instanceof Number` 返回 `false`）。
- 跨 iframe / 跨窗口环境下，不同执行上下文有独立的构造函数，`instanceof` 会失效。

## 3. Array.isArray

`Array.isArray()` 是 `Array` 构造函数的**静态方法**，专门用于判断一个值是否为数组，返回布尔值。弥补了 `typeof` 和 `instanceof` 在判断数组时的不足。

```javascript
Array.isArray([])                // true
Array.isArray(new Array())       // true
Array.isArray({})                // false
Array.isArray('hello')           // false
Array.isArray(arguments)         // false（类数组但不是数组）

// 跨 iframe 环境下依然可靠（不依赖原型链）
```

## 4. Object.prototype.toString.call

这是判断数据类型**最准确**的方法。通过调用 `Object.prototype` 上的 `toString` 方法，返回 `[object Xxx]` 格式的字符串，其中 `Xxx` 为内部 `[[Class]]` 属性的值。

```javascript
Object.prototype.toString.call(42)           // "[object Number]"
Object.prototype.toString.call('hello')      // "[object String]"
Object.prototype.toString.call(true)         // "[object Boolean]"
Object.prototype.toString.call(undefined)    // "[object Undefined]"
Object.prototype.toString.call(null)         // "[object Null]"
Object.prototype.toString.call([])           // "[object Array]"
Object.prototype.toString.call({})           // "[object Object]"
Object.prototype.toString.call(function(){}) // "[object Function]"
Object.prototype.toString.call(new Date())   // "[object Date]"
Object.prototype.toString.call(/regex/)      // "[object RegExp]"
Object.prototype.toString.call(new Map())    // "[object Map]"
Object.prototype.toString.call(new Set())    // "[object Set]"
Object.prototype.toString.call(Symbol())     // "[object Symbol]"
```

**为什么必须用 `call`？** 因为数组、字符串等类型重写了自身的 `toString` 方法，直接调用会返回不同的结果（如数组的 `toString()` 返回以逗号拼接的元素字符串）。必须借用 `Object.prototype` 原生的 `toString` 才能拿到内部类型标记。

## 总结

| 方法 | 判断范围 | 优缺点 |
|------|---------|--------|
| `typeof` | 基本类型 + function | 快速简单，无法区分引用类型，`null` 误判 |
| `instanceof` | 引用类型 | 基于原型链，无法判断基本类型，跨 iframe 失效 |
| `Array.isArray` | 仅数组 | 数组判断专用，跨 iframe 可靠 |
| `Object.prototype.toString.call` | 所有类型 | 最全面准确，写法稍长 |

# 路由加载
在现代 SPA 应用中，前端路由系统通过 **hash** 或 **history** 模式实现页面切换，两者核心区别在于是否会导致浏览器重新请求 HTML 文件。

## Hash 模式
- **原理**：基于 URL 的 `#` 部分（片段标识符），通过监听 `hashchange` 事件更新视图。
- **不会触发请求**：浏览器不会将 `#` 后面的内容发送到服务器，因此只更新哈希值不会导致浏览器重新发起请求获取 HTML 文件。
- **刷新页面时的行为**：刷新页面时，浏览器会向服务器请求 URL 中 `#` 之前的部分（即**基础路径**），服务器返回入口 HTML 文件（如 index.html），然后前端 JavaScript 根据哈希值初始化路由。
- **实现方式**：JavaScript 监听哈希变化，动态匹配路由组件并更新 DOM。

## History 模式

- **原理**：利用 HTML5 History API（`pushState`/`replaceState`）修改 URL 路径，监听 `popstate` 事件。
- **刷新页面**：浏览器会将**完整 URL** 发送到服务器，如果服务器没有配置将所有路由请求重定向到入口 HTML 文件，则会返回 404。
- **服务器要求**：需要服务器配置，将所有路由请求都重定向到 `index.html`，由前端 JavaScript 处理路由。

## 路由懒加载策略
为优化性能，常采用以下加载策略：
- **首页核心组件和相关资源**：静态导入结合`preload`，确保首屏渲染速度。
- **高频访问页面**：动态 `import()` 并配合 `prefetch` 预加载，在浏览器空闲时提前下载。
- **低频访问页面**：仅动态 `import()`，按需加载。

