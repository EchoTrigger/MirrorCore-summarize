# 记忆功能参考（Markdown + 向量检索）

本文面向 vibe coding 上下文使用，提供可直接落地的接口参考与最小集成路径，所有内容均以仓库现有代码为依据。

## 快速集成

- 后端模式：本地 Markdown 记忆（MEMORY.md + daily/*.md）
- 向量检索：memory_vec.json（由 EMBEDDING_PROVIDER 生成）
- 环境变量：EMBEDDING_PROVIDER、OPENAI_EMBEDDING_MODEL、OLLAMA_EMBEDDING_MODEL、EMBEDDING_DIM、MEMORY_VECTOR_ENABLED
- 代码入口：记忆服务 [memory.ts](file:///d:/Projects/MirrorCore/core/src/services/memory.ts)，配置 [config.ts](file:///d:/Projects/MirrorCore/core/src/services/config.ts)

## REST 接口（后端）

- GET /api/settings/memory/status
  - 响应：{ ok, status: { mode: 'markdown', vectorEnabled, embeddingProvider, embeddingModel, embeddingDim } }
- POST /api/settings/memory/refresh
  - 响应：{ ok, status }
- POST /api/settings/memory/maintenance
  - 请求：{ action, daysToKeep? }
  - 响应：{ ok, action, compactedItems, removedLines, prunedDailyFiles, rebuiltIndex }
- GET /api/settings/memory/items
  - 查询：?agentName=&q=&limit=
  - 响应：{ ok, items: [{ type, key, value, updatedAt }] }
- POST /api/settings/memory/upsert
  - 请求：{ agentName, type, key, value }
  - 响应：{ ok }
- DELETE /api/settings/memory/item
  - 请求：{ agentName, type, key }
  - 响应：{ ok }
- 代码入口：[settings.ts](file:///d:/Projects/MirrorCore/core/src/routes/settings.ts)

## 存储实现与检索

- 主记忆：core/data/memory/MEMORY.md
- 日记忆：core/data/memory/daily/YYYY-MM-DD.md
- 向量索引：core/data/memory/memory_vec.json
- 检索策略：MEMORY_VECTOR_ENABLED=true 时使用向量相似度，否则回退到关键词匹配

## 前端可视化

- UI 入口：设置弹窗中的“记忆”标签
- 代码入口：[SettingsModal.tsx](file:///d:/Projects/MirrorCore/ui/src/components/SettingsModal.tsx#L300-L335)
- 当前状态：界面与入口已就绪，功能由后端配置控制

## 快速使用示例

- 写入记忆项：
```bash
curl -X POST http://localhost:3000/api/settings/memory/upsert \
  -H "Content-Type: application/json" \
  -d '{"agentName":"智能助手","type":"profile","key":"allergy","value":"青霉素"}'
```
- 关键词检索：
```bash
curl "http://localhost:3000/api/settings/memory/items?agentName=%E6%99%BA%E8%83%BD%E5%8A%A9%E6%89%8B&q=%E9%9D%92%E9%9C%89%E7%B4%A0&limit=20"
```

## 合规要点

- 不写入敏感凭据；区分 domain；本地数据文件受 .gitignore 管理

## 本仓库依赖

- 内存服务：[memory.ts](file:///d:/Projects/MirrorCore/core/src/services/memory.ts)
- 设置与路由：[settings.ts](file:///d:/Projects/MirrorCore/core/src/routes/settings.ts)
- 配置文件读写：[config.ts](file:///d:/Projects/MirrorCore/core/src/services/config.ts)
