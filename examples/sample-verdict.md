# Sample Verdict — Project A (flu-prediction platform)

> A condensed, anonymized example of what the tribunal produces. Format is real;
> paths and scores come from an actual review (Project A in the precedent library).

## Executive Summary

- **Overall Score: 5.4/10** — Not ready. Needs a focused sprint before delivery.
- Top 3 risks: mock data baked into a deliverable report, frontend excluded from
  Docker deployment, zero test coverage on a platform about to ship.

**Score computation (shown, not vibes):**

```
role scores: PM 6, Product 6.5, Architect 6, Backend 7, Frontend 5.5, Security 5.5, QA 1.5, DevOps 5.5
raw average = 5.44
caps: 2 Critical findings → cap 7 (not binding); QA scored ≤ 3 → cap 6 (not binding)
overall = min(5.44, 7, 6) = 5.4   ← no cap triggered, but QA 1.5 was close
verdict band 5.0–6.9: "Not ready — needs a sprint of work"
```

## Role Scores

| Judge | Score | Key Finding |
|---|:---:|---|
| 🎯 Project Manager | 6/10 | Frontend not in Docker — deliverable is incomplete as packaged |
| 📋 Product Manager | 6.5/10 | Report module shows placeholder values to end users |
| 🏗️ Architect | 6/10 | Clean module layout; pipeline layer under-abstracted |
| ⚙️ Backend | 7/10 | Solid API design, but sync `pd.read_csv` inside `async def` |
| 🎨 Frontend/UI | 5.5/10 | 42 hardcoded hex colors, zero CSS variables |
| 🛡️ Security | 5.5/10 | Delete endpoints carry no auth dependency |
| 🧪 QA | 1.5/10 | **No test suite exists** |
| 🚀 DevOps | 5.5/10 | python-jose 3.3.0 pinned — CVE-2024-33663/33664 |

## Critical Issues (must fix before delivery)

1. 🔴 `backend/services/report.py:118` — report generator embeds **mock district
   data** when upstream data is missing, instead of failing. Generated
   deliverables must never contain fabricated numbers. *[roles: product, backend]*
2. 🔴 `backend/routers/data.py:87` — `DELETE /api/datasets/{id}` has **no
   `Depends(get_current_user)`**. Anyone who can reach the port can delete data.
   **实测**：`curl -X DELETE http://127.0.0.1:8000/api/datasets/1` (no credentials) → `200 OK` — confirmed.
3. 🔴 Deployment package contains backend only; frontend served from a dev
   laptop. Delivery artifact ≠ deliverable. *[roles: pm, devops]*

## MASTER_FIX_PLAN.md — Phase A excerpt

### A2. Add auth dependency to mutating routes 🔴

**文件**：`backend/routers/data.py`（第 87 行）

**当前代码**：
```python
@router.delete("/datasets/{dataset_id}")
async def delete_dataset(dataset_id: int):
```

**期望**：
```python
from core.dependencies import get_current_user

@router.delete("/datasets/{dataset_id}")
async def delete_dataset(dataset_id: int, user=Depends(get_current_user)):
```
Apply to all 4 mutating routes in this file (`grep -n "@router.post\|@router.delete\|@router.put" backend/routers/data.py`).

**验收**：
```bash
curl -s -o /dev/null -w "%{http_code}" -X DELETE http://127.0.0.1:8000/api/datasets/1
# expect: 401
```

---

*Full verdicts run 8–20 pages: warnings, suggestions, conflicting opinions with
tie-breaks, per-role health assessments, the phased fix plan (A=blocking →
F=ops), and a finding→task mapping table proving nothing was dropped.*
