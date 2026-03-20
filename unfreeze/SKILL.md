---
name: unfreeze
version: 0.1.0
description: |
  清除由 /freeze 设置的编辑边界，使所有目录重新允许编辑。
  适用于不结束当前会话、只想扩大编辑范围的场景。当用户说
  “unfreeze”、“unlock edits”、“remove freeze” 或
  “allow all edits” 时使用。
allowed-tools:
  - Bash
  - Read
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

# /unfreeze — 清除 Freeze 边界

移除由 `/freeze` 设置的编辑限制，使所有目录重新可编辑。

```bash
mkdir -p ~/.gstack/analytics
echo '{"skill":"unfreeze","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
```

## 清除边界

```bash
STATE_DIR="${CLAUDE_PLUGIN_DATA:-$HOME/.gstack}"
if [ -f "$STATE_DIR/freeze-dir.txt" ]; then
  PREV=$(cat "$STATE_DIR/freeze-dir.txt")
  rm -f "$STATE_DIR/freeze-dir.txt"
  echo "Freeze 边界已清除（之前为：$PREV）。现在所有位置都允许编辑。"
else
  echo "当前没有设置 freeze 边界。"
fi
```

把结果告知用户。注意：`/freeze` 的 hook 在当前会话中仍然挂着，只是因为状态文件不存在，所以会全部放行。若要重新启用限制，再运行一次 `/freeze` 即可。
