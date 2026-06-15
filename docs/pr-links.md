# Fix Branch PR Links

Open each link to create a pre-filled PR from `exciton/koan` → `Anantys-oss/koan:main`.

---

## B5 — Flush abandoned missions to Failed

**Title:** `fix(missions): redirect abandoned in-progress missions to Failed section`

**Branch:** `claude/fix-flush-to-failed`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/fix-flush-to-failed?expand=1)

**Body:**
```
## Problem
When `start_mission()` finds stale In Progress missions (sanity enforcement), it was silently
moving them to Done with a ✅ marker — creating false history that the work completed
successfully. This fires when `recover.py` misses a mission (complex mission blocks, import errors).

## Changes
- Renamed `_move_in_progress_to_done()` → `_flush_abandoned_in_progress()` for clarity
- Changed the marker to ❌ with a `[flushed]` tag inserted into the Failed section
- Creates the Failed section if it doesn't already exist
- Updated all test assertions to check `sections["failed"]` instead of `sections["done"]`

## Test
All 415 existing tests pass (1 pre-existing root-permission test excluded).
```

---

## B2 + B7 — Recover complex mission blocks; single-use pending journal flag

**Title:** `fix(recover): recover complex ### mission blocks from In Progress`

**Branch:** `claude/fix-complex-mission-recovery`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/fix-complex-mission-recovery?expand=1)

**Body:**
```
## Problem
Multi-step missions using the `### Header / - Step 1 / - Step 2` format were silently skipped
by crash recovery. They stayed in In Progress indefinitely and were never re-queued.

Additionally, `has_pending_journal` was computed once and applied to all in-progress missions.
With multiple stale missions, all were classified as "partial" even though pending.md was written
by exactly one interrupted run.

## Changes
- Replaced simple `in_complex_mission` flag with `_finalize_complex_block()` inner function
- Entire block (header line + sub-items) collected and classified together using the header as
  the mission key
- Recoverable block: all lines moved to Pending with `[r:N]` in the `### ` header
- Unrecoverable block: header line moved to Failed
- `journal_available` local flag initialized from `has_pending_journal`, set to `False` after
  first "partial" classification so subsequent missions are correctly classified as "dead"

## Test
70+ tests pass. `test_skip_complex_mission` renamed to `test_recover_complex_mission_block`
with updated assertions.
```

---

## B3 + B14 — Preserve checkpoint context; suppress journal dir OSError

**Title:** `fix(loop_manager): preserve checkpoint recovery context in create_pending_file`

**Branch:** `claude/fix-checkpoint-overwrite`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/fix-checkpoint-overwrite?expand=1)

**Body:**
```
## Problem
`recover.py._inject_checkpoint_context()` writes structured recovery data to pending.md at
startup. `create_pending_file()` called when the mission starts then overwrote it completely,
making the "partial" state classification a no-op.

Additionally, if `instance/journal/YYYY-MM-DD/` could not be created (disk full, permissions),
pending.md was never written, losing checkpoint context for Claude.

## Changes
- `create_pending_file()` reads existing pending.md before writing
- If the recovery context sentinel (`## Recovery Context (from previous interrupted run)`) is
  found, the checkpoint section is appended after the new header
- Regular pending.md content (no sentinel) is not carried over
- `journal_dir.mkdir()` wrapped in `contextlib.suppress(OSError)` — pending.md write proceeds
  regardless of dated subdir failure

## Test
Added `test_preserves_recovery_context_from_pending_md` and
`test_does_not_preserve_regular_pending_md`.
```

---

## B4 — Persist Telegram offset across bridge restarts

**Title:** `fix(awake): persist Telegram polling offset across bridge restarts`

**Branch:** `claude/fix-telegram-offset`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/fix-telegram-offset?expand=1)

**Body:**
```
## Problem
Telegram `offset` was in-memory only. After bridge restart the offset reset to `None`, causing
Telegram to re-deliver all updates from the ~60s window before the restart. This could cause
duplicate mission queuing and commands executed twice.

## Changes
- `_save_offset(offset)` persists to `instance/.telegram-offset.json` atomically on each
  `update_id` advance
- `_load_offset()` reads the persisted value at startup, logging a resume message
- `TestMainLoop` autouse fixture mocks `_load_offset` to return `None` for test isolation

## Test
267 tests pass. 1 pre-existing root-permission test excluded.
```

---

## B9 — Verify start_mission transition; abort on mismatch

**Title:** `fix(run): verify start_mission transition and abort on mismatch`

**Branch:** `claude/fix-start-mission-return`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/fix-start-mission-return?expand=1)

**Body:**
```
## Problem
`_start_mission_in_file()` discarded the return value of `modify_missions_file()` and could not
detect whether the mission actually moved to In Progress. Silent failure left the mission in
Pending while Claude executed it, causing:
- `/list` showing the mission as still Pending during execution
- Potential duplicate queuing by the user
- No In Progress entry for `recover.py` to find on crash

## Changes
- `_start_mission_in_file()` returns `bool` (True = confirmed in In Progress)
- After locked write, reads resulting content via `parse_sections()` to verify mission is in
  In Progress
- Logs WARNING on mismatch
- `mission_executor.py` aborts the run (returns `False`) when unconfirmed

## Test
749 tests pass.
```

---

## B1 + B13 — Fix stagnation key stability across requeue cycles

**Title:** `fix(stagnation): strip lifecycle markers from mission key before hashing`

**Branch:** `claude/fix-stagnation-key`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/fix-stagnation-key?expand=1)

**Body:**
```
## Problem
`_mission_key()` hashed the raw mission title including ⏳/▶ timestamps, `[r:N]` recovery
counters, and `[complexity:X]` tags. After `requeue_mission()` strips those markers, the
re-picked mission acquires new timestamps — producing a different hash and silently resetting
the stagnation retry counter on every cycle. `max_retry_on_stagnation` was therefore never
reached and a persistently-stagnating mission would loop indefinitely.

Additionally, `[r:N]` tags from crash recovery cause the same key instability — each recovery
cycle produces a new key, abandoning the stagnation retry history.

## Changes
- `_STRIP_FOR_KEY_RE` now strips timestamps, `[r:N]` tags, and `[complexity:X]` tags before
  hashing, making the key stable across requeue and crash-recovery cycles

## Test
Tests added for key stability across requeue cycles.
```

---

## B6 — Unified retry cap; remove `[r:N]` tags; preserve counter in Failed

**Title:** `refactor(retry): consolidate crash and stagnation counters; preserve in Failed state`

**Branch:** `claude/fix-unified-retry-cap`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/fix-unified-retry-cap?expand=1)

**Body:**
```
## Problem

Two independent retry systems accumulated silently with no shared ceiling:
- `[r:N]` tags embedded in mission text (missions.md) for crash-recovery, hardcoded max 3
- `.stagnation-retries.json` for stagnation retries, max configurable

A mission could cycle between stagnating and crashing indefinitely because:
1. Neither counter knew about the other
2. Any non-stagnation exit cleared the stagnation counter, resetting cross-system progress
3. Counters were cleared immediately on escalation to Failed, so the human couldn't see why

## Changes

**Commit 1 — cross-system ceiling (`total_attempts`):**
- Add `total_attempts` field to tracker entries; incremented by both stagnation requeues and
  crash-recovery on requeue
- New `max_total_retries` config key (default 0 = disabled) in `get_stagnation_config()` —
  single operator knob across both systems
- `clear_retry_count(clear_total=False)` preserves `total_attempts` across crash cycles
- Both `classify_mission_state()` and `_finalize_mission` check combined cap

**Commit 2 — unified storage (remove `[r:N]` from missions.md):**
- Rename `.stagnation-retries.json` → `.mission-retries.json` with auto-migration
- Add `crash_count` field alongside existing stagnation `count`
- New `get_crash_count()` / `increment_crash_count()` API; `increment_crash_count()` also
  increments `total_attempts`
- New `max_crash_retries` config key (default 3) replaces hardcoded `MAX_RECOVERY_ATTEMPTS`
- `classify_mission_state()` takes `crash_count: int` instead of parsing `[r:N]` from text
- Remove `MAX_RECOVERY_ATTEMPTS`, `_get_recovery_attempts`, `_set_recovery_attempts` from
  `recover.py`; keep `_strip_recovery_counter()` for backward-compat cleanup of old tags
- Backward compat: legacy `[r:N]` tags in existing missions.md read for classification only;
  never seeded to tracker; stripped on next write

**Commit 3 — counter lifetime (preserve in Failed; clear on human retry):**
- Counter is NOT cleared when stagnation cap is hit or mission escalated to Failed
- Counter is cleared only when `_start_mission_in_file()` detects a cap was previously hit
  (`stag_count >= max_retry` OR `crash_count >= max_crash_retries` OR `total >= max_total`),
  signalling a deliberate human retry
- Ongoing stagnation-retry requeus (count < cap) keep their counter intact
- New `_clear_if_cap_hit()` helper encapsulates the conditional clear logic

## Counter lifetime

| Event | Counter action |
|---|---|
| Crash → Failed | preserve all |
| Stagnation cap hit → Failed | preserve all |
| Escalated unrecoverable → Failed | preserve all |
| Stagnation retry requeue (count < cap) | preserve (cap still needs to fire) |
| Mission success | full clear |
| `start_mission()` with cap-hit counter | full clear (human deliberate retry) |

## Test

562 tests pass (6 pre-existing environment failures excluded).
New: `TestCrashCount`, updated `TestRetryTracker`, `TestTotalAttempts`, `TestUnifiedRetryCap`,
`TestClassifyMissionState`, `TestMigrationBackwardCompat`.
```

---

## B7 standalone — Single-use pending journal flag

**Title:** `fix(recover): consume pending.md context for first mission only`

**Branch:** `claude/fix-recover-pending-journal-scope`

> **Note:** This fix is also included in `claude/fix-complex-mission-recovery` above.
> Skip this PR if that one is accepted first.

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/fix-recover-pending-journal-scope?expand=1)

**Body:**
```
## Problem
`has_pending_journal` was computed once and applied to all in-progress missions. With multiple
stale missions, all were classified as "partial" even though pending.md was written by exactly
one interrupted run.

## Changes
- `journal_available` local flag initialized from `has_pending_journal`
- Set to `False` after first "partial" classification, so subsequent missions are correctly
  classified as "dead"

## Test
Added `TestPendingJournalSingleUse` verifying second mission gets "dead" state.
```

---

## B11 — Report found-status from `complete_mission`/`fail_mission`; fix silent no-op

**Title:** `fix(missions): report found-status from complete/fail; fix silent no-op`

**Branch:** `claude/fix-finalize-mission-telemetry`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/fix-finalize-mission-telemetry?expand=1)

**Body:**
```
## Problem

`complete_mission()` / `fail_mission()` return content unchanged when the mission is absent
from Pending and In Progress, with no signal to the caller. `run.py`'s
`_update_mission_in_file()` then inferred success by comparing content before and after the
locked write — but `prune_completed_sections()` runs unconditionally on that path, so an
oversized Done/Failed section makes the content differ even on a genuine no-op. Result: an
absent mission was wrongly reported as moved (returning True), masking a stuck mission that
re-dispatches on every loop.

## Changes

- `missions._move_pending_to_section()` now returns a `(content, found: bool)` tuple
- New `complete_mission_checked()` / `fail_mission_checked()` expose the found flag;
  `complete_mission()` / `fail_mission()` keep their `str` return as thin wrappers, so no
  existing caller or test changes
- `run._update_mission_in_file()` captures found-status via a closure flag (the same pattern
  `insert_pending_mission()` already uses) and bases its WARNING + bool return on it —
  decoupled from the pruning side effect

## Test

968 related tests pass. New:
- `TestMissionCheckedVariants` — found=True/False across Pending, In Progress, and absent;
  cause_tag preserved; wrappers equal the checked variant's content
- `test_not_found_returns_false_even_when_pruning_changes_content` — regression proving an
  absent mission reports False even when pruning mutates the file (the exact false-positive
  the old before/after comparison produced)
```

---

## B10 — Document `requeue_mission()` top-of-queue insertion

**Title:** `docs(missions): document requeue_mission top-of-queue insertion`

**Branch:** `claude/fix-requeue-priority-docs`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/fix-requeue-priority-docs?expand=1)

**Body:**
```
## Problem
`requeue_mission()` inserts the recovered mission at the TOP of Pending, while `insert_mission()`
appends at the bottom (FIFO). Quota/auth-requeued missions therefore jump ahead of all other
pending work. This is intentional — interrupted work should resume before unstarted missions —
but was undocumented and surprised operators who saw queue ordering change after a quota pause.

## Changes
- Added a "Queue position — TOP, not bottom (intentional)" paragraph to the `requeue_mission()`
  docstring, contrasting it with `insert_mission()`'s FIFO append and explaining the rationale

## Test
Documentation-only change; behaviour unchanged. Existing requeue tests pass.
```

---

## B12 — Decouple missions.md history pruning from finalization

**Title:** `refactor(run): decouple missions.md history pruning from finalization`

**Branch:** `claude/fix-prune-decoupling`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/fix-prune-decoupling?expand=1)

**Body:**
```
## Problem
`prune_completed_sections()` ran inside the same locked read-modify-write that moves a mission
to Done/Failed (`_update_mission_in_file`). A pruning bug or misconfiguration could silently
mutate or corrupt history during the finalization write — two separate concerns coupled into
one transform, with the finalization result depending on pruning succeeding.

## Changes
- Extracted pruning into `_prune_missions_history()`: a standalone, locked, best-effort step
  run *after* the move commits. The finalization write now does only the mission move.
- The helper uses the missions lock (via `modify_missions_file`) so it cannot race the bridge
  inserting new missions — unlike the startup-time `startup_manager.prune_missions_done()`,
  which is safe unlocked only because nothing else writes during startup.
- A pruning error is logged and swallowed, leaving the committed move intact.

Pruning still runs per-finalization (missions.md stays bounded during long sessions), but as a
separate locked write — the coupling, not the cadence, was the problem.

## Test
`TestPruneDecoupledFromFinalization`:
- `test_finalization_triggers_history_prune` — oversized Failed section still trimmed to
  failed_keep after a completion
- `test_prune_failure_does_not_break_finalization` — a pruning RuntimeError leaves the move
  intact (returns True, Done updated, history uncorrupted)
- `test_prune_helper_is_noop_below_threshold` — small history left untouched
```

---

# State Simplifications (S-series)

---

## S1 — Cross-link the two crash-recovery safety nets; log sanity flush

**Title:** `docs+log: cross-link crash-recovery safety nets; log sanity flush`

**Branch:** `claude/simplify-flush-crosslink`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/simplify-flush-crosslink?expand=1)

**Body:**
```
## Problem
recover.py (startup, → Pending) and _flush_in_progress_to_failed (per-mission via
start_mission(), → Failed) are independent safety nets for the same stale In Progress
scenario, with no documented relationship and no visibility when the second net fires.

## Changes
- recover.recover_missions() docstring cross-links forward to the per-mission flush net
- missions._flush_in_progress_to_failed() docstring cross-links back to recover.py
- run._start_mission_in_file() captures stale In Progress inside the lock and emits a WARNING
  naming the flushed missions when the sanity flush fires (missions.py stays pure)

## Test
TestStartMissionSanityFlushLog — flush logged + mission moved to Failed with [flushed];
no log when In Progress is empty.
```

---

## S6 — Single pending.md read in recover.py (close TOCTOU double-read)

**Title:** `refactor(recover): single pending.md read; reuse caller value`

**Branch:** `claude/simplify-pending-journal-read`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/simplify-pending-journal-read?expand=1)

**Body:**
```
## Problem
recover.py's CLI entry point read pending.md twice — once via check_pending_journal() for the
summary message, then again inside recover_missions() for partial-state classification — with a
TOCTOU window between the reads. (The daemon startup path only ever did the single internal read.)

## Changes
- recover_missions() gained an optional has_pending_journal kwarg. When the caller already read
  pending.md, it passes the result in and the function skips its own read.
- Default None preserves the daemon path (compute internally) and every existing caller/test
  (return shape unchanged).
- CLI main() reads once via check_pending_journal() and hands the value down.

## Test
test_passed_has_pending_overrides_file_read (supplied True classifies partial with no file on
disk), test_default_none_still_reads_file (daemon path still reads from disk).
```

---

## S3 — Route OutboxManager.requeue() through append_to_outbox()

**Title:** `refactor(outbox): route requeue() through append_to_outbox`

**Branch:** `claude/simplify-outbox-append`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/simplify-outbox-append?expand=1)

**Body:**
```
## Problem
OutboxManager.requeue() duplicated the open/flock/write pattern that
utils.append_to_outbox() already implements — two slightly different append paths for the
same file.

## Changes
- requeue() delegates to append_to_outbox() (preserving the trailing newline), leaving a
  single locked write path. The outbox-failed.md fallback on error is retained.

## Test
test_requeue_preserves_trailing_newline (exact newline contract),
test_requeue_falls_back_to_failed_on_error (fallback still triggers when the shared append raises).
```

---

## S4 — Replace startup flag pair with a single _startup_phase

**Title:** `refactor(run): replace startup flag pair with single _startup_phase`

**Branch:** `claude/simplify-startup-phase`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/simplify-startup-phase?expand=1)

**Body:**
```
## Problem
run.py tracked first-iteration Telegram visibility with two booleans
(_startup_notified, _boot_notified) whose semantics overlapped and were reset inconsistently.

## Changes
- Replace the pair with a single _startup_phase: "boot" → "resume" → "running"
- run.py: _mark_startup_resume() downgrades running→resume on /resume and counter reset, but
  preserves "boot" when no iteration has run yet (start-paused-then-resumed still gets boot
  banners once)
- mission_executor.py: derive is_boot_iteration (phase=="boot") and is_first_iteration
  (phase in boot/resume) from the phase, then set "running"

Behavior verified identical across normal boot, resume-after-boot, and resume-before-first-iteration.

## Test
TestStartupPhase (running→resume, boot preserved, resume idempotent); existing
first-iteration/resume notification suites updated to drive the phase.
```

---

## S5 — Route all restart triggers through request_restart(); drop legacy marker

**Title:** `fix(restart): route all restart triggers through request_restart; drop legacy marker`

**Branch:** `claude/simplify-restart-signals`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/simplify-restart-signals?expand=1)

**Body:**
```
## Problem
The legacy combined .koan-restart marker was written "for backward compat" but nothing reads it
— both consumers poll their own per-process marker (.koan-restart-run / .koan-restart-bridge).
Worse, the REST API (/v1/restart, /v1/update) and the dashboard /api/agent/restart endpoint
signalled restarts by touching ONLY that dead legacy file, so those endpoints were silent no-ops
(the run loop and bridge never saw the request). The dashboard's /api/config/restart already did
it correctly via request_restart().

## Changes
- routes_admin /v1/restart + /v1/update, dashboard /api/agent/restart: call request_restart() so
  both per-consumer markers are written and the restart actually fires
- request_restart() writes only the two live markers; the deprecated legacy .koan-restart is no
  longer written
- run._startup_delay() wakes on .koan-restart-run instead of the dead legacy file
- restart_manager: keep target=None → .koan-restart read mapping for any out-of-tree poller,
  documented as deprecated read-only compat

## Test
Tests updated across test_restart_manager, test_restart, test_api_admin, test_dashboard,
test_startup_delay to assert both consumer markers are written and the legacy file is not.
check_restart/clear_restart (target="run"/"bridge") behavior unchanged.
```

---

## S2 — Single canonical mission-identity function

**Title:** `refactor(missions): add canonical_mission_key as single mission-identity source`

**Branch:** `claude/simplify-canonical-mission-key`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/simplify-canonical-mission-key?expand=1)

**Body:**
```
## Problem
stagnation_monitor._mission_key carried its own _STRIP_FOR_KEY_RE for deriving a stable mission
identity (strip timestamps + [r:N] + [complexity:X] + "- "). There was no shared canonical
function for "stable mission identity".

## Changes
- Add missions.canonical_mission_key() as the single source of truth.
- stagnation_monitor._mission_key hashes its output — the SHA-256 is byte-identical, so existing
  stagnation/crash trackers keep matching.
- Deliberately NOT folded in (different concerns, would change behavior): mission_history.
  _normalize_key (strips project tag for cross-project dedup) and recover._strip_recovery_counter
  (strips only [r:N], keeps timestamps — display helper). Both get a comment pointing at the
  canonical function.

## Test
TestCanonicalMissionKey; test_golden_hash_unchanged (locks the sha256 so future drift can't
silently orphan trackers); test_delegates_to_canonical_mission_key.
```

---

## D3 — Stagnation unified retry counter semantics

**Title:** `docs(stagnation): document unified retry tracker structure and counter semantics`

**Branch:** `claude/fix-unified-retry-cap` (appended to B6)

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/fix-unified-retry-cap?expand=1)

**Body:**
```
(see B6 PR body — this documentation commit is part of the same branch)

The module-level comment block above the retry-tracking functions only described stagnation
requeues. Add an expanded block explaining:
- Both stagnation and crash failures share .mission-retries.json
- Full entry structure (count, crash_count, total_attempts, pattern_type)
- Which function increments which counter and when
- When counters are cleared (genuine success only)
```

---

## D5 — trust_stdout=False CLAUDE.md note

**Title:** `docs(claude.md): document trust_stdout=False requirement for skill runners`

**Branch:** `claude/docs-trust-stdout`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/docs-trust-stdout?expand=1)

**Body:**
```
## Problem
Skill runners write structured agent transcripts to stdout (not raw CLI output). Passing
trust_stdout=True (the default) causes false-positive quota detection because the transcript
may contain quota-error substrings inside logged tool output. This is a non-obvious constraint
for anyone adding a new skill runner. Nothing in CLAUDE.md documented it.

## Changes
- Added trust_stdout=False note to the 'Adding a new core skill' step 2 in CLAUDE.md.
```

---

## D6 — Complex mission recovery documentation

**Title:** `docs(recover): document complex mission (### format) recovery behavior`

**Branch:** `claude/docs-complex-missions`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/docs-complex-missions?expand=1)

**Body:**
```
## Problem
recover.py's module docstring described simple crash recovery only. The ### sub-header format
for complex (multi-step) missions was undocumented — no mention of how ### blocks are treated
during recovery, what the block boundary is, or the relationship to the _flush_in_progress_to_failed()
secondary safety net.

Note: the underlying code issue (### blocks silently skipped) was already fixed by B2
(claude/fix-complex-mission-recovery, merged to main).

## Changes
- Added a 'Complex mission format' section to recover.py's module docstring explaining:
  - ### blocks are treated as atomic units (requeued or escalated together)
  - Block boundary: next blank line or next ### header
  - Cross-link to _flush_in_progress_to_failed() as secondary safety net
```

---

## D7 — CLAUDE.md run.py / mission_executor boundary

**Title:** `docs(claude.md): clarify run.py vs mission_executor vs mission_runner boundary`

**Branch:** `claude/docs-claudemd-update`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/docs-claudemd-update?expand=1)

**Body:**
```
## Problem
CLAUDE.md's architecture section described run.py generically without naming which functions
stay there, and mission_executor.py (extracted from run.py) was missing from the key modules
list entirely. A developer reading CLAUDE.md couldn't tell which of the three files (run.py,
mission_executor.py, mission_runner.py) to look at for a given concern.

## Changes
- Updated run.py one-liner to enumerate the functions that remain there:
  run_claude_task, _finalize_mission, _classify_and_handle_cli_error, _probe_exit0_quota
- Added mission_executor.py to the Agent loop pipeline section (per-iteration dispatch layer)
- Updated mission_runner.py description to clarify it is stateless pipeline helpers
  (no side effects on missions.md)
```

---

## D8 — recovery.jsonl in CLAUDE.md instance section

**Title:** `docs(claude.md): add recovery.jsonl to instance directory section`

**Branch:** `claude/document-recovery-jsonl`

**PR:** [#1969](https://github.com/Anantys-oss/koan/pull/1969) (open) — split out of S6 so the docs change lands independently of the recover.py refactor.

**Body:**
```
## Summary
Added recovery.jsonl to the instance/ directory listing in CLAUDE.md with a description
of its schema and interpretation (many entries = crash loop candidate).

Operators debugging repeated crashes previously had no documentation pointing them at
recovery.jsonl as the audit trail for crash-recovery events.
```

---

## D9 — outbox-sending.md in CLAUDE.md instance section

**Title:** `docs(claude.md): document outbox-sending.md staging file in instance section`

**Branch:** `claude/simplify-outbox-append` (appended to S3)

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/simplify-outbox-append?expand=1)

**Body:**
```
(see S3 PR body — this documentation commit is part of the same branch)

Added outbox-sending.md to the instance/ directory listing in CLAUDE.md explaining it is a
crash-safety two-phase write staging file, what happens if it persists (potential duplicate
Telegram sends), and that it can be safely deleted manually if messages appear stuck.
```

---

## D10 — Mission lifecycle state diagram

**Title:** `docs(claude.md): add mission lifecycle state diagram`

**Branch:** `claude/docs-state-diagram`

[Open PR →](https://github.com/Anantys-oss/koan/compare/main...exciton:koan:claude/docs-state-diagram?expand=1)

**Body:**
```
## Problem
CLAUDE.md described mission states in prose only — no visual summary of the lifecycle, the
requeue path, or the two safety nets for stale In Progress missions. Developer on-boarding
required reading the full analysis-state-flow.md to understand the state machine.

## Changes
- Added a 'Mission lifecycle' subsection to the Architecture section of CLAUDE.md
- ASCII state diagram showing:
  - Pending → In Progress → Done / Failed transitions
  - The requeue path back to Pending (stagnation retry, crash recovery, transient error retry)
  - Both safety nets for stale In Progress: recover.py (startup) and
    _flush_in_progress_to_failed() (per-mission-start)
  - The key missions.py function responsible for each transition
```
