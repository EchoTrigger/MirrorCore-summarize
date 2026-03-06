# MirrorCore —— 模块化智能对话助手系统 🤖

MirrorCore 面向个人与团队的智能对话助手系统，覆盖实时对话、联网检索、记忆（Markdown + 向量检索）、MCP 工具调用与安全代码执行。架构采用 Web UI（React + Vite）与 Node/Express 后端，强调模块化、可视化与可扩展。

---

## ✨ 现有能力

- 流式对话：WebSocket 实时输出，保留会话历史与引用来源
- 联网检索：Playwright/DuckDuckGo/Bing/Google，支持运行时配置
- 记忆系统：Markdown 记忆 + 向量检索，前端可视化管理
- MCP 工具：统一封装外部服务并提供注册/发现入口
- 代码执行：受控 VM + 工具 API 方式对外暴露能力
- Web UI：聊天、设置与可视化入口一体化

---

## 🧭 系统构成

- 前端：React + Vite + Socket.IO 客户端
- 后端：Node.js + Express + Socket.IO + Playwright
- 数据：文件系统 `data/`（会话、配置、记忆）
- 协议：MCP（Model Context Protocol）Streamable HTTP/stdio

---

## 📚 文档导航

- 部署与运行：[SETUP.md](SETUP.md)
- API 参考：[docs/API.md](docs/API.md)
- 记忆功能：[docs/Memory.md](docs/Memory.md)
- 代码执行：[docs/CodeExecution.md](docs/CodeExecution.md)
- 前端结构：[docs/ui.md](docs/ui.md)
- 医疗问审 RAG：[docs/MedicalAuditRAG.md](docs/MedicalAuditRAG.md)
- 技能设计与路由：[docs/AgentSkills.md](docs/AgentSkills.md)
- 技能资产化架构：[docs/SkillAssetsArchitecture.md](docs/SkillAssetsArchitecture.md)

---

## 🔌 关键入口

- UI 开发环境：`http://localhost:5173/`
- 后端健康检查：`http://localhost:3000/`
- MCP 工具入口：`POST http://localhost:3000/mcp`
- MCP 注册信息：`GET http://localhost:3000/mcp/registry/tools`
- REST API 前缀：`/api/*`（如 `/api/search`、`/api/medical`）
