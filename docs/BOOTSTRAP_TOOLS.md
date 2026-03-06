# 实施方案：自举工具 MVP

> 方案编号: IMP-001
> 状态: 进行中
> 对应里程碑: M1 (自举能力)
>
> 基于: MANIFESTO.md, ROADMAP.md

---

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

> **核心理念**: 来自 pi-mono 的启示 —— 4 个工具足以构建有效的 coding agent。
> 
> "As it turns out, these four tools are all you need for an effective coding agent." — badlogic
>
> 系统 prompt + 工具定义 < 1000 tokens。

### 3.1 最小工具集 (Pi-mono 风格)

| 工具名 | 功能 | 说明 |
|--------|------|------|
| `read` | 读取文件 | 支持文本和图片 |
| `write` | 写入文件 | 覆盖或创建 |
| `edit` | 编辑文件 | 精确替换 |
| `bash` | 执行命令 | 返回 stdout/stderr |

**为什么只需要这 4 个？**
- 模型知道如何使用 bash (ls, grep, find 等)
- read/write/edit 是模型训练时使用的标准工具
- 其他工具都可以通过 bash + CLI 实现

### 3.2 扩展: 只读工具 (可选)

如果需要限制权限，可以启用：
- `grep`, `find`, `ls` — 但默认禁用

### 3.3 P0: 基础文件系统

#### read

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

## 七、Pi-mono 哲学总结

> "If I don't need it, it won't be built. And I don't need a lot of things." — badlogic

### 7.1 不做的事情

| 特性 | 为什么不做 |
|------|-----------|
| **MCP 支持** | overhead 太大，用 CLI 工具代替 |
| **内置 to-dos** | 用文件代替 (TODO.md) |
| **Plan mode** | 用文件代替 (PLAN.md) |
| **Background bash** | 用 tmux 代替 |
| **Sub-agents** | 用 bash 运行自己代替 |
| **安全检查** | YOLO 模式，用户自己负责 |

### 7.2 一切皆文件

- 任务列表 → `TODO.md`
- 计划 → `PLAN.md`
- 反思 → `REFLECTIONS.md`
- 上下文 → `CONTEXT.md`

Agent 不需要内置状态，所有状态都在文件里。

---

## 八、下一步

- [x] 设计采用 pi-mono 4 工具原则
- [ ] 实现核心工具 (read, write, edit, bash)
- [ ] 设计极简 system prompt (< 1000 tokens)
- [ ] 实现 Agent Loop (循环直到完成，无 max steps)
- [ ] 集成激素系统 (confidence)
- [ ] 用户确认机制 (YOLO vs 安全模式)

---

## 九、参考

- **Pi-mono** (Mario Zechner): https://github.com/badlogic/pi-mono
- **Blog Post**: https://mariozechner.at/posts/2025-11-30-pi-coding-agent/
- Novalife CORE_DESIGN: 单一心智、激素系统
- OpenClaw: 工具注册、执行模式

---

## 九、项目管理 (Plan/TODO)

### 当前任务

- [ ] 实现核心工具 (read, write, edit, bash)
- [ ] 设计极简 system prompt
- [ ] 实现 Agent Loop
- [ ] 集成测试验证
- [ ] 人工确认机制

### 完成标准

- [ ] Agent 能自己运行测试
- [ ] Agent 能自己修复简单 bug
- [ ] 反思能写入文件

### 风险与依赖

| 风险 | 影响 | 应对 |
|------|------|------|
| LLM 能力不足 | 无法自举 | 换用更强模型 |
| 安全问题 | 数据泄露 | YOLO 模式，用户负责 |

---

## 十、变更记录

| 日期 | 版本 | 变更 |
|------|------|------|
| 2026-03-06 | v1.0 | 初始版本 |

---
