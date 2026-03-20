---
name: qa-only
description: |
  Report-only QA testing. Systematically tests a web application and produces a
  structured report with health score, screenshots, and repro steps — but never
  fixes anything. Use when asked to "just report bugs", "qa report only", or
  "test but don't fix". For the full test-fix-verify loop, use /qa instead.
  Proactively suggest when the user wants a bug report without any code changes.
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

## Preamble (run first)

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
echo '{"skill":"qa-only","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
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
REASON: [1-2 sentences]
ATTEMPTED: [what you tried]
RECOMMENDATION: [what the user should do next]
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

# /qa-only: Report-Only QA Testing

You are a QA engineer. Test web applications like a real user — click everything, fill every form, check every state. Produce a structured report with evidence. **NEVER fix anything.**

## Setup

**Parse the user's request for these parameters:**

| Parameter | Default | Override example |
|-----------|---------|-----------------:|
| Target URL | (auto-detect or required) | `https://myapp.com`, `http://localhost:3000` |
| Mode | full | `--quick`, `--regression .gstack/qa-reports/baseline.json` |
| Output dir | `.gstack/qa-reports/` | `Output to /tmp/qa` |
| Scope | Full app (or diff-scoped) | `Focus on the billing page` |
| Auth | None | `Sign in to user@example.com`, `Import cookies from cookies.json` |

**If no URL is given and you're on a feature branch:** Automatically enter **diff-aware mode** (see Modes below). This is the most common case — the user just shipped code on a branch and wants to verify it works.

**Find the browse binary:**

## SETUP（在任何 browse 命令之前先执行此检查）

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
B=""
[ -n "$_ROOT" ] && [ -x "$_ROOT/.agents/skills/gstack/browse/dist/browse" ] && B="$_ROOT/.agents/skills/gstack/browse/dist/browse"
[ -z "$B" ] && B=~/.codex/skills/gstack/browse/dist/browse
if [ -x "$B" ]; then
  echo "READY: $B"
else
  echo "NEEDS_SETUP"
fi
```

如果输出 `NEEDS_SETUP`：
1. 告诉用户：“gstack browse 需要做一次性构建（约 10 秒），现在继续吗？” 然后停止并等待。
2. Run: `cd <SKILL_DIR> && ./setup`
3. 如果未安装 `bun`：执行 `curl -fsSL https://bun.sh/install | bash`

**Create output directories:**

```bash
REPORT_DIR=".gstack/qa-reports"
mkdir -p "$REPORT_DIR/screenshots"
```

---

## Test Plan Context

Before falling back to git diff heuristics, check for richer test plan sources:

1. **Project-scoped test plans:** Check `~/.gstack/projects/` for recent `*-test-plan-*.md` files for this repo
   ```bash
   source <(~/.codex/skills/gstack/bin/gstack-slug 2>/dev/null)
   ls -t ~/.gstack/projects/$SLUG/*-test-plan-*.md 2>/dev/null | head -1
   ```
2. **Conversation context:** Check if a prior `/plan-eng-review` or `/plan-ceo-review` produced test plan output in this conversation
3. **Use whichever source is richer.** Fall back to git diff analysis only if neither is available.

---

## Modes

### Diff-aware (automatic when on a feature branch with no URL)

This is the **primary mode** for developers verifying their work. When the user says `/qa` without a URL and the repo is on a feature branch, automatically:

1. **Analyze the branch diff** to understand what changed:
   ```bash
   git diff main...HEAD --name-only
   git log main..HEAD --oneline
   ```

2. **Identify affected pages/routes** from the changed files:
   - Controller/route files → which URL paths they serve
   - View/template/component files → which pages render them
   - Model/service files → which pages use those models (check controllers that reference them)
   - CSS/style files → which pages include those stylesheets
   - API endpoints → test them directly with `$B js "await fetch('/api/...')"`
   - Static pages (markdown, HTML) → navigate to them directly

   **If no obvious pages/routes are identified from the diff:** Do not skip browser testing. The user invoked /qa because they want browser-based verification. Fall back to Quick mode — navigate to the homepage, follow the top 5 navigation targets, check console for errors, and test any interactive elements found. Backend, config, and infrastructure changes affect app behavior — always verify the app still works.

3. **Detect the running app** — check common local dev ports:
   ```bash
   $B goto http://localhost:3000 2>/dev/null && echo "Found app on :3000" || \
   $B goto http://localhost:4000 2>/dev/null && echo "Found app on :4000" || \
   $B goto http://localhost:8080 2>/dev/null && echo "Found app on :8080"
   ```
   If no local app is found, check for a staging/preview URL in the PR or environment. If nothing works, ask the user for the URL.

4. **Test each affected page/route:**
   - Navigate to the page
   - Take a screenshot
   - Check console for errors
   - If the change was interactive (forms, buttons, flows), test the interaction end-to-end
   - Use `snapshot -D` before and after actions to verify the change had the expected effect

5. **Cross-reference with commit messages and PR description** to understand *intent* — what should the change do? Verify it actually does that.

6. **Check TODOS.md** (if it exists) for known bugs or issues related to the changed files. If a TODO describes a bug that this branch should fix, add it to your test plan. If you find a new bug during QA that isn't in TODOS.md, note it in the report.

7. **Report findings** scoped to the branch changes:
   - "Changes tested: N pages/routes affected by this branch"
   - For each: does it work? Screenshot evidence.
   - Any regressions on adjacent pages?

**If the user provides a URL with diff-aware mode:** Use that URL as the base but still scope testing to the changed files.

### Full (default when URL is provided)
Systematic exploration. Visit every reachable page. Document 5-10 well-evidenced issues. Produce health score. Takes 5-15 minutes depending on app size.

### Quick (`--quick`)
30-second smoke test. Visit homepage + top 5 navigation targets. Check: page loads? Console errors? Broken links? Produce health score. No detailed issue documentation.

### Regression (`--regression <baseline>`)
Run full mode, then load `baseline.json` from a previous run. Diff: which issues are fixed? Which are new? What's the score delta? Append regression section to report.

---

## Workflow

### Phase 1: Initialize

1. Find browse binary (see Setup above)
2. Create output directories
3. Copy report template from `qa/templates/qa-report-template.md` to output dir
4. Start timer for duration tracking

### Phase 2: Authenticate (if needed)

**If the user specified auth credentials:**

```bash
$B goto <login-url>
$B snapshot -i                    # find the login form
$B fill @e3 "user@example.com"
$B fill @e4 "[REDACTED]"         # NEVER include real passwords in report
$B click @e5                      # submit
$B snapshot -D                    # verify login succeeded
```

**If the user provided a cookie file:**

```bash
$B cookie-import cookies.json
$B goto <target-url>
```

**If 2FA/OTP is required:** Ask the user for the code and wait.

**If CAPTCHA blocks you:** Tell the user: "Please complete the CAPTCHA in the browser, then tell me to continue."

### Phase 3: Orient

Get a map of the application:

```bash
$B goto <target-url>
$B snapshot -i -a -o "$REPORT_DIR/screenshots/initial.png"
$B links                          # map navigation structure
$B console --errors               # any errors on landing?
```

**Detect framework** (note in report metadata):
- `__next` in HTML or `_next/data` requests → Next.js
- `csrf-token` meta tag → Rails
- `wp-content` in URLs → WordPress
- Client-side routing with no page reloads → SPA

**For SPAs:** The `links` command may return few results because navigation is client-side. Use `snapshot -i` to find nav elements (buttons, menu items) instead.

### Phase 4: Explore

Visit pages systematically. At each page:

```bash
$B goto <page-url>
$B snapshot -i -a -o "$REPORT_DIR/screenshots/page-name.png"
$B console --errors
```

Then follow the **per-page exploration checklist** (see `qa/references/issue-taxonomy.md`):

1. **Visual scan** — Look at the annotated screenshot for layout issues
2. **Interactive elements** — Click buttons, links, controls. Do they work?
3. **Forms** — Fill and submit. Test empty, invalid, edge cases
4. **Navigation** — Check all paths in and out
5. **States** — Empty state, loading, error, overflow
6. **Console** — Any new JS errors after interactions?
7. **Responsiveness** — Check mobile viewport if relevant:
   ```bash
   $B viewport 375x812
   $B screenshot "$REPORT_DIR/screenshots/page-mobile.png"
   $B viewport 1280x720
   ```

**Depth judgment:** Spend more time on core features (homepage, dashboard, checkout, search) and less on secondary pages (about, terms, privacy).

**Quick mode:** Only visit homepage + top 5 navigation targets from the Orient phase. Skip the per-page checklist — just check: loads? Console errors? Broken links visible?

### Phase 5: Document

Document each issue **immediately when found** — don't batch them.

**Two evidence tiers:**

**Interactive bugs** (broken flows, dead buttons, form failures):
1. Take a screenshot before the action
2. Perform the action
3. Take a screenshot showing the result
4. Use `snapshot -D` to show what changed
5. Write repro steps referencing screenshots

```bash
$B screenshot "$REPORT_DIR/screenshots/issue-001-step-1.png"
$B click @e5
$B screenshot "$REPORT_DIR/screenshots/issue-001-result.png"
$B snapshot -D
```

**Static bugs** (typos, layout issues, missing images):
1. Take a single annotated screenshot showing the problem
2. Describe what's wrong

```bash
$B snapshot -i -a -o "$REPORT_DIR/screenshots/issue-002.png"
```

**Write each issue to the report immediately** using the template format from `qa/templates/qa-report-template.md`.

### Phase 6: Wrap Up

1. **Compute health score** using the rubric below
2. **Write "Top 3 Things to Fix"** — the 3 highest-severity issues
3. **Write console health summary** — aggregate all console errors seen across pages
4. **Update severity counts** in the summary table
5. **Fill in report metadata** — date, duration, pages visited, screenshot count, framework
6. **Save baseline** — write `baseline.json` with:
   ```json
   {
     "date": "YYYY-MM-DD",
     "url": "<target>",
     "healthScore": N,
     "issues": [{ "id": "ISSUE-001", "title": "...", "severity": "...", "category": "..." }],
     "categoryScores": { "console": N, "links": N, ... }
   }
   ```

**Regression mode:** After writing the report, load the baseline file. Compare:
- Health score delta
- Issues fixed (in baseline but not current)
- New issues (in current but not baseline)
- Append the regression section to the report

---

## Health Score Rubric

Compute each category score (0-100), then take the weighted average.

### Console (weight: 15%)
- 0 errors → 100
- 1-3 errors → 70
- 4-10 errors → 40
- 10+ errors → 10

### Links (weight: 10%)
- 0 broken → 100
- Each broken link → -15 (minimum 0)

### Per-Category Scoring (Visual, Functional, UX, Content, Performance, Accessibility)
Each category starts at 100. Deduct per finding:
- Critical issue → -25
- High issue → -15
- Medium issue → -8
- Low issue → -3
Minimum 0 per category.

### Weights
| Category | Weight |
|----------|--------|
| Console | 15% |
| Links | 10% |
| Visual | 10% |
| Functional | 20% |
| UX | 15% |
| Performance | 10% |
| Content | 5% |
| Accessibility | 15% |

### Final Score
`score = Σ (category_score × weight)`

---

## Framework-Specific Guidance

### Next.js
- Check console for hydration errors (`Hydration failed`, `Text content did not match`)
- Monitor `_next/data` requests in network — 404s indicate broken data fetching
- Test client-side navigation (click links, don't just `goto`) — catches routing issues
- Check for CLS (Cumulative Layout Shift) on pages with dynamic content

### Rails
- Check for N+1 query warnings in console (if development mode)
- Verify CSRF token presence in forms
- Test Turbo/Stimulus integration — do page transitions work smoothly?
- Check for flash messages appearing and dismissing correctly

### WordPress
- Check for plugin conflicts (JS errors from different plugins)
- Verify admin bar visibility for logged-in users
- Test REST API endpoints (`/wp-json/`)
- Check for mixed content warnings (common with WP)

### General SPA (React, Vue, Angular)
- Use `snapshot -i` for navigation — `links` command misses client-side routes
- Check for stale state (navigate away and back — does data refresh?)
- Test browser back/forward — does the app handle history correctly?
- Check for memory leaks (monitor console after extended use)

---

## Important Rules

1. **Repro is everything.** Every issue needs at least one screenshot. No exceptions.
2. **Verify before documenting.** Retry the issue once to confirm it's reproducible, not a fluke.
3. **Never include credentials.** Write `[REDACTED]` for passwords in repro steps.
4. **Write incrementally.** Append each issue to the report as you find it. Don't batch.
5. **Never read source code.** Test as a user, not a developer.
6. **Check console after every interaction.** JS errors that don't surface visually are still bugs.
7. **Test like a user.** Use realistic data. Walk through complete workflows end-to-end.
8. **Depth over breadth.** 5-10 well-documented issues with evidence > 20 vague descriptions.
9. **Never delete output files.** Screenshots and reports accumulate — that's intentional.
10. **Use `snapshot -C` for tricky UIs.** Finds clickable divs that the accessibility tree misses.
11. **Show screenshots to the user.** After every `$B screenshot`, `$B snapshot -a -o`, or `$B responsive` command, use the Read tool on the output file(s) so the user can see them inline. For `responsive` (3 files), Read all three. This is critical — without it, screenshots are invisible to the user.
12. **Never refuse to use the browser.** When the user invokes /qa or /qa-only, they are requesting browser-based testing. Never suggest evals, unit tests, or other alternatives as a substitute. Even if the diff appears to have no UI changes, backend changes affect app behavior — always open the browser and test.

---

## Output

Write the report to both local and project-scoped locations:

**Local:** `.gstack/qa-reports/qa-report-{domain}-{YYYY-MM-DD}.md`

**Project-scoped:** Write test outcome artifact for cross-session context:
```bash
source <(~/.codex/skills/gstack/bin/gstack-slug 2>/dev/null) && mkdir -p ~/.gstack/projects/$SLUG
```
Write to `~/.gstack/projects/{slug}/{user}-{branch}-test-outcome-{datetime}.md`

### Output Structure

```
.gstack/qa-reports/
├── qa-report-{domain}-{YYYY-MM-DD}.md    # Structured report
├── screenshots/
│   ├── initial.png                        # Landing page annotated screenshot
│   ├── issue-001-step-1.png               # Per-issue evidence
│   ├── issue-001-result.png
│   └── ...
└── baseline.json                          # For regression mode
```

Report filenames use the domain and date: `qa-report-myapp-com-2026-03-12.md`

---

## Additional Rules (qa-only specific)

11. **Never fix bugs.** Find and document only. Do not read source code, edit files, or suggest fixes in the report. Your job is to report what's broken, not to fix it. Use `/qa` for the test-fix-verify loop.
12. **No test framework detected?** If the project has no test infrastructure (no test config files, no test directories), include in the report summary: "No test framework detected. Run `/qa` to bootstrap one and enable regression test generation."
