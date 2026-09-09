---
name: qianfan-doctor
version: 1.0.0
description: "千帆 CLI 体检（qianfan doctor）：一次性检查配置、访问凭证、密钥、BCE V1 签名、当前套餐、chat 模型绑定、API Key 可解析、技能包安装状态，可选 --check chat 探测真实推理连通性。当用户说千帆诊断/体检/自检、为什么调不通、排查千帆问题，或其他 qianfan 命令报错后需要定位根因时使用。反触发：已经明确是未登录就直接转 qianfan-auth；只是缺默认模型转 qianfan-profile。"
license: Apache-2.0
metadata:
  requires:
    bins: ["qianfan"]
  cliHelp: "qianfan doctor --help"
  install:
    - kind: npm
      package: "@baiducloud/qianfan-cli"
---

# Qianfan 诊断

先读 [qianfan-common](../qianfan-common/SKILL.md) 的公共规则。doctor 聚合本地、控制面与推理面的检查，
不索取 SK/API Key，输出中不含敏感值。

## 何时使用本 skill

- 用户说"调不通/推不了"，但错误码指向不明确
- 想一次看清认证、套餐、API Key、模型绑定、技能包的整体状态
- 需要确认推理链路真的可用（`--check chat`）
- 在 CI/脚本里做前置健康检查（按退出码判定）

## 命令一览

| 命令 | 说明 |
| --- | --- |
| `qianfan doctor [--format json]` | 全量只读体检，不消耗推理 token |
| `qianfan doctor --check chat [--format json]` | 额外发一次 `max_tokens=1` 的极小推理，探测真实连通性 |

输出为 `{ "ok": bool, "checks": [{name, ok, detail, code}, ...] }`；text 模式逐行打印
`[OK]/[FAIL] 名称  明细  (错误码)`。

## Agent 快速执行顺序

1. `qianfan doctor --format json`
2. 从上到下找第一个 `ok=false`：认证类（config / access_key / secret_key / bce_v1_sign）优先解决
3. 再看 profile / 模型绑定 / API Key 类问题，按各项 `code` 转对应 skill
4. 需要验证真实推理链路时再加 `--check chat`（会产生一次极小的 token 消耗）
5. 存在 FAIL 项时命令返回 `DIAGNOSTICS_FAILED`、退出码 7，诊断明细已先行输出

## 检查项

依次检查：`config`、`access_key`、`secret_key`、`bce_v1_sign`、`current_profile`、`chat_model_binding`、
`api_key`、（仅 `--check chat`）`inference`、`skills`。每项含 `{name, ok, detail, code}`；
逐项含义与常见 FAIL 处置见 [references/doctor.md](references/doctor.md)。

## 处置路由

| FAIL 项 / 错误码 | 处置 |
| --- | --- |
| `config` / `access_key` / `secret_key`（`CREDENTIAL_NOT_FOUND`） | 转 [qianfan-auth](../qianfan-auth/SKILL.md)：`qianfan auth login` |
| `bce_v1_sign` | 凭证不完整，重新登录 |
| `current_profile` | 转 [qianfan-profile](../qianfan-profile/SKILL.md)：`profile use platform` |
| `chat_model_binding`（`MODEL_NOT_CONFIGURED`） | `qianfan profile set-default --model-type chat <modelId>` |
| `api_key`（`INVALID_ARGUMENT`） | 当前是 TokenPlan 套餐，切回 `platform` |
| `api_key`（`IAM_*` / `API_KEY_*` / `PACKAGE_*` / `SEAT_*`） | 按 `suggestion` 去控制台或联系 IAM 管理员处理 |
| `inference` | 结合 `code` 判断：模型不支持、Key 无权限或网络问题 |
| `skills` | 转 [qianfan-connect](../qianfan-connect/SKILL.md)：`qianfan +connect` |

## 自然语言触发词 + 跨技能指引表

| 用户怎么说 | 走哪个命令/skill |
| --- | --- |
| "帮我体检 / 自检 / 看看环境有没问题" | `qianfan doctor --format json` |
| "为什么调不通 / 推理失败" | `qianfan doctor --check chat --format json` |
| "我是不是没登录" | `qianfan auth status`（更轻）或 doctor 全量体检 |
| "技能包装好了吗" | doctor 的 `skills` 项，或 `qianfan +connect list` |

## 安全与边界

- 默认只读：不改本地状态、不动远端资源；`--check chat` 是唯一会产生 token 消耗的检查，先告知用户。
- 不输出敏感值：只报告"密钥是否可读/签名是否可生成"，不打印 SK、session token 或 API Key。
- 建议不等于执行：doctor 给出的修复命令要先展示给用户，涉及本地写入的命令仍走各自的确认门。
- 不跨账号排查：只诊断当前登录身份。

## 参考

- `references/doctor.md` — 逐项检查含义与 FAIL 处置
- [qianfan-common/references/error-codes.md](../qianfan-common/references/error-codes.md) — 错误码、退出码与处置
