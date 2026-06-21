# process 全局对象

process 是 Node.js 的全局对象，无需 require/import，可直接在任意模块中访问。提供进程信息获取和进程控制能力。

## 进程信息

| API | 返回值 | 说明 |
|-----|--------|------|
| process.version | string | Node.js 版本号，如 v20.11.0 |
| process.platform | string | 操作系统名：win32 / darwin / linux |
| process.arch | string | CPU 架构：x64 / arm64 |
| process.pid | number | 当前进程 PID |
| process.cwd() | string | 当前工作目录路径 |
| process.memoryUsage() | object | 内存使用情况，含 rss / heapTotal / heapUsed 等字段，单位字节 |

```js
console.log('Node.js version:', process.version)
console.log('Platform:', process.platform)
console.log('Architecture:', process.arch)
console.log('PID:', process.pid)
console.log('Current directory:', process.cwd())
console.log('Memory usage:', process.memoryUsage())
```

## 命令行参数 process.argv

`process.argv` 返回数组：
- `argv[0]`: Node.js 可执行文件路径
- `argv[1]`: 当前脚本文件路径（REPL 中无此项）
- `argv[2+]`: 自定义参数

通常用 `process.argv.slice(2)` 剥离前两个固定元素获取纯用户参数。

```js
// node calc.js + 1 2
const ary = process.argv.slice(2)
function calc(operator, data1, data2) {
  if (operator === '+') return Number(data1) + Number(data2)
  if (operator === '-') return Number(data1) - Number(data2)
  if (operator === '*') return Number(data1) * Number(data2)
  return Number(data1) / Number(data2)
}
console.log(calc(...ary))  // calc('+', '1', '2') → 3
```

> argv 中的参数均为字符串类型，数值运算需用 Number() 转换。

## 环境变量 process.env

一个对象，存储当前 shell 的所有环境变量。常用于读取配置（如 NODE_ENV）。

```js
console.log('HOME:', process.env.HOME)
console.log('PATH:', process.env.PATH)
```

## 标准 I/O

- `process.stdout.write(text)`: 向标准输出流写入文本（不自动追加换行）
- `process.stdin.on('data', cb)`: 监听标准输入流的数据事件
- `process.stdin.once('data', cb)`: 只监听一次，适合单次交互

```js
process.stdout.write('请输入你的名字: ')
process.stdin.once('data', (data) => {
  console.log(`你好, ${data.toString().trim()}!`)
  process.exit(0)
})
```

> stdin 读取的 data 是 Buffer 类型，需用 `.toString().trim()` 转为字符串并去除换行。

## 进程控制

### process.exit([code])
终止当前进程，code 0 表示正常退出，1 表示异常退出。

### exit 事件
进程即将退出时触发，可在回调中执行清理工作。

```js
process.on('exit', (code) => {
  console.log(`进程即将退出，退出码: ${code}`)
})
```

> exit 回调中只能执行同步操作，异步操作不会被执行。

---

# fs 文件系统模块

fs 模块提供文件系统操作 API，有同步、回调异步、Promise 三种风格。官方推荐使用 `node:` 前缀引入内置模块：`import fs from 'node:fs'`，不加前缀也等价。

## 同步 API

直接返回结果，阻塞事件循环。适用于 CLI 脚本启动阶段或一次性初始化。

| 方法 | 说明 |
|------|------|
| readFileSync(path, encoding) | 同步读取文件内容 |
| existsSync(path) | 检查路径是否存在，返回 boolean |
| statSync(path) | 获取文件/目录状态信息，含 isDirectory() 等方法 |
| accessSync(path) | 检查路径可访问性，不存在则抛异常 |

```js
import { readFileSync, existsSync, statSync, accessSync } from 'node:fs'

const content = readFileSync('package.json', 'utf-8')
const exists = existsSync('/some/path')
const stats = statSync('/some/path')
stats.isDirectory()  // true / false
stats.isFile()       // true / false
```

### existsSync vs accessSync

| 维度 | existsSync | accessSync |
|------|------------|------------|
| 返回值 | boolean | void（不存在则 throw） |
| 错误处理 | 返回 false 不抛错 | 需 try/catch |
| 适用场景 | 纯存在性判断 | 需同时检查权限时 |

## 异步 API (fs/promises)

返回 Promise，推荐用于 I/O 密集型场景。

| 方法 | 说明 |
|------|------|
| readFile(path, encoding) | 读取文件内容 |
| writeFile(path, data, encoding) | 写入文件 |
| readdir(path, options) | 读取目录列表 |
| copyFile(src, dest) | 复制文件 |
| mkdir(path, options) | 创建目录 |

```js
import fs from 'node:fs/promises'

const content = await fs.readFile('file.md', 'utf-8')
await fs.writeFile('output.md', content, 'utf-8')
await fs.copyFile('source.txt', 'dest.txt')
await fs.mkdir('a/b/c', { recursive: true })
```

### readdir 与目录遍历

```js
const entries = await fs.readdir(docsDir, { withFileTypes: true })
for (const entry of entries) {
  if (entry.isDirectory()) {
    // 递归进入子目录
  } else if (entry.isFile() && entry.name.endsWith('.md')) {
    // 处理 .md 文件
  }
}
```

`{ withFileTypes: true }` 使返回的元素为 Dirent 对象，提供 `isDirectory()`、`isFile()` 等方法，避免额外调用 stat。

---

# path 路径模块

path 模块提供跨平台路径处理工具。Windows 路径分隔符为 `\`，POSIX 为 `/`，path 方法自动适配。

## 路径拼接

| 方法 | 说明 |
|------|------|
| path.join(...segments) | 用平台分隔符拼接路径片段 |
| path.resolve(...segments) | 解析为绝对路径 |

```js
const pkgPath = path.join(dirname, '..', 'package.json')
const absPath = path.resolve(docsDir)
```

`join` 是纯拼接，`resolve` 会从右向左解析 `..` / `.` 并生成绝对路径。

## 路径解析

| 方法 | 说明 |
|------|------|
| path.dirname(p) | 获取目录部分 |
| path.basename(p, [ext]) | 获取文件名，可选去除扩展名 |
| path.parse(p) | 解析为 { root, dir, base, ext, name } 对象 |

```js
path.basename('/foo/bar/file.txt')         // 'file.txt'
path.basename('/foo/bar/file.txt', '.md')  // 'file.txt'（不匹配不变）
path.dirname('/foo/bar/file.txt')          // '/foo/bar'
path.parse('/foo/bar/file.txt')
// { root: '/', dir: '/foo/bar', base: 'file.txt', ext: '.txt', name: 'file' }
```

## 路径关系

`path.relative(from, to)`: 计算从 from 到 to 的相对路径。

```js
path.relative('/a/b', '/a/b/c/d')   // 'c/d'
path.relative('/a/b', '/a/e/f')     // '../e/f'
```

---

# os 操作系统模块

os 模块提供操作系统层面的系统信息查询。

```js
import os from 'os'

os.networkInterfaces()  // 网络接口信息
os.hostname()           // 主机名
os.totalmem()           // 总内存（字节）
os.freemem()            // 空闲内存（字节）
os.cpus()               // CPU 信息
os.type()               // 操作系统名
os.release()            // 操作系统版本
```

---

# ES Modules (ESM)

Node.js 原生支持 ES Modules。在 package.json 中设置 `"type": "module"` 或在文件使用 `.mjs` 扩展名启用。

## import / export

```js
// 命名导出 — array.js
export function chunk(arr, size) { /* ... */ }
export function unique(arr) { /* ... */ }
export function shuffle(arr) { /* ... */ }

// 命名导入
import { chunk, unique, shuffle } from './array.js'

// 重导出 — index.js
export { chunk, unique, shuffle } from './array.js'
```

## import.meta

`import.meta.url` 返回当前模块的 file:// 协议路径，需要用 `fileURLToPath` 转换为普通路径。

```js
import { fileURLToPath } from 'url'
import { dirname } from 'path'

const __filename = fileURLToPath(import.meta.url)  // 当前文件绝对路径
const __dirname = dirname(__filename)               // 当前文件所在目录
```

## CJS vs ESM

| 维度 | CommonJS | ES Modules |
|------|----------|------------|
| 语法 | require / module.exports | import / export |
| 加载机制 | 运行时同步加载 | 编译时静态解析 |
| 文件扩展名 | .js（默认）/ .cjs | .mjs / .js（需 type: module） |
| 顶层 await | 不支持 | 支持 |
| 模块内 this | module.exports | undefined |
| __dirname / __filename | 直接可用 | 需 fileURLToPath 模拟 |

> 新项目优先使用 ESM。如需兼容 CJS 生态，可用 `.mjs` 或 `type: module` + dynamic import()。

