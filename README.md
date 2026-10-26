<div align="center">

# Qianfan CLI

**给你的 AI Agent 赋予千帆平台的能力**

[![npm](https://img.shields.io/npm/v/@baiducloud/qianfan-cli?color=brightgreen&label=npm)](https://www.npmjs.com/package/@baiducloud/qianfan-cli)
[![license](https://img.shields.io/badge/license-Apache--2.0-blue)](./LICENSE)
[![node](https://img.shields.io/badge/node-%3E%3D16-brightgreen)](https://nodejs.org)

千帆的模型发现、文本推理、文本向量、图像生成、视频生成、套餐与默认模型绑定、用量统计和诊断能力，
都可以通过 Qianfan CLI + Skills 交给 Comate、Claude Code 等本地 Agent 调用。

```shell
npm i -g @baiducloud/qianfan-cli@latest
```

</div>

---

## Qianfan CLI 能做什么

- **模型调用**：文本对话、SSE 流式输出、system 提示与采样参数
- **文本向量**：把一段或多段文本转成向量，用于检索、相似度与 RAG
- **图像生成**：按文本提示词生成图像，支持负向提示、分辨率与张数
- **视频生成**：按文本/图片生成视频（异步任务，可查询或轮询到完成）
- **模型发现**：列出当前账号可绑定的模型，拿到 modelId
- **套餐与绑定**：查看和切换套餐 profile，为模型类型绑定默认模型（含 image / video）
- **自定义接入点**：查询、创建、修改、停用自定义服务接入点，配置每接入点的 RPM/TPM 限流，并列出预置服务拿 serviceId
- **Agent 接入**：安装千帆 Skills，让本地 Agent 用自然语言调用千帆能力
- **用量统计**：按模型与时间粒度查调用次数、成功失败数与 token 消耗
- **体检诊断**：一条命令检查登录态、套餐、模型绑定与推理连通性

## 为什么需要 Qianfan CLI

Qianfan CLI 不是让你记住更多命令，而是把千帆平台能力变成 Agent 可以调用的工具箱。

```text
使用链路:
登录
  -> 选择模型
  -> 绑定默认模型
  -> 接入本地 Agent
  -> 调用推理
  -> 查看用量和体检结果
```

## 安装与初始化

```shell
# 1. 安装 CLI
npm i -g @baiducloud/qianfan-cli@latest
qianfan --version

# 2. 登录千帆（系统浏览器完成授权）
qianfan auth login
qianfan auth status

# 3. 把千帆 Skills 装到本机 Agent
qianfan +connect
```

安装期开关：`QIANFAN_SKIP_POSTINSTALL=1` 跳过平台二进制下载（离线环境使用）。

当前发布版本支持 macOS 与 Windows（x64 / arm64）。

## 一次完整任务可以这样开始

你可以直接告诉 Agent 一个目标：

```text
帮我挑一个可用的 chat 模型绑成默认，用它回答一个问题，
再告诉我这周的调用量和 token 消耗。
```

Agent 可以通过 Qianfan CLI 串起完整流程：

```text
qianfan models list
  -> qianfan profile set-default --model-type chat <modelId>
  -> qianfan +chat "..."
  -> qianfan service usage --startTime ... --endTime ...
  -> qianfan doctor
```

用户描述目标，Agent 负责选择命令、调用千帆能力、汇总结果。

## 常用任务

### 选择并调用模型

```shell
qianfan models list --format json
qianfan models list --refresh

qianfan +chat "总结一下 RAG 的核心思想"
qianfan +chat --stream --system "你是资深后端工程师" "解释一下 MVCC"
qianfan +chat --model <modelId> --temperature 0.2 --max-tokens 512 "..."
```

### 文本向量、图像与视频

```shell
# 文本向量（可一次传多段）
qianfan +embedding --format json "白色 T 恤" "黑色卫衣"

# 文生图
qianfan +gen --format json "一只戴墨镜的柴犬，扁平插画风格"

# 视频生成（异步任务）：创建后等待完成，或先拿 taskId 稍后查询
qianfan +video --image ./first-frame.png --wait --format json "镜头缓缓拉远"
qianfan +video --task <taskId> --format json
```

未传 `--model` 时，这些命令会回退到对应类型（`embedding` / `image` / `video`）的默认绑定，
可用 `qianfan profile set-default --model-type <type> <modelId>` 预先绑定。

### 套餐与默认模型

```shell
qianfan profile list
qianfan profile show [profile]
qianfan profile use <profile>
qianfan profile set-default --model-type chat <modelId>
qianfan profile set-default --model-type image <modelId>
qianfan profile set-default --model-type video <modelId>
```

### 接入本地 Agent

```shell
qianfan +connect list        # 查看各 Agent 目录已安装的 qianfan-* 技能包
qianfan +connect --dry-run   # 预演将要写入的目录
qianfan +connect             # 安装到所有检测到的 Agent 技能目录
qianfan +connect uninstall   # 移除本 CLI 装的、且未被本地修改的技能包
```

### 查用量和体检

```shell
qianfan service usage --startTime "2026-09-01 00:00:00" --endTime "2026-09-08 00:00:00"
qianfan service usage --interval 3600 --modelIds <id1>,<id2> --format json

qianfan doctor
qianfan doctor --check chat --format json
```

`doctor` 是只读诊断，不会改动任何资源。

### 自定义接入点

```shell
# 列出预置服务，拿 serviceId
qianfan endpoint services --type chat --format json

# 基于预置服务创建接入点（--rpm/--tpm/--tpm-input/--tpm-output 至少填一个）
qianfan endpoint create --service-id <serviceId> --name my-chat --suffix mychat --rpm 60 --tpm 10000

# 查询与管理
qianfan endpoint list --status Active --format json
qianfan endpoint get <endpointId>
qianfan endpoint modify <endpointId> --name new-name --tpm 20000
qianfan endpoint stop <endpointId>
```

`create` 必填 `--service-id`、`--name`、`--suffix`，限流四项至少指定一个；`modify` 至少指定一项要改的字段。

## 命令入口

| 任务 | 命令 |
|---|---|
| 登录、登出、查看登录态 | `qianfan auth` |
| 文本推理与流式输出 | `qianfan +chat` |
| 文本向量 | `qianfan +embedding` |
| 图像生成 | `qianfan +gen` |
| 视频生成 | `qianfan +video` |
| 查询可绑定模型 | `qianfan models` |
| 套餐切换与默认模型绑定 | `qianfan profile` |
| 调用量与 token 消耗 | `qianfan service usage` |
| 自定义接入点管理 | `qianfan endpoint` |
| 体检与连通性诊断 | `qianfan doctor` |
| 安装 / 卸载 Agent Skills | `qianfan +connect` |

每个命令都自带帮助：

```shell
qianfan auth --help
qianfan +chat --help
qianfan models list --help
qianfan profile --help
qianfan service usage --help
qianfan endpoint --help
qianfan doctor --help
```

## 认证

通过系统浏览器完成授权：

```shell
qianfan auth login
```

登录后 CLI 会自动完成身份、套餐 profile 与 API Key 配置，你不需要手动复制密钥。

```shell
qianfan auth status
qianfan auth logout
```

CLI 同时只保存一个登录账号，换账号先执行 `qianfan auth logout`。本地状态默认位于 `~/.qianfan`，
`auth login` 时可用 `QIANFAN_HOME` 指定其他目录。

## 配置

```shell
# 查看当前套餐 profile
qianfan profile show

# 列出所有 profile
qianfan profile list

# 切换 profile
qianfan profile use <profile-name>

# 为模型类型绑定默认模型
qianfan profile set-default --model-type chat <modelId>
```

更多 profile 管理能力可查看 `qianfan profile --help`。

除 `profile use` 与 `profile set-default` 外，所有命令支持 `--format text|json`（默认 `text`），
JSON 输出用于 Agent 与脚本解析；会改动本地状态的命令支持 `--dry-run` 预演与 `-y/--yes` 免交互确认。

## AI Agent Skills

本仓库的 [`skills/`](https://github.com/baidubce/qianfan-cli/tree/main/skills) 收录了 Qianfan CLI
内置的 Agent Skills。执行：

```shell
qianfan +connect
```

Qianfan CLI 会把这些 Skills 安装到本机 Agent（Comate / Claude Code 等），使 Agent 能通过自然语言调用
千帆能力，包括登录、模型查询、套餐与默认模型绑定、文本推理、文本向量、图像生成、视频生成、自定义接入点管理、
用量统计和体检诊断。技能包只编排 CLI 命令，不读取本地凭证、不直接调用千帆 HTTP 接口。

### 贡献 Skills

CLI 本身闭源发布，`skills/` 下的 Agent Skills 文档开放贡献。每个 skill 是一个 `skills/qianfan-xxx/` 目录，
`SKILL.md` 为必须项，`references/` 存放可选的参考资料。新增或修改时以现有 `skills/qianfan-*/SKILL.md` 为
模板（frontmatter 含 `name` / `version` / `description` / `license` / `metadata`，正文含「何时使用本 skill」
与「命令一览」），新增 skill 还需在 `skills/embed.go` 的 embed 指令中登记。

```shell
# Fork 本仓库并创建分支
git checkout -b feat/improve-skill-xxx

# 修改 skills/ 下的文档，然后提交推送
git commit -am "docs(skills): 说明改动内容"
git push -u origin feat/improve-skill-xxx
```

提交 PR 时说明改动内容和原因。Skills 文档必须与实际命令行为保持一致——Agent 会照着文档执行，
描述与实现不符会直接导致误用。

## 反馈与支持

使用问题、Bug 与需求请提交 [GitHub Issues](https://github.com/baidubce/qianfan-cli/issues)。可稳定复现的
问题建议附上执行的命令、`--format json` 输出与 `qianfan doctor` 结果，便于定位。

请勿在 Issue 或任何公开渠道发送 API Key、AK/SK、Token 等敏感信息。

## 链接

- 千帆大模型平台：<https://console.bce.baidu.com/qianfan/>
- npm：<https://www.npmjs.com/package/@baiducloud/qianfan-cli>
- GitHub Releases：<https://github.com/baidubce/qianfan-cli/releases>
- 内置 Skills：<https://github.com/baidubce/qianfan-cli/tree/main/skills>

## Changelog

Release notes live in [GitHub Releases](https://github.com/baidubce/qianfan-cli/releases).

## License

[Apache-2.0](./LICENSE)
