# ⚖️ Codebase Tribunal

**Your codebase is on trial. Eight AI experts are the judges.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
![Agent Skill](https://img.shields.io/badge/type-Agent%20Skill%20·%20SKILL.md-8b5cf6.svg)
![Field-tested](https://img.shields.io/badge/field--tested-8%20real%20projects-green.svg)
![Precedents](https://img.shields.io/badge/finding%20patterns-44-f97316.svg)

> In this court, your project is **guilty until proven shippable**.

One command convenes a full review tribunal — Project Manager, Product Manager, Architect, Backend Engineer, Frontend/UI Engineer, Security Engineer, QA Engineer, DevOps Engineer. Each judge inspects your project through their own professional lens, **independently, without seeing each other's notes**. Then the court hands down:

- ⚖️ **A quantified verdict** — per-role scores, plus an overall score under *sentencing guidelines*: any Critical finding caps the project at 7/10; three Criticals cap it at 5. A beautiful average can't save you.
- 📋 **A rap sheet with evidence** — every finding cites `path:line`, severity (🔴 Critical / 🟡 Warning / 🟢 Suggestion), and *why it matters*. Dynamic findings are verified by actually running your app locally (`实测` = tested, not guessed). No "the code could be better" hand-waving.
- 🔧 **A court-ordered fix plan** (`MASTER_FIX_PLAN.md`) — every task carries the exact file, line number, current code snippet, expected change, and a copy-pasteable verification command. Hand it to Claude Code / Cursor / your coding agent and walk away.

> **Field-tested, not vibes.** On a real flu-prediction platform, a manual 5-role sequential pass found 16 issues. The same project before this tribunal: **31 issues** — including unauthenticated delete endpoints, path traversal, and a CVE in the auth library that nobody had noticed. Parallel judges with independent context windows surface what a single-pass review misses. All 44 finding patterns in this repo's precedent library were found in the wild, not invented.

---

## Meet the judges

| Judge | Asks | Catches |
|---|---|---|
| 🎯 **Project Manager** | "Is this shippable? What's blocking?" | Mock data in deliverables, missing deployment artifacts, unpinned deps |
| 📋 **Product Manager** | "Does this solve the user's real problem?" | Dead-end user flows, missing feedback loops, placeholder values in user-facing output |
| 🏗️ **Architect** | "Will this structure hold up?" | God files, wrong dependency direction, circular coupling |
| ⚙️ **Backend Engineer** | "What breaks at 3am?" | Zero-auth endpoints, silent exception swallowing, sync I/O inside `async def` |
| 🎨 **Frontend/UI Engineer** | "Would I be proud to show this?" | No loading/error/empty states, 100+ hardcoded hex colors, memory-leaking resize handlers |
| 🛡️ **Security Engineer** | "Where is the trust boundary that wasn't checked?" | Weak JWT secrets, SQL string concatenation, CSRF not initialized, secrets committed to git |
| 🧪 **QA Engineer** | "Is there evidence this works, or just developer optimism?" | Zero test suites, stale tests, "tests" that can't even run in CI |
| 🚀 **DevOps Engineer** | "How does this reach production? How would I know it failed?" | Root-user Dockerfiles, non-existent base-image versions, placeholder systemd paths |

The tribunal adapts to the project: data pipelines get a Data Engineer instead of Frontend; ESP32 firmware gets a Firmware Engineer + Companion App Engineer (blocking `delay()` in `loop()`, WiFi passwords baked into public binaries, BLE MTU traps...).

## How the trial runs

1. **Bailiff work** — stack detection: manifest, framework, lockfile, package manager, deployment artifacts. Never assume the stack.
2. **Jury selection** — roles chosen by stack and project maturity (prototype → 3 judges; production platform → full 8).
3. **Independent deliberation** — each judge reads the code through their own lens (ideally as parallel sub-agents with separate context windows) and returns a signed Output Contract: score + findings + health assessment.
4. **Evidence testing** — Critical security claims are verified by actually running the service locally and probing: no-credential request → 200? confirmed. 401? finding refuted as false positive.
5. **Synthesis** — findings de-duplicated across judges (`[roles: backend, security]` = higher confidence), conflicts surfaced openly with a tie-break, gaps noted.
6. **The verdict** — overall score computed mechanically, caps applied, arithmetic shown:
   `总分 5.0 = min(均值 6.1, 3+ Critical 封顶 5)`
7. **Sentencing** — `REVIEW_REPORT.md` (the audit trail) + `MASTER_FIX_PLAN.md` (the executable sentence), cross-mapped so every finding traces to a task.

### The sentencing guidelines

| Situation | Score cap |
|---|:---:|
| Any Critical finding stands | ≤ 7 |
| 3+ Critical findings | ≤ 5 |
| Any single judge scores ≤ 3 | ≤ 6 |
| Core business logic absent (scaffold only) | ≤ 4 |

Why caps exist — a real case: a production platform averaged 5.9 across roles, but carried 60 completely unauthenticated endpoints. The first review shipped the 5.9 anyway. The re-review recomputed with caps: **5.0**. One severely deficient area is a systemic risk; the average must not launder it.

## Three modes

| Mode | Command vibe | What happens |
|---|---|---|
| **Full Trial** (default) | "review this project" / "审查这个项目" | Whole-project audit, all judges |
| **Spot Check** | "review this PR/diff" | Same lenses, scoped to changes |
| **Appeal** (delta re-review) | "re-review the project" | The parole hearing 🎫 — every previous finding re-verified line-by-line (✅ 已修 / ⚠️ 部分 / ❌ 未修 / ➖ 已失效), N new vs M resolved counted, abandoned-feature residue hunted down |

## The precedent library

`references/findings-patterns.md` — 44 patterns distilled from 8 real project reviews, each with the grep command that finds it and the fix pattern that resolves it. A taste:

- **#1 Zero-auth business endpoints** — appeared in *every* reviewed platform. `grep -rn "@router.post\|@router.delete" | grep -v get_current_user`
- **#28 Non-existent dependency versions** — `pandas==3.0.2` doesn't exist; Docker build fails 100%. (Hallucinated versions from AI coding assistants are now a standard check.)
- **#33 Written-but-never-wired modules** — a fully implemented evaluator that nothing imports, while the README claims the feature. Dead code disguised as features.
- **#36 NaN silently treated as zero** — pandas `.sum(skipna=True)` on surveillance data where NaN means "not tested", not "zero". Corrupts every downstream number without an error message.
- **#43 Generated artifacts landing in a public static dir** — each half looks fine in isolation; only the overlap is the vulnerability.

## Install

Works with any agent that reads `SKILL.md` (Claude Code, ZCode, and compatible runners). Best experience with sub-agent support (parallel judges); falls back to sequential personas automatically.

```bash
# Claude Code (user-level)
git clone https://github.com/MUST-panxiao/codebase-tribunal.git ~/.claude/skills/codebase-tribunal

# or generic agents convention
git clone https://github.com/MUST-panxiao/codebase-tribunal.git ~/.agents/skills/codebase-tribunal

# project-level (any agent): clone into .claude/skills/ or .github/skills/ in your repo
```

Then open your project and say **"review this project"** or **“审查这个项目”**. That's it — the tribunal self-organizes.

## See a verdict

👉 [examples/sample-verdict.md](examples/sample-verdict.md) — a condensed, anonymized real-format verdict (score table, cap computation, findings with evidence, fix-plan excerpt).

## 中文说明

**代码审判庭**：一句话把你的项目送上法庭。5–8 位 AI 专家（项目经理、产品、架构、后端、前端、安全、测试、运维）**各自独立**审查同一个项目，互不通气，避免视角趋同；随后交叉验证、去重合议，产出：

- 量化评分（含封顶规则：存在任一 Critical 直接封顶 7 分）
- 每条发现都带 `文件:行号` 证据和影响说明，安全类发现会**本地起服务实测**
- `MASTER_FIX_PLAN.md` 修复计划——精确到行号、附当前代码和验收命令，可直接交给 Claude Code 执行

支持全量审查、PR 增量审查、以及**复审模式**（旧问题逐项核对：已修/部分/未修/失效，并列出本轮新增）。内置 44 条真实项目沉淀的发现模式（零鉴权接口、路径穿越、NaN 当 0、幻觉依赖版本、写好但从未接线的模块……）。审查可用中文或英文进行。

## Origin

Distilled from 8 real project reviews — delivery-stage platforms, a 20+-module production system, a copy-paste product family, server-rendered Flask apps, an admin-template prototype, a WHO-data forecasting pipeline, and an ESP32 smart-watch firmware. Project names anonymized (Projects A–H); every scar, cap rule, and pattern was earned in the field. Forged in real Hermes Agent and ZCode review sessions, now public for any agent to wield.

## License

[MIT](LICENSE) © 2026 panxiao
