---
name: qianfan-video
description: "通过千帆 CLI（qianfan +video）生成视频：异步任务，默认创建任务并返回 taskId，可用 --task 查询结果或加 --wait 轮询到完成，支持文本提示、输入图片（图生视频）、时长/水印/提示词改写、临时指定模型与套餐。当用户想用千帆文生视频/图生视频、提到 qianfan +video、生成一段视频、把图片变成视频，或 Agent 流程里需要生成视频时使用。反触发：报 MODEL_NOT_CONFIGURED 需要先绑定默认视频模型时转 qianfan-profile；想知道有哪些模型转 qianfan-models；只要生成静态图片转 qianfan-gen。"
license: Apache-2.0
metadata:
  requires:
    bins: ["qianfan"]
  cliHelp: "qianfan +video --help"
  install:
    - kind: npm
      package: "@baiducloud/qianfan-cli"
---

# Qianfan 视频生成

先读 [qianfan-common](../qianfan-common/SKILL.md) 的公共规则。推理使用按 profile 解析的 API Key，由 CLI 自动获取与缓存。

视频生成是**异步任务**：先创建任务拿到 `taskId`，再按 `taskId` 轮询状态，成功后拿到视频下载地址。
视频地址通常有效期有限，拿到后请尽快转存。

## 何时使用本 skill

- 用千帆模型根据文字描述或一张图片生成视频
- 查询之前创建的视频任务进展与结果
- 验证刚绑定的默认 video 模型是否可用

## 命令一览

| 命令 | 说明 |
| --- | --- |
| `qianfan +video <prompt> --image <url|path> --format json` | 创建任务，返回 `{task_id}` |
| `qianfan +video <prompt> --image <url|path> --wait` | 创建后原地轮询直到任务结束，成功打印视频 URL |
| `qianfan +video --task <taskId> --format json` | 查询已有任务，返回 `{task_id, model, status, video_url, error, ...}` |
| `qianfan +video <prompt> --model <modelId>` | 临时指定视频模型，不落盘 |

常用标志：`--duration <秒>`、`--watermark`、`--prompt-extend`（提示词智能改写）、`--poll-interval <秒>`（默认 10）、
`--poll-timeout <秒>`（默认 600，0 表示不设上限）。

- `--image` 接受图片 URL、`data:` 前缀的 base64，或本地文件路径（本地文件会被读入并编码）；图生视频模型要求提供图片。
- 创建任务时 `prompt` 与 `--image` 至少提供其一。
- 任务状态 `status`：排队/生成中（如 `queued` / `running`）、成功（`succeeded`）、失败（`failed`）等；仅成功后才有 `video_url`。

模型解析优先级：`--model` > 当前/指定 profile 的 `model_bindings.video` > 报 `MODEL_NOT_CONFIGURED`。

## Agent 快速执行顺序

1. 创建任务固定带 `--format json`，记下返回的 `task_id`
2. 轮询：优先 `--wait` 让 CLI 代为轮询；或自行 `qianfan +video --task <taskId> --format json`，`status=succeeded` 后读 `video_url`
3. 刚创建后立即查询可能只返回 `task_id`、暂无 `status`，属正常；按 `--poll-interval`（建议 ≥10 秒）间隔再查
4. 报 `MODEL_NOT_CONFIGURED` → 用 [qianfan-models](../qianfan-models/SKILL.md) 找一个视频 modelId，再 `qianfan profile set-default --model-type video <modelId>`，或本次 `--model` 指定
5. 报 `MODEL_NOT_ACTIVATED` → 该预置服务未开通，把 `suggestion` 里的开通地址转达用户，开通后再重试

## 常用调用

```bash
# 图生视频：创建任务并等待完成
qianfan +video --image ./first-frame.png --wait --format json "镜头缓缓拉远，粒子光尾飘动"

# 仅创建任务，稍后再查
qianfan +video --image https://example.com/a.jpg --format json "海浪拍打礁石"
qianfan +video --task <taskId> --format json

# 临时指定模型与时长
qianfan +video --model <modelId> --image ./a.png --duration 5 --format json "樱花飘落"
```

## 错误处理

| 错误码 | 处置 |
| --- | --- |
| `MODEL_NOT_CONFIGURED` | 先绑定默认视频模型 `qianfan profile set-default --model-type video <modelId>`，或本次用 `--model` 指定 |
| `INVALID_ARGUMENT` | 既无 prompt 也无 `--image`、`--image` 本地文件读不到或格式不支持（支持 JPEG/PNG/WEBP）、`--profile` 不可用、`--task` 查询模式又传了 prompt 等，按 message 调整 |
| `MODEL_NOT_ACTIVATED` | 目标预置服务未开通，按 `suggestion` 去模型中心开通后重试；这一次请求没有发出 |
| `CREDENTIAL_NOT_FOUND` | 提示用户 `qianfan auth login` |
| `INFERENCE_ERROR` | 任务创建/查询报错，或 `--wait` 期间任务以失败态结束（会带上服务端 `error.message`）；按提示调整输入或模型 |
| `RATE_LIMIT_EXCEEDED` | 触发限流，降低频率后重试 |
| `NETWORK_ERROR` | 网络或服务端暂不可用，稍后重试 |

视频 `401/403` 由 CLI 内部自动刷新一次 API Key 并重试一次，技能侧无需处理。
`--wait` 超时或被取消时任务仍在服务端继续，可稍后用 `qianfan +video --task <taskId>` 继续查询。

## 与其他 skill 的串联

- 缺默认模型 / 想换套餐：[qianfan-profile](../qianfan-profile/SKILL.md)
- 找视频 modelId：[qianfan-models](../qianfan-models/SKILL.md)
- 登录与凭证：[qianfan-auth](../qianfan-auth/SKILL.md)
- 只要生成静态图片：[qianfan-gen](../qianfan-gen/SKILL.md)

## 自然语言触发词 + 跨技能指引表

| 用户怎么说 | 走哪个命令/skill |
| --- | --- |
| "生成一段视频 / 文生视频 / 把这张图做成视频" | `qianfan +video ... --format json` |
| "查一下那个视频任务好了没" | `qianfan +video --task <taskId> --format json` |
| "等它生成完给我地址" | `qianfan +video ... --wait` |
| "把视频模型设成默认" | 转 `qianfan-profile`：`profile set-default --model-type video <modelId>` |
| "有哪些视频模型" | 转 `qianfan-models` |

## 安全与边界

- 会产生真实调用消耗、且耗时较长：创建任务与轮询前先把命令、预计时长与轮询频率告诉用户。
- 不改本地配置：`--model` / `--profile` 只对本次调用生效。
- 不无限轮询：遵守 `--poll-interval` / `--poll-timeout`，超时后转为按需 `--task` 查询，不要空转。
- 视频地址有效期有限：拿到 `video_url` 后提醒用户尽快下载或转存。

## 参考

- [qianfan-common/references/error-codes.md](../qianfan-common/references/error-codes.md) — 错误码、退出码与处置
