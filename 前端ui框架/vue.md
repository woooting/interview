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

**1.1 ref 的内部分支逻辑**

`ref()` 接收一个值后，会根据数据类型走两条不同的路径：

| 数据类型 | 实现方式 | 返回值 |
|---|---|---|
| 基本类型（number, string, boolean 等） | `class RefImpl` 内部用 getter/setter 持有 `_value` | 返回 `RefImpl` 实例，值存在 `.value` 属性上 |
| 引用类型（Object, Array 等） | 内部调用 `reactive()` → Proxy 代理 | 返回 `RefImpl` 实例，`.value` 指向 reactive 代理对象 |

**无论哪种路径，最终都返回一个 `{ value: xxx }` 形状的 wrapper 对象。** 所以读写时必须 `.value`——你操作的是这个 wrapper 的 `value` 属性，只有通过 getter/setter 才能触发依赖收集和派发更新。

**1.2 reactive 的工作原理：Proxy 代理**

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

**1.3 为什么 Proxy 只能代理引用类型**

- Proxy 的构造函数签名为 `new Proxy(target, handler)`，`target` **必须是对象**（Object / Array / Function 等）。
- 基本类型（如 `42`、`"hello"`）在 JS 中是**值类型**，没有引用地址，无法被 `Proxy` 包装拦截。
- 这就是 `ref()` 必须对基本类型单独处理的原因——自己用 getter/setter 模拟一个"可被追踪"的容器，把值包裹在 `.value` 里。

## 2. 示例代码

**ref 处理基本类型**

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

**ref 处理引用类型（内部转 reactive）**

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

**reactive 直接代理对象**

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

**3.1 核心数据结构**

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

**3.2 track —— 依赖收集**

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

**3.3 trigger —— 派发更新**

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

**3.4 effect —— 注册副作用**

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

**3.5 完整流程示意**

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

**4.1 编译器的语法糖**

Vue 的 SFC（单文件组件）在编译阶段，`<template>` 会被编译成 `render` 函数。编译器**自动识别 ref 变量**，在生成的代码中追加 `.value`。

**4.2 编译前后对比**

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

**4.3 关键结论**

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

**2.1 选项式 API（Vue 2 风格）**

- 变量必须写在 `data() { return {...} }`
- 方法必须写在 `methods: {...}`
- 框架自动读取并挂载到组件实例上，模板通过 `this.xxx` 访问
- **暴露是隐式的**，由 Vue 内部统一处理

**2.2 普通 `<script>` + 组合式 API**

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

**2.3 `<script setup>`（语法糖）**

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

**4.1 为什么"自动"能成立**

- `<script setup>` 整个文件的顶层作用域 = `setup` 函数体的作用域
- 编译器做一次 AST 扫描，把所有顶层声明收集到返回对象
- **纯静态分析**，无反射、无 Proxy 黑魔法

**4.2 顶层 import 的组件自动注册**

- 编译器把所有 `import X from './X.vue'` 收集起来
- 自动生成 `components: { X }`，省去手写注册

**4.3 默认私有性**

- `<script setup>` 所有顶层绑定**仅在当前组件模板和内部可见**
- 不会挂到组件实例上，父组件用 `ref` 拿不到
- 如需对外暴露，需用 `defineExpose({ count })` 显式声明

**4.4 编译时宏**

- `defineProps` / `defineEmits` / `defineModel` / `defineExpose` / `defineOptions` / `defineSlots`
- 看起来像函数，**但不会被 import，也没有运行时实现**
- 编译器识别这些调用，**替换成对应的运行时代码**（如 `props` 选项、`emits` 选项）
- 因此能"凭空"提供完整的 TypeScript 类型推导

---

# vue组件生命周期

## 1. 钩子列表与执行顺序

**1.1 完整钩子列表**

**组合式 API**

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

**选项式 API 对照**

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

**关键注意点**

- **没有 `onBeforeCreate` / `onCreated`** —— setup 函数本身就是 created 阶段
- `onMounted` / `onUpdated` 走 **post flush 队列**（微任务），不在 patch 同步流程里
- 调试钩子 `onRenderTracked` / `onRenderTriggered` 只在开发模式生效
- `onErrorCaptured` 返回 `false` 可以阻止错误继续向上传播

**1.2 完整执行顺序（时间线）**

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

**1.3 关键规则**

- **挂载周期只走一次**：首次 patch 后 `instance.isMounted = true`，之后所有 effect 重跑都走更新分支
- **更新周期可触发多次**：每次响应式数据变化都会重新 patch，触发 beforeUpdate / updated
- **销毁周期触发一次**：组件销毁时跑 beforeUnmount → unmounted
- **setup 不是生命周期钩子**，而是组件实例化时同步执行的初始化函数，介于 beforeCreate 和 created 之间（Vue 3 把这两个钩子删了，因为 setup 取代了它们）

## 2. 内部实现

**2.1 本质**

- 钩子 = 组件实例上的**回调数组**（`instance.mounted = []`、`instance.updated = []` 等）
- 顺序 = 渲染器 **patch 流程代码的物理书写顺序**（写死在哪一步调什么钩子）
- 注册 = `onMounted(fn)` → `instance.mounted.push(fn)`
- 触发 = 渲染器在流程的固定位置**直接遍历调用**：`arr.forEach(hook => hook())`

**2.2 渲染器内部伪代码**

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

**3.1 同步 vs 异步分类**

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

**3.2 异步钩子（mounted / updated）**

**微任务实现**

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

**为什么要异步**

1. **保证 DOM 已更新**——同步 patch 完 DOM 后，浏览器还没布局完成；推微任务能让用户访问 DOM 时拿到正确的尺寸、布局信息
2. **批量更新**——同一个 tick 内的多次数据变化合并成一次 patch，只触发一次 `onUpdated`

**3.3 同步钩子**

**before\* 钩子为什么同步**

- `onBeforeMount`：必须在 patch **之前**触发，否则没意义（DOM 还没创建）
- `onBeforeUpdate`：必须在重新 patch **之前**触发，让你能拦截/做准备工作
- 同步调用能保证拿到当前最新的 props/state

**unmount 钩子为什么同步**

- 卸载是终态操作，没有"批量"或"等待布局"的需求
- 同步触发让 cleanup 逻辑（清理定时器、事件监听）立即执行
- 在 `v-if="false"` 场景下，组件马上要从 DOM 移除，同步触发语义更清晰

**其他同步钩子**

- `onActivated` / `onDeactivated`：keep-alive 切换是直接操作，无异步需求
- `onErrorCaptured`：错误冒泡是同步机制
- `onRenderTracked` / `onRenderTriggered`：effect 追踪是同步流程

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

---

# Effect 的本质与分类

## 响应式变量更新后重新执行的函数

在 Vue 响应式系统中统称为 **effect（响应式函数）**。从设计模式角度（观察者模式）：
- 响应式变量 = **发布者（Observable）**
- 重新执行的函数 = **订阅者（Observer）**

工作流程：
```
effect 执行 → 读取依赖 → track() 收集依赖
依赖更新 → trigger() → 通知所有订阅者 → effect 重新执行
```

## callback 与 effect 的关系

**不完全是**。callback 是普通函数，Vue 内部通过 `effect()` 包装后才成为响应式 effect。

```js
watchEffect(() => {
  console.log(count.value)
})
// callback 本身是普通函数
// Vue 内部用 effect() 包装，才具备依赖收集和触发更新的能力
```

## Vue effect 的异步特性

**不是**，需要区分：

| API | 本质 | 执行时机 | 是否入微任务队列 |
|-----|------|----------|-----------------|
| 底层 `effect`（`@vue/reactivity`） | 基础 effect | **同步立即执行** | 否 |
| `watch` / `watchEffect` | 封装的 effect | 默认异步（`pre`），可配 `sync` 同步 | 默认是 |
| `render` | 渲染副作用 | 异步批量更新 DOM | 是 |
| `computed` | **派生状态**（非副作用） | **懒执行**（读取时同步计算） | **否** |

底层 `effect` 没有 scheduler，依赖变化直接同步执行。`watch/watchEffect` 是在底层 `effect` 上注入了 scheduler 才实现异步批量。

---

# Computed 的本质

## computed 的副作用性质

**不是**。`computed` 是**派生状态（Derived State）**，不是副作用。

- **副作用**：与外部世界交互（修改 DOM、发请求、修改外部变量）
- **computed**：只读取依赖、计算并返回结果，是**纯函数**

```js
// 副作用
watchEffect(() => { document.title = count.value })

// 派生状态（纯函数，无副作用）
const double = computed(() => count.value * 2)
```

虽然 computed 的 getter 会重新执行，但它不修改任何外部状态，所以不是副作用。

## computed 内部的 effect 机制

**有**，但是 **lazy effect（懒执行 effect）**。

| API | effect 类型 | scheduler 行为 |
|-----|------------|---------------|
| `watchEffect` | 普通 effect | 放入微任务队列异步执行 |
| `watch` | 普通 effect | 放入微任务队列异步执行 |
| `computed` | **lazy effect** | 只标记 `dirty = true`，读取时才执行 |
| `render` | 普通 effect | 放入微任务队列异步执行 |

简化实现：
```js
function computed(getter) {
  let dirty = true
  let value
  
  const effect = reactiveEffect(getter, {
    lazy: true,
    scheduler() {
      dirty = true  // 依赖变化时，只标记 dirty
      triggerEffects(dep)  // 通知 computed 的订阅者
    }
  })
  
  return {
    get value() {
      trackEffect(dep)
      if (dirty) {
        value = effect()  // 读取时才执行
        dirty = false
      }
      return value
    }
  }
}
```

## computed 的返回值响应式特性

**是**。返回一个 **ref-like 响应式对象**，可以作为任何 effect 的依赖。

```js
const count = ref(1)
const double = computed(() => count.value * 2)
const quadruple = computed(() => double.value * 2) // computed 链式依赖

watchEffect(() => {
  console.log(quadruple.value)
})
```

## computed 的 track/trigger 实现机制

通过 **getter/setter 属性访问器**，和 ref 基本类型的包装方式相同：

```js
// computed 内部结构（简化版）
{
  _value: undefined,
  dirty: true,
  dep: new Set(),  // 存储订阅者
  get value() {
    trackEffect(this.dep)   // 被访问时收集依赖
    if (this.dirty) {
      this._value = this.effect()  // 懒计算
      this.dirty = false
    }
    return this._value
  }
}
```

完整流程：
```
1. watchEffect 执行 → 访问 double.value → track(double.dep)
2. double 的 effect 执行 → 读取 count.value → track(count.dep)
3. count.value = 2 → trigger(count.dep) → double 的 scheduler 执行
4. scheduler: dirty = true, trigger(double.dep) → watchEffect 重新执行
```

## computed 不采用 Proxy 包装返回值的原因

因为**不是它的职责**：

1. **职责不同**：computed 的职责是"依赖变化时重新计算并缓存"，不是"让返回值变成深度响应式"
2. **依赖方向不同**：computed 的依赖是**输入**（内部读取的响应式数据），不是**输出**（返回值）
3. **设计原则**：computed 的返回值应该是**只读**的派生状态，用户不应该直接修改它
4. **与 ref 的区别**：`ref(obj)` 用户会直接修改 `obj` 的属性，所以需要 Proxy；`computed` 用户不应该修改返回值，所以不需要

```js
// computed 的返回值本来就不应该被修改
const userInfo = computed(() => ({ name: '张三' }))
userInfo.value.name = '李四' // ❌ 这是反模式

// 如果需要深度响应式，自己用 reactive 包装
const userInfo = computed(() => reactive({ name: '张三' }))
```

---

# Proxy vs getter/setter（defineProperty）

## ref 包装引用类型使用 Proxy 的原因

`ref` 的 `.value` 读写始终用 getter/setter，但 `.value` 内部的对象用 `reactive()`（Proxy）包装。

## Object.defineProperty 的三个致命缺陷

| 特性 | defineProperty | Proxy |
|------|---------------|-------|
| 新增/删除属性 | ❌ 不支持 | ✅ 支持 |
| 数组索引/length | ❌ 支持很差 | ✅ 完美支持 |
| 初始化性能 | ❌ 需递归遍历所有层级 | ✅ 懒代理，按需代理 |
| 浏览器兼容性 | IE9+ | IE 不支持 |

**1. 无法拦截属性的新增和删除**

```js
// Object.defineProperty 只能拦截已存在的属性
const obj = { name: 'zs' }
obj.name = 'ls'  // ✅ 能拦截
obj.age = 18     // ❌ 新增属性，拦截不到！
delete obj.name  // ❌ 删除属性，拦截不到！

// Proxy 可以拦截所有操作
const proxy = new Proxy(obj, { ... })
proxy.age = 18     // ✅ 触发 set 拦截
delete proxy.name  // ✅ 触发 deleteProperty 拦截
```

**2. 无法优雅处理数组**

```js
const arr = [1, 2, 3]

// defineProperty 处理数组的痛点：
arr[0] = 99      // 能拦截，但需要遍历数组每个索引初始化，性能差
arr.length = 0   // ❌ 修改 length 拦截不到
arr.push(4)      // ❌ push 导致 length 变化，拦截不到

// Proxy 处理数组非常自然：
proxy[0] = 99    // ✅ 触发 set
proxy.length = 0 // ✅ 触发 set
proxy.push(4)    // ✅ 触发 set (length 和 索引)
```

**3. 性能问题：递归初始化 vs 懒代理**

```js
const deepObj = { a: { b: { c: 1 } } }

// defineProperty (Vue 2)：必须递归遍历所有层级，一次性全部代理
walk(deepObj) // 递归初始化 a, b, c 的 getter/setter

// Proxy (Vue 3)：懒代理，访问到哪一层才代理哪一层
const proxy = new Proxy(deepObj, {
  get(target, key) {
    const res = target[key]
    return isObject(res) ? reactive(res) : res  // 访问时才代理
  }
})
```

 这也是 Vue 2 被迫提供 `Vue.set` / `Vue.delete` 的原因，Vue 3 用 Proxy 彻底解决了这个问题。

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


