# Task Manager (Python) — Algorithm Deconstruction

**Exercise:** Algorithm Deconstruction Challenge

**Algorithm chosen:** Task list merging — `merge_task_lists()` / `resolve_task_conflict()` (`task_list_merge.py`)

---

## Source Code Analyzed

```python
import copy

def merge_task_lists(local_tasks, remote_tasks):
    merged_tasks = {}
    to_create_remote = {}
    to_update_remote = {}
    to_create_local = {}
    to_update_local = {}

    all_task_ids = set(local_tasks.keys()) | set(remote_tasks.keys())

    for task_id in all_task_ids:
        local_task = local_tasks.get(task_id)
        remote_task = remote_tasks.get(task_id)

        if local_task and not remote_task:
            merged_tasks[task_id] = local_task
            to_create_remote[task_id] = local_task
        elif not local_task and remote_task:
            merged_tasks[task_id] = remote_task
            to_create_local[task_id] = remote_task
        else:
            merged_task, should_update_local, should_update_remote = resolve_task_conflict(
                local_task, remote_task
            )
            merged_tasks[task_id] = merged_task
            if should_update_local:
                to_update_local[task_id] = merged_task
            if should_update_remote:
                to_update_remote[task_id] = merged_task

    return (merged_tasks, to_create_remote, to_update_remote, to_create_local, to_update_local)


def resolve_task_conflict(local_task, remote_task):
    merged_task = copy.deepcopy(local_task)
    should_update_local = False
    should_update_remote = False

    if remote_task.updated_at > local_task.updated_at:
        merged_task.title = remote_task.title
        merged_task.description = remote_task.description
        merged_task.priority = remote_task.priority
        merged_task.due_date = remote_task.due_date
        should_update_local = True
    else:
        should_update_remote = True

    if remote_task.status == TaskStatus.DONE and local_task.status != TaskStatus.DONE:
        merged_task.status = TaskStatus.DONE
        merged_task.completed_at = remote_task.completed_at
        should_update_local = True
    elif local_task.status == TaskStatus.DONE and remote_task.status != TaskStatus.DONE:
        should_update_remote = True
    elif remote_task.status != local_task.status:
        if remote_task.updated_at > local_task.updated_at:
            merged_task.status = remote_task.status
            should_update_local = True
        else:
            should_update_remote = True

    all_tags = set(local_task.tags) | set(remote_task.tags)
    merged_task.tags = list(all_tags)

    if set(merged_task.tags) != set(local_task.tags):
        should_update_local = True
    if set(merged_task.tags) != set(remote_task.tags):
        should_update_remote = True

    merged_task.updated_at = max(local_task.updated_at, remote_task.updated_at)

    return merged_task, should_update_local, should_update_remote
```

---

## Prompt 1 — Deciphering the Algorithm

### Initial hypothesis
Reconciles two independent copies of the same task list (e.g. local device and remote server) into one merged list, deciding which side needs updating.

### Algorithm broken into key sections

- **Section A — Set reconciliation**: builds the union of task IDs from both sources so no task is ever silently dropped.
- **Section B — One-sided cases**: a task existing on only one side isn't a real conflict — it's simply queued to be created on the other side.
- **Section C — True conflict resolution** (`resolve_task_conflict`), split into three independent sub-decisions on the same `merged_task`:
  - **C1 — General field merge** (title/description/priority/due_date): most-recent `updated_at` wins.
  - **C2 — Status merge**: a *separate* rule — completed status gets precedence regardless of timestamp.
  - **C3 — Tag merge**: neither side wins; tags always union.

### Concrete walkthrough

`local`: `status=IN_PROGRESS, updated_at=Mon 10:00, tags=["work"]`
`remote`: `status=DONE, updated_at=Mon 09:00, tags=["urgent"]` (remote is older by timestamp, but done)

- C1: `09:00 > 10:00`? No → `should_update_remote = True`; general fields stay local's.
- C2: `remote DONE and local not DONE`? Yes → `merged.status = DONE`, `completed_at` from remote, **`should_update_local = True` too**.
- C3: tags union to `["work","urgent"]` — differs from both → **both** flags become `True`.
- **Result:** local's general fields + remote's DONE status + unioned tags, and **both sides need updating simultaneously**.

### Core technique/pattern
A **field-level merge without a common ancestor** (unlike a true three-way git merge). Decomposes one merge decision into several *independent* per-concern merges (general fields, status, tags) rather than picking one whole "winning" task.

### Non-obvious tricks
- `copy.deepcopy(local_task)` as the starting point means the code only needs to describe *changes*, not rebuild every field.
- `should_update_local`/`should_update_remote` are deliberately **not mutually exclusive** — a merge can require pushing changes both ways at once.
- **Bug spotted, not a trick:** `TaskStatus` is referenced but never imported in `task_list_merge.py` — confirmed later to be a real bug.

---

## Prompt 2 — Untangling Poor Naming/Documentation

### Better names suggested

| Current name | Suggested rename | Why |
|---|---|---|
| `resolve_task_conflict` | `merge_task_versions` | "Conflict" implies manual resolution; this always produces a deterministic result |
| `merged_task` | `result_task` / `reconciled_task` | Avoids confusing the object with the act of merging |
| `should_update_local` / `should_update_remote` | `local_needs_sync` / `remote_needs_sync` | These are hard requirements the caller must act on, not soft suggestions |
| (unnamed phases) | `_merge_general_fields()`, `_merge_status()`, `_merge_tags()` | Makes the "three independent rules" structure explicit instead of implicit |

### Patterns identified
- **Copy-then-patch pattern** — cheaper to read than building a new object field-by-field.
- **Per-field/per-concern conflict resolution** — closer to how CRDTs merge structured data field-by-field (though not a formal CRDT).
- **Dual-flag signaling** instead of a single winner enum — deliberately allows "both sides need syncing."

### Draft documentation written for the function

```
Merge two versions of the same task using three independent rules,
applied in sequence to the same result object:

  1. General fields (title, description, priority, due_date):
     most-recently-updated version wins outright.
  2. Status: completion is sticky — a DONE status always overrides
     a non-DONE one, regardless of which was updated more recently.
     Only when neither side is DONE does "most recent wins" apply.
  3. Tags: never resolved by "winning" — always the union of both
     sides, so neither side can lose a tag the other added.

IMPORTANT: because these rules are independent, BOTH should_update_local
and should_update_remote can be True simultaneously. Callers must handle
both flags, not assume they're mutually exclusive.
```

### Validation questions asked
1. Does `merge_task_lists()` actually act on both flags when both are `True`, or silently only one?
2. Does any test in `tests/test_task_list_merge.py` exercise the case where status and general fields disagree on which side is newer?
3. Does the rest of the codebase have any concept of "sync"/"remote" at all, or is this entirely forward-looking with nothing currently calling it?

### Safe experiments proposed
- Call `resolve_task_conflict()` directly with hand-built fake tasks (pure function, no side effects) and print all three return values.
- Temporarily add `print()`s inside each rule block to confirm which branch executes.
- Deliberately construct the exact-tie case and observe the result.

---

## Prompt 3 — Untangling the Control Flow

### Control flow diagram

```
resolve_task_conflict(local, remote)
|
+- merged = deepcopy(local)
+- should_update_local = False
+- should_update_remote = False
|
+- BLOCK 1 - General fields (always runs, exactly one branch)
|   +- IF remote.updated_at > local.updated_at:
|   |     copy remote's title/description/priority/due_date
|   |     should_update_local = True
|   +- ELSE (remote older OR exactly equal):
|         should_update_remote = True
|
+- BLOCK 2 - Status (always runs, 4 mutually exclusive outcomes)
|   +- IF remote DONE and local not DONE: status=DONE, should_update_local=True
|   +- ELIF local DONE and remote not DONE: should_update_remote=True
|   +- ELIF remote.status != local.status (neither DONE):
|   |     +- IF remote newer: status=remote's, should_update_local=True
|   |     +- ELSE: should_update_remote=True
|   +- (implicit ELSE, real but unwritten): statuses equal -> nothing happens
|
+- BLOCK 3 - Tags (always runs)
|   +- tags = union(local, remote)                [always]
|   +- IF tags != local.tags: should_update_local = True
|   +- IF tags != remote.tags: should_update_remote = True
|
+- merged.updated_at = max(local, remote)   [always runs]
|
+- return merged, should_update_local, should_update_remote
```

### Refactored version (same behavior, clearer structure)

```python
def resolve_task_conflict(local_task, remote_task):
    merged_task = copy.deepcopy(local_task)
    local_needs_sync = False
    remote_needs_sync = False

    local_needs_sync |= _merge_general_fields(merged_task, local_task, remote_task)
    if not local_needs_sync:
        remote_needs_sync = True

    status_local, status_remote = _merge_status(merged_task, local_task, remote_task)
    local_needs_sync |= status_local
    remote_needs_sync |= status_remote

    tags_local, tags_remote = _merge_tags(merged_task, local_task, remote_task)
    local_needs_sync |= tags_local
    remote_needs_sync |= tags_remote

    merged_task.updated_at = max(local_task.updated_at, remote_task.updated_at)
    return merged_task, local_needs_sync, remote_needs_sync
```

### Key decision points and interdependencies
- Block 1 and Block 2 both write to the *same two flags* using `=` rather than `|=` — but since both blocks only ever set `True`, never `False`, the net effect is equivalent to OR-ing. It works correctly, but doesn't read that way at first glance.
- Block 2's four cases are genuinely mutually exclusive; the silent "both statuses equal" fallthrough is real and correct, just undocumented.

### Bugs and edge cases confirmed
- **Real, verified bug:** `TaskStatus.DONE` is referenced but `TaskStatus` is never imported in this file — running this function standalone raises `NameError`.
- **Tie handling is a real, silent default, not a crash:** strict `>` means an exact timestamp tie always falls to `else` (remote flagged for sync, local's fields win).
- **Flags only ever go `True`, never reset:** correct and safe given how each block independently proves a reason to sync, but would be a bug if a future block needed to "cancel" an earlier sync requirement.

### Prediction exercise — worked solutions

**Scenario A — exact tie, identical tasks (`TODO`/`TODO`, same timestamp, same tags):**
- Block 1: tie → `else` fires → `remote_needs_sync = True` (even though nothing is actually different).
- Block 2: no branch fires (equal, non-DONE statuses).
- Block 3: tags identical, no flags touched.
- **Result:** status stays `TODO`; `local_needs_sync = False`, `remote_needs_sync = True`. A redundant sync signal on a tie, even with genuinely identical data.

**Scenario B — remote is DONE but older (`REVIEW`@14:00 vs `DONE`@12:00, different tags):**
- Block 1: remote older → `remote_needs_sync = True`; local's general fields kept.
- Block 2: remote DONE, local not → status becomes `DONE`, `local_needs_sync = True`.
- Block 3: tags union differs from local's → `local_needs_sync` stays `True`; matches remote's → no extra flag.
- **Result:** status = `DONE`; **both** `local_needs_sync` and `remote_needs_sync` = `True` — completion-precedence overrides the timestamp rule entirely.

**Scenario C — both `DONE`, remote newer, different tags:**
- Block 1: remote newer → general fields copied from remote → `local_needs_sync = True`.
- Block 2: both already `DONE`, statuses equal → no branch fires, status untouched.
- Block 3: tags union differs from local's (`[]`) → `local_needs_sync` stays `True`; matches remote's → no extra flag.
- **Result:** status = `DONE`; `local_needs_sync = True`, `remote_needs_sync = False`. Once both sides agree on completion, only the timestamp-based general-field rule matters.

---

## Insights and Learning Points

1. **A function can look like it has one "winner" but actually resolve conflicts per-concern.** `resolve_task_conflict()` never picks one whole task as correct — general fields, status, and tags are each decided independently, which is exactly why both sync flags can be `True` at once. Missing this distinction leads to assuming the flags are mutually exclusive, which they aren't.
2. **A field that reads as "recency-based" can be silently overridden by a stickier business rule.** Completion status ignores the "most recent wins" pattern entirely — a deliberate one-way door, not an inconsistency, but invisible unless you trace status separately from the general-field logic.
3. **Reading real code catches bugs that a docstring alone won't.** The missing `TaskStatus` import would never have surfaced from the function's documentation or naming — only tracing actual execution (or trying to run it) reveals it.
4. **Ties are a real design decision, not an oversight, even when undocumented.** Strict `>` versus `>=` in a comparison is a one-character detail with a concrete behavioral consequence (which side wins on an exact timestamp match) — worth deliberately checking rather than assuming symmetry.
5. **Assignment (`=`) can behave like accumulation (`|=`) by construction, not by accident.** The two sync flags are only ever set `True`, never reset — so plain assignment across multiple blocks has the same net effect as OR-assignment, but a reader unfamiliar with that invariant could misjudge the code as buggy when it isn't.
