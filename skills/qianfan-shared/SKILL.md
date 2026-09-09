---
name: qianfan-shared
version: 1.0.0
description: "千帆 CLI 面向 Agent 集成的顶层定位与安全底线：能力地图、凭证职责边界（STS 临时凭证签管控面 / API Key 认推理面）、敏感信息最小暴露、技能包与 Agent 的协作约束。当需要理解千帆 CLI 整体定位、判断某个动作是否越界，或设计多步千帆任务的编排顺序时使用。反触发：具体命令参数与错误码在 qianfan-common 与各能力 skill，不要在本 skill 里找。"
license: Apache-2.0
metadata:
  requires:
    bins: ["qianfan"]
  cliHelp: "qianfan --help"
  install:
    - kind: npm
      package: "@baiducloud/qianfan-cli"
---

# Qianfan CLI 集成定位与安全底线

通用操作规则（安装、JSON 输出、认证前置检查、profile/模型选择、预演确认、错误码与退出码）在
[qianfan-common](../qianfan-common/SKILL.md)。本 skill 只讲整体定位、能力地图与安全底线。

## 何时使用本 skill

- 第一次接触千帆 CLI，需要知道它能做什么、边界在哪
- 规划多步任务（登录 → 选模型 → 绑定 → 推理 → 看用量）的编排顺序
- 判断某个动作是否越界（能不能读凭证、能不能替用户确认、能不能直接调 HTTP）

## 定位

千帆 CLI 面向已拥有千帆账号的用户，把既有管控 API 与推理 API 封装为本地统一命令。它不替代控制台，
也不引入新的授权体系：用户通过浏览器 OAuth2/OIDC + PKCE 登录，由 STS 签发临时访问凭证（权限与控制台
一致），CLI 用它以 BCE V1 签名向 openapi-iam 取得该账号的 API Key；此后模型列表、用量统计与推理调用
一律走 API Key 的 `Authorization: Bearer`。

## 能力地图

| 环节 | 命令 | skill |
| --- | --- | --- |
| 接入 Agent | `qianfan +connect` | qianfan-connect |
| 登录与凭证 | `qianfan auth login` / `status` / `logout` | qianfan-auth |
| 选套餐与模型 | `qianfan profile` / `qianfan models list` | qianfan-profile / qianfan-models |
| 调用模型 | `qianfan +chat` | qianfan-chat |
| 观测用量 | `qianfan service usage` | qianfan-service |
| 排障 | `qianfan doctor` | qianfan-doctor |

典型链路：`+connect` → `auth login` → `models list` → `profile set-default` → `+chat` → `service usage`，
中途任何一步失败都可以回到 `doctor` 定位。

## 凭证职责边界

- **STS 临时凭证**（访问凭证 + 密钥 + session token）：只用于向 openapi-iam 取/建/解密 API Key 时的
  BCE V1 签名，**不用于推理、也不再直接签业务接口**；由登录流程写入本地存储，密钥与
  session/refresh/id token 分离保存。临期或服务端失效时 CLI 用 refresh token 自动续期并覆盖落盘，
  无需用户重新走浏览器登录；续期失败才降级为 `CREDENTIAL_NOT_FOUND`。
- **API Key**：用于全部业务调用（推理、模型列表、用量统计、用户信息）的 `Authorization: Bearer` 认证，
  **不用于 IAM 签名**。platform 套餐的 Key 由 CLI 经 openapi-iam 按固定名称查找、必要时创建并解密获得，
  登录时导入本地缓存；按 `account + profile` 分区，不跨账号或跨 profile 复用；缓存不设过期时间，
  生命周期与登录态一致（登录/登出都会清空），服务端吊销时由业务调用的 401/403 触发一次刷新。
  TokenPlan 套餐尚未接入 API Key。
- **单账号约束**：本地同时只保存一个登录账号，换账号必须先 `auth logout`（否则 `ALREADY_LOGGED_IN`）。

## 敏感信息最小暴露

密钥、session/refresh/id token 与 API Key 只进入 CLI 内部的安全存储与请求签名/认证路径，
**不写入普通配置、日志、诊断、执行计划或技能包输出**。面向用户或 Agent 的输出中 Access Key 已脱敏，
其余敏感值一律不出现。本地敏感目录与文件保持 `0700`/`0600` 权限。

## Agent 集成约束

- 技能包只编排 `qianfan` CLI：**不读取本地凭证、不直接拼装千帆 HTTP 请求、不输出敏感信息**。
- 遇 `CREDENTIAL_NOT_FOUND`、`LOGIN_FAILED`、`ALREADY_LOGGED_IN`、`API_KEY_NOT_GENERATED` 等错误时，
  只转达 CLI 的 `suggestion`；**不要索取密钥，也不要代替用户完成浏览器授权**。
- `auth logout`、`+connect`、`+connect uninstall` 必须先 `--dry-run --format json`，用户确认后再 `--yes`。
- 会产生 token 消耗的动作（`+chat`、`doctor --check chat`）先把命令和调用规模告诉用户。

## 安全与边界

- 只读优先：能用只读命令（`auth status`、`profile show`、`models list`、`doctor`）确认现状时，不要先写。
- 不猜配置：modelId、profile 名、时间范围等参数缺失时问用户或先查询，不要凭印象填。
- 不越权：CLI 权限等同当前登录身份，跨账号、跨团队的诉求应引导用户去控制台或找管理员。
- 失败不硬重试：同一条失败命令不要原样重试，按错误码分类处置。

## 参考

- [qianfan-common/SKILL.md](../qianfan-common/SKILL.md) — 通用调用规则与环境变量
- [qianfan-common/references/error-codes.md](../qianfan-common/references/error-codes.md) — 错误码、退出码与处置
