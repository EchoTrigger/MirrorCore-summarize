# MirrorCore 部署与运行

本指南基于仓库现状整理，包含 Windows、macOS 与 Linux 的完整部署运行步骤。

---

## 环境要求

- 操作系统：Windows 10/11（推荐）、macOS、Linux
- Node.js ≥ 18 与 npm ≥ 9
- Git
- Playwright Chromium（安装脚本会自动安装）
- 可选：向量嵌入服务（用于记忆语义检索）

---

## 快速开始（Windows 推荐）

1. 克隆项目并进入目录
   ```powershell
   git clone https://github.com/your-username/MirrorCore.git
   cd MirrorCore
   ```
2. 一键安装依赖
   ```powershell
   npm run setup
   ```
3. 配置后端环境变量
   ```powershell
   copy .\core\.env.example .\core\.env
   ```
   在 `core/.env` 中填写你的 AI 配置，例如：
   ```env
   AI_PROVIDER=openai
   OPENAI_API_KEY=你的OpenAI密钥
   OPENAI_MODEL=gpt-5-mini
   ```
4. 启动所有服务（后端 + UI）
   ```powershell
   npm run dev
   ```
5. 访问与联调
   - UI 访问：`http://localhost:5173/`（开发环境）
   - 后端健康检查：`http://localhost:3000/`
   - MCP 工具入口（HTTP Streamable）：`POST http://localhost:3000/mcp`
   - MCP 注册信息：`GET http://localhost:3000/mcp/registry/tools`、`GET http://localhost:3000/mcp/registry/servers`

---

## 跨平台启动（macOS/Linux）

1. 安装依赖
   ```bash
   npm run setup
   ```
2. 配置环境变量
   ```bash
   cp core/.env.example core/.env
   ```
   在 `core/.env` 中填写你的 AI 配置，例如：
   ```env
   AI_PROVIDER=openai
   OPENAI_API_KEY=your_openai_api_key
   OPENAI_MODEL=gpt-5-mini
   ```
3. 启动开发环境
   ```bash
   npm run dev
   # 或分别启动：
   # npm run dev --prefix core
   # npm run dev --prefix ui
   ```

---

## 可选：启用向量语义检索

在 `core/.env` 配置向量提供商：

```env
EMBEDDING_PROVIDER=openai
OPENAI_EMBEDDING_MODEL=text-embedding-3-small
MEMORY_VECTOR_ENABLED=true
```

---

## 端口与入口

- 后端默认端口：`:3000`（通过 `core/.env` 的 `PORT` 可修改）
- REST API 前缀：`/api/*`（如 `/api/search`、`/api/medical`）
- 流式通道：Socket.IO（开发时 UI 自动连接）
- MCP HTTP 入口：`POST /mcp`
- 前端构建预览：`/preview`（需先 `npm run build --prefix ui`）

---

## 常见问题

- Node/npm 版本过低：请升级到 Node ≥ 18、npm ≥ 9
- Playwright 浏览器未安装：后端 `postinstall` 会自动执行 `playwright install chromium`；如失败可手动执行
  ```bash
  npm run postinstall --prefix core || npx playwright install chromium --prefix core
  ```
- 端口被占用：在 `core/.env` 设置 `PORT=3001` 并重新启动
- API Key 未配置：确保 `core/.env` 已正确填写对应提供商的密钥与模型

---

## 停止与重启

- 停止：在各终端窗口按 `Ctrl + C` 或关闭浏览器标签页
- 重启：再次执行 `npm run dev`
