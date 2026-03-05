# 深度研究：NanoClaw 架构设计分析

> 日期: 2026-03-05
> 研究目标: 理解 NanoClaw 的核心理念

---

## 1. 核心理念

### 1.1 "小到能理解"

NanoClaw 的 slogan:
> "Small enough to understand. One process, a few source files."

创始人 (qwibitai) 的核心观点:
- OpenClaw 太大 (50万行代码，53个配置文件)
- 安全只是应用层隔离，不是真正的 OS 级别隔离
- 一切都运行在共享内存的 Node 进程中

**NanoClaw 的回答**:
- 代码量足够小，一个人能完全理解
- Agent 运行在独立的 Linux 容器中
- 容器级别的文件系统隔离

### 1.2 容器隔离模型

```
┌─────────────────────────────────────────────┐
│              Host Machine                     │
├─────────────────────────────────────────────┤
│  NanoClaw (Node.js)                         │
│  ├── 消息路由                               │
│  ├── 数据库 (SQLite)                        │
│  └── IPC 监听                              │
│         │                                    │
│         │ spawn container                    │
│         ▼                                    │
│  ┌─────────────────────────────────────┐    │
│  │     Linux Container                  │    │
│  │  ┌─────────────────────────────┐    │    │
│  │  │   Claude Agent (in container)│    │    │
│  │  │   - 文件系统: 仅挂载的目录   │    │    │
│  │  │   - 无 Host 访问权限        │    │    │
│  │  └─────────────────────────────┘    │    │
│  └─────────────────────────────────────┘    │
│                                              │
│  每个 Group = 独立容器                       │
│  容器之间完全隔离                            │
└─────────────────────────────────────────────┘
```

### 1.3 Group 隔离

- 每个 Group 有独立的:
  - 文件系统 (挂载的 folder)
  - CLAUDE.md (记忆文件)
  - 容器环境

- Main channel (私聊) 有特殊权限:
  - 可以访问项目根目录 (只读)
  - 可以修改自己的 group folder

### 1.4 技能系统

- 不是内置功能，而是通过**技能安装**
- 例如: `/add-telegram`, `/add-whatsapp`
- 技能 = 代码修改 (不是配置)

---

## 2. 架构详解

### 2.1 核心模块

| 模块 | 文件 | 职责 |
|------|------|------|
| 入口 | `src/index.ts` | 状态管理、消息循环、Agent 调用 |
| 路由 | `src/router.ts` | 消息格式化、输出路由 |
| 频道注册 | `src/channels/registry.ts` | 自注册启动 |
| 容器运行 | `src/container-runner.ts` | 启动/管理容器 |
| 数据库 | `src/db.ts` | SQLite 操作 |
| 任务调度 | `src/task-scheduler.ts` | 定时任务 |

### 2.2 消息流程

```
消息进入 → 检查白名单 → 路由到 Group Queue 
         → 等待轮到该 Group → 启动容器 
         → 运行 Agent → 返回结果 → 清理容器
```

### 2.3 容器安全

- **只读挂载**: 项目根目录是只读的，防止 Agent 修改源码
- **Secret 注入**: 通过 stdin 传递，不通过 .env 文件
- **IPC**: 容器内 Agent 通过 IPC 与 Host 通信

---

## 3. 关键创新

### 3.1 Agent Swarms (多 Agent 协作)

> "Spin up teams of agents that collaborate in your chat."

这是 NanoClaw 首创的功能——一个 Group 可以启动多个 Agent 协作。

### 3.2 容器按需启动

- 不是长期运行的 Agent
- 每次消息启动一个容器
- 容器超时自动清理

### 3.3 Group Folder 模式

```
groups/
├── main/              # 主对话
│   ├── CLAUDE.md     # 记忆
│   ├── logs/         # 日志
│   └── (挂载到容器)
├── project-a/         # 项目A
│   └── ...
└── project-b/         # 项目B
    └── ...
```

---

## 4. 与 OpenClaw 的对比

| 特性 | OpenClaw | NanoClaw |
|------|----------|----------|
| 代码量 | 50万行 | 几千行 |
| 隔离级别 | 应用层 | 容器层 |
| Session 管理 | 多 Session | 每个 Group 独立容器 |
| 复杂度 | 高 | 低 |
| 可理解性 | 难 | 易 |

---

## 5. 对 Novalife 的启示

### 5.1 可以借鉴的

| 特性 | Novalife 可以借鉴吗? |
|------|---------------------|
| 容器隔离 | ✅ 可选 (安全优先时) |
| Group Folder | ❌ 无 Group，不需要 |
| 技能安装 | ✅ 类似思路 |
| 按需启动 | ✅ 类似思路 |

### 5.2 需要放弃的

- **Group/容器隔离** — Novalife 追求单一心智
- **每个 Group 独立文件系统** — 单一文件系统

### 5.3 关键问题

**如果不用容器隔离，如何保证安全？**

NanoClaw 的答案是: 容器。
但 Novalife 的问题是: 我们追求"像人一样"，人没有容器隔离。

**可能的答案**:
1. 信任模型: 像信任自己一样信任 Agent
2. 能力限制: 不给危险工具
3. 操作审计: 记录所有操作，可回滚

---

## 6. 总结

NanoClaw 回答了 "如何让 AI Agent 更安全" 这个问题——通过容器隔离。

但 Novalife 问的是不同的问题: "如何让 AI Agent 像人一样思考和行动?"

**核心理念冲突**:
- NanoClaw: 隔离 = 安全
- Novalife: 统一 = 像人

这不矛盾——我们可以:
- 内部是单一心智 (像人)
- 外部可选择容器隔离 (安全)
