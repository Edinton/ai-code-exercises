# Task Manager (Python) - Feature Understanding Document

**Exercise:** Understanding a Specific Feature
**Part 1:** Task Creation and Status Updates

---

## 1. Files Identified

| File | Role |
|---|---|
| `cli.py` | Parses the `create` and `status` subcommands, calls into `TaskManager` |
| `task_manager.py` | Contains `create_task()` and `update_task_status()` — the orchestration logic |
| `models.py` | Defines the `Task` class itself, including `mark_as_done()` and the generic `update()` method |
| `storage.py` | Persists changes via `add_task()`, `update_task()`, and `save()` |

---

## 2. Key Code Snippets Used

```python
# task_manager.py
def update_task_status(self, task_id, new_status_value):
    new_status = TaskStatus(new_status_value)
    if new_status == TaskStatus.DONE:
        task = self.storage.get_task(task_id)
        if task:
            task.mark_as_done()
            self.storage.save()
            return True
    else:
        return self.storage.update_task(task_id, status=new_status)
```

```python
# models.py
def update(self, **kwargs):
    for key, value in kwargs.items():
        if hasattr(self, key):
            setattr(self, key, value)
    self.updated_at = datetime.now()

def mark_as_done(self):
    self.status = TaskStatus.DONE
    self.completed_at = datetime.now()
    self.updated_at = self.completed_at
```

---

## 3. What This Component Does

Creating a task builds a `Task` object with sensible defaults (status always starts at `TODO`, `id` auto-generated) and hands it to storage to persist. Updating status is where behavior branches: most status changes are a simple field swap, but marking something `DONE` needs an extra side effect (recording *when* it was completed), so that one case gets special treatment.

---

## 4. Execution Flow

### Task creation — `python cli.py create "Title" ...`

```
cli.py (parses args, calls task_manager.create_task(...))
  -> task_manager.py: create_task()
      - converts priority int -> TaskPriority enum
      - parses due_date string -> datetime (or bails with an error message)
      - instantiates Task(title, description, priority, due_date, tags)
  -> storage.py: add_task(task)
      - stores task in an in-memory dict keyed by task.id
      - calls save() -> writes the ENTIRE dict to tasks.json
  -> returns task_id back up through task_manager -> cli.py, printed to user
```

### Status update — `python cli.py status <id> done`

```
cli.py -> task_manager.py: update_task_status(task_id, "done")
  - converts "done" string -> TaskStatus.DONE enum
  - IF DONE:
      -> storage.get_task(task_id)   (fetch the live object, not a copy)
      -> task.mark_as_done()          (mutates status + completed_at + updated_at directly)
      -> storage.save()               (persist -- note: NOT storage.update_task())
  - ELSE (any other status):
      -> storage.update_task(task_id, status=new_status)
          -> internally calls task.update(status=new_status), then save()
```

---

## 5. How the Files Interact

`cli.py` never touches a `Task` object directly — it only ever calls `TaskManager` methods. `TaskManager` never touches the filesystem directly — it only ever calls `TaskStorage` methods. `Task` (in `models.py`) never touches storage or the CLI at all — it only knows about its own fields. This is a clean one-directional layering: **CLI → Manager → Storage → Model**, with no layer reaching sideways or backwards.

---

## 6. External Dependencies

None beyond Python's standard library — `datetime`, `uuid`, `json`, `argparse`. No database, no ORM, no network calls.

---

## 7. The Confusing Part, Explained — Why Two Code Paths in `update_task_status()`

`storage.update_task(task_id, status=new_status)` is a **generic setter** — it just does `setattr(task, "status", new_status)` and updates `updated_at`. That's fine for `TODO`/`IN_PROGRESS`/`REVIEW`, because nothing else needs to happen. But `DONE` isn't just "set a field" — the business rule is "completing a task must also record *when*" (`completed_at`), which the generic `update()` method has no way to know about (it has no per-field special-case logic). So `mark_as_done()` exists as a dedicated method precisely to bundle that extra side effect, and `update_task_status()` has to special-case it because it's the one status value with attached business meaning beyond "just a label change."

**Subtle inconsistency spotted:** the `DONE` path calls `self.storage.save()` directly, while the other path calls `self.storage.update_task()` (which itself calls `save()` internally). Same end result, but `DONE` bypasses `update_task()`'s own generic `task.update()` call entirely, since `mark_as_done()` already did the equivalent work manually (including setting `updated_at` itself).

---

## 8. Mental Model

**"One door in, one door out, but a bouncer at the door checks if you're carrying anything extra."** All status changes go through `update_task_status()` (one door in). Simple changes go straight through to storage (one door out via `update_task()`). But `DONE` gets pulled aside because it needs an extra stamp (`completed_at`) that only `mark_as_done()` knows how to apply, before being let through by a slightly different exit (`save()` directly instead of via `update_task()`).

---

## 9. Small Changes to Validate Understanding

1. Make `update_task_status()` route `DONE` through `storage.update_task()` too, passing both `status` and `completed_at` as kwargs, instead of calling `mark_as_done()` and `save()` separately — verify the end result in `tasks.json` is identical to before.
2. Add a requirement that moving a task *away* from `DONE` (e.g. back to `IN_PROGRESS`) must clear `completed_at` back to `None` — implement it and verify nothing else breaks.
3. Add a requirement that `Task.update()` should reject any attempt to set `status` directly through it (forcing all status changes through `mark_as_done()` or a future dedicated method) — see what currently relies on the generic path being allowed to set `status`.

---

## 10. Journal Notes

- **Main components involved:** `cli.py` (presentation), `task_manager.py` (orchestration), `models.py` (domain/mutation), `storage.py` (persistence)
- **Execution flow:** Strictly one-directional — CLI → Manager → Storage → Model — no layer skips or reaches backward
- **Data storage/retrieval:** In-memory dict of `Task` objects in `TaskStorage`, fully rewritten to `tasks.json` via a custom JSON encoder on every single change (no incremental writes/diffs)
- **Interesting design pattern discovered:** A generic `update(**kwargs)` setter pattern on `Task` covers most field changes uniformly, but is deliberately bypassed for the one status (`DONE`) that carries extra business meaning — a good example of when a generic pattern needs a dedicated escape hatch rather than being stretched to cover every case
