# MASTER_FIX_PLAN Template

Used by Step 6 to generate the primary deliverable. Copy and fill in.

---

# {Project Name} 终极修复计划

> **生成时间**：{date}
> **来源**：{review method — e.g., "multi-role-review skill（8 角色并行审查）" 或 "两轮审查合并"}
> **覆盖**：{total} 个发现，{num_phases} 个 Phase
> **使用方式**：从 Phase A 往下逐个执行，每完成一个 Task 打 [x]，每完成一个 Phase 提交一次 commit
> **约束**：不要一次性重写覆盖文件，用精确替换；改完跑类型检查和语法验证

---

## Phase A: {阻断性问题}（上线前必须完成）

### A1. {Task title} 🔴

**文件**：`path/to/file.ext`（第 X 行）

**当前问题**（{severity}，{roles} 标记）：
```{lang}
{current code snippet}
```

**期望**：
1. {Specific change}
2. {Specific change}

**验收**：{How to verify}

---

## Phase B: {安全+健壮性}

### B1. {Task title} 🟡

{Same structure}

---

## Phase C: {架构改进}

### C1. {Task title} 🟢

---

## Phase D: {前端+UI精修}

### D1. {Task title} 🎨

{If detailed CSS exists elsewhere, reference it: "详见 OPTIMIZATION_TASKS.md D1 完整 CSS"}

---

## Phase E: {测试+CI/CD}

### E1. {Task title} 🟢

---

## Phase F: {部署运维加固}

### F1. {Task title} 🟢

---

## 验收 Checklist（全部完成后执行）

### 后端
- [ ] `{python import check}` — 无 import 错误
- [ ] `{pytest command}` — 全部通过
- [ ] `{grep for mock/placeholder}` — 仅注释中

### 前端
- [ ] `{vue-tsc --noEmit}` — 0 errors
- [ ] `{grep for old colors}` — 返回空
- [ ] {Visual checks}

### 部署
- [ ] `{docker compose up}` — 一条命令启动
- [ ] `{container health check}`
- [ ] {Non-root verification}

### 全链路
- [ ] {Core user flow walkthrough}

---

## 附录：完整发现汇总表

| # | 发现 | 来源轮次 | 严重度 | 对应 Task |
|---|------|---------|--------|-----------|
| 1 | {finding} | {round} | 🔴/🟡/🟢/🎨 | {Task ID} |
| 2 | {finding} | {round} | {severity} | {Task ID} |
| ... | ... | ... | ... | ... |

<!--
TEMPLATE NOTES:
- Phase naming convention: A=blocking, B=security/robustness, C=architecture,
  D=frontend/UI, E=testing/CI, F=ops/deployment
- Severity emojis: 🔴 Critical, 🟡 Warning, 🟢 Suggestion, 🎨 Visual
- Every finding from the review MUST appear in the mapping table
- If re-reviewing: mark which findings are "new vs previous review"
- Cross-reference existing plans (OPTIMIZATION_TASKS.md etc.) instead of duplicating
- Verification commands must be copy-pasteable into terminal
-->
