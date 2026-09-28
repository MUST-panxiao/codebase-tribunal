# Review Severity Rubric

Shared reference for all personas. Each role carries short severity tiers inline;
read this file when a finding is hard to classify.

## Severity Tiers

### 🔴 Critical — must fix before delivery

A defect that will cause incorrect behavior, data loss, a security hole, or an outage.
The project should not be delivered until this is resolved.

**Examples:**
- Unvalidated user input reaching a database query
- Missing authorization check on a mutating route
- A migration that drops a column with live data
- A broken data contract between API and UI
- An unhandled rejection on a critical path
- A secret committed to the repo
- Mock/placeholder code in a production user flow
- No deployment artifact for a project being delivered

**Classification test:** *Can this corrupt data, breach security, or break a user-facing path?*
If yes → Critical.

### 🟡 Warning — should fix, not blocking

A real problem that degrades correctness, performance, maintainability, or user experience,
but does not by itself justify blocking delivery. Fix now or file a follow-up.

**Examples:**
- N+1 query on a warm path
- Missing index on a frequently filtered column
- Missing aria-label on an interactive control
- Error swallowed without logging
- Duplicated logic that should be extracted
- Hardcoded colors that should be CSS variables
- API call in a loop without batching
- Missing loading/empty state for an async operation

**Classification test:** *Is this objectively a defect?* If yes → Warning.

### 🟢 Suggestion — optional improvement

A judgment-call improvement: naming, structure, a cleaner pattern, a small DX win.
The developer can take it or leave it.

**Examples:**
- Extract a helper function
- Prefer an existing utility
- Tighten a type annotation
- Add a clarifying comment on non-obvious logic
- Rename a variable for clarity
- Lazy-load a heavy dependency

**Classification test:** *Is this a preference rather than a defect?* If yes → Suggestion.

---

## Classification Guidance

- **When unsure between Critical and Warning:** Ask "can this corrupt data, breach security,
  or break a user-facing path?" → Yes = Critical.

- **When unsure between Warning and Suggestion:** Ask "is this objectively a defect, or a
  preference?" → Defect = Warning. Preference = Suggestion.

- **Severity is about impact, not effort.** A one-line fix can be Critical; a large
  refactor can be a Suggestion.

- **Don't inflate.** Style nitpicks are Suggestions at most. Reserve Critical for things
  that genuinely must not ship.

- **Cite evidence.** Every finding names a `path:line` (or `path` for whole-file concerns)
  and states *why it matters*, not just *what* it is.

- **Compound issues escalate.** If three Suggestions all affect the same area/flow,
  together they may constitute a Warning. If three Warnings cluster, that may be Critical.

---

## Scoring Rubric

Each role produces a score out of 10 for its domain:

| Score Range | Label | Interpretation |
|---|---|---|
| 9.0 — 10.0 | Excellent | Production-ready, minimal or no issues found |
| 7.0 — 8.9 | Good | Solid foundation, some improvements needed but nothing blocking |
| 5.0 — 6.9 | Fair | Functional but has notable gaps or technical debt |
| 3.0 — 4.9 | Poor | Significant issues, would require substantial rework |
| 1.0 — 2.9 | Critical | Major problems throughout, needs fundamental changes |
| 0.0 — 0.9 | Broken | Non-functional or fundamentally flawed |

### Score Deduction Guide

Start at 10, deduct for findings:
- Each Critical: -2 to -3 points
- Each Warning: -0.5 to -1 point
- Each Suggestion: -0.1 to -0.2 points

Floor at 0. Don't give negative scores.

### Overall Project Score

The overall project score is NOT a simple average of role scores. Weight by impact:

- Critical findings from ANY role cap the overall score at 7
- 3+ Critical findings cap at 5
- The overall score should reflect "can this be delivered as-is?"

| Overall Score | Delivery Verdict |
|---|---|
| 8.5+ | Ship it |
| 7.0 — 8.4 | Ship with minor fixes |
| 5.0 — 6.9 | Not ready — needs a sprint of work |
| < 5.0 | Fundamental rework needed |
