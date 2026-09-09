---
name: qianfan-models
version: 1.0.0
description: "查询当前千帆账号可绑定的模型列表（qianfan models list），拿到 modelId 供默认模型绑定或临时指定推理模型。当用户问千帆有哪些模型、可用/可绑定模型、列一下 modelId、qianfan models，或在绑定 chat 默认模型前需要候选模型时使用。反触发：真正要绑定默认模型是 qianfan-profile 的 set-default；要发起推理转 qianfan-chat。"
license: Apache-2.0
metadata:
  requires:
    bins: ["qianfan"]
  cliHelp: "qianfan models --help"
  install:
    - kind: npm
      package: "@baiducloud/qianfan-cli"
---

# Qianfan 模型列表

先读 [qianfan-common](../qianfan-common/SKILL.md) 的公共规则。模型查询只展示服务端返回结果，
受当前登录账号在控制台的资源可见范围约束。

## 何时使用本 skill

- 用户想知道账号下有哪些模型可用
- 绑定默认 chat 模型前需要候选 modelId
- `+chat --model` 需要一个确切的 modelId

## 命令一览

| 命令 | 说明 |
| --- | --- |
| `qianfan models list [--format json]` | 列出可绑定的预置服务，每项含 `id`（modelId，即预置服务名，可直接用于 `--model`）、`type`（模型类别）、`charge_status`（是否已开通后付费调用） |
| `qianfan models list --refresh` | 忽略本地缓存重新拉取，用于刚在控制台开通/新上线模型后立刻看到最新状态 |

`charge_status` 只出现在 `--format json`；text 模式只打两列（`id` 与 `type`），所以要判断开通状态一律用 JSON。

## Agent 快速执行顺序

1. `qianfan models list --format json`
2. 从结果中挑出目标 modelId（结合用户描述的能力/规格，优先 `charge_status` 为 `true` 的）
3. 需要固化为默认模型 → `qianfan profile set-default --model-type chat <modelId>`（见 qianfan-profile）
4. 只想临时用一次 → `qianfan +chat --model <modelId> --format json "..."`

## 行为要点

- 查询固定使用 **platform 套餐的 API Key**（Bearer 认证），**不随 `current_profile` 变化**，
  因此切换套餐不影响结果。
- 列表**只展示 chat 推理链路可用的模型**（服务端类型为 `chat` / `multimodal`）：CLI 当前只有
  `+chat` 一条推理链路，embedding / rerank / image2text 等模型暂不展示。账号下有这类模型但
  `models list` 里看不到是预期行为，不要据此判断账号无权限。
- `charge_status` 为 `false` 的预置服务不能绑定、也不能推理，两条链路都会返回 `MODEL_NOT_ACTIVATED`；
  挑 modelId 时优先选该字段为 `true` 的。字段缺失（服务端未下发）按已开通处理，不要据此判断不可用。
- 结果有最长 5 分钟的本地缓存，按登录账号与环境隔离；刚在控制台开通的服务想立刻看到状态用
  `qianfan models list --refresh`（绑定与推理的开通校验会自行强刷一次，无需手动）。
- 未登录返回 `CREDENTIAL_NOT_FOUND`，提示用户执行 `qianfan auth login`。
- 服务端返回非 2xx 归一为 `CONTROL_PLANE_ERROR`；网络失败为 `NETWORK_ERROR`。
- API Key 失效（401/403）时 CLI 自动刷新一次并重试一次，无需技能侧处理。

## 与其他 skill 的串联

- 绑定默认模型：转 [qianfan-profile](../qianfan-profile/SKILL.md)
- 直接推理验证：转 [qianfan-chat](../qianfan-chat/SKILL.md)
- 未登录 / 取不到 API Key：转 [qianfan-auth](../qianfan-auth/SKILL.md) 或 [qianfan-doctor](../qianfan-doctor/SKILL.md)

## 自然语言触发词 + 跨技能指引表

| 用户怎么说 | 走哪个命令/skill |
| --- | --- |
| "有哪些模型 / 支持什么模型 / 列一下 modelId" | `qianfan models list --format json` |
| "把某个模型设成默认" | 转 `qianfan-profile`：`profile set-default --model-type chat <modelId>` |
| "用某个模型回答一下" | 转 `qianfan-chat`：`+chat --model <modelId>` |
| "查某个模型用了多少 token" | 转 `qianfan-service`：`service usage --modelIds <modelId>` |

## 安全与边界

- 只读查询：不创建、不修改任何模型或服务资源。
- 不猜 modelId：列表里没有的 ID 不要自行拼造或推断，绑定时会直接报 `INVALID_ARGUMENT`。
- 不替用户开通：`charge_status` 为 `false` 时只能引导用户去控制台开通，CLI 无开通能力。
- 不输出凭证：API Key 只在 CLI 内部使用，任何输出都不含 Key。

## 参考

- [qianfan-common/references/error-codes.md](../qianfan-common/references/error-codes.md) — 错误码、退出码与处置
