# Greptile 评论分诊

这是在 GitHub PR 上抓取、过滤和分类 Greptile review comment 的共享参考文档。`/review`（Step 2.5）和 `/ship`（Step 3.75）都会引用它。

## 抓取

先检测当前 PR，再抓取评论。两个 API 调用并行执行。

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner' 2>/dev/null)
PR_NUMBER=$(gh pr view --json number --jq '.number' 2>/dev/null)
```

如果任一命令失败或为空：静默跳过 Greptile 分诊。这个集成是增强项，不应阻塞主流程。

```bash
gh api repos/$REPO/pulls/$PR_NUMBER/comments \
  --jq '.[] | select(.user.login == "greptile-apps[bot]") | select(.position != null) | {id: .id, path: .path, line: .line, body: .body, html_url: .html_url, source: "line-level"}' > /tmp/greptile_line.json &
gh api repos/$REPO/issues/$PR_NUMBER/comments \
  --jq '.[] | select(.user.login == "greptile-apps[bot]") | {id: .id, body: .body, html_url: .html_url, source: "top-level"}' > /tmp/greptile_top.json &
wait
```

若 API 失败，或两个端点都没有 Greptile 评论，也要静默跳过。

`position != null` 用于过滤掉 force-push 后已经过时的行级评论。

## 抑制检查

先推导项目级历史文件路径：

```bash
REMOTE_SLUG=$(browse/bin/remote-slug 2>/dev/null || ~/.claude/skills/gstack/browse/bin/remote-slug 2>/dev/null || basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)")
PROJECT_HISTORY="$HOME/.gstack/projects/$REMOTE_SLUG/greptile-history.md"
```

如果 `$PROJECT_HISTORY` 存在，则读取它。文件每一行表示一次过往分诊结果：

```text
<date> | <repo> | <type:fp|fix|already-fixed> | <file-pattern> | <category>
```

类别固定为：`race-condition`、`null-check`、`error-handling`、`style`、`type-safety`、`security`、`performance`、`correctness`、`other`

匹配规则：

- 只对 `type == fp` 的历史记录做 suppress
- `repo` 必须匹配当前仓库
- `file-pattern` 必须匹配当前评论对应文件
- `category` 必须匹配该评论的问题类型

匹配成功的评论标记为 **SUPPRESSED** 并跳过。

如果历史文件不存在，或某几行无法解析，则忽略那些异常行并继续，不允许因此失败。

## 分类

对每条未被 suppress 的评论：

1. 行级评论：读取指定 `path:line` 及其上下文（±10 行）
2. 顶层评论：读取完整评论正文
3. 将评论与完整 diff（`git diff origin/main`）以及 review checklist 交叉比对
4. 归类为以下四种之一：

- **VALID & ACTIONABLE**：当前代码里真实存在的问题
- **VALID BUT ALREADY FIXED**：问题真实存在，但已经在当前分支后续提交中修复；需指出修复 commit
- **FALSE POSITIVE**：误报、误读、风格噪声，或问题已由别处安全处理
- **SUPPRESSED**：在前面的 suppressions check 已被过滤

## 回复 API

根据评论来源使用不同端点：

**行级评论**：

```bash
gh api repos/$REPO/pulls/$PR_NUMBER/comments/$COMMENT_ID/replies \
  -f body="<reply text>"
```

**顶层评论**：

```bash
gh api repos/$REPO/issues/$PR_NUMBER/comments \
  -f body="<reply text>"
```

如果回复失败，例如 PR 已关闭或没有写权限，应记录警告并继续，不能因此中断流程。

## 回复模板

### Tier 1：第一次回复

**修复完成：**

```text
**Fixed** in `<commit-sha>`.

```diff
- <旧代码>
+ <新代码>
```

**Why:** <一句话说明问题与修复逻辑>
```

**已经在之前提交修复：**

```text
**Already fixed** in `<commit-sha>`.

**What was done:** <1-2 句说明修复方式>
```

**误报：**

```text
**Not a bug.** <一句话说明为什么它不成立>

**Evidence:**
- <具体代码证据>
- <必要时补充框架或实现语义>

**Suggested re-rank:** 这更像是 `<style|noise|misread>`，而不是 `<Greptile 原分类>`。
```

### Tier 2：重复误报后的强硬回复

当同一线程里已经有过 GStack 回复，但 Greptile 仍重复标记时，使用更强证据链：

```text
**This has been reviewed and confirmed as [intentional/already-fixed/not-a-bug].**

```diff
<完整相关 diff>
```

**Evidence chain:**
1. <file:line 证据>
2. <修复 commit SHA，如适用>
3. <架构或设计理由，如适用>

**Suggested re-rank:** 请重新校准，这更接近 `<实际分类>`，而不是 `<Greptile 声称分类>`。
```

## 升级判定

在生成回复前，先判断同一线程里是否已有过 GStack 回复：

1. 行级评论：抓取 replies，检查回复正文中是否包含 `**Fixed**`、`**Not a bug.**`、`**Already fixed**`
2. 顶层评论：扫描该 PR issue comments，寻找在 Greptile 评论之后、且包含上述标记的回复
3. 如果已经有 GStack 回复，而 Greptile 又在同一文件 + 同一类别重复提示，则使用 Tier 2
4. 如果没有，则使用 Tier 1

若升级判定本身失败或有歧义，默认使用 Tier 1，不在模糊场景中升级语气。

## 严重程度重分级

分类时，也要判断 Greptile 暗示的严重级别是否合理：

- 如果 Greptile 把风格 / 性能小问题当成安全 / 正确性 / 竞态问题，应在回复中明确要求重分级
- 如果低严重度问题被写得像致命问题，也应指出
- 必须给出具体代码与行号作为依据，不能只表达主观看法

## 历史文件写入

写入前确保目录存在：

```bash
REMOTE_SLUG=$(browse/bin/remote-slug 2>/dev/null || ~/.claude/skills/gstack/browse/bin/remote-slug 2>/dev/null || basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)")
mkdir -p "$HOME/.gstack/projects/$REMOTE_SLUG"
mkdir -p ~/.gstack
```

对每条分诊结果，向两个文件各追加一行：

- `~/.gstack/projects/$REMOTE_SLUG/greptile-history.md`
- `~/.gstack/greptile-history.md`

格式：

```text
<YYYY-MM-DD> | <owner/repo> | <type> | <file-pattern> | <category>
```

示例：

```text
2026-03-13 | garrytan/myapp | fp | app/services/auth_service.rb | race-condition
2026-03-13 | garrytan/myapp | fix | app/models/user.rb | null-check
2026-03-13 | garrytan/myapp | already-fixed | lib/payments.rb | error-handling
```

## 输出格式

在最终输出头部附加 Greptile 摘要：

```text
+ N Greptile comments (X valid, Y fixed, Z FP)
```

对每条评论显示：

- 分类标签：`[VALID]`、`[FIXED]`、`[FALSE POSITIVE]`、`[SUPPRESSED]`
- `file:line`，若为顶层评论则写 `[top-level]`
- 一行摘要
- 对应 `html_url`
