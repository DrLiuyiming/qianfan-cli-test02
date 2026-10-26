---
name: qianfan-embedding
description: "通过千帆 CLI（qianfan +embedding）把一段或多段文本转成向量（embedding），支持临时指定模型与套餐，JSON 输出返回每段文本的向量与 token 用量。当用户想用千帆做文本向量化/做 embedding/算语义向量、提到 qianfan +embedding，或 Agent 流程里需要为检索/相似度/RAG 生成向量时使用。反触发：报 MODEL_NOT_CONFIGURED 需要先绑定默认向量模型时转 qianfan-profile；想知道有哪些模型转 qianfan-models；要做文本对话转 qianfan-chat。"
license: Apache-2.0
metadata:
  requires:
    bins: ["qianfan"]
  cliHelp: "qianfan +embedding --help"
  install:
    - kind: npm
      package: "@baiducloud/qianfan-cli"
---

# Qianfan 向量生成

先读 [qianfan-common](../qianfan-common/SKILL.md) 的公共规则。推理使用按 profile 解析的 API Key，由 CLI 自动获取与缓存。

## 何时使用本 skill

- 把文本转成向量，用于语义检索、相似度计算、RAG 入库
- 一次为多段文本批量生成向量
- 验证刚绑定的默认 embedding 模型是否可用

## 命令一览

| 命令 | 说明 |
| --- | --- |
| `qianfan +embedding <text> [text...] --format json` | 向量化一段或多段文本，返回 `{request_id, model, data:[{index, embedding}], usage}` |
| `qianfan +embedding <text> --model <modelId>` | 临时指定向量模型，不落盘 |
| `qianfan +embedding <text> --profile platform` | 临时指定套餐，不写入 `current_profile` |

text 模式仅打印每条向量的维数与前几维预览；要拿完整向量一律用 `--format json`。`data[].index` 与输入顺序一一对应。

模型解析优先级：`--model` > 当前/指定 profile 的 `model_bindings.embedding` > 报 `MODEL_NOT_CONFIGURED`。

## Agent 快速执行顺序

1. Agent 场景固定带 `--format json`，从 `data[].embedding` 取向量（按 `index` 对齐输入顺序）
2. 报 `MODEL_NOT_CONFIGURED` → 用 [qianfan-models](../qianfan-models/SKILL.md) 找一个向量 modelId，再 `qianfan profile set-default --model-type embedding <modelId>`，或本次 `--model` 指定
3. 报 `MODEL_NOT_ACTIVATED` → 该预置服务未开通，把 `suggestion` 里的开通地址转达用户，开通后再重试
4. 报 `CREDENTIAL_NOT_FOUND` → 转 [qianfan-auth](../qianfan-auth/SKILL.md) 引导用户本地登录

## 常用调用

```bash
qianfan +embedding --format json "白色 T 恤"
qianfan +embedding --format json "第一段文本" "第二段文本" "第三段文本"
qianfan +embedding --model <modelId> --format json "待向量化的文本"
```

## 错误处理

| 错误码 | 处置 |
| --- | --- |
| `MODEL_NOT_CONFIGURED` | 先绑定默认向量模型 `qianfan profile set-default --model-type embedding <modelId>`，或本次用 `--model` 指定 |
| `MODEL_NOT_ACTIVATED` | 目标预置服务未开通，按 `suggestion` 去模型中心开通后重试；这一次请求没有发出 |
| `CREDENTIAL_NOT_FOUND` | 提示用户 `qianfan auth login` |
| `INVALID_ARGUMENT` | 缺少待向量化文本，或 `--profile` 指向不可用套餐，按 message 调整 |
| `INFERENCE_ERROR` | 向量 API 报错，检查模型是否支持当前请求 |
| `RATE_LIMIT_EXCEEDED` | 触发限流，降低频率后重试 |
| `NETWORK_ERROR` | 网络或服务端暂不可用，稍后重试 |

向量 `401/403` 由 CLI 内部自动刷新一次 API Key 并重试一次，技能侧无需处理。

## 与其他 skill 的串联

- 缺默认模型 / 想换套餐：[qianfan-profile](../qianfan-profile/SKILL.md)
- 找向量 modelId：[qianfan-models](../qianfan-models/SKILL.md)
- 登录与凭证：[qianfan-auth](../qianfan-auth/SKILL.md)
- 想看这次调用消耗了多少 token：[qianfan-service](../qianfan-service/SKILL.md)

## 自然语言触发词 + 跨技能指引表

| 用户怎么说 | 走哪个命令/skill |
| --- | --- |
| "把这段文本做成向量 / embedding" | `qianfan +embedding --format json "..."` |
| "一次给多段文本算向量" | `qianfan +embedding --format json "a" "b" "c"` |
| "把向量模型设成默认" | 转 `qianfan-profile`：`profile set-default --model-type embedding <modelId>` |
| "有哪些向量模型" | 转 `qianfan-models` |

## 安全与边界

- 会产生真实 token 消耗：批量向量化前先把命令与预计条数告诉用户。
- 不传敏感数据：不要把用户凭证、密钥或本地敏感文件内容拼进待向量化文本。
- 不改本地配置：`--model` / `--profile` 只对本次调用生效。

## 参考

- [qianfan-common/references/error-codes.md](../qianfan-common/references/error-codes.md) — 错误码、退出码与处置
