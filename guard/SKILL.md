---
name: guard
version: 0.1.0
description: |
  完整安全模式：破坏性命令警告 + 目录级编辑限制。
  它将 /careful（在 rm -rf、DROP TABLE、force-push 等操作前警告）
  与 /freeze（阻止修改指定目录之外的文件）合并为一个命令。
  适用于生产环境操作或在线系统排障。当用户说 “guard mode”、
  “full safety”、“lock it down” 或 “maximum safety” 时使用。
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "bash ${CLAUDE_SKILL_DIR}/../careful/bin/check-careful.sh"
          statusMessage: "正在检查是否存在破坏性命令..."
    - matcher: "Edit"
      hooks:
        - type: command
          command: "bash ${CLAUDE_SKILL_DIR}/../freeze/bin/check-freeze.sh"
          statusMessage: "正在检查 freeze 边界..."
    - matcher: "Write"
      hooks:
        - type: command
          command: "bash ${CLAUDE_SKILL_DIR}/../freeze/bin/check-freeze.sh"
          statusMessage: "正在检查 freeze 边界..."
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

# /guard — 完整安全模式

同时启用破坏性命令警告和目录级编辑限制，相当于把 `/careful` 与 `/freeze` 合并为一个命令。

**依赖说明：** 这个技能会引用相邻 `/careful` 和 `/freeze` 目录中的 hook 脚本，因此两者都必须存在。正常情况下，gstack 的安装脚本会一并安装它们。

```bash
mkdir -p ~/.gstack/analytics
echo '{"skill":"guard","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
```

## 设置

通过 AskUserQuestion 询问用户要把编辑限制在哪个目录：

- Question: “Guard mode：要把编辑限制在哪个目录？破坏性命令警告始终开启；选定路径之外的文件会被阻止修改。”
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

告诉用户：
- “**Guard mode 已启用。** 当前有两层保护同时生效：”
- “1. **破坏性命令警告**：rm -rf、DROP TABLE、force-push 等操作执行前会先警告（可覆盖）”
- “2. **编辑边界**：文件编辑被限制在 `<path>/`，超出该目录的编辑会被阻止”
- “如需移除编辑边界，运行 `/unfreeze`；如需完全停用，结束当前会话”

## 保护范围

破坏性命令模式与安全例外的完整列表见 `/careful`。编辑边界的工作方式见 `/freeze`。
