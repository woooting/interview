# 项目概述

这是一个前端面试知识库，按技术域分目录，每个目录下用 markdown 文件记录面试知识点。

## 目录结构

```
knowledge/
├── javascript/    # JS 核心：事件循环、原型链、异步、闭包等
├── css/           # CSS 属性、布局、动画等
├── html/          # HTML 标签、语义化、表单等
├── 浏览器/        # 渲染、缓存、存储、安全等
├── 计算机网络/    # HTTP/TCP/UDP/DNS/WebSocket 等
├── 前端ui框架/    # Vue、React、工程化（Webpack/Vite/模块化）
├── Nodejs/        # Node.js 后端相关
├── git/           # Git 操作与工作流
├── 数据库/        # 数据库相关
├── 操作系统/      # 操作系统相关
├── ts/            # TypeScript 相关
└── AI/            # AI 相关
```

## 编写规范

详见仓库根目录的 `STYLE_GUIDE.md`。

## 强制规则

1. **文档必须遵循 STYLE_GUIDE.md**：每次新增或修改 `.md` 文件前，必须先读取 `STYLE_GUIDE.md`，确保格式、标题层级、代码块标注、分隔线、标点等完全符合规范。
2. **对话总结必须使用 knowledge-writer skill**：当用户要求将对话上下文中的知识总结写入知识库文档时（如"总结写入文档"、"写入知识库"、"记录到文档"、"保存"、"写进去"、"存一下"等），必须先加载 knowledge-writer skill，按其流程执行。

## 远程仓库

- Remote: `https://github.com/woooting/interview.git`
- Branch: `master`
