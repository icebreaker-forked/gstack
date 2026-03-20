---
name: freeze
description: |
  在当前会话中将文件编辑限制在指定目录内。所有超出允许路径的 Edit 和
  Write 都会被阻止。适合在排障时防止误改无关代码，或在你希望把修改范围
  收敛到单个模块时使用。当用户说 “freeze”、“restrict edits”、
  “only edit this folder” 或 “lock down edits” 时使用。
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
> **Safety Advisory:** This skill includes safety checks that verify file edits are within the allowed scope boundary before applying, and verify file writes are within the allowed scope boundary before applying. When using this skill, always pause and verify before executing potentially destructive operations. If uncertain about a command's safety, ask the user for confirmation before proceeding.


# /freeze — 将编辑限制在单个目录

将文件编辑锁定在特定目录内。任何指向允许路径之外文件的 Edit 或 Write 操作都会被**直接阻止**，而不是只给警告。

```bash
mkdir -p ~/.gstack/analytics
echo '{"skill":"freeze","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
```

## 设置

通过 AskUserQuestion 询问用户要把编辑限制在哪个目录：

- Question: “要把编辑限制在哪个目录？该路径之外的文件将无法被修改。”
- 使用文本输入，不提供多选。

用户给出目录后：

1. 解析为绝对路径：
```bash
FREEZE_DIR=$(cd "<user-provided-path>" 2>/dev/null && pwd)
echo "$FREEZE_DIR"
```

2. 确保路径带尾随 `/`，并写入 freeze 状态文件：
```bash
FREEZE_DIR="${FREEZE_DIR%/}/"
STATE_DIR="${CLAUDE_PLUGIN_DATA:-$HOME/.gstack}"
mkdir -p "$STATE_DIR"
echo "$FREEZE_DIR" > "$STATE_DIR/freeze-dir.txt"
echo "Freeze boundary set: $FREEZE_DIR"
```

告诉用户：“编辑现已限制在 `<path>/`。该目录之外的任何 Edit 或 Write 都会被阻止。若要修改边界，重新运行 `/freeze`；若要解除限制，运行 `/unfreeze` 或结束当前会话。”

## 工作方式

Hook 会从 Edit / Write 的输入 JSON 中读取 `file_path`，检查该路径是否以 freeze 目录开头。若不是，则返回 `permissionDecision: "deny"` 以阻止操作。

freeze 边界会通过状态文件在当前 session 内持续生效。每次 Edit / Write 调用时，hook 脚本都会重新读取该状态。

## 说明

- freeze 目录末尾的 `/` 可以防止 `/src` 错误匹配到 `/src-old`
- freeze 只作用于 Edit 和 Write；Read、Bash、Glob、Grep 不受影响
- 这是一层“防误改”护栏，不是安全边界；例如 `sed` 这类 Bash 命令仍可修改边界外文件
- 若要停用，运行 `/unfreeze` 或结束当前会话
