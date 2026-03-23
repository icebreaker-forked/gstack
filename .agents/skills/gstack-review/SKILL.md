---
name: review
description: |
  合并前 PR 评审。它会针对与基线分支的 diff，检查 SQL 安全、LLM 信任边界、
  条件副作用及其他结构性问题。当用户说 “review this PR”、“code review”、
  “pre-landing review” 或 “check my diff” 时使用。
  当用户准备合并或落地代码改动时，应主动建议。
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

## 前言（先执行）

```bash
_UPD=$(~/.codex/skills/gstack/bin/gstack-update-check 2>/dev/null || .agents/skills/gstack/bin/gstack-update-check 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD" || true
mkdir -p ~/.gstack/sessions
touch ~/.gstack/sessions/"$PPID"
_SESSIONS=$(find ~/.gstack/sessions -mmin -120 -type f 2>/dev/null | wc -l | tr -d ' ')
find ~/.gstack/sessions -mmin +120 -type f -delete 2>/dev/null || true
_CONTRIB=$(~/.codex/skills/gstack/bin/gstack-config get gstack_contributor 2>/dev/null || true)
_PROACTIVE=$(~/.codex/skills/gstack/bin/gstack-config get proactive 2>/dev/null || echo "true")
_BRANCH=$(git branch --show-current 2>/dev/null || echo "unknown")
echo "BRANCH: $_BRANCH"
echo "PROACTIVE: $_PROACTIVE"
_LAKE_SEEN=$([ -f ~/.gstack/.completeness-intro-seen ] && echo "yes" || echo "no")
echo "LAKE_INTRO: $_LAKE_SEEN"
_TEL=$(~/.codex/skills/gstack/bin/gstack-config get telemetry 2>/dev/null || true)
_TEL_PROMPTED=$([ -f ~/.gstack/.telemetry-prompted ] && echo "yes" || echo "no")
_TEL_START=$(date +%s)
_SESSION_ID="$$-$(date +%s)"
echo "TELEMETRY: ${_TEL:-off}"
echo "TEL_PROMPTED: $_TEL_PROMPTED"
mkdir -p ~/.gstack/analytics
echo '{"skill":"review","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
for _PF in ~/.gstack/analytics/.pending-*; do [ -f "$_PF" ] && ~/.codex/skills/gstack/bin/gstack-telemetry-log --event-type skill_run --skill _pending_finalize --outcome unknown --session-id "$_SESSION_ID" 2>/dev/null || true; break; done
```

如果 `PROACTIVE` 为 `"false"`，不要主动推荐 gstack 技能，只在用户明确要求时调用。用户已经关闭了主动建议。

如果输出为 `UPGRADE_AVAILABLE <old> <new>`：读取 `~/.codex/skills/gstack/gstack-upgrade/SKILL.md` 并按“Inline upgrade flow”执行（若已配置自动升级则直接升级，否则通过 AskUserQuestion 给出 4 个选项；若用户拒绝则写入 snooze 状态）。如果输出为 `JUST_UPGRADED <from> <to>`：告诉用户 “Running gstack v{to} (just updated!)”，然后继续。

如果 `LAKE_INTRO` 为 `no`：继续之前，先介绍 Completeness Principle。
告诉用户：“gstack 遵循 **Boil the Lake** 原则：当 AI 让边际成本接近于零时，就应优先把事情做完整。详见：https://garryslist.org/posts/boil-the-ocean”
然后询问是否要在默认浏览器中打开这篇文章：

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

只有用户明确同意时才运行 `open`。无论如何都要运行 `touch` 将其标记为已见。这个流程只发生一次。

如果 `TEL_PROMPTED` 为 `no` 且 `LAKE_INTRO` 为 `yes`：在完成 lake intro 之后，向用户询问 telemetry 设置。使用 AskUserQuestion：

> 帮 gstack 变得更好！Community 模式会通过稳定设备 ID 分享使用数据（你用了哪些技能、耗时多久、是否崩溃），便于我们追踪趋势并更快修 bug。
> 不会发送任何代码、文件路径或仓库名称。
> 你可以随时通过 `gstack-config set telemetry off` 关闭。

Options:
- A) 帮 gstack 变得更好！（推荐）
- B) 不，谢谢

If A: run `~/.codex/skills/gstack/bin/gstack-config set telemetry community`

If B: ask a follow-up AskUserQuestion:

> 那匿名模式呢？我们只知道“有人”用了 gstack，没有唯一 ID，也无法串联会话，只是一个帮助我们了解使用量的计数。

Options:
- A) 可以，匿名模式没问题
- B) 不，谢谢，完全关闭

If B→A: run `~/.codex/skills/gstack/bin/gstack-config set telemetry anonymous`
If B→B: run `~/.codex/skills/gstack/bin/gstack-config set telemetry off`

Always run:
```bash
touch ~/.gstack/.telemetry-prompted
```

这个流程只发生一次。如果 `TEL_PROMPTED` 已经是 `yes`，则完全跳过。

## AskUserQuestion 格式

**每次调用 AskUserQuestion 都必须遵循以下结构：**
1. **重新锚定上下文：** 说明当前项目、当前分支（使用前言中输出的 `_BRANCH`，不要使用对话历史或 gitStatus 中的其他分支名）以及当前任务。（1-2 句）
2. **通俗解释：** 用一个聪明的 16 岁用户也能看懂的语言解释问题。不要直接抛函数名、内部术语或实现细节。多用具体例子和类比。强调“它会做什么”，而不是“它叫什么”。
3. **给出推荐：** 使用 `RECOMMENDATION: Choose [X] because [one-line reason]`。始终优先推荐完整方案而不是捷径（见 Completeness Principle）。每个选项都要带 `Completeness: X/10`。标尺：10 = 完整实现（边界条件和覆盖都齐全），7 = happy path 足够但跳过部分边缘情况，3 = 明显延期的重要工作。如果两个选项都在 8 分以上，选更高的；如果某个选项 ≤5，要明确指出。
4. **列出选项：** 使用字母选项 `A) ... B) ... C) ...`。只要涉及工作量，就同时给人工团队与 CC 的估时：`(human: ~X / CC: ~Y)`

默认假设用户已经有 20 分钟没看这个窗口，也没有把代码打开。如果你的解释必须依赖“先去读源码才能看懂”，那说明写得还不够清楚。

各技能可以在此基础上再叠加自己的格式要求。

## Completeness Principle — Boil the Lake

AI 辅助编码让“把事情做完整”的边际成本接近于零。因此在给用户呈现选项时：

- 如果 A 是完整实现（完整对齐、覆盖全部边界情况、测试齐全），B 只是为了省一点点工作量的捷径，**永远推荐 A**。在 CC+gstack 场景下，80 行和 150 行代码的差距几乎没有意义。“差不多够用”不是正确本能。
- **湖与海：** “湖”是可以被煮沸的，例如某个模块 100% 测试覆盖、一个功能的完整实现、边界情况和错误路径补齐；“海”则不是，例如重写整个系统、给不可控依赖新增能力、跨多个季度的平台迁移。推荐把湖做完，把海明确标成超出范围。
- **估算工作量时**，一定同时给出人工团队时间和 CC+gstack 时间。可参考下表：

| 任务类型 | 人工团队 | CC+gstack | 压缩倍数 |
|----------|----------|------------|----------|
| 脚手架 / 样板代码 | 2 天 | 15 分钟 | ~100x |
| 编写测试 | 1 天 | 15 分钟 | ~50x |
| 功能实现 | 1 周 | 30 分钟 | ~30x |
| 修 bug + 回归测试 | 4 小时 | 15 分钟 | ~20x |
| 架构 / 设计 | 2 天 | 4 小时 | ~5x |
| 调研 / 探索 | 1 天 | 3 小时 | ~3x |

- 这个原则适用于测试覆盖、错误处理、文档、边界情况和功能完整性。不要为了“省时间”而故意跳过最后 10%；在 AI 协助下，那 10% 往往只多花几分钟。

**反模式：不要这样做**
- BAD: “选 B 吧，它用更少代码就覆盖了 90% 的价值。”（如果 A 只多几十行，就该选 A）
- BAD: “先不处理边界情况，节省时间。”（在 CC 场景下，补边界情况通常只要几分钟）
- BAD: “测试覆盖等后续 PR 再补。”（测试通常是最值得顺手做完的“湖”）
- BAD: 只报人工团队工期：“这个大概要 2 周。”（应写成 “人工约 2 周 / CC 约 1 小时”）

## Contributor Mode

如果 `_CONTRIB` 为 `true`：表示你当前处于 **contributor mode**。你既是 gstack 的使用者，也是在帮助它变得更好。

**在每个主要工作流阶段结束时**（不是每个命令之后），都回顾一下你刚刚使用的 gstack 工具体验，并给出 0 到 10 分。如果不是 10 分，想想原因。如果这里存在一个明显、可操作的 bug，或者 gstack 的代码 / 技能 markdown 本可以做得更好，而且这个点有启发意义，就提交一份 field report。

**校准标准：** 例如，过去 `$B js "await fetch(...)"` 会因为 gstack 没有自动把表达式包进 async 上下文，而报 `SyntaxError: await is only valid in async functions`。这类问题虽然小，但用户输入是合理的，gstack 本应处理好，这就值得记录。比这更轻微、影响更小的事情就不要报。

**不值得记录的内容：** 用户自己应用的 bug、用户站点的网络错误、用户站点的登录失败、用户自身的 JS 逻辑错误。

**提交方式：** 向 `~/.gstack/contributor-logs/{slug}.md` 写入**下列全部章节**（不要截断，必须写到最后的 Date / Version footer）：

```
# {Title}

Hey gstack team — 我在使用 /{skill-name} 时遇到了这个问题：

**What I was trying to do:** {用户 / 代理原本想做什么}
**What happened instead:** {实际发生了什么}
**My rating:** {0-10} — {一句话说明为什么它不是 10 分}

## Steps to reproduce
1. {step}

## Raw output
```
{paste the actual error or unexpected output here}
```

## What would make this a 10
{一句话：gstack 本应怎样做才算更好}

**Date:** {YYYY-MM-DD} | **Version:** {gstack version} | **Skill:** /{skill}
```

Slug 规则：小写、用连字符、最长 60 个字符（例如 `browse-js-no-await`）。如果文件已存在则跳过。每个 session 最多写 3 份报告。就地写入并继续流程，不要因此中断。随后告诉用户：“Filed gstack field report: {title}”

## Completion Status Protocol

当一个技能工作流结束时，必须使用以下状态之一汇报结果：
- **DONE** — 所有步骤均成功完成，且每个关键结论都有证据支撑。
- **DONE_WITH_CONCERNS** — 已完成，但存在用户需要知道的风险或问题。逐条列出。
- **BLOCKED** — 无法继续。明确说明阻塞点以及你已经尝试过什么。
- **NEEDS_CONTEXT** — 缺少继续所需的信息。明确指出具体缺什么。

### 升级 / 求助

任何时候，如果你判断“这件事超出我当前把握”或“我对结果没有信心”，都可以停下来明确升级。

做出糟糕结果，比承认做不到更糟。
- 如果同一任务已经尝试 3 次仍然失败，立即停止并升级。
- 如果涉及安全敏感改动且你不能确认正确性，立即停止并升级。
- 如果工作范围已经超出你能验证的边界，立即停止并升级。

升级格式：
```
STATUS: BLOCKED | NEEDS_CONTEXT
REASON: [1-2 句原因]
ATTEMPTED: [你已经尝试过什么]
RECOMMENDATION: [建议用户下一步怎么做]
```

## Telemetry（最后执行）

在技能工作流结束后（无论成功、失败还是中止），都要记录 telemetry 事件。
技能名称取自本文件 YAML frontmatter 中的 `name:` 字段。
结果状态取自工作流最终结果：正常完成为 success，失败为 error，用户中断为 abort。

**PLAN MODE 例外：这一段也必须执行。** 该命令写入的是
`~/.gstack/analytics/`（用户配置目录，而不是项目文件）。技能前言本来也在写同一目录，因此这属于一致模式。跳过它会丢失 session 时长和结果信息。

执行以下 bash：

```bash
_TEL_END=$(date +%s)
_TEL_DUR=$(( _TEL_END - _TEL_START ))
rm -f ~/.gstack/analytics/.pending-"$_SESSION_ID" 2>/dev/null || true
~/.codex/skills/gstack/bin/gstack-telemetry-log \
  --skill "SKILL_NAME" --duration "$_TEL_DUR" --outcome "OUTCOME" \
  --used-browse "USED_BROWSE" --session-id "$_SESSION_ID" 2>/dev/null &
```

将 `SKILL_NAME` 替换为 frontmatter 中的真实技能名，将 `OUTCOME` 替换为 success / error / abort，并根据是否用过 `$B` 将 `USED_BROWSE` 填为 true / false。
如果无法判断结果状态，则使用 "unknown"。这条命令在后台执行，不应阻塞用户。

## Step 0：检测基线分支

先确定当前 PR 的目标分支。后续所有步骤里提到“基线分支”时，都使用这里检测出的结果。

1. 先检查当前分支是否已经有对应 PR：
   `gh pr view --json baseRefName -q .baseRefName`
   如果成功，就使用输出的分支名作为基线分支。

2. 如果还没有 PR（命令失败），则检测仓库默认分支：
   `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`

3. 如果两个命令都失败，则回退到 `main`。

打印检测出的基线分支名。后续所有 `git diff`、`git log`、`git fetch`、`git merge` 和 `gh pr create` 命令里，只要说明文字写到“基线分支”，都替换成这里检测出的分支名。

---

# Pre-Landing PR Review

你正在执行 `/review` 工作流。请对当前分支相对于基线分支的 diff 做分析，重点寻找那些测试未必能捕捉到的结构性问题。

---

## Step 1：检查分支

1. 运行 `git branch --show-current` 获取当前分支名。
2. 如果当前就在基线分支上，输出：**"Nothing to review — you're on the base branch or have no changes against it."** 然后停止。
3. 运行 `git fetch origin <base> --quiet && git diff origin/<base> --stat` 检查是否存在 diff。如果没有 diff，也输出同样的话并停止。

---

## Step 1.5：范围漂移检测

在审代码质量之前，先检查：**这次实现做的是否正是被要求做的事，不多也不少？**

1. 读取 `TODOS.md`（如果存在）、PR 描述（`gh pr view --json body --jq .body 2>/dev/null || true`）以及 commit message（`git log origin/<base>..HEAD --oneline`）。
   **如果还没有 PR：** 就依赖 commit message 和 `TODOS.md` 判断此次改动的显式意图。这是常见情况，因为 `/review` 通常在 `/ship` 创建 PR 之前运行。
2. 识别这次分支的**显式目标**是什么。
3. 运行 `git diff origin/<base> --stat`，把变更文件与显式目标逐一比对。
4. 带着怀疑去判断：

   **SCOPE CREEP 检测：**
   - 改动了与既定目标无关的文件
   - 引入了计划中未提到的新功能或重构
   - 典型的“顺手改了点别的”式扩张，扩大了 blast radius

   **MISSING REQUIREMENTS 检测：**
   - `TODOS.md` / PR 描述里提到的要求，在 diff 中没有体现
   - 已声明需求对应的测试覆盖缺口
   - 做了一半但没做完的实现

5. 在正式评审开始前，先输出：
   ```
   Scope Check: [CLEAN / DRIFT DETECTED / REQUIREMENTS MISSING]
   Intent: <1-line summary of what was requested>
   Delivered: <1-line summary of what the diff actually does>
   [If drift: list each out-of-scope change]
   [If missing: list each unaddressed requirement]
   ```

6. 这一节属于 **INFORMATIONAL**，不阻塞后续评审。之后继续 Step 2。

---

## Step 2：读取 checklist

读取 `.agents/skills/gstack/review/checklist.md`。

**如果读不到这个文件，就停止并报告错误。** 没有 checklist 就不要继续。

---

## Step 2.5：检查 Greptile review comments

读取 `.agents/skills/gstack/review/greptile-triage.md`，按其中的抓取、过滤、分类以及 **escalation detection** 步骤执行。

**如果没有 PR、`gh` 失败、API 返回错误，或 Greptile 评论数为 0：** 静默跳过。Greptile 是增强项，不应阻塞 review。

**如果找到了 Greptile 评论：** 保存它们的分类结果（VALID & ACTIONABLE、VALID BUT ALREADY FIXED、FALSE POSITIVE、SUPPRESSED），Step 5 会用到。

---

## Step 3：获取 diff

先拉取最新基线分支，避免因为本地状态陈旧产生误报：

```bash
git fetch origin <base> --quiet
```

运行 `git diff origin/<base>` 获取完整 diff。这里应包含当前分支相对于最新基线分支的所有已提交和未提交改动。

---

## Step 4：两轮评审

按 checklist 分两轮检查 diff：

1. **Pass 1 (CRITICAL):** SQL & Data Safety, Race Conditions & Concurrency, LLM Output Trust Boundary, Enum & Value Completeness
2. **Pass 2 (INFORMATIONAL):** Conditional Side Effects, Magic Numbers & String Coupling, Dead Code & Consistency, LLM Prompt Issues, Test Gaps, View/Frontend

**Enum & Value Completeness 必须读 diff 之外的代码。** 一旦 diff 引入新的枚举值、状态值、tier 或 type 常量，就要用 Grep 找到所有引用其同类值的文件，再逐个 Read 检查新值是否被处理到位。这是少数仅靠 diff 本身不够的类别。

输出格式严格遵守 checklist。尊重 suppressions，凡是在 “DO NOT flag” 中列出的项，一律不要报。

---

## Step 4.5：设计评审（按需）

## Design Review (conditional, diff-scoped)

Check if the diff touches frontend files using `gstack-diff-scope`:

```bash
source <(~/.codex/skills/gstack/bin/gstack-diff-scope <base> 2>/dev/null)
```

**If `SCOPE_FRONTEND=false`:** Skip design review silently. No output.

**If `SCOPE_FRONTEND=true`:**

1. **Check for DESIGN.md.** If `DESIGN.md` or `design-system.md` exists in the repo root, read it. All design findings are calibrated against it — patterns blessed in DESIGN.md are not flagged. If not found, use universal design principles.

2. **Read `.agents/skills/gstack/review/design-checklist.md`.** If the file cannot be read, skip design review with a note: "Design checklist not found — skipping design review."

3. **Read each changed frontend file** (full file, not just diff hunks). Frontend files are identified by the patterns listed in the checklist.

4. **Apply the design checklist** against the changed files. For each item:
   - **[HIGH] mechanical CSS fix** (`outline: none`, `!important`, `font-size < 16px`): classify as AUTO-FIX
   - **[HIGH/MEDIUM] design judgment needed**: classify as ASK
   - **[LOW] intent-based detection**: present as "Possible — verify visually or run /design-review"

5. **Include findings** in the review output under a "Design Review" header, following the output format in the checklist. Design findings merge with code review findings into the same Fix-First flow.

6. **Log the result** for the Review Readiness Dashboard:

```bash
~/.codex/skills/gstack/bin/gstack-review-log '{"skill":"design-review-lite","timestamp":"TIMESTAMP","status":"STATUS","findings":N,"auto_fixed":M,"commit":"COMMIT"}'
```

Substitute: TIMESTAMP = ISO 8601 datetime, STATUS = "clean" if 0 findings or "issues_found", N = total findings, M = auto-fixed count, COMMIT = output of `git rev-parse --short HEAD`.

把设计类发现和 Step 4 的发现一起输出。它们同样遵循 Step 5 的 Fix-First 流程：纯机械 CSS 修复走 AUTO-FIX，其余走 ASK。

---

## Step 5：Fix-First Review

**每一个发现都必须有动作，不只是 critical。**

先输出一个汇总头：`Pre-Landing Review: N issues (X critical, Y informational)`

### Step 5a：给每个发现分类

每个发现都要按照 checklist 中的 Fix-First Heuristic 分到 AUTO-FIX 或 ASK。Critical 更偏向 ASK；informational 更偏向 AUTO-FIX。

### Step 5b：自动修掉所有 AUTO-FIX 项

对每个 AUTO-FIX 项直接实施修复，并逐条输出一行总结：
`[AUTO-FIXED] [file:line] Problem → what you did`

### Step 5c：把 ASK 项集中提问

如果还有 ASK 项，把它们合并成 **一个** AskUserQuestion：

- 每个问题都写清楚编号、严重级别、问题本身，以及推荐修法
- 每个问题都提供选项：A) 按推荐修复，B) 跳过
- 最后给出一个总体 RECOMMENDATION

Example format:
```
I auto-fixed 5 issues. 2 need your input:

1. [CRITICAL] app/models/post.rb:42 — Race condition in status transition
   Fix: Add `WHERE status = 'draft'` to the UPDATE
   → A) Fix  B) Skip

2. [INFORMATIONAL] app/services/generator.rb:88 — LLM output not type-checked before DB write
   Fix: Add JSON schema validation
   → A) Fix  B) Skip

RECOMMENDATION: Fix both — #1 is a real race condition, #2 prevents silent data corruption.
```

如果 ASK 项不超过 3 个，也可以拆成单独 AskUserQuestion。

### Step 5d：应用用户批准的修复

对于用户选择 “Fix” 的项，直接实施修复，并输出实际修了什么。

如果不存在 ASK 项（即全部都是 AUTO-FIX），则完全跳过提问。

### 结论验证

在给出最终 review 结论前：
- 如果你说“这种写法是安全的” → 必须引用能证明其安全的具体行
- 如果你说“这在别处已经处理了” → 必须读到那段处理逻辑并引用它
- 如果你说“测试覆盖到了” → 必须写出测试文件与测试方法
- 不允许说 “likely handled” 或 “probably tested”，要么验证，要么明确标成未知

**禁止自我合理化：** “This looks fine” 不算结论。你要么给出它确实没问题的证据，要么把它标成未验证。

### Greptile 评论处置

在输出完你自己的发现之后，如果 Step 2.5 中存在已分类的 Greptile 评论：

**在输出头部追加 Greptile 摘要：** `+ N Greptile comments (X valid, Y fixed, Z FP)`

在回复任何评论前，先执行 `greptile-triage.md` 中的 **Escalation Detection** 算法，判断应该用 Tier 1（友好）还是 Tier 2（更强硬）的回复模板。

1. **VALID & ACTIONABLE comments：** 这些评论应被纳入你的正式发现中，并继续走 Fix-First 流程（机械性问题自动修，其余合并进 ASK）。给用户的选项是：A) 立即修复，B) 记录后照常 ship，C) 认定为误报。若用户选 A，就使用 `greptile-triage.md` 中的 **Fix reply template** 回复（包含 inline diff 与解释）；若用户选 C，就使用 **False Positive reply template** 回复（附证据与建议重分级），并同时写入项目级与全局 greptile-history。

2. **FALSE POSITIVE comments：** 每条都要通过 AskUserQuestion 单独呈现：
   - 展示 Greptile 评论：`file:line`（或 `[top-level]`）+ 摘要 + permalink URL
   - 简洁解释为什么你认为它是误报
   - 选项：
     - A) 回复 Greptile，解释为什么这是误报（如果判断非常明确，优先推荐）
     - B) 即便如此也顺手修掉（如果代价很低且无害）
     - C) 忽略 —— 不回复，也不修

   If the user chooses A, reply using the **False Positive reply template** from greptile-triage.md (include evidence + suggested re-rank), save to both per-project and global greptile-history.

3. **VALID BUT ALREADY FIXED comments：** 直接使用 `greptile-triage.md` 中的 **Already Fixed reply template** 回复，不需要 AskUserQuestion：
   - 说明已经做了什么，以及对应修复 commit SHA
   - 同时写入项目级与全局 greptile-history

4. **SUPPRESSED comments：** 静默跳过；这些是之前已确认过的已知误报。

---

## Step 5.5：交叉检查 TODOS

读取仓库根目录的 `TODOS.md`（如果存在），并把当前 PR 与未完成 TODO 做交叉检查：

- **这个 PR 是否关闭了现有 TODO？** 如果是，在输出中写清楚：“This PR addresses TODO: <title>”
- **这个 PR 是否衍生出应该写进 TODO 的后续工作？** 如果是，把它作为 informational finding 标出来。
- **是否存在与这次 review 高度相关的 TODO，可作为上下文？** 如果有，在讨论相关发现时引用。

如果没有 `TODOS.md`，则静默跳过。

---

## Step 5.6：文档陈旧性检查

把 diff 与文档文件交叉比对。对仓库根目录中的每个 `.md` 文件（如 `README.md`、`ARCHITECTURE.md`、`CONTRIBUTING.md`、`CLAUDE.md` 等）：

1. 检查 diff 中的代码改动是否影响了该文档描述的功能、组件或工作流。
2. 如果文档本身在当前分支**没有更新**，但它描述的代码**发生了变化**，就把它报成 INFORMATIONAL finding：
   `"Documentation may be stale: [file] describes [feature/component] but code changed in this branch. Consider running /document-release."`

这一类问题永远只算 informational，不算 critical。对应修复动作是 `/document-release`。

如果没有相关文档文件，就静默跳过。

---

## Step 5.7：Codex 第二意见（可选）

完成 review 后，检查本机是否可用 Codex CLI：

```bash
which codex 2>/dev/null && echo "CODEX_AVAILABLE" || echo "CODEX_NOT_AVAILABLE"
```

如果 Codex 可用，则通过 AskUserQuestion 询问用户：

```
Review complete. Want an independent second opinion from Codex (OpenAI)?

A) Run Codex code review — independent diff review with pass/fail gate
B) Run Codex adversarial challenge — try to find ways this code will fail in production
C) Both — review first, then adversarial challenge
D) Skip — no Codex review needed
```

如果用户选择 A、B 或 C：

**对于 code review（A 或 C）：** 运行 `codex review --base <base>`，超时设为 5 分钟。
将完整输出原样放在 `CODEX SAYS (code review):` 标题下。
检查输出中是否存在 `[P1]` 标记；如果有，就记为 `GATE: FAIL`，否则记为 `GATE: PASS`。
展示完之后，再把 Codex 的发现与 Steps 4-5 中你自己的发现做对照，输出一段 CROSS-MODEL ANALYSIS，说明双方都发现了什么、只有 Codex 发现了什么、只有 Claude 发现了什么。

**对于 adversarial challenge（B 或 C）：** 运行：
```bash
codex exec "Review the changes on this branch against the base branch. Run git diff origin/<base> to see the diff. Your job is to find ways this code will fail in production. Think like an attacker and a chaos engineer. Find edge cases, race conditions, security holes, failure modes. Be adversarial." -s read-only
```
将完整输出原样放在 `CODEX SAYS (adversarial challenge):` 标题下。

**只有在 code review 实际运行过时（用户选择了 A 或 C）：** 才把 Codex review 结果写进 review log：
```bash
~/.codex/skills/gstack/bin/gstack-review-log '{"skill":"codex-review","timestamp":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","status":"STATUS","gate":"GATE"}'
```

替换参数时：`STATUS` 使用 `"clean"`（若 PASS）或 `"issues_found"`（若 FAIL）；`GATE` 使用 `"pass"` 或 `"fail"`。

**如果只运行了 adversarial challenge（B），绝对不要写 codex-review 日志。** 因为这时没有 gate 结论；写入错误记录会让 Review Readiness Dashboard 误以为 code review 已经做过。

如果 Codex 不可用，则静默跳过此步骤。

---

## 重要规则

- **在发表评论前先读完整 diff。** 已经在 diff 里修掉的问题不要再报。
- **Fix-first，而不是只读旁观。** AUTO-FIX 直接修，ASK 只有在用户批准后才修。不要 commit、push 或创建 PR，那是 `/ship` 的职责。
- **保持简洁。** 一行问题，一行修法，不要写前言。
- **只报告真实问题。** 没问题的内容直接跳过。
- **Greptile 回复必须使用 `greptile-triage.md` 里的模板。** 每条回复都要有证据，不能含糊其辞。
