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

# Fetch 与 Ajax 的区别

## 1. 定义与设计哲学
- **Ajax**：基于 `XMLHttpRequest` 对象，采用事件回调模型，是早期标准方案。
- **Fetch**：新的网络请求 API，基于 Promise 设计，提供更现代、简洁的接口，支持 `async/await`。

## 2. API 与语法
- **Ajax**：需通过 `open()`、`setRequestHeader()` 等方法配置，通过 `onreadystatechange` 或 `onload`/`onerror` 处理响应，响应数据通过 `responseText` 或 `response` 直接获取。
- **Fetch**：通过第二个参数对象统一配置（如 `method`、`headers`、`body`），返回 `Response` 对象，需调用 `.json()`、`.text()` 等方法解析数据，支持链式调用。

## 3. 错误处理
- **Ajax**：网络错误、HTTP 错误状态码（如 404、500）均会触发 `onerror` 或 `onfail`。
- **Fetch**：仅在网络请求失败（如断网）时 reject，HTTP 错误状态码不会 reject，需手动检查 `response.ok`。

## 4. 扩展性与拦截器
- **Ajax**：可通过拦截器（如 Axios 封装）实现请求/响应拦截。
- **Fetch**：原生不支持拦截器，需通过包装函数或第三方库实现。

## 5. 兼容性与现代特性
- **Ajax**：兼容所有现代浏览器及 IE5+，功能相对基础。
- **Fetch**：不兼容 IE，需引入 polyfill；支持流式请求（`ReadableStream`）、请求取消（`AbortController`）、跨域请求配置（`mode: 'cors'`）等现代特性。

## 语法示例
- **Ajax**：
  ```javascript
  const xhr = new XMLHttpRequest();
  xhr.open('GET', '/api/data');
  xhr.onload = function() {
    if (xhr.status === 200) {
      console.log(JSON.parse(xhr.responseText));
    }
  };
  xhr.onerror = function() { console.error('请求失败'); };
  xhr.send();
  ```
- **Fetch**：
  ```javascript
  fetch('/api/data')
    .then(response => {
      if (!response.ok) throw new Error('网络错误');
      return response.json();
    })
    .then(data => console.log(data))
    .catch(error => console.error('请求失败', error));
  ```

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

# JavaScript事件循环

## 1. 宏任务与微任务

JS 中异步任务分为两类：

### 宏任务（Macro Task）

由宿主环境（浏览器/Node.js）发起的任务，每次事件循环从宏任务队列中**取出一个**执行。

**常见宏任务：**

| 来源 | 示例 |
|------|------|
| 整体脚本 | `<script>` 标签内的同步代码整体作为一个宏任务 |
| 定时器 | `setTimeout`、`setInterval` |
| I/O 操作 | XHR、FileReader、`fetch` 回调 |
| 事件回调 | `click`、`keydown`、`scroll` 等 DOM 事件 |
| 浏览器渲染 | UI rendering、`requestAnimationFrame`(部分浏览器归为宏任务) |
| Node.js | `setImmediate`、`I/O 回调` |

### 宏任务的优先级

HTML 规范中定义了多种**任务源（Task Source）**，不同来源的宏任务进入不同的任务队列。浏览器从多个队列中按优先级取任务，不会让低优任务阻塞高优任务：

```
高优先级（浏览器优先取）
  ├── 用户交互队列：click、keydown、input、scroll、mousemove 等
  ├── 渲染队列：requestAnimationFrame、UI 渲染
  └── 网络队列：fetch/XHR 回调

低优先级
  └── 定时器队列：setTimeout、setInterval
```

同一队列内按 FIFO 顺序执行，不同队列间高优先级队列优先被事件循环取出。

例如 `setTimeout` 里写死循环，页面也不会完全卡死——浏览器仍会优先调度你的点击/输入事件处理回调。

微任务不参与宏任务优先级竞争，始终在当前宏任务内部一次性清空。

### 微任务（Micro Task）

由 JS 引擎自身发起的异步任务，在当前宏任务执行完后、下一个宏任务执行前**一次性清空**整个微任务队列。

**常见微任务：**

| 来源 | 示例 |
|------|------|
| Promise | `.then()`、`.catch()`、`.finally()` |
| `async/await` | `await` 后面的代码（本质是 Promise.then） |
| MutationObserver | DOM 变化监听回调 |
| `queueMicrotask()` | 显式创建微任务 |
| Node.js | `process.nextTick`（优先级高于 Promise.then） |

---

## 2. 浏览器线程视角下的完整流程

浏览器是多线程的，不同线程各司其职：

| 线程 | 职责 |
|------|------|
| **JS 引擎线程（主线程）** | 执行 JS 代码、解析 DOM、事件处理、执行栈管理 |
| **GUI 渲染线程** | 页面重绘/回流，与 JS 引擎线程**互斥**（同一时间只有一个在跑） |
| **定时器线程** | 计时 `setTimeout`/`setInterval`，到点后把回调推入宏任务队列 |
| **事件触发线程** | 用户交互事件触发后，把回调推入宏任务队列 |
| **网络线程** | 处理 Ajax/Fetch 请求，响应后把回调推入宏任务队列 |

### 一次完整的事件循环

```
┌─────────────────────────────────────────────┐
│  1. 从宏任务队列取出第一个任务，推入执行栈      │
│  2. 执行栈执行该任务中的同步代码                │
│     → 遇到微任务会放入微任务队列               │
│     → 遇到宏任务丢给对应线程，到点后回调进入队列  │
│  3. 当前宏任务的同步代码执行完毕                │
│  4. 清空微任务队列（期间新产生的微任务也会执行）  │
│  5. 浏览器可能在这时进行 UI 渲染（如果需要）     │
│  6. 取下一个宏任务，回到步骤 1                 │
└─────────────────────────────────────────────┘
```

### JS执行栈

JS 是单线程，通过**调用栈（Call Stack）**管理函数执行。函数被调用时，对应的**执行上下文（Execution Context）**压入栈顶；函数返回后，上下文从栈顶弹出。

```javascript
function a() {
  console.log('a');
  b();
  console.log('a end');
}
function b() {
  console.log('b');
}
a();
```

执行栈变化过程：

```
1. a 调用 → push <a 上下文>
2. a 内部执行 console.log('a') → 输出 'a'
3. a 内部调用 b → push <b 上下文>
4. b 内部执行 console.log('b') → 输出 'b'
5. b 执行完毕 → pop <b 上下文>
6. 回到 a，执行 console.log('a end') → 输出 'a end'
7. a 执行完毕 → pop <a 上下文>，栈空

最终输出：a → b → a end
```

异步回调被推入任务队列，必须等执行栈清空后，事件循环才会将其取出入栈执行：

```javascript
console.log('1');
setTimeout(() => console.log('2'), 0);
console.log('3');

// 输出顺序：1 → 3 → 2
// 执行栈执行完同步代码（1,3）清空后，setTimeout 回调才被推入执行栈
```

---

## 3. 任务执行顺序

### 基本规则

1. **同步代码优先**：执行栈中的同步代码直接执行
2. **微任务清空**：当前宏任务同步代码执行完后，一次性清空微任务队列
3. **取下一个宏任务**：微任务清空后，从宏任务队列取出**一个**压入执行栈
4. 重复步骤 1-3

```
┌─ 宏任务 ───────────────────────────────────┐
│  同步代码 (执行栈)                           │
│  ↓ 同步代码执行完                            │
│  微任务队列 (一次性清空)                      │
│  ↓ 微任务清空                                │
│  [必要时 UI 渲染]                            │
├────────────────────────────────────────────┤
│  下一个宏任务 → 重复                          │
└────────────────────────────────────────────┘
```

### 示例

```javascript
console.log('1');

setTimeout(() => {
  console.log('2');
  Promise.resolve().then(() => console.log('3'));
}, 0);

Promise.resolve().then(() => console.log('4'));

console.log('5');

// 输出：1 → 5 → 4 → 2 → 3
```

流程分析：

```
初始宏任务（script 整体）：
  - 同步代码：'1'、'5'
  - 微任务队列：.then('4')
  - 宏任务队列：setTimeout 回调

清空微任务 → '4'

下一个宏任务（setTimeout 回调）：
  - 同步代码：'2'
  - 微任务队列：.then('3')

清空微任务 → '3'
```

---

## 4. 微任务中产生新微任务

微任务在执行过程中产生了新的微任务，会**追加到当前微任务队列尾部**，在本轮清空阶段继续执行，直到微任务队列彻底为空。

```javascript
console.log('start');

Promise.resolve().then(() => {
  console.log('微1');
  Promise.resolve().then(() => {
    console.log('微2');
    Promise.resolve().then(() => console.log('微3'));
  });
});

setTimeout(() => console.log('宏'), 0);

console.log('end');

// 输出：start → end → 微1 → 微2 → 微3 → 宏
```

流程：微1 执行时产生了微2，微2 加入队列尾部在当前轮继续执行；微2 执行时产生微3，同样继续执行。全部微任务清空后才轮到宏任务。

---

## 5. 宏任务中产生新宏任务

宏任务执行过程中新产生的宏任务（如 `setTimeout` 回调里再调 `setTimeout`），会被**追加到宏任务队列尾部**，等当前宏任务结束且微任务清空后，才会被取出来执行。

```javascript
setTimeout(() => {
  console.log('宏1');
  setTimeout(() => console.log('宏2'), 0);
  Promise.resolve().then(() => console.log('微'));
}, 0);

setTimeout(() => console.log('宏3'), 0);

// 输出：宏1 → 微 → 宏3 → 宏2
```

流程：宏1 执行时把宏2 推入宏任务队列尾部（排在宏3 后面），然后清空微任务（微），接着取宏3，最后才轮到宏2。

---

## 6. 总结流程图

```
             ┌──────────────────┐
             │  执行栈（主线程）   │
             │  执行同步代码      │
             └────────┬─────────┘
                      │ 同步代码执行完毕
                      ▼
             ┌──────────────────┐
             │  清空微任务队列    │──── 微任务产生新微任务 → 继续清空
             └────────┬─────────┘
                      │ 微任务队列为空
                      ▼
             ┌──────────────────┐
             │  [可能 UI 渲染]   │
             └────────┬─────────┘
                      ▼
             ┌──────────────────┐
             │ 取一个宏任务入栈   │──── 宏任务产生新宏任务 → 放入队列尾部
             └──────────────────┘
                      │
                      └────── 循环 ──────┘
```

**口诀**：一个宏任务 → 清空微任务 → 可选渲染 → 下一个宏任务
