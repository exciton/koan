# Kōan — State Consistency Bugs, Simplifications, and Documentation Gaps

Analysis based on reading: `missions.py`, `recover.py`, `mission_executor.py`,
`stagnation_monitor.py`, `outbox_manager.py`, `awake.py`, `loop_manager.py`,
and cross-referencing the state-flow analysis in `analysis-state-flow.md`.

---

## 1. Bugs and Race Conditions

### P1 — Critical

---

#### B1 — Stagnation retry counter resets after every requeue (infinite stagnation loop possible) ✅ FIXED

**Branch:** `claude/fix-stagnation-key`

**PR description:**
> **fix(stagnation): strip lifecycle markers from mission key before hashing**
>
> `_mission_key()` previously hashed the raw mission title including ⏳/▶ timestamps,
> `[r:N]` recovery counters, and `[complexity:X]` tags. After `requeue_mission()` strips
> those markers, the re-picked mission acquires new timestamps — producing a different hash
> and silently resetting the stagnation retry counter on every cycle.
> `max_retry_on_stagnation` was therefore never reached and a persistently-stagnating
> mission would loop indefinitely.
>
> Now `_mission_key()` strips all lifecycle markers before hashing, making the key stable
> across requeue cycles. Also fixes B13 (same root cause). Tests added.

**Files:** `koan/app/stagnation_monitor.py:379-381`, `koan/app/run.py` `_finalize_mission`

**Problem:**
`_mission_key()` hashes the raw `mission_title` string which includes lifecycle
timestamps (`⏳(2026-01-01T12:00) ▶(2026-01-01T12:05)`).

After stagnation, `requeue_mission()` strips those timestamps. When the mission
is re-picked from Pending it acquires new timestamps. The new `_mission_key` is
therefore different from the key used to increment the counter. The retry count
for the new key is 0, so the `retry_count < max_retry` check always passes —
`max_retry_on_stagnation` is never reached.

A mission that stagnates every run will be requeued indefinitely.

---

#### B2 — Complex missions (`### ` format) in In Progress are never recovered ✅ FIXED

**Branch:** `claude/fix-complex-mission-recovery`

**Files:** `koan/app/recover.py`, `koan/tests/test_recover.py`

**Problem:**
`recover_missions()` scans In Progress line-by-line. Lines starting with `### ` 
set `in_complex_mission = True` and are kept in `remaining_in_progress` — they
are never moved to Pending or Failed. On every restart, complex missions
continue to accumulate in In Progress without recovery.

**Fix applied:**
- Replaced the simple `in_complex_mission` flag with a `_finalize_complex_block()` helper
- Entire block (header + sub-items) is collected and classified as a unit
- Recoverable: entire block moves to Pending with `[r:N]` in the `### ` header
- Unrecoverable: header line moves to Failed
- Incomplete sub-items (no strikethrough) are picked as individual missions by `extract_next_pending()`
- Also incorporates the B7 fix (single-use journal_available flag)

<details>
<summary>PR description template</summary>

**Title:** `fix(recover): recover complex ### mission blocks from In Progress`

**Body:**

## Problem
Multi-step missions using the `### Header
- Step 1
- Step 2` format were silently skipped by crash recovery. They stayed in In Progress indefinitely and were never re-queued.

## Changes
- Replaced simple `in_complex_mission` flag with `_finalize_complex_block()` inner function
- Entire block (header line + sub-items) collected and classified together using the header as the mission key
- Recoverable block: all lines moved to Pending with `[r:N]` in the `### ` header
- Sub-items with strikethrough (completed steps) preserved; incomplete steps picked as normal missions
- Also incorporates B7 fix: `has_pending_journal` consumed for first claiming mission only

## Test
70 tests pass. `test_skip_complex_mission` renamed to `test_recover_complex_mission_block` with updated assertions.
</details>

---

#### B3 — `create_pending_file()` overwrites checkpoint recovery context ✅ FIXED

**Branch:** `claude/fix-checkpoint-overwrite`

**Files:** `koan/app/loop_manager.py`, `koan/tests/test_loop_manager.py`

**Problem:**
`recover.py._inject_checkpoint_context()` appends structured checkpoint context
to `journal/pending.md` at startup. When the recovered mission is started,
`create_pending_file()` overwrites pending.md with a fresh header, destroying
the checkpoint context before Claude reads it.

**Fix applied:**
- Added `_RECOVERY_CONTEXT_SENTINEL = "## Recovery Context (from previous interrupted run)"`
- Before writing, `create_pending_file()` checks for the sentinel in the existing pending.md
- If found, appends the checkpoint section after the new header
- Regular pending.md content (no sentinel) is not carried over

<details>
<summary>PR description template</summary>

**Title:** `fix(loop_manager): preserve checkpoint recovery context in create_pending_file`

**Body:**

## Problem
`recover.py._inject_checkpoint_context()` writes structured recovery data to pending.md at startup. `create_pending_file()` called when the mission starts then overwrote it completely, making the "partial" state classification a no-op.

## Changes
- `create_pending_file()` reads existing pending.md before writing
- If the recovery context sentinel is found, the checkpoint section is appended after the new header
- Also wraps `journal_dir.mkdir()` in `contextlib.suppress(OSError)` (B14 fix)

## Test
Added `test_preserves_recovery_context_from_pending_md` and `test_does_not_preserve_regular_pending_md`.
</details>

---



#### B4 — Telegram `offset` is in-memory only; messages re-delivered after bridge restart ✅ FIXED

**Branch:** `claude/fix-telegram-offset`

**Files:** `koan/app/awake.py`

**Problem:**
`offset = None` is a local variable in `main()`. After bridge restart via
`os.execv()`, `offset` is reinitialized to `None`.

Telegram ACK works by calling `getUpdates(offset=N)` which confirms updates
through N-1 server-side. If the bridge is restarted BETWEEN receiving a batch
of updates and calling `getUpdates` again (which would have confirmed them),
all messages in that batch are re-delivered. Each message is then processed
again: missions queued twice, commands executed twice.

**Example window:**
```
getUpdates(offset=5) → returns updates [5, 6, 7]
Process update 5: in-memory offset = 6
Process update 6: in-memory offset = 7
Process update 7: in-memory offset = 8
<-- bridge crashes here -->
getUpdates(offset=8) is NEVER called → 5,6,7 not confirmed on Telegram server
Next run: getUpdates(None) → gets 5,6,7 again → queued as duplicate missions
```

**Fix:**
Persist the last `offset` to disk atomically (e.g., `instance/.telegram-offset`)
after each successful batch. On startup, read this file to resume from the
correct offset.

**Fix applied:**
- Added `_load_offset()` / `_save_offset()` helpers using `instance/.telegram-offset.json`
- `_save_offset(offset)` called after every `update_id` advance (atomic write)
- `_load_offset()` called at bridge startup to resume from persisted offset
- Tests: `_load_offset` mocked in `TestMainLoop` autouse fixture

<details>
<summary>PR description template</summary>

**Title:** `fix(awake): persist Telegram polling offset across bridge restarts`

**Body:**

## Problem
Telegram `offset` was in-memory only. After bridge restart the offset reset to `None`, causing Telegram to re-deliver all updates from the ~60s window before the restart. This could cause duplicate mission queuing.

## Changes
- `_save_offset(offset)` persists to `instance/.telegram-offset.json` atomically on each `update_id` advance
- `_load_offset()` reads the persisted value at startup
- `TestMainLoop` autouse fixture mocks `_load_offset` to return `None` for isolation

## Test
267 tests pass. 1 pre-existing root-permission test excluded.
</details>

---

### P2 — Moderate

---

#### B5 — `_flush_in_progress_to_done()` marks abandoned missions as Done (✅) not Failed ✅ FIXED

**Branch:** `claude/fix-flush-to-failed`

**Files:** `koan/app/missions.py`, `koan/tests/test_missions.py`

**Problem:**
`start_mission()` calls `_flush_in_progress_to_done()` as a safety net before
inserting a new In Progress entry. This marks any stale In Progress missions
with a ✅ Done marker.

Under the expected flow, `recover.py` handles stale In Progress missions at
startup, so this code path should not fire. But it does fire for:
- Complex missions missed by `recover.py` (see B2)
- Any edge-case where `recover.py` fails silently (import errors, malformed
  missions.md sections)

Marking a crashed/abandoned mission as Done creates false history — the user
sees a ✅ for a mission that never actually completed.

**Fix applied:**
- Renamed `_move_in_progress_to_done()` → `_flush_abandoned_in_progress()`
- Changed marker from ✅ Done to ❌ Failed with `[flushed]` tag
- Inserts into the Failed section (creates it if absent)
- All related tests updated to assert on `sections["failed"]`

<details>
<summary>PR description template</summary>

**Title:** `fix(missions): redirect abandoned in-progress missions to Failed section`

**Body:**

## Problem
When `start_mission()` finds stale In Progress missions (sanity enforcement), it was silently moving them to Done with a ✅ marker — creating false history that the work completed successfully. This fires when `recover.py` misses a mission (complex mission blocks, import errors).

## Changes
- Renamed `_move_in_progress_to_done()` → `_flush_abandoned_in_progress()` for clarity
- Changed the marker to ❌ with a `[flushed]` tag inserted into the Failed section
- Creates the Failed section if it doesn't already exist
- Updated all test assertions to check `sections["failed"]` instead of `sections["done"]`

## Test
All 415 existing tests pass (1 pre-existing root-permission test excluded).
</details>

---

#### B6 — Two independent retry systems accumulate silently; combined limit undocumented ✅ FIXED

**Branch:** `claude/fix-unified-retry-cap`

**Files:** `koan/app/stagnation_monitor.py`, `koan/app/recover.py`, `koan/app/run.py`, `koan/app/config.py`, tests

**Problem:**
There were two separate retry counters with no shared awareness:
- `[r:N]` embedded in mission text in missions.md: tracks crash-recovery attempts (max 3, hardcoded)
- `.stagnation-retries.json`: tracks stagnation-requeue attempts (max configurable)

A mission could cycle between stagnating and crashing indefinitely because neither counter knew about the other, and clearing one on any non-stagnation exit reset cross-system progress.

**Fix applied (three commits, one branch):**

Commit 1 — cross-system ceiling (`total_attempts`):
- Added `total_attempts` field to stagnation tracker; incremented by both stagnation requeues and crash-recovery on requeue
- New `max_total_retries` config key (default 0 = disabled) acts as a single shared cap
- `clear_retry_count(clear_total=False)` preserves `total_attempts` across crash cycles
- Both `classify_mission_state()` and the stagnation requeue path in `run.py` check the combined cap

Commit 2 — unified storage (remove `[r:N]` tags from missions.md):
- Renamed `.stagnation-retries.json` → `.mission-retries.json` with auto-migration
- Added `crash_count` field alongside existing stagnation `count`; new `get_crash_count()` / `increment_crash_count()` API
- `increment_crash_count()` also increments `total_attempts` for combined cap
- `max_crash_retries` config key (default 3) replaces hardcoded `MAX_RECOVERY_ATTEMPTS`
- `classify_mission_state()` now takes `crash_count: int` instead of parsing `[r:N]` from mission text
- Backward compat: legacy `[r:N]` tags in existing missions.md are read for classification but never seeded to tracker; stripped on next write

Commit 3 — counter lifetime (preserve in Failed; clear on human retry):
- Counter is **not** cleared when stagnation cap is hit or mission is escalated to Failed — the human can inspect `.mission-retries.json` to see why the mission stopped
- Counter is cleared at `start_mission()` time **only when a cap was previously hit** (`stag_count >= max_retry` OR `crash_count >= max_crash_retries` OR `total >= max_total`), signalling a deliberate human retry
- Ongoing stagnation-retry requeus (count < cap) keep their counter intact so the cap check still fires on the next cycle
- New `_clear_if_cap_hit()` helper encapsulates the conditional clear logic

**Counter clear table:**

| Event | Counter action |
|---|---|
| Crash → Failed | preserve all |
| Stagnation cap hit → Failed | preserve all |
| Escalated unrecoverable → Failed | preserve all |
| Stagnation retry requeue (count < cap) | preserve (cap still needs to fire) |
| Mission success | full clear |
| `start_mission()` with cap-hit counter | full clear (human deliberate retry) |

<details>
<summary>PR description</summary>

See `docs/pr-links.md` — B6 section.
</details>

---

#### B7 — `has_pending_journal` flag in `recover.py` applies to all in-progress missions ✅ FIXED

**Branch:** `claude/fix-recover-pending-journal-scope` (also in `claude/fix-complex-mission-recovery`)

**Files:** `koan/app/recover.py`, `koan/tests/test_recover.py`

**Problem:**
`has_pending_journal` is computed ONCE before the loop over all In Progress
missions. If there are N in-progress missions (unusual but possible under bug
conditions), all of them are classified as "partial" if any `pending.md`
exists — even if only one mission created it.

"partial" missions get checkpoint context injected, which currently does nothing
useful (B3), but the classification itself affects the audit log.

**Fix applied:**
- `journal_available` flag replaces the per-call `has_pending_journal` inside the loop
- Consumed (set to `False`) after the first mission claims "partial" state
- Tested in `TestPendingJournalSingleUse::test_only_first_mission_gets_partial_state`

<details>
<summary>PR description template</summary>

**Title:** `fix(recover): consume pending.md context for first mission only`

**Body:**

## Problem
`has_pending_journal` was computed once and applied to all in-progress missions. With multiple stale missions, all were classified as "partial" even though pending.md was written by exactly one interrupted run.

## Changes
- `journal_available` local flag initialized from `has_pending_journal`
- Set to `False` after first "partial" classification, so subsequent missions are correctly classified as "dead"

## Test
Added `TestPendingJournalSingleUse` verifying second mission gets "dead" state.
</details>

---

#### B8 — GitHub `processed_comments` set is in-memory; bridge restart causes duplicate comment dispatch ✅ ALREADY FIXED

**Files:** `koan/app/github_notifications.py`, `koan/app/github_notification_tracker.py`, `koan/app/github_command_handler.py`

**Problem (as originally analysed):**
The set of already-processed GitHub notification comment IDs is in-memory
(`processed_comments` or equivalent). After bridge restart, the set is empty.
The next GitHub poll returns recently-processed comments (especially those
acknowledged via reaction in the SAME poll cycle as the restart).

Scenario: GitHub @mention arrives → reaction added → mission queued → bridge
restarts → GitHub poll returns the same comment (reaction was just added,
GitHub notification system has ~60s lag) → duplicate mission queued.

The reaction IS checked in `check_already_processed()` but only if the reaction
write in step 1 was processed by GitHub by the time step 2 polls. Under load
the reaction can appear after the next poll.

**Resolution:**
On review of the current code, this is already handled by a persistent tracker —
no further change required. `github_notification_tracker.py` maintains
`instance/.koan-github-processed.json` (comment IDs) and
`instance/.koan-github-processed-threads.json` (assignment-notification keys),
both with a 7-day TTL and a 5000-entry cap. The in-memory `BoundedSet` is now
just a fast first-level cache:

- **Write side:** `github_command_handler.py` calls `track_comment(instance_dir, comment_id)`
  at dispatch time (both for inline-handled commands and queued slash missions),
  persisting the ID before/alongside the mission queue.
- **Read side:** `check_already_processed()` consults the in-memory set, then the
  persistent tracker (`is_comment_tracked()`), then GitHub reactions — in that
  order — so a restart that empties the in-memory set still finds the comment in
  the on-disk tracker.

This is the same persistence pattern as the B4 Telegram-offset fix. The original
proposed file name (`.github-processed-comments.json`, 24h TTL) differs from what
shipped (`.koan-github-processed.json`, 7-day TTL) but the behaviour is equivalent
and stronger. Only this doc was stale.

---

#### B9 — `start_mission()` silently succeeds when mission not found in Pending ✅ FIXED

**Branch:** `claude/fix-start-mission-return`

**Files:** `koan/app/run.py`, `koan/app/mission_executor.py`

**Problem:**
If `_remove_pending_by_text()` returns `None` (mission text doesn't match
anything in Pending), `start_mission()` returns the content unchanged. The
caller in `run.py` does not check whether the transition actually occurred.

The mission is then executed with it still visible in the Pending section.
During execution:
- `/list` command shows the mission as Pending
- Users may queue it again, creating a duplicate
- If the process crashes mid-execution, `recover.py` finds nothing in In Progress
  and does not recover → the mission stays in Pending looking untouched

This can be triggered by missions with embedded double-spaces (fixed in
`_remove_item_by_text` with `re.sub(r"\s+", " ")` but only for whitespace
not for other normalisation differences), or by edge cases in
`_COMPLEXITY_TAG_RE` stripping.

**Fix applied:**
- `_start_mission_in_file()` now returns `bool` (True = confirmed in In Progress)
- After locked write, reads resulting content via `parse_sections()` to verify mission is in In Progress
- Logs WARNING on mismatch
- `mission_executor.py` aborts the run when transition is unconfirmed (returns `False`)

<details>
<summary>PR description template</summary>

**Title:** `fix(run): verify start_mission transition and abort on mismatch`

**Body:**

## Problem
`_start_mission_in_file()` discarded the return value of `modify_missions_file()` and could not detect whether the mission actually moved to In Progress. Silent failure left the mission in Pending while Claude executed it.

## Changes
- `_start_mission_in_file()` returns `bool`
- Reads In Progress section via `parse_sections()` after write to confirm transition
- `mission_executor.py` aborts the run (returns `False`) when unconfirmed

## Test
749 tests pass.
</details>

---

### P3 — Minor

---

#### B10 — `requeue_mission()` inserts at top of Pending queue (undocumented priority behavior)

**Files:** `koan/app/missions.py:1254-1261`

**Problem:**
`requeue_mission()` inserts the re-queued mission at the TOP of the Pending
section (first item after the header). `insert_mission()` inserts at the BOTTOM
(FIFO). This means quota-requeued or auth-requeued missions skip the queue
ahead of all other pending work.

This is intentional (you want the interrupted work to resume immediately) but
is not documented and surprises operators who see queue ordering change
unexpectedly after a quota pause.

**Fix:** Add a docstring note to `requeue_mission()` explaining the
top-of-queue insertion and why it is intentional.

---

#### B11 — No telemetry when `complete_mission()` / `fail_mission()` silently finds nothing ✅ FIXED

**Branch:** `claude/fix-finalize-mission-telemetry`

**Files:** `koan/app/missions.py`, `koan/app/run.py`, `koan/tests/test_missions.py`, `koan/tests/test_run.py`

**Problem:**
`_move_pending_to_section()` returns the content unchanged if the mission is
not found in Pending or In Progress. The caller never knows whether the
transition happened. Silent no-ops here mask data corruption or race
conditions.

Worse, `run.py`'s `_update_mission_in_file()` inferred success by comparing
content *before* and *after* the locked write. Because
`prune_completed_sections()` runs unconditionally on that path, an oversized
Done/Failed section makes the content differ even on a genuine no-op — so an
absent mission was wrongly reported as moved (returning `True`), masking a
stuck mission that re-dispatches on every loop.

**Fix applied:**
- `_move_pending_to_section()` now returns a `(content, found: bool)` tuple
- New `complete_mission_checked()` / `fail_mission_checked()` expose the found
  flag; `complete_mission()` / `fail_mission()` keep their `str` return as thin
  wrappers (no caller/test churn)
- `_update_mission_in_file()` captures found-status via a closure flag (the same
  pattern `insert_pending_mission()` uses) and bases its WARNING + `bool` return
  on it — decoupled from the pruning side effect

<details>
<summary>PR description</summary>

See `docs/pr-links.md` — B11 section.
</details>

---

#### B12 — `prune_completed_sections()` is called inline in `_update_mission_in_file()`

**Files:** `koan/app/run.py` `_update_mission_in_file()`

**Problem:**
Pruning Done/Failed history is a side effect of mission finalization. If
pruning fails or is misconfigured, it silently modifies history during a
`fail_mission()` call, making debugging harder. History trimming and mission
finalization are separate concerns.

**Fix (low priority):** Extract pruning into a scheduled maintenance step
(e.g., once per session at startup or a dedicated post-mission step) rather
than coupling it to the finalization path.

---

#### B13 — Stagnation `[r:N]` key contamination: crash-recovered missions never hit stagnation cap ✅ FIXED

**Branch:** `claude/fix-stagnation-key` (same fix as B1)

**Files:** `koan/app/stagnation_monitor.py:379`

**Problem:**
(Relates to B1.) When `recover.py` requeues a mission with an incremented
`[r:N]` tag, the tag becomes part of the mission text that `_mission_key()`
hashes. So not only do timestamps change the key (B1), but `[r:1]` produces a
different key than `[r:2]`, meaning the stagnation retry history is abandoned
at each crash-recovery cycle, not just each timestamp cycle.

**Fix:** Same as B1 — `_STRIP_FOR_KEY_RE` now strips `[r:N]` tags.

---

#### B14 — `pending.md` journal dir creation failure aborts pending.md write ✅ FIXED

**Branch:** `claude/fix-checkpoint-overwrite` (B3 branch; B14 fix bundled in same commit)

**Files:** `koan/app/loop_manager.py`

**Problem (low severity):** If the per-day journal directory creation fails (disk full, permission denied), the exception propagated and aborted the pending.md write, leaving no checkpoint context for Claude.

**Fix applied:**
- `journal_dir.mkdir()` wrapped in `contextlib.suppress(OSError)`
- On failure the dated subdir is skipped but `pending.md` is still written to `instance/journal/`

<details>
<summary>PR description template</summary>

**Title:** `fix(loop_manager): suppress OSError on dated journal subdir creation`

**Body:**

## Problem
If `instance/journal/YYYY-MM-DD/` could not be created (disk full, permissions), pending.md was never written, losing checkpoint context for Claude.

## Changes
- `journal_dir.mkdir()` replaced with `contextlib.suppress(OSError): journal_dir.mkdir()`
- pending.md write proceeds regardless

## Note
This fix is bundled in the `claude/fix-checkpoint-overwrite` (B3) branch commit.
</details>

---

## 2. State Simplifications

---

### S1 — Merge or cross-link the two crash-recovery systems

`recover.py` (startup, `[r:N]` in missions.md) and `_flush_in_progress_to_done`
(inside `start_mission()`) are independent safety nets for the same scenario.
Their interaction is not documented. Operators debugging stale In Progress
missions must check both code paths.

**Suggested change:** Add a comment at the top of `_flush_in_progress_to_done`
pointing to `recover.py` and explaining when this path fires vs when `recover.py`
fires. Ideally add a log line so operators can see in the logs that a flush
occurred.

---

### S2 — Unify the "mission key" concept across stagnation and recovery counters

Both `stagnation_monitor._mission_key` and `recover.py._strip_recovery_counter`
deal with extracting a stable identity from mission text. There should be a
single canonical function (e.g., `missions.canonical_mission_key(text)`) that:
1. Strips lifecycle timestamps (⏳, ▶, ✅, ❌)
2. Strips `[r:N]` recovery counters
3. Strips `[complexity:X]` tags
4. Strips the `- ` prefix

Used by: `_mission_key()` in stagnation, `mission_history.py`, dedup checks.

---

### S3 — Consolidate outbox writing into a single function

`append_to_outbox()` is called from both `run.py` and `awake.py` (via
`notify.py`). The `OutboxManager.requeue()` method uses raw `open(outbox_file, "a")`
without going through `append_to_outbox`. This creates two slightly different
append paths for the same file.

**Suggested change:** Have `requeue()` call `append_to_outbox()` rather than
duplicating the open/flock/write pattern.

---

### S4 — Replace `_startup_notified` / `_boot_notified` dual flag pair with an enum

`run.py` maintains two booleans (`_startup_notified`, `_boot_notified`) to
distinguish "first iteration since start" vs "first iteration after resume".
Their semantics overlap and neither is reset consistently.

**Suggested change:** Replace with a single `_startup_phase: Literal["boot", "resume", "running"]`
state variable. Clearer semantics, fewer boolean combinations to reason about.

---

### S5 — Signal file proliferation: consolidate restart signals

Three restart-related files exist: `.koan-restart-bridge`, `.koan-restart-run`,
`.koan-restart` (legacy combined). The legacy file is written alongside the
two new ones but not consumed by anything. It accumulates on disk.

**Suggested change:** Remove the legacy `.koan-restart` write from
`restart_manager.request_restart()`. If backward compat is needed, document
that it is deprecated and stop writing it.

---

### S6 — `check_pending_journal()` and `has_pending_journal` in `recover.py` are not TOCTOU-safe

`check_pending_journal()` reads `pending.md` (returns True/False), then later
`recover_missions()` reads it again inside `_recover_transform()`. Between these
two reads, `run.py` could create a new `pending.md` (if it started before
`recover_missions()` completed, which should not happen in normal startup
sequencing but could during tests). The double-read is redundant.

**Suggested change:** Remove the standalone `check_pending_journal()` call from
the startup flow and rely solely on the read inside `_recover_transform()`.

---

## 3. Documentation Gaps

---

### D1 — `start_mission()` docstring omits the sanity-flush side effect

**File:** `koan/app/missions.py:1139`

Current docstring: "Move a mission from Pending to In Progress with a started
timestamp."

Missing: "As a side effect, any existing In Progress missions are silently moved
to Done via `_flush_in_progress_to_done()`. Under normal operation this path
never fires because `recover.py` runs at startup. If it does fire, the In
Progress mission will appear as Done even if it never completed successfully."

---

### D2 — `_flush_in_progress_to_done()` is undocumented as a safety net distinct from `recover.py`

**File:** `koan/app/missions.py:1091`

The existing comment says "Sanity enforcement: only one mission should be in
progress at a time." It does not explain:
- That this is a second line of defence after `recover.py`
- When each fires (startup vs mission-start time)
- That it marks missions Done rather than Failed (a deliberate choice that
  should be documented, or corrected — see B5)

---

### D3 — Stagnation retry counter semantics undocumented (per-run vs per-mission)

**File:** `koan/app/stagnation_monitor.py:363-380`

The module-level comment block above the retry-tracking functions says "counters
are keyed by a stable SHA-256 of the mission title". This implies cross-run
stability, but the title includes timestamps, making the key effectively
per-run (see B1). The word "stable" is misleading.

**Fix:** Update the comment to be explicit: "keyed by SHA-256 of the mission
title TEXT including timestamps — effectively per-execution, not per-mission."
Then either fix B1 or document this as a known limitation.

---

### D4 — `requeue_mission()` priority-insertion is undocumented

**File:** `koan/app/missions.py:1217`

The docstring says "Move a mission from In Progress (or Failed) back to Pending."
It does not mention that the mission is inserted at the TOP of the Pending queue,
not the bottom. This is a behavior difference from `insert_mission()` that
matters for queue ordering and operator expectations.

---

### D5 — `trust_stdout=False` in skill dispatch deserves CLAUDE.md mention

**File:** `koan/app/mission_executor.py:144-159`

The inline comment explains why skill stdout is not trusted for quota detection.
This is a non-obvious design decision that affects any future developer adding
a new skill runner. The CLAUDE.md "Conventions" section should note: "Skill
runners write summarized agent transcripts to stdout (DATA, not CLI output).
Use `trust_stdout=False` in `_classify_and_handle_cli_error()` calls for skill
dispatch to prevent false-positive quota detection."

---

### D6 — Complex missions (`### ` format) in missions.md are undocumented as unsupported by recovery

**File:** `koan/app/recover.py`, `koan/app/missions.py`

The `### project:X` sub-header format inside the Pending section is documented
in `docs/users/user-manual.md`. But there is no documentation that complex
missions using the `### ` format within In Progress are skipped by crash
recovery (see B2). Operators who use the `### ` format for multi-step missions
should know they could get permanently stuck in In Progress after a crash.

---

### D7 — CLAUDE.md lists `mission_executor.py` as new but `run.py` still contains many cycle functions

**File:** `CLAUDE.md` Architecture section

CLAUDE.md says "**`run.py`** (agent loop): Pure-Python main loop with restart
wrapper. Picks pending missions, transitions them through lifecycle, executes
via Claude Code CLI or direct skill dispatch." and separately lists
`mission_executor.py`. But `run.py` still contains `run_claude_task()`,
`_finalize_mission()`, `_classify_and_handle_cli_error()`,
`_probe_exit0_quota()` and other core execution functions. The boundary between
`run.py` and `mission_executor.py` is not clearly documented.

**Fix:** Update CLAUDE.md to describe which execution responsibilities remain
in `run.py` vs which are in `mission_executor.py`, and what the intended
long-term boundary is.

---

### D8 — `recover.py` recovery event log (`recovery.jsonl`) is undocumented in CLAUDE.md

**File:** `koan/app/recover.py:107-138`

`recovery.jsonl` is written for every crash recovery event as an audit trail.
It is listed in Section 1.4 of `analysis-state-flow.md` but not in CLAUDE.md's
Instance directory section. An operator debugging repeated crashes would not
know to look at this file.

**Fix:** Add `recovery.jsonl` to the Instance directory section of CLAUDE.md.

---

### D9 — `outbox-sending.md` staging file undocumented

**File:** `koan/app/outbox_manager.py:103-123`

The staging file `instance/outbox-sending.md` is created between reading and
sending outbox content (crash safety). If the bridge crashes mid-send, this
file persists and is re-processed on restart — which can cause duplicate
Telegram messages. The `OutboxManager.recover_staged()` docstring mentions
this, but there is no user-facing documentation (CLAUDE.md, user-manual) that
explains this file exists and what it means if found.

---

### D10 — State transition diagram in CLAUDE.md is absent

**File:** `CLAUDE.md`

CLAUDE.md describes the architecture in prose but contains no state diagram for
the mission lifecycle (Pending → In Progress → Done/Failed/Pending). The
`analysis-state-flow.md` Section 12 fills this gap, but should be summarised in
CLAUDE.md for developer on-boarding.

**Fix:** Add a compact Mermaid state diagram or ASCII art to the Architecture
section showing the mission lifecycle and the key state transitions (including
crash recovery and requeue paths).
