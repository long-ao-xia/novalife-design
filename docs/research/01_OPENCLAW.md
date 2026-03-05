# 深度研究：OpenClaw 架构设计分析

> 日期: 2026-03-05
> 研究目标: 理解 OpenClaw 的核心理念，为 Novalife 设计提供参考

---

## 1. 核心理念

### 1.1 Session 隔离模型

OpenClaw 的核心设计是 **Session 隔离**：

- **每个对话/群组 = 独立 Session**
- Session 拥有独立的：
  - 消息历史 (transcript)
  - 记忆文件
  - 上下文窗口
  - 文件系统 (可选沙盒)

**Session 隔离的代价**:
- 上下文不共享
- 每个 Session 需要独立管理
- 无法"一心多用"

### 1.2 Gateway 架构

```
┌─────────────────────────────────────────┐
│              Gateway                    │
│  (唯一的消息入口 + 状态管理)             │
├─────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────┐│
│  │Telegram │  │WhatsApp │  │ Discord ││
│  │ Channel │  │ Channel │  │ Channel ││
│  └────┬────┘  └────┬────┘  └────┬────┘│
│       │            │            │       │
│       └────────────┼────────────┘       │
│                    ▼                    │
│            ┌───────────────┐             │
│            │  Message      │             │
│            │  Router       │             │
│            └───────┬───────┘             │
│                    ▼                     │
│            ┌───────────────┐             │
│            │  Session     │             │
│            │  Manager     │             │
│            └───────┬───────┘             │
│                    ▼                     │
│            ┌───────────────┐             │
│            │  Agent Loop   │             │
│            │  (per-session)│             │
│            └───────────────┘             │
└─────────────────────────────────────────┘
```

### 1.3 Agent Loop 生命周期

```
消息 → Context Assembly → Model Inference → Tool Execution → Streaming Reply → Persistence
```

每个 Session 的 Agent Loop 是**序列化**的，保证状态一致性。

### 1.4 记忆系统

- **Daily Log**: `memory/YYYY-MM-DD.md` (每天的日志)
- **Long-term Memory**: `MEMORY.md` (精选的长期记忆)
- **自动 flush**: 当 session 接近 compaction 时，会提醒模型写记忆

### 1.5 Hook 系统

OpenClaw 有两套 Hook:
- **Internal Hooks**: 生命周期事件
- **Plugin Hooks**: Agent 循环中的拦截点

---

## 2. 关键创新

### 2.1 Compaction (上下文压缩)

当 token 接近限制时：
1. 先触发 memory flush (提醒模型写记忆)
2. 然后压缩历史消息
3. 保留关键信息

### 2.2 Skills 系统

- 基于 `SKILL.md` 文件
- 运行时加载
- 支持自注册

### 2.3 多渠道统一

- 一个 Gateway 管理所有渠道
- 消息路由统一处理

---

## 3. 对 Novalife 的启示

### 3.1 可以借鉴的

| 特性 | Novalife 可以借鉴吗? |
|------|---------------------|
| Gateway 架构 | ✅ 简化版 |
| Skills 系统 | ✅ 直接采用 |
| Hook 机制 | ✅ 简化采用 |
| Memory flush | ❌ 无 session，不需要 |

### 3.2 需要放弃的

- **Session 隔离** — Novalife 追求"单一心智"
- **多 Session 管理** — 简化为单一上下文
- **Compaction** — 单一上下文持续增长，用记忆系统解决

---

## 4. 总结

OpenClaw 是一个**成熟的多用户、多会话**系统。它的设计是为了服务多个用户，每个用户有独立的上下文。

**Novalife 的不同点**: 我们追求"像人一样"——一个人只有一个大脑，所有事情在一个大脑里处理。

**关键问题**: 如何在单一心智模型下，管理持续增长的上下文？
- 方案1: 记忆系统 (长期记忆 + 工作记忆)
- 方案2: 注意力机制 (只关注当前重要的)
- 方案3: 定期摘要 (定期压缩)
