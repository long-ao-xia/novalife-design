# 项目管理 - Human-Like Computer Agent

## 项目信息

- **项目名**: human-like-agent
- **类型**: AI Agent / 自动化工具
- **状态**: 初始化

## 工作流

### 设计先行
- 任何新功能先写 DESIGN.md 条目
- 讨论确认后再编码
- 单次设计文档不宜过长

### 测试驱动
- 核心逻辑必须有测试
- 测试文件放在 `tests/`
- 命名规范: `{模块名}.test.ts`

### 提交规范
```
feat: 添加 xx 功能
fix: 修复 xx 问题
refactor: 重构 xx
docs: 更新文档
test: 添加测试
```

## 目录结构

```
human-like-agent/
├── docs/           # 设计文档
│   └── DESIGN.md   # 主设计文档 (Single Source of Truth)
├── src/            # 源代码
├── tests/          # 测试代码
└── README.md       # 项目说明
```

## 沟通节奏

- 每次对话同步: 当前进度 + 下一步计划
- 设计文档更新即时 commit

## Git 使用

- main 分支保持可发布状态
- 功能开发用 feature 分支
- 每次 merge 需要 code review
