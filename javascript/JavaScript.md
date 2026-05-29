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