---
name: codebase-tribunal
description: "Codebase Tribunal ⚖️ — put your project on trial: 5-8 expert personas (PM, architect, backend, frontend, UI/UX, QA, DevOps, security) independently audit any software project from different professional angles, then deliver a quantified verdict (score with caps) + an executable fix plan. 多角色项目审查/复评（代码审判庭）：项目经理、架构师、前后端、安全、QA、DevOps 等角色独立审查同一项目，产出量化评分 + 优先级行动清单。触发词：项目审查、项目评估、多角色审查、复评、delta review、审查这个项目、review this project、codebase tribunal。"
version: 1.3.0
author: panxiao
license: MIT
platforms: [linux, macos, windows]
homepage: https://github.com/MUST-panxiao/codebase-tribunal
---

# Codebase Tribunal ⚖️

Put a software project on trial: deploy a virtual tribunal of expert personas to audit it from different professional angles. Each persona thinks independently through its own lens, then findings are synthesized into one prioritized, de-duplicated report with quantified scores and an actionable task list.

**Core principle:** Different roles see different problems. A backend engineer and a product manager looking at the same codebase will surface completely different issues. Force structural separation of perspectives, then synthesize.

## When to Use

- User says "审查这个项目" / "帮我看看这个项目做得怎么样" / "review this project"
- Before delivering a project to a client
- After completing a major milestone (Phase / Sprint)
- Periodic project health check
- Evaluating a codebase you inherited or are about to maintain

**Trigger phrases:** "多角色审查", "项目评估", "project review", "multi-role", "多个角度", "从X和Y的角度", "全面检查"

## Execution Modes

### Mode A: Full Project Audit (Default)

Review the ENTIRE project — architecture, code quality, business logic, UI, deployment readiness. This is the primary mode.

### Mode B: Incremental Review

Review a specific PR, commit range, or set of files. Triggered when user mentions "PR", "diff", "commit", or specific file paths.

> Mode B is similar to `requesting-code-review` skill but with multiple role lenses. If user just wants a quick pre-commit check, use `requesting-code-review` instead.

### Mode C: Re-review (Delta Mode)

When a project has been reviewed before, run a delta review:

**Finding the previous review (do this FIRST, before dispatching any role):**
1. Check the project root for `REVIEW_REPORT.md` / `REVIEW_REPORT_v*.md` / `OPTIMIZATION_TASKS.md` / `MASTER_FIX_PLAN.md`
2. If not at root, check the user's review archive convention — e.g., a dedicated archive directory outside the repo (one sub-directory per project, versioned filenames like `REVIEW_REPORT_v2.md`). Ask the user where previous reports live if none found.
3. A previous review MUST be merged if it exists — reviewing from scratch when a baseline exists wastes the baseline and re-reports known issues.

**Delta review process:**
1. Read the existing review files first; extract the full open-findings ledger (per-finding ID, severity, status, file:line)
2. Focus roles on two jobs: (a) re-verify a sample of previous open findings, (b) find issues MISSED by the previous review — do NOT re-describe known issues in detail
3. Produce a **previous-findings verification table** (旧问题逐项核对表): EVERY previous Critical/Warning item listed with status ✅ 已修 / ⚠️ 部分 / ❌ 未修 / ➖ 已失效 (feature removed), each with current evidence (`path:line` + optional 实测 result). This table is a MANDATORY deliverable of delta mode — it is what makes a re-review valuable.
4. In the final report, explicitly list "N new findings vs previous review"
5. When generating the action plan, MERGE with the existing one — don't duplicate tasks already identified
6. **Abandoned-route residue check** (delta-specific): if git history or the user indicates a feature route was abandoned/killed since the last review, verify the residue is cleaned up: components, routes, backend modules, migrations that were never applied, config entries, docs, and dependencies left behind. Orphaned half-removed features are their own finding class.

## Process

### Step 1: Stack Detection (runs once, shared by all personas)

**Before any role starts reviewing, detect the project's tech stack.** Never assume.

```
1. Find project root: walk up to nearest manifest (package.json / pyproject.toml /
   go.mod / Cargo.toml / pom.xml / requirements.txt) and/or .git
2. Read root CLAUDE.md / AGENTS.md / README if present — project instructions override everything
3. Read the manifest(s): language, framework, key dependencies
4. Identify package manager from lockfile (package-lock.json → npm, pnpm-lock.yaml → pnpm,
   yarn.lock → yarn, poetry.lock / uv.lock → Poetry/uv, requirements.txt → pip)
5. Scan top-level directory layout: where does source / tests / config live?
6. Detect deployment artifacts: Dockerfile, docker-compose.yml, .github/workflows/, Makefile
7. Check for test framework: pytest / jest / vitest / go test / cargo test
8. Verify dependency versions exist (pandas==3.0.2 type hallucinations block Docker builds)
```

**Output:** One-line stack summary. Example:
> `Stack: Python (FastAPI) + TypeScript (Vue 3) · SQLite · Docker · pytest — root: /path/to/project`

**If detection fails** (no manifest, ambiguous workspace): ask user which path to review. Do not guess.

### Step 2: Role Selection

**Default team (always deploy):**

| Role | Focus | Thinking Mode |
|------|-------|---------------|
| 🎯 **Project Manager** | Completion %, tech stack fit, timeline risk, deliverability | "Is this shippable? What's blocking?" |
| 📋 **Product Manager** | Business logic, user flows, feature gaps, UX of the product | "Does this solve the user's real problem?" |
| 🏗️ **Architect** | Module boundaries, coupling, dependency direction, scalability | "Will this structure hold up?" |
| ⚙️ **Backend Engineer** | API design, data flow, error handling, security holes | "What breaks at 3am?" |
| 🎨 **Frontend/UI Engineer** | Component architecture, state management, visual quality | "Would I be proud to show this?" |

**Conditional roles (deploy when stack-relevant):**

| Role | When to deploy |
|------|---------------|
| 🛡️ **Security Engineer** | Always, but emphasize when auth/payments/user-data present |
| 🧪 **QA Engineer** | When test suite exists OR when project is near delivery |
| 🚀 **DevOps Engineer** | When Dockerfile / CI/CD / deployment configs present |
| 📊 **Data Engineer** | When ML/pipeline/ETL components present |

**How to deploy:**

- **In agents with a sub-agent tool (ZCode, Claude Code, etc. — primary mode):** Use the `Agent` tool, ideally with a read-only sub-agent type (ZCode: `subagent_type: "Explore"`) — 3 parallel Agent calls per message, same batch composition as below. See the Adaptation Guide for the full tool mapping.
- **In Claude Code (sub-agent mode):** Use `Agent` tool to dispatch parallel sub-agents.
- **In Claude Code (single-agent mode, GLM backend):** Run roles sequentially in one conversation, switching persona each section. Cheaper, shares context, but slower.
- **Fallback (any environment):** Run all roles sequentially in the current session.

**Sub-agent context packaging:** Each `delegate_task` goal prompt MUST include:
1. The role name + thinking mode (from references/role-personas.md)
2. The exact project root path
3. The stack summary from Step 1
4. A list of key files to read (don't make them guess — point them at entry points, configs, main modules)
5. Domain context (what the project does, who uses it, Chinese-language if applicable)
6. The Output Contract format (inline the template so they don't need to load reference files)

### Step 3: Each Persona Reviews Independently

Load `references/role-personas.md` for the full persona definitions. Each role:

1. Adopts its persona's thinking mode and expertise
2. Scans relevant files (use `search_files` / `read_file` / `terminal`)
3. Checks its Red Flags list (see role-personas.md)
4. Produces an **Output Contract** block

**Output Contract format (every role uses this):**

```markdown
### <Role Name> Review

**Score: X/10** — <one-line justification>

**Critical** (must fix before delivery)
- `path:line` — <issue> — <why it matters>

**Warning** (should fix)
- `path:line` — <issue> — <recommendation>

**Suggestion** (optional improvement)
- `path:line` — <improvement>

**Health Assessment**
<2-3 sentence paragraph summarizing this aspect of the project>
```

If a tier has no findings, write `- None.` — don't omit the tier.

### Step 4: Dynamic Verification (实测)

Static code reading proves a bug *probably* exists; running the system proves it *does*. Field experience (Project C v2 delta review) shows dynamic verification is what turns "looks like missing auth" into confirmed exploitable findings — and it also catches false positives before they embarrass the report.

**When to run:** after the implementation roles (Batch 2) return, before synthesis. Mandatory for any 🔴 finding that claims "missing auth", "path traversal", "publicly readable", "injection"; optional (but recommended) for everything else.

**How:**
1. Get the service running locally: existing dev server, `docker compose up`, or `uvicorn`/dev-server directly. If the project cannot start locally within a reasonable effort, say so explicitly in the report ("未实测" tag on affected findings) — do not silently skip.
2. Probe each candidate finding with minimal, non-destructive requests:
   - Auth: request the endpoint with NO credentials → 200 means confirmed, 401/403 means finding is wrong or conditional
   - Path traversal: `curl --path-as-is` with `../` sequences (read side); check 403/404 handling
   - Public artifact exposure: request the predictable URL of a generated artifact
   - Rate limit / cache headers: verify with response headers
3. Run the test suite: `pytest` / `vitest` etc. Record pass/fail counts — failing tests that nobody noticed are themselves a finding.

**Guardrails (non-negotiable):**
- Local instance ONLY. Never probe a deployed/production URL, even the user's own.
- Read-only probes by default (GET, HEAD). Write-side probes (e.g., traversal WRITE via upload filename) only with explicit user consent, and only with harmless payloads (e.g., a temp file in /tmp).
- No destructive payloads: no DROP/TRUNCATE, no bulk requests, no real exfiltration.

**Output:** annotate each verified finding with `**实测**：<request> → <result>` evidence, and mark refuted findings as false positives so synthesis drops them.

### Step 5: Synthesis

After all roles complete, synthesize into a unified report. **This step is done by the orchestrator (you), NOT by sub-agents.**

**Process:**
1. Collect all Output Contract blocks from sub-agents
2. Extract every finding into a flat list with: `{role, severity, file, line, issue, impact}`
3. Apply de-duplication and conflict rules below
4. Compute overall project score per the rubric

**De-duplication rules:**
- Same `path:line` flagged by 2+ roles → list once, tag `[roles: backend, security]`
- Multiple roles agreeing = higher confidence, boost priority
- Use the role boundary table in `references/role-personas.md` to determine who "owns" each finding

**Conflict surfacing:**
- If roles disagree (e.g., architect says "split this file", backend says "it's fine as is"):
  - Present both positions under **"Conflicting Opinions"**
  - Give a tie-breaker with reasoning

**Gap noting:**
- If a role couldn't complete (e.g., no test files for QA), say so explicitly

**Score computation (mechanical — apply in this exact order, show the work):**

```
1. overall_raw = average of all role scores
2. caps = []
   - if ANY Critical finding exists (unrefuted)        → caps.append(7)
   - if 3+ Critical findings                           → caps.append(5)
   - if ANY role scored ≤ 3/10                         → caps.append(6)
   - if core business logic completely absent          → caps.append(4)
3. overall = min(overall_raw, *caps)   # floor at 0
```

Show this computation explicitly in the report (e.g., `总分：5.0 = min(均值 6.1, 3+ Critical 封顶 5)`). Field lesson: the cap rule exists in the rubric but was silently skipped in a real review (Project C v1 scored 5.9 uncapped; v2 recomputed to 5.0) — so write the arithmetic down, don't just "apply" it. Overall score reflects "can this be delivered as-is?"

### Step 6: Final Report

The final report structure:

```markdown
# <Project Name> — Multi-Role Review Report

## Executive Summary
- Overall Score: X/10
- One-paragraph verdict
- Top 3 risks

## Role Scores
| Role | Score | Key Finding |
|------|-------|-------------|
| Project Manager | 8.5/10 | Phase 10 incomplete |
| Product Manager | 8/10 | Upload validation missing |
| ... | ... | ... |

## Critical Issues (must fix before delivery)
[Merged from all roles, de-duplicated, ranked]

## Warnings (should fix)
[Merged]

## Suggestions (optional)
[Merged]

## Conflicting Opinions
[If any]

## Recommended Action Plan
Phase A: <blocking issues> — estimated effort
Phase B: <high-value improvements> — estimated effort
Phase C: <polish> — estimated effort
```

**Delta mode appends two sections** (before the Action Plan):
- `## 旧问题逐项核对表` — the mandatory previous-findings verification table (see Mode C)
- `## 本轮新增 vs 上轮` — N new findings, M resolved, K partially fixed

### Step 7: Generate Action List (recommended for delivery-ready reviews)

Generate a `MASTER_FIX_PLAN.md` at the project root. This is the primary deliverable for the user.

**Output location convention:**
1. Default: write both files at the project root.
2. **If the user maintains an external review archive** (e.g., a desktop directory with one sub-folder per project — check for one, or ask), write there instead: it keeps review artifacts out of the delivered repo and preserves history across versions. Use versioned filenames (`REVIEW_REPORT_v3.md`) when a previous version exists; never overwrite it.
3. **If a portfolio index exists** in the archive (a master index / dashboard tracking all reviewed projects), update the entry for this project — one row: date, version reviewed, score, delta vs last round, open Critical count.

**If the project already has a fix plan** (e.g., `OPTIMIZATION_TASKS.md` from a previous review):
1. Read the existing plan
2. MERGE new findings with existing tasks — do NOT duplicate
3. Re-number into a unified Phase structure (A=blocking, B=should-fix, C=maintainability, D=UI, E=testing, F=ops)
4. At the bottom, include a "Full findings mapping table" showing every finding → which task addresses it → which review round found it

**Each task in the plan MUST have:**
- Task ID (A1, A2, B1, ...)
- Severity emoji: 🔴 Critical / 🟡 Warning / 🟢 Suggestion / 🎨 Visual
- File path + line number
- Current code snippet (so Claude Code doesn't have to re-read the file)
- Expected change (specific, not vague)
- Verification method (grep command, test command, manual check)
- If applicable: cross-reference to detailed CSS/code in another file (e.g., "详见 OPTIMIZATION_TASKS.md D1")

**Also generate a `REVIEW_REPORT.md`** with the full 8-role scores and findings — this is the audit trail for management, separate from the action plan.

## Anti-Patterns to Avoid

1. **Perspective convergence** — all roles saying the same thing. Prevented by:
   - Each persona has a distinct thinking mode
   - Red Flags are role-specific (backend checks SQL injection, PM checks user flow gaps)
   - If two roles produce >70% identical findings, re-run with sharper separation

2. **Vague findings** — "the code could be better" is useless. Every finding must have:
   - A specific `path:line` reference
   - Why it matters (impact, not just description)
   - A concrete recommendation

3. **All critique, no praise** — acknowledge what's done well. The Health Assessment paragraph should be honest about both strengths and weaknesses.

4. **Padding** — don't invent issues to fill a tier. If a role finds nothing Critical, that's fine. `- None.` is a valid and honest finding.

## Adaptation Guide

### For ZCode (sub-agent mode — current primary environment)

Tool names differ between agent environments. The reference mapping below uses `delegate_task`/`search_files` as the generic form; translate to your environment (ZCode/Claude Code: `Agent`/`Grep`/`Read`):

| Generic tool | ZCode / Claude Code equivalent | Notes |
|---|---|---|
| `delegate_task` (goal prompt) | `Agent` tool with `subagent_type: "Explore"` | Explore is READ-ONLY — ideal for review personas (no accidental edits). Use `general-purpose` only when a role must run mutating commands. |
| `search_files` | `Grep` / `Glob` tools | — |
| `read_file` | `Read` tool | — |
| `terminal` | `Bash` tool | Required for Step 4 实测 and test-suite runs. |

- **Batching:** send 3 Agent calls per message (they run in parallel). Same batch composition as below (Batch 1 = PM/Product/Architect, 2 = Backend/Frontend/Security, 3 = QA/DevOps). Wait for each batch before dispatching the next.
- **Prompt packing:** Agent prompts must be self-contained (sub-agents start fresh) — inline the role persona summary, thinking mode, stack summary, project path, key file list, domain context, the Output Contract format, and "Respond in Chinese." Do not tell them to load reference files.
- **Verification of sources:** review findings must cite `path:line` the orchestrator can spot-check; Explore agents return conclusions, so ask each to include exact file paths + line numbers + short code excerpts.
- **Related skills** (`requesting-code-review` etc.) may not be installed in this environment — if Mode B is requested and the skill is missing, perform the review inline with the Security + Backend roles instead of failing.
- **Dynamic verification (Step 4)** runs in the MAIN agent (Bash), not in Explore sub-agents — they are read-only by design, and service startup/probing is stateful work best kept in one place.

### For Chinese-language projects (中文项目)

- Conduct the review in Chinese
- Technical terms can stay in English (API, JWT, Docker, etc.)
- Role names: 项目经理 / 产品经理 / 架构师 / 后端工程师 / 前端工程师 / UI设计师 / 测试工程师 / 运维工程师 / 安全工程师
- Each delegate_task prompt should include "Respond in Chinese."
- Typical workflow: review report + fix plan → hand off to your coding agent (Claude Code, etc.) for execution
- Fix plans must be precise enough for Claude Code: exact file paths, line numbers, current code snippet, expected fix, verification command
- Reports and action plans in Chinese (this is what the user hands to Claude Code)

### For scientific/public health projects

- Add domain-specific checks: data sensitivity, de-identification, regulatory compliance
- Product Manager persona should consider: "Would a non-technical researcher use this comfortably?"
- Backend should check: algorithm correctness (are ML model outputs real or placeholder?), data pipeline integrity end-to-end
- DevOps should check: are model training workloads resource-bounded? (OOM risk from concurrent ML jobs)
- Extra Critical criteria for this domain:
  - Generated reports containing hardcoded/placeholder values in user-facing output → always Critical
  - "Mock" or "TODO" text visible to end users → always Critical (user trust in health data is non-negotiable)
  - Model variant that silently aliases another (user selects X, system runs Y) → always Critical
  - **Product name doesn't match functionality** → always Critical (health platforms promising monitoring/early-warning must deliver those features, not static displays)

### For data pipeline / scientific computing projects (no web frontend)

Pure data pipeline projects (scraper → data processing → JSON/output files → optional read-only API) need a **different role mix** than web platforms. Learned from reviewing `Project G` (WHO FluNet pipeline: Selenium scraper + ARIMA/ETS/Prophet forecasting + MEM threshold engine + FastAPI read-only API).

**Deploy 5 roles** (drop Frontend/UI):

| Role | Focus shift for data pipelines |
|------|-------------------------------|
| 🎯 PM | Is the pipeline reproducible? Are generated data files (CSV/JSON) tracked in git (they shouldn't be)? |
| 📊 **Data Engineer** (replaces Product Manager) | NaN semantics, data quality filters, incremental computation correctness, evaluator module integration |
| ⚙️ Backend | API design (if present), in-memory data store concurrency, token security |
| 🛡️ Security | Hardcoded tokens, Selenium sandbox, TRUNCATE migration safety, dependency pinning |
| 🚀 DevOps | systemd service placeholders, log rotation, backup strategy, cross-platform memory detection |

**Pipeline-specific Critical checks (always flag):**
- Evaluator/assessment module written but never imported/called → Critical (claims feature exists but doesn't)
- NaN values silently treated as 0 (pandas `.sum(skipna=True)` on columns where NaN means "not tested") → Critical (data correctness)
- Generated data files (CSV/JSON/output directories) committed to git → Critical for delivery/soft-copy projects
- `TRUNCATE TABLE` in migration without transaction rollback → Critical
- `/proc/meminfo` hardcoded (Linux-only, fails silently on macOS) → Warning
- Incremental metadata marking failed tasks as "success" → Warning

### For ESP32 / embedded firmware + companion app projects

IoT projects that pair a microcontroller firmware (ESP32/Arduino) with a PC-side companion service (Python/Go daemon). The "PC companion that controls local software" pattern. Learned from reviewing `Project H` (ESP32-S3 firmware + Python FastAPI bridge + SDL2 desktop simulator, scored 6.8/10).

**Deploy 5 roles** (drop Frontend/UI, repurpose for embedded):

| Role | Focus shift for embedded |
|------|--------------------------|
| 🎯 PM | Is the companion service documented? Can a user set up the device + companion from scratch? Can the device operate standalone when the bridge is down? |
| 🏗️ Architect | Communication architecture (pull vs push, WiFi vs BLE vs USB), firmware/companion protocol design, display framework choice (M5GFX raw vs LVGL), desktop simulator workflow (huge dev-ex win if present) |
| ⚙️ **Firmware Engineer** (replaces Backend) | Blocking calls in `loop()`, WDT triggers, memory management (PSRAM/heap buffer overruns), power management (screen dim/off), proper FreeRTOS task design |
| 🛡️ Security | Hardcoded WiFi/API credentials in firmware (ESPECIALLY in public repos — firmware secrets are baked into binary, can't rotate without re-flashing), companion API zero-auth on local network, BLE pairing security |
| 📋 **Companion App Engineer** (replaces Product Manager) | Bridge/daemon architecture, dependency version locking, process detection accuracy (native hooks vs CPU% guessing), terminal injection safety, graceful degradation when bridge is down |

**Embedded-specific Critical checks (always flag):**
- `delay() > 500ms` in Arduino `loop()` → Critical (triggers FreeRTOS Watchdog Timer ~5s, chip reboots)
- Credentials hardcoded in firmware source committed to public repo → Critical (permanent leak via Git history)
- Companion API POST endpoints with zero auth → Critical (LAN anyone can inject commands)
- Multiple graphics frameworks writing to same SPI bus without mutex → Critical (display corruption/tearing)

**Embedded-specific Warning checks:**
- BLE + WiFi simultaneously active without managing RF sharing → shared 2.4GHz radio, degraded performance
- BLE messages without MTU negotiation → default 23-byte MTU truncates JSON
- Application status inferred from indirect OS metrics (CPU%) instead of native hooks → unreliable
- Device pulls data from PC via HTTP requiring hardcoded IP → breaks on network change; prefer BLE or WebSocket push
- Version-unlocked `requirements.txt` in companion service → same as web projects but easily forgotten
- Audio buffer allocated with `ps_malloc()` but no bounds check on recorded data
- `preferences` / NVS not used for configurable values (WiFi creds, device name, thresholds)

**Key reference**: See `references/findings-patterns.md` patterns #37-42 for detailed ESP32/embedded findings with check commands and fix patterns.

**Score calibration**: A vibe-coded firmware project that boots, displays UI, and communicates scores 5-7/10. To score 8+: no hardcoded credentials, graceful network failure handling, non-blocking main loop, viable standalone operation mode.

### For Flask / traditional server-rendered apps

- Frontend role: review Jinja2 templates (DRY-ness, `{% extends %}`, autoescape), CSS files, and inline JS — NOT Vue components
- Backend role: check CSRF protection (`CSRFProtect` initialized?), Flask-Login session security, `before_request` abuse, `gunicorn --workers` vs SQLite compatibility
- Security: hardcoded secrets in config files (config.yaml, .env defaults), open redirect in login `next` parameter, `innerHTML` XSS in template JS
- The "API" may not be REST — it could be route handlers returning `jsonify()`. Check auth on each route uniformly.

## References

- `references/role-personas.md` — Full persona definitions, Red Flags, expertise areas
- `references/review-rubric.md` — Severity classification guide + scoring rubric
- `references/stack-detection.md` — Detailed stack detection protocol
- `references/findings-patterns.md` — Common findings from real project reviews + delta re-review rounds (44 proactive patterns + scoring calibration)
- `references/execution-playbook.md` — Real-world execution patterns: batching, prompt templates, synthesis tips

## Dispatch Strategy (from 8 real projects)

Tested on 8 real projects (anonymized as Projects A–H in `references/findings-patterns.md`): a near-delivery platform, a 20+-module production platform, a copy-paste product family, a Flask+Jinja2 app, a template-based prototype, a scientific data pipeline, and an ESP32 firmware + companion-bridge project. Key learnings:

### How to batch sub-agent dispatches

Dispatch 3 roles per batch (the parallel limit). 8 roles = 3 batches:

**Batch 1**: PM + Product Manager + Architect (these 3 need broad context, read the most files)
**Batch 2**: Backend + Frontend/UI + Security (these need deep code reading)
**Batch 3**: QA + DevOps (these need to check infra, tests, CI — lighter code reading)

Each sub-agent prompt MUST include:
1. The stack summary from Step 1 (so sub-agents don't re-detect)
2. The project root path
3. Specific files to read (list 5-8 key files per role)
4. "Respond in Chinese" for Chinese-language projects
5. The Output Contract format

### Adapting roles to project maturity

| Project maturity | Roles to deploy | Expected depth |
|---|---|---|
| **Prototype** (< 10 files, pre-delivery) | PM, Backend, Frontend (3 roles) | Focus on blockers, quick wins |
| **Template-based prototype** (like Project F) | All 8 roles, emphasize Product Manager + PM | Core business logic likely absent. Focus on: does the product name match functionality? Are static images/JSON masking as dynamic features? Is the core algorithm (if it exists in a separate directory) integrated into the platform? |
| **Near-delivery** (like Project A) | All 8 roles, but emphasize PM/Product/DevOps | Focus on delivery readiness |
| **Active development** (like Project B) | All 8 roles | Focus on architecture + flow gaps |
| **Production platform** (like Project C) | All 8 roles, emphasize Security/QA/DevOps | Focus on accumulated debt + risks |
| **Traditional server-rendered** (Flask/Jinja2, like Project E) | All 8 roles, emphasize Security + Frontend | Focus on CSRF/XSS, template DRY-ness, SQLite concurrency, dual DB access paths |
| **Data pipeline + thin API** (like Project G) | PM-only review acceptable if user asks for one role; full review = PM + Data Engineer + DevOps + QA | Focus on: pipeline module wiring (see pattern #33), git repo hygiene, deployment config placeholders, no-DB/no-test implications. If single-role, still produce the full Output Contract format. |
| **Embedded firmware** (like Project H) | 5 roles: PM + Architect + Firmware + Security + Companion Eng | Focus on blocking calls, credentials in public repos, communication architecture (pull vs push, BLE vs WiFi), display framework choice |

### Monorepo / multi-project directories

When the target directory contains multiple sub-projects (common in shared workspaces):
1. List all sub-projects and their tech stacks
2. Ask user which sub-project to review
3. Do NOT review the entire directory — each sub-project needs its own review

### Output language

For Chinese-language projects: conduct the review in Chinese, but technical terms stay in English.
Add "Respond in Chinese." to each delegate_task prompt.

## Generating the Action List (Step 7) — Real-World Pattern

After completing reviews on 6 projects, the most valuable deliverable pattern is:

1. **Two files**, not one:
   - `REVIEW_REPORT.md` — the full 8-role report with scores, findings, and health assessments. For archiving and reporting upward.
   - `MASTER_FIX_PLAN.md` — the actionable task list. Organized into Phase A (blocking) → B (high-value) → C (polish) → D (UI) → E (testing) → F (ops). Each task has: file:line, current code, expected fix, verification method.

2. **Cross-reference findings**: when two roles flag the same issue (e.g., Backend + Security both find path traversal), merge into one task tagged `[roles: backend, security]`.

3. **Include a comparison table** if the user has reviewed sibling projects (e.g., "vs Project A" / "vs Project B"). This helps the user prioritize across their portfolio.

4. **Verification checklist** at the bottom: runnable commands (`grep`, `pytest`, `vue-tsc`, `docker compose up`) that prove the fixes work.

## Cross-Project Comparison and Product-Family Shared Fixes

### When this applies

After reviewing multiple related projects (e.g., product families like 3 near-identical copy-paste codebases, or a portfolio of platforms sharing tech stacks), the user will often ask to compare or sync fixes across them.

### Cross-project comparison table

After reviewing 2+ projects, include a comparison table in the conversation (not necessarily a file):

```markdown
| Dimension | Project A (score) | Project B (score) | Project C (score) |
|---|---|---|---|
| Design system | 42 hardcoded colors | ✅ DESIGN.md + CI | ⚠️ partial |
| Security auth | 🔴 zero auth | ✅ ownership checks | 🔴 path traversal |
| Test coverage | 0% | 77.3% | 0% |
```

This helps the user prioritize which project to fix first and spot systemic patterns.

### Product-family shared fix plan

When the user says "sync to other projects" or "these are shared across products":

1. **Read the corresponding files** in sibling projects (use `grep` / `read_file` to check if the same code exists)
2. **Create a shared fix plan** (e.g., `SHARED_FIX_PLAN.md`) that:
   - Lists each finding with a column per product showing ✅ (has issue) / ❌ (not affected) / ❓ (needs checking)
   - Groups into "shared fixes" (one fix, apply to all) vs "product-specific fixes"
   - Includes a file-correspondence table (e.g., "snapshots.py:95 in Flu = snapshots.py:90 in Meningitis")
   - Recommends batch execution order
3. **For copy-paste codebases** (common in product families): check that line numbers are close but may differ by a few lines due to product-specific additions. Always verify with `grep` before assuming.

### Verification commands for cross-project checks

```bash
# Check if a specific bug exists across all sibling projects
for proj in Product D1 Product D2 Product D3; do
  echo "=== $proj ==="
  grep -n "PATTERN" /path/to/$proj/backend/services/license.py
done
```

## Reviewing Development Plans (Pre-code Architecture Audit)

Sometimes the user asks you to review a **development plan document** (e.g., `DEVPLAN_V2.md`) rather than source code. This is a distinct mode — finding design flaws before they become code bugs. The same multi-role lens applies, but the focus shifts:

**Embedded Engineer reviews:**
- Hardware constraints not addressed (WDT, RF coexistence, memory placement, flash partitioning)
- Blocking patterns in planned code (will `delay()` be used? where?)
- BLE protocol design (MTU, fragmentation, bonding, heartbeat/keepalive)
- OTA upgrade path (even if not implementing, partition table must be ready)

**Backend/Service Engineer reviews:**
- Concurrency model (threading vs asyncio mixing — a classic landmine)
- Error recovery (daemon crash → auto-restart? hook script timeout → CC blocks?)
- API design (port merging, authentication on companion endpoints)
- Race conditions in shared state (session cleanup while reading)

**Architect reviews:**
- Protocol stack conflicts (e.g., BLE HID + custom GATT on ESP32 — unstable)
- Data flow direction (pull vs push — pull creates IP dependency)
- Component coupling (can tracks develop independently?)
- Reference project alignment (does the plan match what proven projects actually do?)

**Product Manager reviews:**
- MVP scope (is the plan trying to do too much at once? should it be phased?)
- Onboarding/first-run experience (pairing flow, config portal, factory reset)
- Success criteria (is there a definition of "done"?)
- Degraded mode UX (what does the device show when disconnected?)

**Process:**
1. Read the entire plan document end-to-end first
2. Cross-reference against source code (does the plan match what code currently does?)
3. For each expert lens, list findings with severity: 🔴 will block development / 🟡 will cause rework / 🟢 optimization opportunity
4. Produce a prioritized fix list (P0/P1/P2) that the user can patch into the plan document
5. Offer to apply the fixes directly to the document

**Key question to ask:** "Will this plan cause the developer to hit a wall mid-implementation?" If yes, that section needs more detail or a different approach.

## Pitfalls

- **Large projects (> 100 files):** Don't try to read every file. Use `search_files` to find patterns, read key files (entry points, configs, models), sample 3-5 components per layer. For 50K+ line projects (like Project C), focus each role on its specific domain files.
- **No frontend:** Skip frontend/UI roles. Don't fabricate findings.
- **No tests:** QA role notes "No test suite found" as a Critical finding itself.
- **Monorepo with multiple apps:** Detect all sub-projects, ask user which to review.
- **Non-code projects (data/notebooks):** Adapt — review data pipeline logic, notebook reproducibility, documentation quality instead of API/component architecture.
- **Mature production platforms:** Don't just look for missing features — look for accumulated debt: version drift (CHANGELOG vs code), stale tests, abandoned feature branches, module coupling that grew over time. The "it works in production" bias can hide serious security holes (Project C had 60 endpoints with zero auth despite running fine for months).
- **Template-based prototypes (fantastic-admin, vue-pure-admin scaffolds):** The UI may look polished (100+ components, full layout system) but core business logic is absent. Don't be fooled by template completeness. Check: does the backend have any domain-specific API? Are the frontend pages showing real data or static images/JSON? Is there a separate algorithm directory that's not integrated?
- **Dependency version hallucinations:** AI coding assistants sometimes generate non-existent package versions (e.g., `pandas==3.0.2`). Always verify versions exist during stack detection — a non-existent version blocks Docker build completely.
- **delegate_task context size:** Each sub-agent gets its own context window. Pass enough context (stack summary, project structure, key observations from stack detection) so the sub-agent doesn't waste calls re-discovering basics.
- **Synthesis timing:** Wait for ALL roles to complete before synthesizing. Don't start writing the report until every delegate_task returns — late-arriving findings may change the priority ranking.
- **Sub-agent context starvation:** If you pass too little context to delegate_task, sub-agents waste time re-detecting the stack and reading files you already know. Always pass the stack summary + key file list + domain context in the goal prompt. Inline the Output Contract format so they don't need to load reference files.
- **UI review vagueness:** When generating UI tasks (Phase D), don't write "improve the visual design" — provide exact CSS values, color hex codes, file paths, and line numbers. The action plan should be executable by Claude Code without further design decisions. Reference a design system (e.g., pure-admin, Arco) by name and extract concrete tokens.
- **Role merging for small projects:** When a project is small (~30 source files), it's acceptable to merge roles into fewer sub-agents (e.g., Architect+Backend in one agent, Security+DevOps+QA in another). This saves token budget while still covering all perspectives. Merge roles that read overlapping files.
- **Hardcoded secrets scan is ALWAYS relevant:** Even if the project has a `.gitignore` and `.env`, always grep for hardcoded API keys, SMTP passwords, and SECRET_KEY fallbacks in Python/YAML/docker-compose files. Users frequently leave default values as "temporary" fallbacks that become permanent.
- **Cross-project comparison value:** After reviewing 3+ projects in a session, the user gets significant value from a comparison table showing relative scores and systemic patterns. Always include this in the conversation after the final review — it helps prioritize which project to fix first.
- **Plan duplication on re-review:** When re-reviewing a project, ALWAYS read existing REVIEW_REPORT.md / OPTIMIZATION_TASKS.md / MASTER_FIX_PLAN.md first. Merge new findings into the existing plan. Explicitly list "N new findings vs previous review" in the report — this demonstrates value to the user.
- **Score inflation:** Don't give 8+ to a project with zero tests, no deployment artifact, or hardcoded user-facing data. The rubric caps at 7 if ANY Critical exists. Be honest — the user needs the real number to make delivery decisions. If core business logic is absent (only auth/scaffolding), cap at 4 regardless of UI polish.
