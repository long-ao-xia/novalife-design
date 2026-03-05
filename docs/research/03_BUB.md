# 深度研究：Bub 架构设计分析

> 日期: 2026-03-05
> 研究目标: 理解 Bub 的核心理念

---

## 1. 核心理念

### 1.1 Tape Storage

Bub 的核心创新是 **Tape Storage** (磁带存储):

> "An append-only log that serves as the single source of truth for agent state."

```
┌─────────────────────────────────────────┐
│              Tape (Append-only)          │
├─────────────────────────────────────────┤
│  ┌─────────────────────────────────────┐ │
│  │ Message 1                           │ │
│  ├─────────────────────────────────────┤ │
│  │ Message 2                           │ │
│  ├─────────────────────────────────────┤ │
│  │ Tool Call 1                         │ │
│  ├─────────────────────────────────────┤ │
│  │ Tool Result 1                       │ │
│  ├─────────────────────────────────────┤ │
│  │ ... (持续追加)                       │ │
│  └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

### 1.2 Anchor (锚点)

Bub 引入了 **Anchor** 概念:

- 在 Tape 上标记重要位置
- 可以"跳转"到锚点
- 支持上下文恢复

```python
# 类似书签
anchor = tape.create_anchor("important-decision")
# 后续可以恢复到该位置
tape.seek_to(anchor)
```

### 1.3 无 Session 隔离

Bub 和 Novalife 一样，**不区分 Session**!

> "All messages go to the same tape."

但 Bub 的实现方式不同:
- 用 Tape 存储所有历史
- 用 Anchor 标记不同"会话"
- 可以随时恢复到任意锚点

---

## 2. 架构详解

### 2.1 核心模块

| 模块 | 目录 | 职责 |
|------|------|------|
| core | `src/bub/core/` | Agent loop, Router, Model runner |
| tape | `src/bub/tape/` | 磁带存储服务 |
| tools | `src/bub/tools/` | 工具注册 |
| skills | `src/bub/skills/` | 技能发现 |
| channels | `src/bub/channels/` | 消息通道 |

### 2.2 Agent Loop

```python
# 简化版
async def agent_loop(tape: TapeService, model: ModelRunner):
    while True:
        # 从 tape 读取最新消息
        messages = tape.read()
        
        # 发送给模型
        response = await model.complete(messages)
        
        # 写入 tape
        tape.append(response)
        
        # 如果有工具调用，执行并追加结果
        if response.tool_calls:
            for tool in response.tool_calls:
                result = await tool.execute()
                tape.append(result)
```

### 2.3 Router

Bub 的 Router 负责:
- 检测命令 (`/start`, `/stop` 等)
- 格式化输入/输出
- 管理多渠道

---

## 3. 关键创新

### 3.1 Tool Progressive Rendering

工具结果不是一次性显示，而是**渐进式渲染**:

```
┌─────────────────────────────────────┐
│ Thinking...                          │
├─────────────────────────────────────┤
│ ┌─────────────────────────────────┐ │
│ │ Tool: search                    │ │
│ │ Progress: ████████░░ 80%        │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘
```

### 3.2 FileTapeStore

- 底层存储是文件 (JSONL)
- 支持大上下文
- 可追溯、可调试

### 3.3 命令检测

```python
# 自动检测命令
if message.startswith('/'):
    command = parse_command(message)
    # 执行命令，不走 Agent
```

---

## 4. 对 Novalife 的启示

### 4.1 可以借鉴的

| 特性 | Novalife 可以借鉴吗? |
|------|---------------------|
| Tape Storage | ✅ 核心存储机制 |
| Anchor | ✅ 可用于标记会话 |
| 无 Session | ✅ 与 Novalife 一致 |
| 命令检测 | ✅ 简化输入处理 |

### 4.2 关键差异

| Bub | Novalife |
|-----|----------|
| Python | TypeScript (可能) |
| 单一 Tape | 可能有多个记忆层 |
| 无激素系统 | 有激素系统 |

### 4.3 核心问题

**Tape vs 记忆系统**

Bub 的 Tape = 完整的消息历史
Novalife 需要:
- Tape (完整历史)
- 记忆 (长期存储)
- 激素状态 (动态)

**整合方案**:
```
输入 → Router → [Tape: 完整历史]
              → [Memory: 长期记忆]
              → [Hormones: 当前状态]
              → [LLM: 推理]
```

---

## 5. 总结

Bub 的 Tape Storage 是**非常好的灵感**:
- 简单: 追加写
- 可追溯: 完整历史
- 高效: 不需要复杂索引

**对 Novalife 的建议**:
1. 采用 Tape 作为核心存储
2. 用 Anchor 标记不同上下文
3. 叠加记忆层和激素层

这与 Novalife 的"单一心智"理念非常契合。
