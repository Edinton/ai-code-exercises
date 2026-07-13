# Task Manager (Python) — Code Documentation Exercise

**WeThinkCode — Generative AI Program**
**Exercise:** Code Documentation
**Code chosen:** Task list merging — `merge_task_lists()` / `resolve_task_conflict()` (`task_list_merge.py`)

---

## 1. Original Code (Undocumented)

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

## 2. Documentation Generated Using Prompt 1 (Comprehensive Function Documentation)

```python
def merge_task_lists(local_tasks, remote_tasks):
    """
    Reconcile two independent collections of tasks (e.g. a local device and
    a remote server) into a single merged collection, determining which
    tasks need to be created or updated on each side to bring both sources
    into agreement.

    For any task ID present on only one side, the missing side is simply
    queued for creation. For task IDs present on both sides, conflicts are
    resolved field-by-field via resolve_task_conflict() -- see that
    function's docstring for the exact precedence rules.

    Args:
        local_tasks (dict[str, Task]): Tasks from the local source, keyed
            by task ID.
        remote_tasks (dict[str, Task]): Tasks from the remote source, keyed
            by task ID.

    Returns:
        tuple[dict, dict, dict, dict, dict]: A 5-tuple containing:
            - merged_tasks: the fully reconciled task collection
            - to_create_remote: tasks present only locally, need creating remotely
            - to_update_remote: tasks on both sides where remote is now stale
            - to_create_local: tasks present only remotely, need creating locally
            - to_update_local: tasks on both sides where local is now stale

    Raises:
        NameError: Indirectly, via resolve_task_conflict() -- a missing
            TaskStatus import in this module. This is a bug, not intended
            behavior.

    Note:
        A single task ID can appear in BOTH to_update_local and
        to_update_remote simultaneously -- this is expected, not a bug.
        Callers must check both dictionaries, not assume mutual exclusivity.
    """
```

```python
def resolve_task_conflict(local_task, remote_task):
    """
    Merge two versions of the same task, produced by independently editing
    a local and a remote copy, into a single reconciled version using
    three separate, independently-applied rules.

    Resolution rules, applied in order to the same result object:
        1. General fields (title, description, priority, due_date):
           more recent updated_at wins. Exact ties keep local's fields.
        2. Status: completion is sticky. DONE on either side wins
           regardless of recency, unless both sides are non-DONE and
           differ, in which case recency applies.
        3. Tags: always the union of both sides -- neither side loses a tag.

    Args:
        local_task (Task): The local version of the task.
        remote_task (Task): The remote version of the task. Must share
            the same task ID as local_task (not verified by this function).

    Returns:
        tuple[Task, bool, bool]: (merged_task, should_update_local,
        should_update_remote).

    Raises:
        NameError: TaskStatus is referenced but not imported anywhere in
            this module -- calling this function as written will raise
            NameError: name 'TaskStatus' is not defined.

    Note:
        should_update_local and should_update_remote are NOT mutually
        exclusive. Both can be True simultaneously (e.g. local wins
        general fields, remote wins status). Check both independently.
    """
```

---

## 3. Insights and Improvements from Prompt 2 (Intent and Logic Explanation)

**High-level intent:** Supports offline-first, multi-device sync — reconciling tasks edited independently on two disconnected sources without losing data from either.

**Assumptions/edge cases identified:**
- Assumes both `Task` objects share the same ID — never explicitly checked; the caller must guarantee this.
- Assumes `updated_at` is always populated and comparable — no null-check before the comparison.
- Assumes exact timestamp ties should favor local (strict `>`, not `>=`) — an implicit, undocumented design choice.
- **No handling for task deletion** — a task deleted on one side but still present on the other is currently indistinguishable from a "new" task and would be silently recreated rather than deleted. Flagged as a real gap for future sync work.

**Suggested inline comments** (for the trickiest parts): tie-handling in the general-fields rule, the sticky-completion override in the status rule, and the always-union behavior of the tags rule — see the fully commented version below.

**Potential improvements (functionality unchanged):**
- Import `TaskStatus` — currently missing, a real bug.
- Make the tie-handling behavior an explicit, named/commented decision rather than an implicit side-effect of using `>` over `>=`.
- Add a precondition check/docstring note that `local_task.id == remote_task.id`.
- Consider a "deleted"/tombstone concept if deletion sync is ever needed.

---

## 4. Final Combined Documentation

```python
import copy

def merge_task_lists(local_tasks, remote_tasks):
    """
    Reconcile two independent task collections (e.g. local device and
    remote server) into one merged collection, determining which tasks
    need creating or updating on each side to bring both into agreement.

    Supports offline-first, multi-device sync: tasks edited independently
    on two sources are merged field-by-field rather than one side simply
    overwriting the other.

    Args:
        local_tasks (dict[str, Task]): Tasks from the local source, keyed
            by task ID.
        remote_tasks (dict[str, Task]): Tasks from the remote source, keyed
            by task ID.

    Returns:
        tuple[dict, dict, dict, dict, dict]: (merged_tasks, to_create_remote,
        to_update_remote, to_create_local, to_update_local). See inline
        comments below for exactly when each dict is populated.

    Raises:
        NameError: Indirectly via resolve_task_conflict() -- TaskStatus is
            not currently imported in this module. Must be fixed before
            calling this function.

    Note:
        A single task ID can appear in BOTH to_update_local and
        to_update_remote simultaneously. This is expected, not a bug.

    Known gap:
        No handling for task deletion -- a task removed on one side but
        still present on the other will be treated as "new" and silently
        recreated, rather than deleted. Flag before using this for real
        sync involving deletions.
    """
    merged_tasks = {}
    to_create_remote = {}
    to_update_remote = {}
    to_create_local = {}
    to_update_local = {}

    # Union of IDs from both sides -- nothing is silently dropped.
    all_task_ids = set(local_tasks.keys()) | set(remote_tasks.keys())

    for task_id in all_task_ids:
        local_task = local_tasks.get(task_id)
        remote_task = remote_tasks.get(task_id)

        # Present only locally -> simply needs creating remotely.
        if local_task and not remote_task:
            merged_tasks[task_id] = local_task
            to_create_remote[task_id] = local_task

        # Present only remotely -> simply needs creating locally.
        elif not local_task and remote_task:
            merged_tasks[task_id] = remote_task
            to_create_local[task_id] = remote_task

        # Present on both sides -> real conflict resolution required.
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
    """
    Merge two versions of the same task (same ID) using three independent
    rules, applied in sequence to the same result object:

        1. General fields (title, description, priority, due_date):
           most-recently-updated version wins. Exact timestamp ties
           favor local (uses strict '>'; not documented elsewhere as
           intentional, but confirmed by testing).
        2. Status: completion is sticky. DONE overrides non-DONE
           regardless of recency. Only when neither side is DONE does
           "most recent wins" apply to status.
        3. Tags: always unioned -- neither side can lose a tag.

    Args:
        local_task (Task): Local version of the task.
        remote_task (Task): Remote version of the task. Must share the
            same task ID as local_task (precondition, not enforced here).

    Returns:
        tuple[Task, bool, bool]: (merged_task, should_update_local,
        should_update_remote). The two bool flags are NOT mutually
        exclusive -- both can be True if different rules favor different
        sides (e.g. local wins general fields, remote wins status).

    Raises:
        NameError: TaskStatus is referenced but not imported in this
            module -- a real, unresolved bug as of this documentation.

    Known gaps:
        - No None-check on updated_at before comparison.
        - No explicit precondition check that local_task.id == remote_task.id.
        - No concept of task deletion/tombstones.
    """
    merged_task = copy.deepcopy(local_task)
    should_update_local = False
    should_update_remote = False

    # Rule 1: recency wins for descriptive fields. Ties (equal timestamps)
    # fall to `else`, so local's fields are kept on an exact tie.
    if remote_task.updated_at > local_task.updated_at:
        merged_task.title = remote_task.title
        merged_task.description = remote_task.description
        merged_task.priority = remote_task.priority
        merged_task.due_date = remote_task.due_date
        should_update_local = True
    else:
        should_update_remote = True

    # Rule 2: completion is sticky and overrides rule 1 for status
    # specifically -- a DONE task can never be "un-completed" by sync.
    if remote_task.status == TaskStatus.DONE and local_task.status != TaskStatus.DONE:
        merged_task.status = TaskStatus.DONE
        merged_task.completed_at = remote_task.completed_at
        should_update_local = True
    elif local_task.status == TaskStatus.DONE and remote_task.status != TaskStatus.DONE:
        should_update_remote = True
    elif remote_task.status != local_task.status:
        # Neither side is DONE but they disagree -- same recency tiebreak as rule 1.
        if remote_task.updated_at > local_task.updated_at:
            merged_task.status = remote_task.status
            should_update_local = True
        else:
            should_update_remote = True
    # (implicit else: statuses already equal -- nothing to do)

    # Rule 3: tags are never resolved by precedence -- always union both
    # sides so a tag added on either device is never lost.
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

## 5. Discussion Notes (for Partner Comparison)

- **Prompt 1** produced formal, complete API-reference-style docstrings (types, returns, raises) — ideal for anyone calling this code without reading its internals.
- **Prompt 2** surfaced the *why* behind the code and real gaps (deletion handling, unchecked assumptions) that Prompt 1's structure doesn't naturally invite — better for someone about to modify or extend this logic.
- **Combined version** keeps Prompt 1's formal structure as the "reference," with Prompt 2's inline comments and "Known gaps" sections layered in — giving both a quick-reference and a deeper-context reading path in the same place.
