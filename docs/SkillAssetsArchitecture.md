# Skills 资产化架构映射与迁移路线

本方案将现有的 Skills 逻辑映射为可装卸资产，并统一支持对话驱动与自主 Agent 的工具治理。

## 架构分层与当前模块映射

### 1) Skill 资产层（Asset Layer）
- 定义：技能的静态资产与运行参数，支持装卸
- 当前模块：
  - skills/*.md
  - core/src/services/skill_manifest.ts
- 现状说明：
  - 通过 Markdown frontmatter 读取 allowedTools 与 budgets
  - 启动时 refreshSkillManifestsFromDisk 加载

### 2) Skill 选择层（Pre-Router Layer）
- 定义：LLM 路由前的“技能池收敛”与策略过滤
- 当前模块：
  - core/src/services/skill_router.ts（规则路由）
- 现状说明：
  - 规则驱动的意图识别与 continue 逻辑
  - 允许工具与预算优先读取 manifest

### 3) 路由层（Router Layer）
- 定义：在候选技能池内做最终技能决策
- 当前模块：
- core/src/services/skill_router.ts（routeSkill）
- 现状说明：
  - 每轮消息调用 routeSkill 并记录 operationLogs
  - 由 agent_loop 写回 activeSkillId

### 4) 执行层（Executor Layer）
- 定义：统一工具治理、预算控制与审计
- 当前模块：
- core/src/services/agent_loop.ts（getAllowedTools / getSkillBudgets / 对话执行）
  - core/src/services/agent/stream_orchestrator.ts（流式工具执行编排）
  - core/src/services/agent/tool_runner.ts（工具白名单与预算控制）
  - core/src/services/agent-exec.ts（代码执行沙箱）
  - core/src/services/mcp.ts（工具调用入口）

### 5) 状态层（State Layer）
- 定义：Skill 上下文与会话状态持久化
- 当前模块：
  - core/src/services/conversation.ts（activeSkillId / skillContexts）

## 对话与自主 Agent 的统一执行流程

1. 输入（用户消息 / 任务节点）
2. SkillSelector 收敛候选技能池
3. Router 决策 activeSkillId
4. SkillExecutor 按 manifest 执行 SOP
5. 记录状态与日志，写回 conversationState

## 本仓库依赖

- skills/*.md（技能资产）
- [skill_manifest.ts](file:///d:/Projects/MirrorCore/core/src/services/skill_manifest.ts)
- [skill_router.ts](file:///d:/Projects/MirrorCore/core/src/services/skill_router.ts)
- [agent_loop.ts](file:///d:/Projects/MirrorCore/core/src/services/agent_loop.ts)
- [stream_orchestrator.ts](file:///d:/Projects/MirrorCore/core/src/services/agent/stream_orchestrator.ts)
- [tool_runner.ts](file:///d:/Projects/MirrorCore/core/src/services/agent/tool_runner.ts)
- [conversation.ts](file:///d:/Projects/MirrorCore/core/src/services/conversation.ts)
- [agent-exec.ts](file:///d:/Projects/MirrorCore/core/src/services/agent-exec.ts)
- [mcp.ts](file:///d:/Projects/MirrorCore/core/src/services/mcp.ts)

## 验收标准

1. 新增 Skill 仅需新增 md 与可选 hooks
2. LLM 路由仅在候选池内选择
3. 工具白名单与预算完全由 manifest 决定
4. 对话与自主 Agent 使用同一执行入口

## 风险与约束

1. 不能允许 LLM 选择不存在的 Skill
2. 不能让工具可见性依赖 LLM 输出
3. 预算与工具白名单必须由 manifest 强约束

