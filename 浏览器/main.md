# 浏览器强缓存和协商缓存

缓存和协商缓存都是浏览器的缓存技术，目的是为了减少对服务器的资源请求，从浏览器本地拿数据，减少服务器压力，提升访问速度，但是强缓存和协商缓存又有不同。

## 强缓存

强缓存通过请求头中设置的Expires和Cache-Control来控制，其中Expires是http1.0时的协商缓存请求头，Cache-Control是http1.1推出的请求头，现代浏览器大部分情况下使用Cache-Control来控制强缓存。

**Expires**

Expires是通过设置绝对时间来控制强缓存的时间的，如Expires: Wed, 21 Oct 2025 07:28:00 GMT。这就导致用户可以通过修改系统时间来导致强缓存失效，所以Cache-Control出现之后Expires就基本不使用了。

**Cache-Control**

Cache-Control的主要属性值包括：

- **private**：响应仅能被客户端缓存，不能被代理服务器缓存
- **public**：响应可以被客户端和代理服务器缓存
- **max-age**：设置缓存的最大有效期（秒）
- **no-cache**：**每次使用缓存前必须向服务器验证**（启用协商缓存）
- **no-store**：禁止任何形式的缓存
- **must-revalidate**：缓存过期后必须向服务器验证
- **immutable**：资源在有效期内不会改变（适合版本化资源）

**缓存位置（内存 vs 磁盘）**

当要请求一个资源时，如果启用了强缓存，第一次缓存会请求服务器然后会将资源缓存到本地内存或磁盘，如果缓存没有过期的情况下再次请求就会直接从内存或磁盘拿数据，不会请求服务器，大幅减少了服务器压力，也提高了前端的响应速度。

那么什么时候存到内存，什么时候存到磁盘呢？一般来说体积小、多次复用、过期时间短的会存在内存，反之存到磁盘，这时由浏览器决定的，是一个黑盒的过程，但是我们可以通过某种手段去影响浏览器的判断，比如将过期时间设置的短一点，就越有可能存到内存中。

**优缺点**

优点是设置了过期时间之后不会请求服务器，减少了服务器的压力和宽带开销；缺点是过期时间之前缓存的数据是不会更新的，如果服务器的数据更新了，但是本地缓存还没有过期，就会导致数据不同步的问题，所以，一般来讲，强缓存要存储不常改变的资源，如vue.js

## 协商缓存

协商缓存通过服务端返回的响应头 `Last-Modified`/`ETag`，与浏览器后续请求自动携带的请求头 `If-Modified-Since`/`If-None-Match` 配合控制。其中 `Last-Modified`/`If-Modified-Since` 是 HTTP/1.0 引入的机制，`ETag`/`If-None-Match` 是 HTTP/1.1 推出的新方案（更精确），现代浏览器大部分情况下优先使用 ETag。

**Last-Modified / If-Modified-Since**

Last-Modified（If-Modified-Since）是通过判断文件的修改时间来确定是否使用缓存资源的，不论是否真正的修改内容。这就导致某些情况下浪费宽带

**ETag / If-None-Match**

ETag （If-None-Match）是通过判断文件的内容是否修改来确定是否使用缓存的，相比于Last-Modified（If-Modified-Since）更加精细实用。具体的过程是，当启用Etag协商缓存时，第一次请求并不会携带请求验证，服务器在返回资源时会在响应头中返回一个Etag，这个值是服务器根据资源内容生成的唯一标识符，当下次请求时就会在请求头带入If-None-Match，值就是第一次请求时返回的Etag值。请求过去的时候服务器会比较If-None-Match和新的Etag值是否一样，如果一样就返回**304 Not Modified**不返回资源，意思是通知前端资源没有变可以从浏览器拿取缓存，如果值不一样，说明资源改变，这时就返回200状态码并返回新的资源

**优缺点**

优势是可以实时的更新最新数据不害怕资源过期，但是比起强缓存，协商缓存多了网络请求的往返，这样会导致网络延迟的发生。所以协商缓存适合动态内容、API响应、可能频繁更新的资源（如未版本化的HTML）

---

# 浏览器存储

## 存储方式

**localStorage**

便于客户端储存数据，在**本地保存**(需要手动删除)，可储存大小5MB以上，以键值对(Key-Value)的方式存储，多页面共享

**sessionStorage**

不在不同的浏览器页面中共享，即使是同一个页面，数据在当前浏览器关闭后自动删除

**cookie**

通过请求头发送给服务器，最大4kb，每次都会携带。

**三者都只能存储字符串键值对，如果需要存储对象/数组，需要进行json序列化写入**

## 应用场景

**localStorage**

- **用户设置**：主题（深色/浅色）、语言偏好、字体大小
- **不常变动的静态数据**：省市区数据、配置信息
- **PWA离线缓存**：离线可访问的数据
- **最近浏览记录**：不敏感的浏览历史
- **新手引导标记**：是否已展示过引导

**sessionStorage**

- **表单草稿**：多步骤表单的临时数据
- **页面内状态**：当前页面的滚动位置、选项卡索引
- **单次访问的数据**：用户本次操作的中间状态
- **刷页面保留数据**：防止刷新丢失的临时数据
- **多标签页隔离的数据**：不同标签页需要独立的状态

**cookie**

- **会话标识**：Session ID（配合 HttpOnly）
- **用户认证**：Token 存储（需配合 Secure、SameSite）
- **服务端读取的数据**：需要发送到服务器的信息
- **广告追踪**：用户行为追踪（第三方Cookie）
- **CSRF 防护**：CSRF Token 存储
- **子域名共享数据**：通过 Domain 参数实现

## 安全注意事项

需要注意的是，不是什么数据都适合放在 Cookie、localStorage 和 sessionStorage 中的，因为它们保存在本地容易被篡改，使用它们的时候，需要时刻注意是否有代码存在 XSS 注入的风险。所以千万不要用它们存储你系统中的敏感数据。

---

# 跨标签页通信

## BroadcastChannel

**原理**

可以视作**发布-订阅**的实现，**遵循同源协议**

**优势**

- API 极其简单，几行代码即可实现
- 性能最优，延迟最低
- 自动处理页面关闭，无需手动清理
- 不依赖存储空间

**劣势**

- 不支持点对点通信（只有广播）
- 无法持久化消息
- 较新的 API，IE 不支持

**适用场景**

- 简单的广播通知
- 实时状态同步
- 现代浏览器项目

**示例代码**

```js
// 标签页 A (发送方)
const channel = new BroadcastChannel('app_updates');
channel.postMessage({ type: 'user_login', username: 'Alice' });

// 标签页 B (接收方)
const channel = new BroadcastChannel('app_updates');
channel.onmessage = (event) => {
  if (event.data.type === 'user_login') {
    console.log('用户登录:', event.data.username);
  }
};
// 页面卸载时，请记得调用 channel.close() 释放资源
```

## window.postMessage

**原理**

HTML5 提供的跨窗口消息传递 API，主要用于父窗口与 iframe 之间通信。

**优势**

- 可以跨域通信
- 性能优秀
- 安全（可验证来源）
- 支持双向通信

**劣势**

- 只能在有直接引用关系的窗口间使用
- 需要明确的窗口引用
- 不适合多标签页通信

**适用场景**

- 父页面与 iframe 通信
- 跨域嵌入式应用
- 第三方组件集成

**示例代码**

```js
// 父窗口
const child = window.open('child.html');
child.postMessage('乖儿子', 'https://same-origin.com');

// 子窗口
window.opener.postMessage('老爸好！', 'https://same-origin.com');

// 两边都要监听
window.addEventListener('message', (event) => {
  if (event.origin !== 'https://same-origin.com') return;
  console.log('收到:', event.data);
});
```

## localStorage + storage 事件

**原理**

利用 localStorage 的 storage 事件，当一个页面修改 localStorage 时，同源的其他页面会触发 storage 事件。

**优势**

- 兼容性最好（IE8+）
- 实现简单
- 数据持久化
- 不需要服务器

**劣势**

- 有存储空间限制（5-10MB）
- 延迟较高（10-50ms）
- 修改页面本身不触发 storage 事件
- 同步操作，可能阻塞主线程

**适用场景**

- 需要兼容旧浏览器
- 简单的状态同步
- 需要数据持久化

**示例代码**

```js
// 标签页 A (修改存储)
const message = { action: 'refresh_data', timestamp: Date.now() };
//localstorage只能存字符串，所以需要json序列化引用类型数据
localStorage.setItem('cross_tab_msg', JSON.stringify(message));

// 标签页 B (接收通知)
window.addEventListener('storage', (event) => {
  if (event.key === 'cross_tab_msg' && event.newValue) {
    try {
      const data = JSON.parse(event.newValue);
      console.log('收到通知:', data);
    } catch (e) { /* 处理解析错误 */ }
  }
});
```

## SharedWorker

**原理**

多个页面共享的 Worker 线程，通过 MessagePort 进行双向通信。

**优势**

- 性能优秀，延迟极低
- 支持点对点和广播
- 不需要 HTTPS
- 资源共享（减少重复计算）

**劣势**

- Safari 需要手动开启实验性功能
- 所有页面关闭后 Worker 终止
- 调试相对困难

**适用场景**

- 需要共享计算资源
- 复杂的页面间通信
- 不需要持久化运行

**示例代码**

```js
// shared-worker.js
const ports = []; // 连接的所有标签页

onconnect = (e) => {
  const port = e.ports[0];
  ports.push(port);
  
  port.onmessage = (event) => {
    // 广播给其他同事
    ports.forEach(p => p !== port && p.postMessage(event.data));
  };
};

// 标签页代码
const worker = new SharedWorker('shared-worker.js');
worker.port.onmessage = (event) => {
  console.log('办公室通知:', event.data);
};
worker.port.postMessage('大家好呀！');
```

## Service Worker

**原理**

运行在浏览器后台的独立脚本，不依赖页面生命周期，可以在页面关闭后继续运行。

**优势**

- 持久化运行，页面关闭后仍可工作
- 支持离线缓存（PWA 核心技术）
- 可以拦截网络请求
- 支持推送通知

**劣势**

- 必须使用 HTTPS（开发环境可用 localhost）
- 首次注册需要刷新页面
- 实现复杂度较高
- 调试相对困难

**适用场景**

- PWA 应用
- 需要离线功能
- 复杂的跨页面通信和资源管理

**示例代码**

```js
// service-worker.js
self.addEventListener('message', (event) => {
  // 告诉所有标签页
  self.clients.matchAll().then(clients => {
    clients.forEach(client => client.postMessage(event.data));
  });
});

// 标签页代码
navigator.serviceWorker.onmessage = (event) => {
  console.log('邮差送来消息:', event.data);
};

// 发送消息
navigator.serviceWorker.controller.postMessage('快递到啦！');
```

**文章参考链接** [通信方案大全及其应用对比](https://juejin.cn/post/7593252939709677604)

---

# 浏览器渲染路径

![关键渲染路径](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/1/7/16827ff376991063~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)

## 解析 HTML → 构建 DOM 树

- 词法分析：是将输入的字符，分解成一个个有效token，这个过程也被称为Tokenization，这里的token不是令牌的意思，而是标记的意思，包括开始符号如\<body\>，结束符号\</body\>，属性名称，属性值等
- 语法分析：将标记按照HTML语言的语法规则分析文档的结构，从而构建解析树，这里的解析树就是DOM tree
- 遇到 JS 可能会导致HTML解析暂停，而先下载并执行script；
- JavaScript可能会改变HTML文档的结构，从而导致DOM tree发生改变，所以必须等JavaScript执行完成之后才能继续对HTML文档流的解析；
- 浏览器会采用预加载扫描器的功能，就是在解析HTML的同时，会提前搜索外部引用的内容，并提前下载，这样当解析器解析到他们的位置时，资源已经准备好了；
- 开发人员可以在脚本标签上增加async 或 defer 属性，告知渲染进程这个脚本可以在解析完成后再执行，这样可以提升页面渲染性能

## 解析 CSS → 构建 CSSOM 树

- 解析和构造CSSOM与解析DOM是可以并行的；
- 但下载和解析CSSOM会阻塞JavaScript的执行，因为JavaScript经常会用于查询元素的CSS样式信息。
- css资源常常放在body开头，css文件的下载和执行解析不会阻塞DOM树构建，所以我们可以看到带样式的html，如果放在body末尾，会出现无样式闪烁，显示默认html结构，然后闪烁变成携带了样式的html

## 合并 → 生成 Render Tree

- 这个渲染树上的节点只包括可见节点visible，那些不会出现在页面上的节点，比如\<html\>,\<body\>，不会出现在渲染树上；
- 渲染进程会遍历DOM上的每一个节点，再将可见的节点对应的属性匹配到这个节点上；
- 要注意，渲染树上的节点和DOM上的节点并不是一一对应的，有一些DOM节点可能对应多个渲染树节点，比如多行文本；
- 生成的渲染树，会用来供下一步计算每个节点的具体布局位置

## 布局 (Layout)

- 在创建完成渲染树后，这里面并没有节点的位置和大小信息，计算位置信息的过程称为布局，也可以叫重排(reflow)。
- 计算布局的方式是盒子模型（box model)，浏览器把页面对象都看作一个个盒子，并计算他们的尺寸，计算的依据是屏幕的尺寸；

## 绘制 (Paint)

- 绘制也可以说是一个光栅化(rasterizing)的过程，将布局最终转化为屏幕上的像素
- 光栅化即将屏幕分块，确定每个块中的元素具体样式；
- 在绘制的过程中，还会将layout分解成多个层layer，以此来决定绘制的顺序；
- 光栅化线程会光栅化每一层，然后将信息储存在GPU的内存中

## 合成 (Composite)

- 合成线程是将上一步光栅化的层，按照不同优先级和清晰度，进行组合，构建一个合成帧（compositor frame）
- 优先级是根据浏览器的窗口位置，在展示窗口内的，或者靠近展示窗口的，会被优先合成；
- 上面的步骤完成之后，渲染进程就会通过IPC向浏览器进程（browser process）提交（commit）一个合成帧compositor frame；
- 合成帧都会被发送给GPU从而展示在屏幕上。如果浏览器监听到页面滚动事件，就会通知渲染进程构建另外一个合成帧来更新页面；

---

# 跨域与同源策略

## 同源策略

浏览器安全基石。**同源**指协议、域名、端口三者一致。不同源的页面间，浏览器限制以下行为：

- **DOM 访问**：阻止跨域页面读取 iframe 的内容
- **数据读取**：阻止跨域请求的响应被 JS 读取（除非服务端允许）
- **存储访问**：`localStorage`、`sessionStorage`、`Cookie` 均受同源限制

## 跨域解决方案

**1. CORS（跨域资源共享）**

服务端通过 HTTP 响应头告知浏览器允许跨域，是最主流的方案。

**响应头：**

| 响应头 | 作用 |
|--------|------|
| `Access-Control-Allow-Origin` | 允许的源，可填 `*` 或具体域名 |
| `Access-Control-Allow-Methods` | 允许的 HTTP 方法 |
| `Access-Control-Allow-Headers` | 允许的请求头 |
| `Access-Control-Allow-Credentials` | 是否允许携带凭据（cookie） |
| `Access-Control-Max-Age` | 预检请求结果缓存时间（秒） |
| `Access-Control-Expose-Headers` | 允许 JS 读取的响应头白名单 |

**请求头：**

| 请求头 | 作用 |
|--------|------|
| `Origin` | 标识请求来源的源 |
| `Access-Control-Request-Method` | 预检请求中，告知实际请求的 HTTP 方法 |
| `Access-Control-Request-Headers` | 预检请求中，告知实际请求的自定义头 |

**简单请求 vs 复杂请求**

**简单请求**需同时满足以下条件：

- 方法：`GET`、`HEAD`、`POST` 之一
- 请求头：仅包含以下安全头（`Accept`、`Accept-Language`、`Content-Language`、`Content-Type` 限 `application/x-www-form-urlencoded`、`multipart/form-data`、`text/plain`）
- 请求中没有 `ReadableStream` 对象

**复杂请求（非简单请求）**：不满足上述任一条件的请求，如：
- 使用 `PUT`、`DELETE`、`PATCH` 等方法
- `Content-Type` 为 `application/json`
- 携带自定义请求头（如 `Authorization`、`X-Requested-With`）

复杂请求会在实际请求前**自动发送一次 OPTIONS 预检请求**，询问服务端是否允许该跨域请求。

```http
// 预检请求（浏览器自动发出）
OPTIONS /api/data HTTP/1.1
Origin: https://example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Authorization

// 预检响应
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: PUT
Access-Control-Allow-Headers: Authorization
Access-Control-Max-Age: 86400
```

**示例——服务端设置 CORS（Node.js）：**

```javascript
// 简单请求
app.get('/api/data', (req, res) => {
  res.setHeader('Access-Control-Allow-Origin', 'https://example.com');
  res.json({ message: 'ok' });
});

// 复杂请求——处理预检
app.options('/api/data', (req, res) => {
  res.setHeader('Access-Control-Allow-Origin', 'https://example.com');
  res.setHeader('Access-Control-Allow-Methods', 'PUT, DELETE');
  res.setHeader('Access-Control-Allow-Headers', 'Authorization, Content-Type');
  res.setHeader('Access-Control-Max-Age', '86400');
  res.sendStatus(204);
});

app.put('/api/data', (req, res) => {
  res.setHeader('Access-Control-Allow-Origin', 'https://example.com');
  res.json({ message: 'updated' });
});
```

**携带凭据（cookie）：**

```javascript
// 前端
fetch('https://api.example.com/data', {
  credentials: 'include', // 携带 cookie
});

// 服务端
res.setHeader('Access-Control-Allow-Origin', 'https://example.com');
res.setHeader('Access-Control-Allow-Credentials', 'true');
// 注意：Allow-Origin 不能为 *
```

**2. JSONP**

利用 `<script>` 标签不受同源策略限制，通过动态创建 script 标签加载跨域数据。

```html
<!-- 前端 -->
<script>
function handleCallback(data) {
  console.log(data);
}
</script>
<script src="https://api.example.com/data?callback=handleCallback"></script>
```

```javascript
// 服务端（Node.js）
app.get('/data', (req, res) => {
  const callback = req.query.callback;
  const data = { message: 'hello' };
  res.send(`${callback}(${JSON.stringify(data)})`);
  // 实际返回：handleCallback({"message":"hello"})
});
```

**缺点：** 只支持 GET，需要服务端配合，无法处理错误，存在安全风险（XSS）。

**3. 代理转发**

同源策略只限制浏览器，浏览器与服务器之间通过代理转发，让请求走同源路径。

**开发环境（Vite）：**

```javascript
// vite.config.js
export default {
  server: {
    proxy: {
      '/api': {
        target: 'https://api.example.com',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
    },
  },
};
```

**生产环境（Nginx）：**

```nginx
server {
    location /api/ {
        proxy_pass https://api.example.com/;
        proxy_set_header Host $host;
    }
}
```

**4. postMessage**

`window.postMessage` 允许不同源的 iframe 或窗口之间安全通信。

```javascript
// 父页面（https://parent.com）
const iframe = document.querySelector('iframe');
iframe.contentWindow.postMessage({ type: 'hello', data: '来自父页面' }, 'https://child.com');

// iframe 子页面（https://child.com）
window.addEventListener('message', (event) => {
  // 验证来源
  if (event.origin !== 'https://parent.com') return;
  console.log(event.data);
  event.source.postMessage({ type: 'reply' }, event.origin);
});
```

**5. WebSocket**

WebSocket 协议本身不受同源策略限制，握手时通过 `Origin` 头验证。

```javascript
const ws = new WebSocket('wss://api.example.com/socket');

ws.onopen = () => ws.send('连接建立');
ws.onmessage = (event) => console.log(event.data);
```

**6. document.domain**

适用于主域名相同的跨子域通信（如 `a.example.com` 和 `b.example.com`）。

```javascript
// 两个页面都设置
document.domain = 'example.com';
// 之后即可通过 iframe 的 contentDocument 访问对方 DOM
```
