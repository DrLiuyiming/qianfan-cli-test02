# 千帆 CLI 错误码

CLI 所有失败路径的输出格式一致，`code` 稳定可供脚本识别，`suggestion` 为面向用户的处置引导。
错误始终写入 stderr，形态由命令的 `--format` 决定。

`--format json` 时为信封结构：

```json
{
  "error": {
    "code": "API_KEY_NOT_GENERATED",
    "message": "当前套餐尚未生成可用 API Key",
    "suggestion": "前往 API Key 管理页生成 API Key 后重新执行 qianfan profile use platform",
    "request_id": "..."
  }
}
```

默认（`--format text`，以及命令未定义 `--format` 或 `--format` 取值本身非法时）为两行文本：

```text
error: API_KEY_NOT_GENERATED: 当前套餐尚未生成可用 API Key
hint: 前往 API Key 管理页生成 API Key 后重新执行 qianfan profile use platform
```

无 `suggestion` 时省略 `hint` 行。

## 错误码对照

- `CREDENTIAL_NOT_FOUND`：未登录、本地密钥不可读，或登录已过期且用 refresh token 续期失败
  → 提示用户执行 `qianfan auth login`。
- `ALREADY_LOGGED_IN`：已有账号处于登录状态（CLI 同时只支持一个登录账号）→ 先执行 `qianfan auth logout`
  再执行 `qianfan auth login`；退出码 8。
- `LOGIN_FAILED`：浏览器登录未完成或回调/ID Token 校验失败 → 提示用户在本地终端重新执行
  `qianfan auth login` 并在浏览器中完成授权。
- `AUTH_FAILED`：签名失败或无权限（含 `AccessDenied`、`SignatureDoesNotMatch`）→ CLI 已在取 API Key 时
  强制续期并重试过一次；仍失败说明会话彻底失效或账号缺少权限，需重新登录或检查账号权限。
- `IAM_ERROR`：IAM 接口（API Key 查询/创建/解密）返回错误 → 结合 `message`/`request_id` 排查，必要时重新登录。
- `IAM_PERMISSION_DENIED`：openapi-iam 权限校验失败（签名已通过，服务端返回 `PermissionValidateFailedException`）→
  当前登录身份缺少 API Key 操作权限，需联系 IAM 管理员授权；重新登录或重试均无效，CLI 不会自动续期重试。
- `MODEL_NOT_CONFIGURED`：当前套餐未绑定 chat 模型 → `qianfan profile set-default --model-type chat <modelId>`。
- `MODEL_NOT_ACTIVATED`：目标预置服务未开通后付费调用（服务端 `chargeStatus` 明确为 `false`）→ 前往
  `suggestion` 给出的模型中心地址开通后重试。绑定默认模型与发起推理都会被拦下，且**推理不会发出请求**、
  绑定不会写入配置。CLI 在判定前已强制刷新过一次模型列表，因此重试前必须确认用户真的完成了开通，
  不要原样重试。
- `PACKAGE_NOT_ACTIVATED`：套餐未开通 → 前往套餐订阅页开通，profile 不会切换。
- `SEAT_NOT_ASSIGNED`：企业席位未分配 → 联系企业管理员分配席位，profile 不会切换。
- `API_KEY_NOT_GENERATED`：未生成推理 API Key → 前往 API Key 管理页生成后重试。
- `API_KEY_PERMISSION_DENIED`：API Key 无服务权限 → 为 API Key 配置模型服务权限。
- `API_KEY_INVALID`：推理 API Key 无效 → 重新执行目标 profile 或检查 Key 状态。
- `NETWORK_ERROR`：网络或服务端暂不可用 → 检查网络后重试，缓存与 profile 不变。
- `INVALID_ARGUMENT`：参数或命令非法（如未知命令/子命令、未知 flag、`--dry-run` 与 `--yes` 同时使用、
  未知 profile、非法 `--format`、非法 `--model-type`、`service usage` 的 `--interval`/`--protocolVersion`
  取值不在允许集合），以及**对 TokenPlan 套餐取 API Key**（"当前套餐（TokenPlan）暂未接入 API Key"，
  切回 `platform` 后重试）→ 检查命令与参数。
- `INFERENCE_ERROR`：推理 API 返回错误或无候选结果 → 检查模型是否支持当前请求。
- `CONTROL_PLANE_ERROR`：管控 API 返回未归类错误 → 结合 `message`/`request_id` 排查。
- `DIAGNOSTICS_FAILED`：`qianfan doctor` 存在未通过检查项（命令以非零退出码结束）→ 逐个处理 FAIL 项。
- `CONFIRMATION_REQUIRED`：非交互环境未提供 `--yes` → 先 `--dry-run` 检查计划，再以 `--yes` 执行。
- `OPERATION_CANCELLED`：用户拒绝或取消确认 → 操作未执行，无需补偿。
- `INTERNAL_ERROR`：未归类的内部错误（如本地配置目录不可用导致依赖装配失败）→ 结合 `message` 排查本地环境。

## 退出码

脚本与 CI 可直接按退出码分流，取值来自 `clierr.ExitCode`：

| 退出码 | 含义 |
| --- | --- |
| 0 | 成功 |
| 1 | 其他错误（未在下表列出的错误码） |
| 2 | `INVALID_ARGUMENT` |
| 3 | `CREDENTIAL_NOT_FOUND` |
| 4 | `AUTH_FAILED` |
| 5 | `CONFIRMATION_REQUIRED` |
| 6 | `OPERATION_CANCELLED` |
| 7 | `DIAGNOSTICS_FAILED` |
| 8 | `ALREADY_LOGGED_IN` |

## 处理原则

1. 按 `code` 分类处置，向用户转达 `suggestion`；不要对同一失败命令原样重试。
2. `401/403` 类推理错误由 CLI 内部自动刷新一次 API Key 并重试一次，无需在技能侧重复处理。
3. 权限/套餐/席位类错误（`PACKAGE_NOT_ACTIVATED`、`SEAT_NOT_ASSIGNED`、`API_KEY_*`、`MODEL_NOT_ACTIVATED`）
   均需用户在控制台处理，CLI 不会切换 profile、伪造 Key 或替用户开通服务。
