# 讲座检索系统 - 前端

基于 Vue 3 + TypeScript 的讲座检索系统前端，提供全文检索与 AI 智能助手功能。

## 功能特性

- **多维度检索** - 支持全文检索、按标题检索、按演讲者检索三种模式
- **关键词高亮** - 搜索结果自动高亮匹配关键词
- **拼写纠错** - 自动检测并提供拼写建议，一键修正
- **AI 智能助手** - 基于 SSE 流式传输的 AI 对话，支持 Markdown 渲染与代码高亮
- **响应式设计** - 适配桌面端与移动端，小屏幕自动调整布局

## 技术栈

| 类别 | 技术 |
|------|------|
| 框架 | Vue 3 (Composition API + `<script setup>`) |
| 语言 | TypeScript |
| 构建工具 | Vite |
| 状态管理 | Pinia |
| HTTP 客户端 | Axios |
| Markdown 渲染 | Marked.js |
| 代码高亮 | Highlight.js |
| 关键词标记 | Mark.js |

## 项目结构

```
src/
├── assets/
│   ├── base.css               # CSS 变量与基础样式
│   ├── main.css               # 全局样式
│   └── logo.svg               # 应用 Logo
├── components/
│   ├── ChatInterface.vue      # AI 智能助手组件
│   ├── CustomSelect.vue       # 自定义下拉选择组件
│   └── SearchPartInterface.vue # 检索主界面组件
├── App.vue                    # 根组件（布局与 AI 助手入口）
└── main.ts                    # 应用入口
```

## 快速开始

### 环境要求

- Node.js >= 18
- npm >= 9

### 安装与运行

```bash
# 克隆仓库
git clone https://github.com/wybbb1/lecture-search-system-front.git

# 进入项目目录
cd lecture-search-system-front

# 安装依赖
npm install

# 启动开发服务器
npm run dev
```

### 构建生产版本

```bash
npm run build
```

## API 接口

前端默认连接后端服务 `http://localhost:8080`，需要配合 [讲座检索系统后端](https://github.com/wybbb1/lecture-search-system) 使用。

| 接口 | 方法 | 说明 |
|------|------|------|
| `/search` | GET | 检索讲座（参数: `query`, `type` 其中 type: 0=全文, 1=标题, 2=演讲者） |
| `/search/advice` | GET | 拼写纠错建议（参数: `query`） |
| `/chat` | POST | AI 对话（SSE 流式，参数: `memoryId`, `userMessage`） |
| `/reset` | GET | 重置对话（参数: `memoryId`） |

## 相关仓库

- [讲座检索系统后端](https://github.com/wybbb1/lecture-search-system) - Spring Boot 后端服务

## 许可证

MIT
