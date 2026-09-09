# qianfan +connect 行为细节

## 命令

```
qianfan +connect [--target <dir>] [--dry-run] [--yes] [--format text|json]   # 安装/更新
qianfan +connect list [--target <dir>] [--format text|json]                  # 枚举（只读）
qianfan +connect uninstall [--target <dir>] [--purge-prefix] [--dry-run] [--yes] [--format text|json]  # 卸载
```

## 目标目录检测

未显式 `--target` 时，安装到**所有检测到的 Agent** 技能目录。检测依据是 Agent 主目录是否存在（而不是
skills 子目录是否存在：刚装好的 Agent 往往还没有 skills 目录，要求它存在会漏装）：

- `~/.comate` → `~/.comate/skills`
- `~/.claude` → `~/.claude/skills`
- `~/.codex` / `~/.cursor` / `~/.config/opencode` / `~/.agents` → `~/.agents/skills`（共享目录）
- `~/.openclaw` → `~/.openclaw/skills`
- `~/.gemini` → `~/.gemini/skills`
- `~/.qwen` → `~/.qwen/skills`
- `~/.iflow` → `~/.iflow/skills`
- `~/.trae` → `~/.trae/skills`
- 一个都没检测到 → 回退 `~/.qianfan/skills`

Codex / Cursor / OpenCode 都会读共享的 `~/.agents/skills`，所以统一落到那里而不是各自的私有目录：
两处都装会让同一个技能包被扫描两遍，出现重复条目或同名冲突。多个 Agent 指向同一目录时按路径去重，
只安装一次，展示名取具体 Agent。`--target <dir>` 是隔离操作：只作用于该目录，不扫描 Agent。

## 多目标下的失败隔离

某个目标目录不可写或读不了时，其余目标照常处理：失败的目标在输出里单独标一行 `failed`（JSON 里是
`error` 字段），命令整体返回非零退出码。`--dry-run` 同样不会因为一个目录读不了而整体失败，
该目录在计划里记为 `skip unreadable skills directory`。

## 所有权清单

每个技能目录下维护 `.qianfan-managed-skills.json`：

```json
{ "version": 1, "skills": { "qianfan-chat": "<sha256 摘要>", "qianfan-auth": "<sha256 摘要>" } }
```

摘要按目录树内容计算（相对路径 + 文件内容）。它回答两个问题：哪些目录是本 CLI 装的，以及装完之后有没有被改过。
清单缺失或损坏时按「没有所有权记录」处理——宁可少删，也不误删用户的目录。

## 安装语义

- 内嵌技能包（全部 `qianfan-*`）先写入目标目录下的临时目录，再通过 rename **原子替换**同名目录。
- 幂等：重复安装等价于更新；同名目录被整体替换，旧内容不会残留。
- 替换失败时保留原目录，不会产生半安装状态。
- 清单记录过、但已从内嵌清单下架且内容未被改动的技能包会被顺带清理，输出里记为 `pruned`；改过的保留。

## 卸载语义

- 默认只删「清单记录 + 内容摘要未变」的目录。改过的保留并记为 `kept`（JSON 里是 `skipped`）。
- 没有清单的旧版本安装：目录内容与当前内嵌技能包逐字节一致时同样认作自己装的，可以正常卸载。
- 既不在清单、也不与内嵌内容一致的 `qianfan-*` 目录默认不认领、不删除。
- 摘要算不出来（权限不足等）按「对不上」处理：只保留这一个目录，不影响同目录其他技能包的清理。
- 软链形式的 `qianfan-*` 条目会被列出并可被清理，但只摘掉链接本身，不动它指向的真实目录。
- `--purge-prefix` 是显式破坏性清理：删除目标目录下所有 `qianfan-*` 目录，含用户手工维护的同名目录，并移除清单。

## 预演与确认

- `--dry-run --format json` 输出执行计划：安装列 `create`（新增）/ `replace`（更新）/ `delete`（清理已下架），
  卸载列 `delete`，不写入任何文件。
- 确认后加 `--yes` 执行。
- `--dry-run` 与 `--yes` 互斥（`INVALID_ARGUMENT`）；非交互环境未传 `--yes` 返回 `CONFIRMATION_REQUIRED`。

## 输出示例（--format json）

```json
{
  "targets": [
    { "agent": "comate", "dir": "/home/u/.comate/skills", "skills": ["qianfan-auth", "qianfan-chat", "..."] },
    { "agent": "claude-code", "dir": "/home/u/.claude/skills", "skills": ["qianfan-auth", "qianfan-chat", "..."] }
  ]
}
```

`skills` 是本次安装/删除/枚举到的技能包；`pruned` 为清理掉的已下架技能包，`skipped` 为被保留的本地改动，
`error` 为该目标的失败原因，为空时省略。文本输出按目标分段，逐行给出 `installed` / `removed` / `pruned` /
`kept ... (locally modified)` / `failed: ...`。

## 安全约束

- `+connect list` 不联网、不需要凭证。
- 安装的技能包只编排 `qianfan` CLI，不读取本地凭证、不直接构造千帆 HTTP 请求、不输出敏感信息。
