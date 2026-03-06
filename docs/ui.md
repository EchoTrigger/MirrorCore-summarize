# 前端结构与 UI 规范

## 现状概览
当前前端位于 `ui/src`，使用 React + Vite。入口为 `main.tsx`，布局在 `App.tsx`。核心聊天区与设置弹窗已完成拆分，主要功能分布在 `components`、`hooks`、`store`、`api` 与 `styles`。

## 当前目录结构
```
ui/
└── src/
    ├── api/
    │   └── index.ts
    ├── assets/
    │   └── react.svg
    ├── components/
    │   ├── chat/
    │   │   ├── ChatArea.tsx
    │   │   ├── InputComposer.tsx
    │   │   ├── MarkdownRenderer.tsx
    │   │   ├── MessageItem.tsx
    │   │   └── MessageList.tsx
    │   ├── panels/
    │   │   ├── EnvStatusPanel.tsx
    │   │   ├── ReplayPanel.tsx
    │   │   └── TerminalPanel.tsx
    │   ├── settings/
    │   │   ├── SettingsChat.tsx
    │   │   ├── SettingsGeneral.tsx
    │   │   ├── SettingsMemory.tsx
    │   │   └── SettingsPlaner.tsx
    │   ├── ui/
    │   │   ├── button.tsx
    │   │   ├── dialog.tsx
    │   │   ├── input.tsx
    │   │   ├── label.tsx
    │   │   ├── select.tsx
    │   │   ├── slider.tsx
    │   │   ├── switch.tsx
    │   │   ├── tabs.tsx
    │   │   └── textarea.tsx
    │   ├── SettingsModal.tsx
    │   └── Sidebar.tsx
    ├── hooks/
    │   ├── useEnvSessions.ts
    │   ├── useSocket.ts
    │   └── useTerminalSessions.ts
    ├── lib/
    │   └── utils.ts
    ├── store/
    │   └── appStore.ts
    ├── styles/
    │   ├── base.css
    │   ├── chat.css
    │   ├── forms.css
    │   ├── layout.css
    │   └── panels.css
    ├── types/
    │   ├── chat.ts
    │   └── settings.ts
    ├── App.tsx
    ├── App.css
    ├── index.css
    └── main.tsx
```

## 模块职责
- `main.tsx` 负责挂载根组件
- `App.tsx` 负责整体布局与页面组合
- `components/chat` 承载聊天主区域、消息展示与输入相关 UI
- `components/panels` 承载环境状态、回放与终端面板
- `components/settings` 承载设置弹窗的各分区内容
- `components/ui` 复用基础 UI 组件
- `hooks` 管理 Socket 连接、环境会话与终端会话逻辑
- `store` 负责全局状态与持久化
- `styles` 负责按领域拆分的样式文件
- `types` 负责前端内部类型定义

## 交互规范
- 侧边栏滑动交互不绑定上键与下键
- 侧边栏滑动仅处理左右方向与指针/触控输入

## 圆角规范
- 全局圆角变量定义于 `ui/src/styles/base.css` 的 `:root`
- 通用圆角使用 `--radius-sm/md/lg/xl`（当前均为 12px）与 `--radius-full`
- 输入框气泡使用 `.input-wrapper` 的 `border-radius: 33px` 与 `min-height: 65px`

## 示例片段
```ts
import { API } from '../api';

const conversations = await API.getConversations();
```

## 本仓库依赖
- [main.tsx](file:///d:/Projects/MirrorCore/ui/src/main.tsx)
- [App.tsx](file:///d:/Projects/MirrorCore/ui/src/App.tsx)
- [ChatArea.tsx](file:///d:/Projects/MirrorCore/ui/src/components/chat/ChatArea.tsx)
- [Sidebar.tsx](file:///d:/Projects/MirrorCore/ui/src/components/Sidebar.tsx)
- [SettingsModal.tsx](file:///d:/Projects/MirrorCore/ui/src/components/SettingsModal.tsx)
- [useSocket.ts](file:///d:/Projects/MirrorCore/ui/src/hooks/useSocket.ts)
- [useEnvSessions.ts](file:///d:/Projects/MirrorCore/ui/src/hooks/useEnvSessions.ts)
- [useTerminalSessions.ts](file:///d:/Projects/MirrorCore/ui/src/hooks/useTerminalSessions.ts)
- [appStore.ts](file:///d:/Projects/MirrorCore/ui/src/store/appStore.ts)
- [App.css](file:///d:/Projects/MirrorCore/ui/src/App.css)
- [index.css](file:///d:/Projects/MirrorCore/ui/src/index.css)
- [styles](file:///d:/Projects/MirrorCore/ui/src/styles)
