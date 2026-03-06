# Agent Skills（技能路由 + SOP + 工具白名单）设计稿

本文面向 MirrorCore 的对话系统，定义“按技能选择 SOP + 工具白名单”的运行方式，用于避免“所有能力都跑一遍”的 agent loop，并支持同一会话内在不同技能间自然切换。

相关实现位置（供落地时对照）：
- MCP 工具注册：[mcp.ts](file:///d:/Projects/MirrorCore/core/src/services/mcp.ts)
- 代码执行/MCP 形态说明：[CodeExecution.md](file:///d:/Projects/MirrorCore/docs/CodeExecution.md)
- 医疗问审/RAG 参考实现：[MedicalAuditRAG.md](file:///d:/Projects/MirrorCore/docs/MedicalAuditRAG.md)

---

## 1. 目标与非目标

### 目标

- 用“Skill → SOP → allowedTools”替代“全能力 planner loop”。
- Skill 状态与会话（conversation）解耦：同一会话内可以在多个 Skill 间来回切换。
- 医疗问询（MedicalConsult）支持多轮采集、检索、评估与显式收尾；收尾后回退 GeneralChat。
- 工具调用可控：每个 Skill 有明确的工具白名单、预算与停止条件。

### 非目标

- 不追求全自动“通用代理”覆盖所有领域。
- 不在路由阶段调用工具（避免为了选 Skill 反而触发 loop）。
- 不把医疗问询做成诊断系统（输出必须遵守合规约束）。

---

## 2. 核心概念

### Skill

Skill 是一组能力的“最小运行单元”，包含：

- 路由规则（matcher）：判断当前用户消息是否应由该 Skill 处理。
- SOP（标准作业流程）：确定本轮要问什么、查什么、怎么组织答案。
- 工具白名单（allowedTools）：本 Skill 允许调用的 MCP 工具集合。
- 预算（budgets）：每轮最多工具调用次数、最多检索条数、最大 token 等。
- 过滤器（filters）：输入/证据/输出的约束与纠错策略。

### Session / Conversation

- Conversation（会话）用于承载消息历史与跨轮上下文。
- SkillContext 是挂在 Conversation 下、按 Skill 维度分区的“可暂停状态”。

关键点：

- 会话是一个；Skill 可以多个。
- activeSkill 只是“当前指针”，不意味着其他 SkillContext 被销毁。

### SkillContext

SkillContext 推荐最小字段：

- skillId：Skill 唯一标识
- phase：当前 SOP 阶段（例如 MedicalConsult 的 intake/clarify/…）
- slots：结构化采集结果（MedicalConsult 的核心）
- artifacts：可复用的中间产物（例如已检索到的引用、摘要、triage 结果）
- lastUpdatedAt：用于过期或回收策略

---

## 3. Skill 列表（v1）

当前现有 6 个 Skill：

### 3.1 GeneralChat

- 适用：闲聊、写作/编程问答、非天气非医疗的日常问题。
- 特征：默认不调用外部工具；必要时只做最小化推理。
- 收尾：每轮独立完成。

### 3.2 WeatherAssistant

- 适用：天气、空气质量、生活指数、未来预报。
- 特征：强工具依赖，工具集合固定。
- 收尾：用户得到所需信息即结束。

### 3.3 MedicalConsult

- 适用：医疗健康问询、症状解释、用药/就医建议（非诊断）。
- 特征：多轮 slots 采集 + triage + 循证建议。
- 收尾：显式“完成问询”后将 phase 置为 closed，并回退 GeneralChat。

### 3.4 MedicalRagSkill

- 适用：循证医学检索与引用式回答。
- 特征：面向证据检索、重排与引用输出。
- 收尾：命中 closeSignals 或用户明确停止。

### 3.5 WebResearchSkill

- 适用：通用联网检索与结果整理。
- 特征：优先明确 query 与范围，再检索并给出来源。
- 收尾：命中 closeSignals 或用户明确停止。

### 3.6 SiteSearchSkill

- 适用：站内搜索入口与快捷跳转。
- 特征：识别站点与关键词，输出搜索入口或指引。
- 收尾：命中 closeSignals 或用户明确停止。

---

## 4. Skill 路由（匹配规则）

路由目标：在不调用任何工具的前提下，用“确定性规则优先 + 轻量回退”选出 activeSkill。

### 4.1 输入与上下文

routerInput：

- text：本轮用户输入（trim 后）
- conversationId：会话标识
- activeSkillId：上轮选中的 Skill
- skillContexts：各 Skill 的 phase/slots 是否处于“未完成状态”

### 4.2 确定性规则（Hard Rules）

优先级从高到低：

1) MedicalConsult 继续态
- 如果 MedicalConsult.phase 不是 closed，并且用户输入满足“继续医疗问询”信号，则直接路由 MedicalConsult。
- 继续信号示例：补充症状/时间、回答澄清问题、追问“那我该怎么办/要不要去医院/吃什么药”。
- MedicalConsult 需启用开关：`SKILL_MEDICAL_ENABLE=true`。

2) SiteSearchSkill 站内搜索意图
- 命中“在XX搜/搜索XX平台/打开XX站点”等站内搜索或跳转意图。

3) WeatherAssistant 继续态
- activeSkill 为 WeatherAssistant 且未完成，并且输入仍为天气相关补充。

4) WeatherAssistant 天气强意图
- 命中天气关键词/句式：天气、气温、降雨、风、湿度、体感、预报、空气质量、AQI、污染、紫外线、穿衣、指数、台风、暴雨。
- 或者包含明确地名/定位意图：例如“上海今天/明天/这周天气”。

5) MedicalRagSkill 循证意图
- 命中“循证/文献/指南/证据/引用/出处/MSD/默沙东”等检索意图。
- MedicalRagSkill 需启用开关：`SKILL_MEDICAL_ENABLE=true`。

6) MedicalConsult 医疗强意图
- 命中医疗关键词：症状（疼、发烧、咳嗽、腹泻、皮疹、头晕…）、疾病名、用药、检查、就医、急诊、怀孕/儿童/老人等高风险群体提示。
- 命中“风险/紧急程度”句式：要不要去医院、会不会很严重、急不急。
- MedicalConsult 需启用开关：`SKILL_MEDICAL_ENABLE=true`。

7) 默认 GeneralChat

---

## 5. 执行模型（SOP + 工具白名单 + 预算）

每轮对话按如下顺序执行：

1) router 选 activeSkill
2) 加载/创建该 SkillContext
3) 运行该 Skill 的 SOP（phase 驱动）
4) SOP 内按需调用工具，但必须受 allowedTools 与 budgets 限制
5) 应用 filters（输入/证据/输出）
6) 产出回复，并更新 SkillContext

### 5.1 budgets（现状）

- maxToolCallsPerTurn：1（WeatherAssistant）/ 4（MedicalConsult、MedicalRagSkill）/ 1（WebResearchSkill、SiteSearchSkill）/ 0（GeneralChat）
- maxSearchResults：5（MedicalConsult、MedicalRagSkill、WebResearchSkill）
- maxRerankChunks：30（MedicalConsult、MedicalRagSkill）

### 5.2 停止条件（避免 loop）

- 本轮已达到 maxToolCallsPerTurn：必须直接回答或提出澄清问题，不再检索。
- 同一 query 连续检索 2 次仍无新证据：停止检索，转为“信息不足”的建议与下一步。
- 医疗问询未收集到最小 slots：优先提问，不检索。

### 5.3 上下文与记忆治理

- 上下文优先级：当前用户消息 > 当前 SkillContext 摘要 > 历史摘要 > 工作记忆。
- 工作记忆仅保留结构化状态：skillId、phase、关键 slots 摘要；不注入 tool_output 原文。
- GeneralChat 只回答当前问题，不续答已完成的旧任务。
- 当技能被禁用或不可用时，返回确定性兜底答案，不回退自由生成。

### 5.4 可观测性与回放

- 路由日志需包含：candidateSkillIds、reason、confidence、prevSkillId。
- 工具日志以结构化字段存储：tool、query、count、latency、error。
- 每轮保存 SOP 阶段转移记录，便于回放与问题定位。

---

## 6. MedicalConsult（多轮问询设计）

### 6.1 输出合规边界

- 不做诊断；使用“可能/常见原因”表述。
- 必须明确“紧急程度建议”与“何时就医”。
- 引用式回答：凡是给出医学事实/用药/处置建议，尽量附 Merck 引用（URL + 摘要）。
- 避免收集可识别身份信息；如用户主动提供，slots 里不保留原文标识。

### 6.2 slots（字段定义）

MedicalConsult.slots 是结构化问询的核心，用于跨轮持续。

字段表（v1）：

| 字段 | 类型 | 必填 | 采集方式 | 校验/归一 | 说明 |
|---|---|---:|---|---|---|
| chiefComplaint | string | 是 | 直接询问“最困扰你的症状是什么” | 去除无意义寒暄 | 主诉一句话 |
| symptoms | string[] | 是 | 让用户列出症状清单 | 去重、小写/中文同义归一（可选） | 用于 triage 与检索 |
| onsetAt | string\|null | 否 | “什么时候开始的” | 允许自然语言（昨晚/三天前） | 可选时间锚 |
| durationHours | number\|null | 否 | 时长换算（小时） | 非负；未知则 null | triage 常用 |
| severity | 'mild'\|'moderate'\|'severe'\|null | 否 | 0-10 或描述转等级 | 不确定则 null | 影响风险评估 |
| age | number\|null | 否 | 询问年龄段或年龄 | 0..120 | 儿童/老人风险 |
| sex | 'female'\|'male'\|'other'\|'unknown' | 否 | 可不问 | 默认 unknown | 合规不强制 |
| pregnant | boolean\|null | 否 | 仅当相关时询问 | 不确定则 null | 高风险分层 |
| comorbidities | string[] | 否 | 既往史/慢病 | 去重 | 糖尿病/心衰等 |
| medications | string[] | 否 | 现用药/近期用药 | 去重 | 用药建议前置 |
| allergies | string[] | 否 | 药物/食物过敏 | 去重 | 用药风险 |
| vitals | { hr?:number; sbp?:number; dbp?:number; tempC?:number; spo2?:number }\|null | 否 | 可选询问 | 合理范围校验（可选） | triage 输入 |
| redFlags | string[] | 否 | 通过问答与规则触发 | 去重 | 危险信号列表 |
| constraints | { locale?: string; preferredOutput?: 'concise'|'detailed' } | 否 | 从用户偏好推断 | 默认 zh + detailed | 输出风格 |
| userGoal | string\|null | 否 | “你最想确认什么” | 可空 | 例如“要不要去医院” |

最小可运行 slots（进入检索/生成前必须满足）：

- chiefComplaint 非空
- symptoms 至少 1 个

### 6.3 phase（状态机）

MedicalConsult.phase（v1）：

- intake：收集主诉与症状，确认是否是医疗问询
- clarify：补齐关键 slots（时长/严重度/年龄/危险信号）
- triage：调用 triage 工具给出紧急程度建议
- retrieve：从知识库检索相关证据
- synthesize：基于证据组织解释与建议
- evaluate：检查是否回答了 userGoal，是否存在遗漏的危险信号
- close：用户确认“完成/已解决”，输出最终总结并收尾

核心转移规则：

- intake → clarify：识别为医疗主题且用户有明确困扰
- clarify → triage：最小 slots 满足，且不再需要关键澄清
- triage → retrieve：需要医学解释/处置建议时
- synthesize → evaluate：生成草稿后
- evaluate → clarify：仍缺关键信息（例如高风险人群、危险信号无法排除）
- evaluate → close：用户表示“明白了/谢谢/就这样/不用了”或已明确下一步计划

### 6.4 allowedTools（白名单）

MedicalConsult 只允许调用医疗相关 MCP 工具：

- med__triage_score
- med__merck_search
- med__rerank_chunks

工具定义参见：[mcp.ts](file:///d:/Projects/MirrorCore/core/src/services/mcp.ts)

### 6.5 filters（过滤器）

输入过滤（Input Filter）：

- 若用户请求“诊断结论/处方/替代医生”，输出时必须加入边界声明，并引导到就医或进一步信息采集。
- 若用户输入包含自伤/他伤、胸痛呼吸困难、意识障碍等高危信号：直接提升 triage 优先级，跳过检索。

证据过滤（Evidence Filter）：

- 仅接受知识库检索结果作为证据来源。
- 每个引用至少包含：来源标识、摘录文本、与回答的关联点。
- 如果缺乏可靠证据：明确写“未检索到可靠来源”，不要编造。

输出过滤（Output Filter）：

- 输出必须包含：紧急程度建议、下一步行动、何时就医、引用列表（若有检索）。
- 对不确定信息使用概率措辞，避免绝对化。

### 6.6 建议的回答模板

MedicalConsult 的最终回答建议固定结构，便于用户扫描与后续产品化：

1) 你目前的情况（基于已采集 slots 的复述）
2) 需要再确认的问题（如果仍在 clarify/evaluate）
3) 紧急程度建议（triage 结论 + 理由）
4) 可能原因/机制（不诊断，基于证据）
5) 你现在可以做什么（自我处理/观察要点）
6) 何时就医/急诊（危险信号清单）
7) 参考来源（知识库来源列表 + 摘要）

### 6.7 “完成问询”的判定

满足任一即可进入 close：

- 用户明确表示结束：例如“谢谢/明白了/不用了”。
- userGoal 已被回答，且 evaluate 阶段未发现未处理的红旗信号。
- 已给出明确的下一步计划（观察/门诊/急诊），并提示了回访条件。

close 阶段要做：

- 输出最终总结（≤ 10 行核心要点）
- 把 phase 设为 closed
- activeSkill 回退 GeneralChat

---

## 7. WeatherAssistant（最小 SOP）

### 7.1 allowedTools

- weather__now
- weather__hourly
- weather__daily
- indices__forecast
- air__current / air__hourly / air__daily（如产品需要空气质量）

### 7.2 SOP（每轮）

- 若缺 location：先问“你在哪个城市/地区”。
- 根据用户时间范围选择 now/hourly/daily。
- 输出时带上更新时间与关键指标（温度/降水概率/风/空气质量）。

---

## 8. GeneralChat（最小 SOP）

- 默认不调用工具。
- 当用户的问题明确需要外部检索时，优先建议切换到具备检索工具的 Skill。

---

## 本仓库依赖

- [skill_router.ts](file:///d:/Projects/MirrorCore/core/src/services/skill_router.ts)
- [chat_orchestrator.ts](file:///d:/Projects/MirrorCore/core/src/services/chat_orchestrator.ts)
- [conversation.ts](file:///d:/Projects/MirrorCore/core/src/services/conversation.ts)
- [mcp.ts](file:///d:/Projects/MirrorCore/core/src/services/mcp.ts)
- [MedicalConsult.md](file:///d:/Projects/MirrorCore/skills/MedicalConsult.md)
- [WeatherAssistant.md](file:///d:/Projects/MirrorCore/skills/WeatherAssistant.md)
