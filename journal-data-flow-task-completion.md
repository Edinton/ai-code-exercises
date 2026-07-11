# Task Manager (Python) — Data Flow Mapping Journal


**Exercise:** Mapping Data Flow
**Part 3:** Data Flow and State Management — "Mark Task as Complete"

---

## 1. Entry Points and Components Involved

**Entry point:** `python cli.py status <task_id> done`

**Components in the chain:**

| File | Role |
|---|---|
| `cli.py` | Parses args, dispatches to `TaskManager` |
| `task_manager.py` | `update_task_status()` — decides DONE needs special handling |
| `models.py` | `Task.mark_as_done()` — mutates the object |
| `storage.py` | `TaskStorage.save()`, `TaskEncoder` — writes to `tasks.json` |

---

## 2. Data Flow Diagram

```
USER INPUT
  "python cli.py status abc-123 done"
        |
        v
+-------------------------------------------+
| cli.py                                     |
|  argparse parses: args.task_id="abc-123",  |
|                    args.status="done"       |
+------------------+--------------------------+
                   | calls
                   v
+---------------------------------------------+
| task_manager.py: update_task_status()          |
|  1. TaskStatus("done") -> TaskStatus.DONE enum  |
|  2. new_status == DONE? -> yes, branch taken     |
+------------------+---------------------------+
                   | calls
                   v
+---------------------------------------------+
| storage.py: get_task("abc-123")                 |
|  returns self.tasks["abc-123"]                    |
|  -> the LIVE object reference (no copy)             |
+------------------+---------------------------+
                   | returns Task object
                   v
+---------------------------------------------+
| models.py: task.mark_as_done()                  |
|  MUTATES the object in place:                    |
|   status -> DONE                                    |
|   completed_at -> datetime.now()                     |
|   updated_at -> same value as completed_at            |
+------------------+---------------------------+
                   | (object already mutated in the dict,
                   |  since it's the same reference)
                   v
+---------------------------------------------+
| storage.py: save()                               |
|  1. list(self.tasks.values()) -- ALL tasks,        |
|     not just this one                              |
|  2. json.dump(..., cls=TaskEncoder)                 |
+------------------+---------------------------+
                   | calls .default() per Task
                   v
+---------------------------------------------+
| TaskEncoder.default()                           |
|  Task object -> plain dict:                       |
|   .priority -> .value (int)                        |
|   .status -> .value (string)                        |
|   datetimes -> .isoformat() strings                   |
+------------------+---------------------------+
                   |
                   v
              tasks.json (disk)
                   |
                   v
+---------------------------------------------+
| cli.py: prints "Updated task status to done"     |
+---------------------------------------------+
```

---

## 3. State Changes During Task Completion

- `Task.status`: whatever it was → `TaskStatus.DONE`
- `Task.completed_at`: `None` → `datetime.now()`
- `Task.updated_at`: whatever it was → set equal to the new `completed_at` (not a separate fresh `datetime.now()` call)
- In-memory dict `TaskStorage.tasks[task_id]` — mutated **in place** (same object reference returned by `get_task()`), not replaced
- On-disk `tasks.json` — fully rewritten (every task, not just the one that changed) via `TaskEncoder`

**Where state lives overall:** exactly two places, which can briefly disagree — the in-memory `self.tasks` dict inside `TaskStorage` (source of truth *during* a single CLI invocation) and `tasks.json` on disk (source of truth *between* invocations, since every CLI call is a fresh process that reloads from disk in `TaskStorage.__init__`). There's no dedicated "state manager" class — `TaskStorage` fills that role implicitly.

---

## 4. Potential Points of Failure

- **Task not found:** `get_task()` returns `None` if the ID doesn't exist. Handled via `if task:`, but `update_task_status()` has no explicit `return False` in that branch — it relies on an implicit `None` return (falsy), which works today but is fragile to future edits.
- **Invalid status string:** `TaskStatus(new_status_value)` raises `ValueError` for invalid strings, with **no try/except** in `update_task_status()`. In practice this is prevented upstream by argparse's `choices=[...]` on the CLI argument, but that's an external safety net — calling this method directly (e.g. in a test) is unprotected.
- **Silent save failure:** `storage.save()` catches `Exception` broadly but only prints the error — it doesn't raise or signal failure back to the caller. `update_task_status()` can return `True` (in-memory mutation succeeded) even if the disk write silently failed, leaving memory and disk out of sync until the process exits, at which point the change is lost entirely.
- **No atomic write:** `save()` overwrites `tasks.json` fully on every call with no write-to-temp-then-rename step, so a crash mid-write could corrupt the entire file, not just the change being made.

---

## 5. How the Application Persists These Changes

Every mutation to a `Task` (however small) triggers a **full rewrite** of `tasks.json` — there's no incremental/partial persistence. Serialization goes through a hand-written `TaskEncoder`/`TaskDecoder` pair, since `Task` objects, enums, and datetimes aren't natively JSON-serializable. These two classes are independent mirrors of each other with no shared schema definition — a field added to `Task` requires manually updating both, or data will silently fail to round-trip correctly.

---

## 6. Design Insight — Mutate-by-Reference vs. Fetch/Edit/Resave-a-Copy

`get_task()` returns the live object stored in `TaskStorage.tasks`, not a copy — so `mark_as_done()` mutating it directly is automatically reflected in storage with no explicit "put it back" step. This is simpler (no need for `mark_as_done()` to know how to reinsert itself), but it means two code paths that fetched the "same" task and mutated it differently before either called `save()` would silently have the last `save()` win, with no conflict detection. A much bigger risk in a hypothetical multi-user version of this app than in its current single-user CLI form — but worth remembering if this codebase ever grows in that direction.

---

## 7. Debugging Approach for This Flow

Given the silent-failure pattern in `save()`, the CLI's printed success message ("Updated task status to done") cannot be fully trusted on its own — it prints even if the underlying disk write failed. The first check when something seems "lost" should be whether `tasks.json` actually reflects the last operation, not just the console output. A breakpoint or print statement immediately after `self.storage.save()` in `task_manager.py`, inspecting whether the file write actually succeeded, would catch this class of bug that the current error handling hides.
