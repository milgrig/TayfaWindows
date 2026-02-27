# Project Audit — TayfaWindows
**Date**: 2026-02-18 | **Author**: developer

---

## Executive Summary

Tayfa — multi-agent AI orchestration system (FastAPI + Claude Code CLI + Web UI). ~37K lines of Python, 48 API endpoints, 12 agents, file-based state (JSON + Markdown). The system is functional and well-architected at the conceptual level, but has accumulated technical debt that will hurt reliability, maintainability, and scalability.

Below: **categorized findings** with severity, effort estimates, and concrete next steps.

---

## 1. ARCHITECTURE — God Object & Monolith

### Problem
`kok/app.py` is **2,694 lines** with **106 functions** and **75+ endpoints**. It handles routing, business logic, process management, config, error handling, and more. This is the #1 maintainability risk.

### Recommendation
Split into FastAPI routers:

| New Module | Endpoints | Est. Lines |
|---|---|---|
| `api/agents.py` | /agents, /create-agent, /kill-agents, /ensure-agents | ~400 |
| `api/tasks.py` | /tasks-list/*, /running-tasks, /trigger | ~500 |
| `api/sprints.py` | /sprints/*, /release-ready, /report | ~300 |
| `api/projects.py` | /projects/*, /current-project, /browse-folder | ~200 |
| `api/chat.py` | /send-prompt, /chat-history/* | ~200 |
| `api/settings.py` | /settings, /backlog/*, /employees/* | ~200 |
| `core/errors.py` | Error classification, failure logging | ~150 |
| `core/config.py` | All config reading, defaults, validation | ~100 |

**Effort**: ~2 sprints | **Impact**: HIGH — enables parallel development, testing, code review

---

## 2. CODE QUALITY — Critical Issues in app.py

### 2.1 Blocking call in async context (CRITICAL)
**Line ~1260**: `subprocess.run()` (synchronous) inside async endpoint `api_browse_folder()`. Blocks the entire event loop.

**Fix**: Replace with `asyncio.create_subprocess_exec()`.

### 2.2 Non-thread-safe global state (HIGH)
```python
running_tasks: dict = {}          # Line 299
task_trigger_counts: dict = {}    # Line 303
```
These are mutated from async handlers without locks. Race conditions possible under concurrent requests.

**Fix**: Use `asyncio.Lock()` around mutations, or replace with a proper state store.

### 2.3 Silent exception swallowing (HIGH)
**40+ bare `except Exception: pass`** blocks throughout app.py. Examples:
- Line 27, 95, 328, 418, 599, 888, 1619 — silently discard errors
- Many have no logging at all

**Fix**: Replace with specific exception types + `logger.exception()`.

### 2.4 Oversized functions (HIGH)
| Function | Lines | Issue |
|---|---|---|
| `api_trigger_task` | 250 | Does: validation, loop detection, prompt building, API call, retry, error logging, result save |
| `ensure_agents` | 173 | Does: list agents, compare with employees, update/create/delete agents |

**Fix**: Extract sub-functions: `_validate_trigger()`, `_build_prompt()`, `_handle_retry()`, etc.

### 2.5 Code duplication (MEDIUM)
- Agent payload construction repeated ~4 times (lines 1665, 1700, 1736, 2446)
- Claude API call + error handling pattern duplicated ~6 times
- Path construction logic scattered throughout

**Fix**: Extract `_build_agent_payload()`, `_safe_claude_api_call()` helpers.

### 2.6 Hardcoded values (MEDIUM)
- `CURSOR_CLI_TIMEOUT = 600.0`, `SHUTDOWN_TIMEOUT = 120.0`, `MAX_FAILURE_LOG_ENTRIES = 1000`
- UUID detection via magic: `len(line) == 36 and line.count("-") == 4`

**Fix**: Move to config.json or constants module.

---

## 3. TEST COVERAGE — Major Gaps

### Current State
- **7 test files**, **130 tests passing**, **~2,100 lines**
- **Only ~8 of 48 endpoints** have any test coverage
- Tests are heavily mocked — almost no real integration tests

### Untested Critical Functionality (0% coverage)

| Area | Endpoints | Risk |
|---|---|---|
| Task CRUD & lifecycle | /tasks-list/* (8 endpoints) | CRITICAL |
| Agent communication | /send-prompt, /send-prompt-cursor | CRITICAL |
| Backlog | /backlog/* (5 endpoints) | HIGH |
| Employees | /employees/* (3 endpoints) | HIGH |
| Chat history | /chat-history/* (2 endpoints) | MEDIUM |
| Settings | /settings (2 endpoints) | MEDIUM |
| Projects | /projects/* (6 endpoints) | MEDIUM |
| Server lifecycle | /start-server, /stop-server, /status | LOW |

### Test Quality Issues
1. **No conftest.py** — fixtures duplicated across files
2. **State leak** between tests (e.g. `task_trigger_counts` — fixed in T043)
3. **Over-mocking** — tests verify mock behavior, not real behavior
4. **No negative tests** — no tests for 400, 404, 500 responses
5. **E2E test not marked** — `test_t036_draft_persistence.py` lacks `@pytest.mark.e2e`

### Recommendation
Priority test additions:
1. **P0**: Task trigger pipeline (trigger → agent → result → status change)
2. **P0**: Backlog CRUD operations
3. **P1**: Employee registration + agent creation flow
4. **P1**: Create shared `conftest.py` with common fixtures
5. **P2**: Settings persistence tests
6. **P2**: Add negative test cases for input validation

---

## 4. FRONTEND — Monolithic HTML

### Problem
`kok/static/index.html` is a **~55KB single file** with inline HTML + CSS + JS. No framework, no bundling, no frontend tests.

### Recommendation
Short-term (low effort):
- Extract JS into `static/app.js`, CSS into `static/style.css`
- Add minification step

Medium-term:
- Evaluate lightweight framework (Alpine.js or Vue 3 for reactivity)
- Add Playwright E2E tests for critical UI flows

---

## 5. SECURITY & SECRETS

### 5.1 Plaintext GitHub token (CRITICAL)
`kok/secret_settings.json` stores GitHub token in plaintext. Read on every git operation.

**Fix**: Use environment variable `GITHUB_TOKEN` or OS keyring. At minimum, encrypt at rest.

### 5.2 No input validation (MEDIUM)
Most endpoints accept arbitrary JSON without schema validation. DoS via oversized inputs possible.

**Fix**: Add Pydantic request models for all POST/PUT endpoints.

### 5.3 Shell injection risk (MEDIUM)
Cursor CLI integration builds shell commands with string interpolation (lines 843-848).

**Fix**: Use subprocess array form consistently, avoid shell=True.

### 5.4 File path traversal (LOW)
Config file reading doesn't validate paths are within expected directories.

---

## 6. INFRASTRUCTURE & DevOps

### 6.1 No Docker support (HIGH)
No Dockerfile, no docker-compose. Manual setup only.

**Fix**: Create `Dockerfile` + `docker-compose.yml` for reproducible deployment.

### 6.2 Log rotation missing (HIGH)
`tayfa_server.log` grows unbounded (currently 2.8MB). No rotation configured.

**Fix**: Use `logging.handlers.RotatingFileHandler` with max 5MB × 3 backups.

### 6.3 In-memory state lost on restart (HIGH)
`running_tasks` and `task_trigger_counts` are in-memory dicts. Server restart = lost state.

**Fix**: Persist to JSON file or SQLite. Load on startup.

### 6.4 No structured logging (MEDIUM)
All logs are plain text. Hard to parse, filter, or forward to monitoring.

**Fix**: Use `python-json-logger` for structured JSON logs.

### 6.5 Unpinned dependencies (MEDIUM)
`requirements.txt` uses `>=` without upper bounds. `fastapi>=0.115.0` allows any future version.

**Fix**: Pin exact versions for production: `fastapi==0.115.1`. Use `pip-compile` for lock file.

### 6.6 CI pipeline gaps (LOW)
- No E2E tests in CI
- Linting is non-blocking (`--exit-zero`)
- No security scanning (bandit, safety)
- No Docker build in pipeline

---

## 7. DATA INTEGRITY

### 7.1 No file locking (HIGH)
`tasks.json`, `backlog.json`, `employees.json` — all read/written without file locks. Concurrent writes corrupt data.

**Fix**: Use `fcntl.flock()` (Unix) or `msvcrt.locking()` (Windows), or migrate to SQLite.

### 7.2 Git transaction safety (MEDIUM)
Sprint release: merge to main can succeed but push can fail. Partial rollback exists but is incomplete.

**Fix**: Implement full rollback: if push fails, revert merge and restore branch state.

### 7.3 Chat history unbounded (LOW)
500 messages per agent × 12 agents = growing storage. No compression, no archival.

---

## 8. TASK MANAGER IMPROVEMENTS

### 8.1 Git logic in task_manager.py (MEDIUM)
`task_manager.py` (1,288 lines) mixes task CRUD with git operations (branch creation, merging, tagging). Should delegate to `git_manager.py`.

### 8.2 Hardcoded path traversal (LOW)
`TASKS_FILE.parent.parent.parent` — fragile, breaks if directory structure changes.

### 8.3 No task archival (LOW)
Completed tasks stay in `tasks.json` forever. Eventually slows down reads.

---

## 9. PRIORITIZED ROADMAP

### Immediate (this week) — Safety & Stability
| # | Item | Effort | Impact |
|---|---|---|---|
| 1 | Move GitHub token to env variable | 1h | CRITICAL security |
| 2 | Fix blocking subprocess.run in async (line ~1260) | 30m | CRITICAL reliability |
| 3 | Add log rotation | 30m | HIGH ops |
| 4 | Add asyncio.Lock for running_tasks | 1h | HIGH reliability |

### Short-term (1-2 sprints) — Quality & Testing
| # | Item | Effort | Impact |
|---|---|---|---|
| 5 | Create conftest.py + shared fixtures | 2h | HIGH testing |
| 6 | Add tests for task trigger pipeline | 4h | HIGH coverage |
| 7 | Add tests for backlog CRUD | 2h | HIGH coverage |
| 8 | Replace bare except blocks (top 20) | 3h | HIGH reliability |
| 9 | Extract 2 largest functions into sub-functions | 3h | HIGH maintainability |
| 10 | Add Pydantic request models for key endpoints | 4h | MEDIUM security |

### Medium-term (2-4 sprints) — Architecture
| # | Item | Effort | Impact |
|---|---|---|---|
| 11 | Split app.py into FastAPI routers | 8h | HIGH maintainability |
| 12 | Persist running_tasks/trigger_counts to disk | 3h | HIGH reliability |
| 13 | Add file locking to JSON state files | 4h | HIGH data integrity |
| 14 | Create Dockerfile + docker-compose | 4h | HIGH deployment |
| 15 | Add structured logging (JSON) | 3h | MEDIUM observability |

### Long-term (future sprints) — Scale & Polish
| # | Item | Effort | Impact |
|---|---|---|---|
| 16 | Frontend: extract JS/CSS, add framework | 8h | MEDIUM UX |
| 17 | Migrate state to SQLite | 16h | MEDIUM scale |
| 18 | Add Prometheus metrics | 4h | MEDIUM observability |
| 19 | Implement full git transaction rollback | 4h | MEDIUM reliability |
| 20 | Add E2E tests to CI pipeline | 4h | MEDIUM quality |

---

## 10. POSITIVE OBSERVATIONS

The project has strong fundamentals worth preserving:

1. **Clean agent architecture** — file-based communication is simple, transparent, debuggable
2. **Well-defined task state machine** — clear status flow with role separation
3. **Discussion system** — excellent audit trail for agent decisions
4. **Sprint workflow** — automatic branch/tag/release management
5. **Error classification** — `_classify_error()` with retry logic is well thought out
6. **Atomic file writes** in chat_history_manager (temp file + rename)
7. **Minimal dependencies** — only 7 packages, no bloat
8. **Theme support** — 4 themes including the iconic "girly" 🎀
9. **Auto-shutdown** — browser tab close detection prevents orphan processes
10. **Artifact size limit** (T043) — proactive task decomposition rule

---

*This audit is a snapshot. Recommend re-running after each major refactoring sprint.*
