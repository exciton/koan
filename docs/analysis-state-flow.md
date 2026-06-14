# Kōan — State Flow Analysis

A comprehensive map of every code path from input to output, documenting every state change at every step, including both happy paths and error/crash paths.

---

## 1. State Inventory

### 1.1 In-Memory State (process-local, lost on crash)

**run.py:**
| Variable | Type | Location | Purpose |
|----------|------|----------|---------|
| `_last_mission_timed_out` | `bool` | `run.py` module | Set by `ProcessWatchdog`; read by `run_claude_task` to set exit_code=1 |
| `_last_mission_aborted` | `bool` | `run.py` module | Set by SIGUSR1 handler; read by `run_claude_task` |
| `_last_mission_stagnated` | `threading.Event` | `run.py` module | Written by stagnation monitor daemon thread; read/cleared by `_finalize_mission` |
| `_stagnation_pattern_type` | `str` | `run.py` module | Pattern classification from stagnation monitor |
| `_stagnation_pattern_excerpt` | `str` | `run.py` module | Sample text from stagnation monitor |
| `_startup_notified` / `_boot_notified` | `bool` | `run.py` module | Prevent duplicate startup Telegram bursts |
| `_sig: SignalState` | class | `run.py` module | Double-tap CTRL-C state: `task_running`, `first_ctrl_c`, `claude_proc`, `phase` |
| `count` | `int` | `main_loop()` | Run counter since last resume; reset to 0 on pause/resume |
| `consecutive_errors` | `int` | `main_loop()` | Consecutive iteration failures; triggers pause at `max_consecutive_errors` |
| `consecutive_idle` | `int` | `main_loop()` | Consecutive idle iterations; triggers auto-pause at `MAX_CONSECUTIVE_IDLE` (30) |
| `_warned_missing_projects` | `set` | `run.py` module | One-time warning suppression for missing project paths |

**mission_executor.py (called from main_loop via _run_iteration):**

`mission_executor._run_iteration()` is the per-iteration execution entry point. It owns the dedup guard, git prep, `_start_mission_in_file`, skill dispatch routing, Claude subprocess invocation (via `run.run_claude_task`), retry logic (`_maybe_retry_mission`), and the post-mission pipeline call. It holds no persistent module-level state — all mutations go through shared signal files or `run.py` module-level variables.

**awake.py:**
| Variable | Type | Purpose |
|----------|------|---------|
| `_last_update_id` | `int` | Highest Telegram update ID consumed; prevents re-processing |
| `_pending_missions` | `dict` | In-flight async worker threads keyed by task ID |

**iteration_manager.py (module-level):**
| Variable | Type | Purpose |
|----------|------|---------|
| `_branch_saturated_logged` | `set` | Projects already logged as branch-saturated this session |
| `_no_github_url_logged` | `set` | Projects already logged as lacking `github_url` |
| `_pr_limited_logged` | `set` | Projects already logged as PR-limited |

### 1.2 Markdown State Files (in `instance/`)

| File | Writers | Readers | Purpose |
|------|---------|---------|---------|
| `missions.md` | run.py, awake.py (both via `modify_missions_file`) | Both processes | Mission queue: CI / Pending / In Progress / Done / Failed / Ideas sections |
| `outbox.md` | run.py (`append_to_outbox`), awake.py | awake.py outbox flusher | Queued Telegram messages; consumed by background flusher thread |
| `journal/pending.md` | run.py (`create_pending_file`), skill subprocess stream | Agent (Claude CLI) reads it | Checkpoint/recovery context injected before each mission |
| `memory/global.md` | Claude agent | Claude agent, context builder | Global project summary |
| `memory/projects/{name}/learnings.md` | `pr_review_learning.py` (Claude) | Context builder | Per-project learnings from PR reviews |

### 1.3 Signal / Flag Files (in `$KOAN_ROOT/`)

All signal file name constants live in `koan/app/signals.py` — a centralized registry. Import from there rather than hardcoding names.

**Process lifecycle:**

| File | Written by | Read/consumed by | Semantics |
|------|-----------|-----------------|-----------|
| `.koan-pause` | `pause_manager.create_pause()`, quota handler, error handler | `run.py` top of every iteration | Pause execution. 3-line format: reason/reset_ts/display |
| `.koan-stop` | `make stop`, awake.py `/stop` | `run.py` top of iteration (consumed) | One-shot stop of agent loop |
| `.koan-shutdown` | awake.py `/shutdown` | run.py AND awake.py (consumed by run.py) | Stop both processes |
| `.koan-restart-bridge` | `restart_manager.request_restart()` | awake.py (consumed) | Bridge restarts via `os.execv()` |
| `.koan-restart-run` | `restart_manager.request_restart()` | run.py (consumed) | Run loop restarts via `sys.exit(42)` + `os.execv()` |
| `.koan-restart` | `restart_manager.request_restart()` | (legacy, written for compat) | Legacy combined restart signal; no active consumer |
| `.koan-cycle` | awake.py `/update` command | run.py top of iteration (consumed) | Trigger update after current mission completes |
| `.koan-abort` | awake.py `/abort`, SIGUSR1 handler | run.py poll loop inside `run_claude_task` | Kill current Claude subprocess |
| `.koan-reset-counter` | awake.py `/reset` | run.py top of iteration (consumed) | Reset `count` to 0 |

**Pause / quota:**

| File | Written by | Read/consumed by | Semantics |
|------|-----------|-----------------|-----------|
| `.koan-quota-reset` | legacy quota path | `command_handlers` (legacy fallback read) | Superseded by `.koan-pause` with reason="quota"; kept for backward compat |
| `.koan-skip-start-pause` | awake.py `/resume` auto-restart path | `startup_manager.handle_start_on_pause()` | Skip the 30s startup pause; consumed at startup |

**Mode flags:**

| File | Written by | Read/consumed by | Semantics |
|------|-----------|-----------------|-----------|
| `.koan-focus` | `focus_manager.create_focus()` (awake.py `/focus`) | `iteration_manager._check_focus()` | Focus mode: no contemplation, mode capped at implement. JSON format |
| `.koan-passive` | `passive_manager.create_passive()` (awake.py `/passive`) | `iteration_manager._check_passive()` | Passive mode: no execution. JSON format |
| `.koan-verbose` | awake.py `/verbose` | `prompt_builder` | Enables verbose mode section in agent system prompt |

**Status / heartbeat:**

| File | Written by | Read/consumed by | Semantics |
|------|-----------|-----------------|-----------|
| `.koan-status` | `run.py set_status()` (written every state change) | awake.py (`/status`), dashboard, `agent_state.get_agent_state()` | Human-readable current loop status |
| `.koan-project` | `run.py` startup + project rotation | `run.py _read_current_project()`, `agent_state` | Currently-active project name |
| `.koan-heartbeat` | bridge (awake.py) | health checker | Bridge liveness signal |
| `.koan-run-heartbeat` | run.py | health checker | Runner liveness signal |
| `.koan-daily-report` | `daily_report` module | startup, scheduler | Trigger / record daily report generation |
| `.koan-debug.log` | debug module | operator | Debug output log |

**Notification signals:**

| File | Written by | Read/consumed by | Semantics |
|------|-----------|-----------------|-----------|
| `.koan-check-notifications` | `github_webhook.py` | `loop_manager._consume_check_notifications_signal()` | Force immediate GitHub notification poll |

**Misc:**

| File | Written by | Read/consumed by | Semantics |
|------|-----------|-----------------|-----------|
| `.koan-onboarding.json` | onboarding module | onboarding module | Onboarding state tracking |
| `.koan-last-cleanup` | startup_manager cleanup | startup_manager | Throttle marker for per-startup cleanup (24h default) |

**Process instance locks** (named `.koan-pid-<name>` via `signals.pid_file()`):

| File | Written by | Read/consumed by | Semantics |
|------|-----------|-----------------|-----------|
| `.koan-pid-run` / `.koan-pid-awake` / etc. | `pid_manager.acquire_pidfile()` | `pid_manager.check_pidfile()` | Exclusive flock-based process instance locks |

**E-stop state** (not yet wired into the main loop; exists as infrastructure):

| File | Written by | Read/consumed by | Semantics |
|------|-----------|-----------------|-----------|
| `.koan-estop` | `estop_manager.activate_estop()` | (not yet consumed by run.py/mission_executor) | Emergency stop gate signal (existence = stopped) |
| `.koan-estop-state` | `estop_manager.activate_estop()` | `estop_manager.get_estop_state()` | Rich JSON e-stop state: level (full vs project_freeze), frozen project list, reason |

### 1.4 JSON State Files (in `instance/`)

| File | Purpose | Writer | Atomic? |
|------|---------|--------|---------|
| `.burn-rate.json` | Rolling 20-sample burn-rate buffer with `last_warned_at` | `burn_rate.record_run()` | Yes (fcntl.flock LOCK_EX) |
| `.ci-dispatch-tracker.json` | CI fix mission dedup keyed by `pr:sha:job:run_id` | `ci_dispatch` | Yes |
| `.review-dispatch-tracker.json` | PR review comment fingerprint dedup, per-project cooldown | `review_comment_dispatch` | Yes |
| `.stagnation-retries.json` | Per-mission stagnation retry counter, keyed by SHA-256 of stripped mission title | `stagnation_monitor` | Yes (locked JSON modify) |
| `.branch-cleanup-tracker.json` | Per-project branch cleanup cooldown (default 24h) | `git_sync` | Yes |
| `.commit-tracker.json` | Koan's own HEAD SHA across restarts (for startup diff) | `auto_update` | Yes |
| `.head-tracker.json` | Remote HEAD branch change detection, 12h throttle | `head_tracker` | Yes |
| `.api-missions.json` | REST API mission sidecar index | `api/mission_index.py` | Yes (`atomic_write_json`) |
| `.diagnostic-cooldowns.json` | Per-project autonomous health diagnostic cooldown | `iteration_manager` | Yes |
| `.selection-audit.json` | Thompson Sampling project-selection ring buffer | `iteration_manager` | Yes |
| `.telegram-offset.json` | Persisted Telegram poll offset; survives bridge restarts | `awake._save_offset()` | Yes |
| `usage_state.json` | Token accumulator for usage estimation | `usage_estimator` | Yes |
| `session_outcomes.json` | Per-project session outcome history | `session_tracker` | Varies |
| `recovery.jsonl` | Recovery event audit log | `recover.py` | Locked append (`locked_jsonl_append`) |
| `journal/checkpoints/<hash>.json` | Structured mission progress checkpoint: branch, steps_done/remaining, timestamps. Created when mission starts, updated from `CHECKPOINT:` stdout markers and pending.md, deleted on clean completion. Used by `recover.py` to inject structured recovery context. | `checkpoint_manager` | Yes (`atomic_write`) |

### 1.5 Git / GitHub External State

- **Branches**: `koan/<branch>` branches in project repos
- **GitHub PRs**: Open/closed/merged PR objects with head SHA
- **GitHub reactions**: Emoji reactions on @mention comments (dedup marker for `check_already_processed()`)
- **GitHub review threads**: Resolved/unresolved inline review threads
- **GitHub check runs**: CI pass/fail status per commit SHA

---

## 2. Telegram Message to Mission

### 2.1 Happy Path — User Queues a Mission

```
Step 1: User sends Telegram message (plain text, not /command)
  State change: Telegram server stores update (unread)

Step 2: awake.py polls Telegram API every 3s
  State change: (in-memory) _last_update_id examined

Step 3: Message classified as "mission" (text content, not /command prefix)
  State change: (none yet)

Step 4: command_handlers.handle_mission() called
  State change: missions.md[Pending] += "- <text> ⏳(timestamp)" entry
              (via utils.insert_pending_mission -> modify_missions_file
               -> fcntl.flock LOCK_EX + threading.Lock + atomic_write)
  Note: sanitize_mission_text() collapses newlines so multi-line messages
        become single list items.

Step 5: Telegram update offset acknowledged (_last_update_id = update_id + 1)
  State change: (in-memory; Telegram server marks update as read)

Step 6: Reply sent to user ("Mission queued" or similar)
  State change: outbox.md += reply (or direct send via send_telegram)
```

**Failure: missions.md write fails**

```
Step 4 fails: OSError from modify_missions_file
  State change: missions.md NOT modified (atomic write uses temp file + os.replace;
                incomplete writes leave original untouched)
  Action: Exception caught, error logged, user notified via Telegram reply
  Note: Mission is LOST. User must re-send. No retry mechanism.
```

### 2.2 Chat Message (Instant Reply)

```
Step 1-2: Same as above.

Step 3: Message classified as "chat" (question to bot, not a task)

Step 4: awake.py builds context prompt (journals, memory, missions state)
  State change: READ-ONLY (no mutations to any file)

Step 5: format_and_send() -> Claude CLI invoked
  State change: (Claude subprocess; no persistent state mutations)

Step 6: Reply sent via send_telegram()
  State change: Telegram message sent; outbox.md NOT used (direct send)
```

### 2.3 Command: /pause [duration]

```
Step 1-2: Telegram poll receives /pause [duration]

Step 3: command_handlers.handle_pause_command()
  State change: .koan-pause written (3-line: reason="manual"/timestamp/display)
              via pause_manager.create_pause() -> atomic_write()

Step 4: outbox.md += "Paused" reply

Step 5: run.py reads .koan-pause at top of next iteration
  State change: run.py enters handle_pause() -> skips all mission execution
              .koan-status = "Paused (HH:MM)"
```

### 2.4 Command: /stop

```
Step 1-2: Telegram poll receives /stop

Step 3: command_handlers -> Path(koan_root, STOP_FILE).touch()
  State change: .koan-stop created

Step 4: run.py reads .koan-stop at top of next iteration
  State change: .koan-stop REMOVED (consumed)
              .koan-status REMOVED
              PID file released (fcntl.flock released)
              run.py process exits

Note: awake.py continues independently unless .koan-shutdown was used.
```

### 2.5 Command: /resume

```
Step 1-2: Telegram poll receives /resume

Step 3: command_handlers -> pause_manager.remove_pause()
  State change: .koan-pause REMOVED

Step 4: run.py's handle_pause() inner loop detects .koan-pause absence
  State change: handle_pause() returns "resume"
              count = 0, consecutive_errors = 0, consecutive_idle = 0
              _reset_usage_session() -> usage_state.json and usage.md reset
              _startup_notified = False
```

---

## 3. GitHub @mention to Mission

### 3.1 Happy Path — Authorized @mention Command

```
Step 1: Human comments "@koan-bot /review https://github.com/owner/repo/pull/123"
  State change: GitHub stores comment (unread notification for koan)

Step 2: loop_manager.process_github_notifications() called
  Trigger: periodic poll (every ~60-180s) OR .koan-check-notifications signal
  State change: .koan-check-notifications REMOVED if present (consumed)

Step 3: github_notifications.fetch_unread_notifications()
  State change: READ-ONLY GitHub API call (notifications endpoint)

Step 4: check_already_processed()
  Checks: in-memory processed_comments set, then reaction marker on comment
  State change: (read-only)

Step 5: parse_mention_command() -> extracts "/review" + PR URL
  State change: (in-memory parsing)

Step 6: validate_command() via skill registry
  State change: READ-ONLY registry lookup

Step 7: check_user_permission()
  Either: explicit allowlist check (in-memory) or GitHub API collaborator check
  State change: permission result cached in memory (NotificationTracker)

Step 8: add_reaction() — marks comment as "seen"
  State change: GitHub API adds emoji reaction to comment (external dedup)
              In-memory: processed_comments set += comment_id

Step 9: is_duplicate_mission() checks missions.md Pending + In Progress
  State change: READ-ONLY

Step 10: build_mission_from_command() constructs mission string
         insert_pending_mission() (or modify_missions_file)
  State change: missions.md[Pending] += "- /review https://... [project:name] ⏳(ts)" entry
```

### 3.2 Error: Unknown Command

```
Steps 1-6 as above. validate_command() returns False.
  State change: missions.md NOT modified

  (Optional NLP path) if natural_language enabled:
    State change: missions.md[Pending] += "/gh_request <full-comment-text>" entry

  post_error_reply() -> GitHub comment created
    State change: GitHub API creates error reply comment on PR/issue
```

### 3.3 Error: Duplicate Mission Already Pending/In Progress

```
Steps 1-9 as above. is_duplicate_mission() returns True.
  State change: missions.md NOT modified
  Note: Reaction already added in Step 8 — notification silently dropped.
  Note: Dedup only works for GitHub-action missions (/review, /rebase, etc.)
        with matching command:url signature. Generic text missions have no dedup.
```

---

## 4. GitHub PR Review Comment to Mission (review_dispatch)

### 4.1 Happy Path

```
Step 1: Human leaves review comment on Koan's open PR
  State change: GitHub stores review comment (unresolved)

Step 2: loop_manager.check_and_dispatch_review_comments() called
  (wired into process_github_notifications, runs each notification cycle)

Step 3: fetch_koan_open_prs() -> lists PRs where branch starts with koan prefix
  State change: READ-ONLY GitHub API

Step 4: fetch_unresolved_review_comments() + fetch_review_body_comments()
  State change: READ-ONLY

Step 5: compute_comment_fingerprint() -> SHA-256 of sorted (id, body) pairs
  State change: (in-memory computation)

Step 6: Load .review-dispatch-tracker.json, check stored fingerprint
  Per-project cooldown check (default 30 min)
  State change: READ-ONLY

Step 7: Fingerprint differs from stored (new/changed comments)
  State change: missions.md[Pending] += "/review https://... [project:name] ⏳(ts)"
              .review-dispatch-tracker.json updated (new fingerprint + timestamp)
              (atomic write)
```

---

## 5. CI Failure to Mission (ci_dispatch)

### 5.1 Happy Path

```
Step 1: CI check fails on a Koan-authored PR
  State change: GitHub marks check run with conclusion="failure"

Step 2: iteration_manager._dispatch_ci_fixes() called each iteration
  (Before mission pick, after recurring injection)

Step 3: fetch_koan_open_prs() + fetch_failing_check_runs()
  State change: READ-ONLY GitHub API

Step 4: compute_ci_fingerprint() -> "pr_number:head_sha:job_name:run_id"
  State change: (in-memory)

Step 5: Load .ci-dispatch-tracker.json, check existing key
  Per-project cooldown check (default 30 min)
  State change: READ-ONLY

Step 6: Key absent -> dispatch fix mission
  State change: missions.md[Pending] += "/fix [project:name] CI: <job> failed\n<log_snippet>"
              .ci-dispatch-tracker.json updated (atomic write)
```

---

## 6. Timer Events to Mission

### 6.1 Recurring Missions

```
Each iteration: iteration_manager._inject_recurring()
  Reads: instance/recurring.json (schedule definitions)

  If any entry is due:
    State change: missions.md[Pending] += recurring mission text ⏳(ts)
                instance/recurring.json updated: next_due timestamp advanced
                (atomic write)
```

### 6.2 One-Shot Event Scheduler

```
Each iteration: event_scheduler.tick()
  Reads: instance/events/*.json files

  If any event's scheduled datetime has passed:
    State change: missions.md[Pending] += event's mission text ⏳(ts)
                instance/events/<id>.json DELETED (consumed)
```

---

## 7. Core Agent Loop — Mission Execution

### 7.1 Happy Path — Mission Picked, Executed Successfully

```
Step 1: run.py main_loop() — top-of-loop signal checks
  Checks: .koan-stop, .koan-cycle, .koan-shutdown, .koan-restart-run, .koan-pause
  State change: Signals consumed (files removed) if present.
              If paused: no further execution in this iteration.

Step 2: iteration_manager.plan_iteration() called
  a. event_scheduler.tick() — inject scheduled one-shot missions
     State change: (see Section 6.2)
  b. _refresh_usage() -> reads usage_state.json, writes usage.md
     State change: instance/usage.md rewritten with current session/weekly %
  c. _maybe_warn_burn_rate() — burn-rate alert if imminent
     State change: if triggered: outbox.md += alert; .burn-rate.json last_warned_at set
  d. _get_usage_decision() -> autonomous_mode = "deep" | "implement" | "review" | "wait"
     State change: READ-ONLY (reads usage.md)
  e. _inject_recurring() — inject due recurring missions
     State change: (see Section 6.1)
  f. _drain_ci_queue() — process one CI queue entry
     State change: missions.md[CI] entry updated or removed (attempt counter incremented)
  g. _dispatch_ci_fixes() — queue fix missions for failing CI
     State change: (see Section 5)
  h. _pick_mission() -> (project_name, mission_title) from missions.md Pending
     State change: READ-ONLY (pick_mission.py reads missions.md, no write)
  i. _classify_mission() -> complexity tier classification + caching
     State change: if uncached: missions.md[Pending][mission] += "[complexity:tier]"
                 (via tag_complexity_in_pending -> modify_missions_file)

Step 3: mission_executor._run_iteration() called with plan dict

Step 4: Dedup guard — is_duplicate_in_progress() check
  State change: READ-ONLY

Step 5: Git preparation
  State change: Git branch created/checked out in project repository

Step 6: _start_mission_in_file() — CRITICAL ATOMIC TRANSITION
  State change: missions.md[Pending] entry REMOVED
              missions.md[In Progress] += same entry with "▶(timestamp)" appended
              (via modify_missions_file -> fcntl.flock LOCK_EX + threading.Lock)
  CRASH WINDOW: if process dies here, mission is in In Progress with no agent.
               recover.py handles this on next startup.

Step 7: create_pending_file() — inject context before Claude runs
  State change: instance/journal/pending.md CREATED with mission context,
              focus area, memory summary, and prior checkpoint if any.
              If a recovery context sentinel is present in the existing
              pending.md (written by recover.py._inject_checkpoint_context),
              that section is PRESERVED in the new file.

Step 7b: checkpoint_manager.create_checkpoint() — structured checkpoint created
  State change: instance/journal/checkpoints/<hash>.json CREATED with
              mission text, project name, run_num, started_at timestamp.
              (hash = first 12 chars of SHA-256 of stripped mission text)

Step 8: devcontainer.ensure_container_up() (if project has devcontainer: true)
  State change: Docker container started; git credentials configured in container

Step 9: build_mission_command() or build_skill_command()
  State change: (in-memory only — no file mutations)

Step 10: run_claude_task() — launches Claude CLI subprocess
  State change: stdout_file + stderr_file CREATED as tempfiles
              _sig.claude_proc = proc (in-memory)
              _sig.task_running = True (in-memory)
              [Concurrent] StagnationMonitor daemon thread STARTED
              [Concurrent] ProcessWatchdog timer thread STARTED
              [Concurrent] cli_journal_streamer writes stdout to daily journal in real-time

Step 11: Claude CLI agent executes
  State change: Project files potentially MODIFIED (git working tree)
              instance/journal/pending.md UPDATED by Claude as it works
              instance/memory/ potentially UPDATED
              Git commits possibly made by Claude in project repo
              instance/journal/checkpoints/<hash>.json UPDATED periodically:
                - branch name recorded after first commit
                - steps_done/remaining parsed from `CHECKPOINT: {...}` stdout lines
                - pending.md content synced to checkpoint (`update_from_pending`)

Step 12: Claude CLI exits (exit_code returned)
  State change: _sig.claude_proc = None
              _sig.task_running = False
              StagnationMonitor STOPPED; if stagnated: _last_mission_stagnated.set()
              ProcessWatchdog CANCELLED; if fired: _last_mission_timed_out = True

Step 13: _probe_exit0_quota() — scan for quota signals even on exit_code=0
  State change: if quota pattern detected:
              .koan-pause WRITTEN (reason="quota", reset_ts)
              _requeue_mission_in_file() -> missions.md[In Progress] -> missions.md[Pending]

Step 14: run_post_mission() — full post-mission pipeline
  a. checkpoint_manager.delete_checkpoint() — structured checkpoint removed on clean completion
     State change: instance/journal/checkpoints/<hash>.json DELETED (on exit_code=0)
  b. update_usage() — parse token costs from stdout, update usage_state.json + usage.md
     State change: instance/usage_state.json UPDATED (token accumulators)
                 instance/usage.md UPDATED (percentage display)
  c. quota detection from stdout/stderr
     State change: if detected: .koan-pause WRITTEN; mission requeued
  d. archive_pending() — move pending.md checkpoint to archive
     State change: instance/journal/pending.md REMOVED or archived
  e. Verification, lint gate, quality gate, reflection
     State change: Claude may make additional commits in project repo
                 Journal entries written
  f. security_review.differential_review()
     State change: Security audit log UPDATED
  g. check_auto_merge() -> auto-merge if conditions met
     State change: GitHub PR MERGED + CLOSED (if auto-merge config matches)
                 Remote branch DELETED
                 Local branch DELETED
                 Journal entry written
  h. hooks.fire_hook("post_mission")
     State change: User hook handlers run (side effects vary)
  i. maybe_queue_autoreview()
     State change: possibly missions.md[Pending] += "/review PR_URL" entry

Step 15: _finalize_mission() — CRITICAL ATOMIC TRANSITION
  a. Read + clear _last_mission_stagnated flag
  b. If stagnated AND retry_count < max_retry:
     State change: .stagnation-retries.json[mission_hash].count INCREMENTED
                 missions.md[In Progress] -> missions.md[Pending] (requeue)
                 Telegram: stagnation retry notification
     Return (no Done/Failed entry written)
  c. If stagnated AND retry_count >= max_retry:
     State change: .stagnation-retries.json[mission_hash] CLEARED
                 cause_tag = "stagnation:<pattern>"
                 (fall through to _update_mission_in_file with Failed)
  d. _update_mission_in_file() (non-stagnation or stagnation cap reached):
     State change: if exit_code=0: missions.md[In Progress] -> missions.md[Done] ✅(ts)
                 if exit_code!=0: missions.md[In Progress] -> missions.md[Failed] ❌(ts) [cause_tag]
                 prune_completed_sections() called inline: oldest Done/Failed TRIMMED
  e. mission_history.record_execution()
     State change: instance/mission_history.jsonl += execution record

Step 16: _notify_mission_end() — Telegram notification
  State change: outbox.md += "✅/❌ [project] Run N/M — <title>" message

Step 17: commit_instance() — journal/memory git commit
  State change: git add + git commit + git push in koan repo (if changes exist)

Step 18: Periodic git sync (every git_sync_interval iterations)
  State change: Merged branches DELETED locally and remotely
              .branch-cleanup-tracker.json UPDATED (cooldown timestamp)
              Orphan branches detected -> outbox.md += notification

Step 19: _sleep_between_runs() or interruptible_sleep()
  State change: .koan-status UPDATED ("Idle — sleeping Ns")
              Wakes early if missions.md[Pending] becomes non-empty
```

### 7.2 Failure Path — Watchdog Timeout

```
Steps 1-10: Same as happy path.

Step 11: ProcessWatchdog fires at mission_timeout seconds
  State change: SIGKILL sent to Claude subprocess process group
              watchdog.fired = True (in-memory)

Step 12: run_claude_task() returns
  State change: _last_mission_timed_out = True (in-memory)
              exit_code = 1

Step 13: _maybe_retry_mission() — single retry on transient errors
  If retry conditions met (not already retried, not watchdog/abort/stagnation):
    State change: Mission stays In Progress (NOT finalized)
                Second Claude invocation launched (back to Step 10)
  If not retriable:
    Proceed to finalize as failure

Step 14: _finalize_mission() (exit_code=1)
  State change: missions.md[In Progress] -> missions.md[Failed] ❌(ts)

Step 15: Telegram notification: "❌ [project] Run N/M — Failed: <title>"
```

### 7.3 Failure Path — Stagnation Detected

```
Steps 1-10: Same as happy path.

Step 11: StagnationMonitor fires after K identical hashes (default K=3)
  State change: on_abort() called: SIGKILL sent to subprocess
              _last_mission_stagnated.set() (threading.Event)
              _stagnation_pattern_type/excerpt captured (in-memory)

Step 12: run_claude_task() returns exit_code = 1

Step 13: _finalize_mission() reads _last_mission_stagnated.is_set() -> True
  State change: _last_mission_stagnated.clear()

  If retry_count < max_retry_on_stagnation (config):
    State change: .stagnation-retries.json[sha256(title)] counter INCREMENTED
                missions.md[In Progress] -> missions.md[Pending] (requeue, clean entry)
    Telegram: stagnation retry notification
    Return (iteration ends without Done/Failed)

  If retry_count >= max_retry_on_stagnation:
    State change: .stagnation-retries.json[sha256(title)] CLEARED
                missions.md[In Progress] -> missions.md[Failed] ❌(ts) [stagnation:pattern]
    Telegram: stagnation abort notification
```

### 7.4 Failure Path — Quota Exhaustion Mid-Mission

```
Steps 1-12: Mission runs; Claude exits with non-zero exit code.

Step 13: _classify_and_handle_cli_error() called
  classify_cli_error() identifies ErrorCategory.QUOTA

  _handle_quota_error() called:
    State change: _requeue_mission_in_file():
                  missions.md[In Progress] -> missions.md[Pending] (stripped entry, no timestamps)
                handle_quota_exhaustion():
                  .koan-pause WRITTEN (reason="quota", reset_ts, display)
                  Journal entry appended to project journal

  Telegram: "⏸️ API quota exhausted. Mission moved back to Pending."

Step 14: run.py next iteration reads .koan-pause
  State change: handle_pause() entered; no missions executed
              .koan-status = "Paused (quota)"
```

### 7.5 Failure Path — Auth Error

```
Similar to quota path:
  _handle_auth_error():
    State change: missions.md[In Progress] -> missions.md[Pending] (requeue)
                .koan-pause WRITTEN (reason="auth")
  Telegram: "Provider is logged out. Mission moved back to Pending."
```

### 7.6 Skill Dispatch Path (/plan, /rebase, /review, etc.)

```
Steps 1-6: Same as normal mission through _start_mission_in_file().

Step 7: dispatch_skill_mission() detects /command prefix
  Returns skill_cmd (list of args for Python subprocess)

Step 8: create_pending_file() for /live streaming support
  State change: journal/pending.md CREATED

Step 9: _run_skill_mission() launches skill subprocess (not Claude directly)
  State change: Tempfiles created (stdout_file, stderr_file, stream_usage_file)
              journal/pending.md opened for streaming (line-by-line from stdout)
              ProcessWatchdog + LivenessWatchdog started
              _sig.claude_proc = proc (in-memory)

Step 10: Skill subprocess runs (e.g. rebase_pr.py, review.py)
  State change: project repo possibly modified (commits made by skill)
              pending.md updated in real-time (visible via /live)

Step 11: Skill exits
  State change: if koan repo branch changed by skill (e.g. /rebase):
              _restore_koan_branch() checks and restores
              _sig.claude_proc = None

Step 12: run_post_mission() — same pipeline as normal mission
  State change: (see Step 14 in happy path)

Step 13: Quota/auth check via _classify_and_handle_cli_error()
  IMPORTANT: trust_stdout=False because skill stdout is DATA (agent transcript
             that may quote "quota_exhausted: false" from CI logs).
             Only stderr + provider runtime JSON lines are scanned.

Step 14: _finalize_mission() -> same as normal mission path
```

---

## 8. Crash Recovery

### 8.1 Startup Recovery (recover_missions)

```
Step 1: run.py starts after crash or kill

Step 2: run_startup() -> startup_manager.run_startup()

Step 3: recover.recover_missions() called:
  a. Reads missions.md — scans In Progress section
  b. Checks journal/pending.md non-empty (has_pending_journal)
  c. For each "- " mission line (and `### ` complex-mission blocks as units):
     - Count [r:N] recovery counter (0 if absent)
     - Try to read structured checkpoint via checkpoint_manager.read_checkpoint()
       (looks up instance/journal/checkpoints/<hash>.json)
     - Classify: unrecoverable (r >= 3), partial (has checkpoint OR pending.md, first only),
       dead (neither). `has_pending_journal` is consumed (set to False) after the first
       "partial" classification so subsequent missions are classified as "dead".

Step 4: For "dead" and "partial" missions:
  State change: missions.md[In Progress] entry MOVED to missions.md[Pending]
              [r:N] tag INCREMENTED in moved entry (e.g. [r:1], [r:2])
              recovery.jsonl += {"action": "recovered", "attempts": N+1, ...}

Step 5: For "unrecoverable" missions (r >= 3):
  State change: missions.md[In Progress] entry MOVED to missions.md[Failed]
              Entry prefixed with "❌ needs_input:"
              recovery.jsonl += {"action": "escalated", "attempts": 3}

Step 6: _inject_checkpoint_context() — for first mission with checkpoint:
  State change: journal/pending.md UPDATED with structured checkpoint data
              (checkpoint_manager.format_recovery_context() formats steps_done,
               steps_remaining, branch, etc. from the JSON checkpoint file)
              A sentinel header "## Recovery Context (from previous interrupted run)"
              is written so create_pending_file() can detect and preserve this
              section when the mission restarts.

Step 7: format_and_send() — Telegram notification
  State change: outbox.md += restart message + escalation warnings (if any)
```

### 8.2 Sanity Flush During start_mission()

```
_flush_in_progress_to_failed() → _flush_abandoned_in_progress() is called INSIDE
start_mission() before inserting the new In Progress entry. Any existing In Progress
missions are moved to Failed (not Done) with a [flushed] tag.

State change: (within modify_missions_file lock)
  missions.md[In Progress] ALL entries -> missions.md[Failed] ❌(ts) [flushed]
  New mission inserted into In Progress

Note: This is a "clean restart" safety mechanism. Under normal operation,
      In Progress should be empty when start_mission() is called.
      Under a crash scenario, recover.py already handled In Progress
      before we get here — so this flush in start_mission() handles
      the edge case where recover.py ran and moved a mission to Pending,
      but a second stale In Progress entry was missed. Missions are marked
      Failed (not Done) so history correctly reflects that they did not complete.
```

---

## 9. Quota Pause and Auto-Resume Cycle

### 9.1 Quota Exhaustion Path (Multiple Entry Points)

```
Entry points:
  A. _classify_and_handle_cli_error() (non-zero exit with QUOTA classification)
  B. run_post_mission() (quota flag in structured pipeline output)
  C. _probe_exit0_quota() (quota signal on exit 0)
  D. _handle_wait_pause() (UsageTracker decides WAIT mode)
  E. preflight_quota_check() (pre-mission API probe)

All paths converge at: create_pause(koan_root, "quota", reset_ts, display)
  State change: .koan-pause WRITTEN (3-line format: "quota" / reset_unix_ts / display)
              Journal entry written with reset time
              Mission requeued to Pending (all paths except D)

Main loop reads .koan-pause -> handle_pause() entered:
  State change: .koan-status = "Paused (quota)"
              No missions executed; no contemplation; no autonomous work
              GitHub notifications still processed (inbox check throttled to 1h)

Auto-resume check (every 5 minutes in handle_pause()):
  should_auto_resume() -> checks: is timed? has reset_ts elapsed?
  When elapsed:
    State change: .koan-pause REMOVED (remove_pause())
                _reset_usage_session() -> usage_state.json reset, usage.md rewritten
                count = 0, consecutive_* = 0 (main loop)
    Telegram: "Koan auto-resumed"
```

---

## 10. Process Lifecycle

### 10.1 Stack Startup (make start / pid_manager.start_all())

```
Step 1: Provider auto-detection (claude vs ollama)
  State change: (in-memory; no files yet)

Step 2: start_awake() -> subprocess.Popen
  State change: awake.pid CREATED (bash PID, no flock — best-effort cleanup)

Step 3: start_runner() -> subprocess.Popen
  State change: run.pid ACQUIRED (Python fcntl.flock LOCK_EX — exclusive)
              .koan-stop, .koan-shutdown, .koan-cycle, .koan-abort,
              .koan-reset-counter, .koan-restart-run CLEARED on startup
              (stale signals from previous session discarded)

Step 4: run.py startup sequence (delegated to startup_manager.run_startup()):
  Protected phase "Startup checks":
  a. config_validator.validate_config_or_raise() — strict config validation (hard stop on error)
  b. recover.recover_missions() — crash recovery (see Section 8.1)
  c. run_pending_migrations() — schema migrations applied
  d. ensure_projects_yaml() — projects.yaml auto-creation from env vars if missing
  e. migrate_memory_to_jsonl() — one-shot memory format migration
  f. ensure_github_urls() — github_url fields auto-populated from git remotes
  g. discover_workspace() — workspace discovery (finds additional project repos)
  h. remote_rename_detector.check_and_fix() — fix renamed remotes
  i. run_sanity_checks() — instance directory consistency checks
  j. cleanup_memory() — throttled per-startup memory compaction (24h default)
  k. prune_missions_done() — trim oversized Done/Failed sections at startup
  l. cleanup_mission_history() — evict stale mission_history.jsonl entries
  m. check_health() — agent health check
  Protected phase "Self-reflection check":
  n. check_self_reflection() — optional Claude-powered self-reflection session
  Start on pause / passive:
  o. handle_start_on_pause() — if KOAN_SKIP_START_PAUSE=1 or .koan-skip-start-pause present, skip startup pause
  p. handle_start_passive() — enter passive mode if configured
  Git identity / auth:
  q. setup_git_identity() — configure GIT_AUTHOR_EMAIL from env/.env
  r. setup_github_auth() — verify GitHub CLI auth
  s. Telegram startup notification sent (provider, max_runs, interval, projects, status)
  Protected phase "Git sync":
  t. git_sync.sync_and_report() — branch cleanup report
  u. head_tracker.check_and_update() — detect HEAD branch changes (12h throttle)
  v. track_koan_commits() — record current HEAD SHA; diff vs previous for changelog
  w. check_auto_update() — fetch upstream; if new commits, pull + sys.exit(42) restart
  Daily report / morning ritual:
  x. run_daily_report() — generate daily summary if due
  y. run_morning_ritual() — Claude-powered session-start reflection (~90s, skippable)
  Hook system:
  z. init_hooks() — discover instance/hooks/*.py modules
  z2. fire_hook("session_start") — user lifecycle hooks
  Returns (max_runs, interval, branch_prefix) to main_loop()

Note: The old 30s startup delay (_startup_delay) has been superseded by
      handle_start_on_pause() + the .koan-skip-start-pause mechanism.
```

### 10.2 Restart Signal Flow

```
Step 1: /update or auto_update triggers restart
  State change: .koan-restart-bridge WRITTEN
              .koan-restart-run WRITTEN
              .koan-restart WRITTEN (legacy)

Step 2: run.py detects .koan-restart-run at top of iteration
  State change: .koan-restart-run CONSUMED (clear_restart removes file)
              sys.exit(RESTART_EXIT_CODE=42) raised

Step 3: main() restart wrapper catches SystemExit(42)
  State change: PID file released (in main_loop() finally block)
              os.execv() called: new interpreter image loaded with same PID
  Note: If os.execv fails, in-process restart continues (stale modules in memory)

Step 4: awake.py detects .koan-restart-bridge
  State change: .koan-restart-bridge CONSUMED
              os.execv() self-restart (reexec_bridge())
```

---

## 11. Bridge Lifecycle (awake.py)

### 11.1 Outbox Flushing

```
Background thread in awake.py runs continuously:

Step 1: Read outbox.md with fcntl.flock LOCK_EX
  State change: (read with lock held)

Step 2: Parse messages (separated by "---" markers)
  State change: (in-memory)

Step 3: For each message, call send_telegram() with flood protection
  State change: Telegram message sent
              Flood protection: if >N messages in window, throttle/drop lower-priority

Step 4: Truncate/rewrite outbox.md to remove consumed entries
  State change: outbox.md TRUNCATED (entries consumed)
              fcntl.flock released

Concurrent writers (run.py, notify.py):
  All use append_to_outbox() -> fcntl.flock LOCK_EX for append
  Risk: If awake.py crashes mid-flush (between Step 3 and Step 4),
        messages are lost (already sent but outbox not yet cleared is safe;
        read but not yet sent is lost).
```

---

## 12. State Transition Summary

```
Mission Lifecycle State Machine:

  (not exist)
       |
       v  insert_mission() [Telegram/GitHub/@mention/recurring/event]
  +---------+
  | Pending |
  +----+----+
       |  start_mission()            start_mission() also calls:
       |  [modify_missions_file]     _flush_in_progress_to_done()
       v                             (moves any stale In Progress -> Done)
  +-------------+
  | In Progress |
  +------+------+
         |
    +----+-----------------------------+
    |                                  |
exit_code=0                       exit_code=1
    |                                  |
    v                                  v
complete_mission()              [quota/auth?] ---> requeue -> Pending
[In Progress -> Done]           [stagnated?] ---> requeue OR fail [stagnation]
                                [otherwise]  ---> fail_mission()
                                                  [In Progress -> Failed]

  Pending <-- recover_missions() (r < 3) -- In Progress    (on startup)
  Failed  <-- recover_missions() (r >= 3) -- In Progress   (on startup)
  Failed  <-- _flush_abandoned_in_progress() -- In Progress  (start_mission sanity, [flushed] tag)

Pause State Machine:

  (active)
       |
       v  create_pause(reason)
  +--------+
  | Paused |  reasons: quota, auth, idle_timeout, errors, max_runs, manual
  +---+----+
      |
      +-- remove_pause() [/resume command]           -> (active)
      +-- should_auto_resume() [time-based, ~5h]     -> (active) + usage reset

Process State Machine:

  (stopped) -> start_all() -> (running) -> stop_processes() -> (stopped)
                                  |
                                  +-- sys.exit(42) + os.execv() -> (restarted)
```

---

## 13. Concurrent Access Patterns and Locking

### 13.1 missions.md — Two-Process Shared State

Both `run.py` and `awake.py` read and write `missions.md` concurrently.

**Write paths from run.py:**
- `start_mission()`, `complete_mission()`, `fail_mission()`, `requeue_mission()`
- `tag_complexity_in_pending()`, `insert_mission()` (for diagnostics, autoreview queue)
- `prune_completed_sections()` (inside `_update_mission_in_file`)
- `add_ci_item()`, `remove_ci_item()`, `update_ci_item_attempt()` (CI queue)

**Write paths from awake.py:**
- `insert_pending_mission()` — mission queuing from Telegram
- `cancel_pending_mission()` / `cancel_pending_missions_bulk()` — /cancel command
- `reorder_mission()` / `reorder_missions_bulk()` — /priority command
- `edit_pending_mission()` — /edit command

**Locking mechanism (`utils.modify_missions_file`):**
1. Acquire `threading.Lock()` (in-process protection)
2. Acquire `fcntl.flock(fd, LOCK_EX)` (cross-process exclusive lock)
3. Read file, apply transform function, write to temp file
4. `os.replace(temp, missions_path)` — atomic rename
5. Release flock, release threading lock

### 13.2 outbox.md — Append-Write and Consume Pattern

- Writers: `append_to_outbox()` in run.py, awake.py — uses `fcntl.flock(LOCK_EX)` + append
- Reader/consumer: awake.py outbox flusher thread — uses flock to read then truncate

### 13.3 Signal Files — One-Shot Flag Pattern

Signal files (`.koan-pause`, `.koan-stop`, etc.) follow the pattern:
- Writer creates file (often with content)
- Reader checks existence, then removes (consumes)
- `unlink(missing_ok=True)` used for safe removal
- No explicit locking needed: `unlink()` is atomic on Linux

All signal file name constants are centralized in `koan/app/signals.py`. Callers import from there; hardcoding `.koan-*` strings is discouraged.
