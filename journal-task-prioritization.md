# Task Manager (Python) — Deepened Understanding Journal


**Exercise:** Deepen Understanding Through Guided Questions
**Part 2:** The Task Prioritization System

---

## 1. Initial Understanding vs. What Was Discovered

**Initial understanding :**
- `TaskPriority` is a static enum (`LOW=1, MEDIUM=2, HIGH=3, URGENT=4`) set at task creation
- `task_priority.py` computes a separate, richer score (`calculate_task_score`) blending priority + due date + status + tags + recency
- **Assumption made, then verified false by grep:** believed this score must drive the ordering of the `list` command's output, since that would be the obvious place a "smart" ranking would appear

**What was actually discovered:**
- `calculate_task_score()`, `sort_tasks_by_importance()`, and `get_top_priority_tasks()` are **never called anywhere** outside `task_priority.py` itself — confirmed via grep across `cli.py`, `task_manager.py`, `storage.py`, and `models.py`
- The `list` command returns tasks in whatever order the underlying dict happens to iterate in — no scoring or sorting logic is applied at all
- This confirms and extends the "orphaned module" pattern first noticed in the domain model exercise, now verified specifically for the prioritization feature

---

## 2. Key Insights Uncovered Through Guided Questioning

**Why the priority weights are `1, 2, 4, 6` instead of `1, 2, 3, 4`:**
The wider gap between HIGH and URGENT (roughly exponential spacing rather than linear) means priority alone can swing the score by up to 60 points, while the single biggest situational bonus (overdue) only adds 35. This is a deliberate design signal: priority is meant to dominate the score more than any one situational factor — an URGENT task's base score can't be caught by a LOW task even with every other bonus stacked (10 + 35 + 8 + 5 = 58, still under URGENT's 60 base alone).

**Overdue is a flat penalty, not a graded one:**
The code only checks `days_until_due < 0` — a task overdue by 1 day scores identically to one overdue by 100 days (both get a flat `+35`). This reveals the author cared about a binary "is this late at all?" signal, not a severity gradient. That's a real limitation if "critically overdue" is ever meant to matter more than "just became overdue."

**No caching, and that's consistent with the rest of the app, not an outlier:**
`calculate_task_score()` recalculates from scratch on every call, with no memoization. Checking `storage.py` confirmed this isn't unusual — `get_tasks_by_status()`, `get_tasks_by_priority()`, and `get_overdue_tasks()` are all plain linear scans with no indexing either. The whole app is built for small personal task lists, not scale, and the prioritization module fits that same philosophy rather than breaking from it.

**How it would actually get wired in:**
`TaskManager.list_tasks()`'s existing signature (`status_filter=None, priority_filter=None, show_overdue=False`) already has the exact pattern to copy: `show_overdue` is a boolean flag checked first, short-circuiting to a different storage call. Wiring in importance sorting would mean adding a `show_important=False` parameter and mirroring `--overdue`'s `action="store_true"` pattern in `cli.py` — extending the existing convention rather than inventing a new one.

**The two orphaned modules are unrelated, not one shared unfinished feature:**
`task_priority.py` and `task_list_merge.py` don't share imports with each other (both only import from `models.py`) and address completely different concerns — ranking vs. sync/conflict resolution. This suggests two independently unfinished features, or an exercise scaffold, rather than one abandoned feature branch.

**One line worth double-checking before trusting this code in production:**
```python
days_since_update = (datetime.now() - task.updated_at).days
```
`.days` truncates rather than rounds — a task updated 23 hours ago and one updated 30 minutes ago both evaluate to `.days == 0`, so the "recently updated" bonus is coarser than the variable name suggests. Worth confirming this granularity is intentional before relying on it.

---

## 3. Misconceptions Clarified

| Misconception | Reality |
|---|---|
| Assumed `calculate_task_score` powers the `list` command's ordering | It's never called anywhere outside its own file — completely disconnected from the live app |
| Assumed priority weights would be evenly spaced (1–4) matching the enum | Weights are deliberately uneven (1, 2, 4, 6) so priority dominates over situational bonuses |
| Assumed "overdue" scoring might scale with how overdue a task is | It's a flat bonus regardless of degree — 1 day and 100 days overdue score identically |
| Assumed the lack of caching might be a performance oversight specific to this module | It's consistent with the rest of the codebase's design — no part of the app is built with scale in mind |
| Assumed the two orphaned modules (`task_priority.py`, `task_list_merge.py`) might be part of one shared unfinished feature | They share no imports or concerns with each other — independently unfinished, not one abandoned branch |
