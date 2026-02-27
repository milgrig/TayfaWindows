# Tayfa Project Audit — 2026-02-18

Full audit of architecture, code quality, UX, and processes.
Prioritized by impact. Quick wins marked with (*).

---

## CRITICAL

### 1. No file-level locking on tasks.json
**Area:** task_manager.py
**Problem:** Every write is a full read-modify-write with no lock. Two agents finishing simultaneously can silently overwrite each other's status update. Same for backlog.json, agent_failures.json.
**Fix:** Add `asyncio.Lock` at the app layer for serializing task writes; or switch to SQLite.

### 2. No authentication on API
**Area:** app.py, claude_api.py
**Problem:** All endpoints (kill agents, shutdown server, git commit, task management) are completely unauthenticated. Server binds to `0.0.0.0` — reachable from local network.
**Fix:** Add at minimum a token-based auth middleware; bind to `127.0.0.1` by default.

### 3. `os._exit(0)` bypasses cleanup on shutdown
**Area:** app.py lines 524, 1041
**Problem:** Skips all `finally` blocks, `atexit` handlers, and FastAPI lifespan teardown. Claude API subprocess may be left orphaned.
**Fix:** Use `uvicorn` shutdown mechanism or `sys.exit()`.

### 4. Sprint lifecycle has no "cancelled" state
**Area:** task_manager.py
**Problem:** `SPRINT_STATUSES = ["active", "completed", "released"]`. No way to cancel or abort a sprint. Stuck sprints have no recovery path. No git branch cleanup.
**Fix:** Add `"cancelled"` and `"paused"`. Implement `cancel_sprint()`.

---

## HIGH

### 5. (*) Developer prompt.md uses Russian status names
**Area:** .tayfa/developer/prompt.md
**Problem:** Contains `--status в_работе` and `status T003 на_проверке`. task_manager.py only accepts English (`pending`, `in_progress`, `in_review`). Agent CLI calls fail silently.
**Fix:** 5 min. Search all prompts for Russian status strings, replace with English.

### 6. (*) run_tests.sh referenced at wrong path
**Area:** agent-base.md, tester_checklist.md
**Problem:** Tell testers to run `bash ./run_tests.sh` but file is at `scripts/run_tests.sh`. Every tester gets "file not found".
**Fix:** 2 min. Add root-level wrapper or update references.

### 7. Dependency enforcement absent at trigger time
**Area:** app.py `api_trigger_task`
**Problem:** Tasks have `depends_on` but trigger never checks if dependencies are `done`. Any task can start out of order. Finalize task can run before other tasks complete.
**Fix:** Check all `depends_on` are `done`/`cancelled` before triggering. Return HTTP 409 if blocked.

### 8. `git add -A` commits everything blindly
**Area:** app.py `_perform_auto_commit`
**Problem:** On every `in_review` transition, stages ALL working directory changes. Will silently commit unrelated work, debug logs, potentially secrets.
**Fix:** Commit only files declared in task's `files_changed`, or at least log what's being staged.

### 9. `_run_claude` silently swallows non-zero return codes
**Area:** claude_api.py
**Problem:** When Claude CLI exits with error but produces parseable JSON, returns as "success". `api_trigger_task` never checks return code — shows success to user even on agent failure.
**Fix:** Check `result["code"] != 0` and surface the error.

### 10. `ensure_agents` is 200 lines doing 3+ things
**Area:** app.py lines 1608-1775
**Problem:** Handles request parsing, project resolution, agent listing, workdir fixing, prompt migration, and agent creation in one function.
**Fix:** Extract `_fix_agent_workdir()`, `_migrate_agent_prompt()`, `_create_agent()`.

### 11. `api_trigger_task` is 250 lines with all logic inline
**Area:** app.py lines 2270-2517
**Problem:** Loop detection, prompt assembly, runtime dispatch, retry logic, failure logging, chat saving, artifact checking — all inline.
**Fix:** Extract `_build_task_prompt()`, `_execute_with_retry()`, etc.

---

## MEDIUM — Frontend

### 12. Zero accessibility
**Area:** index.html
**Problem:** Not a single `aria-` attribute, no `role=`, no focus trapping in modals, no keyboard navigation on sidebar.
**Fix:** Add `role="dialog"`, `aria-modal`, `aria-labelledby`, focus trapping to modals. Add `role="navigation"` to sidebar.

### 13. 50+ native `alert()`/`confirm()` calls
**Area:** index.html
**Problem:** Blocks UI thread, can't be styled. App already has `openModal()` and `showGitToast()`.
**Fix:** Replace all with existing modal/toast system. Purely mechanical.

### 14. Five simultaneous polling timers, 1-second poll never stops
**Area:** index.html
**Problem:** `globalRunningPollTimer` (1s), `boardAutoRefreshTimer` (5s), `checkStatus` (10s), `gitAutoRefreshTimer` (30s), ping (5s). The 1s poll runs forever even with zero running tasks. No visibility-based pausing.
**Fix:** Reduce to 5s, stop when `runningTasks` is empty. Pause all polls when tab not visible.

### 15. Full DOM rebuild on every board refresh
**Area:** `refreshTasksBoardNew()`
**Problem:** Tears down and rebuilds entire kanban via `innerHTML`. Destroys running elapsed timers, causes visual flashes.
**Fix:** Diff-based update or at least preserve DOM nodes that haven't changed.

### 16. Three near-identical date formatting functions
**Area:** `formatLastOpened()`, `formatDate()`, `formatGitTime()`
**Fix:** Merge into one `relativeTime(date)` function.

### 17. No loading/disabled state on git buttons
**Problem:** Double-clicks send duplicate push/commit/PR requests.
**Fix:** Disable button + show spinner during async operations.

---

## MEDIUM — Backend/Workflow

### 18. Backlog items permanently destroyed on sprint import
**Area:** task_manager.py
**Problem:** `create_sprint(include_backlog=True)` deletes `next_sprint` items from backlog. If sprint abandoned, items are lost forever.
**Fix:** Mark items as `promoted_to: T0xx` instead of deleting. Add `demote_task_to_backlog()`.

### 19. sprint-archive.md describes archival that doesn't exist
**Problem:** Rules describe `task_manager.py archived` CLI. Not implemented. tasks.json grows unboundedly.
**Fix:** Implement `archive_sprint()`, call at finalize completion.

### 20. Loop detection resets tasks to "pending" — invisible as blocked
**Area:** app.py
**Problem:** When loop fires, task goes to `pending` — indistinguishable from new task. Loop message in `result` will be overwritten by next agent.
**Fix:** Add `blocked` status, or write loop message to discussion file where it persists.

### 21. Temp file round-trip for system prompts is a no-op
**Area:** claude_api.py lines 301-322
**Problem:** Writes prompt to temp file, reads it back, passes as CLI argument. The temp file solves nothing — string still goes as argv. Windows cmd limit not actually fixed.
**Fix:** Use stdin pipe or `--system-prompt-file` flag if Claude CLI supports it.

### 22. `call_claude_api` creates new HTTP client on every call
**Area:** app.py line 608
**Problem:** No connection pooling. New TCP connection per request.
**Fix:** Use a shared `httpx.AsyncClient` managed via FastAPI lifespan.

### 23. `_classify_error` uses fragile string matching
**Problem:** Relies on substrings like `"overflow"`, `"budget"` in error text. Will break if Claude API changes messages.
**Fix:** Classify by HTTP status codes and structured error fields.

### 24. Debug logging permanently baked in
**Area:** app.py, claude_api.py
**Problem:** `debug-6f4251.log` written on every agent operation. Not gated by any flag. Hardcoded session ID.
**Fix:** Remove entirely or gate behind `DEBUG` env var.

### 25. No path-traversal protection in project_manager.py
**Problem:** `init_project()` accepts any path, copies template there.
**Fix:** Reject paths with `..` or inside the app directory.

---

## LOW — Missing Features

### 26. No task editing after creation
Can't edit title, description, assignees, sprint, or dependencies once task exists.

### 27. No sprint editing
Can't rename, change description, or move tasks between sprints.

### 28. No task detail/expand view
Results truncated at 200 chars, descriptions at 120 chars, no expand action.

### 29. No search or filter on kanban board
Backlog has filters, but main task board has none.

### 30. No undo for destructive actions
Delete backlog, cancel task, delete agent, clear chat — all irreversible.

### 31. No keyboard shortcuts
Only Enter to send chat. No shortcuts for agent switching, view nav, refresh.

### 32. No structured handoff validation
`HANDOFF_SCHEMA` defined in claude_api.py but `api_trigger_task` never uses `use_structured_output=True`. Agent decisions are unstructured text.

### 33. No agent health/liveness checks
Only a binary "API responding" check. No per-agent health, no detection of permanently stuck agents.

### 34. No queue/concurrency limit for agent executions
Multiple tasks launch simultaneously with no global limit, no queue, no back-pressure.

### 35. No audit trail for agent decisions
Failures logged, but no persistent record of status transitions agents requested. Chat history is not queryable for orchestration decisions.

### 36. Backlog has no accept/reject workflow
Flat list with `priority` and `next_sprint` boolean. No `status`, `reviewed_by`, `review_note` fields.

### 37. No CI/CD pipeline
Rules describe GitHub Actions but no `.github/workflows/` exists.

---

## Quick Wins — Top 10 (do first)

| # | What | Time | Impact |
|---|------|------|--------|
| 1 | Fix Russian status names in developer/prompt.md | 5 min | HIGH — agents failing silently |
| 2 | Fix run_tests.sh path in rules + checklist | 2 min | HIGH — testers always fail |
| 3 | Add dependency check to `api_trigger_task` | 20 min | HIGH — tasks can't start out of order |
| 4 | Add "cancelled" to SPRINT_STATUSES | 20 min | CRITICAL — stuck sprints recoverable |
| 5 | Replace `alert()`/`confirm()` with modal/toast | 1 hr | MEDIUM — UX dramatically better |
| 6 | Stop 1s poll when no running tasks | 10 min | MEDIUM — CPU/network savings |
| 7 | Remove debug-6f4251.log instrumentation | 15 min | MEDIUM — clean up noise |
| 8 | Bind server to 127.0.0.1 | 2 min | HIGH — basic security |
| 9 | Add asyncio.Lock for task writes | 30 min | CRITICAL — data integrity |
| 10 | Check `_run_claude` return code in trigger | 15 min | HIGH — stop silent failures |
