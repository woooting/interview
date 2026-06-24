# 堆内存与栈内存

## 1. 栈与堆概述

### 1.1 栈内存

栈是**连续内存**区域，采用**后进先出（LIFO）**结构，由系统自动分配和释放，速度快。用于存储基本类型值和引用类型的地址指针。

- **大小固定**：编译时确定，每个变量所需空间已知
- **自动管理**：函数调用时入栈分配，返回时自动弹栈释放
- **访问快速**：连续内存布局，通过栈指针偏移直接读写

### 1.2 堆内存

堆是**非连续内存**区域，大小动态，需垃圾回收器（GC）自动回收，速度较慢。用于存储**引用类型的实际数据**。

- **大小不固定**：对象的属性可动态增删，运行时分配
- **生命周期不定**：可能被函数外部引用，不能随函数结束自动释放
- **通过GC回收**：当没有引用指向对象时，GC 在合适时机回收

### 1.3 完整对比

| 维度 | 栈 | 堆 |
|------|-----|-----|
| 结构 | 连续内存，LIFO | 非连续内存 |
| 分配方式 | 编译时确定，自动分配 | 运行时动态分配 |
| 释放方式 | 函数返回自动弹栈 | 垃圾回收器（GC） |
| 速度 | 快 | 慢 |
| 空间大小 | 小（~1MB） | 大（GB 级） |
| 存储内容 | 基本类型 + 引用地址 | 引用类型实际数据 |
| 线程安全 | 每个线程独立栈 | 堆共享，需同步机制 |

## 2. 引用类型为何存在堆上

### 2.1 三个原因

**大小不确定**：对象可动态增删属性，编译时无法确定大小，栈要求编译时知道每个变量占多少字节。

```javascript
const obj = {}
obj.a = 1
obj.b = { c: { d: [1, 2, 3] } }
```

**生命周期不确定**：对象可能被函数外部引用（闭包、全局变量），不能随函数结束销毁。栈的生命周期固定为函数调用期间。

```javascript
function createUser() {
  const user = { name: 'Tom' }
  return user
}
const u = createUser()  // 栈帧已销毁，堆上的 user 仍存活
```

**引用共享**：同一对象可能被多个变量引用，栈只存地址，多个栈帧中的地址指向同一堆内存。

```javascript
const a = { value: 1 }
const b = a
a.value = 2
console.log(b.value)  // 2 — 共享堆中的同一份数据
```

### 2.2 变量赋值的本质

```javascript
let num = 42        // 栈：存值 42
let obj = { a: 1 }  // 栈：存地址 → 堆：存 { a: 1 }

let num2 = num      // 栈：复制值（独立副本）
let obj2 = obj      // 栈：复制地址（指向堆中同一对象）

num2 = 100          // 不影响 num
obj2.a = 2          // obj.a 也变成 2
```

```
             栈内存                        堆内存
    ┌──────────────────┐          ┌──────────────────┐
    │  num: 42          │          │                  │
    │  obj: ─────────────┼─────────┼─→│ { a: 1 }     │ │
    │  num2: 100        │          │  └─────────────┘ │
    │  obj2: ────────────┼─────────┼─→      ↑         │
    └──────────────────┘          │       共享        │
                                  └──────────────────┘
```

### 2.3 闭包与堆内存

闭包中的变量虽然声明在函数内，但被内部函数引用后，会被提升到**堆上存储**。因为栈帧释放后闭包还需要持有这些变量。

```javascript
function outer() {
  let count = 0       // count 不在栈上，而在堆上（闭包将其"闭包化"）
  return function inner() {
    count++
    return count
  }
}
const fn = outer()    // outer 栈帧释放，但 count 仍在堆上存活
```

## 3. 栈溢出（Stack Overflow）

函数调用嵌套过深导致栈空间耗尽时发生。浏览器约 1~2MB，Node.js 默认约 984KB。

```javascript
function infinite() { return infinite() }
infinite()  // RangeError: Maximum call stack size exceeded
```

## 4. 面试总结

> "栈是连续内存，LIFO 结构，编译时确定大小，自动分配释放，速度快，存基本类型和引用地址；
> 堆是非连续内存，运行时动态分配，GC 回收，速度慢，存引用类型实际数据。
> 引用类型必须放堆上，因为对象大小不确定、生命周期不固定、需要引用共享——栈无法满足。
> 闭包中的变量看似在函数内，但其实也被提升到了堆上，确保函数执行完后变量仍存活。"


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


# 执行上下文、作用域与闭包

## 1. 执行上下文（Execution Context）

### 1.1 三种执行上下文

| 类型 | 创建时机 |
|------|---------|
| **全局执行上下文** | JS 引擎启动时创建，只有一个，压入栈底 |
| **函数执行上下文** | 每次调用函数时创建，函数执行完毕弹出 |
| **eval 执行上下文** | `eval()` 代码执行时创建（不推荐使用） |

### 1.2 执行上下文的组成

每个执行上下文包含三个核心部分：

```
执行上下文
├── 词法环境（LexicalEnvironment）
│   ├── 环境记录（Environment Record）
│   │   ├── 声明式环境记录 → let、const、函数声明
│   │   └── 对象式环境记录 → 全局对象（window/globalThis）
│   └── 外部词法环境引用（[[OuterEnv]]） → 指向外层词法环境（形成作用域链）
│
├── 变量环境（VariableEnvironment）
│   ├── 环境记录 → var 声明的变量
│   └── 外部词法环境引用
│
└── this 绑定
```

**词法环境 vs 变量环境的区别：**

| | 词法环境 | 变量环境 |
|---|---|---|
| 用于 | `let`、`const` 声明 | `var`、函数声明（function declaration） |
| 初始化 | 声明提升但**不初始化**（暂时性死区） | 声明提升并初始化为 `undefined`（可提前访问） |

### 1.3 执行上下文的生命周期

```javascript
// 全局执行上下文：JS 引擎启动时创建，页面关闭时销毁
var a = 1
let b = 2   // b 在词法环境中

function foo() {
  var c = 3   // c 在变量环境中
  let d = 4   // d 在词法环境中（外层函数的作用域）
  if (true) {
    let e = 5 // e 在块级词法环境中（块级作用域）
  }
}
foo()
```

---

## 2. 作用域与作用域链

### 2.1 作用域的定义

作用域是变量和函数的**可访问范围**，决定了代码在何处可以访问某个变量。JS 采用**词法作用域（静态作用域）**——变量的作用域在代码编写（定义）时就已经确定，而非在运行时。

### 2.2 作用域的种类

| 作用域类型 | 范围 | 识别方式 |
|-----------|------|---------|
| **全局作用域** | 整个程序 | 代码最外层，或未在任何函数/块中定义的变量 |
| **函数作用域** | 函数内部 | 以 `function` 关键字或箭头函数定义的函数体 |
| **块级作用域** | `{}` 内部 | `let` / `const` 在 `{}` 内声明即产生块级作用域 |

### 2.3 作用域链

当访问一个变量时，JS 引擎先从当前作用域查找，如果没找到，就通过词法环境中的**外部环境引用（[[OuterEnv]]）**逐层向外查找，直到全局作用域。这条查找路径就是**作用域链**。

```
当前作用域 → 外层作用域 → 更外层作用域 → ... → 全局作用域（尽头）
```

```javascript
const globalVar = '全局'
function outer() {
  const outerVar = '外层'
  function inner() {
    const innerVar = '内层'
    console.log(innerVar)   // 当前作用域 → 找到
    console.log(outerVar)   // 当前作用域未找到 → 沿作用域链向外 → outer 中找到
    console.log(globalVar)  // 当前 → outer → 全局中找到
  }
  inner()
}
outer()
```

---

## 3. let / const / var 区别（结合作用域）

### 3.1 核心区别

| 特性 | `var` | `let` | `const` |
|------|-------|-------|---------|
| **作用域** | 函数作用域（无视 `{}`） | 块级作用域 | 块级作用域 |
| **变量提升** | 提升，初始化为 `undefined` | 提升，但未初始化（TDZ） | 提升，但未初始化（TDZ） |
| **重复声明** | 允许 | 不允许 | 不允许 |
| **重新赋值** | 允许 | 允许 | 不允许（必须初始化） |
| **挂载到全局** | 会（`window.xxx`） | 不会 | 不会 |

### 3.2 变量提升（Hoisting）与暂时性死区（TDZ）

```javascript
// var 提升：声明提升到作用域顶部，初始化为 undefined
console.log(a)  // undefined（不会报错）
var a = 1

// let / const 提升但不初始化：存在暂时性死区
console.log(b)  // ReferenceError: Cannot access 'b' before initialization
let b = 2
```

TDZ 从作用域开始到变量声明语句之间，该变量处于"暂时性死区"，在此期间访问会抛出 `ReferenceError`。

### 3.3 经典面试题：循环中的 var 与 let

```javascript
// 面试题：以下代码输出什么？
for (var i = 0; i < 5; i++) {
  setTimeout(() => console.log(i), 0)
}
// 输出：5 5 5 5 5

// 为什么？var i 是函数作用域，整个循环共享同一个 i
// 循环结束后 i = 5，五个 setTimeout 回调执行时都访问到同一个 i

// 如何修复？
// 方案一：用 let（块级作用域，每次迭代创建独立绑定）
for (let i = 0; i < 5; i++) {
  setTimeout(() => console.log(i), 0)
}
// 输出：0 1 2 3 4

// 方案二：闭包（IIFE，创建独立作用域）
for (var i = 0; i < 5; i++) {
  ((j) => setTimeout(() => console.log(j), 0))(i)
}
// 输出：0 1 2 3 4
```

**执行上下文角度解释**：每次 `for` 循环迭代，用 `let` 声明时都会创建一个新的词法环境，将当前 `i` 的值绑定到该环境中；而 `var` 声明在函数作用域中只有一份绑定，所有回调共享同一个变量。

---

## 4. 闭包（Closure）

### 4.1 闭包的定义

闭包是指函数**记住并访问其词法作用域**的能力——即使函数在其词法作用域之外执行。

核心机制：函数在定义时，内部的 `[[Environment]]` 属性记录了当前词法环境的引用，使得函数可以持续访问外层作用域的变量，即使外层函数已经执行完毕。

```javascript
function outer() {
  let count = 0
  return function inner() {
    count++          // inner 通过 [[Environment]] 找到 outer 的词法环境
    return count
  }
}
const counter = outer()   // outer 执行完毕，但 count 不会被 GC 回收
console.log(counter())     // 1
console.log(counter())     // 2
// outer 的执行上下文已弹出栈，但 outer 的词法环境仍被 inner 持有引用
```

### 4.2 闭包的应用场景

#### 防抖（Debounce）

```javascript
function debounce(fn, delay) {
  let timer = null
  return function (...args) {
    clearTimeout(timer)
    timer = setTimeout(() => fn.apply(this, args), delay)
  }
}
```

#### 节流（Throttle）

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

#### 私有变量

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

#### 函数柯里化

```javascript
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) return fn(...args)
    return (...next) => curried(...args, ...next)
  }
}
```

### 4.3 闭包与内存泄漏

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

**如何定位：**

**Chrome DevTools Memory 面板：**
1. 录制堆快照（Heap Snapshot），搜索 `(closure)` 查看闭包详情
2. 对比两个快照（操作前后），看哪些对象未被回收
3. Performance 面板录制，勾选 Memory，观察内存曲线是否持续上升不回落

**关键排查对象：**
- 全局变量引用的闭包
- 未移除的事件监听器（`Elements` → `Event Listeners`）
- 未清理的定时器
- 闭包中引用的已移除 DOM 元素

**如何防范与解决：**

| 做法 | 说明 |
|------|------|
| 避免将闭包赋值给**全局变量** | 全局引用不会被 GC 回收 |
| 及时清理事件监听 | `removeEventListener` 移除不再需要的监听 |
| 及时清理定时器 | `clearInterval` / `clearTimeout` |
| 不再需要时**置 null** | 切断闭包中的引用链 |
| 用 `WeakMap/WeakSet` 作缓存 | 键为弱引用，不影响 GC |
| 组件卸载时清理副作用 | React `useEffect` 返回清理函数 |


# js中的this

## 1. this 的本质

`this` 不是一个词法变量——它不是定义时决定的，而是**每次函数调用时动态创建**的一个隐式参数。同一个函数，不同调用方式 → 不同的 `this`。

可以把 `this` 理解为函数执行上下文中的一个属性，由调用位置和调用方式决定。

## 2. 四条绑定规则

### 2.1 默认绑定

裸函数调用（前面没有对象点号）时，`this` 指向全局对象。严格模式下指向 `undefined`。

```javascript
function foo() { console.log(this) }
foo()  // window（严格模式下 undefined）

const obj = { bar: function() { console.log(this) } }
const bar = obj.bar
bar()  // window —— 函数引用被剥离，变为裸调用
```

**隐式丢失**是最常见的坑：一旦函数从对象上取出、赋值给变量或作为回调传入，就失去了隐式绑定，退回默认绑定。

### 2.2 隐式绑定

以 `对象.方法()` 的形式调用时，`this` 指向调用链上的**最后一个对象**。

```javascript
const obj = {
  name: 'obj',
  child: {
    name: 'child',
    foo: function() { console.log(this.name) }
  }
}
obj.child.foo()  // 'child' ← 最后一个点号前面的对象
```

### 2.3 显式绑定（call / apply / bind）

通过 `call`、`apply`、`bind` 显式指定 `this`，详见下一节。

### 2.4 new 绑定

使用 `new` 调用构造函数时，引擎创建一个新对象作为 `this`。`new` 做了四件事：

1. 创建一个空对象 `{}`
2. 将该对象的 `[[Prototype]]` 指向构造函数的 `prototype`
3. 将该对象作为 `this` 传入构造函数执行
4. 如果构造函数没有返回对象，则返回这个新对象

```javascript
function Person(name) {
  this.name = name
}
const p = new Person('Tom')  // this → 新创建的对象
```

**`new` 绑定 > `bind` 绑定**：即使构造函数先被 `bind` 锁死了 `this`，`new` 调用时依然会创建一个新对象覆盖 `bind` 锁定的值。

## 3. 箭头函数的 this

箭头函数**没有自己的 `this` 槽位**，它从外层词法作用域捕获 `this`，在定义时就确定，永久不变。

```javascript
const obj = {
  name: 'obj',
  fn: () => console.log(this.name),          // this → window（定义时的外层）
  fn2: function() {
    return () => console.log(this.name)       // this → obj（外层 fn2 的 this）
  }
}
obj.fn()     // undefined（window.name）
obj.fn2()()  // 'obj'
```

一句话：**向上找最近一层非箭头函数的 `this`**，找不到就是全局。

`call`/`apply`/`bind` 对箭头函数无效——它们改变的是函数调用时的 `this`，但箭头函数根本没有这个槽位。

## 4. 优先级总结

```
new 绑定  >  显式绑定（call / apply / bind）  >  隐式绑定  >  默认绑定
```

**例外**：箭头函数无视以上所有规则，`this` 固定为定义时的词法 `this`。

## 5. 典型场景

| 场景 | this 指向 |
|------|----------|
| `obj.foo()` | `obj`（隐式绑定） |
| `foo()` | `window` / `undefined`（默认绑定） |
| `foo.call(obj)` | `obj`（显式绑定） |
| `new Foo()` | 新创建的对象（new 绑定） |
| `() => this` | 定义时外层作用域的 `this`（永远的） |
| `btn.addEventListener('click', fn)` | `btn` 元素（DOM 规范定义，非 JS 规则） |
| `setTimeout(fn, 0)` | `window` / `undefined`（相当于裸调用） |
| 类字段 `fn = () => ...` | 绑定为当前实例（等价于构造函数中 `this.fn = () => ...`） |

## 6. 底层原理

在 ECMAScript 规范中，函数调用通过内部方法 `[[Call]](thisArgument, argumentsList)` 执行，`thisArgument` 就是一条额外的隐式实参。不同调用方式决定了 `thisArgument` 的值：

- **裸调用** → `undefined`（严格模式）或全局对象
- **方法调用** `obj.foo()` → 表达式 `obj.foo` 返回的是一个 **Reference Record**（`[base: obj, name: "foo"]`），引擎从中取出 `base` 作为 `this`
- **`call`/`apply`/`bind`** → 显式传入的值
- **`new`** → 忽略传入的 `thisArgument`，创建新对象
- **箭头函数** → 内部 `[[ThisMode]]` 为 `lexical`，直接跳过 `thisArgument`，引用词法环境中的 `this`

**Reference Record 是理解隐式丢失的关键**：`const bar = obj.foo` 这个赋值操作不仅取了函数值，还丢弃了 Reference Record 中的 `base` 信息——函数被"提取"成了裸值，调用时无法再找回原本的 `this`。


# apply、call、bind 区别及底层实现原理

## 1. 语法区别

```javascript
func.call(thisArg, arg1, arg2, arg3, ...)   // 逐个传参
func.apply(thisArg, [arg1, arg2, arg3])      // 数组传参
func.bind(thisArg, arg1, arg2, ...)          // 返回新函数，不立即执行
```

| | call | apply | bind |
|---|---|---|---|
| 入参方式 | 逗号分隔，逐个传 | 数组/类数组统一传 | 逗号分隔，逐个传 |
| 是否立即调用 | ✅ 立即调用 | ✅ 立即调用 | ❌ 返回新函数，需手动调用 |
| 返回值 | 函数执行结果 | 函数执行结果 | 绑定了 this 的新函数 |
| 二次绑定 | — | — | 无效，bind 锁死不可再变 |

**记忆口诀**：call 用逗号，apply 用数组（A for Array），bind 返回新函数（B for bind 是"打包"）。

## 2. 底层实现原理

### call 实现

```javascript
Function.prototype.myCall = function(context, ...args) {
  // 1. 处理 null/undefined → 指向全局
  context = context ?? window
  // 2. 将调用函数临时挂载到 context 上（作为方法）
  const symbolKey = Symbol()
  context[symbolKey] = this         // this 就是调用 myCall 的那个函数
  // 3. 以 context.xxx() 的形式调用 → 触发隐式绑定 → this 指向 context
  const result = context[symbolKey](...args)
  // 4. 清理临时属性
  delete context[symbolKey]
  return result
}
```

核心思想：**将函数挂到 context 对象上成为方法，以 `对象.方法()` 的方式调用，利用隐式绑定规则让 `this` 指向 context**。`Symbol` 确保不污染 context 的原有属性。

### apply 实现

原理与 call 完全相同，唯一区别是参数以数组形式接收：

```javascript
Function.prototype.myApply = function(context, args = []) {
  context = context ?? window
  const symbolKey = Symbol()
  context[symbolKey] = this
  const result = context[symbolKey](...args)
  delete context[symbolKey]
  return result
}
```

### bind 实现

```javascript
Function.prototype.myBind = function(context, ...bindArgs) {
  const originalFn = this  // 原函数
  return function boundFn(...callArgs) {
    // new 调用时，this 是 boundFn 的实例，应优先使用
    // 普通调用时，使用绑定的 context
    return originalFn.apply(
      this instanceof boundFn ? this : context,
      [...bindArgs, ...callArgs]
    )
  }
}
```

**关键点：**
1. `bind` 返回一个闭包函数 `boundFn`
2. `boundFn` 内部用 `apply` 实现 this 绑定
3. 合并 `bind` 时的参数和调用时的参数（柯里化特性）
4. **`new` 优先级问题**：当 `new boundFn()` 时，`this instanceof boundFn` 为 `true`，使用新创建的对象而不是绑定的 context，确保 `new` 优先级高于 `bind`
5. **不可二次绑定**：返回的 `boundFn` 不会再被 `bind` 影响（因为它只是一个普通函数，里面用 `apply` 固定了逻辑）

### 为什么 bind 不可二次更改

```javascript
function fn() { console.log(this.name) }
const bound1 = fn.bind({ name: 'A' })
const bound2 = bound1.bind({ name: 'B' })
bound2()  // 'A'，不是 'B'
```

原因在于 `bind` 返回的是一个新的 `boundFn`，而非原来的 `fn`。对 `bound1` 它的内部实现已经固定了对原函数的 `apply(context)`，第二次 `bind` 改变的是 `boundFn` 这个包装函数的上下文，而 `boundFn` 内部调用的始终是第一次绑定时的 `context`。

### 箭头函数为何失效

箭头函数没有自己的 `this`，它内部的 `this` 是从定义时词法环境捕获的常量。`call`/`apply`/`bind` 试图改变的是函数调用时的 `thisArgument` 隐式参数，但箭头函数的 `[[ThisMode]]` 是 `lexical`，引擎直接忽略这个参数——传了也白传。


# 箭头函数特性

## 1. 核心特性

箭头函数除了语法更短，在语义上有 5 个核心差异：

### 1.1 没有自己的 this（最重要的区别）

箭头函数的 `this` 是从**定义时**外层词法作用域捕获的，是一个常量，永远不会变。详见 `js中的this > 箭头函数的 this`。

```javascript
const obj = {
  name: 'obj',
  greet: function() { setTimeout(function() { console.log(this.name) }, 100) },
  greetArrow: function() { setTimeout(() => console.log(this.name), 100) }
}
obj.greet()       // undefined（普通函数 this → window）
obj.greetArrow()  // 'obj'（箭头函数 this 继承自 greetArrow 的 this）
```

### 1.2 没有 arguments 对象

箭头函数内部没有 `arguments`，需用 rest 参数替代：

```javascript
const fn = () => {
  console.log(arguments)  // 引用外层函数的 arguments（如果有的话）
}

function outer() {
  const inner = () => console.log(arguments)
  inner()
}
outer(1, 2, 3)  // Arguments(3) [1, 2, 3] — 拿到的是 outer 的 arguments

// 正确做法：用 rest 参数
const sum = (...args) => args.reduce((a, b) => a + b, 0)
sum(1, 2, 3)  // 6
```

### 1.3 不能用作构造函数

箭头函数没有 `[[Construct]]` 内部方法，`new` 会直接抛出 TypeError：

```javascript
const Foo = () => {}
new Foo()  // TypeError: Foo is not a constructor
```

这也意味着箭头函数没有 `prototype` 属性：

```javascript
const fn = () => {}
console.log(fn.prototype)  // undefined
```

### 1.4 不能用作 Generator

箭头函数内部不能使用 `yield`：

```javascript
const gen = () => { yield 1 }  // SyntaxError
```

### 1.5 不能用作类方法（没有 super）

箭头函数没有 `[[HomeObject]]`，无法使用 `super`：

```javascript
class Parent {
  greet() { return 'hello' }
}
class Child extends Parent {
  greet = () => super.greet()  // SyntaxError
}

// 正确：普通方法或简写方法
class Child2 extends Parent {
  greet() { return super.greet() }  // ✅
}
```

## 2. 与普通函数的完整对比

| 特性 | 普通函数 | 箭头函数 |
|------|---------|---------|
| **`this`** | 调用时动态决定 | 定义时词法捕获，不可变 |
| **`call/apply/bind`** | 可以改变 `this` | 无效，`this` 恒定 |
| **`arguments`** | 有 | 无，需用 `...args` |
| **`new` 调用** | ✅ 可作为构造函数 | ❌ TypeError |
| **`prototype`** | 有（普通函数） | ❌ undefined |
| **`super`** | 类方法中可用 | ❌ 不可用 |
| **`yield`** | 可用（Generator） | ❌ 不可用 |
| **语法** | `function() { return x }` | `() => x`（隐式返回表达式） |
| **简写** | — | 单参数可省 `()`；单表达式可省 `{}` 和 `return` |

## 3. 适用场景

| 场景 | 推荐 | 原因 |
|------|------|------|
| 回调函数（定时器、事件） | ✅ 箭头函数 | 自动继承外层 `this`，省去 `bind` |
| 对象方法 | ❌ 普通函数 | 箭头函数的 `this` 不会指向该对象 |
| 构造函数 / 类 | ❌ 普通函数 | 箭头函数不能 `new` |
| 数组方法（map/filter） | ✅ 均可 | 不涉及 `this` 时箭头函数更简洁 |
| 动态 `this` 场景 | ❌ 普通函数 | 需要运行时决定 `this` 指向 |
| 需要 `arguments` | ❌ 普通函数 | 箭头函数没有 `arguments` |
| React 类组件方法 | ✅ 箭头函数 | 自动绑定实例，回调引用不丢失 `this` |
| Vue methods | ✅ 普通函数 | Vue 会自动绑定 `this`，箭头函数反而拿不到 Vue 实例 |

**一句话记忆**：需要自己的 `this` 用普通函数，不需要自己的 `this` 用箭头函数。


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

# Fetch 与 Ajax (XHR)

Ajax 是基于 `XMLHttpRequest` 的事件回调方案；Fetch 是 ES6 基于 Promise 的原生 API。核心区别如下：

1. **编程模型** — XHR 事件回调（`onload`/`onerror`），Fetch 的 Promise 链式调用，可用 `async/await` 扁平化。

   ```javascript
   // XHR
   const xhr = new XMLHttpRequest()
   xhr.open('GET', '/api/data')
   xhr.onload = () => {
     if (xhr.status === 200) JSON.parse(xhr.responseText)
   }
   xhr.send()

   // Fetch
   const res = await fetch('/api/data')
   if (!res.ok) throw new Error(`HTTP ${res.status}`)
   const data = await res.json()
   ```

2. **错误处理** — 两者行为一致：HTTP 404/500 都不算异常。XHR 进 `onload`，Fetch 正常 resolve，都需手动检查 `status`/`response.ok`。只有断网等网络层错误才会触发 `onerror` 或 reject。

   ```javascript
   // XHR：404 进 onload，需手动判断
   xhr.onload = () => { if (xhr.status >= 400) { /* HTTP 错误 */ } }
   xhr.onerror = () => { /* 网络层错误 */ }

   // Fetch：404 正常 resolve，需检查 ok
   fetch('/api/data').then(res => {
     if (!res.ok) throw new Error(`HTTP ${res.status}`)
   }).catch(err => { /* 网络层错误 + 手动 throw */ })
   ```

3. **请求取消** — XHR 用 `xhr.abort()` 直接取消单个请求；Fetch 用 `AbortController` + `signal`，一个信号可同时取消多个请求。

   ```javascript
   const ctrl = new AbortController()
   fetch('/a', { signal: ctrl.signal })
   fetch('/b', { signal: ctrl.signal })
   ctrl.abort()  // 同时取消 a 和 b
   // 超时：AbortSignal.timeout(5000)
   ```

4. **超时** — XHR 内置 `xhr.timeout` + `ontimeout` 事件；Fetch 无内置，需用 `AbortSignal.timeout()` 或手动 `setTimeout` + `abort()`。

5. **Cookie 携带** — XHR 同域默认携带；Fetch 默认不携带，需显式设置 `credentials: 'include'`（同域）/ `'same-origin'`（默认）/ `'omit'`。

6. **进度监听** — XHR 原生支持 `progress` 事件（下载）和 `upload.onprogress`（上传）；Fetch 不直接支持，需借助 `ReadableStream` 逐块计算。

   ```javascript
   xhr.upload.onprogress = (e) => {
     console.log(`${e.loaded}/${e.total}`)
   }
   ```

7. **流式读取** — Fetch 独有，`response.body` 是 `ReadableStream`，可逐块处理响应（AI 对话流式输出、大文件下载）。XHR 只能等完整响应。

   ```javascript
   const res = await fetch('/api/stream')
   const reader = res.body.getReader()
   const decoder = new TextDecoder()
   while (true) {
     const { done, value } = await reader.read()
     if (done) break
     console.log('收到:', decoder.decode(value, { stream: true }))
   }
   ```

8. **Service Worker** — SW 中只能用 Fetch，不支持 XHR。
9. **重定向控制** — Fetch 可通过 `redirect: 'manual'` / `'error'` 控制；XHR 默认跟随不可控。
10. **兼容性** — XHR（IE5+）优于 Fetch（IE 不支持, Chrome 42+ / Firefox 39+ / Safari 10+）。

## 为什么 Ajax/Fetch 能实现无刷新更新

传统表单/链接请求会卸载整页重新加载；Ajax/Fetch 将请求控制权交给 JS，网络线程异步通信不阻塞主线程，拿到响应数据后通过 DOM API 局部更新页面，无需整页替换。同时浏览器不会卸载当前页面，页面状态得以保持。


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


# File、Blob、ArrayBuffer

## 关系总览

```
File 继承 Blob，Blob 是 File 的父类。
ArrayBuffer 是底层二进制缓冲区，可被 Blob 引用，也可通过 TypedArray/DataView 读写。

                  Blob（不可变，带 MIME 类型）
                 /      \
           File（带文件名）  new Blob()（程序化生成）
              │
     ArrayBuffer（内存中的原始字节，可读写）
              │
     Uint8Array / DataView（操作 ArrayBuffer 的视图）
```

### 核心区别

| 维度 | Blob / File | ArrayBuffer |
|------|-------------|-------------|
| 本质 | 浏览器管理的不可变二进制"文件" | JS 堆内存中的可变二进制缓冲区 |
| 读写 | ❌ 只读（slice 也只能产出新 Blob） | ✅ 通过 TypedArray 直接改每个字节 |
| 元信息 | 有 `size`、`type`（MIME）、File 还有 `name` | 只有 `byteLength` |
| 内存位置 | 浏览器内核管理，不在 JS 堆 | V8 的 ArrayBuffer 分配区 |
| 典型场景 | 文件上传、图片预览、下载、切片 | 图片像素操作、音频解码、加密、协议解析、WebGL |

---

## Blob

### 是什么

不可变的二进制数据容器，代表一个"文件片段"，有 `size` 和 `type`（MIME 类型）。

### 常见来源

```js
new Blob(['hello'], { type: 'text/plain' })
canvas.toBlob()
fetch(url).then(r => r.blob())
file.slice()         // ⚠️ 返回 Blob，不是 File
```

### 常见 API

| API | 说明 | 返回值 |
|-----|------|--------|
| `blob.size` | 字节大小 | number |
| `blob.type` | MIME 类型 | string |
| `blob.slice(start?, end?, type?)` | 切片，返回新 Blob（零拷贝） | Blob |
| `blob.text()` | 读为字符串 | Promise\<string\> |
| `blob.arrayBuffer()` | 读为 ArrayBuffer | Promise\<ArrayBuffer\> |
| `blob.stream()` | 读为 ReadableStream | ReadableStream |

### 应用场景

- **JS 生成文件下载**：`new Blob([data], { type })` → `URL.createObjectURL(blob)` → `<a download>`
- **切片上传**：`file.slice(...)` 切出多块，放入 FormData 发送
- **文件预览**：`URL.createObjectURL(blob)` 给 `<img>` / `<video>` / `<iframe>`

---

## File

### 是什么

Blob 的子类，增加了文件名和最后修改时间。

```js
File.prototype.__proto__ === Blob.prototype  // true
```

### 常见来源

```js
<input type="file"> .files[0]
拖拽事件的 e.dataTransfer.files[0]
new File([blob], 'name.txt', { type: 'text/plain', lastModified: Date.now() })
```

### 额外属性

| 属性 | 说明 |
|------|------|
| `file.name` | 文件名（含扩展名） |
| `file.lastModified` | 最后修改时间戳 |
| `file.webkitRelativePath` | 目录上传时的相对路径 |

### 应用场景

- **用户上传文件**：`<input type="file">` 拿到的就是 File
- **需要文件名时**：切片上传时用 `file.name` 传给服务端做标识
- **File 和 Blob 互换**：File 可以直接当 Blob 用（所有 Blob API 都能调）

---

## ArrayBuffer

### 是什么

内存中一段固定长度的原始二进制缓冲区，**不能直接读写**，必须通过 TypedArray 或 DataView 操作。

### 常见来源

```js
new ArrayBuffer(1024)                        // 创建 1KB 的缓冲区
await blob.arrayBuffer()                     // Blob 转 ArrayBuffer
await fetch(url).then(r => r.arrayBuffer())  // 网络请求读为二进制
reader.readAsArrayBuffer(file)               // FileReader 方式
crypto.subtle.digest('SHA-256', buffer)      // 加密 API 返回 ArrayBuffer
```

### 视图操作

```js
const buf = new ArrayBuffer(8)

// 按不同视图读同一块内存
const u8  = new Uint8Array(buf)    // 8 个字节
const u16 = new Uint16Array(buf)   // 4 个 short
const u32 = new Uint32Array(buf)   // 2 个 int
const f64 = new Float64Array(buf)  // 1 个 double
const dv  = new DataView(buf)      // 混合类型 + 控制字节序

u8[0] = 0xFF  // 直接改字节
```

### 常见 API（TypedArray）

| API | 说明 |
|-----|------|
| `new Uint8Array(buffer, byteOffset?, length?)` | 在 buffer 上创建视图，不拷贝 |
| `typedArray.byteLength` | 视图占用的字节数 |
| `typedArray.buffer` | 指向底层的 ArrayBuffer |
| `typedArray.slice(start, end)` | 拷贝一段新 TypedArray |
| `typedArray.set(array, offset)` | 从 offset 开始批量赋值 |

### 应用场景

- **图片像素操作**：`ctx.getImageData().data` → Uint8Array 改每个 RGBA 值
- **二进制协议解析**：DataView 按大端/小端读 int/float
- **WebSocket 二进制通信**：`socket.binaryType = 'arraybuffer'`
- **WebGL 顶点数据**：Float32Array 传给 GPU
- **加密/哈希**：`crypto.subtle.digest('SHA-256', buffer)`
- **WASM 内存交互**：共享 WASM 的 `memory.buffer`
- **Web Worker 零拷贝传数据**：Transferable 对象 `postMessage(buf, [buf])`
- **音频处理**：`decodeAudioData(arrayBuffer)`

---

## 转换关系

```js
// Blob → ArrayBuffer（读进内存）
const buffer = await blob.arrayBuffer()

// ArrayBuffer → Blob（加上 MIME）
const blob2 = new Blob([buffer], { type: 'image/png' })

// Blob → File（加上文件名）
const file = new File([blob], 'photo.png', { type: blob.type })

// File → Blob（无损耗，File 就是 Blob）
const blob3 = file  // 直接当 Blob 用

// Blob → Base64
function blobToBase64(blob) {
  return new Promise(resolve => {
    const reader = new FileReader()
    reader.onload = () => resolve(reader.result.split(',')[1])
    reader.readAsDataURL(blob)
  })
}

// Base64 → Blob
function base64ToBlob(b64, type) {
  const bin = atob(b64)
  const u8 = new Uint8Array(bin.length)
  for (let i = 0; i < bin.length; i++) u8[i] = bin.charCodeAt(i)
  return new Blob([u8], { type })
}

// ArrayBuffer 转 Base64
function arrayBufferToBase64(buffer) {
  const u8 = new Uint8Array(buffer)
  const chars = u8.reduce((acc, b) => acc + String.fromCharCode(b), '')
  return btoa(chars)
}
```

---

## 选择决策

```
需要操作文件？
├── 只要上传/预览/下载（不改数据）→ Blob / File
│   ├── 用户选文件 → File
│   ├── JS 生成数据 → new Blob()
│   └── 切片上传 → Blob.slice() + FormData
│
├── 要读写字节（改像素/解析协议/加密）→ ArrayBuffer
│   └── blob.arrayBuffer() → Uint8Array → 改完转回 Blob
│
└── 要流式处理大文件 → ReadableStream
    └── blob.stream() 或 fetch.body 直接传流
```

---

## 上传流程完整示例

```js
// 前端：切片 + 并发上传
async function uploadFile(file) {
  const CHUNK_SIZE = 5 * 1024 * 1024
  const totalChunks = Math.ceil(file.size / CHUNK_SIZE)
  const chunks = []

  for (let i = 0; i < totalChunks; i++) {
    chunks.push(file.slice(i * CHUNK_SIZE, (i + 1) * CHUNK_SIZE))
  }

  async function uploadChunk(blob, index) {
    const fd = new FormData()
    fd.append('chunk', blob, file.name)
    fd.append('filename', file.name)
    fd.append('index', index)
    fd.append('totalChunks', totalChunks)
    return fetch('/upload', { method: 'POST', body: fd })
  }

  // 并发 3 个，失败单个重试
  for (let i = 0; i < chunks.length; i += 3) {
    const batch = chunks.slice(i, i + 3).map((blob, j) =>
      uploadChunk(blob, i + j).catch(() => uploadChunk(blob, i + j))
    )
    await Promise.all(batch)
  }

  // 通知服务端合并
  await fetch('/upload/merge', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ filename: file.name, totalChunks })
  })
}

// Node 服务端：接收 + 合并
const express = require('express')
const fs = require('fs')
const path = require('path')
const multer = require('multer')
const upload = multer({ dest: 'tmp/' })
const app = express()

app.post('/upload', upload.single('chunk'), (req, res) => {
  const { filename, index } = req.body
  const dir = path.join('tmp', filename)
  fs.mkdirSync(dir, { recursive: true })
  fs.renameSync(req.file.path, path.join(dir, `part_${index}`))
  res.json({ ok: true })
})

app.post('/upload/merge', express.json(), (req, res) => {
  const { filename, totalChunks } = req.body
  const dir = path.join('tmp', filename)
  const dest = fs.createWriteStream(path.join('uploads', filename))
  for (let i = 0; i < totalChunks; i++) {
    dest.write(fs.readFileSync(path.join(dir, `part_${i}`)))
  }
  dest.end()
  fs.rmSync(dir, { recursive: true })
  res.json({ ok: true })
})

app.listen(3000)
```

# 可选链和空值合并符

## 1. 可选链（Optional Chaining）`?.`

### 作用

`?.` 允许在访问深层嵌套的对象属性时，如果中间某个值为 `null` 或 `undefined`，立即短路返回 `undefined`，而不是抛出 `TypeError`。

三种语法形式：
- `obj?.prop` — 安全访问属性
- `obj?.[expr]` — 安全动态属性访问
- `func?.()` — 安全调用函数

### 对比示例

```javascript
// 没有可选链：中间属性为 null/undefined 时报错
const user = { profile: null }
user.profile.address.city
// TypeError: Cannot read properties of null (reading 'address')

// 使用可选链：安全返回 undefined
user?.profile?.address?.city
// undefined（不会报错）
```

相比传统写法：

```javascript
// 传统方案：&& 逐层判断（冗长易漏）
const city = user && user.profile && user.profile.address && user.profile.address.city

// 可选链：链式直达
const city = user?.profile?.address?.city
```

### 适用场景

| 场景 | 说明 |
|------|------|
| API 响应解析 | 深层嵌套的 JSON 数据，部分字段可能缺失 |
| 可选回调 | `callback?.()` 避免 `callback is not a function` |
| DOM 查询 | 元素可能不存在时安全访问属性 |
| 配置对象 | 嵌套的配置项可能有缺省 |

---

## 2. 空值合并符（Nullish Coalescing）`??`

### 作用

`??` 仅在左侧表达式为 `null` 或 `undefined` 时返回右侧默认值。与 `||` 的核心区别：`??` 不会将 `''`、`0`、`false` 视为"空值"而替换。

### 对比示例

```javascript
// || 的陷阱：所有假值都会被替换
const count = 0 || 10      // 10（0 被错误替换）
const name = '' || '默认'   // '默认'（空字符串被错误替换）
const valid = false || true // true（false 被错误替换）

// ?? 只替换 null/undefined
const count = 0 ?? 10         // 0 ✅（0 是有效值，保留）
const name = '' ?? '默认'     // '' ✅（空串是有效值，保留）
const valid = false ?? true   // false ✅（false 是有效值，保留）
const value = null ?? '默认'  // '默认'（null 才替换）
const value2 = undefined ?? '默认'  // '默认'
```

### 适用场景

| 场景 | 说明 |
|------|------|
| 函数参数默认值 | 区分"传了 0/false"和"没传" |
| 数值计算 | 避免错误替换有效值 0 |
| 配合可选链 | `user?.age ?? 18` 优雅提供兜底 |

---

## 3. 两者配合使用

```javascript
// 从深层对象安全取值 + 提供默认值
const city = user?.profile?.address?.city ?? '未知城市'
```

## 4. 注意事项

- `?.` 和 `??` 都是 **ES2020** 特性，现代浏览器和 Node 14+ 原生支持，TypeScript 3.7+ 支持编译
- 禁止将 `??` 与 `&&` 或 `||` 直接混用（需要用括号明确优先级），但 `?.` 和 `??` 搭配是安全的

---

# 内存泄露

## 1. 导致前端内存泄露的常见情况

1. **意外的全局变量** — 未声明的变量挂到全局对象上，或函数内 `this` 指向全局导致属性泄露：
   ```javascript
   function foo() {
     leak = '全局变量'       // 未声明 → window.leak
     this.bar = '也泄露了'   // 普通函数 this → window
   }
   foo()
   ```

2. **被遗忘的定时器** — `setInterval` / `setTimeout` 未清理，回调持续持有大对象引用：
   ```javascript
   const bigData = new Array(1000000)
   setInterval(() => {
     console.log(bigData)   // bigData 永远无法被回收
   }, 1000)
   // clearInterval 未被调用
   ```

3. **未移除的事件监听** — DOM 元素已移除，但其监听器中引用的对象仍被持有：
   ```javascript
   const btn = document.getElementById('btn')
   const bigData = new Array(1000000)
   btn.addEventListener('click', () => {
     console.log(bigData)   // 闭包持有 bigData
   })
   // btn 从 DOM 移除后，监听器未被 removeEventListener → bigData 无法回收
   ```

4. **闭包持有大对象** — 闭包中引用的大对象不会被释放，即使闭包本身不再需要：
   ```javascript
   function createLeak() {
     const bigData = new Array(1000000)
     return function() {
       console.log('hello')
     }
   }
   const leakFn = createLeak()
   ```

5. **DOM 引用未清理** — JS 变量保存已移除 DOM 元素的引用，DOM 树无法回收：
   ```javascript
   const elements = { button: document.getElementById('btn') }
   document.body.innerHTML = ''   // DOM 清空
   // elements.button 仍指向该 DOM 节点
   ```

6. **脱离 DOM 树的节点引用** — 变量保存的 DOM 节点即使被移除也不会被回收：
   ```javascript
   const ul = document.createElement('ul')
   const li = document.createElement('li')
   ul.appendChild(li)
   document.body.appendChild(ul)
   ul.remove()
   // li 变量仍持有引用 → li 不会被 GC
   ```

7. **Map / Set 中遗忘的键** — 对象被 GC 后 `Map` 条目仍存在：
   ```javascript
   const cache = new Map()
   function process(obj) {
     cache.set(obj, { result: 'large data' })
   }
   ```

8. **控制台输出** — DevTools 控制台打印大对象，会被控制台引用而无法 GC（仅 DevTools 打开时）。

## 2. 怎么预防内存泄露

| 预防措施 | 说明 |
|---------|------|
| **使用严格模式** | `'use strict'` 阻止意外全局变量 |
| **及时清理定时器** | 组件卸载时调用 `clearInterval` / `clearTimeout` |
| **移除事件监听** | `removeEventListener` 配对，或用 `AbortController` 统一清理 |
| **使用 WeakMap / WeakSet** | 键为弱引用，不影响 GC，适合缓存场景 |
| **断开无用引用** | 不再需要的对象 `= null` 切断引用链 |
| **组件卸载清理副作用** | React `useEffect` 返回清理函数；Vue `onUnmounted` 清理 |
| **避免闭包持有不必要的大对象** | 闭包只引用真正需要的变量 |
| **慎用全局变量** | 全局变量不会被回收，尽量局限在模块作用域内 |

**WeakMap / WeakSet** 作缓存：键为弱引用，对象被 GC 后条目自动清除。
**AbortController** 批量清理事件监听：统一 `controller.abort()` 移除所有绑定的监听器。

```javascript
// WeakMap 替代 Map 做缓存
const cache = new WeakMap()
function process(obj) {
  if (cache.has(obj)) return cache.get(obj)
  cache.set(obj, expensiveData)
}

// AbortController 批量清理
class Component {
  constructor() {
    this.controller = new AbortController()
    const signal = this.controller.signal
    window.addEventListener('resize', this.onResize, { signal })
    document.addEventListener('click', this.onClick, { signal })
  }
  destroy() { this.controller.abort() }
}
```

## 3. 怎么通过 Chrome DevTools 排查内存泄露

1. **Performance 面板（宏观排查）** — 录制一段时间操作，观察内存曲线：
   - 打开 DevTools → **Performance** → 勾选 **Memory** → 录制 → 执行操作 → 停止
   - ✅ JS Heap 锯齿状（上升-回落）→ GC 正常
   - ❌ 阶梯状持续不回落 → 有内存泄露

2. **Memory 面板（精确定位）**：
   - **堆快照对比**：操作前拍快照 A → 重复操作 → 拍快照 B → **Comparison** 视图按 **Delta** 排序，看哪些对象持续增加
   - **查看 retainers**：搜索类名/对象 → 查看保留树，找到谁还在引用它；注意 `(closure)` 闭包引用
   - **Allocation timeline**：录制后查看哪些函数分配了大量对象且未被回收

3. **关键排查点**：

   | 排查对象 | 操作方法 |
   |---------|---------|
   | **Detached DOM 节点** | 快照中搜索 `Detached` |
   | **闭包引用** | 快照搜索 `(closure)` |
   | **事件监听器** | Elements → Event Listeners |
   | **定时器** | Sources 面板查看 |
   | **构造函数实例** | 快照搜索自定义类名 |

4. **典型排查流程**：
   ```
   Performance 确认曲线不回落
     → 快照 A → 操作 → 快照 B（Comparison 对比）
     → 搜索 Detached / 类名 / (closure)
     → 查看 Retainers 定位引用链 → 修复代码
   ```

