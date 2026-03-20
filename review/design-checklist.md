# 设计评审清单（Lite）

> 这是 `DESIGN_METHODOLOGY` 的子集。若修改这里，也要同步更新 `scripts/gen-skill-docs.ts` 中的 `generateDesignMethodology()`。

## 说明

这份清单只针对 **diff 中涉及的前端源码**，不是页面实际渲染结果。请阅读每个变更过的前端文件全文，而不是只看 diff hunk，然后标记其中的反模式。

**触发条件：** 只有当 diff 涉及前端文件时才运行。可用 `gstack-diff-scope` 检测：

```bash
source <(~/.claude/skills/gstack/bin/gstack-diff-scope <base> 2>/dev/null)
```

如果 `SCOPE_FRONTEND=false`，则静默跳过设计评审。

**DESIGN.md 校准：** 如果仓库根目录存在 `DESIGN.md` 或 `design-system.md`，先读它，再按该设计系统来判断。文档中明确允许的模式不要报。若没有设计系统文档，则按通用设计原则判断。

## 置信度分层

- **[HIGH]**：可以通过 grep / 模式匹配可靠识别
- **[MEDIUM]**：可以通过启发式规则识别，可能有少量噪声
- **[LOW]**：需要理解视觉意图，只能作为“可能问题”提示

## 分类

**AUTO-FIX** 仅限高置信度、纯机械性 CSS 修复：

- `outline: none` 且没有替代焦点样式
- 新增 CSS 中使用 `!important`
- 正文文字 `font-size` 小于 16px

**ASK：**

- 其余所有需要设计判断的问题

**LOW 置信度项：**

- 一律写成 “Possible: ...，请视觉核验或运行 /design-review”
- 不允许自动修

## 输出格式

```text
Design Review: N issues (X auto-fixable, Y need input, Z possible)

**AUTO-FIXED:**
- [file:line] 问题 → 已应用修复

**NEEDS INPUT:**
- [file:line] 问题描述
  Recommended fix: 建议修复方式

**POSSIBLE (verify visually):**
- [file:line] 可能问题 — 请用 /design-review 验证
```

如果没有问题：`Design Review: No issues found.`

如果没有前端文件变更：静默跳过。

## 类别

### 1. AI Slop 检测

这些是典型的“像 AI 拼出来的 UI，而不是成熟设计师会交付的界面”。

- **[MEDIUM]** 紫色 / 靛色渐变背景或蓝紫配色
- **[LOW]** 典型三栏功能网格：彩色圆形图标 + 粗体标题 + 两行描述，重复 3 次
- **[LOW]** 把彩色圆形图标当节装饰
- **[HIGH]** 全部居中：标题、描述、卡片内容大量 `text-align: center`
- **[MEDIUM]** 所有元素统一套大圆角
- **[MEDIUM]** 通用 hero 文案，例如 “Welcome to ...” / “Unlock the power of ...”

### 2. 字体排印

- **[HIGH]** 正文字体小于 16px
- **[HIGH]** diff 中引入超过 3 种字体家族
- **[HIGH]** 标题层级跳跃，例如同文件 `h1` 后直接跟 `h3`
- **[HIGH]** 使用黑名单字体：Papyrus、Comic Sans、Lobster、Impact、Jokerman

### 3. 间距与布局

- **[MEDIUM]** 在 DESIGN.md 定义了间距体系时，出现不在 4px / 8px 标尺上的随意值
- **[MEDIUM]** 容器使用固定宽度但没有响应式兜底
- **[MEDIUM]** 文本容器缺失 `max-width`，导致行长过长
- **[HIGH]** 新 CSS 里出现 `!important`

### 4. 交互状态

- **[MEDIUM]** 按钮、链接、输入缺失 hover / focus 状态
- **[HIGH]** `outline: none` 或 `outline: 0` 且无替代焦点提示
- **[LOW]** 交互目标可能小于 44px

### 5. DESIGN.md 违例

仅当存在 `DESIGN.md` 或 `design-system.md` 时检查：

- **[MEDIUM]** 使用了设计系统外的颜色
- **[MEDIUM]** 使用了设计系统外的字体
- **[MEDIUM]** 使用了设计系统外的间距值

## 不要报告这些

- 在 DESIGN.md 中明确声明为有意为之的模式
- 第三方 / vendor CSS
- reset / normalize
- 测试夹具文件
- 生成或压缩后的 CSS
