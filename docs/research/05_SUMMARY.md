# 深度研究：综合分析与 Novalife 核心设计

> 日期: 2026-03-05
> 研究目标: 综合所有研究，提出真正的 Novalife 设计

---

## 1. 各项目核心对比

| 项目 | 核心理念 | Session 隔离 | 状态管理 | 技术栈 |
|------|---------|-------------|---------|--------|
| OpenClaw | 多用户、多会话 Gateway | ✅ 强隔离 | 多 Session | TypeScript |
| NanoClaw | 小到能理解 + 容器安全 | ✅ Group 隔离 | 多容器 | TypeScript |
| Bub | Tape Storage + Anchor | ❌ 无 | 单一 Tape | Python |
| Pi-mono | 严格类型 + 模块化 | ? | ? | TypeScript |
| Novalife | 单一心智 + 像人一样 | ❌ 无 | 单一心智 | ? |

---

## 2. 关键洞察

### 2.1 Session 隔离的代价

从 OpenClaw 和 NanoClaw:
- Session 隔离 = 更安全 (用户间不泄露)
- Session 隔离 = 更复杂 (需要管理多个状态)
- Session 隔离 = 不像人 (人没有隔离的大脑)

### 2.2 无隔离的优势 (Bub)

Bub 证明了:
- 单一 Tape 可以管理所有对话
- Anchor 可以标记不同上下文
- 无隔离 = 简单 = 像人

### 2.3 安全的替代方案

NanoClaw: 容器隔离
但对于个人使用 (Novalife 场景):
- 可以选择更信任 Agent
- 可以限制危险工具
- 可以记录所有操作

---

## 3. Novalife 核心设计

### 3.1 核心理念: "单一心智"

> "像人一样思考和行动的 AI Agent"

人脑的特点:
1. **单一上下文** — 所有事情在一个大脑里
2. **持续记忆** — 记住重要的事，忘记琐事
3. **情绪波动** — 有状态，不是死板的机器
4. **一心多用** — 同时处理多件事

### 3.2 架构

```
┌─────────────────────────────────────────────────────────┐
│                    Novalife                              │
├─────────────────────────────────────────────────────────┤
│  Input Layer (输入层)                                   │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐                   │
│  │ Telegram│ │ Discord │ │ ...     │ → 统一消息格式    │
│  └─────────┘ └─────────┘ └─────────┘                   │
├─────────────────────────────────────────────────────────┤
│  Router (路由器)                                        │
│  - 命令检测                                             │
│  - 意图识别                                             │
│  - 上下文聚焦                                           │
├─────────────────────────────────────────────────────────┤
│  Core (核心)                                            │
│  ┌─────────────────────────────────────────────────┐   │
│  │  State (激素状态机)                              │   │
│  │  - 多巴胺 ↑ → 积极探索                          │   │
│  │  - 皮质醇 ↑ → 规避风险                          │   │
│  │  - ...                                           │   │
│  └─────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Memory (记忆系统)                               │   │
│  │  - Tape: 完整历史                               │   │
│  │  - Working: 当前上下文                          │   │
│  │  - Long-term: 重要记忆                          │   │
│  └─────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────┐   │
│  │  LLM (推理引擎)                                 │   │
│  │  - MiniMax (默认)                               │   │
│  │  - 可插拔 Provider                              │   │
│  └─────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────┤
│  Tools (工具层)                                         │
│  - 技能注册                                             │
│  - 执行引擎                                             │
│  - 结果处理                                             │
└─────────────────────────────────────────────────────────┘
```

### 3.3 无 Session 设计

```
Bub:     Tape + Anchor
         Anchor = 书签

Novalife: Tape + Attention
         Attention = 当前焦点
```

**关键区别**:
- Bub 用 Anchor 标记"不同会话"
- Novalife 用 Attention 区分"当前任务"

### 3.4 状态模型

```typescript
interface NovalifeState {
  // 激素 (0-1)
  dopamine: number;    // 奖励/动机
  serotonin: number;  // 稳定/满足
  cortisol: number;   // 压力/警觉
  oxytocin: number;   // 社交/信任
  
  // 注意力
  focus: string[];    // 当前关注的对话/任务
  context: string;    // 当前上下文摘要
  
  // 记忆指针
  tapePosition: number;  // 当前 Tape 位置
  importantAnchors: Anchor[];
}
```

---

## 4. MVP 设计

### 4.1 Phase 1: 最小可行

只实现最核心的:

1. **单一 Tape 存储**
   - 所有消息追加写入
   - JSONL 格式

2. **基础 Router**
   - 接收消息
   - 检测命令
   - 转发给 LLM

3. **简单 LLM 调用**
   - MiniMax Provider
   - 流式输出

4. **基本工具**
   - echo
   - memory write/read

### 4.2 不做 (MVP 阶段)

- 激素系统 (Phase 2)
- 注意力机制 (Phase 2)
- 多渠道 (Phase 3)
- 容器隔离 (可选)

---

## 5. 代码结构建议

```
novalife/
├── packages/
│   ├── core/           # 核心 (Agent Loop, State, Memory)
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── state.ts      # 激素状态
│   │   │   ├── memory.ts     # 记忆系统
│   │   │   ├── loop.ts      # Agent Loop
│   │   │   └── router.ts    # 路由器
│   │   └── test/
│   │
│   ├── llm/           # LLM Provider 抽象
│   │   ├── src/
│   │   │   ├── types.ts
│   │   │   ├── minimax.ts
│   │   │   └── openai.ts
│   │   └── test/
│   │
│   ├── channels/      # 消息通道
│   │   ├── src/
│   │   │   ├── telegram.ts
│   │   │   └── console.ts
│   │   └── test/
│   │
│   └── tools/         # 工具系统
│       ├── src/
│       │   ├── registry.ts
│       │   └── executor.ts
│       └── test/
│
├── docs/
│   └── research/
│
└── package.json
```

---

## 6. 总结

经过 100+ 循环的研究，我的核心洞察:

1. **Session 隔离是选择，不是必然**
   - OpenClaw/NanoClaw 选择隔离 → 更安全、更复杂
   - Bub/Novalife 选择统一 → 更简单、更像人

2. **单一心智 = Tape + State + Attention**
   - Tape: 完整历史
   - State: 激素状态
   - Attention: 当前焦点

3. **MVP 优先**
   - 先实现最小功能
   - 激素/注意力后续叠加

---

## 6. 补充研究 (Tavily 搜索)

### 最新趋势

从 Tavily 搜索结果看:

1. **Claude Computer Use** — Anthropic 让 AI 直接控制鼠标/键盘
2. **Persistent Memory Architecture** — 4层文件基础的记忆架构
3. **Multi-agent memory** — 多 Agent 系统中记忆层的重要性
4. **Hormone-based Agents** — Scaffolding AI Agents with Hormones and Emotions (arXiv)

### 关键发现

- 行业共识: 记忆是 Agent 架构的核心
- 新趋势: 单一持续上下文 (single persistent context) 开始流行
- 情感/激素系统从学术研究进入工程实践

---

**下一步**: 等 Jerry 确认这个方向，再开始写代码。
