---
name: qianfan-endpoint
description: "管理千帆自定义服务接入点（qianfan endpoint）：查询/查看/创建/修改/停用接入点，配置每接入点的 RPM/TPM/TPMInput/TPMOutput 限流，并列出预置服务拿 serviceId。当用户提到千帆接入点、自定义接入点、endpoint、开接入点、改限流、停用接入点、查预置服务/serviceId、按服务创建接入点时使用。反触发：只是想发起推理（对话/向量/文生图/视频）转对应推理 skill；查可绑定模型 modelId 转 qianfan-models；查调用量转 qianfan-service。"
license: Apache-2.0
metadata:
  requires:
    bins: ["qianfan"]
  cliHelp: "qianfan endpoint --help"
  install:
    - kind: npm
      package: "@baiducloud/qianfan-cli"
---

# Qianfan 自定义接入点管理

先读 [qianfan-common](../qianfan-common/SKILL.md) 的公共规则。接入点管控走**管控面 AK/SK 签名**
（用当前登录账号的 STS 临时凭证签名），与推理、用量统计的 API Key（Bearer）链路相互独立。
因此只需**已登录**即可（`qianfan auth status --format json` 的 `configured=true`），不要求特定 profile。

## 何时使用本 skill

- 查看、创建、修改、停用自定义服务接入点
- 调整某个接入点的 RPM / TPM / 输入 token / 输出 token 限流
- 创建接入点前，先列出预置服务拿 `serviceId`

## 命令一览

| 命令 / 参数 | 说明 |
| --- | --- |
| `qianfan endpoint services` | 列预置服务，拿 `serviceId`（创建接入点的入参） |
| `qianfan endpoint list` | 列自定义接入点（含限流） |
| `qianfan endpoint get <endpointId>` | 查单个接入点详情 |
| `qianfan endpoint create ...` | 基于预置服务创建接入点 |
| `qianfan endpoint modify <endpointId> ...` | 改名称/描述/限流 |
| `qianfan endpoint stop <endpointId>` | 停用（Deactivate）接入点 |

所有子命令支持 `--format text|json`（默认 `text`）。

### services 过滤

- `--type <t1,t2>`：按服务类型过滤，取值 `chat` / `completions` / `text2image` / `image2text`
- `--protocol-version 1|2`：服务协议版本，默认 `2`
- 输出每行：`serviceId  name  type=<类型>  charge=<计费类型>/<计费状态>`

### list 过滤与排序

- `--service-id <id>`：只看某预置服务下的接入点
- `--status <Active,Stopped>`：按状态过滤，逗号分隔
- `--name <关键字>`：按名称模糊匹配
- `--page <n>` / `--page-size <n>`（起始页 1，页大小上限 100）
- `--sort-by createTime|updateTime` + `--sort-order asc|desc`
- 输出每行：`endpointId  name  service=<serverId>  status=<状态>  rpm=.. tpm=.. tpmInput=.. tpmOutput=..`

### create（必填项多，注意顺序）

```bash
# 1) 先拿 serviceId
qianfan endpoint services --type chat --format json

# 2) 创建：--service-id/--name/--suffix 必填，限流四项至少填一个
qianfan endpoint create \
  --service-id <serviceId> --name my-chat --suffix mychat \
  --rpm 60 --tpm 10000 --format json
```

- 必填：`--service-id`、`--name`、`--suffix`
- 可选：`--description`
- 限流：`--rpm` / `--tpm` / `--tpm-input` / `--tpm-output`，**至少指定一个**，否则 `INVALID_ARGUMENT`
- 成功返回新接入点 ID（json：`{"endpointId": "..."}`）

### modify / stop

```bash
qianfan endpoint modify <endpointId> --name new-name --tpm 20000 --format json
qianfan endpoint stop <endpointId> --format json
```

- `modify` 至少指定 `--name` / `--description` / 任一限流项，否则 `INVALID_ARGUMENT`；未传的限流项保持原值
- `stop` 只停用、不删除

## Agent 快速执行顺序

1. 确认已登录：`qianfan auth status --format json`，`configured=false` 先引导用户 `qianfan auth login`
2. 要创建接入点：先 `endpoint services --format json` 选出目标 `serviceId`
3. 执行目标命令统一带 `--format json`，消费结构化输出
4. 报错按 `code` 分类处置（见下），把 `suggestion` 转达用户，不原样重试

## 错误处理

| 错误码 | 处置 |
| --- | --- |
| `INVALID_ARGUMENT` | 缺必填项或限流一个都没填、`modify` 没给要改的字段 → 按提示补参数 |
| `CREDENTIAL_NOT_FOUND` | 未登录或登录失效 → 引导 `qianfan auth login` |
| `AUTH_FAILED` | 签名/凭证失效（CLI 已自动强制续期 STS 重试一次仍失败）→ 重新登录或检查账号权限 |
| `CONTROL_PLANE_ERROR` | 管控 API 返回未归类错误，结合 `message` / `request_id` 排查 |
| `NETWORK_ERROR` | 网络不可达，检查网络或 `QIANFAN_CONTROL_PLANE_ENDPOINT` 后重试 |

## 与其他 skill 的串联

- 未登录 / 凭证问题：转 [qianfan-auth](../qianfan-auth/SKILL.md)
- 想发起推理而非管理接入点：转 [qianfan-chat](../qianfan-chat/SKILL.md) 等推理 skill
- 需要 modelId（绑定默认模型）：转 [qianfan-models](../qianfan-models/SKILL.md)
- 查调用量：转 [qianfan-service](../qianfan-service/SKILL.md)

## 安全与边界

- create / modify / stop 改动的是**服务端接入点资源**，不是本地状态，没有 `--dry-run`；执行前先向用户确认接入点与限流值。
- `stop` 是停用不是删除；需要彻底清理请到控制台处理。
- 只编排 CLI：不读取本地凭证、不自行签名或直连管控 HTTP 接口。

## 参考

- [qianfan-common/references/error-codes.md](../qianfan-common/references/error-codes.md) — 错误码、退出码与处置
