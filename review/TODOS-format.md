# TODOS.md 格式参考

这是 `TODOS.md` 规范格式的共享参考。`/ship`（Step 5.5）和 `/plan-ceo-review`（TODOS 更新部分）都会引用它，以保证 TODO 项结构一致。

## 文件结构

```markdown
# TODOS

## <Skill/Component>     ← 例如：## Browse、## Ship、## Review、## Infrastructure
<按优先级排序，P0 在最前>

## Completed
<已完成事项，附完成版本与日期>
```

**分组原则：** 按技能或组件组织，例如 `## Browse`、`## Ship`、`## Review`、`## QA`、`## Retro`、`## Infrastructure`。每组内部按优先级排序，P0 在顶部。

## TODO 项格式

每个 TODO 项在所属分组下使用三级标题：

```markdown
### <标题>

**What:** 一句话描述要做什么。

**Why:** 解决什么问题，或释放什么价值。

**Context:** 让三个月后接手的人也能理解动机、当前状态和起点。

**Effort:** S / M / L / XL
**Priority:** P0 / P1 / P2 / P3 / P4
**Depends on:** <依赖项，或 "None">
```

**必填字段：** `What`、`Why`、`Context`、`Effort`、`Priority`  
**可选字段：** `Depends on`、`Blocked by`

## 优先级定义

- **P0**：阻塞项，下个版本前必须完成
- **P1**：关键项，本周期应完成
- **P2**：重要项，在 P0 / P1 清空后处理
- **P3**：可选增强，等待更多使用反馈后再评估
- **P4**：长期想法，不急

## 已完成项格式

当某项完成后，将其移动到 `## Completed`，保留原有内容，并在末尾追加：

```markdown
**Completed:** vX.Y.Z (YYYY-MM-DD)
```
