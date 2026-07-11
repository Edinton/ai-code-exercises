# Task Manager (Python) — Domain Model Document

**Exercise:** Knowing Where to Start
**Part 3:** Understanding Domain Model

---

## 1. Extracted Domain Model

**Core entity classes:** `Task`, `TaskStatus` (enum), `TaskPriority` (enum) — all defined in `models.py`.

**Business logic found — spread across three files, not just `models.py`:**

| File | Business logic |
|---|---|
| `models.py` | Entity-level rules: `mark_as_done()`, `is_overdue()`, `update()` |
| `task_manager.py` | Orchestration rules: status change to `DONE` routes specifically through `mark_as_done()`, not a generic field update; `get_statistics()` computes derived facts like "completed in last 7 days" |
| `task_priority.py` | A scoring/ranking rule (`calculate_task_score`) — a *computed* importance score blending priority + due-date urgency + status + tags + recency |
| `task_list_merge.py` | A conflict-resolution rule for reconciling two versions of the same task (local vs remote), including the rule that **"done" status always wins** in a conflict |

**App-specific terminology spotted:**
- **"Overdue"** — due date has passed AND status is not done
- **"Importance score"** — computed on the fly, never stored
- **"Conflict resolution"** with a completed-status override rule
- **"blocker" / "critical" / "urgent" tags** — special strings that boost the importance score, distinct from (but easily confused with) the `TaskPriority.URGENT` enum value

---

## 2. Initial Understanding (Before AI Analysis)

### Entity relationship sketch

```
Task
 ├── has one   TaskStatus   (todo → in_progress → review → done)
 ├── has one   TaskPriority (low / medium / high / urgent)
 └── has many  tags (plain strings, no separate Tag entity)
```

### What I thought each entity represented

- **Task** — the single aggregate root; everything else hangs off it, no separate entities for tags/comments/subtasks
- **TaskStatus** — a workflow/lifecycle state, implying a linear pipeline (todo → in_progress → review → done), though nothing in the code visibly enforces that order
- **TaskPriority** — a static importance label set by the user, separate from the computed importance score in `task_priority.py`

### Questions/confusion noted before asking AI

1. Is `TaskStatus` a strict state machine, or can you jump from `todo` straight to `done`?
2. Why does "done" status win in merge conflicts even if the other version was edited more recently — intentional rule or oversight?
3. Are the "critical/blocker/urgent" tags a controlled vocabulary, or can users type anything and only these three get special treatment?

---

## 3. AI-Assisted Analysis — Domain Understanding Prompt Findings

### Corrected understanding

`TaskStatus` **looks like** it implies a linear pipeline, but nothing enforces transition order — `update_task_status()` accepts any `TaskStatus` value and applies it directly. The domain model as currently coded is "a label from a fixed set of four," not a true state machine with guarded transitions. Any transition-order business rule would be the *first* real workflow enforcement added, not an extension of existing enforcement.

### Core domain concepts identified

- **Task** — the aggregate root; a unit of work with a lifecycle
- **Status** — *where in the workflow* a task currently sits (process concept)
- **Priority** — *how important the user says it is* (static, user-assigned label)
- **Importance score** — a derived, ephemeral concept, not a stored field; recalculated fresh every time from priority + urgency + status + tags + recency
- **Sync conflict resolution** — a separate concern layered on top of the same `Task` entity: how two divergent copies of the same task reconcile

### Relationships in business terms

Priority and status are not peers. Priority is "how the user judges the work"; status is "what state the team's process says the work is in." The importance score fuses both into one actionable number, with a real-world twist: overdue and recently-touched tasks get boosted, meaning recency and lateness can outweigh the user's static priority label (a HIGH priority task untouched for weeks can be outranked by a MEDIUM one due tomorrow).

### Domain terminology/patterns previously missed

- **Completed-status precedence** — you cannot silently un-complete a task via sync. If either copy says DONE, the merged result is DONE, regardless of timestamps — a deliberate one-way door, common in sync systems.
- **"Overdue" is a computed predicate, not a stored status** — `is_overdue()` is a real-time check, so a task's overdue-ness changes automatically as time passes with no explicit event marking it overdue.
- **Tags are dual-purpose** — most tags are just user-facing labels, but `"blocker"`, `"critical"`, and `"urgent"` are silently treated as a controlled vocabulary by the scoring logic — an implicit, undocumented rule.

### Connection to user-facing features

| Domain concept | User-facing feature |
|---|---|
| `Task` + `TaskStatus`/`TaskPriority` | `create` / `update-status` / `update-priority` CLI commands |
| `calculate_task_score` | Not currently exposed in `cli.py` — conceptually powers a future "what should I work on right now" view |
| `merge_task_lists` | Not currently exposed in `cli.py` — conceptually the engine behind a future "sync across devices" feature |

---

## 4. Testing My Knowledge

**Q1: HIGH priority untouched for 3 weeks, no due date vs. LOW priority due tomorrow — which scores higher?**
**Answer:** The LOW-priority task due tomorrow likely scores higher. HIGH priority alone contributes a fixed base (40 points: `4 × 10`), but with no due date there's no urgency bonus, and after 3 weeks untouched there's no recency bonus either. The LOW-priority task (10 points base) gets a same-week due-date bonus (+10) and, if updated recently, a recency bonus (+5) — closing much of the gap, and easily overtaking it if the HIGH task has also drifted into REVIEW (−15) or picked up no boosting tags.

**Q2: local = REVIEW (2 days old) vs remote = IN_PROGRESS (5 minutes old) — which status wins?**
**Answer:** IN_PROGRESS (remote) wins. Since neither status is DONE, this falls under the "different non-completed status — most recent wins" rule, and remote's `updated_at` is more recent.

**Q3: Why does `is_overdue()` check `status != DONE` instead of just `due_date < now()`?**
**Answer:** Because a completed task with a past due date shouldn't still be flagged as "overdue" — that label is meant to prompt action, and a done task needs none. Without this check, every task ever finished even one day late would remain permanently marked overdue.

**Q4: Adding a rule that tasks can't jump TODO → DONE directly — which method to modify, and why not `models.py`?**
**Answer:** `TaskManager.update_task_status()` in `task_manager.py`, since that's the orchestration point where a status *change* (as opposed to raw data) is applied — it already special-cases the DONE transition. `models.py`'s `Task` class holds data and simple self-contained behaviours, but has no awareness of "previous status," so a transition-guard rule doesn't fit its scope without passing in extra context that breaks the class's current simplicity.

**Q5: Is `TaskPriority.URGENT` the same concept as the `"urgent"` tag?**
**Answer:** No — they're unrelated in the code, despite the shared word. `TaskPriority.URGENT` is a fixed enum value set at task creation. The `"urgent"` tag is just a string in a list, incidentally checked by `calculate_task_score`'s tag-boost logic. A task could have `TaskPriority.LOW` and the tag `"urgent"` and get both the low base score *and* the tag boost — or have `TaskPriority.URGENT` with no `"urgent"` tag and miss the boost entirely. Risk: assuming these sync would lead to broken expectations about why two similarly-labeled tasks score differently.

---

## 5. Revised Entity Diagram

```
                    ┌─────────────────┐
                    │      Task        │  (aggregate root)
                    │───────────────────│
                    │ id, title,        │
                    │ description,      │
                    │ created_at,       │
                    │ updated_at,       │
                    │ due_date,         │
                    │ completed_at,     │
                    │ tags: [string]    │
                    └─────────┬─────────┘
                    has one   │  has one
              ┌───────────────┴───────────────┐
              ▼                                 ▼
     ┌─────────────────┐              ┌──────────────────┐
     │   TaskStatus      │              │   TaskPriority     │
     │  (process state,   │              │  (static, user-set  │
     │  NOT enforced as   │              │  importance label)  │
     │  a state machine)  │              └──────────────────┘
     │ todo → in_progress │
     │  → review → done   │
     └─────────────────┘

  Derived / computed concepts (not stored on Task):
  ┌────────────────────────────┐   ┌──────────────────────────────┐
  │ is_overdue()                │   │ calculate_task_score()          │
  │ = due_date < now()          │   │ = f(priority, due date urgency,  │
  │   AND status != DONE        │   │   status, boosting tags,         │
  └────────────────────────────┘   │   recency of update)             │
                                     └──────────────────────────────┘

  Sync concern (operates on two Task copies, not a stored relationship):
  ┌───────────────────────────────────────────────────┐
  │ resolve_task_conflict(local Task, remote Task)        │
  │  → DONE always wins · most-recent-update wins otherwise │
  │  → tags always merge as a union                          │
  └───────────────────────────────────────────────────┘
```

---

## 6. Glossary of Domain Terms

| Term | Meaning in this codebase |
|---|---|
| **Task** | The core unit of work; the aggregate root entity everything else attaches to |
| **Status** | A task's current workflow stage (`todo`, `in_progress`, `review`, `done`); currently unguarded — any transition is allowed |
| **Priority** | A static, user-assigned importance label (`low`, `medium`, `high`, `urgent`) set at creation and editable, but never automatically changed by the system |
| **Importance score** | A computed (not stored) numeric ranking from `calculate_task_score`, blending priority, due-date urgency, status, tags, and recency — used for sorting, not persisted |
| **Overdue** | A computed predicate: due date has passed AND status is not `done`. Recalculates in real time, no explicit "became overdue" event |
| **Boosting tags** | The specific strings `"blocker"`, `"critical"`, `"urgent"` — an undocumented controlled vocabulary that increases a task's importance score when present in `tags` |
| **Conflict resolution** | The process of reconciling two divergent copies of the same task (e.g. local vs remote) into one merged version |
| **Completed-status precedence** | The rule that `DONE` status always wins in a merge conflict, regardless of which copy was more recently updated — a one-way door preventing accidental "un-completion" |
| **Tags** | Free-form string labels on a task; mostly just user-facing metadata, except for the three boosting tags above |
