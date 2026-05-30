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

