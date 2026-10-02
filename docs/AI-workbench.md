可以，整理成一个比较成型的方案，基本已经能直接拿去让 Claude 开工了。

## AI Workbench：当前方案

核心目标不是做一个新的 IDE，而是做一个 **以项目状态为中心的 AI 多任务工作台**。

传统 IDE 的中心是：

```text
文件 / Repo / Editor / Terminal
```

这个 Workbench 的中心则是：

```text
Project
├─ 当前状态
├─ 项目文档
└─ AI 执行环境
```

重点解决的是：

> 同时维护多个 AI 项目时，快速知道每个项目做到哪、还有什么没做、AI 是否正在工作，并能随时切进去继续。

---

## 1. 首页

首页展示所有项目，每个项目是一张卡片。

建议字段：

```text
项目名称
项目概要

当前状态
已完成 / 未完成
Next Action
Last Updated

● AI Active
```

视觉类似：

```text
┌────────────────────────────┐
│ Data Forge              ●  │
│ 部门数据分析工具            │
│                            │
│ Doing                      │
│ 完成 7 / 未完成 3           │
│                            │
│ Next                       │
│ 验证 component tool         │
│                            │
│ Updated 10 min ago          │
└────────────────────────────┘
```

其中那个点主要表示：

```text
● = 该项目当前有 AI Session 活动
○ = 当前没有
```

以后如果有能力识别更多状态，可以扩展成：

```text
● Working
◉ Needs You
✓ Finished
○ Idle
```

但第一版不用复杂。

---

## 2. 点击项目后的页面

项目页基本三块。

```text
┌────────────────────────────────────┐
│ ← Projects        Data Forge       │
├────────────────────────────────────┤
│ Current State                      │
│                                    │
│ Doing · 7/10                       │
│ Next: validate component tool      │
├───────────────┬────────────────────┤
│ Documents     │ Markdown Preview   │
│               │                    │
│ ▼ docs        │ # Current status   │
│   status.md   │                    │
│   design.md   │ ...                │
│   decisions/  │                    │
│     xxx.md    │                    │
├───────────────┴────────────────────┤
│ AI / CLI                           │
│                                    │
│ Claude Code...                     │
└────────────────────────────────────┘
```

三块职责分别是：

```text
STATE
我现在做到哪里

CONTEXT
这个项目为什么是现在这样

EXECUTION
继续让 AI 做事
```

---

## 3. 项目状态：每项目一个 JSON

目前讨论下来，最合理的是：

**每个项目自己维护一个状态 JSON。**

例如：

```text
project/
├─ .workbench/
│  └─ state.json
├─ docs/
│  ├─ status.md
│  ├─ architecture.md
│  └─ decisions.md
└─ ...
```

`state.json` 建议固定 schema：

```json
{
  "schemaVersion": 1,
  "projectId": "data-forge",
  "name": "Data Forge",

  "status": "doing",
  "summary": "部门数据分析 PoC",

  "progress": {
    "completed": 7,
    "remaining": 3
  },

  "nextAction": "验证 component tool",
  "blockers": [],

  "updatedAt": "2026-10-03T08:00:00+09:00"
}
```

WorkBench 首页直接扫描这些 JSON。

因此：

```text
state.json
= 项目的机器可读当前状态
```

复杂背景不要塞进去。

---

## 4. Markdown = 人类可读 Context

复杂信息继续用 Markdown：

```text
docs/
├─ STATUS.md
├─ ARCHITECTURE.md
├─ PLAN.md
├─ DECISIONS.md
└─ ...
```

Workbench：

- 保留真实目录树
- 只显示 `.md`
- 点击以后浏览器直接渲染 Markdown
- 不做完整代码 Explorer
- 不做代码 Editor

这是一个刻意的限制。

因为这里不是 IDE。

代码让 Claude / Codex 管。

人主要看：

```text
状态
架构
计划
决策
上下文
```

---

## 5. 文件实时更新

本地 backend 使用：

```text
fs.watch
```

或者：

```text
chokidar
```

监听：

```text
state.json
docs/**/*.md
```

Claude 修改：

```text
state.json
```

首页立即刷新。

Claude 修改：

```text
architecture.md
```

如果当前正打开这份文档，Markdown Preview 自动刷新。

所以本地这一部分完全可以做到接近实时。

---

# 公司版

公司环境增加 Microsoft Lists。

但重要原则：

## JSON 是 Truth，Lists 是 Projection

即：

```text
Project A/state.json ─┐
Project B/state.json ─┼→ Workbench
Project C/state.json ─┘
          │
          ↓
       Sync
          ↓
Microsoft Lists
          ↓
Chat / Claude / Copilot
```

不让 JSON 和 Lists 同时成为真相源。

否则很容易：

```text
JSON: Doing

Lists: Waiting
```

最后不知道谁对。

---

## 6. Lists 的用途

Lists 不负责本地项目运行。

主要解决：

> Chat 想快速知道所有项目现状。

例如 Lists 保存：

```text
Project
Status
Summary
Completed
Remaining
Next Action
State Updated At
Synced At
```

然后 Chat 可以直接问：

> 我现在所有 active 项目分别什么状态？

不用手工复制 8 个 JSON。

复杂项目内容仍然留在：

```text
本地 Markdown
```

所以：

```text
Lists
→ 跨项目索引

Markdown
→ 单项目深入 Context
```

---

## 7. 同步方式

建议：

### 平时

`state.json` 修改后自动轻量同步 Lists。

```text
state.json changed
↓
Graph API
↓
更新对应 List Item
```

### 每天

再执行一次全量 reconcile：

```text
扫描全部项目 JSON
↓
与 Lists 对比
↓
重新校正
```

即：

```text
实时轻同步
+
每日全量校正
```

这样 Lists 不容易漂。

---

## 8. Microsoft Graph

公司 Workbench 读写 Lists：

```text
React
↓
Node backend
↓
Microsoft Graph API
↓
Microsoft Lists
```

Graph 只负责 M365 一侧。

本地：

```text
文件
Claude CLI
JSON
```

完全不需要经过 Graph。

未来还可以接：

```text
SharePoint
OneDrive
Outlook
Calendar
```

但第一版完全没必要。

---

# AI 执行部分

这里私人版和公司版可以不同。

## 私人

你已经觉得 Codex Desktop 很好用，所以没必要为了技术洁癖重新造一个 CLI 前端。

可以：

```text
AI Workbench
↓
状态 / 文档
↓
Open in Codex
```

所以私人 Workbench 重点是：

```text
跨项目认知层
```

Codex Desktop 负责：

```text
代码
diff
Agent
执行
```

两边分工。

---

## 公司

公司可以考虑嵌 Claude Code CLI。

技术栈：

```text
Browser
↓
xterm.js
↓
WebSocket
↓
Node
↓
node-pty
↓
Windows ConPTY
↓
Claude Code
```

是真 Terminal，不是假输出框。

---

# 9. CLI 返回首页以后不能关闭

这一点已经确定：

**CLI Session 属于 Project，不属于页面。**

所以：

```text
进入 Project A
↓
attach A session

返回首页
↓
detach UI
↓
Claude A 继续运行

进入 Project B
↓
attach B session

再进入 A
↓
reattach A
```

所以可以：

```text
Project A → Claude running
Project B → Claude running
Project C → Claude waiting
Project D → no session
```

同时存在。

这就是整个 AI Workbench 比 IDE 更关键的地方。

---

## 10. AI Session 管理

第一版建议：

```text
一个项目 = 一个主 AI Session
```

不要一开始就：

```text
Project A
├ Main
├ Research
├ Coding
├ Test
└ Review
```

否则马上变 session manager。

以后真有需要再扩展。

---

## 11. CLI 视觉

不一定要黑色 cmd 风格。

最简单可靠的方式仍然是 xterm.js，但 CSS 做成整个浏览器一致：

```text
Claude Code                ● Active

› Analyzing project state...

  Read    ARCHITECTURE.md
  Edit    api.ts

  Running tests...

› _
```

可以：

- 与网页相同背景
- 圆角
- 边框
- padding
- 柔和 ANSI 色
- 没有 Windows Terminal 外框
- 没有 `C:\xxx>` 那种传统视觉感

本质还是 Terminal，所以兼容性最好。

以后可以进一步做：

```text
Browser-native activity panel
+
可展开 Raw Terminal
```

但第一版不值得解析 Claude CLI 输出。

---

# 12. 首页真正重要的不是 Terminal，而是调度

WorkBench 最终想回答的是：

```text
哪些项目正在推进？
哪些项目卡住？
哪些 AI 正在工作？
哪些结果已经完成？
哪些需要我？
```

所以未来首页可能：

```text
Data Forge
7 / 10
● Working

Radar
14 / 15
◉ Needs you

Pokémon
8 / 12
○ Idle

AULOS
21 / 24
✓ Finished
```

这才是这个工具的核心。

---

# 13. 和普通 AI IDE 的区别

普通 AI IDE：

```text
Repo
→ Files
→ Editor
→ Agent
```

你的设计：

```text
Project
→ State
→ Context
→ Executor
```

AI 是 Executor。

今天可以是：

```text
Claude Code
```

明天：

```text
Codex
```

以后：

```text
Qwen
```

都无所谓。

项目状态不能跟着某个 AI Session 一起消失。

---

# 14. 私人 MVP

其实现在已经可以非常小：

```text
React + Vite
Node backend
JSON state
Markdown renderer
filesystem watcher
```

功能：

```text
① 首页扫描所有 project state.json

② 显示：
   项目概要
   已完成
   未完成
   Next
   Active

③ 点击进入项目

④ 显示 Current State

⑤ 显示 Markdown 文件树

⑥ 点击 md → 浏览器渲染

⑦ 文件变化 → 自动刷新
```

CLI/Codex 都可以先不做。

这已经能验证：

> “项目中心式 Workbench 到底是不是比现在找文件、找聊天记录舒服。”

---

# 15. 公司第二阶段

私人版验证交互以后，公司版加：

```text
Graph Adapter
↓
Lists sync

Terminal Adapter
↓
Claude Code
```

也就是：

```text
私人
Local JSON + Markdown + Codex Desktop

公司
Local JSON + Markdown + Lists + Claude Code
```

Workbench UI 基本不需要重做。

---

最后压缩成一句话，就是：

> **每个项目用 JSON 保存确定性状态，用 Markdown 保存上下文；Workbench 负责跨项目观察和下钻，公司再把 JSON 摘要同步到 Lists 供 Chat 查询，AI 执行环境则作为可替换的独立层。**

这个架构已经相当完整了。现在最值得先做的其实不是继续设计功能，而是把私人 MVP 跑起来，看看你一天真的会不会一直开着它。