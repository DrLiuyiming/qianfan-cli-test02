---
name: qianfan-common
version: 1.0.0
description: "千帆 CLI（qianfan 命令）的通用操作规则，供所有 qianfan-* skill 复用：CLI 安装、JSON 输出与解析、认证前置检查、套餐 profile 与模型选择、本地变更的预演与确认、环境变量、统一错误码与退出码。调用任何 qianfan 子命令（auth / profile / models / +chat / service / doctor / +connect）前先加载本 skill。反触发：用户只想执行某个具体能力（如登录、推理、查用量）时不要停在本 skill，读完规则立刻转对应能力 skill。"
license: Apache-2.0
metadata:
  requires:
    bins: ["qianfan"]
  cliHelp: "qianfan --help"
  install:
    - kind: npm
      package: "@baiducloud/qianfan-cli"
---

# Qianfan CLI 通用规则

所有 qianfan-* skill 共用的操作规则。自动化调用一律通过 `qianfan` 命令完成，
**不读取本地凭证文件，不直接拼装千帆 HTTP 请求**。

## 何时使用本 skill

- 准备调用任意 `qianfan` 子命令，需要先确认调用约定与前置条件
- 拿到 CLI 错误码，需要判断该重试、该转人工，还是该转其他 skill
- 需要知道当前支持哪些套餐 profile、模型类型与模型解析优先级
- 要执行会改动本地凭证或技能目录的命令，需要先走预演与确认门

## 准备 qianfan CLI

```bash
npm i -g @baiducloud/qianfan-cli
qianfan --version
```

也可从 GitHub Release 下载对应平台的静态二进制。安装过程不索取 AK/SK、不访问千帆服务。

## 命令一览

| 命令 | 说明 | 对应 skill |
| --- | --- | --- |
| `qianfan auth login` / `status` / `logout` | 浏览器登录、查看本地凭证状态、清理登录态 | qianfan-auth |
| `qianfan profile list` / `show` / `use` / `set-default` | 套餐查看切换与默认模型绑定 | qianfan-profile |
| `qianfan models list` | 列出可绑定模型 | qianfan-models |
| `qianfan +chat <prompt>` | 文本推理，支持 `--stream` | qianfan-chat |
| `qianfan service usage` | 服务调用统计与 token 消耗 | qianfan-service |
| `qianfan doctor` | 认证/套餐/API Key/模型/连通性体检 | qianfan-doctor |
| `qianfan +connect` / `list` / `uninstall` | 安装、枚举、卸载本机 Agent 技能包 | qianfan-connect |

除 `profile use` 与 `profile set-default` 外都支持 `--format text|json`（默认 `text`）；非法取值统一报
`INVALID_ARGUMENT`。那两条命令只输出一行确认文本，没有 `--format` 参数。

## Agent 快速执行顺序

1. 不确定认证状态时先 `qianfan auth status --format json`，`configured=false` 即未登录
2. 需要凭证的命令（`profile use` / `profile set-default` / `models list` / `+chat` / `service usage` / `doctor`）在未登录时一律先引导用户本地执行 `qianfan auth login`
3. 业务命令统一带 `--format json`，只消费 CLI 的结构化输出
4. 报错先按 `code` 分类（见 `references/error-codes.md`），把 `suggestion` 转达用户，不要原样重试同一条失败命令
5. 涉及本地写入的命令先 `--dry-run --format json` 展示计划，得到用户确认后再 `--yes`

## Profile 与模型选择

- 套餐 profile：`platform`（平台预置服务，**当前唯一可用**）、`tokenplan_personal`（Token Plan 个人版）、
  `tokenplan_enterprise`（Token Plan 企业版）。后两者已在配置中预置但**尚未接入 API Key**：
  对其执行 `profile use` / `+chat --profile` 返回 `INVALID_ARGUMENT`（"当前套餐（TokenPlan）暂未接入 API Key"），
  处置是切回 `platform`，不要反复重试。
- 模型类型：`chat`、`reasoning`、`vision`、`embedding`、`rerank`；当前推理入口仅支持 `chat`。
- 模型解析优先级：`+chat --model` > 当前/指定 profile 的 `model_bindings.chat` > 报 `MODEL_NOT_CONFIGURED`。
- 预置服务开通状态：`models list --format json` 的 `charge_status` 为 `false` 时，绑定默认模型与发起推理
  都会被拦下并返回 `MODEL_NOT_ACTIVATED`（推理不会发出请求）。CLI 判定前会强制刷新一次模型列表，
  所以只需引导用户去控制台开通，不要靠重试或等缓存过期。
- 推理用的 API Key 由 CLI 自动获取并缓存（platform 在登录时导入），技能层不感知、不传递 Key。

## 本地变更的预演与确认

`auth logout`、`+connect`、`+connect uninstall` 会改动本地凭证或技能目录，统一遵循：

1. 先 `--dry-run --format json` 拿到执行计划（`{command, requires_confirm, changes[]}`）并展示给用户；
2. 用户明确确认后再带 `--yes`（简写 `-y`）执行；
3. 非交互环境未传 `--yes` 返回 `CONFIRMATION_REQUIRED`；`--dry-run` 与 `--yes` 互斥（`INVALID_ARGUMENT`）；
4. 交互式终端未传 `--yes` 时 CLI 询问 `Continue? [Y/N]`，回答非 y/yes 返回 `OPERATION_CANCELLED`。

## 环境变量（一般无需设置）

| 变量 | 作用 |
| --- | --- |
| `QIANFAN_HOME` | 只有 `auth login` 读取，用于指定本地配置/凭证/缓存根目录（默认 `~/.qianfan`）。登录把最终路径记进 `~/.qianfan/home.json`，其余指令一律按该指针解析、不读此变量；登出不会重置它 |
| `QIANFAN_CONTROL_PLANE_ENDPOINT` | 覆盖控制面域名 |
| `QIANFAN_INFERENCE_ENDPOINT` | 覆盖推理域名。推理与控制面**分开配置**，覆盖控制面不会改推理地址 |
| `QIANFAN_CONSOLE_IAM_ENDPOINT` | 覆盖 console-iam（OAuth）域名 |
| `QIANFAN_OPENAPI_IAM_ENDPOINT` | 覆盖 openapi-iam（API Key）域名 |
| `QIANFAN_STS_ENDPOINT` | 覆盖 STS 域名 |
| `QIANFAN_DEBUG` | 置 `1`/`true` 时向 stderr 输出请求诊断（签名与令牌脱敏） |

## 安全与边界

- 只编排 CLI：不读取 `~/.qianfan` 下的凭证、会话、API Key 缓存，不自行签名或调千帆 HTTP 接口。
- 不索取密钥：登录只走浏览器 OAuth，Agent 不得要求用户粘贴 AK/SK/API Key，也不能代替用户完成授权。
- 写操作有确认门：预演与确认不可跳过，禁止"先执行再回滚"模拟 dry-run。
- 失败不硬重试：权限/套餐/席位类错误必须回到用户或控制台处理，CLI 不会伪造 Key 或切换 profile。

## 参考

- `references/error-codes.md` — 全部错误码、输出形态、退出码与处置
