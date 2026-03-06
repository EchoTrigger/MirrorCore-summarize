# 医疗问审功能参考（RAGFlow 知识库）

本指南提供面向知识库检索路径的最小集成方案、接口与事件说明，适合用于落地与答辩展示。

## 一、总体流程（评估闭环）

**简短版摘要**

- 输入医疗问题，系统先做分诊，再从知识库检索证据
- 证据经可解释重排后生成建议，并输出引用来源
- 若证据不足，明确提示并收尾

**流程图（文本版）**

```
用户问题
  ↓
意图识别与澄清
  ↓
分诊评估（triage）
  ↓
知识库检索（search）
  ↓
证据片段汇总（fetch）
  ↓
证据重排（rerank）
  ↓
回答生成 + 引用
  ↓
结果评估与收尾
```

1) 意图识别与澄清：确认医疗场景与必要信息  
2) 分诊评估：生成紧急程度建议  
3) 知识库检索：向量检索 RAGFlow  
4) 证据重排：对检索片段进行重排  
5) 回答生成：结合证据与分诊给出建议  
6) 结果评估与收尾：确认覆盖需求并完成收尾

## 二、快速集成

**RAGFlow 配置**

- 设置 RAGFLOW_BASE_URL 与 RAGFLOW_API_KEY
- 未配置时相关检索接口返回空结果
- 可选：配置 RAGFLOW_DATASET_IDS 或通过运行时接口写入 datasetIds

## 三、REST 接口（后端）

**POST /api/medical/triage**

- 请求：{ question?, symptoms?, durationHours?, age?, pregnant?, comorbidities?, vitals? }
- 响应：{ ok, triage: { score, category, advice, reasons, flags } }

**POST /api/medical/search**

- 请求：{ query, limit?, engine?, timeoutMs?, headless?, locale? }
- 响应：{ ok, results: [{ title?, url, snippet?, sourceUrl? }], count, query }
  - 说明：当前实现基于 RAGFlow 检索，url 为 ragflow:// 形式

**POST /api/medical/rerank**

- 请求：{ query, chunks: [{ id, text }], limit? }
- 响应：{ ok, query, results: [{ id, text, score }] }

**POST /api/medical/rerank/compare**

- 请求：{ query, chunks: [{ id, text }], limit? }
- 响应：{ ok, query, rule: [...], bm25: [...] }

**POST /api/medical/ragflow/retrieve**

- 请求：{ question, dataset_ids: string[], document_ids?, metadata_condition?, similarity_threshold? }
- 响应：{ ok, question, results: [{ id?, text?, score?, dataset_id?, document_id?, metadata? }], count }

**POST /api/medical/ragflow/answer**

- 请求：{ question, dataset_ids: string[], document_ids?, top_k? }
- 响应：{ ok, answer, references }

代码入口：core/src/routes/medical.ts

## 四、WebSocket 事件流

**事件名**

- stream_medical_rag

**入参**

- { question, locale?, limit?, conversationId? }

**事件流**

- stream_medical_rag_start  
- stream_medical_rag_progress(stage: triage/search/fetch/rerank)  
- stream_medical_rag_content({ content, references, triage })  
- stream_medical_rag_end  

**stage 说明**

- triage：分诊评估  
- search：知识库检索  
- fetch：从检索结果整理片段  
- rerank：证据重排

代码入口：core/src/server.ts

## 五、MCP 工具

**med__triage_score**

- 输入：{ question?, symptoms?, durationHours?, age?, pregnant?, comorbidities?, vitals? }
- 输出：{ score, category: 'self_care'|'clinic'|'urgent'|'emergency', advice, reasons[], flags[] }

**med__merck_search**

- 输入：{ query, limit?, engine?, timeoutMs?, headless?, locale? }
- 输出：{ query, count, results: [{ title?, url, snippet?, sourceUrl? }] }

**med__rerank_chunks**

- 输入：{ query, chunks: [{ id, text }], limit? }
- 输出：{ query, results: [{ id, text, score }] }

代码入口：core/src/services/mcp.ts

## 六、数据模型

- Citation：{ source:'attachment'|'dataset'|'web', docId?, datasetId?, url?, chunkId?, page?, offset?, length? }

## 七、最小使用流程

- 文本问题 → /api/medical/search → /api/medical/rerank → 生成答案  
- 结合 RAGFlow → /api/medical/ragflow/retrieve → /api/medical/ragflow/answer

## 八、合规要点

- 不做诊断；必须附带 Citation；患者标识脱敏；密钥/env 不入库。

## 九、本仓库依赖

- 后端 WebSocket/医疗 RAG：[server.ts](file:///d:/Projects/MirrorCore/core/src/server.ts)
- 医疗 REST 路由：[medical.ts](file:///d:/Projects/MirrorCore/core/src/routes/medical.ts)
- MCP 工具注册实现：[mcp.ts](file:///d:/Projects/MirrorCore/core/src/services/mcp.ts)
- RAGFlow 客户端封装：[ragflow.ts](file:///d:/Projects/MirrorCore/core/src/services/ragflow.ts)
- 前端接入入口：通用聊天与 Socket 连接（[ChatArea.tsx](file:///d:/Projects/MirrorCore/ui/src/components/ChatArea.tsx)、[useSocket.ts](file:///d:/Projects/MirrorCore/ui/src/hooks/useSocket.ts)）

---
