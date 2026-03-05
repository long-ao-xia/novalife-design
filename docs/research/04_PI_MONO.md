# 深度研究：Pi-mono 架构设计分析

> 日期: 2026-03-05
> 研究目标: 理解 Pi-mono 的核心理念

---

## 1. 核心理念

### 1.1 严格类型

Pi-mono (badlogic) 的核心特点:**严格类型**:

```typescript
// 绝不使用 any
// 必须有完整类型定义
interface AgentMessage {
  id: string;
  role: 'user' | 'assistant' | 'tool';
  content: string;
  timestamp: number;
}
```

从 AGENTS.md:
> - No `any` types unless absolutely necessary
> - Check node_modules for external API type definitions instead of guessing
> - NEVER use inline imports

### 1.2 模块化架构

```
packages/
├── ai/           # LLM 抽象层
├── agent/        # Agent 核心
├── tui/          # 终端界面
├── web-ui/       # 网页界面
├── coding-agent/ # 编码专用
├── mom/          # 消息管理
└── pods/         # ???
```

每个 package:
- 独立版本 (lockstep)
- 独立 CHANGELOG
- 可独立测试

### 1.3 多 Provider 支持

Pi-mono 的 `ai` 包支持:
- OpenAI
- Anthropic
- Google
- Ollama
- OpenRouter
- 等等

通过统一的流式接口:
```typescript
stream('openai/gpt-4', messages, tools)
stream('anthropic/claude-3', messages, tools)
```

---

## 2. 架构详解

### 2.1 Agent Loop

```typescript
// packages/agent/src/agent-loop.ts
export async function* agentLoop(
  agent: Agent,
  messages: Message[],
  tools: Tool[]
): AsyncGenerator<AgentEvent> {
  // 1. 构建 prompt
  const prompt = buildPrompt(agent, messages);
  
  // 2. 流式调用 LLM
  for await (const event of llm.stream(prompt, tools)) {
    // 3. 处理事件
    yield event;
    
    // 4. 如果有工具调用
    if (event.type === 'tool_call') {
      const result = await executeTool(event.tool);
      messages.push(result);
    }
  }
}
```

### 2.2 Model Runner

```typescript
// packages/ai/src/
// 统一的模型运行器
export async function* stream(
  api: Api,
  messages: Message[],
  options: StreamOptions
): AsyncGenerator<StreamEvent> {
  // 根据 api 类型选择 provider
  const provider = getProvider(api);
  
  // 流式返回
  for await (const chunk of provider.stream(messages)) {
    yield parseChunk(chunk);
  }
}
```

---

## 3. 关键创新

### 3.1 Lockstep Versioning

所有 packages 同步版本:

```bash
npm run release:patch  # 全部更新 patch
npm run release:minor  # 全部更新 minor
```

### 3.2 多 LLM 流式接口

统一的接口，不同的 provider:

```typescript
// 添加新 provider 只需
// 1. 在 providers/ 添加文件
// 2. 在 stream.ts 注册
// 3. 在 types.ts 添加类型
```

### 3.3 严格的 Git 规则

> - NEVER use `git add -A`
> - ONLY commit files YOU changed
> - 每个 PR 只改自己的文件

---

## 4. 对 Novalife 的启示

### 4.1 可以借鉴的

| 特性 | Novalife 可以借鉴吗? |
|------|---------------------|
| 严格类型 | ✅ 核心原则 |
| 模块化 | ✅ 独立 package |
| 统一流式接口 | ✅ 支持多 LLM |
| Lockstep 版本 | ⚠️ 可能过度设计 |

### 4.2 关键问题

**TypeScript vs Python**

NanoClaw: TypeScript
Bub: Python
Pi-mono: TypeScript

**选择 TypeScript 的理由**:
- 与 OpenClaw 技能系统兼容
- 类型安全
- 生态丰富

### 4.3 架构建议

```typescript
// packages/novalife-core
// - Agent Loop
// - State (激素)
// - Memory

// packages/novalife-channels
// - Telegram
// - Discord
// - etc.

// packages/novalife-tools
// - 工具注册
// - 执行
```

---

## 5. 总结

Pi-mono 的核心价值:
- **类型即文档**: 严格类型让代码自解释
- **模块化**: 清晰的边界
- **可测试**: 每个模块独立

**对 Novalife**:
1. 采用 TypeScript
2. 严格类型约束
3. 模块化设计

这与 Novalife 的"简单优先"理念一致——清晰的类型让代码更容易理解。
