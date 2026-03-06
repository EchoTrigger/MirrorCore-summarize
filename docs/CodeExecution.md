# 代码执行（Agent Exec）

MirrorCore 提供一个受控的 JavaScript 执行环境，通过将能力以“工具 API”形式暴露，使代理以代码方式编排任务，避免提示词堆叠与上下文膨胀。

---

## 接口

- `POST /api/agent/exec`
  - 请求体：`{ code: string, params?: any, timeoutMs?: number, searchOptions?: any }`
  - 响应：`{ ok: boolean, output?: any, error?: string }`
- 实现参考：[agent.ts](file:///d:/Projects/MirrorCore/core/src/routes/agent.ts)

---

## 访问控制与安全策略

- 开关与鉴权：
  - 环境变量：`CODE_EXECUTION_ENABLE`（启用开关）、`CODE_EXECUTION_ALLOW_REMOTE`（允许远程）、`CODE_EXECUTION_TOKEN`（远程访问令牌）。
- 访问规则：本地 IP 允许；远程仅在允许远程且令牌匹配时通过；否则返回 403。实现参考：[agent.ts](file:///d:/Projects/MirrorCore/core/src/routes/agent.ts)。
- 沙箱限制：
  - 禁用关键词：`require`、`process`、`child_process`、`fs`、`net`、`http`、`https`、`dns`、`vm`、`eval`、`Function`、`global`、`Buffer`。
  - 工具白名单：`search`（默认可用）；`weatherNow`、`weatherHourly`、`cameraCapture` 在默认实现中不可用。
  - 动态注入：`mcp` 可按需注入，不提供时拒绝调用。
- 超时与输出：默认超时 `8000ms`（可传 `timeoutMs` 覆盖）；字符串或 JSON 输出大小上限 `512KB`。实现参考：[agent-exec.ts](file:///d:/Projects/MirrorCore/core/src/services/agent-exec.ts)。
- MCP 桥接：
- 代码中的 `tools.search` 通过 MCP 工具 `mirrorcore__web_search` 调用后端搜索能力；工具入参校验采用统一模式。实现参考：[agent.ts](file:///d:/Projects/MirrorCore/core/src/routes/agent.ts)。

---

## 目录结构（mcp/ 包）

```
mcp/
├── package.json
├── tsconfig.json
├── src/
│   ├── callMCPTool.ts               # 通用调用层（正式 MCP 客户端：Client + HTTP/stdio）
│   ├── discovery.ts                 # 文件系统工具发现：servers/*
│   ├── index.ts                     # 列出本地代码封装的服务器与工具
│   └── servers/
│       ├── camera/
│       │   ├── captureImage.ts      # 工具封装：camera__capture_image
│       │   └── index.ts
│       ├── salesforce/
│       │   ├── updateRecord.ts      # 工具封装：salesforce__update_record
│       │   └── index.ts
│       ├── weather/
│       │   ├── now.ts               # 工具封装：weather__now
│       │   ├── hourly.ts            # 工具封装：weather__hourly
│       │   └── index.ts
│       ├── mirrorcore/
│       │   └── search.ts            # MirrorCore 后端能力的 MCP 工具封装
│       └── playweightagent/
│           ├── search.ts            # MirrorCore 搜索封装（Playwright/duckduckgo）
│           └── index.ts
├── examples/
│   ├── camera_describe.ts           # 摄像头示例
│   └── list_remote_tools.ts         # 枚举远端 MCP 工具
└── python/
    └── server.py                    # Python 示例服务器
```

---

## 工具转发接口

这些接口受同一套代码执行权限控制，适合在不执行任意代码的情况下直接调用工具。

- `POST /api/agent/tools/weather/now`
- `POST /api/agent/tools/weather/hourly`
- `POST /api/agent/tools/weather/daily`
- `POST /api/agent/tools/air/current`
- `POST /api/agent/tools/air/hourly`
- `POST /api/agent/tools/air/daily`
- `POST /api/agent/tools/indices/forecast`

---

## 代码调用示例（Weather → Salesforce）

```ts
import { weatherNow } from './mcp/src/servers/weather';
import { updateRecord } from './mcp/src/servers/salesforce';

const now = await weatherNow({ location: 'Shanghai', lang: 'zh' });
await updateRecord({
  objectType: 'SalesMeeting',
  recordId: '00Q5f000001abcXYZ',
  data: { Notes: `天气：${now.now.text}，温度：${now.now.temp}` }
});
```

> 运行示例：
> 1) 在 `mcp/` 安装依赖：`npm install`（需 Node ≥ 18）
> 2) 配置 MCP 服务器：设置 `MCP_SERVER_URL` 指向可达的 MCP HTTP 端点（例如 `http://localhost:3000/mcp`），或配置 `MCP_TRANSPORT=stdio`
> 3) 枚举远端工具：`npm run example:list-tools`
> 4) 运行摄像头示例：`npx tsx examples/camera_describe.ts`（需远端 MCP 提供 `camera__capture_image`）

---

## 本仓库依赖

- [agent.ts](file:///d:/Projects/MirrorCore/core/src/routes/agent.ts)
- [agent-exec.ts](file:///d:/Projects/MirrorCore/core/src/services/agent-exec.ts)
- [mcp.ts](file:///d:/Projects/MirrorCore/core/src/services/mcp.ts)
- [callMCPTool.ts](file:///d:/Projects/MirrorCore/mcp/src/callMCPTool.ts)
