---
name: qianfan-chat
version: 1.0.0
description: "通过千帆 CLI（qianfan +chat）调用文本推理模型，支持一次性回复与 SSE 流式输出、system 提示、temperature/top-p/max-tokens 采样参数、临时指定模型与套餐。当用户想用千帆做文本生成/对话/问答、提到 qianfan +chat、用千帆跑一个 prompt、流式输出，或 Agent 流程里需要调用千帆大模型生成文本时使用。反触发：报 MODEL_NOT_CONFIGURED 需要先绑定默认模型时转 qianfan-profile；想知道有哪些模型转 qianfan-models。"
license: Apache-2.0
metadata:
  requires:
    bins: ["qianfan"]
  cliHelp: "qianfan +chat --help"
  install:
    - kind: npm
      package: "@baiducloud/qianfan-cli"
---

# Qianfan 文本推理

先读 [qianfan-common](../qianfan-common/SKILL.md) 的公共规则。推理使用按 profile 解析的 API Key
（`Authorization: Bearer`），由 CLI 自动获取与缓存；管控用的 STS 临时凭证不参与推理调用。

## 何时使用本 skill

- 用千帆模型做文本生成、问答、改写、总结
- 需要流式输出（边生成边打印）
- 需要临时换模型或换套餐跑一次，不改本地默认配置
- 验证刚绑定的默认 chat 模型是否可用

## 命令一览

| 命令 | 说明 |
| --- | --- |
| `qianfan +chat <prompt> --format json` | 非流式推理，返回 `{request_id, model, content}` |
| `qianfan +chat <prompt> --stream` | SSE 流式输出，text 模式逐片段实时打印 |
| `qianfan +chat <prompt> --model <modelId>` | 临时指定模型，不落盘 |
| `qianfan +chat <prompt> --profile platform` | 临时指定套餐，不写入 `current_profile` |
| `qianfan +chat <prompt> --system <text>` | 附加 system message |

采样参数：`--temperature`、`--top-p`、`--max-tokens`，不传则不下发（用服务端默认）。

## Agent 快速执行顺序

1. Agent 场景固定带 `--format json`，只读取 `content`（需要溯源时再看 `request_id`、`model`）
2. 报 `MODEL_NOT_CONFIGURED` → 转 [qianfan-profile](../qianfan-profile/SKILL.md) 绑定默认 chat 模型后重试
3. 报 `MODEL_NOT_ACTIVATED` → 该预置服务未开通，把 `suggestion` 里的开通地址转达用户；开通后才重试，不要空转
4. 报 `CREDENTIAL_NOT_FOUND` → 转 [qianfan-auth](../qianfan-auth/SKILL.md) 引导用户本地登录
5. 报 `INVALID_ARGUMENT` 且用了 `--profile` → 只有 `platform` 可用，去掉该参数或改成 platform

## 常用调用

```bash
qianfan +chat --format json "你好"
qianfan +chat --model <modelId> --format json "你好"
qianfan +chat --profile platform --format json "你好"
qianfan +chat --system "You are a helpful assistant." "解释 RAG"
qianfan +chat --stream "写一首诗"
qianfan +chat --temperature 0.7 --top-p 0.9 --max-tokens 1024 --format json "写一段介绍"
```

模型解析优先级：`--model` > 当前/指定 profile 的 `model_bindings.chat` > 报 `MODEL_NOT_CONFIGURED`。
完整参数、输出形态与失效重试细节见 [references/chat.md](references/chat.md)。

## 错误处理

| 错误码 | 处置 |
| --- | --- |
| `MODEL_NOT_CONFIGURED` | 先 `qianfan profile set-default --model-type chat <modelId>` |
| `MODEL_NOT_ACTIVATED` | 目标预置服务未开通，按 `suggestion` 去模型中心开通后重试；**这一次请求没有发出**，不消耗 token |
| `CREDENTIAL_NOT_FOUND` | 提示用户 `qianfan auth login`（过期时 CLI 已尝试自动续期） |
| `INVALID_ARGUMENT` | `--profile` 指向不存在的 profile，或指向尚未接入 API Key 的 TokenPlan 套餐 |
| `INFERENCE_ERROR` | 推理 API 报错或未返回候选结果，检查模型是否支持当前请求 |
| `NETWORK_ERROR` | 网络或服务端暂不可用，稍后重试 |

推理 `401/403` 由 CLI 内部自动刷新一次 API Key 并重试一次，技能侧无需处理。

## 与其他 skill 的串联

- 缺默认模型 / 想换套餐：[qianfan-profile](../qianfan-profile/SKILL.md)
- 找 modelId：[qianfan-models](../qianfan-models/SKILL.md)
- 登录与凭证：[qianfan-auth](../qianfan-auth/SKILL.md)
- 推理不通要定位根因：[qianfan-doctor](../qianfan-doctor/SKILL.md)（`doctor --check chat` 会做连通性探测）
- 想看这次调用消耗了多少 token：[qianfan-service](../qianfan-service/SKILL.md)

## 自然语言触发词 + 跨技能指引表

| 用户怎么说 | 走哪个命令/skill |
| --- | --- |
| "帮我用千帆回答/生成/总结…" | `qianfan +chat --format json "..."` |
| "边生成边输出 / 流式" | `qianfan +chat --stream "..."` |
| "用 xxx 模型试一下" | `qianfan +chat --model <modelId>` |
| "没配默认模型怎么办" | 转 `qianfan-profile` |
| "调用消耗了多少 token" | 转 `qianfan-service` |

## 安全与边界

- 会产生真实 token 消耗：批量或长文本调用前先把命令与预计次数告诉用户。
- 不传敏感数据：不要把用户凭证、密钥或本地敏感文件内容拼进 prompt。
- 不改本地配置：`--model` / `--profile` 只对本次调用生效，不写入 `current_profile`。
- 不重复重试：4xx 类错误（除 401/403 由 CLI 内部处理）不要原样重试，先按 `code` 处置。

## 参考

- `references/chat.md` — 完整参数、输出格式、示例与失效重试语义
- [qianfan-common/references/error-codes.md](../qianfan-common/references/error-codes.md) — 错误码、退出码与处置
