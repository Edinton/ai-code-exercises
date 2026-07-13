# README & User Guide Documentation Exercise — Task Manager


**Exercise:** README and User Guide Documentation

**Project chosen:** Task Manager (Python CLI project from the Code Comprehension exercise)

---

## 1. Project Information Provided to the Prompts

- **Project name:** Task Manager
- **Description:** A command-line task management tool for tracking work items through a simple lifecycle, with priorities, due dates, and tags — no external dependencies.
- **Key features:**
  - Create tasks with title, description, priority, due date, and tags
  - List and filter tasks by status, priority, or overdue state
  - Update task status, priority, due date, and tags independently
  - View detailed task info and aggregate statistics
  - Persistent storage to a local JSON file
- **Technologies used:** Python 3.11+, standard library only (`argparse`, `json`, `uuid`, `datetime`, `enum`)
- **Installation requirements:** Python 3.11 or higher, no external dependencies
- **Code structure overview:** `cli.py` (entry point), `models.py` (Task/TaskStatus/TaskPriority), `task_manager.py` (business logic), `storage.py` (JSON persistence), plus `tests/` (unittest suite). Three additional modules (`task_parser.py`, `task_priority.py`, `task_list_merge.py`) exist in the codebase but are not currently wired into the CLI — verified by tracing imports in earlier exercises.

---

## 2. README.md Generated Using Prompt 1

```markdown
# Task Manager

A command-line task management tool for tracking work items through a simple
lifecycle — create, prioritize, tag, and complete tasks, all from your terminal.
Built entirely on the Python standard library, with zero external dependencies.

## Features

- **Task lifecycle tracking** — move tasks through `todo`, `in_progress`,
  `review`, and `done` states
- **Priorities** — four levels (low, medium, high, urgent) to help you focus
- **Due dates and overdue detection** — flag tasks that need attention
- **Tags** — free-form labels for flexible organization
- **Statistics** — a quick snapshot of your workload by status and priority
- **Local JSON persistence** — your tasks are saved to `tasks.json`, no
  database or server required

## Installation

### Prerequisites

- Python 3.11 or higher
- No additional packages needed — everything is standard library

### Setup

```bash
git clone <repository-url>
cd task-manager
```

That's it — no `pip install` step required.

## Usage

### Creating a task

```bash
python cli.py create "Write project proposal" \
  --description "Draft v1 for review" \
  --priority 3 \
  --due 2026-08-01 \
  --tags "writing,q3"
```

### Listing tasks

```bash
python cli.py list                     # all tasks
python cli.py list --status todo       # filter by status
python cli.py list --priority 3        # filter by priority (1=low .. 4=urgent)
python cli.py list --overdue           # only overdue tasks
```

### Updating a task

```bash
python cli.py status <task_id> in_progress
python cli.py priority <task_id> 4
python cli.py due <task_id> 2026-08-15
python cli.py tag <task_id> "urgent"
python cli.py untag <task_id> "urgent"
```

### Viewing details and statistics

```bash
python cli.py show <task_id>
python cli.py stats
```

### Deleting a task

```bash
python cli.py delete <task_id>
```

## Configuration

There is no separate configuration file. The only configurable value is the
storage location, which defaults to `tasks.json` in the working directory
(set via the `TaskStorage` constructor if embedding this project elsewhere).

## Troubleshooting

**"Task not found" on an update/delete/show command**
Double-check the task ID with `python cli.py list` — IDs are full UUIDs
internally, though `list`/`show` display only the first 8 characters for
readability. Use the full ID from `tasks.json` if the short form doesn't match.

**Due date update fails silently**
Dates must be in `YYYY-MM-DD` format exactly; anything else is rejected.

**Command not recognized**
Run `python cli.py` with no arguments to print the full list of available
subcommands and their options.

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/your-feature`)
3. Add tests for any new behavior under `tests/`, following the existing
   one-file-per-module convention
4. Run the test suite: `python -m unittest discover tests`
5. Open a Pull Request

## License

Add your project's license here (not specified in the current codebase).

## Notes on Project Structure

Three modules — `task_parser.py` (free-text task parsing), `task_priority.py`
(computed importance scoring), and `task_list_merge.py` (multi-source sync
conflict resolution) — are present in the codebase but not currently called
from `cli.py`. They appear to be either planned features or standalone
utilities awaiting integration; verified by tracing all imports across the
project.
```

---

## 3. Step-by-Step Guide Created Using Prompt 2

**Filled-in prompt:**

> Please create a step-by-step guide for how to **organize and manage a task from creation through completion, using priorities, due dates, and tags** in our application.
> Process overview: **A user wants to create a task, set it up properly with a priority/due date/tags, track it as work progresses, and finally mark it complete.**
> User experience level: **Beginner**

**Generated guide:**

```markdown
# How to Manage a Task from Creation to Completion

This guide walks you through the full lifecycle of a single task in Task
Manager — from creating it, to organizing it with metadata, to marking it
done.

## Prerequisites

- Task Manager installed (Python 3.11+, no extra setup needed)
- A terminal open in the project directory

## Step 1: Create the Task

Run the `create` command with a title (required) and any optional details:

```bash
python cli.py create "Prepare quarterly report" \
  --description "Compile Q3 metrics for leadership review" \
  --priority 3 \
  --due 2026-08-01 \
  --tags "reporting,q3"
```

This prints a confirmation with the new task's ID:
```
Created task with ID: 7f3a1c9e-...
```

**Common mistake:** forgetting quotes around a multi-word title — the shell
will otherwise treat each word as a separate argument and the command will fail.

## Step 2: Confirm It Was Created Correctly

```bash
python cli.py show 7f3a1c9e
```

You'll see the task's status (starts as `todo`), priority, due date, and tags
all displayed together. Double-check these match what you intended before
moving on.

## Step 3: Update Status as Work Progresses

As you start working on it:

```bash
python cli.py status 7f3a1c9e in_progress
```

Valid status values are `todo`, `in_progress`, `review`, and `done` — any
other value will be rejected by the command itself, so you can't typo your
way into an invalid state.

**Common mistake:** trying to set status with a value like `"in progress"`
(with a space) instead of `in_progress` (underscore) — the exact string
matters.

## Step 4: Adjust Priority or Due Date If Plans Change

```bash
python cli.py priority 7f3a1c9e 4      # bump to urgent
python cli.py due 7f3a1c9e 2026-08-10   # push the deadline back
```

## Step 5: Add or Remove Tags as Needed

```bash
python cli.py tag 7f3a1c9e "blocked"
python cli.py untag 7f3a1c9e "blocked"
```

## Step 6: Mark the Task Complete

```bash
python cli.py status 7f3a1c9e done
```

This automatically records a completion timestamp behind the scenes — you
don't need to do anything extra.

## Step 7: Review Your Progress

```bash
python cli.py stats
```

This shows your total task count, breakdown by status and priority, how many
tasks are currently overdue, and how many you've completed in the last 7 days.

## Troubleshooting

**"Failed to update task status. Task not found."**
The task ID you used doesn't match any existing task. Run `python cli.py list`
to confirm the correct ID.

**Task doesn't appear in `list` after creating it**
Make sure you're not accidentally filtering — try `python cli.py list` with
no flags first to confirm it exists, before narrowing with `--status` or
`--priority`.

**Due date rejected**
Confirm the format is exactly `YYYY-MM-DD` (e.g. `2026-08-01`), with no
time component and no slashes.
```

---

## 4. FAQ Document Created Using Prompt 3

**Filled-in prompt:**

> Basic information: **Task Manager, a Python CLI task tracker.** Target audience: **Developers trying it out for personal task tracking, likely first-time users of the tool.** Areas to focus on: **basic usage, filtering/organization, data storage/persistence.**

**Generated FAQ:**

```markdown
# Task Manager — Frequently Asked Questions

## Getting Started

### What is Task Manager?
A command-line tool for tracking tasks — create, prioritize, tag, and
complete work items without leaving your terminal. No database, no server,
no external dependencies.

### How do I install it?
Clone the repository and run it directly with Python 3.11+:
```bash
git clone <repository-url>
cd task-manager
python cli.py create "My first task"
```
No `pip install` step is required — everything is standard library.

### How do I see all available commands?
Run `python cli.py` with no arguments — it prints the full help text with
every subcommand.

## Basic Usage

### How do I create a task?
```bash
python cli.py create "Task title" --priority 3 --due 2026-08-01 --tags "work"
```
Only the title is required; everything else is optional.

### How do I see my tasks?
```bash
python cli.py list
```
Add `--status`, `--priority`, or `--overdue` to filter results.

### How do I mark a task done?
```bash
python cli.py status <task_id> done
```

### Can I see just the overdue tasks?
Yes:
```bash
python cli.py list --overdue
```
A task counts as overdue if its due date has passed and it isn't marked `done`.

## Organization: Priorities and Tags

### What do the priority numbers mean?
`1` = Low, `2` = Medium, `3` = High, `4` = Urgent.

### Can a task have multiple tags?
Yes — add tags one at a time with `tag`, or set several at creation with a
comma-separated list (`--tags "work,urgent"`).

### How do I remove a tag?
```bash
python cli.py untag <task_id> "tag-name"
```

## Data and Storage

### Where is my data stored?
In a local `tasks.json` file in your working directory — plain JSON, human-readable.

### Is my data safe if the program crashes mid-write?
Currently, no — the entire file is rewritten on every single change, with no
backup or atomic-write safeguard. If you're storing anything critical,
consider backing up `tasks.json` periodically.

### Can I sync tasks across two devices?
Not currently through the CLI — there's a `task_list_merge.py` module in the
codebase designed for exactly this, but it isn't wired into any command yet.

## Troubleshooting

### "Task not found" when updating or showing a task
Double-check the task ID with `python cli.py list` — you may be using an
outdated or mistyped ID.

### My due date isn't being accepted
Make sure it's in `YYYY-MM-DD` format exactly (e.g. `2026-08-01`).

### The `list` command shows nothing
If you've added `--status` or `--priority` filters, try running `list` with
no flags first to confirm tasks actually exist.

## Advanced / Roadmap

### Is there a way to export my tasks to another format (e.g. CSV)?
Not yet — this isn't currently supported, though it would be a natural
addition given the existing JSON-based storage layer.

### Is there an automatic "smart sort" by importance?
There's a `task_priority.py` module that computes a weighted importance score
(blending priority, due date, tags, and recency), but it isn't currently
connected to the `list` command — sorting today is just insertion order.
```

---

## 5. Discussion Notes

Compared to writing this from scratch, having the actual verified command set (rather than a guessed one) made the biggest difference in accuracy — several details (e.g. the exact subcommand being `status` rather than `update-status`, or `in_progress` needing an underscore) would have been easy to get subtly wrong without checking `cli.py` directly first. The FAQ's "Advanced / Roadmap" section was also only possible to write honestly because of the earlier code-comprehension exercises that confirmed which modules are built but not wired in — without that context, a generated FAQ might have falsely implied CSV export or smart sorting already work.
