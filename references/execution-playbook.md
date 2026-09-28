# Execution Playbook

Real-world patterns from field-testing this skill on actual projects.
Read this when planning a multi-role review execution.

## delegate_task Batching Strategy

Tested on GLM-5.2 backend. Sub-agents are independent — they get their own context window and tool access.

**Optimal batch size: 3 sub-agents per delegate_task call.**

Larger batches work but take longer (serialized execution within the call).
Smaller batches underutilize parallelism.

### Recommended Batching (8-role full audit)

**Batch 1** (business + structure):
- Project Manager
- Product Manager
- Architect

**Batch 2** (implementation):
- Backend Engineer
- Frontend/UI Engineer
- Security Engineer

**Batch 3** (quality + ops):
- QA Engineer
- DevOps Engineer

Wait for each batch to complete before starting the next. This lets you
reference findings from earlier batches when crafting later prompts
(e.g., if PM found a deployment gap, DevOps can dig deeper).

## Sub-Agent Prompt Template

```text
You are a {ROLE_EMOJI} {ROLE_NAME} reviewing the {PROJECT_NAME} at {PROJECT_PATH}.

Stack: {STACK_SUMMARY}

Read these key files: {FILE_LIST}

Your thinking mode: "{THINKING_MODE}"

Assess:
{ASSESSMENT_CHECKLIST}

Produce Output Contract:
### {Role Name} Review
**Score: X/10** — one-line justification
**Critical** (must fix before delivery)
- `path:line` — issue — why it matters
**Warning** (should fix)
**Suggestion** (optional)
**Health Assessment** (2-3 sentences)

If a tier has no findings write `- None.`
```

## Context to Pack for Each Role

| Role | Must-read files | Domain context to include |
|------|----------------|--------------------------|
| PM | README, dev plan, docker-compose, requirements/package.json | Who is the end user? What's the delivery deadline? |
| Product | routes.py, services.py (pipeline section), main UI views | Target user persona (technical? non-technical?) |
| Architect | services.py (FULL), __init__.py, main.py, models/, router/ | Monolith vs microservices? Expected scale? |
| Backend | routes.py (FULL), services.py (upload+pipeline), core/ (security, db, config) | Auth mechanism? Database type? Async framework? |
| Frontend | All .vue files, router, stores, composables, package.json, styles/ | UI framework? Component library? Design system? |
| Security | security.py, config.py, dependencies.py, routes.py (auth checks), .gitignore, Dockerfile | Any known CVEs in deps? Handling PII? |
| QA | Search for test files, services.py (testability), composables | Is there CI/CD? What's tested vs untested? |
| DevOps | Dockerfile(s), docker-compose.yml, nginx.conf, main.py (lifespan), .env.example | Target deployment env? Resource constraints? |

## Synthesis Speed Tips

1. **Don't re-read files during synthesis.** You already have all 8 Output Contracts from sub-agents.
2. **Build a flat finding list first**, then sort/filter. Don't try to write the final report top-down.
3. **The mapping table at the end of MASTER_FIX_PLAN.md is the most valuable artifact.** It proves completeness and traces every task back to a finding + reviewer.

## What Makes This Skill Better Than Single-Pass Review

Field-tested on a real flu-prediction platform (Project A):
- First pass (manual 5-role sequential): found 16 issues
- Second pass (skill-driven 8-role parallel): found 15 ADDITIONAL issues (31 total)
- New findings included: CVE vulnerabilities, fake model variants, Docker root access,
  polling error recovery gaps, XSS vectors, and concurrency risks
- **Key insight: parallel sub-agents with independent context windows surface issues
  that sequential single-context review misses** — each agent reads different files
  and applies a different lens without anchoring bias from earlier findings.

## Action Plan Generation Checklist

When generating MASTER_FIX_PLAN.md:
- [ ] Every finding from every role is mapped to a task (or explicitly noted as "won't fix")
- [ ] Tasks are ordered by severity (Critical → Warning → Suggestion → Visual)
- [ ] Each task has: file path, line number, current code, expected change, verification
- [ ] If existing OPTIMIZATION_TASKS.md exists, its content is referenced not duplicated
- [ ] Verification checklist at the bottom covers all dimensions (backend/frontend/deploy/E2E)
- [ ] Finding mapping table at the end shows: # | finding | round | severity | task ID
