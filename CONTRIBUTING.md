# Contributing to Qianfan CLI

感谢您对千帆 CLI 项目的关注！

## 项目说明

千帆 CLI 的核心代码为闭源发布，但 **Agent Skills 文档**（`skills/` 目录）完全开放贡献。Skills 是让 AI Agent（如 Comate、Claude Code 等）能够调用千帆 CLI 能力的关键文档，欢迎社区参与完善。

## 如何提交 Issue

- 使用 [GitHub Issues](https://github.com/baidubce/qianfan-cli/issues) 报告 bug 或提出建议
- 提交 bug 时请包含：操作系统、CLI 版本（`qianfan --version`）、复现步骤、实际与预期行为

## 贡献 Skills 文档

### Skill 目录结构

```
skills/
└── qianfan-xxx/
    ├── SKILL.md          ← Skill 能力说明文档（必须）
    └── references/       ← 参考资料（可选）
        └── *.md
```

### SKILL.md 格式规范

```markdown
---
name: qianfan-xxx
description: 一句话说明该 skill 的用途
---

# 能力说明

...（描述该 skill 提供的能力）

## 命令列表

...（列出相关命令及用法）
```

### 提交流程

1. Fork 本仓库
2. 创建分支：`git checkout -b feat/improve-skill-xxx`
3. 修改 `skills/` 下的文档
4. 提交 PR，描述改动内容和原因

### 注意事项

- Skills 文档应与实际 CLI 命令行为保持一致
- 保持语言简洁，避免歧义，AI Agent 需要准确理解文档内容
- 中英文均可，建议与现有文档保持语言一致

## 行为准则

请保持友善、尊重，共同维护良好的社区氛围。
