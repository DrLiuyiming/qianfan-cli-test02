---
name: qianfan-auth
version: 1.0.0
description: "千帆 CLI 认证管理：浏览器 OAuth2/OIDC + PKCE 登录（STS 签发临时凭证）、查看本地凭证状态、清理登录态。当用户要初始化千帆凭证、排查鉴权问题、查看当前登录账号，或任何 qianfan 命令返回 CREDENTIAL_NOT_FOUND / AUTH_FAILED / LOGIN_FAILED / ALREADY_LOGGED_IN 时使用。触发词：千帆登录、登出、浏览器授权、认证状态、qianfan auth。反触发：用户想换套餐或绑定默认模型时转 qianfan-profile；想查为什么推理不通的完整体检转 qianfan-doctor。"
license: Apache-2.0
metadata:
  requires:
    bins: ["qianfan"]
  cliHelp: "qianfan auth --help"
  install:
    - kind: npm
      package: "@baiducloud/qianfan-cli"
---

# Qianfan 认证

先读 [qianfan-common](../qianfan-common/SKILL.md) 的公共规则。登录通过系统浏览器完成 OAuth2/OIDC + PKCE
授权，由 STS 签发临时访问凭证；**技能层不索取、不代填任何密钥**，密钥与 session token 只进入本地安全存储，
任何输出都不含敏感值。

## 何时使用本 skill

- 首次配置千帆 CLI，需要登录
- 查看当前是否已登录、登录的是哪个账号、当前套餐与模型绑定
- 换账号（必须先登出）
- 其他业务命令因未登录、凭证过期、身份冲突被阻塞

## 命令一览

| 命令 | 说明 |
| --- | --- |
| `qianfan auth status [--format json]` | 只读查看本地凭证状态，不发网络请求 |
| `qianfan auth login [--dry-run] [--yes] [--format json]` | 浏览器登录并落盘临时凭证，需用户本地终端执行 |
| `qianfan auth logout [--dry-run] [--yes] [--format json]` | 清空当前账号的全部本地登录痕迹 |

## Agent 快速执行顺序

1. `qianfan auth status --format json` 判断登录态：`configured=false` 即未登录
2. 未登录 → 提示用户在**本地终端**执行 `qianfan auth login`（Agent 无法代做浏览器授权）
3. 返回 `ALREADY_LOGGED_IN` → 先 `qianfan auth logout`（走确认门）再登录
4. 登录后如需切套餐或绑定模型，转 [qianfan-profile](../qianfan-profile/SKILL.md)

## 查看认证状态

```bash
qianfan auth status --format json
```

返回 `configured`、`account`、脱敏后的 `access_key`、`secret_key_configured`、`current_profile`、
脱敏后的 `api_key`（未缓存时不输出该字段，字段缺失即代表本地没有可用的 API Key 缓存）、
`model_bindings`（当前 profile 的绑定）。`access_key` 与 `api_key` 都只保留前 4 后 4 位，拿不到可用凭证，
仅供核对身份；`configured=false` 表示未登录：text 模式此时返回 `CREDENTIAL_NOT_FOUND`，json 模式仍输出
`configured:false` 供脚本解析。

## 登录

```bash
qianfan auth login
```

CLI 打开系统浏览器等待授权回调；未能自动打开时会打印授权链接供用户手动访问。授权成功后换取 STS 临时
凭证并落盘：访问凭证写入 config、密钥写入凭证存储、session/refresh/id token 写入会话存储，同时初始化默认
profile 并把当前 profile 设为 `platform`。三处写入任一步失败会整体回滚，不留下分属不同账号的混合状态。
随后 CLI 尝试**为 platform 套餐导入永久 API Key**（经 openapi-iam 按固定名称查找，不存在则创建后解密取
明文）写入本地缓存；导入失败不影响登录，首次使用时再在线获取。登录失败返回 `LOGIN_FAILED`。

- **同时只支持一个登录账号**：已有登录态时执行 `auth login` 直接返回 `ALREADY_LOGGED_IN`，浏览器都不会
  打开（`--dry-run` 也会如实报出该冲突）。换账号必须先 `qianfan auth logout`。
- **本地账号名取自 ID Token 的 `principal_id`**（不再是固定的 `default`），`auth status` 的 `account`
  就是它；凭证、会话、API Key 缓存都按这个名字分目录存放，因此不同身份的本地状态不会互相覆盖。
- **凭证与签发它的后端绑定**：登录时把本次登录对应的后端记进 config 的账号条目，之后所有命令都按它
  解析域名。一份登录态里的凭证只在签发它的那套后端有效，因此这项在登录后不可变更，只能 `logout`
  后重新登录。
- 登录本身不需要确认门（不会覆盖已有凭证），`--dry-run` 可预览将写入的三项变更。
- 登录态由 refresh token 维持：STS 凭证临期或失效时 CLI 自动静默续期，用户无需重复浏览器登录；
  续期也失败才返回 `CREDENTIAL_NOT_FOUND`。

## 登出（受保护）

```bash
qianfan auth logout --dry-run --format json   # 先看计划
qianfan auth logout --yes                     # 用户确认后执行
```

`logout` 清空当前账号的登录痕迹：密钥、会话（session/refresh/id token）、全部 API Key 缓存，
以及 config 里的账号条目与当前套餐指向。各套餐的模型绑定属于用户偏好，登出后保留，重新登录可直接复用；
输出格式与状态目录（home）等与账号无关的偏好同样保留——home 只有下一次登录能改。
非交互环境未传 `--yes` 返回 `CONFIRMATION_REQUIRED`，交互式终端拒绝确认返回 `OPERATION_CANCELLED`。

## 与其他 skill 的串联

- `qianfan-profile` / `qianfan-models` / `qianfan-chat` / `qianfan-service` 被鉴权错误阻塞时，先回到这里
- 排查"登录了但还是调不通"，转 [qianfan-doctor](../qianfan-doctor/SKILL.md) 做全链路体检

## 自然语言触发词 + 跨技能指引表

| 用户怎么说 | 走哪个命令/skill |
| --- | --- |
| "登录千帆 / 我要授权 / 帮我登录" | `qianfan auth login`（提示用户本地执行） |
| "我登录了吗 / 当前是哪个账号 / 认证状态" | `qianfan auth status --format json` |
| "换个账号 / 切账号" | 先 `auth logout` 再 `auth login` |
| "退出登录 / 清理本地凭证" | `qianfan auth logout`（先 dry-run） |
| "换套餐 / 绑定默认模型" | 转 `qianfan-profile` |
| "为什么推理调不通" | 转 `qianfan-doctor` |

## 安全与边界

- 不索取密钥：任何情况下不要让用户粘贴 AK/SK 或 API Key，也不要代替用户完成浏览器授权。
- 只报脱敏值：`auth status` 的 `access_key` 与 `api_key` 都只保留前 4 后 4 位；SK、session token、
  refresh token 不出现在任何输出中，脱敏也不例外。
- 登出是不可逆本地删除：必须先 `--dry-run` 展示计划并取得用户确认。
- 不猜账号：CLI 只支持单账号，遇 `ALREADY_LOGGED_IN` 时不要自行登出，先问用户。

## 参考

- [qianfan-common/references/error-codes.md](../qianfan-common/references/error-codes.md) — 错误码、退出码与处置
