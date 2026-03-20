# 更新日志

> 本文件为中文版摘要，聚焦用户可感知变化与近阶段版本演进。

## [0.9.0.1] - 2026-03-19

### 变更

- **遥测默认建议社区模式。** 首次提示会直接问你是否愿意“帮助 gstack 变得更好”。如果拒绝，还会提供匿名模式作为第二选择。

### 修复

- **Plan mode 下的 review 日志与遥测恢复正常。** 之前在 `/plan-ceo-review`、`/plan-eng-review`、`/plan-design-review` 中完成的评审结果不会可靠落盘，导致面板显示过期数据；现在已修复。

## [0.9.0] - 2026-03-19

### 重点

- **gstack 现在可运行于 Codex、Gemini CLI、Cursor 等支持 `SKILL.md` 标准的代理环境。**
- **一次安装，支持多宿主。** Claude 仍读取 `.claude/skills/`，其他宿主读取 `.agents/skills/`。
- **Codex 输出做了宿主适配。** 例如路径从 `~/.claude/` 改写到 `~/.codex/`，frontmatter 也简化为更适合 Codex 的格式。
- **CI 同时校验 Claude 与 Codex 两套生成结果。**

## [0.8.6] - 2026-03-19

### 新增

- **本地使用分析面板。** 可查看你最常用哪些技能、耗时如何、成功率如何。
- **社区遥测。** 首次运行可选择加入匿名或社区级使用统计，不会上传代码和文件路径。
- **社区健康面板。** 可以查看整个 gstack 社区的技能热度、崩溃聚类和版本分布。
- **升级漏斗与安装基数跟踪。**
- **`/retro` 集成 gstack 使用数据。**
- **按会话隔离的 pending marker。** 修复并发会话间的竞态。

## [0.8.5] - 2026-03-19

### 修复

- **`/retro` 现在按完整自然日统计。**
- **review 日志不再被带 `/` 的分支名破坏。**
- **所有技能模板进一步去平台耦合。**
- **`/ship` 从 `CLAUDE.md` 读取测试命令，而不是写死命令。**

### 新增

- 在 `CLAUDE.md` 中正式写入“平台无关设计原则”
- 增加面向 `/ship` 的 `## Testing` 约定

## [0.8.4] - 2026-03-19

### 新增

- **`/ship` 现在会自动调用 `/document-release`，同步 README、ARCHITECTURE、CONTRIBUTING、CLAUDE 等文档。**
- **文档覆盖更多技能。** 包括 `/codex`、`/careful`、`/freeze`、`/guard`、`/unfreeze`、`/gstack-upgrade`。
- **`$B handoff` / `$B resume` 的浏览器交接能力在各主要文档中完成说明。**
- **根技能的主动建议逻辑现在认识全部主要技能。**

## [0.8.3] - 2026-03-19

### 新增

- **Plan review 会建议下一步该运行什么。**
- **Review 会记录运行时对应的 commit，并提示结果是否已过期。**
- **`skip_eng_review` 在所有推荐链路中得到一致尊重。**
- **轻量设计检查也开始记录 commit，支持 stale 判断。**

### 修复

- browse 阻止跳转到危险 URL，如 `file://`、`javascript:`、`data:` 和云 metadata 端点
- `./setup` 在缺少 `bun` 时提供清晰错误提示
- `/debug` 重命名为 `/investigate`，避免与 Claude 内建命令冲突
- 去除技能模板中使用 `eval $(...)` 的注入面
- 新增大量安全测试

## [0.8.2] - 2026-03-19

### 新增

- **无头浏览器卡住时，可交接给可见 Chrome。**
- **连续失败 3 次后会主动建议 handoff。**
- **为 handoff / resume 增加完整测试。**

### 变更

- `recreateContext()` 复用状态保存 / 恢复逻辑
- `browser.close()` 增加超时，避免 macOS 下卡死

## [0.8.1] - 2026-03-19

### 修复

- **`/qa` 即使在后端改动场景下也会实际打开浏览器。** 如果 diff 无法推断出具体页面，就自动退化为 Quick smoke test。

## [0.8.0] - 2026-03-19

### 重点

- **新增 `/codex`：用 OpenAI Codex CLI 获得第二意见。**
- 支持 review、challenge 和自由咨询三种模式
- `/review` 与 `/codex review` 同时运行后，可获得跨模型分析
- `/ship`、`/plan-eng-review` 等流程开始整合 Codex 结果
- **同时加入主动技能建议能力**

## [0.7.x] - 2026-03-18 ~ 2026-03-19

这一阶段的核心变化：

- `/qa` 和 `/design-review` 能正确处理脏工作树
- **新增 `/careful`、`/freeze`、`/guard`、`/unfreeze`**
- `/retro` 的时间窗口和时区处理更加准确
- **gstack 开始根据自然语言与工作阶段主动推荐技能**
- **新增 `/office-hours`**
- `/debug` / `/investigate` 建立“先找根因再修”的工作流

## [0.6.x] - 2026-03-17

这一阶段的核心变化：

- **`/plan-design-review` 变成交互式设计评审，而不只是生成报告**
- **`/review` 与 `/ship` 自动触发前端设计检查**
- **plan reviews 引入一系列高层认知模式**
- **E2E 与 LLM-judge 测试开始按 diff 选择性运行**
- **Completeness Principle（Boil the Lake）成为全局原则**

## 说明

- 旧版本的英文细节说明已在本次中文化中收敛为摘要
- 若后续需要保留“逐版本完整条目”，建议新增 `CHANGELOG.en.md` 或将历史版本拆分归档
