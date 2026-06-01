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

### 1.1 ref 的内部分支逻辑

`ref()` 接收一个值后，会根据数据类型走两条不同的路径：

| 数据类型 | 实现方式 | 返回值 |
|---|---|---|
| 基本类型（number, string, boolean 等） | `class RefImpl` 内部用 getter/setter 持有 `_value` | 返回 `RefImpl` 实例，值存在 `.value` 属性上 |
| 引用类型（Object, Array 等） | 内部调用 `reactive()` → Proxy 代理 | 返回 `RefImpl` 实例，`.value` 指向 reactive 代理对象 |

**无论哪种路径，最终都返回一个 `{ value: xxx }` 形状的 wrapper 对象。** 所以读写时必须 `.value`——你操作的是这个 wrapper 的 `value` 属性，只有通过 getter/setter 才能触发依赖收集和派发更新。

### 1.2 reactive 的工作原理：Proxy 代理

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

每次对代理对象的属性进行**读取**时，拦截 `get` → 执行 `track()` 收集依赖；**写入**时，拦截 `set` → 执行 `trigger()` 通知更新。

### 1.3 为什么 Proxy 只能代理引用类型

- Proxy 的构造函数签名为 `new Proxy(target, handler)`，`target` **必须是对象**（Object / Array / Function 等）。
- 基本类型（如 `42`、`"hello"`）在 JS 中是**值类型**，没有引用地址，无法被 `Proxy` 包装拦截。
- 这就是 `ref()` 必须对基本类型单独处理的原因——自己用 getter/setter 模拟一个"可被追踪"的容器，把值包裹在 `.value` 里。

## 2. 示例代码

### ref 处理基本类型

```js
function ref(value) {
  return new RefImpl(value)
}

class RefImpl {
  _value   // 实际存储的值
  __v_isRef = true // 标记位，isRef() 用它判断

  constructor(value) {
    this._value = value
  }

  // 读取 .value 时收集依赖
  get value() {
    track(this, 'value')
    return this._value
  }

  // 写入 .value 时触发更新
  set value(newVal) {
    if (newVal !== this._value) {
      this._value = newVal
      trigger(this, 'value')
    }
  }
}
```

```js
const name = ref('张三')       // name → RefImpl { _value: '张三' }
console.log(name.value)       // '张三'  ← 走 get value() → track 收集依赖
name.value = '李四'            // 走 set value() → trigger 派发更新
```

### ref 处理引用类型（内部转 reactive）

```js
function ref(value) {
  if (isObject(value)) {
    // 引用类型：直接交给 reactive 做深层代理
    return new RefImpl(reactive(value))
  }
  // 基本类型：自己用 getter/setter 持有
  return new RefImpl(value)
}

class RefImpl {
  _value
  __v_isRef = true

  constructor(value) {
    // _value 可能是基本值，也可能是一个 reactive proxy
    this._value = value
  }

  get value() {
    track(this, 'value')
    return this._value
  }

  set value(newVal) {
    if (newVal !== this._value) {
      this._value = newVal
      trigger(this, 'value')
    }
  }
}
```

```js
const state = ref({ count: 0 })
// state._value → reactive({ count: 0 }) → Proxy 代理对象
// state.value → 走 get value() → track + 返回 Proxy 对象
// state.value.count = 1 → 触发 Proxy 的 set → trigger

state.value.count = 1     // ✅ 正确：先取 .value 拿到 proxy，再改 .count
// state.count = 1        // ❌ 错误：RefImpl 上没有 count 属性，改了也不会触发更新
```

### reactive 直接代理对象

```js
const state = reactive({ count: 0 })
// state → Proxy { get→track, set→trigger }
state.count = 1  // ✅ 直接读写，不需要 .value
```

## 3. 响应式完整流程：依赖收集 → 派发更新

整个响应式系统由三个核心函数驱动：

```
effect()       注册副作用函数（组件渲染函数就是一个 effect）
track()        收集依赖——记录"哪个 effect 依赖了哪个响应式数据的哪个属性"
trigger()      派发更新——数据变化时，找到所有依赖它的 effect 重新执行
```

### 3.1 核心数据结构

```js
// 全局 targetMap: WeakMap<target, Map<key, Set<effect>>>
// 结构示意：
//   targetMap = {
//     [rawObj] → Map {
//       'count' → Set { effect1, effect2 },
//       'name'  → Set { effect1 }
//     }
//   }
const targetMap = new WeakMap()
let activeEffect = null  // 当前正在执行的 effect
```

### 3.2 track —— 依赖收集

```js
function track(target, key) {
  if (!activeEffect) return  // 没有正在执行的 effect，不需要收集

  // 取出该 target 对应的 dep 映射表
  let depsMap = targetMap.get(target)
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map()))
  }

  // 取出该 key 对应的 effect 集合
  let deps = depsMap.get(key)
  if (!deps) {
    depsMap.set(key, (deps = new Set()))
  }

  // 把当前 activeEffect 加入依赖集合
  if (!deps.has(activeEffect)) {
    deps.add(activeEffect)
    // 同时让 effect 也记住自己依赖了哪些 dep，用于 cleanup
    activeEffect.deps.push(deps)
  }
}
```

### 3.3 trigger —— 派发更新

```js
function trigger(target, key) {
  const depsMap = targetMap.get(target)
  if (!depsMap) return

  const deps = depsMap.get(key)
  if (!deps) return

  // 用一个新的 Set 复制，避免在执行过程中 deps 被修改导致死循环
  const effectsToRun = new Set(deps)
  effectsToRun.forEach(effect => {
    effect()  // 重新执行副作用
  })
}
```

### 3.4 effect —— 注册副作用

```js
function effect(fn) {
  const effectFn = () => {
    // 1. 清理旧的依赖关系（每次重新执行前先解除上次收集的所有 dep）
    cleanup(effectFn)

    // 2. 设置为当前活跃 effect
    activeEffect = effectFn

    // 3. 执行传入的函数 → 过程中访问响应式数据 → 触发 track 收集依赖
    fn()

    // 4. 执行完毕，恢复
    activeEffect = null
  }

  effectFn.deps = []  // 记录它依赖了哪些 dep Set，用于 cleanup
  effectFn()           // 立即执行一次
}

function cleanup(effectFn) {
  // 从所有 dep Set 中移除当前 effect
  effectFn.deps.forEach(dep => dep.delete(effectFn))
  effectFn.deps.length = 0
}
```

### 3.5 完整流程示意

```
1. 组件初始化
   ↓
2. effect(componentRenderFn)  ← 把渲染函数注册为副作用
   ↓
3. 渲染函数执行，读取 ref.value / reactive.key
   ↓
4. 触发 getter → track(target, key) → 记录依赖关系
   ↓
5. 渲染完成，页面上显示数据
   ↓
6. 用户操作导致数据变更（如 ref.value = xxx）
   ↓
7. 触发 setter → trigger(target, key) → 找到所有依赖该 key 的 effect
   ↓
8. effect 重新执行 → 组件重新渲染 → 视图更新
```

## 4. 为什么 template 中 ref 不需要 .value

### 4.1 编译器的语法糖

Vue 的 SFC（单文件组件）在编译阶段，`<template>` 会被编译成 `render` 函数。编译器**自动识别 ref 变量**，在生成的代码中追加 `.value`。

### 4.2 编译前后对比

**模板代码：**

```html
<template>
  <div>{{ count }}</div>
  <button @click="increment">+1</button>
</template>

<script setup>
import { ref } from 'vue'

const count = ref(0)

function increment() {
  count.value++  // script 中必须写 .value
}
</script>
```

**编译后的 render 函数（简化）：**

```js
// 模板 <div>{{ count }}</div> 编译后大致等价于：
function render(ctx) {
  return h('div', ctx.count.value)  // ← 编译器自动加了 .value
}
```

### 4.3 关键结论

| 场景 | 是否需要 .value | 原因 |
|---|---|---|
| `<script setup>` 中读写 ref | **需要** | 你直接操作 JS 对象，拿到的就是 `RefImpl` 实例 |
| `<template>` 中引用 ref | **不需要** | 编译器在生成 render 函数时会自动解包，追加 `.value` |
| `<script setup>` 中解构 `reactive()` | **需要 `toRef()` / `toRefs()`** | 直接解构会把值拷贝出来，丢失响应式 |

编译器做的这层"自动 `.value`"本质上是一种**开发体验优化**——模板中频繁写 `.value` 会让代码很啰嗦，而编译阶段做这件事没有运行时成本。

---

# script+setup

## 1. 核心前提：模板本质是一个函数

- 模板会被编译成 `render(ctx)` 函数，渲染 UI 时只能通过 `ctx`（上下文）访问变量
- 这个 `ctx` 是什么，取决于你写的 script 形式：
  - 选项式 API → `ctx` 是**组件实例的 Proxy**
  - 组合式 API（普通 setup）→ `ctx` 是 `setup()` **函数的返回值**

## 2. 三种"暴露"机制

### 2.1 选项式 API（Vue 2 风格）

- 变量必须写在 `data() { return {...} }`
- 方法必须写在 `methods: {...}`
- 框架自动读取并挂载到组件实例上，模板通过 `this.xxx` 访问
- **暴露是隐式的**，由 Vue 内部统一处理

### 2.2 普通 `<script>` + 组合式 API

```js
import { ref } from 'vue'
import Foo from './Foo.vue'

export default {
  components: { Foo },        // 组件要手动注册
  setup(props) {
    const count = ref(0)
    const add = () => count.value++

    return { count, add, Foo } // ← 必须手动 return，模板才能用
  }
}
```

- `setup()` 本质是普通 JS 函数，函数内的局部变量函数外访问不到
- **必须手动 return** 告诉 Vue 哪些变量暴露给模板
- 组件要手动注册到 `components`

### 2.3 `<script setup>`（语法糖）

```vue
<script setup>
import { ref } from 'vue'
import Foo from './Foo.vue'    // 自动注册

const count = ref(0)
const add = () => count.value++
// 不写 return
</script>
```

- **编译器自动收集**顶层所有绑定（变量、函数、import 进来的组件），塞到 `__returned__` 对象里
- 编译后等价于手写版的 `return { count, add, Foo }`

## 3. `<script setup>` 编译后大致等价于

```js
import { ref } from 'vue'
import Foo from './Foo.vue'

export default {
  components: { Foo },   // 编译器从 import 中收集
  setup(__props) {
    const count = ref(0)
    const add = () => count.value++
    const __returned__ = { count, add, Foo, ref }  // 自动 return
    return __returned__
  }
}
```

## 4. 几个深层细节

### 4.1 为什么"自动"能成立

- `<script setup>` 整个文件的顶层作用域 = `setup` 函数体的作用域
- 编译器做一次 AST 扫描，把所有顶层声明收集到返回对象
- **纯静态分析**，无反射、无 Proxy 黑魔法

### 4.2 顶层 import 的组件自动注册

- 编译器把所有 `import X from './X.vue'` 收集起来
- 自动生成 `components: { X }`，省去手写注册

### 4.3 默认私有性

- `<script setup>` 所有顶层绑定**仅在当前组件模板和内部可见**
- 不会挂到组件实例上，父组件用 `ref` 拿不到
- 如需对外暴露，需用 `defineExpose({ count })` 显式声明

### 4.4 编译时宏

- `defineProps` / `defineEmits` / `defineModel` / `defineExpose` / `defineOptions` / `defineSlots`
- 看起来像函数，**但不会被 import，也没有运行时实现**
- 编译器识别这些调用，**替换成对应的运行时代码**（如 `props` 选项、`emits` 选项）
- 因此能"凭空"提供完整的 TypeScript 类型推导

---

# vue组件生命周期

## 1. 钩子列表与执行顺序

### 1.1 完整钩子列表

#### 组合式 API

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

#### 选项式 API 对照

| 选项式 | 对应组合式 |
|---|---|
| `beforeCreate` | ❌ 无（被 setup 取代） |
| `created` | ❌ 无（被 setup 取代） |
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

#### 关键注意点

- **没有 `onBeforeCreate` / `onCreated`** —— setup 函数本身就是 created 阶段
- `onMounted` / `onUpdated` 走 **post flush 队列**（微任务），不在 patch 同步流程里
- 调试钩子 `onRenderTracked` / `onRenderTriggered` 只在开发模式生效
- `onErrorCaptured` 返回 `false` 可以阻止错误继续向上传播

### 1.2 完整执行顺序（时间线）

```
setup()                    ← 同步执行，不是钩子，是初始化函数
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

### 1.3 关键规则

- **挂载周期只走一次**：首次 patch 后 `instance.isMounted = true`，之后所有 effect 重跑都走更新分支
- **更新周期可触发多次**：每次响应式数据变化都会重新 patch，触发 beforeUpdate / updated
- **销毁周期触发一次**：组件销毁时跑 beforeUnmount → unmounted
- **setup 不是生命周期钩子**，而是组件实例化时同步执行的初始化函数，介于 beforeCreate 和 created 之间（Vue 3 把这两个钩子删了，因为 setup 取代了它们）

## 2. 内部实现

### 2.1 本质

- 钩子 = 组件实例上的**回调数组**（`instance.mounted = []`、`instance.updated = []` 等）
- 顺序 = 渲染器 **patch 流程代码的物理书写顺序**（写死在哪一步调什么钩子）
- 注册 = `onMounted(fn)` → `instance.mounted.push(fn)`
- 触发 = 渲染器在流程的固定位置**直接遍历调用**：`arr.forEach(hook => hook())`

### 2.2 渲染器内部伪代码

```js
// 入口
function patch(vnode, container) {
  if (vnode.type.isComponent) {
    if (!vnode.component) {
      mountComponent(vnode)              // 首次挂载
    } else {
      updateComponent(vnode.component)   // 更新
    }
  }
}

// 挂载流程
function mountComponent(vnode) {
  const instance = createComponentInstance(vnode)   // 1. 创实例
  setupComponent(instance)                          // 2. 跑 setup
  setupRenderEffect(instance)                        // 3. 注册 effect
}

function setupRenderEffect(instance) {
  effect(() => {
    if (!instance.isMounted) {
      // ---------- 首次挂载分支 ----------
      callHook(instance.beforeMount)      // 4. 触发 onBeforeMount
      const subTree = instance.render()   // 5. 调 render 函数
      patch(subTree, container)           // 6. patch DOM
      instance.isMounted = true
      queuePostFlushCb(() => {
        callHook(instance.mounted)        // 7. 触发 onMounted（异步）
      })
    } else {
      // ---------- 更新分支 ----------
      callHook(instance.beforeUpdate)     // 8. 触发 onBeforeUpdate
      const nextTree = instance.render()
      patch(prevTree, nextTree, container) // 9. diff patch
      queuePostFlushCb(() => {
        callHook(instance.updated)        // 10. 触发 onUpdated（异步）
      })
    }
  })
}

// 卸载流程
function unmount(instance) {
  callHook(instance.beforeUnmount)        // 触发 onBeforeUnmount（同步）
  // ... 卸载 DOM
  callHook(instance.unmounted)            // 触发 onUnmounted（同步）
}
```

## 3. 任务类型

### 3.1 同步 vs 异步分类

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

### 3.2 异步钩子（mounted / updated）

#### 微任务实现

```js
const resolvedPromise = Promise.resolve()

function queuePostFlushCb(cb) {
  queue.push(cb)
  if (!currentFlushPromise) {
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}
```

post flush 队列内部用 `Promise.resolve().then(...)`，是**微任务**（不是 setTimeout 那种宏任务）。

#### 为什么要异步

1. **保证 DOM 已更新**——同步 patch 完 DOM 后，浏览器还没布局完成；推微任务能让用户访问 DOM 时拿到正确的尺寸、布局信息
2. **批量更新**——同一个 tick 内的多次数据变化合并成一次 patch，只触发一次 `onUpdated`

### 3.3 同步钩子

#### before* 钩子为什么同步

- `onBeforeMount`：必须在 patch **之前**触发，否则没意义（DOM 还没创建）
- `onBeforeUpdate`：必须在重新 patch **之前**触发，让你能拦截/做准备工作
- 同步调用能保证拿到当前最新的 props/state

#### unmount 钩子为什么同步

- 卸载是终态操作，没有"批量"或"等待布局"的需求
- 同步触发让 cleanup 逻辑（清理定时器、事件监听）立即执行
- 在 `v-if="false"` 场景下，组件马上要从 DOM 移除，同步触发语义更清晰

#### 其他同步钩子

- `onActivated` / `onDeactivated`：keep-alive 切换是直接操作，无异步需求
- `onErrorCaptured`：错误冒泡是同步机制
- `onRenderTracked` / `onRenderTriggered`：effect 追踪是同步流程



