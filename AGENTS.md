# gstack：AI 工程工作流

gstack 是一组 `SKILL.md` 文件，为 AI 代理提供结构化的软件开发分工。每个技能都对应一个明确角色，例如 CEO 评审、工程经理、设计师、QA 负责人、发布工程师、调试专家等。

## 可用技能

技能文件位于 `.agents/skills/`。可直接用名称调用，例如 `/office-hours`。

| 技能 | 用途 |
|------|------|
| `/office-hours` | 建议从这里开始。在写代码前先重构你的产品思路。 |
| `/plan-ceo-review` | CEO 级别的方案评审，找出需求里真正的 10 星产品。 |
| `/plan-eng-review` | 锁定架构、数据流、边界情况和测试策略。 |
| `/plan-design-review` | 按 0-10 分评估设计维度，并说明满分长什么样。 |
| `/design-consultation` | 从零建立完整设计系统。 |
| `/review` | 合并前 PR 审查，找出 CI 过了但线上会炸的问题。 |
| `/debug` | 系统化根因排查；不先调查，不允许直接修。 |
| `/design-review` | 设计审计并进入修复闭环，要求原子提交。 |
| `/qa` | 打开真实浏览器，找 bug、修 bug、再验证。 |
| `/qa-only` | 与 `/qa` 相同，但只出报告，不改代码。 |
| `/ship` | 跑测试、做 review、推送并创建 PR，一条命令完成。 |
| `/document-release` | 将所有文档同步到刚刚发布的实际状态。 |
| `/retro` | 每周复盘，按人拆分贡献和交付节奏。 |
| `/browse` | 无头浏览器能力，真实 Chromium、真实点击，单命令约 100ms。 |
| `/setup-browser-cookies` | 从真实浏览器导入 cookie，用于登录态测试。 |
| `/careful` | 在破坏性命令前提醒，例如 `rm -rf`、`DROP TABLE`、强推。 |
| `/freeze` | 将编辑范围锁定到一个目录，是真正阻止而不是口头提醒。 |
| `/guard` | 同时启用 `careful` 和 `freeze`。 |
| `/unfreeze` | 移除目录编辑限制。 |
| `/gstack-upgrade` | 将 gstack 升级到最新版本。 |

## 构建命令

```bash
bun install              # 安装依赖
bun test                 # 运行测试（免费，<5 秒）
bun run build            # 生成文档并编译二进制
bun run gen:skill-docs   # 由模板重新生成 SKILL.md
bun run skill:check      # 检查所有技能的健康状态
```

## 关键约定

- `SKILL.md` 文件是由 `.tmpl` 模板**生成**的。请修改模板，不要直接修改生成产物。
- 用 `bun run gen:skill-docs --host codex` 生成面向 Codex 的输出。
- `browse` 二进制负责无头浏览器访问；在技能中使用 `$B <command>`。
- 安全类技能（`careful`、`freeze`、`guard`）采用内联提示方式；执行破坏性操作前始终要确认。
