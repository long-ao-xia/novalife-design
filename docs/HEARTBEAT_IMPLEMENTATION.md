# 实施方案：M1.5 心跳与记忆系统

> 方案编号: IMP-002
> 状态: 进行中
> 对应里程碑: M1 (自举能力) 扩展
>
> 基于: MANIFESTO.md, ROADMAP.md, AGENTS.md, SOUL.md

---

## 一、背景

参考 OpenClaw 的设计，为 Novalife 添加：

1. **心跳机制** — 周期性自检
2. **记忆系统** — 长期 + 短期记忆
3. **工作偏好** — 遵循 Jerry 的工作方式

---

## 二、心跳机制

### 2.1 设计来源

参考 AGENTS.md 的 Heartbeats 章节：

- 周期性自检，不依赖外部触发
- 检查关键服务状态
- 自动恢复

### 2.2 Novalife 心跳

```typescript
interface HeartbeatConfig {
  interval: number;      // 间隔 (ms)
  checks: HealthCheck[];
}

interface HealthCheck {
  name: string;
  check: () => Promise<boolean>;
  recover?: () => Promise<void>;
}
```

### 2.3 检查项

| 检查项 | 说明 | 恢复方式 |
|--------|------|----------|
| Bot 进程 | 检查 Telegram Bot 是否运行 | 重启进程 |
| LLM 连接 | 检查 API 是否可连接 | 重试 |
| 内存状态 | 检查 Tape 是否可写 | 重建 |

### 2.4 实现

```typescript
// heartbeat.ts
export class Heartbeat {
  private timer: NodeJS.Timeout | null = null;
  
  constructor(private config: HeartbeatConfig) {}
  
  start() {
    this.timer = setInterval(() => this.tick(), this.config.interval);
  }
  
  stop() {
    if (this.timer) clearInterval(this.timer);
  }
  
  private async tick() {
    for (const check of this.config.checks) {
      const healthy = await check.check();
      if (!healthy && check.recover) {
        await check.recover();
      }
    }
  }
}
```

---

## 三、记忆系统

### 3.1 设计来源

参考 AGENTS.md 的 Memory 章节：

- **Daily notes**: `memory/YYYY-MM-DD.md` — 原始日志
- **Long-term**: `MEMORY.md` — 精选记忆

### 3.2 Novalife 记忆

| 类型 | 存储位置 | 说明 |
|------|----------|------|
| 即时记忆 | Tape (tape.jsonl) | 当前对话上下文 |
| 重要记忆 | REFLECTIONS.md | 反思记录 |
| 长期记忆 | MEMORY.md | 人工精选 |

### 3.3 记忆工具

```typescript
// 扩展现有工具
const memoryTools = {
  // 读取即时记忆
  recall: async (args: { count?: number }) => {
    // 从 tape 读取
  },
  
  // 写入重要记忆
  remember: async (args: { content: string; tag?: string }) => {
    // 写入 MEMORY.md
  },
  
  // 反思
  reflect: async (args: { attempt: string; result: string; analysis?: string }) => {
    // 写入 REFLECTIONS.md
  }
};
```

---

## 四、工作偏好

### 4.1 来源

参考 SOUL.md 中 Jerry 的工作偏好：

1. **测试优先** — 合理数量的测试
2. **设计文档优先** — 基于设计讨论
3. **MVP 思维** — 能跑起来 > 完美
4. **Single Source of Truth** — 不重复
5. **增量推进** — 小步快跑

### 4.2 体现

- 代码必须有测试
- 先有设计再编码
- 每个 commit 有明确成果

---

## 五、实现

### 5.1 心跳模块

```typescript
// src/heartbeat.ts
import { execSync } from 'child_process';

export interface HeartbeatConfig {
  interval: number;  // ms
  botToken?: string;
  projectDir: string;
}

export class Heartbeat {
  private timer: NodeJS.Timeout | null = null;
  
  constructor(private config: HeartbeatConfig) {}
  
  start() {
    console.log('❤️ 启动心跳机制');
    this.timer = setInterval(() => this.tick(), this.config.interval);
  }
  
  stop() {
    if (this.timer) {
      clearInterval(this.timer);
      this.timer = null;
    }
  }
  
  private async tick() {
    // 检查 Bot 进程
    await this.checkBotProcess();
  }
  
  private async checkBotProcess() {
    try {
      const pidFile = this.config.projectDir + '/bot.pid';
      const fs = await import('fs');
      
      if (!fs.existsSync(pidFile)) {
        console.log('❌ Bot 未运行');
        return this.restartBot();
      }
      
      const pid = parseInt(fs.readFileSync(pidFile, 'utf-8'));
      
      // 检查进程是否存在
      try {
        process.kill(pid, 0);
        console.log('✅ Bot 运行正常 (PID:', pid, ')');
      } catch {
        console.log('❌ Bot 进程不存在');
        return this.restartBot();
      }
    } catch (e) {
      console.error('心跳检查错误:', e);
    }
  }
  
  private async restartBot() {
    console.log('🔄 重启 Bot...');
    try {
      execSync(`cd ${this.config.projectDir} && TELEGRAM_BOT_TOKEN=${this.config.botToken} nohup node dist/telegram-bot.js > nohup.out 2>&1 &`, {
        stdio: 'ignore'
      });
      console.log('✅ Bot 已重启');
    } catch (e) {
      console.error('❌ Bot 重启失败:', e);
    }
  }
}
```

### 5.2 集成到 Telegram Bot

```typescript
// telegram-bot.ts 追加
import { Heartbeat } from './heartbeat.js';

const heartbeat = new Heartbeat({
  interval: 60000,  // 1 分钟
  botToken: TOKEN,
  projectDir: '/Users/lax/.openclaw/workspace/novalife'
});

heartbeat.start();
```

---

## 六、验收标准

### 6.1 心跳机制

- [x] 心跳进程启动
- [x] 每分钟检查一次
- [x] Bot 进程不存在时自动重启
- [x] 日志记录心跳状态

### 6.2 记忆系统

- [x] recall 读取即时记忆
- [x] remember 写入重要记忆
- [x] reflect 写入反思文件

### 6.3 测试

- [x] heartbeat.test.ts (3 tests)
- [ ] memory.test.ts

---

## 七、测试用例

### 7.1 心跳测试

```typescript
// tests/heartbeat.test.ts
- should start and execute checks
- should detect missing PID file and restart
- should detect invalid PID and restart
```

结果: ✅ 14 tests passed (11 core + 3 heartbeat)

### 7.2 记忆测试

(待实现)

---

## 八、项目管理

### 任务

- [x] 实现 Heartbeat 类
- [x] 集成到 Telegram Bot
- [x] 添加心跳测试
- [ ] 添加记忆测试

### 里程碑

- M1.5: 心跳 + 记忆 ✅ 心跳已完成

---

**方案版本**: v1.0
**最后更新**: 2026-03-07
