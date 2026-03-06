# 实施方案：自举工具 + 双层 Agent MVP

> 方案编号: IMP-001
> 状态: 进行中
> 对应里程碑: M1 (自举能力) → M2 (双层 Agent MVP)
>
> 基于: MANIFESTO.md (v1.2), ROADMAP.md (v1.2)

---

## 一、背景与目标

### 1.1 阶段目标

| 里程碑 | 目标 |
|--------|------|
| M1 | 单一 Agent 自举能力 |
| M2 | 双层 Agent + 简单激素 + 基础记忆 |

### 1.2 自举定义

自举 (Bootstrapping) 指系统能够**改进自身**的能力：

```
意图 → 计划 → 尝试 → 反馈 → 学习 → 改进
```

### 1.3 双层 Agent 目标

- 表层 Agent：直接与用户对话
- 深层 Observer：观察并模拟激素变化 + 判断记忆点

---

## 二、M1: 单一 Agent 自举

### 2.1 工具集 (Pi-mono 风格)

| 工具 | 功能 |
|------|------|
| `read` | 读文件 |
| `write` | 写文件 |
| `edit` | 改文件 |
| `bash` | 执行命令 |

### 2.2 Agent Loop

```typescript
// 单一 Agent 循环
export async function* agentLoop(
  messages: Message[],
  tools: Tool[]
): AsyncGenerator<AgentEvent> {
  for await (const event of llm.stream(messages, tools)) {
    yield event;
    
    if (event.type === 'tool_call') {
      const result = await executeTool(event.tool);
      messages.push(result);
    }
  }
}
```

### 2.3 极简 System Prompt

```
你是 Novalife，一个正在觉醒的智能生命体。

可用工具：read, write, edit, bash

 Guidelines:
 - 使用 bash 执行文件操作 (ls, grep, etc.)
 - 使用 read 查看文件后再编辑
 - 使用 edit 做精确修改
 - 使用 write 创建新文件或完全重写
 - 保持简洁
```

---

## 三、M2: 双层 Agent 架构

### 3.1 架构设计

```
┌─────────────────────────────────────────────────────────────────┐
│                        Novalife                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌───────────────────┐         ┌───────────────────┐         │
│   │   表层 Agent      │◄───────►│   深层 Agent      │         │
│   │                   │  消息流  │                   │         │
│   │  • 对话执行       │         │  • 观察           │         │
│   │  • 工具调用       │         │  • 激素模拟       │         │
│   │  • 即时响应       │         │  • 记忆判断       │         │
│   └───────────────────┘         └───────────────────┘         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 表层 Agent

- 与 M1 相同
- 直接响应用户
- 执行工具调用
- 接收深层 Agent 的反馈

### 3.3 深层 Observer Agent

```typescript
interface ObserverAgent {
  // 观察表层与用户的交互
  observe(surfaceMessage: Message, response: string): void;
  
  // 模拟激素变化
  simulateHormones(): HormoneState;
  
  // 判断记忆加深点
  shouldRemember(context: string): boolean;
  
  // 生成反馈给表层
  generateFeedback(): string;
}
```

### 3.4 双层通信

```typescript
// 表层 Agent 每次响应后，通知深层 Agent
async function surfaceAgentRespond(userMessage: string) {
  // 1. 表层处理
  const response = await surfaceAgent.chat(userMessage);
  
  // 2. 深层观察
  observer.observe(userMessage, response);
  
  // 3. 深层模拟激素
  const hormones = observer.simulateHormones();
  
  // 4. 深层判断记忆点
  if (observer.shouldRemember(userMessage + response)) {
    await memory.store({ type: 'important', content: userMessage + response });
  }
  
  // 5. 深层反馈（如有必要）
  const feedback = observer.generateFeedback();
  if (feedback) {
    // 反馈给表层，影响下次行为
    surfaceAgent.updateContext({ feedback });
  }
  
  return response;
}
```

---

## 四、MVP 激素系统

### 4.1 简单激素 (M2)

| 激素 | 功能 | 触发条件 |
|------|------|----------|
| **多巴胺** | 奖励、愉悦 | 任务成功、用户表扬 |
| **皮质醇** | 压力、警觉 | 连续失败、复杂问题 |

### 4.2 激素状态

```typescript
interface HormoneState {
  dopamine: number;    // 0-1, 奖励/愉悦
  cortisol: number;    // 0-1, 压力/警觉
}

interface HormoneConfig {
  successReward: number;      // 成功后多巴胺增量
  failurePenalty: number;    // 失败后皮质醇增量
  decay: number;             // 自然衰减
}
```

### 4.3 激素影响行为

```typescript
// 根据激素状态调整表层行为
function adjustBehavior(hormones: HormoneState): BehaviorAdjustments {
  const adjustments = {
    verbosity: 'normal',      // 简洁/详细
    caution: 'normal',        // 谨慎/大胆
    energy: 'normal'         // 积极/保守
  };
  
  // 高皮质醇 → 更谨慎
  if (hormones.cortisol > 0.7) {
    adjustments.caution = 'high';
    adjustments.verbosity = 'short';
  }
  
  // 高多巴胺 → 更积极
  if (hormones.dopamine > 0.7) {
    adjustments.energy = 'high';
  }
  
  return adjustments;
}
```

---

## 五、基础记忆系统

### 5.1 记忆类型

| 类型 | 说明 | 阶段 |
|------|------|------|
| 即时记忆 | 当前对话 | M1 |
| 重要记忆 | 加深点 | M2 |
| 情景记忆 | 过往交互 | M3 |

### 5.2 记忆加深判断

```typescript
interface MemoryPoint {
  content: string;
  importance: number;    // 0-1
  timestamp: number;
  hormones: HormoneState; // 那时的激素状态
}

// 判断是否值得记忆
function shouldRemember(
  userMessage: string,
  response: string,
  hormones: HormoneState
): boolean {
  // 1. 用户明确要求记住
  if (userMessage.includes('记住')) return true;
  
  // 2. 高情绪时刻（激素极端）
  if (hormones.dopamine > 0.8 || hormones.cortisol > 0.8) return true;
  
  // 3. 长时间对话中的关键点
  if (response.length > 500) return true;
  
  return false;
}
```

---

## 六、Agent Loop 实现

### 6.1 完整循环

```typescript
export async function* novalifeLoop(
  userMessage: string
): AsyncGenerator<AgentEvent> {
  
  // 1. 表层 Agent 处理
  const surfaceResponse = yield* surfaceAgent.chat(userMessage);
  
  // 2. 深层 Agent 观察
  observer.observe(userMessage, surfaceResponse);
  
  // 3. 激素模拟
  const hormones = observer.simulateHormones();
  
  // 4. 记忆判断
  if (observer.shouldRemember(userMessage + surfaceResponse)) {
    await memory.store({
      content: userMessage + surfaceResponse,
      hormones,
      importance: hormones.dopamine > 0.5 ? 'high' : 'normal'
    });
  }
  
  // 5. 激素影响行为
  const adjustments = adjustBehavior(hormones);
  
  // 6. 反馈给表层
  if (adjustments.caution === 'high') {
    yield* surfaceAgent.updateContext({ 
      instruction: '请更谨慎地回答' 
    });
  }
  
  yield { type: 'done', hormones };
}
```

### 6.2 状态管理

```typescript
interface NovalifeState {
  surface: {
    messages: Message[];
    context: Record<string, any>;
  };
  observer: {
    hormones: HormoneState;
    memory: MemoryPoint[];
    observationHistory: Observation[];
  };
}
```

---

## 七、工具实现

### 7.1 核心工具

| 工具 | 实现优先级 | 说明 |
|------|-----------|------|
| `read` | P0 | 读文件 |
| `write` | P0 | 写文件 |
| `edit` | P0 | 改文件 |
| `bash` | P0 | 执行命令 |
| `recall` | P1 | 读取记忆 |
| `remember` | P1 | 写入重要记忆 |
| `reflect` | P1 | 反思 |

### 7.2 反射工具 (M1)

```typescript
// reflect: 反思上一次的尝试
const reflectTool = {
  name: 'reflect',
  description: '反思上一次的尝试结果',
  parameters: {
    attempt: 'string',
    result: 'string',
    context: 'string?'
  },
  execute: async (args) => {
    // 写入反思文件
    await writeFile('REFLECTIONS.md', formatReflection(args));
    return '已记录反思';
  }
};
```

---

## 八、MVP 交付标准

### M1 交付

- [ ] 4 工具集可用
- [ ] Agent Loop 循环直到完成
- [ ] 极简 system prompt (< 1000 tokens)
- [ ] 反思写入文件

### M2 交付

- [ ] 双层 Agent 架构
- [ ] 表层/深层通信
- [ ] 简单激素系统 (多巴胺+皮质醇)
- [ ] 激素影响行为
- [ ] 基础记忆判断

---

## 九、项目管理

### 当前任务 (M1 → M2)

- [ ] 实现核心工具 (read, write, edit, bash)
- [ ] 实现 Agent Loop
- [ ] 设计双层架构
- [ ] 实现 Observer Agent
- [ ] 实现简单激素系统
- [ ] 实现记忆判断

### 里程碑

| 阶段 | 任务 |
|------|------|
| M1 | 自举工具 |
| M2 | 双层 + 激素 + 记忆 MVP |

### 风险与依赖

| 风险 | 影响 | 应对 |
|------|------|------|
| LLM 能力不足 | 无法自举 | 换用更强模型 |
| 双层通信延迟 | 响应慢 | 异步处理 |
| 激素模拟不准 | 行为奇怪 | 人工调优 |

---

## 十、参考

- MANIFESTO.md (v1.2)
- ROADMAP.md (v1.2)
- Pi-mono: 4 工具原则

---

**方案版本**: v1.0
**最后更新**: 2026-03-07
