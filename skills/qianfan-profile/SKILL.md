---
name: qianfan-profile
version: 1.0.0
description: "千帆 CLI 套餐（profile）与默认模型绑定管理：列出/查看套餐、切换当前套餐、为模型类型绑定默认 modelId。当用户提到千帆套餐、切换 profile、绑定默认模型、platform / tokenplan 个人版 / 企业版、qianfan profile，或推理报 MODEL_NOT_CONFIGURED 时使用。反触发：想知道有哪些 modelId 可绑定先转 qianfan-models；只是未登录转 qianfan-auth。"
license: Apache-2.0
metadata:
  requires:
    bins: ["qianfan"]
  cliHelp: "qianfan profile --help"
  install:
    - kind: npm
      package: "@baiducloud/qianfan-cli"
---

# Qianfan 套餐与模型绑定

先读 [qianfan-common](../qianfan-common/SKILL.md) 的公共规则。profile 表示套餐类型，每个 profile 下维护
"模型类型 → modelId" 的绑定；API Key 的解析与缓存由 CLI 完成，技能层不感知。

## 何时使用本 skill

- 查看当前生效的套餐与模型绑定
- 切换当前套餐
- 为 `chat` 等模型类型绑定默认 modelId，解决 `MODEL_NOT_CONFIGURED`
- 确认某个套餐是否可用（TokenPlan 目前不可用）

## 命令一览

| 命令 | 说明 |
| --- | --- |
| `qianfan profile list [--format json]` | 列出全部套餐，text 模式以 `*` 标记当前套餐 |
| `qianfan profile show [profile] [--format json]` | 查看指定/当前套餐的名称、API Key 来源与模型绑定 |
| `qianfan profile use <profile>` | 先解析该套餐 API Key，**成功后才切换**当前套餐 |
| `qianfan profile set-default --model-type chat <modelId>` | 为当前套餐绑定某模型类型的默认 modelId，**校验通过才写入**（modelId 存在 + 类型匹配 + 预置服务已开通） |

`use` 与 `set-default` 没有 `--format` 参数，只输出一行确认文本。

## Agent 快速执行顺序

1. `qianfan profile show --format json` 看当前套餐与 `model_bindings`
2. 缺 chat 绑定 → 先用 [qianfan-models](../qianfan-models/SKILL.md) 取候选 modelId，再 `set-default`
3. 需要换套餐 → `qianfan profile use platform`（TokenPlan 会失败，见下）
4. 切换/绑定后可用 `qianfan +chat` 验证

## 可用套餐

- `platform`（平台预置服务）：**当前唯一可用**。
- `tokenplan_personal` / `tokenplan_enterprise`：配置里已预置，但**尚未接入 API Key**。对其执行
  `profile use` 返回 `INVALID_ARGUMENT`（"当前套餐（TokenPlan）暂未接入 API Key"）且**不会切换**，
  处置是留在或切回 `platform`，不要反复重试。

`list --format json` 输出整份配置（`current_profile`、`profiles`、`accounts`，其中 Access Key 已脱敏；
SK 与 API Key 本就不在配置里）；未登录或本地没有配置时 `list` 返回 `CREDENTIAL_NOT_FOUND`（退出码 3），
错误落 stderr、stdout 保持干净。`show` 返回 `display_name`、`api_key_source`、`model_bindings`，
不存在的 profile 报 `INVALID_ARGUMENT`；不带参数且没有当前套餐（未登录/已登出）时报
`INVALID_ARGUMENT`（"缺少 profile 参数"），需显式传入套餐名或先登录。

## 切换套餐与绑定模型

```bash
qianfan profile list --format json
qianfan profile show --format json
qianfan profile use platform
qianfan profile set-default --model-type chat <modelId>
```

`--model-type` 只接受 `chat`/`reasoning`/`vision`/`embedding`/`rerank`，其他取值返回 `INVALID_ARGUMENT`。
`set-default` 会在写入前做三道校验（会查一次模型列表，可能触发网络请求）：modelId 必须存在于当前账号的
预置服务列表（否则 `INVALID_ARGUMENT`）、类型必须可用于该 `--model-type`（例如 text2image 模型不能绑成
`chat`，返回 `INVALID_ARGUMENT` 并在 message 里给出真实类型）、且该预置服务必须已开通后付费调用
（未开通返回 `MODEL_NOT_ACTIVATED`）。任一校验失败都**不写入配置**，先按 `suggestion` 处置再重试。
当前推理入口只用到 `chat`。
套餐未开通、企业席位未分配或未生成 API Key 时 `use` 同样不切换，分别返回 `PACKAGE_NOT_ACTIVATED`、
`SEAT_NOT_ASSIGNED`、`API_KEY_NOT_GENERATED`，按 `suggestion` 引导用户去控制台处理。

## 与其他 skill 的串联

- 需要候选 modelId：转 [qianfan-models](../qianfan-models/SKILL.md)
- 未登录 / 凭证失效：转 [qianfan-auth](../qianfan-auth/SKILL.md)
- 绑定后验证推理：转 [qianfan-chat](../qianfan-chat/SKILL.md)
- `use` 报套餐/权限类错误定位不清：转 [qianfan-doctor](../qianfan-doctor/SKILL.md)

## 自然语言触发词 + 跨技能指引表

| 用户怎么说 | 走哪个命令/skill |
| --- | --- |
| "我现在用的哪个套餐 / 当前 profile" | `qianfan profile show --format json` |
| "有哪些套餐 / 列出 profile" | `qianfan profile list --format json` |
| "切到 platform / 换套餐" | `qianfan profile use platform` |
| "设置默认模型 / 绑定 chat 模型" | `qianfan profile set-default --model-type chat <modelId>` |
| "有哪些模型可以用" | 转 `qianfan-models` |
| "绑定报 MODEL_NOT_ACTIVATED" | 该预置服务未开通，按 `suggestion` 去模型中心开通后重试 `set-default` |
| "切 tokenplan 套餐" | 告知该套餐暂未接入 API Key，仅 `platform` 可用 |

## 安全与边界

- 只在校验通过后落盘：`set-default` 会查一次模型列表做校验，只写本地配置；`use` 只在 API Key 解析成功后才切换。
- 不伪造 Key、不替用户开通：套餐/席位/Key/预置服务未开通类错误必须回到控制台处理，CLI 不会生成或猜测 API Key。
- 输出不含敏感值：`list --format json` 的 Access Key 已脱敏，SK/API Key 不在配置中。

## 参考

- [qianfan-common/references/error-codes.md](../qianfan-common/references/error-codes.md) — 错误码、退出码与处置
