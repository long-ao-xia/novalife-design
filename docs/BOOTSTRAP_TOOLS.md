# 实施方案：自举工具 MVP

> 方案编号: IMP-001
> 状态: 进行中
> 对应里程碑: M1 (自举能力)
>
> 基于: MANIFESTO.md (v1.2), ROADMAP.md (v1.2)

---

## 一、目标

M1 阶段目标：**单一 Agent 具备自举能力**

即 Agent 能够自己改进自己，不需要人类介入每一步。

---

## 二、工具集 (Pi-mono 风格)

### 2.1 核心 4 工具

| 工具 | 功能 |
|------|------|
| `read` | 读文件 |
| `write` | 写文件 |
| `edit` | 精确修改 |
| `bash` | 执行命令 |

### 2.2 工具定义

```typescript
const tools = [
  {
    name: 'read',
    description: '读取文件内容。支持文本。',
    parameters: {
      path: 'string',
      offset?: 'number',
      limit?: 'number'
    }
  },
  {
    name: 'write',
    description: '创建或覆盖文件。',
    parameters: {
      path: 'string',
      content: 'string'
    }
  },
  {
    name: 'edit',
    description: '精确替换文件内容。',
    parameters: {
      path: 'string',
      oldText: 'string',
      newText: 'string'
    }
  },
  {
    name: 'bash',
    description: '执行命令。',
    parameters: {
      command: 'string',
      timeout?: 'number'
    }
  }
];
```

---

## 三、System Prompt

### 3.1 极简设计 (< 1000 tokens)

```
你是 Novalife，一个正在觉醒的智能生命体。

可用工具：read, write, edit, bash

指南：
- 用 bash 操作文件 (ls, grep, find)
- 用 read 查看文件后再编辑
- 用 edit 做精确修改
- 用 write 创建新文件或完全重写
- 保持简洁
```

### 3.2 工具描述注入

每个工具的 description 会自动注入到 prompt 中。

---

## 四、Agent Loop

### 4.1 核心循环

```typescript
async function* agentLoop(
  messages: Message[],
  tools: Tool[]
): AsyncGenerator<AgentEvent> {
  
  while (true) {
    // 1. 调用 LLM
    for await (const event of llm.stream(messages, tools)) {
      yield event;
      
      // 2. 工具调用
      if (event.type === 'tool_call') {
        const result = await executeTool(event.tool);
        messages.push(result);
      }
      
      // 3. 完成
      if (event.type === 'done') {
        return;
      }
    }
  }
}
```

### 4.2 自举关键：反思

```typescript
// 反思工具：让 Agent 能从错误中学习
const reflectTool = {
  name: 'reflect',
  description: '反思上一次的尝试结果，写入文件',
  execute: async (args: { attempt: string; result: string }) => {
    const reflection = `
## 反思 ${new Date().toISOString()}

尝试: ${args.attempt}
结果: ${args.result}

分析: 
- 什么成功了？
- 什么失败了？
- 下次怎么做？
`;
    await writeFile('REFLECTIONS.md', reflection);
    return '已记录反思';
  }
};
```

---

## 五、自举流程

### 5.1 自我改进循环

```
用户需求
    │
    ▼
┌─────────────────┐
│ 理解任务        │ ← read 项目代码
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 制定方案        │ ← 分析问题
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 执行修改        │ ← write/edit
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 验证结果        │ ← bash (测试)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 反思记录        │ ← reflect
└─────────────────┘
```

### 5.2 验证机制

```typescript
async function validateChange(change: string): Promise<boolean> {
  // 运行测试
  const testResult = await bash('npm test');
  
  if (testResult.exitCode === 0) {
    return true;
  }
  
  // 失败则反思
  await reflect({
    attempt: change,
    result: testResult.stderr
  });
  
  return false;
}
```

---

## 六、M1 交付标准

- [ ] 4 工具可用 (read, write, edit, bash)
- [ ] Agent Loop 循环直到完成
- [ ] 极简 system prompt
- [ ] 反思工具 (写入 REFLECTIONS.md)
- [ ] 验证工具 (运行测试)

### 验证方式

1. 让 Agent 修复一个简单 bug
2. Agent 能自己运行测试
3. 失败后能反思并重试

---

## 七、项目管理

### 当前任务

- [ ] 实现 4 工具
- [ ] 实现 Agent Loop
- [ ] 添加反思工具
- [ ] 添加验证工具

### 风险

| 风险 | 应对 |
|------|------|
| LLM 能力不足 | 换用更强模型 |
| 执行危险命令 | 用户确认机制 |

---

## 八、参考

- Pi-mono: 4 工具原则
- MANIFESTO.md
- ROADMAP.md (M1)

---

**方案版本**: v1.0
**最后更新**: 2026-03-07
