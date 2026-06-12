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

#### B2 — Complex missions (`### ` format) in In Progress are never recovered

**Files:** `koan/app/recover.py:244-256`

**Problem:**
`recover_missions()` scans In Progress line-by-line. Lines starting with `### ` 
set `in_complex_mission = True` and are kept in `remaining_in_progress` — they
are never moved to Pending or Failed. On every restart, complex missions
continue to accumulate in In Progress without recovery.

Because `_flush_in_progress_to_done()` in `start_mission()` eventually marks
any stale In Progress entries as Done (✅), a complex mission that timed out or
crashed gets a false success marker.

**Fix:**
Treat `### ` complex mission blocks the same as simple `- ` missions during
recovery — extract the first `- ` line within the block as the recovery needle,
move the whole block to Pending with incremented `[r:N]`.

---

#### B3 — `create_pending_file()` overwrites checkpoint recovery context

**Files:** `koan/app/loop_manager.py:187-230`, `koan/app/recover.py:371-405`

**Problem:**
`recover.py._inject_checkpoint_context()` appends structured checkpoint context
to `journal/pending.md` at startup. A few seconds later, when the recovered
mission is actually picked and started, `create_pending_file()` calls
`atomic_write(pending_path, content)` with a fresh header — completely
overwriting the checkpoint context before Claude reads it.

Result: missions classified as "partial" (has checkpoint) behave identically
to "dead" (no checkpoint) in practice — the `partial` classification is
currently a no-op.

**Fix:**
Option A (minimal): In `create_pending_file()`, check for existing checkpoint
context in the staging file and append it after the header.

Option B (clean): Remove `_inject_checkpoint_context()` from `recover.py`.
Instead, pass the checkpoint data forward to `create_pending_file()` and have
it incorporate checkpoint sections directly.

---

#### B4 — Telegram `offset` is in-memory only; messages re-delivered after bridge restart

**Files:** `koan/app/awake.py:833`, `koan/app/awake.py:857`

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

---

### P2 — Moderate

---

#### B5 — `_flush_in_progress_to_done()` marks abandoned missions as Done (✅) not Failed

**Files:** `koan/app/missions.py:1091-1110`, `koan/app/missions.py:1139-1180`

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

**Fix:**
In `_move_in_progress_to_done()`, use ❌ with a `[flushed]` cause tag instead
of ✅:

```python
entry = f"- {display} ❌ ({timestamp}) [flushed]"
```

Rename the function to `_flush_in_progress_to_failed()` for clarity.

---

#### B6 — Two independent retry systems accumulate silently; combined limit undocumented

**Files:** `koan/app/recover.py:36`, `koan/app/stagnation_monitor.py:55`

**Problem:**
There are two separate retry counters:
- `[r:N]` embedded in missions.md: tracks crash-recovery attempts, max 3
- `.stagnation-retries.json`: tracks stagnation-requeue attempts, max configurable

These counters are independent and have no cross-awareness. A mission can:
1. Stagnate 2 times (requeued via stagnation counter, never reaching max due to B1)
2. Then crash-recover 3 times (via `[r:N]` counter)
3. Then stagnate again (new run, new stagnation key) — infinite loop

Neither counter is reset when the other fires. There is no global "give up on
this mission" threshold.

**Fix (short term):** Fix B1 first — this automatically makes the stagnation
cap work as intended and bounds the combined retry total.

**Fix (long term):** Consider a unified "total_attempts" counter that resets
only on genuine success, giving operators a single knob.

---

#### B7 — `has_pending_journal` flag in `recover.py` applies to all in-progress missions

**Files:** `koan/app/recover.py:203-208`, `koan/app/recover.py:270-273`

**Problem:**
`has_pending_journal` is computed ONCE before the loop over all In Progress
missions. If there are N in-progress missions (unusual but possible under bug
conditions), all of them are classified as "partial" if any `pending.md`
exists — even if only one mission created it.

"partial" missions get checkpoint context injected, which currently does nothing
useful (B3), but the classification itself affects the audit log.

**Fix:**
Move the `pending.md` check inside the per-mission loop. For the first
`classified as "partial"` mission, mark `journal_consumed = True` and set
`has_pending_journal = False` for subsequent iterations so only the first
mission claims the checkpoint.

---

#### B8 — GitHub `processed_comments` set is in-memory; bridge restart causes duplicate comment dispatch

**Files:** `koan/app/github_notifications.py`, `koan/app/awake.py`

**Problem:**
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

**Fix:**
Persist the processed comment IDs to disk (`instance/.github-processed-comments.json`)
with a TTL of 24h to bound file growth. On startup, load from disk to restore
the in-memory set.

---

#### B9 — `start_mission()` silently succeeds when mission not found in Pending

**Files:** `koan/app/missions.py:1148-1151`

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

**Fix:**
In `run.py`'s `_start_mission_in_file()`, verify the transition succeeded by
reading the resulting content and checking that In Progress now contains the
mission. If not, raise an exception or fall back to marking it as failed.

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

#### B11 — No telemetry when `complete_mission()` / `fail_mission()` silently finds nothing

**Files:** `koan/app/missions.py:1054-1058`

**Problem:**
`_move_pending_to_section()` returns the content unchanged if the mission is
not found in Pending or In Progress. The caller never knows whether the
transition happened. Silent no-ops here mask data corruption or race
conditions.

**Fix:**
Add a boolean return value (`True` if transition occurred, `False` if not found)
and have `run.py`'s `_update_mission_in_file()` log a WARNING when `False`.

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

#### B14 — `pending.md` journal dir created with wrong path when `instance_dir` ends with `/`

**Files:** `koan/app/loop_manager.py:209`

**Problem (low severity):** `Path(instance_dir) / "journal" / ...` is safe, but
if the per-day journal directory creation fails (disk full, permission), the
exception propagates and aborts the pending.md write, leaving no checkpoint
context for Claude. There is no fallback to write `pending.md` without the
date-stamped journal directory.

**Fix:** Wrap `journal_dir.mkdir()` in a try/except and continue even if the
daily dir can't be created (write `pending.md` to `instance/journal/` root as
fallback).

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
