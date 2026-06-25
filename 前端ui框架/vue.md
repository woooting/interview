# 虚拟dom

1. **优点：**
  - 声明式编程的基础——你描述"UI 应该长这样"，框架自动算出要改哪些 DOM，不需要人肉维护数据→DOM 操作映射表
  - 跨平台——同一套 VNode 描述可以渲染到 DOM、Native、Canvas 等不同目标
  - 避免 layout thrashing——diff 纯读 JS 对象、patch 纯写 DOM，天然读写分离，不容易触发强制同步布局
2. **缺点：**
   - 单次操作的绝对性能比直接改 DOM 差——多了一层 VNode 创建 + diff 的开销
   - 内存开销——每个组件维护一套 JS 对象树
   - 需要额外的优化手段——key、shouldComponentUpdate、shallowRef 等，本质是为了弥补 diff 算法的不足
3. **为什么我们需要虚拟dom：**
   - 复杂页面中，数据变化后"**哪些 DOM 需要更新**"这件事，人工穷举分支的复杂度会爆炸。虚拟 DOM 用**运行时 diff** 把这层映射关系自动化了。它**不是性能最优解**，但它是组件化声明式架构下实现成本最低、适配范围最广的通用更新策略。

---

# 为什么读写ref代理的响应式变量时，需要带上.value

## 1. 响应式原理

1. **ref 的内部分支逻辑**

   `ref()` 接收一个值后，会根据数据类型走两条不同的路径：

   | 数据类型 | 实现方式 | 返回值 |
   |---|---|---|
   | 基本类型（number, string, boolean 等） | `class RefImpl` 内部用 getter/setter 持有 `_value` | 返回 `RefImpl` 实例，值存在 `.value` 属性上 |
   | 引用类型（Object, Array 等） | 内部调用 `reactive()` → Proxy 代理 | 返回 `RefImpl` 实例，`.value` 指向 reactive 代理对象 |

   无论哪种路径，最终都返回一个 `{ value: xxx }` 形状的 wrapper 对象，所以读写时必须 `.value`——你操作的是这个 wrapper 的 `value` 属性，只有通过 getter/setter 才能触发依赖收集和派发更新。

2. **reactive 的工作原理：Proxy 代理**

   `reactive()` 使用 ES6 的 `Proxy` 对目标对象进行全量代理：

   ```
   const raw = { count: 0 }
   const proxy = new Proxy(raw, {
     get(target, key, receiver) {
       track(target, key)          // 依赖收集
       return Reflect.get(target, key, receiver)
     },
     set(target, key, value, receiver) {
       const oldValue = target[key]
       const result = Reflect.set(target, key, value, receiver)
       if (oldValue !== value) {
         trigger(target, key)      // 派发更新
       }
       return result
     }
   })
   ```

3. **为什么 Proxy 只能代理引用类型**

   - Proxy 的构造函数签名为 `new Proxy(target, handler)`，`target` **必须是对象**
   - 基本类型在 JS 中是**值类型**，没有引用地址，无法被 `Proxy` 包装拦截
   - 这就是 `ref()` 必须对基本类型单独处理的原因——自己用 getter/setter 模拟一个"可被追踪"的容器

## 2. 示例代码

ref 处理基本类型：

```js
function ref(value) {
  return new RefImpl(value)
}
class RefImpl {
  _value
  __v_isRef = true
  constructor(value) { this._value = value }
  get value() { track(this, 'value'); return this._value }
  set value(newVal) {
    if (newVal !== this._value) { this._value = newVal; trigger(this, 'value') }
  }
}
const name = ref('张三')
console.log(name.value)   // get value() → track
name.value = '李四'        // set value() → trigger
```

ref 处理引用类型（内部转 reactive）：

```js
function ref(value) {
  return isObject(value) ? new RefImpl(reactive(value)) : new RefImpl(value)
}
class RefImpl {
  _value
  __v_isRef = true
  constructor(value) { this._value = value }
  get value() { track(this, 'value'); return this._value }
  set value(newVal) {
    if (newVal !== this._value) { this._value = newVal; trigger(this, 'value') }
  }
}
const state = ref({ count: 0 })
state.value.count = 1   // 先取 .value 拿 proxy，再改 .count
```

reactive 直接代理对象：

```js
const state = reactive({ count: 0 })
state.count = 1  // 直接读写，不需要 .value
```

## 3. 响应式完整流程：依赖收集 → 派发更新

整个响应式系统由三个核心函数驱动：

```
effect()       注册副作用函数（组件渲染函数就是一个 effect）
track()        收集依赖——记录"哪个 effect 依赖了哪个响应式数据的哪个属性"
trigger()      派发更新——数据变化时，找到所有依赖它的 effect 重新执行
```

1. **核心数据结构**

   ```js
   const targetMap = new WeakMap()  // WeakMap<target, Map<key, Set<effect>>>
   let activeEffect = null          // 当前正在执行的 effect
   ```

2. **track —— 依赖收集**

   ```js
   function track(target, key) {
     if (!activeEffect) return
     let depsMap = targetMap.get(target)
     if (!depsMap) targetMap.set(target, (depsMap = new Map()))
     let deps = depsMap.get(key)
     if (!deps) depsMap.set(key, (deps = new Set()))
     if (!deps.has(activeEffect)) {
       deps.add(activeEffect)
       activeEffect.deps.push(deps)
     }
   }
   ```

3. **trigger —— 派发更新**

   ```js
   function trigger(target, key) {
     const depsMap = targetMap.get(target)
     if (!depsMap) return
     const deps = depsMap.get(key)
     if (!deps) return
     const effectsToRun = new Set(deps)
     effectsToRun.forEach(effect => effect())
   }
   ```

4. **effect —— 注册副作用**

   ```js
   function effect(fn) {
     const effectFn = () => {
       cleanup(effectFn)
       activeEffect = effectFn
       fn()
       activeEffect = null
     }
     effectFn.deps = []
     effectFn()
   }
   function cleanup(effectFn) {
     effectFn.deps.forEach(dep => dep.delete(effectFn))
     effectFn.deps.length = 0
   }
   ```

5. **完整流程示意**

   ```
   effect(componentRenderFn) → 读取 ref/reactive
   → track() 收集依赖 → 数据变化 → trigger()
   → effect 重新执行 → 组件重新渲染 → 视图更新
   ```

## 4. 为什么 template 中 ref 不需要 .value

1. **编译器的语法糖**

   Vue 的 SFC 在编译阶段，`<template>` 会被编译成 `render` 函数。编译器**自动识别 ref 变量**，在生成的代码中追加 `.value`。

2. **编译前后对比**

   模板代码：
   ```html
   <template>
     <div>{{ count }}</div>
     <button @click="increment">+1</button>
   </template>
   <script setup>
   const count = ref(0)
   function increment() { count.value++ }
   </script>
   ```
   编译后 render 函数：
   ```js
   function render(ctx) {
     return h('div', ctx.count.value)  // ← 编译器自动加了 .value
   }
   ```

3. **关键结论**

   | 场景 | 是否需要 .value | 原因 |
   |---|---|---|
   | `<script setup>` 中读写 ref | **需要** | 直接操作 JS 对象，拿到的是 `RefImpl` 实例 |
   | `<template>` 中引用 ref | **不需要** | 编译器在 render 函数中自动追加 `.value` |
   | `<script setup>` 中解构 `reactive()` | **需要 `toRef()` / `toRefs()`** | 直接解构会把值拷贝出来，丢失响应式 |

   编译器做的"自动 `.value`"本质上是一种**开发体验优化**，编译阶段处理没有运行时成本。

---

# script+setup

## 1. 核心前提：模板本质是一个函数

- 模板会被编译成 `render(ctx)` 函数，渲染 UI 时只能通过 `ctx`（上下文）访问变量
- 这个 `ctx` 是什么，取决于你写的 script 形式：
  - 选项式 API → `ctx` 是**组件实例的 Proxy**
  - 组合式 API（普通 setup）→ `ctx` 是 `setup()` **函数的返回值**

## 2. 三种"暴露"机制

1. **选项式 API（Vue 2 风格）**
   - 变量写在 `data() { return {...} }`，方法写在 `methods: {...}`
   - 框架自动读取并挂载到组件实例上，模板通过 `this.xxx` 访问
   - 暴露是隐式的，由 Vue 内部统一处理

2. **普通 `<script>` + 组合式 API**
   ```js
   import { ref } from 'vue'
   import Foo from './Foo.vue'
   export default {
     components: { Foo },
     setup(props) {
       const count = ref(0)
       const add = () => count.value++
       return { count, add, Foo }  // ← 必须手动 return
     }
   }
   ```
   - `setup()` 内的变量函数外访问不到，**必须手动 return**
   - 组件要手动注册到 `components`

3. **`<script setup>`（语法糖）**
   ```vue
   <script setup>
   import { ref } from 'vue'
   import Foo from './Foo.vue'    // 自动注册
   const count = ref(0)
   const add = () => count.value++
   </script>
   ```
   - 编译器自动收集顶层绑定塞到 `__returned__` 对象里，等价于手写 `return { count, add, Foo }`

## 3. `<script setup>` 编译后大致等价于

```js
import { ref } from 'vue'
import Foo from './Foo.vue'
export default {
  components: { Foo },   // 编译器从 import 中收集
  setup(__props) {
    const count = ref(0)
    const add = () => count.value++
    const __returned__ = { count, add, Foo, ref }
    return __returned__
  }
}
```

## 4. 几个深层细节

1. **为什么"自动"能成立**
   - `<script setup>` 的顶层作用域等同 `setup` 函数体
   - 编译器做 AST 扫描收集所有顶层声明，**纯静态分析**

2. **顶层 import 的组件自动注册**
   - 编译器收集 `import X from './X.vue'`，自动生成 `components: { X }`

3. **默认私有性**
   - 绑定**仅在当前组件模板和内部可见**，不会挂到实例上
   - 如需对外暴露，用 `defineExpose({ count })` 显式声明

4. **编译时宏**
   - `defineProps` / `defineEmits` / `defineModel` / `defineExpose` / `defineOptions` / `defineSlots`
   - 看起来像函数但**不会被 import，也没有运行时实现**
   - 编译器识别这些调用并**替换成对应运行时代码**

---

# Vue 组件生命周期

## 1. 钩子列表与执行顺序

1. **完整钩子列表**

   组合式 API：

   | 钩子 | 触发时机 |
   |---|---|
   | `onBeforeMount` | DOM 挂载前 |
   | `onMounted` | DOM 挂载完成（post flush 队列） |
   | `onBeforeUpdate` | 响应式数据变化，DOM 重新 patch 前 |
   | `onUpdated` | DOM 重新 patch 完成（post flush 队列） |
   | `onBeforeUnmount` | 组件卸载前 |
   | `onUnmounted` | 组件卸载完成 |
   | `onActivated` | keep-alive 缓存的组件被激活 |
   | `onDeactivated` | keep-alive 缓存的组件被切走 |
   | `onErrorCaptured` | 捕获后代组件抛出的错误 |
   | `onRenderTracked` | 渲染时追踪到响应式依赖（调试用） |
   | `onRenderTriggered` | 响应式依赖触发重新渲染时（调试用） |

   选项式 API 对照：

   | 选项式 | 对应组合式 |
   |---|---|
   | `beforeCreate` | 无（被 setup 取代） |
   | `created` | 无（被 setup 取代） |
   | `beforeMount` | `onBeforeMount` |
   | `mounted` | `onMounted` |
   | `beforeUpdate` | `onBeforeUpdate` |
   | `updated` | `onUpdated` |
   | `beforeUnmount` | `onBeforeUnmount` |
   | `unmounted` | `onUnmounted` |
   | `activated` | `onActivated` |
   | `deactivated` | `onDeactivated` |
   | `errorCaptured` | `onErrorCaptured` |
   | `renderTracked` | `onRenderTracked` |
   | `renderTriggered` | `onRenderTriggered` |

   关键注意点：
   - 没有 `onBeforeCreate` / `onCreated`，setup 函数本身就是 created 阶段
   - `onMounted` / `onUpdated` 走 post flush 队列（微任务），不在 patch 同步流程里
   - 调试钩子 `onRenderTracked` / `onRenderTriggered` 只在开发模式生效
   - `onErrorCaptured` 返回 `false` 可以阻止错误继续向上传播

2. **完整执行顺序（时间线）**

   ```
   setup()                    ← 同步执行，初始化函数
      ↓
   onBeforeMount              ← 首次 patch DOM 前（同步）
      ↓
   （首次 patch，DOM 插入）
      ↓
   onMounted                  ← patch DOM 后（异步微任务）
      ↓
   （响应式数据变化）
      ↓
   onBeforeUpdate             ← 重新 patch 前（同步）
      ↓
   （重新 patch）
      ↓
   onUpdated                  ← 重新 patch 后（异步微任务）
      ↓  （重复 N 次）
   onBeforeUnmount            ← 卸载前（同步）
      ↓
   （DOM 移除）
      ↓
   onUnmounted                ← 卸载完成（同步）
   ```

3. **关键规则**

   - **挂载周期只走一次**：首次 patch 后 `instance.isMounted = true`，之后所有 effect 重跑都走更新分支
   - **更新周期可触发多次**：每次响应式数据变化都会重新 patch，触发 beforeUpdate / updated
   - **销毁周期触发一次**：组件销毁时跑 beforeUnmount → unmounted
   - **setup 不是生命周期钩子**，而是组件实例化时同步执行的初始化函数

## 2. 内部实现

1. **组件实例构造的函数调用链**
   - 组件从创建到挂载经过三个核心函数：
   ```
   mountComponent(vnode, container)
     ├── createComponentInstance(vnode)
     ├── setupComponent(instance)
     └── setupRenderEffect(instance)
   ```
   - `createComponentInstance`：分配空组件实例，`isMounted` 初始化为 `false`，钩子数组（`bm`、`m`、`bu`、`u`、`bum`、`um`）初始化为 `null`
   - `setupComponent`：执行 `setup()`，用户调用的 `onMounted(fn)` 等通过 `injectHook` 将 `fn` 推入对应钩子数组，此时仅注册不执行
   ```
   onMounted(fn)  →  injectHook('m', fn, currentInstance)  →  instance.m.push(fn)
   ```
   - `setupRenderEffect`：用 `new ReactiveEffect(componentUpdateFn)` 创建渲染副作用，数据变化时触发 `componentUpdateFn` 重新执行

2. **状态判断与钩子调用时机**
   - `componentUpdateFn` 以 `instance.isMounted` 作为唯一状态标识：

   | 状态 | 分支 | 执行顺序 |
   |------|------|----------|
   | `isMounted === false` | 首次挂载 | 遍历 `bm` 数组 → 执行 render → patch 递归挂载子组件 → 设 `isMounted = true` → `queuePostFlushCb(m)` |
   | `isMounted === true` | 更新 | 遍历 `bu` 数组 → 执行 render → patch 递归 diff 子组件 → `queuePostFlushCb(u)` |

   - **挂载钩子**：`beforeMount` 在 patch DOM 前同步调用。`mounted` 通过 `queuePostFlushCb` 放入微任务队列，确保子组件全部挂载完才执行
   - **更新钩子**：`beforeUpdate` 在重新 patch 前同步调用。`updated` 同样入微任务队列，保证 DOM 更新完毕
   - **卸载钩子**：`unmountComponent` 遍历 `bum` 数组 → 移除 DOM → 遍历 `um` 数组，前后同步调用

## 3. 任务类型

1. **同步 vs 异步分类**

   | 钩子 | 执行时机 | 同步/异步 |
   |---|---|---|
   | `onBeforeMount` | patch DOM **前** | **同步** |
   | `onMounted` | patch DOM **后** | 异步（post flush 队列，微任务） |
   | `onBeforeUpdate` | 重新 patch **前** | **同步** |
   | `onUpdated` | 重新 patch **后** | 异步（post flush 队列，微任务） |
   | `onBeforeUnmount` | 卸载 DOM **前** | **同步** |
   | `onUnmounted` | 卸载 DOM **后** | **同步** |
   | `onActivated` / `onDeactivated` | keep-alive 切换 | 同步 |
   | `onErrorCaptured` | 错误冒泡 | 同步 |
   | `onRenderTracked` / `onRenderTriggered` | effect 追踪/触发 | 同步 |

2. **异步钩子（mounted / updated）**

   post flush 队列内部用 `Promise.resolve().then(...)` 实现，是**微任务**：
   ```js
   const resolvedPromise = Promise.resolve()
   function queuePostFlushCb(cb) {
     queue.push(cb)
     if (!currentFlushPromise) {
       currentFlushPromise = resolvedPromise.then(flushJobs)
     }
   }
   ```
   异步原因：保证 DOM 已更新（浏览器布局完成）、同一 tick 内多次数据变化合并成一次 patch

3. **同步钩子**

      - **before 钩子**（beforeMount / beforeUpdate）：必须在 patch **之前**触发，同步调用能拿到最新 props/state
   - **unmount 钩子**（beforeUnmount / unmounted）：卸载是终态操作，同步触发让 cleanup 立即执行，无批量需求
   - **其他**（activated / deactivated / errorCaptured / renderTracked / renderTriggered）：均为同步机制，无异步需求

---

# watch 和 watchEffect 的区别

| 特性 | `watch` | `watchEffect` |
|------|---------|---------------|
| 数据源 | 显式指定（ref、reactive、getter） | 自动收集回调内用到的响应式依赖 |
| 执行时机 | 数据变化后执行（lazy，默认不立即执行） | 立即执行一次，之后依赖变化时执行 |
| 新旧值 | 可拿到 `newVal` / `oldVal` | 无参数，拿不到旧值 |
| 副作用清理 | 手动处理 | 回调接收 `onCleanup`，自动处理竞态 |
| 性能 | 精确控制，只监听指定源 | 自动追踪，可能收集多余依赖 |

## "显式指定"的含义

```js
// watch：手动写出监听源
watch(count, (newVal, oldVal) => { ... })
watch([count, name], ([newC, newN], [oldC, oldN]) => { ... })
watch(() => state.count, (newVal) => { ... })

// watchEffect：框架自动追踪
watchEffect(() => {
  console.log(state.count, state.name)  // 用了谁就监听谁
})
```

## 刷新时机（flush）

| flush | 执行时机 | 说明 |
|-------|----------|------|
| `pre`（默认） | 组件 DOM 更新**前** | 批量异步，多次变化只执行一次 |
| `post` | 组件 DOM 更新**后** | 可拿到更新后的 DOM |
| `sync` | 依赖变化时**立即同步** | 性能差，少用 |

默认异步批量执行，通过 scheduler 放入微任务队列，避免同步代码中多次修改导致重复执行。

## 场景选择

| 场景 | 选择 | 原因 |
|------|------|------|
| 需要根据新旧值做不同逻辑 | watch | watchEffect 拿不到旧值 |
| 监听多个数据源联动 | watch | 显式控制更清晰 |
| 数据变化后发请求，需要取消上一次 | watchEffect | `onCleanup` 自动处理竞态 |
| 组件初始化就要执行的副作用 | watchEffect | 默认立即执行 |
| 监听深层对象某个属性 | watch + getter | 精确控制，避免性能问题 |

## 实现原理：effect 包装差异

`ReactiveEffect` 在不同 API 中接收的第一个参数（即被追踪的 fn）不同：

| API | ReactiveEffect(fn, options) | 说明 |
|-----|---------------------------|------|
| watch | `ReactiveEffect(getter, scheduler)` | getter 是 `() => source.value`，用于追踪 source 的变化 |
| watchEffect | `ReactiveEffect(callback, scheduler)` | 开发者传入的 callback 直接作为 effect |
| computed | `ReactiveEffect(getter, { lazy, scheduler })` | 开发者传入的 getter 作为 effect，lazy 模式下不立即执行 |

**watch**：`ReactiveEffect` 包装的是内部的 source getter，用于追踪依赖变化。你传入的 callback 只在 scheduler 中被普通调用，**不被 ReactiveEffect 包装**，不参与依赖收集。这就是 callback 内部读写响应式数据也不会被自动追踪的原因。

**watchEffect**：`ReactiveEffect` 包装的就是开发者传入的 callback 本身。因此 callback 内部读到的每个响应式变量都会被 `track` 收集，实现自动依赖追踪。

**computed**：`ReactiveEffect` 包装的是开发者传入的 getter 计算函数。与 watch 和 watchEffect 不同，它带 `lazy: true` 标记，创建时不立即执行，依赖变化时 scheduler 只设 `dirty = true`，**只有读取 `.value` 时才重新执行 getter**。

---

# vue中的effect

## effect 的作用

**effect 是** Vue 响应式系统的核心执行单元，负责在依赖的响应式数据发生变化时自动重新执行。它是 **source（响应式数据）和 scheduler（反应行为）之间的桥梁**：

```
source 变化 → trigger() → effect 收到通知 → 调用自身的 scheduler
                                                    │
                          ┌──────────────────────────┐
                          │  scheduler 决定下一步做什么 │
                          ├ 无：同步执行 fn           │
                          ├ queueJob：入微任务队列渲染  │
                          ├ queueFlushCb：入微任务队列  │
                          └ 标记 dirty：读取时才计算   │
```

整个响应式更新的流程为：

- **track（依赖收集）**：effect 执行时读取响应式数据，触发 `track()` 将当前 effect 记录到该数据的依赖集合中
- **trigger（派发更新）**：响应式数据被修改时，触发 `trigger()` 从依赖集合中找到所有相关 effect，逐一通知执行

```
effect 首次执行 → 读取 ref/reactive → track() 收集依赖
数据变化 → trigger() → 通知 effect → 执行 scheduler 或 fn
```

## effect 的由来

effect 是 `new ReactiveEffect(fn, options?)` 创建的实例。Vue 内部不同场景传给 `ReactiveEffect` 的 fn 和 options 不同，从而产生不同类型的 effect：

| API | 传给 ReactiveEffect 的 fn | options | 行为 |
|-----|--------------------------|---------|------|
| 组件渲染 | 组件的 render 函数 | `{ scheduler: queueJob }` | 数据变化时重新执行 render 并 patch DOM，异步批量入渲染队列 |
| watch | source getter（`() => source.value`） | `{ scheduler: queueFlushCb }` | 只有 getter 被包装为 effect，开发者传入的 callback 不作为 effect |
| watchEffect | 开发者传入的整个 callback | `{ scheduler: queueFlushCb }` | callback 本身作为 effect，内部读到的响应式变量自动被追踪 |
| computed | 开发者传入的 getter 计算函数 | `{ lazy: true, scheduler: 标记 dirty }` | getter 作为 effect，懒执行，读取 `.value` 时才计算 |
| 裸 effect | 开发者自定义的函数 | 无 | 无 scheduler，无懒执行，变化即同步执行 |

**watch 与 watchEffect 的差异根源**：watch 把 getter 包装为 effect（用于追踪 source），callback 只是 scheduler 中的普通调用；watchEffect 直接把 callback 包装为 effect。这是 `watch` 需要显式声明 source、而 `watchEffect` 可以自动收集依赖的根本原因。

**computed 的特殊性**：computed 的 effect 带 `lazy: true`，创建时不立即执行，依赖变化时 scheduler 只设 `dirty = true` 并通知下游订阅者，**读取 `.value` 时才重新执行 getter**。这种机制称为 lazy effect。

---

# Proxy 与 getter/setter 及 Object.defineProperty

## Proxy 概述

**Proxy 是** ES6 提供的原生构造函数，用于对目标对象创建代理层，拦截并自定义对象的基本操作（属性读写、删除、函数调用等）。

```js
const target = { count: 0 }
const proxy = new Proxy(target, {
  get(target, key) { return target[key] },
  set(target, key, value) {
    target[key] = value
    // 触发更新逻辑
  }
})
```

Vue 3 选择 Proxy 替代 Vue 2 的 `Object.defineProperty` 作为响应式核心，原因在于 Proxy 能拦截**所有**对象操作（增删属性、数组索引/length 变更等），且支持懒代理——不需要在初始化时递归遍历所有嵌套对象，只在访问时才代理。

## Object.defineProperty 的缺陷

| 特性 | defineProperty | Proxy |
|------|---------------|-------|
| 新增/删除属性 | 不支持 | 支持 |
| 数组索引/length | 支持很差 | 完美支持 |
| 初始化性能 | 需递归遍历所有层级 | 懒代理，按需代理 |

1. **无法拦截属性的新增和删除**

   ```js
   const obj = { name: 'zs' }
   obj.name = 'ls'   // ✅ defineProperty 能拦截
   obj.age = 18      // ❌ 新增属性，拦截不到
   delete obj.name   // ❌ 删除属性，拦截不到
   const proxy = new Proxy(obj, { ... })
   proxy.age = 18    // ✅ 触发 set
   delete proxy.name // ✅ 触发 deleteProperty
   ```

2. **无法优雅处理数组**

   ```js
   const arr = [1, 2, 3]
   arr[0] = 99      // 能拦截但需每个索引初始化，性能差
   arr.length = 0   // ❌ 拦截不到
   arr.push(4)      // ❌ 拦截不到
   const proxy = new Proxy(arr, { ... })
   proxy[0] = 99    // ✅ 触发 set
   proxy.length = 0 // ✅ 触发 set
   ```

3. **性能问题：递归初始化 vs 懒代理**

   ```js
   const deepObj = { a: { b: { c: 1 } } }
   // defineProperty：需递归遍历所有层级，一次性全部代理
   walk(deepObj)
   // Proxy：访问到哪一层才代理哪一层
   const proxy = new Proxy(deepObj, {
     get(target, key) {
       return isObject(target[key]) ? reactive(target[key]) : target[key]
     }
   })
   ```

这也是 Vue 2 被迫提供 `Vue.set` / `Vue.delete` 的原因，Vue 3 用 Proxy 彻底解决了这个问题。

## ref 对不同数据类型的分类处理

**ref 返回一个 wrapper 对象**（一个带有 `.value` getter/setter 的普通对象），内部通过 `rawValue` 持有实际值，对 `.value` 进行读写拦截实现依赖收集和派发更新。

```js
function ref(value) {
  let rawValue =
    value !== null && typeof value === 'object' ? reactive(value) : value

  const wrapper = {   
    get value() {
      track(wrapper, 'value')
      return rawValue
    },
    set value(newVal) {
      if (rawValue === newVal) return
      rawValue =
        newVal !== null && typeof newVal === 'object'
          ? reactive(newVal)
          : newVal
      trigger(wrapper, 'value')
    },
  }

  return wrapper
}
```

**关键点：**

- **类型决定是否转为 Proxy**：初始化和赋值时都做类型判断——基本类型直接存 `rawValue`，引用类型先调用 `reactive()` 转为 Proxy 再存储
- **新旧值比较跳过无效更新**：`if (rawValue === newVal) return` 避免重复赋值触发不必要的更新
- **赋值时重新判断类型**：新值是引用类型则需重新 `reactive()` 包装，基本类型直接替换
- **两层拦截链**：引用类型场景下，`.value.count = 1` 先触发 `get value()`（track），再触发 Proxy 的 `set`（trigger）

这就是 ref 能统一处理所有数据类型的原理——用 getter/setter 解决了 Proxy 无法代理基本类型的问题，又通过内部调用 `reactive()` 弥补了 getter/setter 无法拦截深层属性的不足。

---

# Vue 组件通信方式

组件间的关系主要分为三种：**父→子、子→父、跨层级（兄弟/祖先/任意）**。不同的场景应选择不同的方案。

## 1. props / $emit

**场景：** 最基础、最常用的父子通信方式。父组件通过 props 向子组件传递数据，子组件通过 $emit 触发父组件的事件来通知变化。适合简单场景。

```vue
<!-- Parent.vue -->
<template>
  <Child :title="pageTitle" @update="handleUpdate" />
</template>

<script setup>
import { ref } from 'vue'
import Child from './Child.vue'

const pageTitle = ref('首页')

function handleUpdate(newTitle) {
  pageTitle.value = newTitle
}
</script>
```

```vue
<!-- Child.vue -->
<template>
  <div>
    <h1>{{ title }}</h1>
    <button @click="$emit('update', '新标题')">修改标题</button>
  </div>
</template>

<script setup>
defineProps({ title: String })
defineEmits(['update'])
</script>
```

**约束：** props 是单向数据流，子组件不应直接修改 props。需提交变化时通过 $emit 让父组件处理。

## 2. v-model

**场景：** 本质上是 props + $emit 的语法糖，适用于表单控件或需要双向绑定的自定义组件。Vue 3 支持多个 v-model 绑定。

```vue
<!-- Parent.vue -->
<template>
  <Child v-model:title="pageTitle" v-model:content="pageContent" />
</template>

<script setup>
import { ref } from 'vue'
const pageTitle = ref('首页')
const pageContent = ref('内容')
</script>
```

```vue
<!-- Child.vue -->
<template>
  <input :value="title" @input="$emit('update:title', $event.target.value)" />
</template>

<script setup>
defineProps({ title: String })
defineEmits(['update:title'])
</script>
```

Vue 3 用 `defineModel` 进一步简化：

```vue
<!-- Child.vue (Vue 3.4+) -->
<script setup>
const model = defineModel()
</script>

<template>
  <input v-model="model" />
</template>
```

## 3. $refs

**场景：** 父组件需要直接调用子组件的方法或访问子组件的属性。适用于"父组件主动触发的操作"，比如手动聚焦输入框、调用子组件的重置方法。缺点是耦合度高。

```vue
<!-- Parent.vue -->
<template>
  <Child ref="childRef" />
  <button @click="focusInput">聚焦子组件输入框</button>
</template>

<script setup>
import { ref } from 'vue'
import Child from './Child.vue'

const childRef = ref()

function focusInput() {
  childRef.value.focus()
}
</script>
```

```vue
<!-- Child.vue -->
<template>
  <input ref="inputRef" type="text" />
</template>

<script setup>
import { ref } from 'vue'

const inputRef = ref()

function focus() {
  inputRef.value.focus()
}

defineExpose({ focus })
</script>
```

**注意：** `<script setup>` 中的变量默认不暴露给父组件，必须用 `defineExpose` 显式声明。

## 4. provide / inject

**场景：** 祖先向所有后代传递**不常变化**的依赖，如主题配置、国际化语言包、API 基地址、权限设置等"配置型"数据。

```vue
<!-- App.vue -->
<script setup>
import { provide } from 'vue'

const i18n = { hello: '你好', world: '世界' }
const theme = { primary: '#1890ff', bg: '#fff' }

provide('i18n', i18n)
provide('theme', theme)
</script>
```

```vue
<!-- DeepChild.vue -->
<script setup>
import { inject } from 'vue'

const i18n = inject('i18n')
</script>

<template>
  <div>{{ i18n.hello }}</div>
</template>
```

**设计定位：** provide/inject 的初衷是传递**稳定的配置/依赖**，避免逐层 props 透传。它不适合做频繁变化的跨组件状态共享，原因有二：

- **依赖关系隐式**：代码中追溯不出"谁 inject 了它"、"谁修改了它"，调试困难
- **缺少规范更新路径**：不像 Pinia 有明确的 action/mutation 和 devtools

> 如果数据需要被多个组件频繁读写，用 **Pinia** 或 **父组件中转**。不要把 provide 当全局 store 用。

## 5. EventBus（事件总线）

**场景：** 任意两个组件之间的通信，无需组件层级关系。适合简单项目中的跨组件事件通知，如"某个操作完成后通知其他组件刷新"。

Vue 2 常用 `new Vue()` 实现：

```js
// eventBus.js (Vue 2)
import Vue from 'vue'
export const bus = new Vue()
```

```vue
<!-- ComponentA.vue -->
<script>
import { bus } from './eventBus'
export default {
  methods: {
    notify() {
      bus.$emit('refresh', { id: 1 })
    }
  }
}
</script>
```

```vue
<!-- ComponentB.vue -->
<script>
import { bus } from './eventBus'
export default {
  created() {
    bus.$on('refresh', (data) => {
      console.log('收到刷新通知', data)
    })
  },
  beforeDestroy() {
    bus.$off('refresh')  // 必须清理，防止内存泄漏
  }
}
</script>
```

Vue 3 不再提供实例的 `$on` / `$off`，推荐使用第三方库（如 `mitt`）：

```js
// eventBus.js (Vue 3 + mitt)
import mitt from 'mitt'
export const bus = mitt()
```

```vue
<script setup>
import { onUnmounted } from 'vue'
import { bus } from './eventBus'

bus.on('refresh', (data) => { /* ... */ })
onUnmounted(() => bus.off('refresh'))
</script>
```

**缺点：** 事件满天飞时难以维护，不明确谁触发谁监听，大型项目不推荐。

## 6. Vuex / Pinia（状态管理）

**场景：** 复杂的状态共享，如用户登录信息、购物车、全局配置等。适合中大型项目，有规范的更新流程和开发者工具支持。

**Pinia（Vue 3 推荐）**

```js
// stores/user.js
import { defineStore } from 'pinia'
import { ref } from 'vue'

export const useUserStore = defineStore('user', () => {
  const token = ref('')
  const userInfo = ref(null)

  function login(credentials) {
    // 调用登录 API
    token.value = 'xxx'
    userInfo.value = { name: '张三' }
  }

  function logout() {
    token.value = ''
    userInfo.value = null
  }

  return { token, userInfo, login, logout }
})
```

```vue
<!-- AnyComponent.vue -->
<script setup>
import { useUserStore } from '@/stores/user'

const userStore = useUserStore()

function handleLogin() {
  userStore.login({ username: 'admin', password: '123' })
}
</script>

<template>
  <div>{{ userStore.userInfo?.name }}</div>
</template>
```

**Vuex（Vue 2 / 旧项目）**

```js
// store/index.js
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

export default new Vuex.Store({
  state: { count: 0 },
  mutations: {
    increment(state) { state.count++ }
  },
  actions: {
    incrementAsync({ commit }) {
      setTimeout(() => commit('increment'), 1000)
    }
  }
})
```

```vue
<!-- AnyComponent.vue -->
<script>
export default {
  computed: {
    count() { return this.$store.state.count }
  },
  methods: {
    add() { this.$store.commit('increment') }
  }
}
</script>
```

**选择：** 新项目用 Pinia（更轻量、完整 TS 支持、无 mutation），老项目保持 Vuex。

## 7. $attrs

**场景：** 组件封装时，将父组件传入但未声明的属性和事件自动传递到子组件。常用于高阶组件、UI 库二次封装。

```vue
<!-- BaseInput.vue -->
<template>
  <input v-bind="$attrs" />
</template>

<script setup>
// 不声明 props，所有传入属性都会在 $attrs 中
</script>
```

```vue
<!-- Parent.vue -->
<template>
  <BaseInput type="text" placeholder="请输入姓名" maxlength="10" />
</template>
```

Vue 3 中 `$attrs` 包含 class、style、事件监听器等所有外传属性。配合 `inheritAttrs: false` 可控制标签继承行为。

## 8. slot / 作用域插槽

**场景：** 父组件向子组件传递模板内容。适合布局组件、列表项自定义、弹窗内容分发等场景。

```vue
<!-- Card.vue -->
<template>
  <div class="card">
    <header><slot name="header" /></header>
    <main><slot /></main>
    <footer><slot name="footer" /></footer>
  </div>
</template>
```

```vue
<!-- Parent.vue -->
<template>
  <Card>
    <template #header>标题</template>
    <p>默认插槽内容</p>
    <template #footer>底部信息</template>
  </Card>
</template>
```

**作用域插槽：** 子组件向父组件传递数据，让父组件决定渲染方式。

```vue
<!-- List.vue -->
<template>
  <ul>
    <li v-for="item in items" :key="item.id">
      <slot :item="item" :index="index" />
    </li>
  </ul>
</template>

<script setup>
defineProps({ items: Array })
</script>
```

```vue
<!-- Parent.vue -->
<template>
  <List :items="list">
    <template #default="{ item, index }">
      <span>{{ index + 1 }}. {{ item.name }}</span>
    </template>
  </List>
</template>
```

## 9. $parent / $children

**场景：** 直接访问父组件或子组件的实例。Vue 3 移除了 `$children`，Vue 2 可用但耦合度高，不推荐。

```vue
<!-- Vue 2 -->
<script>
export default {
  mounted() {
    this.$parent.someMethod()    // 访问父组件
    this.$children[0].someMethod() // 访问第一个子组件（Vue 3 不可用）
  }
}
</script>
```

Vue 3 中如需访问子组件实例，用 `$refs` 替代。

## 10. 全局挂载

**场景：** 将全局配置、工具函数等挂载到 Vue 实例上，所有组件通过 `this.xxx` 或 `xxx` 直接访问。

```js
// Vue 2
Vue.prototype.$api = apiService
Vue.prototype.$utils = utils

// Vue 3
const app = createApp(App)
app.config.globalProperties.$api = apiService
app.config.globalProperties.$utils = utils
```

**注意：** 全局挂载污染全局命名空间，仅适合真正全局不变的内容（如 axios 实例、工具函数），业务数据慎用。

---

## 选型建议

| 组件关系 | 推荐方案 |
|---------|---------|
| 父→子 传递数据 | props |
| 子→父 通知变化 | $emit |
| 父子双向绑定 | v-model |
| 父调用子方法 | $refs + defineExpose |
| 祖先→后代 跨多层 | provide / inject |
| 任意组件 简单通知 | EventBus（mitt） |
| 任意组件 复杂状态 | Pinia / Vuex |
| 组件封装 透传属性 | $attrs |
| 组件模板自定义 | slot / 作用域插槽 |

> 优先用 props / emit，保持数据流清晰。项目复杂度提升后再引入 Pinia，不要一开始就用全局状态管理。

