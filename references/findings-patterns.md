# Common Findings Across 8 Reviewed Projects

> Compiled from real reviews — every pattern below was found in the wild, not invented.
> Project names anonymized:
> **A** flu-prediction platform (single module, near delivery) ·
> **B** flu platform (medium, active dev) ·
> **C** 20+-module production platform ·
> **D1/D2/D3** copy-paste desktop product family ·
> **E** Flask+Jinja2 server-rendered app ·
> **F** template-based (admin-scaffold) prototype ·
> **G** Python data pipeline + thin FastAPI (no frontend/DB/Docker) ·
> **H** ESP32-S3 firmware + Python bridge (embedded/IoT).
> These patterns are so common they should be checked proactively on every review.

---

## Patterns That Appear in EVERY Project

### 1. Business endpoints with zero authentication
**Found in**: Project A (delete ops unprotected), Project C (60+ endpoints across 4 modules completely unprotected)
**Check**: grep for `@router.post` / `@router.delete` / `@router.put` WITHOUT `Depends(get_current_user)` or `require_role_level`
**Severity**: Always Critical — anyone can upload/delete/trigger expensive computations

### 2. Path traversal in file serving
**Found in**: Project A (had protection via resolve+relative_to), Project C (completely unprotected), Product D1 (snapshot upload unsanitized filename), Product D2 + Product D3 (same snapshot bug)
**Check**: Any endpoint serving/saving files from disk that concatenates user input into a path
**Fix pattern**: `Path(base_dir / user_path).resolve()` then verify `.relative_to(base_dir.resolve())` doesn't raise. For filenames: `Path(re.sub(r'[^a-zA-Z0-9._\-\u4e00-\u9fff]', '_', filename)).name`

### 3. Synchronous I/O in async handlers
**Found in**: Project A (pd.read_csv in async), Project C (email SMTP, WeasyPrint PDF, zipfile), Project B (SSE with sync Redis), Product D1 (NumPy/SciPy simulation in async def)
**Check**: `grep -rn "async def" --include="*.py"` then look for sync calls (pd.read_*, open(), os.path.*, smtplib) inside
**Fix**: Change `async def` to `def` (FastAPI auto-threadpools) OR `await asyncio.to_thread(sync_function, args)`

### 4. Zero or stale test coverage
**Found in**: Project A (zero tests), Project C (tests from v0.7.0, code at v0.17.1), Project B (77% but Celery at 15.9%), Product D1 (zero tests, scientific algorithm unverified)
**Check**: `find . -name "test_*.py" -o -name "*_test.py"` + check CI workflow files
**Note**: The QA role should ALWAYS check test AGE, not just existence — stale tests give false confidence

### 5. Hardcoded hex colors / no CSS design tokens
**Found in**: Project A (42 colors, zero CSS vars), Project B (15 colors, zero CSS vars), Project C (25 baseline + 332 in JS), Product D1 (100+ hardcoded rgba/hex)
**Check**: `grep -rn "#[0-9a-fA-F]\\{6\\}" --include="*.vue" --include="*.css" | wc -l`
**Note**: Project C had a DESIGN.md system with CI lint — check if enforcement is real or theater

### 6. Docker deployment issues
**Found in**: Project A (frontend not in Docker), Project B (frontend Dockerfile runs dev server, PG18 image doesn't exist), Project C (empty SQL init file, no auto-migration), Product D1 (root user, no resource limits)
**Check**: Read Dockerfile + docker-compose.yml end-to-end. Verify: frontend build included? image versions exist? entrypoint runs migrations? non-root user? healthcheck checks DB?

### 7. python-jose CVE
**Found in**: Project A (3.3.0, vulnerable), Project B (3.5.0, fixed), Project C (>=3.3, uncertain)
**Check**: `grep python-jose requirements*.txt pyproject.toml`
**CVE**: CVE-2024-33664 + CVE-2024-33663, fixed in 3.4.0+

### 7a. Hardcoded secrets / API keys in source code
**Found in**: Project E (one LLM provider's API key in a config file, SMTP password hardcoded in two service modules, a second LLM provider's key in a scraper module, SECRET_KEY fallback in docker-compose.yml)
**Check**: `grep -rn "sk-\|password.*=\|api_key.*=\|secret.*=" --include="*.py" --include="*.yaml" --include="*.yml" --include="*.env" | grep -v "os.environ\|getenv\|config\|example"`
**Severity**: Always Critical — secrets committed to Git are permanently leaked via history
**Fix pattern**: Remove all hardcoded defaults, use `os.environ.get('VAR') or raise RuntimeError('VAR not set')`, add `.env.example` documenting required vars

### 7b. CSRF protection not enabled (Flask projects)
**Found in**: Project E (Flask-WTF installed but CSRFProtect never initialized)
**Check**: `grep -rn "CSRFProtect\|WTF_CSRF" --include="*.py"` — if empty despite Flask-WTF in requirements, CSRF is missing
**Severity**: Always Critical — all POST forms vulnerable to cross-site request forgery
**Fix**: `from flask_wtf import CSRFProtect; csrf = CSRFProtect(); csrf.init_app(app)` + add `{{ csrf_token() }}` to all forms

### 7c. LLM/AI-generated content injected via innerHTML without sanitization
**Found in**: Project E (4 predict pages use innerHTML with DeepSeek output, DOMPurify loaded but never called), Product D1 (mammoth v-html without DOMPurify)
**Check**: `grep -rn "innerHTML\|v-html" --include="*.html" --include="*.vue"` + verify if DOMPurify.sanitize() wraps the assignment
**Severity**: Always Critical — XSS attack surface from untrusted AI output

---

## Patterns Specific to Mature Platforms (like Project C)

These only appear in projects with long development history (10+ versions):

### 8. Version number drift across files
config.py says one version, package.json another, README a third, CHANGELOG stops 2 versions ago.
**Found also in**: Product D1 (one version string in the README and two modules, a newer one in the entrypoint and package.json — 8 files inconsistent)
**Check**: Compare version strings in config.py, package.json, README.md, CHANGELOG.md

### 9. Abandoned feature branches
3+ branches behind main by 70+ commits, containing significant code (1000+ lines).
**Risk**: Code rot, merge conflicts growing, developer confusion about what's "in" vs "out"
**Check**: `git branch -vv | grep "behind"` or review branch list

### 10. Module placement inconsistency (models)
Early models in `models/`, later models in `modules/*/models.py`. Creates confusion about where to look.
**Check**: Compare what's in `app/models/` vs `app/modules/*/models.py`

### 11. API path prefix inconsistency
Three styles coexist: `/api` added in main.py, full path in router definition, no prefix at all.
**Check**: Read `main.py` router registration section

### 12. Code copy-paste across sibling modules
4 modules doing similar things (e.g., regional prediction) each with their own copy of the pipeline composable, the task-polling hook, and the chart-theme helper.
**Check**: Compare directory structures of sibling modules — if they have identical file names, they're probably duplicated

---

## Patterns Specific to Desktop/Embedded Products (like Product D1)

These appear in products distributed as desktop EXEs or on-premise appliances:

### 13. Missing imports causing runtime NameError
**Found in**: Product D1 (a router module used 4 error constants + HTTPException that were never imported)
**Check**: For each router file, compare `from ... import` line against all symbols used. Run `pyflakes` or `ruff` to detect undefined names.
**Severity**: Always Critical — the code path crashes 100% of the time when triggered
**Why it happens**: Code is written incrementally, error constants added to errors.py but import statement not updated

### 14. Hardware license fingerprint using deprecated OS commands
**Found in**: Product D1 + Product D2 + Product D3 (all use `wmic` which is removed in Windows 11 24H2)
**Check**: `grep -rn "wmic\|system_profiler\|ioreg" <license-service module>`
**Fix**: Replace with PowerShell `Get-CimInstance` or platform-native APIs

### 15. RSA signature with PKCS1v15 padding
**Found in**: All 3 Deep products (`padding.PKCS1v15()` instead of PSS)
**Check**: `grep -rn "PKCS1v15\|padding\.PKCS" <license-service module>`
**Risk**: Bleichenbacher attack (theoretical but known)

### 16. ECharts resize memory leak
**Found in**: Product D1 (anonymous resize handlers never removed), NOT in Product D3 (correctly used named handler + removeEventListener)
**Check**: `grep -rn "addEventListener.*resize" frontend/src/` then check if `onUnmounted` has matching `removeEventListener`
**Why it happens**: Copy-paste of chart setup code without thinking about lifecycle cleanup

### 17. Vue computed ref used as object key without .value
**Found in**: Product D1 (a panel component indexes a reactive object with a computed ref as key, missing `.value` — `state[currentModule]` instead of `state[currentModule.value]`)
**Check**: `grep -rn '\[[a-zA-Z]*\]' frontend/src/` where the variable is a `computed` ref

---

## Patterns Specific to Early-Stage Template-Based Projects

These appear in projects that started from an admin template (fantastic-admin, vue-pure-admin, etc.) but haven't built real business logic yet:

### 27. Product name doesn't match actual functionality
**Found in**: Project F (called "监测预警平台" but has no monitoring, no early warning, no prediction — just static JSON data display + PNG image viewer)
**Check**: Ask the Product Manager persona: "Does the product name accurately describe what the user gets?" Compare the project name / README description against actual API endpoints and frontend functionality.
**Severity**: Critical — the gap between promise and delivery erodes user trust, especially in health/scientific domains
**Action**: Recommend either renaming the product to match current scope, or implementing the missing core features

### 28. Non-existent dependency versions blocking Docker build
**Found in**: Project F (`pandas==3.0.2` — this version doesn't exist on PyPI, latest is 2.x; Docker build fails 100%)
**Check**: For each dependency in requirements.txt / package.json, verify the version exists. Run `pip install --dry-run -r requirements.txt` or check PyPI/npm registry.
**Severity**: Always Critical — if Docker can't build, deployment is impossible
**Common causes**: Typos, hallucinated versions from AI coding assistants, copy-paste from wrong source

### 29. Module-level side effects (import triggers I/O or computation)
**Found in**: Project F (one module loads entire JSON datasets at import time; another calls `db.create_all()` at module level; class attributes compute year totals at definition time → data becomes stale after New Year)
**Check**: Read the first/last 20 lines of each Python module. Look for: file I/O, network calls, `db.create_all()`, global variable assignment from functions, class attributes that call computation functions.
**Severity**: Critical in multi-worker deployments (Gunicorn forks → each worker re-executes the side effect → race conditions + 4× memory waste)
**Fix**: Lazy-load with caching functions, move side effects into `if __name__ == '__main__'` or inside `create_app()`

### 30. Multiple UI frameworks loaded simultaneously
**Found in**: Project F (Element Plus + reka-ui/Fa* components + TDesign, all bundled into the same app — TDesign used for just ONE page)
**Check**: `grep -E "element-plus|tdesign|ant-design|arco|naive-ui|reka-ui|primevue" package.json` — if 2+ appear, flag it.
**Severity**: Warning — unnecessary bundle bloat (200KB+), confused design system, inconsistent component APIs
**Fix**: Standardize on one framework. If a specific component from another lib is needed, use tree-shaking or extract the CSS only.

### 31. Core algorithm code exists but is not integrated into the web platform
**Found in**: Project F (a separate algorithm directory outside the web project root contains the actual scientific model code — two computation modules — but it is completely disconnected from the Flask backend; the web platform shows static pre-rendered PNG images instead)
**Check**: Look for directories OUTSIDE the web project root (e.g., dedicated `scripts/`, `算法/`, `models/` directories) that contain scientific computation code. Compare against what the API actually exposes.
**Severity**: Critical — the platform's core scientific value proposition is not accessible through the platform itself
**Action**: Recommend an integration plan: wrap the algorithm in a service layer, expose via API, replace static images with dynamic computation results

### 32. Test files that cannot run in CI
**Found in**: Project F (the "test" file blocks on `input()`, hardcodes a wrong port, uses `requests` against a live server instead of the Flask test client, and has zero assert statements)
**Check**: Read test files and ask: "Can this run headlessly in CI without manual interaction?" Look for: `input()`, hardcoded URLs/ports, dependency on external services being up, no assert/expect statements.
**Severity**: Critical — the "tests" give false confidence while being completely non-functional

---

## Patterns Specific to Data Pipeline / Scientific Python Projects

These appear in projects that are primarily data processing pipelines (not web apps) — ETL + ML/statistical computation + JSON/CSV output, often with a thin FastAPI layer on top.

### 33. Written-but-never-wired module (dead code disguised as features)
**Found in**: Project G (a fully implemented evaluator module — holdout + rolling-origin evaluation, ~100 lines — that the pipeline runner never imports; a config flag even enables rolling evaluation; README claims evaluation metrics exist; the API response schema declares an `evaluation` field; zero eval output files on disk)
**Check**: For every module in a `pipeline/` or `modules/` directory, grep for its import across the project: `grep -rn "from.*module_name\|import.*module_name" --include="*.py" .` — if only the module file itself matches, it's dead code. Also: cross-check config flags (`ENABLE_* = True`) against the code that's supposed to consume them.
**Why it's Critical, not just Warning**: The feature is **claimed in documentation and API contracts** but doesn't exist in reality. For scientific/academic deliverables (软著, 横向课题), this is a credibility gap — a reviewer who tests the API evaluation endpoint gets empty/None and concludes the project is incomplete.
**Fix**: Wire the module into the orchestration entry point (usually ~4 lines: import + call inside the per-item processing function). Verify output files are generated.

### 34. Git repo tracking thousands of generated data files
**Found in**: Project G (thousands of git-tracked generated data files — several GB across a raw-data backup directory, pipeline output directories, and per-country JSON exports, plus a multi-MB raw-data CSV; `.gitignore` covered only trivia like `.DS_Store` and `__pycache__/`)
**Check**: `git ls-files | wc -l` — if > 1000 for a project that should be pure source code, investigate. Then: `git ls-files | sed 's|/.*||' | sort | uniq -c | sort -rn` to see which directories bloat the repo. Check `.gitignore` for data directories.
**Severity**: Warning (engineering hygiene) — bloats clone time, makes git operations slow, confuses 软著 source-code line counts with data-file counts, and risks committing sensitive raw data.
**Fix**: Add data directories to `.gitignore`, then `git rm -r --cached <dir>` for each. Keep a small sample or `.gitkeep` if the directory structure needs to exist. For 软著 submission, the repo should contain only human-authored source.

### 35. Deployment config with placeholder/obsolete values
**Found in**: Project G (the systemd service file had a literal placeholder working-directory; `ExecStart` used one port while the README documented another everywhere; `User=www-data` but actual deployment ran under a personal account)
**Check**: Read every systemd/Dockerfile/nginx config file and ask: "If someone copy-pastes this verbatim, does it work?" Grep for `/path/to/`, `change-me`, `example.com`, `localhost` in deployment files. Cross-reference ports/users/paths between deployment configs and application configs/README.
**Severity**: Critical — service won't start or starts on the wrong port, creating silent misconfiguration that only surfaces at deploy time.
**Common cause**: Config was written during prototyping, README updated later, nobody reconciled the two.

---

### 36. NaN values silently treated as zero in data pipelines
**Found in**: Project G (a data-loading step sums columns with pandas' default `skipna=True`, silently treating NaN as 0. The raw surveillance data is >70% NaN in one subtype column — those NaNs mean "not tested", not "zero positives", systematically overestimating positive rates for the affected subtypes)
**Check**: For any pandas `.sum()`, `.mean()`, `.fillna(0)` on surveillance/lab data, ask: "Does NaN mean 'zero' or 'not measured'?" If the latter, `skipna=True` (the pandas default) silently corrupts downstream calculations. Check: `df[columns].isna().mean()` — if NaN rate > 30% on a measurement column, it's a data quality bomb.
**Severity**: Critical — propagates through the entire pipeline (positive rate → MEM thresholds → forecasts → evaluations), silently biasing all results. Hardest to detect because the code "works" without errors.
**Fix**: Differentiate "all-NaN" rows from "some-NaN" rows. For all-NaN: keep as NaN (no data). For partial NaN: either interpolate or sum only non-NaN columns, but document the assumption.

--- 

## Patterns in Product Families (copy-paste codebases)

When a project is part of a product family (same codebase copied for different domains):

### 18. Shared bugs across siblings
If Product D1 has a path traversal in the snapshot-upload module, Product D2 and Product D3 almost certainly have the same bug — they were copy-pasted.
**Action**: After reviewing one product, proactively offer to check siblings. Use `grep` to verify the same code exists.
**File correspondence**: Line numbers may differ by a few lines due to product-specific additions, but the code pattern is identical.

### 19. Some siblings are more evolved than others
Product D3 had the `.value` fix and ECharts cleanup that Product D1 lacked — suggesting D3 was developed later or more carefully.
**Action**: When generating a shared fix plan, note which product has the "correct" implementation and use it as the reference template.

### 20. Fake i18n (not using real vue-i18n)
**Found in**: Project C (6 tool pages each implement fake `t()` with hardcoded translation tables, no vue-i18n installed). Product D1 has REAL vue-i18n — use as positive reference.
**Check**: `grep -rn "const t =\|function t(" frontend/src/` + check if `vue-i18n` is in package.json
**Contrast**: Product D1's real vue-i18n setup (complete locale files × 2 languages) is the gold standard among the reviewed projects.

---

## Patterns Specific to Flask / Traditional Server-Rendered Apps

These appear in Jinja2/template-based apps (not Vue/React SPAs):

### 21. Zero Jinja2 template inheritance — every page is a full HTML copy
**Found in**: Project E (every template — over a dozen — completely duplicates the head/nav/footer block, zero `{% extends %}`)
**Check**: `grep -rn "extends" templates/` — if empty, templates don't inherit
**Impact**: Any nav/CSS change requires editing N files
**Fix**: Extract `base.html` with `{% block %}` sections

### 22. SQLite + Gunicorn multi-worker write conflicts
**Found in**: Project E (4 sync workers + SQLite = `database is locked` time bomb)
**Check**: Read Dockerfile CMD / docker-compose for `--workers` > 1 + database URL containing `sqlite`
**Fix**: Either `--workers 1 --threads N` OR migrate to PostgreSQL. Also enable `PRAGMA journal_mode=WAL`.

### 23. Dual database access paths (ORM + raw driver)
**Found in**: Project E (SQLAlchemy ORM in some files, raw `sqlite3.connect()` in two service modules)
**Check**: `grep -rn "sqlite3.connect\|psycopg2.connect\|mysql.connect" --include="*.py"` in a project that also uses SQLAlchemy
**Impact**: Transaction isolation breaks, connection pool bypassed

### 24. Background tasks started in before_request hook
**Found in**: Project E (threading + file lock launched inside Flask `@app.before_request`)
**Check**: `grep -rn "before_request.*Thread\|before_request.*schedule" --include="*.py"`
**Impact**: Circular dependency (app → run → app), unpredictable start time, file lock residue after restart

### 25. Bootstrap version mismatch with JS polyfill
**Found in**: Project E (HTML uses BS5 `data-bs-*` attributes, but loads BS4 JS; a custom polyfill script bridges the gap)
**Check**: Compare `data-bs-toggle` in HTML vs bootstrap.js version in `<script>` src
**Fix**: Upgrade bootstrap.js to match, or change HTML to BS4 syntax

### 26. Global removal of keyboard focus (accessibility)
**Found in**: Project E (`outline: none !important` on all focusable elements in the main stylesheet)
**Check**: `grep -rn "outline.*none.*important" --include="*.css"`
**Severity**: Warning — WCAG 2.1 SC 2.4.7 violation

---

## Scoring Calibration (from 6 real projects)

| Project | Stack | Overall | Best Role | Worst Role | Key Pattern |
|---------|-------|---------|-----------|------------|-------------|
| Project A | FastAPI+Vue+SQLite | 5.4/10 | Backend 7/10 | QA 1.5/10 | Mock data in deliverables + frontend not in Docker |
| Project B | FastAPI+Celery+Vue+PG | 6.2/10 | PM/Arch 7.5/10 | QA 6.5/10 | Flow navigation breaks + dev-only Docker config |
| Project C | FastAPI+Vue+PG | 5.9/10 | PM 7.5/10 | QA 3 + Sec 4 | Design system excellent but 60 endpoints zero auth |
| Product D1 | FastAPI+Vue+SQLite | 6.0/10 | PM/Product 7.5/10 | Sec+Dev+QA 4.5 | Missing imports crash + path traversal + zero tests |
| Project E | Flask+Jinja2+SQLite | 5.4/10 | Product 8/10 | Sec+Dev+QA 4 | 4 hardcoded secrets + CSRF missing + XSS + 14 templates zero inheritance |
| Project F | Flask+Vue3(fantastic-admin)+SQLite | 3.4/10 | Sec 4.5 + Frontend 4/10 | QA 2/10 | Name≠function + pandas version doesn't exist + zero business API + template套壳 |
| Project G | Python pipeline+FastAPI (no frontend, no DB, JSON file store) | 5.0/10 (5-role) | PM+Backend 6.5/10 | DevOps 3.5/10 | Evaluator never wired + NaN→0 data bug + thousands of git-tracked data files + ADMIN_TOKEN fallback |
| Project H | ESP32-S3 firmware (Arduino) + Python FastAPI bridge + SDL2 simulator | 6.8/10 (5-role) | Architect 7.5/10 | Security 5/10 | WiFi password+API key in public repo + bridge zero auth + CPU% status guessing + delay() blocking |

---

## Patterns Specific to ESP32 / Embedded Firmware Projects

These appear in Arduino/ESP32 firmware + Python bridge companion apps — the "IoT device that controls PC software" pattern.

### 37. Hardcoded credentials in firmware committed to public repo
**Found in**: Project H (the firmware config header has `WIFI_PASSWORD = "<real password, redacted>"`; both companion services have hardcoded LLM API key fallback strings; the repo was public)
**Check**: `grep -rn "PASSWORD\|API_KEY\|SECRET\|SSID.*=" --include="*.h" --include="*.cpp" --include="*.py"` in firmware repos. Then verify repo visibility: `gh repo view --json visibility`
**Severity**: Critical — WiFi password and API keys are permanently leaked via Git history. Anyone who clones the repo has network access credentials.
**Fix**: ESP32: use `Preferences` (NVS) with a first-run BLE/WiFi config portal. Bridge: use `os.environ.get('KEY') or raise RuntimeError('KEY not set')` — no fallback defaults.
**IoT-specific note**: Unlike web apps where secrets can be rotated server-side, firmware credentials are baked into the binary. Rotating requires re-flashing every device.

### 38. Companion API with zero authentication on local network
**Found in**: Project H (the bridge's terminal-injection endpoint accepts POSTs with no auth — anyone on the same WiFi can send arbitrary commands to the coding agent)
**Check**: Read all API route definitions in the bridge/companion service. Look for any POST/PUT/DELETE without token/API-key/session validation.
**Severity**: Critical — on shared networks (office, conference, dormitory), this is a direct remote code execution vector.
**Fix**: Add a simple shared-token header (`X-Device-Token`) generated at pairing time and stored in both firmware NVS and bridge config.

### 39. Blocking calls in Arduino main loop
**Found in**: Project H (the WiFi-reconnect helper calls `delay()` for up to 60s, freezing the UI and missing all button events; an audio-upload routine in the main loop blocks ~5s)
**Check**: Read the `loop()` function and every function it calls. Look for `delay()`, `while()` without timeout, blocking `HTTPClient.GET()`.
**Severity**: Warning — during the block, the device is unresponsive (no button handling, no screen updates, no sensor reads). User perceives device as frozen.
**Fix**: Use non-blocking state machines: set a timer, check `millis()` each loop iteration, spread work across iterations. For WiFi: `WiFi.begin()` returns immediately, check `WiFi.status()` in loop.

### 40. Process CPU% used to infer application state (guessing instead of asking)
**Found in**: Project H (bridge monitors Claude Code by `pgrep -f claude` + `ps -o %cpu`, threshold 5% → "running", below → "idle". This misses thinking phases where CPU dips, and can't distinguish "waiting for user input" from "processing")
**Check**: Look for status detection via indirect OS metrics (CPU%, memory, network activity) when the target application has its own lifecycle hooks/events.
**Severity**: Warning — status display is unreliable, giving users false "idle" signals when the app is actually thinking, or false "running" when it's stuck.
**Fix**: Use the application's native instrumentation. For Claude Code: configure hooks in `~/.claude/settings.json` (UserPromptSubmit, PreToolUse, PostToolUse, Notification, Stop) that POST state changes to the bridge. Direct > inferred.

### 41. Device pulls from PC instead of PC pushes to device (network fragility)
**Found in**: Project H (ESP32 polls bridge every 1.5-3s via HTTP GET, requires knowing Mac's IP; mDNS helps but device must re-resolve after WiFi reconnect. If Mac changes IP (router restart, DHCP lease expiry), device can't find bridge until mDNS re-resolution)
**Check**: Identify the communication initiator. If the embedded device initiates all connections to a PC service, the device must know the PC's network address — creating an IP dependency.
**Severity**: Warning — not a bug, but an architectural fragility. Any IP change (new router, DHCP, different network) breaks the link.
**Fix**: Prefer push architecture (PC → device): WebSocket (device connects once, PC pushes updates) or BLE (connection-oriented, no IP needed). For WiFi: at minimum use mDNS (already a good practice if present). Best long-term: BLE for CC-related communication, WiFi only for independent cloud API calls.

### 42. Version-unlocked dependencies in companion service
**Found in**: Project H (both companion services ship `requirements.txt` with bare `fastapi` / `uvicorn` / `httpx`-style entries and no version pins)
**Check**: Read requirements.txt — any line without `==` version pin is a time bomb.
**Severity**: Warning — a breaking change in fastapi/uvicorn/httpx will cause mysterious failures months later when someone reinstalls.
**Fix**: `pip-compile` or manually pin all versions. For firmware: PlatformIO's `lib_deps` can also benefit from version tags.

**Observation**: QA and Security tend to be the lowest-scoring roles across ALL projects. This is not coincidence — testing and authorization are the most commonly deprioritized aspects in solo/small-team development.

**Score floor calibration**: Project F at 3.4/10 represents the "template + auth skeleton only, zero business logic" floor. If a project's core domain functionality doesn't exist yet, the score should not exceed 4 regardless of how polished the UI or auth module is.

**Data pipeline projects** (like Project G): no frontend/database/docker/CI means fewer surfaces to fail, so raw structural scores can reach 6-6.5. BUT the full 5-role review (adding Security + DevOps) revealed hidden depth: ADMIN_TOKEN hardcoded fallback, TRUNCATE without transaction, zero version locking, service file with placeholder path — these pulled the overall score to 5.0. **Lesson**: even "thin API" projects benefit from the full 8-role review. The "written-but-never-wired module" pattern (#33) and the "NaN-as-zero" pattern (#36) are both uniquely common in data pipelines and can silently invalidate claimed features.

**Overall score formula that emerged**:
- Start at 10
- Average all role scores
- If ANY Critical finding exists: cap overall at 7
- If ANY role scored ≤ 3: cap overall at 6 (one severely deficient area is a systemic risk)
- If 3+ Critical findings: cap at 5
- If core business logic is completely absent (only auth/scaffolding exists): cap at 4
- Final score = min(average, all applicable caps). **Show the computation in the report** — field lesson: the cap rule existed but was silently skipped in a real review (Project C v1 5.9 uncapped, v2 recomputed 5.0)
- The overall score should reflect "can this be delivered/maintained as-is?"

---

## Patterns from Delta Re-reviews (found on second look at "mature" projects)

These surfaced only when re-reviewing a project that had already been reviewed once — new code kept compounding old patterns in new forms. Read them even if the target project is not a web platform: the underlying trust-boundary mistake generalizes.

### 43. Generated artifact output directory overlaps a publicly-served static area
**Found in**: Project C v2 (the directory the auto-report module writes internal reports into sits INSIDE the directory mounted by a no-auth static file route, with a compose volume pinned to that same path. Every internal report the system generates — under a predictable filename scheme — becomes publicly readable at a predictable URL the moment it exists)
**Check**: Map every "artifact output directory" (generated reports, exports, uploads, PDFs) against every route that serves static files WITHOUT auth, plus every volume mount / volume path in docker-compose. They must not intersect — including via symlinks and volume mounts. Also check what filename scheme the generator uses: predictable names + public dir = enumerable leak even without a directory listing.
**Severity**: Critical when the artifacts are sensitive (internal reports, user data, anything behind login elsewhere in the app); Warning if artifacts are genuinely public.
**Fix**: Separate trust zones: generated sensitive artifacts go to a non-public directory (or object storage with signed URLs); the public static route serves only a whitelisted allow-list subdirectory; assert the separation at startup (fail fast if output dir is inside a public mount).
**Why reviews miss it**: each half looks fine in isolation — "auto_report writes reports" ✅, "/reports serves static files" ✅. Only the *overlap* is the vulnerability. Cross-check mounts and routes in the same breath.

### 44. Upload filename write-traversal combined with missing auth = unauthenticated arbitrary file write
**Found in**: Project C v2 (upload handlers concatenated the raw client-supplied filename into the stored path without sanitization — a traversal payload as the filename escapes the data directory. Same shape in three sibling modules, all on zero-auth routers)
**Check**: For every file upload endpoint: (1) is the route authenticated? (2) is the client-supplied filename sanitized with `Path(filename).name` (or an allowlist regex) BEFORE it is used in any path join? (3) is the final path re-validated inside the intended base dir after join?
**Severity**: Critical — read-side traversal (#2) leaks files; write-side traversal plants them. Combined with #1 (zero auth) it is unauthenticated remote file write = potential RCE (overwritable config, cron, static template).
**Fix**: `safe_name = f"{file_id}_{Path(filename).name}"` + resolve-and-fence the final path (same pattern as #2). Note the asymmetry: pattern #2's fix mentions filename sanitization, but WRITE traversal deserves its own check — reviewers scanning for "file serving" miss upload handlers.
**Sibling propagation**: this shape copied itself across 3 modules in one project. When found once, grep the whole repo for the concatenation idiom (`f"{...}_{filename}"`, `filename` from `UploadFile`).
