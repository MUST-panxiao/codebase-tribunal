# OPTIMIZATION_TASKS Template

This template is used by Step 6 of the multi-role-review skill to generate an
actionable task list that can be handed off to Claude Code or another developer.

Copy this structure and fill in project-specific findings.

---

# {Project Name} 优化清单

> **生成时间**：{date}
> **审查来源**：多角色（{list roles}）深度代码审查
> **使用方式**：从上往下逐个 Task 执行，每完成一个打 [x]，改完一个 Phase 提交一次 commit
> **约束**：不要一次性重写覆盖文件，用精确替换；改完跑类型检查和语法验证

---

## Phase A: {阻断性问题}（上线前必须修复）

### A1. {Task title} 🔴

**文件**：`path/to/file.ext`（第 X 行）

**当前问题**：
```{lang}
{current code snippet}
```

**期望**：
- {Specific change 1}
- {Specific change 2}

**验收**：{How to verify the fix is correct}

---

### A2. {Task title} 🔴

{Same structure}

---

## Phase B: {高优先级改进}

### B1. {Task title} 🟡

{Same structure}

---

## Phase C: {可维护性改进}

### C1. {Task title} 🟢

{Same structure}

---

## Phase D: {视觉/体验精修}

### D1. {Task title} 🎨

{Same structure}

---

## 验收 Checklist（全部完成后执行）

- [ ] `{type check command}` — 0 errors
- [ ] `{lint command}` — 0 errors
- [ ] `{build command}` — build succeeds
- [ ] `{test command}` — all tests pass
- [ ] `{deploy command}` — deployment works
- [ ] {Domain-specific check}

---

## 附录：审查中发现但不在本清单中的观察

1. **{Observation}**：{description}. {Recommendation if addressed later}.
2. **{Observation}**：{description}.

---

<!-- Template Notes:

FORMATTING RULES:
1. Each task MUST have: file path, line number (if applicable), current code, expected change, verification method
2. Use exact file paths from the project — no generic placeholders
3. Include actual code snippets so Claude Code doesn't have to guess
4. Severity emojis: 🔴 Critical, 🟡 Warning, 🟢 Suggestion, 🎨 Visual
5. Phase naming: A = blocking, B = high-value, C = maintenance, D = polish

TASK GRANULARITY:
- One task = one file or one tightly-coupled set of changes
- Don't bundle unrelated changes into one task
- If a fix touches 3+ files, list all file paths
- If a fix requires a new file (e.g., Dockerfile), say "CREATE" not just "改"

VERIFICATION METHODS:
- TypeScript: `npx vue-tsc --noEmit` or `npx tsc --noEmit`
- Python: `python -c "from app.main import app; print('OK')"`
- Build: `npm run build` or `docker compose up -d --build`
- Tests: `npm test` or `python -m pytest`
- Grep checks: `grep -rn "pattern" path/ --include="*.ext"`
-->
