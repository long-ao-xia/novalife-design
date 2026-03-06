# Novalife 自举工具 MVP 设计

> 版本: v1.0
> 日期: 2026-03-06
> 状态: 初稿

---

## 一、背景与目标

### 1.1 什么是自举？

自举 (Bootstrapping) 指一个系统能够**改进自身**的能力。

对于 Novalife，自举不是"自己改代码"那么科幻，而是建立一个**自我改进的循环**：

```
意图 → 计划 → 尝试 → 反馈 → 学习 → 改进
```

### 1.2 参考：Pi-mono 的 Agent Loop

Pi-mono 的核心理念体现在严格的类型定义和模块化架构：

```typescript
// Pi-mono Agent Loop 核心模式
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
    
    // 4. 工具调用
    if (event.type === 'tool_call') {
      const result = await executeTool(event.tool);
      messages.push(result);
    }
  }
}
```

**Pi-mono 关键原则**：
- **严格类型**：绝不使用 `any`
- **模块化**：每个功能独立包
- **流式优先**：事件驱动

---

## 二、自举 Agent Loop 设计

### 2.1 核心循环

```
┌─────────────────────────────────────────────────────────────────┐
│                     Novalife Agent Loop                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────┐                                              │
│   │ 感知输入    │  ── 用户消息/自我检测/定时触发                 │
│   └──────┬──────┘                                              │
│          │                                                      │
│          ▼                                                      │
│   ┌─────────────┐                                              │
│   │ 意图识别    │  ── 用户想做什么？我需要改进什么？             │
│   └──────┬──────┘                                              │
│          │                                                      │
│          ▼                                                      │
│   ┌─────────────┐                                              │
│   │ 计划制定    │  ── 需要哪些工具？执行顺序？                   │
│   └──────┬──────┘                                              │
│          │                                                      │
│          ▼                                                      │
│   ┌─────────────┐     ┌─────────────┐                          │
│   │ 执行工具    │────▶│ 验证结果    │                          │
│   └──────┬──────┘     └──────┬──────┘                          │
│          │                   │                                 │
│          ▼                   ▼                                 │
│   ┌─────────────┐     ┌─────────────┐                          │
│   │ 记录结果    │     │ 反思学习    │                          │
│   └──────┬──────┘     └──────┬──────┘                          │
│          │                   │                                 │
│          └─────────┬─────────┘                                 │
│                    ▼                                           │
│            ┌─────────────┐                                      │
│            │  更新状态   │  ── 激素/记忆/信心                    │
│            └─────────────┘                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 类型定义 (严格类型风格)

```typescript
// 基础类型 - 遵循 Pi-mono 严格类型原则
interface AgentEvent {
  type: 'text' | 'tool_call' | 'tool_result' | 'error' | 'done';
  content?: string;
  tool_call?: ToolCall;
  timestamp: number;
}

interface ToolCall {
  id: string;
  name: string;
  arguments: Record<string, unknown>;
}

interface ExecutionResult {
  success: boolean;
  output: string;
  error?: string;
  duration: number;
}

interface Reflection {
  whatHappened: string;
  whatWorked: string;
  whatFailed: string;
  learning: string;
  nextAttempt: string;
}

// 自举循环状态
interface BootstrapState {
  intent: string | null;
  plan: string[];
  attempts: Attempt[];
  confidence: number;  // 0-1, 影响下次尝试意愿
}

interface Attempt {
  id: string;
  tools: ToolCall[];
  result: ExecutionResult;
  reflection: Reflection;
}
```

---

## 三、MVP 工具设计

### 3.1 工具列表

| 工具名 | 功能 | 优先级 |
|--------|------|--------|
| `read_file` | 读取文件内容 | P0 |
| `write_file` | 写入/修改文件 | P0 |
| `list_dir` | 列出目录内容 | P0 |
| `execute` | 执行命令/代码 | P0 |
| `git_diff` | 查看 Git 变更 | P1 |
| `git_commit` | 提交更改 | P1 |
| `test` | 运行测试 | P1 |
| `reflect` | 反思学习 | P2 |

### 3.2 P0: 基础文件系统

#### read_file

```typescript
interface ReadFileInput {
  path: string;           // 文件路径 (绝对或相对)
  offset?: number;        // 起始行 (默认 1)
  limit?: number;         // 行数限制
}

interface ReadFileOutput {
  success: true;
  content: string;
  lines: number;
} | {
  success: false;
  error: string;
}
```

#### write_file

```typescript
interface WriteFileInput {
  path: string;           // 文件路径
  content: string;        // 新内容
  mode: 'overwrite' | 'append';
}

interface WriteFileOutput {
  success: true;
  path: string;
  lines: number;
} | {
  success: false;
  error: string;
}
```

#### list_dir

```typescript
interface ListDirInput {
  path: string;           // 目录路径
  recursive?: boolean;     // 是否递归
}

interface ListDirOutput {
  success: true;
  entries: Array<{
    name: string;
    type: 'file' | 'directory';
    path: string;
  }>;
} | {
  success: false;
  error: string;
}
```

### 3.3 P0: 代码执行

#### execute

```typescript
interface ExecuteInput {
  command: string;         // 要执行的命令
  cwd?: string;           // 工作目录
  timeout?: number;       // 超时毫秒 (默认 30000)
  env?: Record<string, string>;  // 环境变量
}

interface ExecuteOutput {
  success: boolean;
  stdout: string;
  stderr: string;
  exitCode: number;
  duration: number;
}
```

**安全约束**：
- 不执行交互式命令
- 超时保护
- stderr 单独返回

### 3.4 P1: Git 操作

#### git_diff

```typescript
interface GitDiffInput {
  path?: string;          // 只看特定文件
  staged?: boolean;       // 只看已暂存的
}

interface GitDiffOutput {
  success: true;
  diff: string;           // 标准 diff 格式
  files: string[];        // 变更的文件列表
} | {
  success: false;
  error: string;
}
```

#### git_commit

```typescript
interface GitCommitInput {
  message: string;       // 提交信息
  files?: string[];      // 指定提交文件 (默认 all)
}

interface GitCommitOutput {
  success: true;
  hash: string;          // commit hash
  message: string;
} | {
  success: false;
  error: string;
}
```

### 3.5 P1: 验证

#### test

```typescript
interface TestInput {
  pattern?: string;       // 测试文件 pattern
  watch?: boolean;        // watch 模式
}

interface TestOutput {
  success: boolean;
  passed: number;
  failed: number;
  duration: number;
  errors?: Array<{
    test: string;
    error: string;
  }>;
}
```

### 3.6 P2: 反思 (核心自举能力)

#### reflect

```typescript
interface ReflectInput {
  attempt: Attempt;       // 上一次的尝试
  context?: string;       // 额外上下文
}

interface ReflectOutput {
  success: true;
  reflection: Reflection;
  confidenceDelta: number; // 信心变化 (-0.1 ~ +0.1)
  shouldRetry: boolean;   // 是否应该重试
  newApproach?: string;   // 新方法建议
}
```

**反思 prompt 模板**：

```
你刚刚尝试了: {attempt_description}

结果: {success/failure}

输出: {output}

请反思:
1. 什么成功了？
2. 什么失败了？
3. 学到了什么？
4. 下次应该怎么做？
```

---

## 四、Agent Loop 实现

### 4.1 核心流程

```typescript
export async function* bootstrapLoop(
  state: BootstrapState,
  tools: ToolRegistry
): AsyncGenerator<AgentEvent> {
  
  // Step 1: 意图识别
  const intent = yield* recognizeIntent(state);
  state.intent = intent;
  
  // Step 2: 制定计划
  const plan = yield* makePlan(intent, state);
  state.plan = plan;
  
  // Step 3: 执行循环
  for (const step of plan) {
    const result = yield* executeStep(step, tools);
    
    // Step 4: 验证
    if (!result.success) {
      yield { type: 'error', content: `步骤失败: ${step}` };
      break;
    }
    
    // Step 5: 记录
    state.attempts.push(result);
  }
  
  // Step 6: 反思
  const reflection = yield* reflect(state);
  state.confidence = Math.max(0, Math.min(1, 
    state.confidence + reflection.confidenceDelta
  ));
  
  yield { type: 'done' };
}
```

### 4.2 与现有架构集成

```
┌─────────────────────────────────────────────────────────────────┐
│                     Novalife Core                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │  输入层     │───▶│  Router    │───▶│  Agent Loop         │  │
│  │ (Telegram) │    │            │    │  ┌───────────────┐  │  │
│  └─────────────┘    └─────────────┘    │  │ 默认模式      │  │  │
│                                          │  ├───────────────┤  │  │
│                                          │  │ 自举模式      │  │  │
│                                          │  │ (bootstrap)   │  │  │
│                                          │  └───────────────┘  │  │
│                                          └─────────────────────┘  │
│                                                                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │  激素系统   │◀───│  状态更新   │◀───│  工具层             │  │
│  │             │    │             │    │  read_file          │  │
│  │  信心激素   │    │  confidence │    │  write_file         │  │
│  │  好奇激素   │    │  curiosity  │    │  execute           │  │
│  │             │    │             │    │  git_commit        │  │
│  └─────────────┘    └─────────────┘    │  reflect           │  │
│                                          └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 五、使用示例

### 5.1 场景：改进 prompt

```
用户: " Novalife, 你的回复太长了，简化一下"

1. Agent Loop 识别意图: "用户希望缩短回复"
2. 制定计划:
   - 读取当前 system prompt
   - 修改 max_tokens 或添加长度约束
   - 测试效果
3. 执行...
4. 反思: "添加长度约束后，用户满意度提升"
```

### 5.2 场景：自动修复 bug

```
用户: "有个 bug，搜索功能返回空结果"

1. Agent Loop 识别意图: "修复搜索 bug"
2. 制定计划:
   - 读取搜索相关代码
   - 尝试复现问题
   - 定位原因
   - 修复
   - 测试
3. 执行...
4. 反思: "原来是 API 返回格式变了"
```

---

## 六、限制与安全

### 6.1 初期限制

- **只读模式为主**: 默认不允许自动 commit
- **人工确认**: 重大修改需要用户确认
- **沙箱执行**: 代码执行在隔离环境

### 6.2 安全边界

| 操作 | 需要确认 | 说明 |
|------|---------|------|
| read_file | ❌ | 安全 |
| list_dir | ❌ | 安全 |
| execute | ✅ | 确认命令 |
| write_file | ✅ | 确认内容 |
| git_commit | ✅ | 确认提交 |

---

## 七、下一步

- [ ] 实现 P0 工具 (read_file, write_file, list_dir, execute)
- [ ] 实现 P1 工具 (git_diff, git_commit, test)
- [ ] 实现基础 Agent Loop
- [ ] 集成激素系统 (confidence 激素)
- [ ] 添加用户确认机制

---

## 八、参考

- Pi-mono: 严格类型、模块化
- Novalife CORE_DESIGN: 单一心智、激素系统
- OpenClaw: 工具注册、执行模式

---

**文档版本**: v1.0
**最后更新**: 2026-03-06
