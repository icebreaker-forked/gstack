# QA 报告：{APP_NAME}

| 字段 | 值 |
|------|----|
| **日期** | {DATE} |
| **URL** | {URL} |
| **分支** | {BRANCH} |
| **提交** | {COMMIT_SHA} ({COMMIT_DATE}) |
| **PR** | {PR_NUMBER} ({PR_URL}) 或 `—` |
| **层级** | Quick / Standard / Exhaustive |
| **范围** | {SCOPE or "Full app"} |
| **耗时** | {DURATION} |
| **访问页面数** | {COUNT} |
| **截图数** | {COUNT} |
| **框架** | {DETECTED or "Unknown"} |
| **索引** | [所有 QA 运行记录](./index.md) |

## 健康评分：{SCORE}/100

| 类别 | 分数 |
|------|------|
| Console | {0-100} |
| Links | {0-100} |
| Visual | {0-100} |
| Functional | {0-100} |
| UX | {0-100} |
| Performance | {0-100} |
| Accessibility | {0-100} |

## 优先修复的 3 件事

1. **{ISSUE-NNN}: {title}** — {one-line description}
2. **{ISSUE-NNN}: {title}** — {one-line description}
3. **{ISSUE-NNN}: {title}** — {one-line description}

## Console 健康状况

| 错误 | 次数 | 首次出现位置 |
|------|------|--------------|
| {error message} | {N} | {URL} |

## 汇总

| 严重级别 | 数量 |
|----------|------|
| Critical | 0 |
| High | 0 |
| Medium | 0 |
| Low | 0 |
| **总计** | **0** |

## 问题列表

### ISSUE-001: {Short title}

| 字段 | 值 |
|------|----|
| **严重级别** | critical / high / medium / low |
| **分类** | visual / functional / ux / content / performance / console / accessibility |
| **URL** | {page URL} |

**问题描述：** {What is wrong, expected vs actual.}

**复现步骤：**

1. 打开 {URL}
   ![Step 1](screenshots/issue-001-step-1.png)
2. {Action}
   ![Step 2](screenshots/issue-001-step-2.png)
3. **观察到：** {what goes wrong}
   ![Result](screenshots/issue-001-result.png)

---

## 已应用修复（如适用）

| 问题 | 修复状态 | 提交 | 修改文件 |
|------|----------|------|----------|
| ISSUE-NNN | verified / best-effort / reverted / deferred | {SHA} | {files} |

### 修复前后证据

#### ISSUE-NNN: {title}
**修复前：** ![Before](screenshots/issue-NNN-before.png)
**修复后：** ![After](screenshots/issue-NNN-after.png)

---

## 回归测试

| 问题 | 测试文件 | 状态 | 说明 |
|------|----------|------|------|
| ISSUE-NNN | path/to/test | committed / deferred / skipped | description |

### 延后补充的测试

#### ISSUE-NNN: {title}
**前置条件：** {setup state that triggers the bug}  
**操作：** {what the user does}  
**期望：** {correct behavior}  
**为何延后：** {reason}

---

## 发版就绪度

| 指标 | 值 |
|------|----|
| 健康分 | {before} → {after} ({delta}) |
| 发现问题 | N |
| 已修复 | N（verified: X，best-effort: Y，reverted: Z） |
| 延后项 | N |

**PR 摘要：** `QA found N issues, fixed M, health score X → Y.`

---

## 回归对比（如适用）

| 指标 | 基线 | 当前 | Delta |
|------|------|------|-------|
| 健康分 | {N} | {N} | {+/-N} |
| 问题数 | {N} | {N} | {+/-N} |

**相对基线已修复：** {list}  
**相对基线新增：** {list}
