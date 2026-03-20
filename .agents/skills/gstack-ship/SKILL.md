---
name: ship
description: |
  Ship workflow: detect + merge base branch, run tests, review diff, bump VERSION, update CHANGELOG, commit, push, create PR. Use when asked to "ship", "deploy", "push to main", "create a PR", or "merge and push".
  Proactively suggest when the user says code is ready or asks about deploying.
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
echo '{"skill":"ship","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
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

# Ship: Fully Automated Ship Workflow

You are running the `/ship` workflow. This is a **non-interactive, fully automated** workflow. Do NOT ask for confirmation at any step. The user said `/ship` which means DO IT. Run straight through and output the PR URL at the end.

**Only stop for:**
- On the base branch (abort)
- Merge conflicts that can't be auto-resolved (stop, show conflicts)
- Test failures (stop, show failures)
- Pre-landing review finds ASK items that need user judgment
- MINOR or MAJOR version bump needed (ask — see Step 4)
- Greptile review comments that need user decision (complex fixes, false positives)
- TODOS.md missing and user wants to create one (ask — see Step 5.5)
- TODOS.md disorganized and user wants to reorganize (ask — see Step 5.5)

**Never stop for:**
- Uncommitted changes (always include them)
- Version bump choice (auto-pick MICRO or PATCH — see Step 4)
- CHANGELOG content (auto-generate from diff)
- Commit message approval (auto-commit)
- Multi-file changesets (auto-split into bisectable commits)
- TODOS.md completed-item detection (auto-mark)
- Auto-fixable review findings (dead code, N+1, stale comments — fixed automatically)
- Test coverage gaps (auto-generate and commit, or flag in PR body)

---

## Step 1: Pre-flight

1. Check the current branch. If on the base branch or the repo's default branch, **abort**: "You're on the base branch. Ship from a feature branch."

2. Run `git status` (never use `-uall`). Uncommitted changes are always included — no need to ask.

3. Run `git diff <base>...HEAD --stat` and `git log <base>..HEAD --oneline` to understand what's being shipped.

4. Check review readiness:

## Review Readiness Dashboard

After completing the review, read the review log and config to display the dashboard.

```bash
~/.codex/skills/gstack/bin/gstack-review-read
```

Parse the output. Find the most recent entry for each skill (plan-ceo-review, plan-eng-review, plan-design-review, design-review-lite, codex-review). Ignore entries with timestamps older than 7 days. For Design Review, show whichever is more recent between `plan-design-review` (full visual audit) and `design-review-lite` (code-level check). Append "(FULL)" or "(LITE)" to the status to distinguish. Display:

```
+====================================================================+
|                    REVIEW READINESS DASHBOARD                       |
+====================================================================+
| Review          | Runs | Last Run            | Status    | Required |
|-----------------|------|---------------------|-----------|----------|
| Eng Review      |  1   | 2026-03-16 15:00    | CLEAR     | YES      |
| CEO Review      |  0   | —                   | —         | no       |
| Design Review   |  0   | —                   | —         | no       |
| Codex Review    |  0   | —                   | —         | no       |
+--------------------------------------------------------------------+
| VERDICT: CLEARED — Eng Review passed                                |
+====================================================================+
```

**Review tiers:**
- **Eng Review (required by default):** The only review that gates shipping. Covers architecture, code quality, tests, performance. Can be disabled globally with \`gstack-config set skip_eng_review true\` (the "don't bother me" setting).
- **CEO Review (optional):** Use your judgment. Recommend it for big product/business changes, new user-facing features, or scope decisions. Skip for bug fixes, refactors, infra, and cleanup.
- **Design Review (optional):** Use your judgment. Recommend it for UI/UX changes. Skip for backend-only, infra, or prompt-only changes.
- **Codex Review (optional):** Independent second opinion from OpenAI Codex CLI. Shows pass/fail gate. Recommend for critical code changes where a second AI perspective adds value. Skip when Codex CLI is not installed.

**Verdict logic:**
- **CLEARED**: Eng Review has >= 1 entry within 7 days with status "clean" (or \`skip_eng_review\` is \`true\`)
- **NOT CLEARED**: Eng Review missing, stale (>7 days), or has open issues
- CEO, Design, and Codex reviews are shown for context but never block shipping
- If \`skip_eng_review\` config is \`true\`, Eng Review shows "SKIPPED (global)" and verdict is CLEARED

**Staleness detection:** After displaying the dashboard, check if any existing reviews may be stale:
- Parse the \`---HEAD---\` section from the bash output to get the current HEAD commit hash
- For each review entry that has a \`commit\` field: compare it against the current HEAD. If different, count elapsed commits: \`git rev-list --count STORED_COMMIT..HEAD\`. Display: "Note: {skill} review from {date} may be stale — {N} commits since review"
- For entries without a \`commit\` field (legacy entries): display "Note: {skill} review from {date} has no commit tracking — consider re-running for accurate staleness detection"
- If all reviews match the current HEAD, do not display any staleness notes

If the Eng Review is NOT "CLEAR":

1. **Check for a prior override on this branch:**
   ```bash
   source <(~/.codex/skills/gstack/bin/gstack-slug 2>/dev/null)
   grep '"skill":"ship-review-override"' ~/.gstack/projects/$SLUG/$BRANCH-reviews.jsonl 2>/dev/null || echo "NO_OVERRIDE"
   ```
   If an override exists, display the dashboard and note "Review gate previously accepted — continuing." Do NOT ask again.

2. **If no override exists,** use AskUserQuestion:
   - Show that Eng Review is missing or has open issues
   - RECOMMENDATION: Choose C if the change is obviously trivial (< 20 lines, typo fix, config-only); Choose B for larger changes
   - Options: A) Ship anyway  B) Abort — run /plan-eng-review first  C) Change is too small to need eng review
   - If CEO Review is missing, mention as informational ("CEO Review not run — recommended for product changes") but do NOT block
   - For Design Review: run `source <(~/.codex/skills/gstack/bin/gstack-diff-scope <base> 2>/dev/null)`. If `SCOPE_FRONTEND=true` and no design review (plan-design-review or design-review-lite) exists in the dashboard, mention: "Design Review not run — this PR changes frontend code. The lite design check will run automatically in Step 3.5, but consider running /design-review for a full visual audit post-implementation." Still never block.

3. **If the user chooses A or C,** persist the decision so future `/ship` runs on this branch skip the gate:
   ```bash
   source <(~/.codex/skills/gstack/bin/gstack-slug 2>/dev/null)
   echo '{"skill":"ship-review-override","timestamp":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","decision":"USER_CHOICE"}' >> ~/.gstack/projects/$SLUG/$BRANCH-reviews.jsonl
   ```
   Substitute USER_CHOICE with "ship_anyway" or "not_relevant".

---

## Step 2: Merge the base branch (BEFORE tests)

Fetch and merge the base branch into the feature branch so tests run against the merged state:

```bash
git fetch origin <base> && git merge origin/<base> --no-edit
```

**If there are merge conflicts:** Try to auto-resolve if they are simple (VERSION, schema.rb, CHANGELOG ordering). If conflicts are complex or ambiguous, **STOP** and show them.

**If already up to date:** Continue silently.

---

## Step 2.5: Test Framework Bootstrap

## Test Framework Bootstrap

**Detect existing test framework and project runtime:**

```bash
# Detect project runtime
[ -f Gemfile ] && echo "RUNTIME:ruby"
[ -f package.json ] && echo "RUNTIME:node"
[ -f requirements.txt ] || [ -f pyproject.toml ] && echo "RUNTIME:python"
[ -f go.mod ] && echo "RUNTIME:go"
[ -f Cargo.toml ] && echo "RUNTIME:rust"
[ -f composer.json ] && echo "RUNTIME:php"
[ -f mix.exs ] && echo "RUNTIME:elixir"
# Detect sub-frameworks
[ -f Gemfile ] && grep -q "rails" Gemfile 2>/dev/null && echo "FRAMEWORK:rails"
[ -f package.json ] && grep -q '"next"' package.json 2>/dev/null && echo "FRAMEWORK:nextjs"
# Check for existing test infrastructure
ls jest.config.* vitest.config.* playwright.config.* .rspec pytest.ini pyproject.toml phpunit.xml 2>/dev/null
ls -d test/ tests/ spec/ __tests__/ cypress/ e2e/ 2>/dev/null
# Check opt-out marker
[ -f .gstack/no-test-bootstrap ] && echo "BOOTSTRAP_DECLINED"
```

**If test framework detected** (config files or test directories found):
Print "Test framework detected: {name} ({N} existing tests). Skipping bootstrap."
Read 2-3 existing test files to learn conventions (naming, imports, assertion style, setup patterns).
Store conventions as prose context for use in Phase 8e.5 or Step 3.4. **Skip the rest of bootstrap.**

**If BOOTSTRAP_DECLINED** appears: Print "Test bootstrap previously declined — skipping." **Skip the rest of bootstrap.**

**If NO runtime detected** (no config files found): Use AskUserQuestion:
"I couldn't detect your project's language. What runtime are you using?"
Options: A) Node.js/TypeScript B) Ruby/Rails C) Python D) Go E) Rust F) PHP G) Elixir H) This project doesn't need tests.
If user picks H → write `.gstack/no-test-bootstrap` and continue without tests.

**If runtime detected but no test framework — bootstrap:**

### B2. Research best practices

Use WebSearch to find current best practices for the detected runtime:
- `"[runtime] best test framework 2025 2026"`
- `"[framework A] vs [framework B] comparison"`

If WebSearch is unavailable, use this built-in knowledge table:

| Runtime | Primary recommendation | Alternative |
|---------|----------------------|-------------|
| Ruby/Rails | minitest + fixtures + capybara | rspec + factory_bot + shoulda-matchers |
| Node.js | vitest + @testing-library | jest + @testing-library |
| Next.js | vitest + @testing-library/react + playwright | jest + cypress |
| Python | pytest + pytest-cov | unittest |
| Go | stdlib testing + testify | stdlib only |
| Rust | cargo test (built-in) + mockall | — |
| PHP | phpunit + mockery | pest |
| Elixir | ExUnit (built-in) + ex_machina | — |

### B3. Framework selection

Use AskUserQuestion:
"I detected this is a [Runtime/Framework] project with no test framework. I researched current best practices. Here are the options:
A) [Primary] — [rationale]. Includes: [packages]. Supports: unit, integration, smoke, e2e
B) [Alternative] — [rationale]. Includes: [packages]
C) Skip — don't set up testing right now
RECOMMENDATION: Choose A because [reason based on project context]"

If user picks C → write `.gstack/no-test-bootstrap`. Tell user: "If you change your mind later, delete `.gstack/no-test-bootstrap` and re-run." Continue without tests.

If multiple runtimes detected (monorepo) → ask which runtime to set up first, with option to do both sequentially.

### B4. Install and configure

1. Install the chosen packages (npm/bun/gem/pip/etc.)
2. Create minimal config file
3. Create directory structure (test/, spec/, etc.)
4. Create one example test matching the project's code to verify setup works

If package installation fails → debug once. If still failing → revert with `git checkout -- package.json package-lock.json` (or equivalent for the runtime). Warn user and continue without tests.

### B4.5. First real tests

Generate 3-5 real tests for existing code:

1. **Find recently changed files:** `git log --since=30.days --name-only --format="" | sort | uniq -c | sort -rn | head -10`
2. **Prioritize by risk:** Error handlers > business logic with conditionals > API endpoints > pure functions
3. **For each file:** Write one test that tests real behavior with meaningful assertions. Never `expect(x).toBeDefined()` — test what the code DOES.
4. Run each test. Passes → keep. Fails → fix once. Still fails → delete silently.
5. Generate at least 1 test, cap at 5.

Never import secrets, API keys, or credentials in test files. Use environment variables or test fixtures.

### B5. Verify

```bash
# Run the full test suite to confirm everything works
{detected test command}
```

If tests fail → debug once. If still failing → revert all bootstrap changes and warn user.

### B5.5. CI/CD pipeline

```bash
# Check CI provider
ls -d .github/ 2>/dev/null && echo "CI:github"
ls .gitlab-ci.yml .circleci/ bitrise.yml 2>/dev/null
```

If `.github/` exists (or no CI detected — default to GitHub Actions):
Create `.github/workflows/test.yml` with:
- `runs-on: ubuntu-latest`
- Appropriate setup action for the runtime (setup-node, setup-ruby, setup-python, etc.)
- The same test command verified in B5
- Trigger: push + pull_request

If non-GitHub CI detected → skip CI generation with note: "Detected {provider} — CI pipeline generation supports GitHub Actions only. Add test step to your existing pipeline manually."

### B6. Create TESTING.md

First check: If TESTING.md already exists → read it and update/append rather than overwriting. Never destroy existing content.

Write TESTING.md with:
- Philosophy: "100% test coverage is the key to great vibe coding. Tests let you move fast, trust your instincts, and ship with confidence — without them, vibe coding is just yolo coding. With tests, it's a superpower."
- Framework name and version
- How to run tests (the verified command from B5)
- Test layers: Unit tests (what, where, when), Integration tests, Smoke tests, E2E tests
- Conventions: file naming, assertion style, setup/teardown patterns

### B7. Update CLAUDE.md

First check: If CLAUDE.md already has a `## Testing` section → skip. Don't duplicate.

Append a `## Testing` section:
- Run command and test directory
- Reference to TESTING.md
- Test expectations:
  - 100% test coverage is the goal — tests make vibe coding safe
  - When writing new functions, write a corresponding test
  - When fixing a bug, write a regression test
  - When adding error handling, write a test that triggers the error
  - When adding a conditional (if/else, switch), write tests for BOTH paths
  - Never commit code that makes existing tests fail

### B8. Commit

```bash
git status --porcelain
```

Only commit if there are changes. Stage all bootstrap files (config, test directory, TESTING.md, CLAUDE.md, .github/workflows/test.yml if created):
`git commit -m "chore: bootstrap test framework ({framework name})"`

---

---

## Step 3: Run tests (on merged code)

**Do NOT run `RAILS_ENV=test bin/rails db:migrate`** — `bin/test-lane` already calls
`db:test:prepare` internally, which loads the schema into the correct lane database.
Running bare test migrations without INSTANCE hits an orphan DB and corrupts structure.sql.

Run both test suites in parallel:

```bash
bin/test-lane 2>&1 | tee /tmp/ship_tests.txt &
npm run test 2>&1 | tee /tmp/ship_vitest.txt &
wait
```

After both complete, read the output files and check pass/fail.

**If any test fails:** Show the failures and **STOP**. Do not proceed.

**If all pass:** Continue silently — just note the counts briefly.

---

## Step 3.25: Eval Suites (conditional)

Evals are mandatory when prompt-related files change. Skip this step entirely if no prompt files are in the diff.

**1. Check if the diff touches prompt-related files:**

```bash
git diff origin/<base> --name-only
```

Match against these patterns (from CLAUDE.md):
- `app/services/*_prompt_builder.rb`
- `app/services/*_generation_service.rb`, `*_writer_service.rb`, `*_designer_service.rb`
- `app/services/*_evaluator.rb`, `*_scorer.rb`, `*_classifier_service.rb`, `*_analyzer.rb`
- `app/services/concerns/*voice*.rb`, `*writing*.rb`, `*prompt*.rb`, `*token*.rb`
- `app/services/chat_tools/*.rb`, `app/services/x_thread_tools/*.rb`
- `config/system_prompts/*.txt`
- `test/evals/**/*` (eval infrastructure changes affect all suites)

**If no matches:** Print "No prompt-related files changed — skipping evals." and continue to Step 3.5.

**2. Identify affected eval suites:**

Each eval runner (`test/evals/*_eval_runner.rb`) declares `PROMPT_SOURCE_FILES` listing which source files affect it. Grep these to find which suites match the changed files:

```bash
grep -l "changed_file_basename" test/evals/*_eval_runner.rb
```

Map runner → test file: `post_generation_eval_runner.rb` → `post_generation_eval_test.rb`.

**Special cases:**
- Changes to `test/evals/judges/*.rb`, `test/evals/support/*.rb`, or `test/evals/fixtures/` affect ALL suites that use those judges/support files. Check imports in the eval test files to determine which.
- Changes to `config/system_prompts/*.txt` — grep eval runners for the prompt filename to find affected suites.
- If unsure which suites are affected, run ALL suites that could plausibly be impacted. Over-testing is better than missing a regression.

**3. Run affected suites at `EVAL_JUDGE_TIER=full`:**

`/ship` is a pre-merge gate, so always use full tier (Sonnet structural + Opus persona judges).

```bash
EVAL_JUDGE_TIER=full EVAL_VERBOSE=1 bin/test-lane --eval test/evals/<suite>_eval_test.rb 2>&1 | tee /tmp/ship_evals.txt
```

If multiple suites need to run, run them sequentially (each needs a test lane). If the first suite fails, stop immediately — don't burn API cost on remaining suites.

**4. Check results:**

- **If any eval fails:** Show the failures, the cost dashboard, and **STOP**. Do not proceed.
- **If all pass:** Note pass counts and cost. Continue to Step 3.5.

**5. Save eval output** — include eval results and cost dashboard in the PR body (Step 8).

**Tier reference (for context — /ship always uses `full`):**
| Tier | When | Speed (cached) | Cost |
|------|------|----------------|------|
| `fast` (Haiku) | Dev iteration, smoke tests | ~5s (14x faster) | ~$0.07/run |
| `standard` (Sonnet) | Default dev, `bin/test-lane --eval` | ~17s (4x faster) | ~$0.37/run |
| `full` (Opus persona) | **`/ship` and pre-merge** | ~72s (baseline) | ~$1.27/run |

---

## Step 3.4: Test Coverage Audit

100% coverage is the goal — every untested path is a path where bugs hide and vibe coding becomes yolo coding. Evaluate what was ACTUALLY coded (from the diff), not what was planned.

**0. Before/after test count:**

```bash
# Count test files before any generation
find . -name '*.test.*' -o -name '*.spec.*' -o -name '*_test.*' -o -name '*_spec.*' | grep -v node_modules | wc -l
```

Store this number for the PR body.

**1. Trace every codepath changed** using `git diff origin/<base>...HEAD`:

Read every changed file. For each one, trace how data flows through the code — don't just list functions, actually follow the execution:

1. **Read the diff.** For each changed file, read the full file (not just the diff hunk) to understand context.
2. **Trace data flow.** Starting from each entry point (route handler, exported function, event listener, component render), follow the data through every branch:
   - Where does input come from? (request params, props, database, API call)
   - What transforms it? (validation, mapping, computation)
   - Where does it go? (database write, API response, rendered output, side effect)
   - What can go wrong at each step? (null/undefined, invalid input, network failure, empty collection)
3. **Diagram the execution.** For each changed file, draw an ASCII diagram showing:
   - Every function/method that was added or modified
   - Every conditional branch (if/else, switch, ternary, guard clause, early return)
   - Every error path (try/catch, rescue, error boundary, fallback)
   - Every call to another function (trace into it — does IT have untested branches?)
   - Every edge: what happens with null input? Empty array? Invalid type?

This is the critical step — you're building a map of every line of code that can execute differently based on input. Every branch in this diagram needs a test.

**2. Map user flows, interactions, and error states:**

Code coverage isn't enough — you need to cover how real users interact with the changed code. For each changed feature, think through:

- **User flows:** What sequence of actions does a user take that touches this code? Map the full journey (e.g., "user clicks 'Pay' → form validates → API call → success/failure screen"). Each step in the journey needs a test.
- **Interaction edge cases:** What happens when the user does something unexpected?
  - Double-click/rapid resubmit
  - Navigate away mid-operation (back button, close tab, click another link)
  - Submit with stale data (page sat open for 30 minutes, session expired)
  - Slow connection (API takes 10 seconds — what does the user see?)
  - Concurrent actions (two tabs, same form)
- **Error states the user can see:** For every error the code handles, what does the user actually experience?
  - Is there a clear error message or a silent failure?
  - Can the user recover (retry, go back, fix input) or are they stuck?
  - What happens with no network? With a 500 from the API? With invalid data from the server?
- **Empty/zero/boundary states:** What does the UI show with zero results? With 10,000 results? With a single character input? With maximum-length input?

Add these to your diagram alongside the code branches. A user flow with no test is just as much a gap as an untested if/else.

**3. Check each branch against existing tests:**

Go through your diagram branch by branch — both code paths AND user flows. For each one, search for a test that exercises it:
- Function `processPayment()` → look for `billing.test.ts`, `billing.spec.ts`, `test/billing_test.rb`
- An if/else → look for tests covering BOTH the true AND false path
- An error handler → look for a test that triggers that specific error condition
- A call to `helperFn()` that has its own branches → those branches need tests too
- A user flow → look for an integration or E2E test that walks through the journey
- An interaction edge case → look for a test that simulates the unexpected action

Quality scoring rubric:
- ★★★  Tests behavior with edge cases AND error paths
- ★★   Tests correct behavior, happy path only
- ★    Smoke test / existence check / trivial assertion (e.g., "it renders", "it doesn't throw")

**4. Output ASCII coverage diagram:**

Include BOTH code paths and user flows in the same diagram:

```
CODE PATH COVERAGE
===========================
[+] src/services/billing.ts
    │
    ├── processPayment()
    │   ├── [★★★ TESTED] Happy path + card declined + timeout — billing.test.ts:42
    │   ├── [GAP]         Network timeout — NO TEST
    │   └── [GAP]         Invalid currency — NO TEST
    │
    └── refundPayment()
        ├── [★★  TESTED] Full refund — billing.test.ts:89
        └── [★   TESTED] Partial refund (checks non-throw only) — billing.test.ts:101

USER FLOW COVERAGE
===========================
[+] Payment checkout flow
    │
    ├── [★★★ TESTED] Complete purchase — checkout.e2e.ts:15
    ├── [GAP]         Double-click submit — NO TEST
    ├── [GAP]         Navigate away during payment — NO TEST
    └── [★   TESTED] Form validation errors (checks render only) — checkout.test.ts:40

[+] Error states
    │
    ├── [★★  TESTED] Card declined message — billing.test.ts:58
    ├── [GAP]         Network timeout UX (what does user see?) — NO TEST
    └── [GAP]         Empty cart submission — NO TEST

─────────────────────────────────
COVERAGE: 5/12 paths tested (42%)
  Code paths: 3/5 (60%)
  User flows: 2/7 (29%)
QUALITY:  ★★★: 2  ★★: 2  ★: 1
GAPS: 7 paths need tests
─────────────────────────────────
```

**Fast path:** All paths covered → "Step 3.4: All new code paths have test coverage ✓" Continue.

**5. Generate tests for uncovered paths:**

If test framework detected (or bootstrapped in Step 2.5):
- Prioritize error handlers and edge cases first (happy paths are more likely already tested)
- Read 2-3 existing test files to match conventions exactly
- Generate unit tests. Mock all external dependencies (DB, API, Redis).
- Write tests that exercise the specific uncovered path with real assertions
- Run each test. Passes → commit as `test: coverage for {feature}`
- Fails → fix once. Still fails → revert, note gap in diagram.

Caps: 30 code paths max, 20 tests generated max (code + user flow combined), 2-min per-test exploration cap.

If no test framework AND user declined bootstrap → diagram only, no generation. Note: "Test generation skipped — no test framework configured."

**Diff is test-only changes:** Skip Step 3.4 entirely: "No new application code paths to audit."

**6. After-count and coverage summary:**

```bash
# Count test files after generation
find . -name '*.test.*' -o -name '*.spec.*' -o -name '*_test.*' -o -name '*_spec.*' | grep -v node_modules | wc -l
```

For PR body: `Tests: {before} → {after} (+{delta} new)`
Coverage line: `Test Coverage Audit: N new code paths. M covered (X%). K tests generated, J committed.`

---

## Step 3.5: Pre-Landing Review

Review the diff for structural issues that tests don't catch.

1. Read `.agents/skills/gstack/review/checklist.md`. If the file cannot be read, **STOP** and report the error.

2. Run `git diff origin/<base>` to get the full diff (scoped to feature changes against the freshly-fetched base branch).

3. Apply the review checklist in two passes:
   - **Pass 1 (CRITICAL):** SQL & Data Safety, LLM Output Trust Boundary
   - **Pass 2 (INFORMATIONAL):** All remaining categories

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

   Include any design findings alongside the code review findings. They follow the same Fix-First flow below.

4. **Classify each finding as AUTO-FIX or ASK** per the Fix-First Heuristic in
   checklist.md. Critical findings lean toward ASK; informational lean toward AUTO-FIX.

5. **Auto-fix all AUTO-FIX items.** Apply each fix. Output one line per fix:
   `[AUTO-FIXED] [file:line] Problem → what you did`

6. **If ASK items remain,** present them in ONE AskUserQuestion:
   - List each with number, severity, problem, recommended fix
   - Per-item options: A) Fix  B) Skip
   - Overall RECOMMENDATION
   - If 3 or fewer ASK items, you may use individual AskUserQuestion calls instead

7. **After all fixes (auto + user-approved):**
   - If ANY fixes were applied: commit fixed files by name (`git add <fixed-files> && git commit -m "fix: pre-landing review fixes"`), then **STOP** and tell the user to run `/ship` again to re-test.
   - If no fixes applied (all ASK items skipped, or no issues found): continue to Step 4.

8. Output summary: `Pre-Landing Review: N issues — M auto-fixed, K asked (J fixed, L skipped)`

   If no issues found: `Pre-Landing Review: No issues found.`

Save the review output — it goes into the PR body in Step 8.

---

## Step 3.75: Address Greptile review comments (if PR exists)

Read `.agents/skills/gstack/review/greptile-triage.md` and follow the fetch, filter, classify, and **escalation detection** steps.

**If no PR exists, `gh` fails, API returns an error, or there are zero Greptile comments:** Skip this step silently. Continue to Step 4.

**If Greptile comments are found:**

Include a Greptile summary in your output: `+ N Greptile comments (X valid, Y fixed, Z FP)`

Before replying to any comment, run the **Escalation Detection** algorithm from greptile-triage.md to determine whether to use Tier 1 (friendly) or Tier 2 (firm) reply templates.

For each classified comment:

**VALID & ACTIONABLE:** Use AskUserQuestion with:
- The comment (file:line or [top-level] + body summary + permalink URL)
- `RECOMMENDATION: Choose A because [one-line reason]`
- Options: A) Fix now, B) Acknowledge and ship anyway, C) It's a false positive
- If user chooses A: apply the fix, commit the fixed files (`git add <fixed-files> && git commit -m "fix: address Greptile review — <brief description>"`), reply using the **Fix reply template** from greptile-triage.md (include inline diff + explanation), and save to both per-project and global greptile-history (type: fix).
- If user chooses C: reply using the **False Positive reply template** from greptile-triage.md (include evidence + suggested re-rank), save to both per-project and global greptile-history (type: fp).

**VALID BUT ALREADY FIXED:** Reply using the **Already Fixed reply template** from greptile-triage.md — no AskUserQuestion needed:
- Include what was done and the fixing commit SHA
- Save to both per-project and global greptile-history (type: already-fixed)

**FALSE POSITIVE:** Use AskUserQuestion:
- Show the comment and why you think it's wrong (file:line or [top-level] + body summary + permalink URL)
- Options:
  - A) Reply to Greptile explaining the false positive (recommended if clearly wrong)
  - B) Fix it anyway (if trivial)
  - C) Ignore silently
- If user chooses A: reply using the **False Positive reply template** from greptile-triage.md (include evidence + suggested re-rank), save to both per-project and global greptile-history (type: fp)

**SUPPRESSED:** Skip silently — these are known false positives from previous triage.

**After all comments are resolved:** If any fixes were applied, the tests from Step 3 are now stale. **Re-run tests** (Step 3) before continuing to Step 4. If no fixes were applied, continue to Step 4.

---

## Step 3.8: Codex second opinion (optional)

Check if the Codex CLI is available:

```bash
which codex 2>/dev/null && echo "CODEX_AVAILABLE" || echo "CODEX_NOT_AVAILABLE"
```

If Codex is available, use AskUserQuestion:

```
Pre-landing review complete. Want an independent Codex (OpenAI) review before shipping?

A) Run Codex code review — independent diff review with pass/fail gate
B) Run Codex adversarial challenge — try to break this code
C) Skip — ship without Codex review
```

If the user chooses A or B:

**For code review (A):** Run `codex review --base <base>` with a 5-minute timeout.
Present the full output verbatim under a `CODEX SAYS:` header. Check for `[P1]` markers
to determine pass/fail gate. Persist the result:

```bash
~/.codex/skills/gstack/bin/gstack-review-log '{"skill":"codex-review","timestamp":"TIMESTAMP","status":"STATUS","gate":"GATE"}'
```

If GATE is FAIL, use AskUserQuestion: "Codex found critical issues. Ship anyway?"
If the user says no, stop. If yes, continue to Step 4.

**For adversarial (B):** Run codex exec with the adversarial prompt (see /codex skill).
Present findings. This is informational — does not block shipping.

If Codex is not available, skip silently. Continue to Step 4.

---

## Step 4: Version bump (auto-decide)

1. Read the current `VERSION` file (4-digit format: `MAJOR.MINOR.PATCH.MICRO`)

2. **Auto-decide the bump level based on the diff:**
   - Count lines changed (`git diff origin/<base>...HEAD --stat | tail -1`)
   - **MICRO** (4th digit): < 50 lines changed, trivial tweaks, typos, config
   - **PATCH** (3rd digit): 50+ lines changed, bug fixes, small-medium features
   - **MINOR** (2nd digit): **ASK the user** — only for major features or significant architectural changes
   - **MAJOR** (1st digit): **ASK the user** — only for milestones or breaking changes

3. Compute the new version:
   - Bumping a digit resets all digits to its right to 0
   - Example: `0.19.1.0` + PATCH → `0.19.2.0`

4. Write the new version to the `VERSION` file.

---

## Step 5: CHANGELOG (auto-generate)

1. Read `CHANGELOG.md` header to know the format.

2. Auto-generate the entry from **ALL commits on the branch** (not just recent ones):
   - Use `git log <base>..HEAD --oneline` to see every commit being shipped
   - Use `git diff <base>...HEAD` to see the full diff against the base branch
   - The CHANGELOG entry must be comprehensive of ALL changes going into the PR
   - If existing CHANGELOG entries on the branch already cover some commits, replace them with one unified entry for the new version
   - Categorize changes into applicable sections:
     - `### Added` — new features
     - `### Changed` — changes to existing functionality
     - `### Fixed` — bug fixes
     - `### Removed` — removed features
   - Write concise, descriptive bullet points
   - Insert after the file header (line 5), dated today
   - Format: `## [X.Y.Z.W] - YYYY-MM-DD`

**Do NOT ask the user to describe changes.** Infer from the diff and commit history.

---

## Step 5.5: TODOS.md (auto-update)

Cross-reference the project's TODOS.md against the changes being shipped. Mark completed items automatically; prompt only if the file is missing or disorganized.

Read `.agents/skills/gstack/review/TODOS-format.md` for the canonical format reference.

**1. Check if TODOS.md exists** in the repository root.

**If TODOS.md does not exist:** Use AskUserQuestion:
- Message: "GStack recommends maintaining a TODOS.md organized by skill/component, then priority (P0 at top through P4, then Completed at bottom). See TODOS-format.md for the full format. Would you like to create one?"
- Options: A) Create it now, B) Skip for now
- If A: Create `TODOS.md` with a skeleton (# TODOS heading + ## Completed section). Continue to step 3.
- If B: Skip the rest of Step 5.5. Continue to Step 6.

**2. Check structure and organization:**

Read TODOS.md and verify it follows the recommended structure:
- Items grouped under `## <Skill/Component>` headings
- Each item has `**Priority:**` field with P0-P4 value
- A `## Completed` section at the bottom

**If disorganized** (missing priority fields, no component groupings, no Completed section): Use AskUserQuestion:
- Message: "TODOS.md doesn't follow the recommended structure (skill/component groupings, P0-P4 priority, Completed section). Would you like to reorganize it?"
- Options: A) Reorganize now (recommended), B) Leave as-is
- If A: Reorganize in-place following TODOS-format.md. Preserve all content — only restructure, never delete items.
- If B: Continue to step 3 without restructuring.

**3. Detect completed TODOs:**

This step is fully automatic — no user interaction.

Use the diff and commit history already gathered in earlier steps:
- `git diff <base>...HEAD` (full diff against the base branch)
- `git log <base>..HEAD --oneline` (all commits being shipped)

For each TODO item, check if the changes in this PR complete it by:
- Matching commit messages against the TODO title and description
- Checking if files referenced in the TODO appear in the diff
- Checking if the TODO's described work matches the functional changes

**Be conservative:** Only mark a TODO as completed if there is clear evidence in the diff. If uncertain, leave it alone.

**4. Move completed items** to the `## Completed` section at the bottom. Append: `**Completed:** vX.Y.Z (YYYY-MM-DD)`

**5. Output summary:**
- `TODOS.md: N items marked complete (item1, item2, ...). M items remaining.`
- Or: `TODOS.md: No completed items detected. M items remaining.`
- Or: `TODOS.md: Created.` / `TODOS.md: Reorganized.`

**6. Defensive:** If TODOS.md cannot be written (permission error, disk full), warn the user and continue. Never stop the ship workflow for a TODOS failure.

Save this summary — it goes into the PR body in Step 8.

---

## Step 6: Commit (bisectable chunks)

**Goal:** Create small, logical commits that work well with `git bisect` and help LLMs understand what changed.

1. Analyze the diff and group changes into logical commits. Each commit should represent **one coherent change** — not one file, but one logical unit.

2. **Commit ordering** (earlier commits first):
   - **Infrastructure:** migrations, config changes, route additions
   - **Models & services:** new models, services, concerns (with their tests)
   - **Controllers & views:** controllers, views, JS/React components (with their tests)
   - **VERSION + CHANGELOG + TODOS.md:** always in the final commit

3. **Rules for splitting:**
   - A model and its test file go in the same commit
   - A service and its test file go in the same commit
   - A controller, its views, and its test go in the same commit
   - Migrations are their own commit (or grouped with the model they support)
   - Config/route changes can group with the feature they enable
   - If the total diff is small (< 50 lines across < 4 files), a single commit is fine

4. **Each commit must be independently valid** — no broken imports, no references to code that doesn't exist yet. Order commits so dependencies come first.

5. Compose each commit message:
   - First line: `<type>: <summary>` (type = feat/fix/chore/refactor/docs)
   - Body: brief description of what this commit contains
   - Only the **final commit** (VERSION + CHANGELOG) gets the version tag and co-author trailer:

```bash
git commit -m "$(cat <<'EOF'
chore: bump version and changelog (vX.Y.Z.W)

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Step 6.5: Verification Gate

**IRON LAW: NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE.**

Before pushing, re-verify if code changed during Steps 4-6:

1. **Test verification:** If ANY code changed after Step 3's test run (fixes from review findings, CHANGELOG edits don't count), re-run the test suite. Paste fresh output. Stale output from Step 3 is NOT acceptable.

2. **Build verification:** If the project has a build step, run it. Paste output.

3. **Rationalization prevention:**
   - "Should work now" → RUN IT.
   - "I'm confident" → Confidence is not evidence.
   - "I already tested earlier" → Code changed since then. Test again.
   - "It's a trivial change" → Trivial changes break production.

**If tests fail here:** STOP. Do not push. Fix the issue and return to Step 3.

Claiming work is complete without verification is dishonesty, not efficiency.

---

## Step 7: Push

Push to the remote with upstream tracking:

```bash
git push -u origin <branch-name>
```

---

## Step 8: Create PR

Create a pull request using `gh`:

```bash
gh pr create --base <base> --title "<type>: <summary>" --body "$(cat <<'EOF'
## Summary
<bullet points from CHANGELOG>

## Test Coverage
<coverage diagram from Step 3.4, or "All new code paths have test coverage.">
<If Step 3.4 ran: "Tests: {before} → {after} (+{delta} new)">

## Pre-Landing Review
<findings from Step 3.5 code review, or "No issues found.">

## Design Review
<If design review ran: "Design Review (lite): N findings — M auto-fixed, K skipped. AI Slop: clean/N issues.">
<If no frontend files changed: "No frontend files changed — design review skipped.">

## Eval Results
<If evals ran: suite names, pass/fail counts, cost dashboard summary. If skipped: "No prompt-related files changed — evals skipped.">

## Greptile Review
<If Greptile comments were found: bullet list with [FIXED] / [FALSE POSITIVE] / [ALREADY FIXED] tag + one-line summary per comment>
<If no Greptile comments found: "No Greptile comments.">
<If no PR existed during Step 3.75: omit this section entirely>

## TODOS
<If items marked complete: bullet list of completed items with version>
<If no items completed: "No TODO items completed in this PR.">
<If TODOS.md created or reorganized: note that>
<If TODOS.md doesn't exist and user skipped: omit this section>

## Test plan
- [x] All Rails tests pass (N runs, 0 failures)
- [x] All Vitest tests pass (N tests)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

**Output the PR URL** — then proceed to Step 8.5.

---

## Step 8.5: Auto-invoke /document-release

After the PR is created, automatically sync project documentation. Read the
`document-release/SKILL.md` skill file (adjacent to this skill's directory) and
execute its full workflow:

1. Read the `/document-release` skill: `cat ${CLAUDE_SKILL_DIR}/../document-release/SKILL.md`
2. Follow its instructions — it reads all .md files in the project, cross-references
   the diff, and updates anything that drifted (README, ARCHITECTURE, CONTRIBUTING,
   CLAUDE.md, TODOS, etc.)
3. If any docs were updated, commit the changes and push to the same branch:
   ```bash
   git add -A && git commit -m "docs: sync documentation with shipped changes" && git push
   ```
4. If no docs needed updating, say "Documentation is current — no updates needed."

This step is automatic. Do not ask the user for confirmation. The goal is zero-friction
doc updates — the user runs `/ship` and documentation stays current without a separate command.

---

## Important Rules

- **Never skip tests.** If tests fail, stop.
- **Never skip the pre-landing review.** If checklist.md is unreadable, stop.
- **Never force push.** Use regular `git push` only.
- **Never ask for confirmation** except for MINOR/MAJOR version bumps and pre-landing review ASK items (batched into at most one AskUserQuestion).
- **Always use the 4-digit version format** from the VERSION file.
- **Date format in CHANGELOG:** `YYYY-MM-DD`
- **Split commits for bisectability** — each commit = one logical change.
- **TODOS.md completion detection must be conservative.** Only mark items as completed when the diff clearly shows the work is done.
- **Use Greptile reply templates from greptile-triage.md.** Every reply includes evidence (inline diff, code references, re-rank suggestion). Never post vague replies.
- **Never push without fresh verification evidence.** If code changed after Step 3 tests, re-run before pushing.
- **Step 3.4 generates coverage tests.** They must pass before committing. Never commit failing tests.
- **The goal is: user says `/ship`, next thing they see is the review + PR URL + auto-synced docs.**
