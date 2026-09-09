---
name: qianfan-connect
version: 1.0.0
description: "把 qianfan CLI 内嵌的 qianfan-* Agent 技能包安装到本机 Agent 技能目录（qianfan +connect），并支持枚举与卸载。当用户提到安装/更新千帆技能包、把千帆技能装到 Agent、qianfan +connect、列出或卸载已安装技能包，或希望本地 Agent 具备调用千帆 CLI 的能力时使用。反触发：用户想调用千帆能力本身（推理、用量、套餐）时转对应能力 skill，不必重新安装。"
license: Apache-2.0
metadata:
  requires:
    bins: ["qianfan"]
  cliHelp: "qianfan +connect --help"
  install:
    - kind: npm
      package: "@baiducloud/qianfan-cli"
---

# Qianfan 技能包安装

先读 [qianfan-common](../qianfan-common/SKILL.md) 的公共规则。`+connect` 只操作本地文件系统，
安装的技能包只编排 qianfan CLI，不读取凭证、不联网、不输出敏感信息。

## 何时使用本 skill

- 首次让本机 Agent 获得千帆能力
- CLI 升级后同步更新技能包
- 查看本机装了哪些 `qianfan-*` 技能包
- 清理不再需要的技能包

## 命令一览

| 命令 | 说明 |
| --- | --- |
| `qianfan +connect list [--target <dir>] [--format json]` | 枚举各目标目录已安装的技能包，只读、不联网、不需凭证 |
| `qianfan +connect [--target <dir>] [--dry-run] [--yes] [--format json]` | 安装/更新到所有检测到的 Agent（同名目录原子替换），受保护 |
| `qianfan +connect uninstall [--target <dir>] [--dry-run] [--yes] [--format json]` | 只卸载本 CLI 装的、且用户没改过的技能包，受保护 |
| `qianfan +connect uninstall --purge-prefix [--yes]` | 显式破坏性清理：删除目标目录下所有 `qianfan-*` 目录，含用户手工维护的同名目录 |

## Agent 快速执行顺序

1. `qianfan +connect list --format json` 看有哪些目标目录与现状
2. 需要安装/更新 → `qianfan +connect --dry-run --format json` 展示将 `create`/`replace`/`delete` 的目录
3. 用户确认后 → `qianfan +connect --yes`
4. 卸载同样先 `uninstall --dry-run --format json`，确认后 `--yes`；只有用户明确要求连手工目录一起清时才提 `--purge-prefix`

## 目标目录与安装语义

未显式 `--target` 时安装到**所有检测到的 Agent**：按 Agent 主目录是否存在判定（`~/.comate`、`~/.claude`、
`~/.codex`、`~/.cursor`、`~/.config/opencode`、`~/.agents`、`~/.openclaw`、`~/.gemini`、`~/.qwen`、`~/.iflow`、
`~/.trae`），落点按目录去重；Codex / Cursor / OpenCode 统一装进共享的 `~/.agents/skills`，避免同一技能包被扫描两遍。
一个 Agent 都没检测到时回退 `~/.qianfan/skills`。`--target <dir>` 是隔离操作：只作用于该目录，不扫描 Agent。

安装为幂等的原子替换：内嵌技能包先写入临时目录再 rename 替换同名目录，替换失败保留原目录，
不会产生半安装状态。每个技能目录下维护一份 `.qianfan-managed-skills.json` 所有权清单（技能名 + 内容摘要），
据此在安装时清理已下架的旧技能包（`pruned`）、在卸载时避免误删用户改过或手工维护的目录（`kept`）。
多目标下单个目录失败不影响其他目录：失败的目标在输出里单独标 `failed`，命令整体仍返回非零退出码。
JSON 输出形如 `{"targets": [{"agent": "comate", "dir": "...", "skills": ["qianfan-auth", "..."]}]}`；
更多细节见 [references/connect.md](references/connect.md)。

## 约束

- 安装/卸载属于本地变更：非交互环境未传 `--yes` 返回 `CONFIRMATION_REQUIRED`；
  `--dry-run` 与 `--yes` 不能同时使用（`INVALID_ARGUMENT`）；交互式拒绝确认返回 `OPERATION_CANCELLED`。
- `list` 是唯一无需确认的只读子命令。
- `--purge-prefix` 会删除用户手工维护的 `qianfan-*` 目录，只在用户明确要求全清时使用，并先 `--dry-run`。

## 与其他 skill 的串联

- 装完后要验证能力可用：转 [qianfan-doctor](../qianfan-doctor/SKILL.md)（`skills` 检查项）
- 尚未登录，装完也调不通：转 [qianfan-auth](../qianfan-auth/SKILL.md)

## 自然语言触发词 + 跨技能指引表

| 用户怎么说 | 走哪个命令/skill |
| --- | --- |
| "把千帆技能装到我的 Agent / 同步 skills" | `qianfan +connect`（先 dry-run） |
| "装了哪些千帆技能" | `qianfan +connect list --format json` |
| "卸载千帆技能包" | `qianfan +connect uninstall`（先 dry-run） |
| "我改过的技能包被留下了 / 想连改过的一起删" | `qianfan +connect uninstall --purge-prefix`（先 dry-run 并说明风险） |
| "只装到某个目录 / 装到项目里" | 加 `--target <dir>` |
| "装完还是不能用" | 转 `qianfan-doctor` / `qianfan-auth` |

## 安全与边界

- 只写技能目录：不改凭证、不改 `~/.qianfan` 下的敏感文件，不联网。
- 覆盖前必须确认：`replace` 会整体替换同名目录，先用 `--dry-run` 让用户看清受影响路径。
- 默认卸载保守：只删所有权清单记录且内容未改动的目录；`--purge-prefix` 才连手工目录一起删，属破坏性操作。
- 不擅自换目录：`--target` 只在用户明确要求时使用，默认对所有检测到的 Agent 生效。

## 参考

- `references/connect.md` — 目标目录检测、所有权清单、原子替换与卸载判定
- [qianfan-common/references/error-codes.md](../qianfan-common/references/error-codes.md) — 错误码、退出码与处置
