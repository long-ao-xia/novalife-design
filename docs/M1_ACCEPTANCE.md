# M1 验收标准

> 版本: v1.0
> 里程碑: M1 (自举能力)

---

## 一、验收标准

### 1.1 核心功能

| 编号 | 标准 | 测试方法 | 状态 |
|------|------|----------|------|
| M1-F1 | read 工具能读取文件 | 执行 read({ path: "package.json" }) | ✅ |
| M1-F2 | write 工具能写入文件 | 执行 write({ path: "test.txt", content: "hello" }) | ✅ |
| M1-F3 | edit 工具能精确修改文件 | 执行 edit({ path, oldText, newText }) | ✅ |
| M1-F4 | bash 工具能执行命令 | 执行 bash({ command: "ls" }) | ✅ |
| M1-F5 | reflect 工具能写入反思 | 执行 reflect({ attempt, result }) 并验证文件 | ✅ |
| M1-F6 | test 工具能运行测试 | 执行 test() 并验证输出 | ✅ |

### 1.2 系统能力

| 编号 | 标准 | 测试方法 | 状态 |
|------|------|----------|------|
| M1-S1 | Telegram Bot 能接收消息 | Telegram 发送消息 | ✅ |
| M1-S2 | Telegram Bot 能回复消息 | 验证回复内容 | ✅ |
| M1-S3 | 心跳机制正常运行 | 检查日志中的心跳输出 | ✅ |
| M1-S4 | Bot 崩溃自动重启 | 杀掉进程后验证重启 | ✅ |

### 1.3 质量标准

| 编号 | 标准 | 测试方法 | 状态 |
|------|------|----------|------|
| M1-Q1 | 所有测试通过 | npm test | ✅ |
| M1-Q2 | 无 JSON 解析错误 | 检查日志无解析错误 | ✅ |
| M1-Q3 | 不保存空消息到 tape | 验证 tape 内容 | ✅ |

---

## 二、测试用例

### 2.1 工具测试

```
# read 工具
read({ path: "package.json" })
预期: 返回文件内容

# write 工具
write({ path: "test.txt", content: "hello" })
预期: 文件创建成功

# edit 工具
edit({ path: "test.txt", oldText: "hello", newText: "world" })
预期: 文件内容被修改

# bash 工具
bash({ command: "echo test" })
预期: 返回 "test"

# reflect 工具
reflect({ attempt: "测试", result: "成功", analysis: "正常" })
预期: REFLECTIONS.md 内容增加

# test 工具
test()
预期: 运行测试并返回结果
```

### 2.2 集成测试

```
# Telegram 对话
用户发送: "你好"
预期: Novalife 回复内容非空

# 心跳检查
等待 60 秒
预期: 日志出现心跳输出
```

---

## 三、验收流程

1. 运行 `npm test`，确保所有测试通过
2. 测试各工具功能
3. 测试 Telegram Bot 对话
4. 检查日志无错误
5. 验证心跳机制

---

## 四、签署

| 日期 | 验收人 | 结果 |
|------|--------|------|
| 2026-03-07 | Jerry | - |

---

**版本**: v1.0
**最后更新**: 2026-03-07
