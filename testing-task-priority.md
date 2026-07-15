# Using AI to Help with Testing — Task Prioritization Algorithm

**Exercise:** Using AI to help with testing

**Function under test:** `calculate_task_score`, `sort_tasks_by_importance`, `get_top_priority_tasks` (`task_priority.py`, Python implementation)

All tests in this document were actually written and executed (via `unittest`) against the real function, not just theorized — final result: **18/18 tests passing.**

---

## Function Under Test (Verified Exact Source)

```python
from datetime import datetime
from models import TaskStatus, TaskPriority

def calculate_task_score(task):
    priority_weights = {
        TaskPriority.LOW: 1, TaskPriority.MEDIUM: 2,
        TaskPriority.HIGH: 4, TaskPriority.URGENT: 6
    }
    score = priority_weights.get(task.priority, 0) * 10

    if task.due_date:
        days_until_due = (task.due_date - datetime.now()).days
        if days_until_due < 0:
            score += 35
        elif days_until_due == 0:
            score += 20
        elif days_until_due <= 2:
            score += 15
        elif days_until_due <= 7:
            score += 10

    if task.status == TaskStatus.DONE:
        score -= 50
    elif task.status == TaskStatus.REVIEW:
        score -= 15

    if any(tag in ["blocker", "critical", "urgent"] for tag in task.tags):
        score += 8

    days_since_update = (datetime.now() - task.updated_at).days
    if days_since_update < 1:
        score += 5

    return score

def sort_tasks_by_importance(tasks):
    task_scores = [(calculate_task_score(task), task) for task in tasks]
    return [task for _, task in sorted(task_scores, key=lambda x: x[0], reverse=True)]

def get_top_priority_tasks(tasks, limit=5):
    return sort_tasks_by_importance(tasks)[:limit]
```

---

## Part 1.1 — Behavior Analysis

**AI's question:** What do you think this function does, and what behaviors should it have?

**Answer:** It computes a numeric importance score from priority (weighted), due-date proximity, status, tags, and recency of update — higher score means more urgent/important.

**AI's follow-up on missed behaviors:** Did you consider that `DONE` status *subtracts* 50 points, and that the recency boost only applies within the first 24 hours (`< 1` day), not several days?

**Correction acknowledged:** Missed the `DONE` penalty initially — a real behavior worth explicit test coverage, since a `DONE` + `URGENT` task could still land at a low or even negative score.

**AI's edge-case prompt:** What about a task with `due_date=None`? What happens right at each numeric boundary (2 days, 7 days, 24 hours)?

### 7 Test Cases Identified (all implemented and passing)

1. Base priority score with no other factors active
2. No due date → no due-date bonus applied
3. `DONE` status applies its penalty correctly
4. Overdue task gets the maximum due-date bonus
5. A boosting tag (`"urgent"`, `"critical"`, `"blocker"`) adds its fixed bonus
6. Recently updated task (within 24h) gets the recency bonus
7. **Boundary edge case:** updated *exactly* 24 hours ago does **not** get the recency bonus (`.days` truncates, so `24h → days=1`, failing `< 1`)

```python
class TestCalculateTaskScoreBehaviors(unittest.TestCase):
    def test_base_priority_score_with_no_other_factors(self):
        task = make_task(priority=TaskPriority.MEDIUM, updated_hours_ago=100)
        self.assertEqual(calculate_task_score(task), 20)  # 2 * 10

    def test_no_due_date_gets_no_due_date_bonus(self):
        task = make_task(priority=TaskPriority.LOW, due_in_days=None, updated_hours_ago=100)
        self.assertEqual(calculate_task_score(task), 10)

    def test_done_status_applies_penalty(self):
        task = make_task(priority=TaskPriority.URGENT, status=TaskStatus.DONE, updated_hours_ago=100)
        self.assertEqual(calculate_task_score(task), 10)  # 60 - 50

    def test_overdue_task_gets_maximum_due_date_bonus(self):
        task = make_task(priority=TaskPriority.LOW, due_in_days=-5, updated_hours_ago=100)
        self.assertEqual(calculate_task_score(task), 45)  # 10 + 35

    def test_boosting_tag_adds_fixed_bonus(self):
        task = make_task(priority=TaskPriority.LOW, tags=["urgent"], updated_hours_ago=100)
        self.assertEqual(calculate_task_score(task), 18)  # 10 + 8

    def test_recently_updated_task_within_a_day_gets_recency_bonus(self):
        task = make_task(priority=TaskPriority.LOW, updated_hours_ago=0.5)
        self.assertEqual(calculate_task_score(task), 15)  # 10 + 5

    def test_updated_exactly_24_hours_ago_does_not_get_recency_bonus(self):
        task = make_task(priority=TaskPriority.LOW, updated_hours_ago=24)
        self.assertEqual(calculate_task_score(task), 10)
```

---

## Part 1.2 — Test Plan Document

| Priority | Test | Type | Dependencies | Expected outcome |
|---|---|---|---|---|
| High | Base priority scoring (all 4 levels) | Unit | `make_task` factory | Exact score matches `priority_weights` table |
| High | Due-date brackets (overdue, today, ≤2d, ≤7d, none) | Unit | `make_task` with `due_in_days` | Exact bonus per bracket |
| High | Status effects (`DONE` penalty, `REVIEW` penalty, others neutral) | Unit | `make_task` with `status` | Exact penalty applied |
| Medium | Boosting tags present/absent | Unit | `make_task` with `tags` | +8 only if a boosting tag is present |
| Medium | Recency bonus + 24h boundary | Unit | `make_task` with `updated_hours_ago` | +5 only if `days_since_update < 1` |
| Medium | `sort_tasks_by_importance` ordering | Integration | `calculate_task_score` (used internally) | Descending score order |
| Medium | `get_top_priority_tasks` agrees with full sort | Integration | Both other functions | First N of the full sort, no divergence |
| Low | Empty task list handling | Unit/Integration | None | No error, empty result |
| Low | Multiple boundary values at once (combined factors) | Unit | `make_task` | Sum of applicable bonuses/penalties |

**Test types needed:** Primarily unit tests for `calculate_task_score` (many small, isolated behaviors), plus integration tests for `sort_tasks_by_importance`/`get_top_priority_tasks` working together, since their correctness depends entirely on `calculate_task_score`'s output being consistent.

**Test dependencies:** All tests depend on a reliable way to construct `Task` objects with fully controlled fields (priority, due date, status, tags, `updated_at`) — built as a `make_task()` factory rather than relying on `Task`'s default `datetime.now()` behavior, which would make tests non-deterministic.

---

## Part 2.1 — Improving a Single Test

**Original naive test (intentionally simple):**
```python
def test_score_is_a_number(self):
    task = make_task()
    score = calculate_task_score(task)
    self.assertTrue(score > 0)
```

**AI feedback received:**
- What is this test actually trying to verify? "Returns something greater than zero" doesn't specify *which* behavior is under test.
- This checks an implementation detail (a loose range) rather than a specific behavior (an exact expected score for known inputs).
- Missing: what if priority alone should be verified in isolation from all other factors?

**Improved test:**
```python
def test_urgent_priority_alone_scores_exactly_60(self):
    """
    Improved from a naive 'score > 0' test. Asserts an EXACT value,
    isolating priority as the only active factor (no due date, TODO
    status, no tags, updated long ago) so the assertion is unambiguous
    about which behavior is being verified.
    """
    task = make_task(priority=TaskPriority.URGENT, updated_hours_ago=100)
    self.assertEqual(calculate_task_score(task), 60)
```
**Result: PASS.**

---

## Part 2.2 — Due Date Calculation Deep Dive

**Initial pseudocode idea:** "Create a task due in a few days, check the score goes up."

**AI guidance received:** A good test for bracketed logic like this should test *each boundary explicitly* — the exact day counts where behavior changes (`0`, `2`→`3`, `7`→`8`), not just "a few days" generically, since off-by-one errors are exactly where bracket logic tends to break.

**Real discovery during implementation:** The first attempt at boundary tests (`due_in_days=3` expecting the `≤7` bracket, `due_in_days=8` expecting no bonus) **actually failed** on first run:

```
AssertionError: 25 != 20   (due_in_days=3 case)
AssertionError: 20 != 10   (due_in_days=8 case)
```

Root cause investigated directly:
```python
due = datetime.now() + timedelta(days=3)
time.sleep(0.01)
diff = due - datetime.now()
print(diff, diff.days)
# -> 2 days, 23:59:59.985486   .days = 2
```
The microsecond delay between constructing the test's `due_date` and `calculate_task_score` evaluating it nudges the true difference just under 3 full days, and `.days` floors that down to `2` — pulling the test into the wrong bracket. **This is a test-construction bug, not a bug in the function under test.**

**Fix applied:** Added a 1-hour buffer in the `make_task` test helper when constructing `due_date`, so processing delay can never cross a day boundary:
```python
task.due_date = datetime.now() + timedelta(days=due_in_days, hours=1)
```

**Final passing tests:**
```python
class TestDueDateCalculation(unittest.TestCase):
    def test_due_in_exactly_2_days_gets_near_term_bonus(self):
        task = make_task(priority=TaskPriority.LOW, due_in_days=2, updated_hours_ago=100)
        self.assertEqual(calculate_task_score(task), 25)  # 10 + 15

    def test_due_in_3_days_falls_to_week_bonus_not_near_term(self):
        task = make_task(priority=TaskPriority.LOW, due_in_days=3, updated_hours_ago=100)
        self.assertEqual(calculate_task_score(task), 20)  # 10 + 10

    def test_due_in_8_days_gets_no_due_date_bonus(self):
        task = make_task(priority=TaskPriority.LOW, due_in_days=8, updated_hours_ago=100)
        self.assertEqual(calculate_task_score(task), 10)
```
**Result: PASS (after the test helper fix).**

---

## Part 3.1 — TDD for a New Feature (Current-User Assignment Boost)

**Feature:** Tasks assigned to the current user should get a score boost of `+12`.

**AI's first question:** What should the first test be, and why?

**Answer:** The simplest possible case first: a task assigned to the current user gets exactly base score + 12, with every other factor neutral — isolating the new behavior before testing interactions.

**Test written first (fails initially, since no implementation exists yet):**
```python
def test_task_assigned_to_current_user_gets_plus_12(self):
    task = make_task(priority=TaskPriority.LOW, updated_hours_ago=100)
    task.assigned_to = "tumie"
    base_score = calculate_task_score(task)
    boosted_score = calculate_task_score_with_assignment(task, current_user="tumie")
    self.assertEqual(boosted_score, base_score + 12)
```

**Minimal implementation to make it pass:**
```python
def calculate_task_score_with_assignment(task, current_user=None):
    """
    Minimal TDD implementation: wraps the existing calculate_task_score and
    adds +12 if the task is assigned to the current user. Kept as a wrapper
    here rather than editing task_priority.py directly during the exercise;
    in a real implementation this would be merged into calculate_task_score
    itself once verified.
    """
    score = calculate_task_score(task)
    if current_user is not None and getattr(task, "assigned_to", None) == current_user:
        score += 12
    return score
```

**Next tests added (guided by "what edge case comes next?"):**
```python
def test_task_assigned_to_someone_else_gets_no_boost(self):
    task = make_task(priority=TaskPriority.LOW, updated_hours_ago=100)
    task.assigned_to = "other_user"
    base_score = calculate_task_score(task)
    self.assertEqual(calculate_task_score_with_assignment(task, current_user="tumie"), base_score)

def test_task_with_no_assignment_gets_no_boost(self):
    task = make_task(priority=TaskPriority.LOW, updated_hours_ago=100)
    base_score = calculate_task_score(task)
    self.assertEqual(calculate_task_score_with_assignment(task, current_user="tumie"), base_score)
```
**All 3 pass.** No refactor was necessary — the wrapper approach kept the change small and isolated without touching the existing, already-tested function.

---

## Part 3.2 — TDD for a "Bug Fix" (Adapted)

**Note on adaptation:** the exercise's described bug ("should use `.days` instead of dividing by milliseconds/`ChronoUnit`") is JS/Java-specific and **does not apply here** — this Python implementation already correctly uses `.days`. Rather than fabricate a bug that doesn't exist in this codebase, this section investigates a real, verified subtlety of `.days` truncation discovered during this exercise.

**Behavior to reproduce:** `.days` truncates rather than rounds, so the 24-hour recency window has a hard, abrupt cliff rather than a gradual falloff.

**Test that reproduces/demonstrates the behavior:**
```python
def test_23_hours_59_minutes_still_gets_recency_bonus(self):
    task = make_task(priority=TaskPriority.LOW, updated_hours_ago=23.98)
    self.assertEqual(calculate_task_score(task), 15)  # gets +5

def test_24_hours_1_minute_loses_recency_bonus_entirely(self):
    """
    Demonstrates the real behavior: crossing from 23h59m to 24h01m removes
    the ENTIRE +5 bonus abruptly, not gradually. Confirmed as-designed
    (not something to silently "fix"), but a boundary worth explicit
    documentation and test coverage.
    """
    task = make_task(priority=TaskPriority.LOW, updated_hours_ago=24.02)
    self.assertEqual(calculate_task_score(task), 10)  # no +5
```
**Both pass** — confirming this is existing, working-as-designed behavior rather than a defect. **Decision:** no code change made; the boundary is now explicitly documented and regression-protected by these two tests, which is itself the valuable outcome of this exercise even without a "fix."

**Regression tests to prevent future issues:** these two tests themselves serve that purpose — any future change to the `< 1` threshold or to how `days_since_update` is calculated would now be caught immediately.

---

## Part 4.1 — Integration Test for the Full Workflow

**AI's guidance questions:** What scenarios should an integration test verify? (Answer: that sorting is consistent with individual scores, and that `get_top_priority_tasks` genuinely agrees with the full sort rather than using separate logic.) What test data would exercise the whole workflow? (Answer: a mix of tasks spanning every major scoring factor — urgent+overdue, low+far-out, done+penalized, high+due-soon — so the ordering outcome isn't trivially obvious from a single factor.)

```python
class TestPriorityWorkflowIntegration(unittest.TestCase):
    def test_sort_and_top_n_agree_and_respect_score_ordering(self):
        urgent_overdue = make_task(priority=TaskPriority.URGENT, due_in_days=-1, updated_hours_ago=100)
        low_far_out = make_task(priority=TaskPriority.LOW, due_in_days=30, updated_hours_ago=100)
        medium_done = make_task(priority=TaskPriority.MEDIUM, status=TaskStatus.DONE, updated_hours_ago=100)
        high_soon = make_task(priority=TaskPriority.HIGH, due_in_days=1, updated_hours_ago=100)

        tasks = [low_far_out, medium_done, high_soon, urgent_overdue]

        sorted_tasks = sort_tasks_by_importance(tasks)
        top_2 = get_top_priority_tasks(tasks, limit=2)

        scores = [calculate_task_score(t) for t in sorted_tasks]
        self.assertEqual(scores, sorted(scores, reverse=True))
        self.assertEqual(top_2, sorted_tasks[:2])
        self.assertIs(sorted_tasks[0], urgent_overdue)
        self.assertIs(sorted_tasks[-1], medium_done)

    def test_empty_task_list_does_not_error(self):
        self.assertEqual(sort_tasks_by_importance([]), [])
        self.assertEqual(get_top_priority_tasks([], limit=5), [])
```
**Both pass.** Confirms `get_top_priority_tasks` is a genuine, consistent slice of `sort_tasks_by_importance`'s output (not independently-derived logic that could silently diverge), and that empty input is handled gracefully by both.

---

## Final Test Run Summary

```
Ran 18 tests in 0.001s

OK
```

All 18 tests across Parts 1–4 pass against the real, verified `task_priority.py` implementation.

---

## Reflection

Writing these tests surfaced two things that reading the code alone would not have: first, the exact numeric values in `calculate_task_score` (e.g., the `DONE` penalty, the `<1` recency threshold) needed re-verification against the actual source rather than trusted from memory, since a few details from earlier analysis sessions turned out to be slightly wrong. Second, and more valuable — the due-date boundary test failures weren't a bug in the function at all, but a bug in *how the test itself constructed its input data*, caused by a microsecond race between building a `timedelta`-based due date and evaluating it later. That's a genuinely transferable lesson: tests involving time-based boundaries need deliberate buffers to avoid flakiness, independent of whatever function they're testing. The TDD sections also reinforced that "minimal code to pass" doesn't have to mean editing the original function directly — a wrapper function was sufficient and kept the already-tested `calculate_task_score` untouched, which is a pattern worth reusing when adding new behavior around stable, well-tested code.
