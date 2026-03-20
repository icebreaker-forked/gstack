---
name: review
version: 1.0.0
description: |
  Pre-landing PR review. Analyzes diff against the base branch for SQL safety, LLM trust
  boundary violations, conditional side effects, and other structural issues. Use when
  asked to "review this PR", "code review", "pre-landing review", or "check my diff".
  Proactively suggest when the user is about to merge or land code changes.
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
  - Grep
  - Glob
  - AskUserQuestion
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

## Preamble (run first)

```bash
_UPD=$(~/.claude/skills/gstack/bin/gstack-update-check 2>/dev/null || .claude/skills/gstack/bin/gstack-update-check 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD" || true
mkdir -p ~/.gstack/sessions
touch ~/.gstack/sessions/"$PPID"
_SESSIONS=$(find ~/.gstack/sessions -mmin -120 -type f 2>/dev/null | wc -l | tr -d ' ')
find ~/.gstack/sessions -mmin +120 -type f -delete 2>/dev/null || true
_CONTRIB=$(~/.claude/skills/gstack/bin/gstack-config get gstack_contributor 2>/dev/null || true)
_PROACTIVE=$(~/.claude/skills/gstack/bin/gstack-config get proactive 2>/dev/null || echo "true")
_BRANCH=$(git branch --show-current 2>/dev/null || echo "unknown")
echo "BRANCH: $_BRANCH"
echo "PROACTIVE: $_PROACTIVE"
_LAKE_SEEN=$([ -f ~/.gstack/.completeness-intro-seen ] && echo "yes" || echo "no")
echo "LAKE_INTRO: $_LAKE_SEEN"
_TEL=$(~/.claude/skills/gstack/bin/gstack-config get telemetry 2>/dev/null || true)
_TEL_PROMPTED=$([ -f ~/.gstack/.telemetry-prompted ] && echo "yes" || echo "no")
_TEL_START=$(date +%s)
_SESSION_ID="$$-$(date +%s)"
echo "TELEMETRY: ${_TEL:-off}"
echo "TEL_PROMPTED: $_TEL_PROMPTED"
mkdir -p ~/.gstack/analytics
echo '{"skill":"review","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
for _PF in ~/.gstack/analytics/.pending-*; do [ -f "$_PF" ] && ~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type skill_run --skill _pending_finalize --outcome unknown --session-id "$_SESSION_ID" 2>/dev/null || true; break; done
```

如果 `PROACTIVE` 为 `"false"`，不要主动推荐 gstack 技能，只在用户明确要求时调用。用户已经关闭了主动建议。

如果输出为 `UPGRADE_AVAILABLE <old> <new>`：读取 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` 并按“Inline upgrade flow”执行（若已配置自动升级则直接升级，否则通过 AskUserQuestion 给出 4 个选项；若用户拒绝则写入 snooze 状态）。如果输出为 `JUST_UPGRADED <from> <to>`：告诉用户 “Running gstack v{to} (just updated!)”，然后继续。

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

If A: run `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

If B: ask a follow-up AskUserQuestion:

> 那匿名模式呢？我们只知道“有人”用了 gstack，没有唯一 ID，也无法串联会话，只是一个帮助我们了解使用量的计数。

Options:
- A) 可以，匿名模式没问题
- B) 不，谢谢，完全关闭

If B→A: run `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
If B→B: run `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

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
~/.claude/skills/gstack/bin/gstack-telemetry-log \
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

You are running the `/review` workflow. Analyze the current branch's diff against the base branch for structural issues that tests don't catch.

---

## Step 1: Check branch

1. Run `git branch --show-current` to get the current branch.
2. If on the base branch, output: **"Nothing to review — you're on the base branch or have no changes against it."** and stop.
3. Run `git fetch origin <base> --quiet && git diff origin/<base> --stat` to check if there's a diff. If no diff, output the same message and stop.

---

## Step 1.5: Scope Drift Detection

Before reviewing code quality, check: **did they build what was requested — nothing more, nothing less?**

1. Read `TODOS.md` (if it exists). Read PR description (`gh pr view --json body --jq .body 2>/dev/null || true`).
   Read commit messages (`git log origin/<base>..HEAD --oneline`).
   **If no PR exists:** rely on commit messages and TODOS.md for stated intent — this is the common case since /review runs before /ship creates the PR.
2. Identify the **stated intent** — what was this branch supposed to accomplish?
3. Run `git diff origin/<base> --stat` and compare the files changed against the stated intent.
4. Evaluate with skepticism:

   **SCOPE CREEP detection:**
   - Files changed that are unrelated to the stated intent
   - New features or refactors not mentioned in the plan
   - "While I was in there..." changes that expand blast radius

   **MISSING REQUIREMENTS detection:**
   - Requirements from TODOS.md/PR description not addressed in the diff
   - Test coverage gaps for stated requirements
   - Partial implementations (started but not finished)

5. Output (before the main review begins):
   ```
   Scope Check: [CLEAN / DRIFT DETECTED / REQUIREMENTS MISSING]
   Intent: <1-line summary of what was requested>
   Delivered: <1-line summary of what the diff actually does>
   [If drift: list each out-of-scope change]
   [If missing: list each unaddressed requirement]
   ```

6. This is **INFORMATIONAL** — does not block the review. Proceed to Step 2.

---

## Step 2: Read the checklist

Read `.claude/skills/review/checklist.md`.

**If the file cannot be read, STOP and report the error.** Do not proceed without the checklist.

---

## Step 2.5: Check for Greptile review comments

Read `.claude/skills/review/greptile-triage.md` and follow the fetch, filter, classify, and **escalation detection** steps.

**If no PR exists, `gh` fails, API returns an error, or there are zero Greptile comments:** Skip this step silently. Greptile integration is additive — the review works without it.

**If Greptile comments are found:** Store the classifications (VALID & ACTIONABLE, VALID BUT ALREADY FIXED, FALSE POSITIVE, SUPPRESSED) — you will need them in Step 5.

---

## Step 3: Get the diff

Fetch the latest base branch to avoid false positives from stale local state:

```bash
git fetch origin <base> --quiet
```

Run `git diff origin/<base>` to get the full diff. This includes both committed and uncommitted changes against the latest base branch.

---

## Step 4: Two-pass review

Apply the checklist against the diff in two passes:

1. **Pass 1 (CRITICAL):** SQL & Data Safety, Race Conditions & Concurrency, LLM Output Trust Boundary, Enum & Value Completeness
2. **Pass 2 (INFORMATIONAL):** Conditional Side Effects, Magic Numbers & String Coupling, Dead Code & Consistency, LLM Prompt Issues, Test Gaps, View/Frontend

**Enum & Value Completeness requires reading code OUTSIDE the diff.** When the diff introduces a new enum value, status, tier, or type constant, use Grep to find all files that reference sibling values, then Read those files to check if the new value is handled. This is the one category where within-diff review is insufficient.

Follow the output format specified in the checklist. Respect the suppressions — do NOT flag items listed in the "DO NOT flag" section.

---

## Step 4.5: Design Review (conditional)

## Design Review (conditional, diff-scoped)

Check if the diff touches frontend files using `gstack-diff-scope`:

```bash
source <(~/.claude/skills/gstack/bin/gstack-diff-scope <base> 2>/dev/null)
```

**If `SCOPE_FRONTEND=false`:** Skip design review silently. No output.

**If `SCOPE_FRONTEND=true`:**

1. **Check for DESIGN.md.** If `DESIGN.md` or `design-system.md` exists in the repo root, read it. All design findings are calibrated against it — patterns blessed in DESIGN.md are not flagged. If not found, use universal design principles.

2. **Read `.claude/skills/review/design-checklist.md`.** If the file cannot be read, skip design review with a note: "Design checklist not found — skipping design review."

3. **Read each changed frontend file** (full file, not just diff hunks). Frontend files are identified by the patterns listed in the checklist.

4. **Apply the design checklist** against the changed files. For each item:
   - **[HIGH] mechanical CSS fix** (`outline: none`, `!important`, `font-size < 16px`): classify as AUTO-FIX
   - **[HIGH/MEDIUM] design judgment needed**: classify as ASK
   - **[LOW] intent-based detection**: present as "Possible — verify visually or run /design-review"

5. **Include findings** in the review output under a "Design Review" header, following the output format in the checklist. Design findings merge with code review findings into the same Fix-First flow.

6. **Log the result** for the Review Readiness Dashboard:

```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"design-review-lite","timestamp":"TIMESTAMP","status":"STATUS","findings":N,"auto_fixed":M,"commit":"COMMIT"}'
```

Substitute: TIMESTAMP = ISO 8601 datetime, STATUS = "clean" if 0 findings or "issues_found", N = total findings, M = auto-fixed count, COMMIT = output of `git rev-parse --short HEAD`.

Include any design findings alongside the findings from Step 4. They follow the same Fix-First flow in Step 5 — AUTO-FIX for mechanical CSS fixes, ASK for everything else.

---

## Step 5: Fix-First Review

**Every finding gets action — not just critical ones.**

Output a summary header: `Pre-Landing Review: N issues (X critical, Y informational)`

### Step 5a: Classify each finding

For each finding, classify as AUTO-FIX or ASK per the Fix-First Heuristic in
checklist.md. Critical findings lean toward ASK; informational findings lean
toward AUTO-FIX.

### Step 5b: Auto-fix all AUTO-FIX items

Apply each fix directly. For each one, output a one-line summary:
`[AUTO-FIXED] [file:line] Problem → what you did`

### Step 5c: Batch-ask about ASK items

If there are ASK items remaining, present them in ONE AskUserQuestion:

- List each item with a number, the severity label, the problem, and a recommended fix
- For each item, provide options: A) Fix as recommended, B) Skip
- Include an overall RECOMMENDATION

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

If 3 or fewer ASK items, you may use individual AskUserQuestion calls instead of batching.

### Step 5d: Apply user-approved fixes

Apply fixes for items where the user chose "Fix." Output what was fixed.

If no ASK items exist (everything was AUTO-FIX), skip the question entirely.

### Verification of claims

Before producing the final review output:
- If you claim "this pattern is safe" → cite the specific line proving safety
- If you claim "this is handled elsewhere" → read and cite the handling code
- If you claim "tests cover this" → name the test file and method
- Never say "likely handled" or "probably tested" — verify or flag as unknown

**Rationalization prevention:** "This looks fine" is not a finding. Either cite evidence it IS fine, or flag it as unverified.

### Greptile comment resolution

After outputting your own findings, if Greptile comments were classified in Step 2.5:

**Include a Greptile summary in your output header:** `+ N Greptile comments (X valid, Y fixed, Z FP)`

Before replying to any comment, run the **Escalation Detection** algorithm from greptile-triage.md to determine whether to use Tier 1 (friendly) or Tier 2 (firm) reply templates.

1. **VALID & ACTIONABLE comments:** These are included in your findings — they follow the Fix-First flow (auto-fixed if mechanical, batched into ASK if not) (A: Fix it now, B: Acknowledge, C: False positive). If the user chooses A (fix), reply using the **Fix reply template** from greptile-triage.md (include inline diff + explanation). If the user chooses C (false positive), reply using the **False Positive reply template** (include evidence + suggested re-rank), save to both per-project and global greptile-history.

2. **FALSE POSITIVE comments:** Present each one via AskUserQuestion:
   - Show the Greptile comment: file:line (or [top-level]) + body summary + permalink URL
   - Explain concisely why it's a false positive
   - Options:
     - A) Reply to Greptile explaining why this is incorrect (recommended if clearly wrong)
     - B) Fix it anyway (if low-effort and harmless)
     - C) Ignore — don't reply, don't fix

   If the user chooses A, reply using the **False Positive reply template** from greptile-triage.md (include evidence + suggested re-rank), save to both per-project and global greptile-history.

3. **VALID BUT ALREADY FIXED comments:** Reply using the **Already Fixed reply template** from greptile-triage.md — no AskUserQuestion needed:
   - Include what was done and the fixing commit SHA
   - Save to both per-project and global greptile-history

4. **SUPPRESSED comments:** Skip silently — these are known false positives from previous triage.

---

## Step 5.5: TODOS cross-reference

Read `TODOS.md` in the repository root (if it exists). Cross-reference the PR against open TODOs:

- **Does this PR close any open TODOs?** If yes, note which items in your output: "This PR addresses TODO: <title>"
- **Does this PR create work that should become a TODO?** If yes, flag it as an informational finding.
- **Are there related TODOs that provide context for this review?** If yes, reference them when discussing related findings.

If TODOS.md doesn't exist, skip this step silently.

---

## Step 5.6: Documentation staleness check

Cross-reference the diff against documentation files. For each `.md` file in the repo root (README.md, ARCHITECTURE.md, CONTRIBUTING.md, CLAUDE.md, etc.):

1. Check if code changes in the diff affect features, components, or workflows described in that doc file.
2. If the doc file was NOT updated in this branch but the code it describes WAS changed, flag it as an INFORMATIONAL finding:
   "Documentation may be stale: [file] describes [feature/component] but code changed in this branch. Consider running `/document-release`."

This is informational only — never critical. The fix action is `/document-release`.

If no documentation files exist, skip this step silently.

---

## Step 5.7: Codex second opinion (optional)

After completing the review, check if the Codex CLI is available:

```bash
which codex 2>/dev/null && echo "CODEX_AVAILABLE" || echo "CODEX_NOT_AVAILABLE"
```

If Codex is available, use AskUserQuestion:

```
Review complete. Want an independent second opinion from Codex (OpenAI)?

A) Run Codex code review — independent diff review with pass/fail gate
B) Run Codex adversarial challenge — try to find ways this code will fail in production
C) Both — review first, then adversarial challenge
D) Skip — no Codex review needed
```

If the user chooses A, B, or C:

**For code review (A or C):** Run `codex review --base <base>` with a 5-minute timeout.
Present the full output verbatim under a `CODEX SAYS (code review):` header.
Check the output for `[P1]` markers — if found, note `GATE: FAIL`, otherwise `GATE: PASS`.
After presenting, compare Codex's findings with your own review findings from Steps 4-5
and output a CROSS-MODEL ANALYSIS showing what both found, what only Codex found,
and what only Claude found.

**For adversarial challenge (B or C):** Run:
```bash
codex exec "Review the changes on this branch against the base branch. Run git diff origin/<base> to see the diff. Your job is to find ways this code will fail in production. Think like an attacker and a chaos engineer. Find edge cases, race conditions, security holes, failure modes. Be adversarial." -s read-only
```
Present the full output verbatim under a `CODEX SAYS (adversarial challenge):` header.

**Only if a code review ran (user chose A or C):** Persist the Codex review result to the review log:
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"codex-review","timestamp":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","status":"STATUS","gate":"GATE"}'
```

Substitute: STATUS ("clean" if PASS, "issues_found" if FAIL), GATE ("pass" or "fail").

**Do NOT persist a codex-review entry when only the adversarial challenge (B) ran** —
there is no gate verdict to record, and a false entry would make the Review Readiness
Dashboard believe a code review happened when it didn't.

If Codex is not available, skip this step silently.

---

## Important Rules

- **Read the FULL diff before commenting.** Do not flag issues already addressed in the diff.
- **Fix-first, not read-only.** AUTO-FIX items are applied directly. ASK items are only applied after user approval. Never commit, push, or create PRs — that's /ship's job.
- **Be terse.** One line problem, one line fix. No preamble.
- **Only flag real problems.** Skip anything that's fine.
- **Use Greptile reply templates from greptile-triage.md.** Every reply includes evidence. Never post vague replies.
