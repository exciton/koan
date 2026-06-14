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
