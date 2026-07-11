# Task Manager (Python) — Feature Location Document


**Exercise:** Knowing Where to Start
**Part 2:** Finding Feature Implementation — "Task Export to CSV"

---

## 1. Initial Search

**Search terms used:** `export`, `csv`, `open(`, `.write(`, `.read(`, `json`

**Directories checked:** project root, `tests/`

**Findings:**
- No existing `export` or `csv` functionality anywhere in the codebase — this is a genuinely new feature, not a refactor of something existing.
- `storage.py` is the **only** file in the entire project that performs file I/O (`open()`, `json.dump`/`json.load`). Every other module works purely with in-memory `Task` objects.
- Three standalone modules already exist that follow a "self-contained, takes `Task` objects, does one job" pattern: `task_parser.py`, `task_priority.py`, `task_list_merge.py` — none are wired into `cli.py`, which is itself a useful cautionary pattern to avoid repeating.

---

## 2. Hypothesis (Before AI Analysis)

- **Where I thought export functionality belongs:** A new standalone module (e.g. `task_export.py`), following the same pattern as `task_parser.py`/`task_priority.py`/`task_list_merge.py`, rather than inside `storage.py` (JSON-specific) or `task_manager.py` (business logic).
- **Existing components likely needing changes:**
  - `cli.py` — new `export` subcommand
  - `task_manager.py` — possibly a new method to keep `cli.py` thin
  - `models.py` — assumed no changes needed, since `Task` already exposes all fields as plain attributes

---

## 3. AI-Assisted Analysis — Feature Location Prompt Findings

### Files ranked by relevance

| File | Why it matters for this feature |
|---|---|
| `cli.py` | Every command follows an identical `argparse` subparser pattern. The new `export` command should mirror the `stats` command (no filters) or `list` command (has filters). |
| `storage.py` | Not reused directly (JSON-specific), but its `TaskEncoder.default()` shows the *shape* a `Task` needs to be flattened into before writing — CSV export needs the same flattening logic. |
| `models.py` | Defines the exact field list a CSV export should mirror: `id, title, description, priority, status, created_at, updated_at, due_date, completed_at, tags`. |
| `task_manager.py` | Where the new `export_tasks()` method should live. Can reuse `self.storage.get_all_tasks()` — no new storage method needed to fetch data. |

### More effective search terms identified

- `subparsers.add_parser` — finds every existing CLI command in one search, showing the pattern to copy
- `get_all_tasks` — finds every place task data is already pulled in bulk (the export's data source)
- `strftime` / `isoformat` — finds existing date-formatting conventions to stay consistent with

### How the feature splits across the codebase

1. **Data retrieval** — reuse `TaskStorage.get_all_tasks()` as-is, zero changes needed
2. **Format transformation** (Task objects → CSV rows) — new logic; doesn't belong in `storage.py` (JSON-only) or `models.py` (no format-specific code belongs in the domain layer)
3. **File writing** — Python's stdlib `csv` module, no new dependency required
4. **CLI wiring** — new subcommand in `cli.py`

### Self-check questions used while exploring

- Does this module already know how to serialize a `Task`, or does it just consume already-serialized data?
- If I follow the same pattern as the 3 orphaned modules, would this new module get wired into `cli.py`, or become a 4th orphan? (Decision: wire it in this time.)
- What does `Task.__dict__` actually contain, and do enums/datetimes need special handling? (Yes — confirmed by `storage.py`'s encoder.)

---

## 4. Implementation Map

### Where I would implement this feature

**New file: `task_export.py`** (top-level, alongside the other standalone modules)
- `export_tasks_to_csv(tasks, filepath)` — takes a list of `Task` objects and a destination path, writes a CSV using the stdlib `csv` module
- Flattens each `Task` the same way `storage.py`'s `TaskEncoder` does: priority/status enums → `.value`/`.name`, datetimes → `.isoformat()` or `.strftime(...)`, tags list → joined string (e.g. `;`-separated, since CSV already uses commas)

### Related components affected

| Component | Change required |
|---|---|
| `task_manager.py` | Add `export_tasks(filepath, status_filter=None, priority_filter=None)` — reuses existing `list_tasks()` filtering logic, then calls `task_export.export_tasks_to_csv()` |
| `cli.py` | Add an `export` subparser (path argument, optional `--status`/`--priority` filters mirroring `list`), wire to `task_manager.export_tasks()` |
| `models.py` | No changes — `Task` already exposes everything needed as plain attributes |
| `storage.py` | No changes — CSV export is a read-only operation on already-loaded tasks, doesn't touch persistence |
| `tests/` | New `tests/test_task_export.py` following the existing one-test-file-per-module convention |

### Implementation plan

1. Write `task_export.py` with `export_tasks_to_csv(tasks, filepath)`, using `csv.DictWriter` with a fixed fieldnames list matching `Task`'s attributes.
2. Add `TaskManager.export_tasks()` in `task_manager.py`, reusing `list_tasks()` for filtering so export respects the same `--status`/`--priority` options as `list`.
3. Add the `export` subcommand to `cli.py`: `python cli.py export tasks.csv [--status todo] [--priority 3]`.
4. Write `tests/test_task_export.py`: cover empty task list, tasks with/without due dates or tags, and enum-to-string conversion correctness.
5. Manually verify: run `export`, open the resulting `.csv` in a spreadsheet app, confirm columns and values match what `show`/`list` display for the same tasks.

---

## 5. Challenge Self-Check

**Challenge:** From memory, list all fields on `Task` (there are 10), then verify against `models.py`.

**Fields:** `id`, `title`, `description`, `priority`, `status`, `created_at`, `updated_at`, `due_date`, `completed_at`, `tags`

*(Note: `completed_at` is easy to miss if you only skim `__init__` — it's set in `mark_as_done()`, not the constructor's visible defaults, which is a good reminder that a class's full shape can span multiple methods.)*
