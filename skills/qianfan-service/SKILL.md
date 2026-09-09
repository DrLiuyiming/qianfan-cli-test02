---
name: qianfan-service
version: 1.0.0
description: "查询千帆服务调用统计（qianfan service usage）：按模型/服务/应用维度的调用次数、成功失败数与 token 消耗，支持时间范围、按天/小时/分钟粒度与指定 modelIds。当用户问千帆用量、调用量、消耗了多少 token、服务调用统计、看看这段时间的调用情况，或要核对某个 modelId 的调用明细时使用。反触发：只是想发起一次推理转 qianfan-chat；查有哪些模型转 qianfan-models。"
license: Apache-2.0
metadata:
  requires:
    bins: ["qianfan"]
  cliHelp: "qianfan service usage --help"
  install:
    - kind: npm
      package: "@baiducloud/qianfan-cli"
---

# Qianfan 服务调用统计

先读 [qianfan-common](../qianfan-common/SKILL.md) 的公共规则。用量查询走管控接口，但与推理一样用
**API Key（Bearer）认证**，不需要本地 Secret Key；统计维度为「当前 profile 解析出的 API Key + modelId」，
因此需已登录且**当前 profile 为 platform**（TokenPlan 套餐取不到 API Key，会返回 `INVALID_ARGUMENT`）。

## 何时使用本 skill

- 查某段时间的调用次数、成功/失败数、token 消耗
- 按天/小时/分钟粒度看调用趋势
- 核对某个或某几个 modelId 的调用明细
- 排查"调用是否真的打到了服务端""失败集中在什么时候"

## 命令一览

| 命令 / 参数 | 说明 |
| --- | --- |
| `qianfan service usage --startTime <t> --endTime <t>` | 必填时间范围，见下方"时间格式" |
| `--modelIds <id1>,<id2>` | 只看指定模型，逗号分隔 |
| `--interval 86400\|3600\|60` | 时间粒度（秒），默认 `86400`（天） |
| `--protocolVersion 1\|2` | 服务版本，默认 `1` |
| `--format text\|json` | 输出格式，默认 `text` |

## 时间格式

输入接受四种写法，CLI 统一换算成 UTC 的 RFC3339 后发给服务端：

- `2026-08-19T16:00:00Z` — UTC，原样使用
- `2026-08-20T00:00:00+08:00` — 带偏移，按该偏移换算
- `2026-08-20 00:00:00` / `2026-08-20T00:00:00` — 不带时区，按**执行机器的本地时区**解释
- `2026-08-20` — 本地时区当天零点

无法解析时返回 `INVALID_ARGUMENT`，`suggestion` 里列出全部可接受写法。响应里的 `startTime`/`endTime`
回显的是换算后的 UTC 值，所以传本地时间时看到的回显会有时差，这是正常的。

## Agent 快速执行顺序

1. 确认当前 profile 是 `platform`（`qianfan profile show --format json`）
2. 把用户的自然语言时间范围换算成上述任一格式的起止时间（跨时区场景建议直接给 UTC 的 `...Z`，避免依赖执行机器时区）
3. 执行 `qianfan service usage --startTime ... --endTime ... --format json`
4. 汇总时遍历 `items`：每条是「模型 + Key + 时间桶」的聚合，按 `model` 求和得总量，按 `time` 排序看趋势

## 示例

```bash
qianfan service usage --startTime "2026-08-01 00:00:00" --endTime "2026-08-21 00:00:00" --format json

qianfan service usage --modelIds <modelId1>,<modelId2> \
  --startTime "2026-08-20 00:00:00" --endTime "2026-08-21 00:00:00" --format json

qianfan service usage --interval 3600 \
  --startTime "2026-08-20 00:00:00" --endTime "2026-08-21 00:00:00" --format json
```

`--interval` 只接受 86400/3600/60，`--protocolVersion` 只接受 1/2，其他取值返回 `INVALID_ARGUMENT`。

## 输出结构

```json
{
  "startTime": "2026-08-18T16:00:00Z",
  "endTime": "2026-08-27T07:59:59Z",
  "items": [
    {
      "model": "deepseek-v4-pro",
      "accessKeyId": "ALTAK-xxxxxxxx",
      "time": 1787068800,
      "callTotal": 6, "succeedCall": 0, "failureCall": 6,
      "totalTokens": 0, "inputTokens": 0, "outputTokens": 0,
      "searchTokens": 0, "cachedTokens": 0, "reasoningTokens": 0
    }
  ]
}
```

`items` 是拉平的聚合记录，一条代表「某个模型 + 某把 Key + 某个时间桶」，没有服务/应用层嵌套。
`time` 是时间桶起点的 Unix 秒（桶宽由 `--interval` 决定），`accessKeyId` 是发起调用那把 Key 的 AK 段。
区间内没有调用时 `items` 为空数组，text 模式打印 `(no usage in this range)`。
text 模式每条记录一行：时间桶（UTC RFC3339）、模型、调用数、token 数。

## 错误处理

| 错误码 | 处置 |
| --- | --- |
| `INVALID_ARGUMENT` | 缺 `--startTime`/`--endTime`、`--interval`/`--protocolVersion` 非法，或当前 profile 为 TokenPlan（先 `qianfan profile use platform`） |
| `CREDENTIAL_NOT_FOUND` | 未登录或登录已过期且续期失败 → 提示 `qianfan auth login` |
| `AUTH_FAILED` | API Key 无效或无权限（CLI 已自动刷新 Key 重试过一次）→ 重新登录或检查账号权限 |
| `PACKAGE_NOT_ACTIVATED` / `SEAT_NOT_ASSIGNED` / `API_KEY_NOT_GENERATED` | 当前套餐解析不出 API Key，按 `suggestion` 去控制台处理，可先用 qianfan-doctor 定位 |
| `CONTROL_PLANE_ERROR` | 管控 API 返回未归类错误，结合 `message`/`request_id` 排查 |

## 与其他 skill 的串联

- 当前套餐不对：转 [qianfan-profile](../qianfan-profile/SKILL.md)
- 未登录 / 凭证问题：转 [qianfan-auth](../qianfan-auth/SKILL.md)
- 取不到 API Key 需要定位：转 [qianfan-doctor](../qianfan-doctor/SKILL.md)
- 需要 modelId：转 [qianfan-models](../qianfan-models/SKILL.md)

## 自然语言触发词 + 跨技能指引表

| 用户怎么说 | 走哪个命令/skill |
| --- | --- |
| "这个月用了多少 token / 用量" | `qianfan service usage --startTime ... --endTime ...` |
| "按小时看昨天的调用" | 加 `--interval 3600` |
| "某个模型调了多少次" | 加 `--modelIds <modelId>` |
| "失败率怎么样" | 读每条 `items[].failureCall` / `succeedCall` |
| "费用多少 / 账单" | 说明 CLI 只提供调用量与 token 统计，计费金额请查控制台 |

## 安全与边界

- 只读查询：不改任何远端或本地状态，不消耗推理 token。
- 不做计费换算：CLI 不返回金额，不要按 token 数替用户估算费用。
- 时间范围要用户确认：换算自然语言时间（"上周""本月"）后先说明用的具体起止时间。
- 统计范围锁定当前登录的那把 Key：请求里的 `apiKeyPrefix` 由 CLI 用当前 profile 的 API Key 的 AK 段填充，
  无法指定其他 Key，也没有全账号汇总。同一账号下换过 Key（例如登出重登后 CLI 重新创建了 Key）时，
  旧 Key 产生的调用查不到，响应里的 `items[].accessKeyId` 就是当次统计所属的那把 Key，可用来核对。
- 输出只含 AK 段：`accessKeyId` 与请求里的 `apiKeyPrefix` 都只是 API Key 的中间段，不含密文段，
  不构成可用凭证；完整 Key 始终只在 CLI 内部的 Authorization 头里。

## 参考

- [qianfan-common/references/error-codes.md](../qianfan-common/references/error-codes.md) — 错误码、退出码与处置
