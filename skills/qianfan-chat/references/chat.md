# qianfan +chat 参数与输出

## 命令

```
qianfan +chat <prompt> [flags]
```

## 参数

- `prompt`（必填）：用户消息。
- `--model <modelId>`：临时指定模型，覆盖 profile 的 chat 绑定，不落盘。
- `--profile <profile>`：临时套餐，不写入 `current_profile`。当前只有 `platform` 可用；
  `tokenplan_personal`/`tokenplan_enterprise` 尚未接入 API Key，指定后返回 `INVALID_ARGUMENT`。
- `--system <text>`：system message。
- `--stream`：开启 SSE 流式输出。
- `--temperature <float>`：采样温度（不传则不下发）。
- `--top-p <float>`：nucleus 采样概率（不传则不下发）。
- `--max-tokens <int>`：最大生成 token 数（不传则不下发）。
- `--format text|json`：输出格式，默认 `text`。

## 输出格式

非流式 `--format json`：

```json
{
  "request_id": "...",
  "model": "deepseek-v3.1-250821",
  "content": "..."
}
```

- text 模式：直接打印 `content`。
- `--stream` + text：逐片段实时打印，结束换行。
- `--stream` + json：累积完整内容后输出一次上述结构。

## 示例

```bash
qianfan +chat --format json "你好"
qianfan +chat --model deepseek-v3.1-250821 --format json "你好"
qianfan +chat --profile platform --format json "你好"
qianfan +chat --system "You are a helpful assistant." --stream "解释 RAG"
qianfan +chat --temperature 0.7 --top-p 0.9 --max-tokens 1024 --format json "写一段介绍"
```

## 说明

- 推理走 `POST <推理域名>/v2/chat/completions`，Bearer API Key 认证。推理域名与控制面**分开配置**：
  优先 `QIANFAN_INFERENCE_ENDPOINT`，否则按当前登录态解析推理地址，不复用 API Key 缓存记录里的
  控制面 `base_url`；API Key 本身取自本地缓存。
- `401/403` 时 CLI 自动在线重取一次 API Key 并仅重试一次（重取成功才覆盖缓存，失败保留原 Key）；
  其余 4xx 不重试，归一为 `INFERENCE_ERROR`。
- 推理返回体无候选结果时同样返回 `INFERENCE_ERROR`；网络失败返回 `NETWORK_ERROR`。
- 未绑定 chat 模型时返回 `MODEL_NOT_CONFIGURED`，先执行
  `qianfan profile set-default --model-type chat <modelId>`。
- 发请求前 CLI 会校验目标模型：类型必须可用于 chat，且该预置服务的 `chargeStatus` 不能为 `false`。
  未开通时返回 `MODEL_NOT_ACTIVATED` 并**不发出推理请求**（不消耗 token）；判定前会强制刷新一次模型列表，
  所以刚在控制台开通即可生效，无需等缓存过期。服务端未返回该字段时按已开通放行。
