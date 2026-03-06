# MirrorCore API 简明文档 🤖

基地址：`http://localhost:3000`

---

## 聊天（Chat）

- `POST /api/chat/message`
  - 请求体：`{ conversationId?, message?, imageData?, model?, temperature?, maxTokens?, agentName?, personalityPrompt? }`
  - 响应：`{ conversationId, message, response }`
- 实现参考：`core/src/routes/chat.ts`

- WebSocket 事件：`stream_chat`
  - 客户端发送：`{ conversationId?, message, imageData?, options? }`
  - 服务端事件：`stream_start`、`stream_content`、`stream_end`
- 事件实现：`core/src/server.ts`、客户端接入：`ui/src/components/ChatArea.tsx`、`ui/src/hooks/useSocket.ts`

- `GET /api/chat/conversations`
  - 查询：`limit`、`offset`
  - 用途：分页获取对话列表
- 实现参考：`core/src/routes/chat.ts:568-586`

- `GET /api/chat/conversations/:conversationId`
  - 用途：获取特定对话详情
- 实现参考：`core/src/routes/chat.ts:589-603`

- `DELETE /api/chat/conversations/:conversationId`
  - 用途：删除对话
- 实现参考：`core/src/routes/chat.ts:606-620`

---

## 搜索与页面抓取（Search & Fetch）

- `GET /api/search`
  - 查询：`query|q`、`mode`、`engine`、`limit`、`headless`、`locale`、`timeoutMs`、`rich`
  - 用途：统一搜索入口，内部经 MCP `mirrorcore__web_search`
- 实现参考：`core/src/routes/search.ts`

- `GET /api/search/fetch`
  - 查询：`url`、`headless`、`timeoutMs`、`locale`
  - 用途：抓取页面内容，经 MCP `mirrorcore__fetch_page`
- 实现参考：`core/src/routes/search.ts`

---

## 运行时设置（Settings）

- `GET /api/settings/search` / `PUT /api/settings/search`
  - 用途：读取/更新搜索运行时配置
- `GET /api/settings/chat` / `PUT /api/settings/chat`
  - 用途：读取/更新聊天注入参数（历史 TopK、摘要长度等）
- `GET /api/settings/ragflow` / `PUT /api/settings/ragflow`
  - 用途：读取/更新 RAGFlow datasetIds
- `GET /api/settings/planer` / `PUT /api/settings/planer`
  - 用途：读取/更新 Planner 运行时配置
- 实现参考：`core/src/routes/settings.ts`

---

## 记忆（Memory）

- `GET /api/settings/memory/status`
  - 返回：`{ mode: 'markdown', vectorEnabled, embeddingProvider, embeddingModel, embeddingDim }`
- `POST /api/settings/memory/refresh`
  - 用途：刷新向量索引
- `POST /api/settings/memory/maintenance`
  - 用途：执行维护操作（compact/prune/rebuild 等）
- 实现参考：`core/src/routes/settings.ts`、`core/src/services/memory.ts`

- `GET /api/settings/memory/items`
  - 查询：`agentName|agent`、`q`、`limit`
  - 用途：关键词 TopK 或最近更新的记忆项
- 实现参考：`core/src/routes/settings.ts`

- `POST /api/settings/memory/upsert`
  - 请求体：`{ agentName, type, key, value }`
  - 用途：插入或更新记忆项
- 实现参考：`core/src/routes/settings.ts`

- `DELETE /api/settings/memory/item`
  - 请求体或查询：`{ agentName, type, key }`
  - 用途：删除记忆项
- 实现参考：`core/src/routes/settings.ts`

详见专题文档：`docs/Memory.md`

---

## 站点搜索（Site Search）

- `GET /api/sites`
  - 用途：获取站点列表与别名
- `GET /api/sites/search-url`
  - 查询：`siteKey`、`q`
  - 用途：生成站内搜索链接
- 实现参考：`core/src/routes/sites.ts`

---

## 站内搜索意图（Intent）

- `POST /api/intent/site-search`
  - 请求体：`{ text }`
  - 用途：解析站内搜索意图并返回结构化结果
- 实现参考：`core/src/routes/intent.ts`

---

## 代码执行（Agent Exec）

- `POST /api/agent/exec`
  - 请求体：`{ code, params?, timeoutMs?, searchOptions? }`
  - 响应：`{ ok, output? , error? }`
- 实现参考：`core/src/routes/agent.ts`

详见专题文档：`docs/CodeExecution.md`

---

## 代码执行工具转发

- `POST /api/agent/tools/weather/now`
- `POST /api/agent/tools/weather/hourly`
- `POST /api/agent/tools/weather/daily`
- `POST /api/agent/tools/air/current`
- `POST /api/agent/tools/air/hourly`
- `POST /api/agent/tools/air/daily`
- `POST /api/agent/tools/indices/forecast`
- 实现参考：`core/src/routes/agent.ts`

---

## MCP（工具调用入口）

- `POST /mcp`
  - 用途：Streamable HTTP MCP 入口
- 实现参考：`core/src/server.ts`

- `GET /mcp/registry/tools`
  - 用途：查看当前注册工具
- 实现参考：`core/src/server.ts`

- `GET /mcp/registry/servers`
  - 用途：查看远端 MCP 服务器列表
- 实现参考：`core/src/server.ts`

- `POST /mcp/registry/servers`
  - 用途：注册远端 MCP 服务器（baseUrl/httpPath/transport）
- 实现参考：`core/src/server.ts`

- `DELETE /mcp/registry/servers/:key`
  - 用途：移除已注册远端 MCP 服务器
- 实现参考：`core/src/server.ts`

- `POST /mcp/plugins/load`
  - 用途：加载本地插件并刷新工具注册
- 实现参考：`core/src/server.ts`

- `DELETE /mcp/plugins/:name`
  - 用途：卸载本地插件并刷新工具注册
- 实现参考：`core/src/server.ts`

---

## 本仓库依赖

- [server.ts](file:///d:/Projects/MirrorCore/core/src/server.ts)
- [chat.ts](file:///d:/Projects/MirrorCore/core/src/routes/chat.ts)
- [search.ts](file:///d:/Projects/MirrorCore/core/src/routes/search.ts)
- [settings.ts](file:///d:/Projects/MirrorCore/core/src/routes/settings.ts)
- [agent.ts](file:///d:/Projects/MirrorCore/core/src/routes/agent.ts)

---
