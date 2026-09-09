# qianfan doctor 检查项

`qianfan doctor [--check chat] [--format json]` 依次执行以下检查，输出为
`{ "ok": bool, "checks": [ {name, ok, detail, code}, ... ] }`。

## 检查项与含义

- `config`：是否存在有效配置（已登录）。FAIL → 未登录，执行 `qianfan auth login`。
- `access_key`：访问凭证是否存在。FAIL(`CREDENTIAL_NOT_FOUND`) → 登录。
- `secret_key`：密钥是否可读。FAIL(`CREDENTIAL_NOT_FOUND`) → 登录。
- `bce_v1_sign`：能否用当前临时凭证生成 BCE V1 签名（离线，不发网络请求）。FAIL → 凭证不完整，重新登录。
- `current_profile`：当前 profile 是否设置且存在。
- `chat_model_binding`：当前 profile 是否绑定 chat 模型。FAIL(`MODEL_NOT_CONFIGURED`) →
  `qianfan profile set-default --model-type chat <modelId>`。
- `api_key`：当前 profile 的 API Key 是否可解析（缓存未命中时会请求 openapi-iam）。FAIL 可能为
  `INVALID_ARGUMENT`（当前 profile 为 TokenPlan，尚未接入 API Key）/ `IAM_ERROR` / `IAM_PERMISSION_DENIED` /
  `PACKAGE_NOT_ACTIVATED` / `SEAT_NOT_ASSIGNED` / `API_KEY_NOT_GENERATED` / `API_KEY_PERMISSION_DENIED`。
- `inference`（仅 `--check chat`）：推理连通性探测（以 `max_tokens=1` 发一次极小请求）。FAIL 结合 `code` 判断。
- `skills`：本地技能包安装状态。未安装时提示 `qianfan +connect` 安装。

## 使用建议

1. 先跑 `qianfan doctor --format json`，从上到下定位第一个 `ok=false` 的检查项。
2. 认证类失败先解决（config/access_key/secret_key/bce_v1_sign），再看 profile/模型/API Key。
3. 仅当需要验证真实推理链路时加 `--check chat`（会产生一次极小的推理调用）。
4. 存在 FAIL 项时命令返回 `DIAGNOSTICS_FAILED` 且退出码非零，可直接用于脚本/CI 判定。
5. 按各 FAIL 项的 `code` 转达 `suggestion`，处置见
   [qianfan-common/references/error-codes.md](../../qianfan-common/references/error-codes.md)。
