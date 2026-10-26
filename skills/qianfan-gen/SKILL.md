---
name: qianfan-gen
description: "通过千帆 CLI（qianfan +gen）用文本提示词生成图像，支持负向提示、输出分辨率与张数、临时指定模型与套餐。当用户想用千帆文生图/画图/生成图片、提到 qianfan +gen、text2image，或 Agent 流程里需要根据描述生成图像时使用。反触发：报 MODEL_NOT_CONFIGURED 需要先绑定默认生图模型时转 qianfan-profile；想知道有哪些模型转 qianfan-models；要生成视频转 qianfan-video。"
license: Apache-2.0
metadata:
  requires:
    bins: ["qianfan"]
  cliHelp: "qianfan +gen --help"
  install:
    - kind: npm
      package: "@baiducloud/qianfan-cli"
---

# Qianfan 图像生成

先读 [qianfan-common](../qianfan-common/SKILL.md) 的公共规则。推理使用按 profile 解析的 API Key，由 CLI 自动获取与缓存。

## 何时使用本 skill

- 用千帆模型根据文字描述生成图像
- 需要指定负向提示、分辨率或一次生成多张
- 验证刚绑定的默认 image 模型是否可用

## 命令一览

| 命令 | 说明 |
| --- | --- |
| `qianfan +gen <prompt> --format json` | 生图，返回 `{request_id, model, created, data:[{index, url, b64_json}]}` |
| `qianfan +gen <prompt> --model <modelId>` | 临时指定生图模型，不落盘 |
| `qianfan +gen <prompt> --negative-prompt <text>` | 指定不希望出现的内容 |
| `qianfan +gen <prompt> --size 1024x1024` | 指定输出分辨率 |
| `qianfan +gen <prompt> --n 2` | 一次生成多张 |

模型解析优先级：`--model` > 当前/指定 profile 的 `model_bindings.image` > 报 `MODEL_NOT_CONFIGURED`。

## Agent 快速执行顺序

1. Agent 场景固定带 `--format json`，从 `data[].url` 取图片地址（部分模型以 `b64_json` 返回，此时 text 模式会提示改用 JSON）
2. 报 `MODEL_NOT_CONFIGURED` → 用 [qianfan-models](../qianfan-models/SKILL.md) 找一个生图 modelId，再 `qianfan profile set-default --model-type image <modelId>`，或本次直接 `--model` 指定
3. 报 `MODEL_NOT_ACTIVATED` → 该预置服务未开通，把 `suggestion` 里的开通地址转达用户，开通后再重试
4. 报 `CREDENTIAL_NOT_FOUND` → 转 [qianfan-auth](../qianfan-auth/SKILL.md) 引导用户本地登录

## 常用调用

```bash
qianfan +gen --format json "一只戴墨镜的柴犬，扁平插画风格"
qianfan +gen --model <modelId> --size 1024x1024 --format json "赛博朋克城市夜景"
qianfan +gen --negative-prompt "文字, 水印" --n 2 --format json "水彩风格的山谷"
```

## 错误处理

| 错误码 | 处置 |
| --- | --- |
| `MODEL_NOT_CONFIGURED` | 先绑定默认生图模型 `qianfan profile set-default --model-type image <modelId>`，或本次用 `--model` 指定 |
| `MODEL_NOT_ACTIVATED` | 目标预置服务未开通，按 `suggestion` 去模型中心开通后重试；这一次请求没有发出 |
| `CREDENTIAL_NOT_FOUND` | 提示用户 `qianfan auth login` |
| `INVALID_ARGUMENT` | 参数非法（如 `--profile` 指向不可用套餐、`--size` 取值不被模型支持），按 message 调整 |
| `INFERENCE_ERROR` | 生图 API 报错，检查模型是否支持当前请求 |
| `RATE_LIMIT_EXCEEDED` | 触发限流，降低频率后重试 |
| `NETWORK_ERROR` | 网络或服务端暂不可用，稍后重试 |

生图 `401/403` 由 CLI 内部自动刷新一次 API Key 并重试一次，技能侧无需处理。

## 与其他 skill 的串联

- 缺默认模型 / 想换套餐：[qianfan-profile](../qianfan-profile/SKILL.md)
- 找生图 modelId：[qianfan-models](../qianfan-models/SKILL.md)
- 登录与凭证：[qianfan-auth](../qianfan-auth/SKILL.md)
- 生成视频：[qianfan-video](../qianfan-video/SKILL.md)

## 自然语言触发词 + 跨技能指引表

| 用户怎么说 | 走哪个命令/skill |
| --- | --- |
| "帮我画 / 生成一张图 / 文生图" | `qianfan +gen --format json "..."` |
| "用 xxx 模型生成图片" | `qianfan +gen --model <modelId> "..."` |
| "把生图模型设成默认" | 转 `qianfan-profile`：`profile set-default --model-type image <modelId>` |
| "有哪些生图模型" | 转 `qianfan-models` |

## 安全与边界

- 会产生真实 token / 调用消耗：批量或多张生成前先把命令与预计次数告诉用户。
- 不改本地配置：`--model` / `--profile` 只对本次调用生效。
- 不重复硬重试：4xx 类错误（除 401/403 由 CLI 内部处理）先按 `code` 处置再决定是否重试。

## 参考

- [qianfan-common/references/error-codes.md](../qianfan-common/references/error-codes.md) — 错误码、退出码与处置
