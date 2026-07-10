Exercise: Knowing Where to Start

Part 1: Understanding Project Structure and Technology Stack

1. Initial Understanding (Before AI Analysis)

How I thought the codebase was organized:
A small, single-package command-line application (no app.py, templates/, or static/ — ruling out a web framework). Flat structure at the root, no src/ layering.
•	cli.py — entry point, parses command-line args and wires user commands to underlying logic
•	models.py — probably a Task class/data structure (title, description, priority, status, due date, tags)
•	storage.py — saving/loading tasks, likely to/from JSON or a text file
•	task_manager.py — the "brain" — creates, updates, and queries tasks, sitting between cli.py and storage.py
•	task_parser.py — parses raw input (dates, tags, or import/export formats) into structured data
•	task_priority.py — logic specific to the priority field
•	task_list_merge.py — least obvious from the name alone — guessed at merging two task lists (dedup/conflict resolution)
•	tests/ — one test file per module, using unittest

Technologies/frameworks guessed
•	Pure Python standard library 
•	argparse for the CLI
•	unittest for testing 
•	json or plain text file for persistence
•	Possibly dataclasses for the Task model

Areas of lowest confidence
•	What task_list_merge.py actually does
•	Whether task_parser.py and task_priority.py are truly separate concerns from task_manager.py
•	What format storage.py persists to

2. Misconceptions Corrected by AI Analysis
•	Biggest misconception: assumed all seven top-level modules were part of one connected pipeline. In reality, only cli.py, models.py, storage.py, and task_manager.py are wired together — task_parser.py, task_priority.py, and task_list_merge.py are never imported by any of them. This was only discoverable by tracing imports (grep "^import|^from" *.py), not by reading file names or skimming individually.
•	Confirmed correct: the "no external dependencies" claim — every import across the project (argparse, json, os, re, copy, uuid, enum, datetime) is Python standard library.
•	Partially right: guessed dataclasses for the Task model — it's actually a plain class with a manual __init__, not a @dataclass.
•	Confirmed: storage.py does persist to a flat JSON file (tasks.json by default), using custom TaskEncoder/TaskDecoder classes since Task objects and enums aren't natively JSON-serializable.
•	Minor dead code found: task_manager.py imports argparse at the top but never uses it in that file — argparse is only actually needed in cli.py.

3. Entry Points and Architectural Patterns
Entry point
cli.py  →  main()  guarded by  if __name__ == "__main__":
Run with: python cli.py <command> [options]  (e.g. create, list, status, priority, due, tag, untag, show, delete, stats)

Architectural pattern
A clean layered/pipeline architecture, despite the flat folder structure — no MVC or web framework, but a clear separation of concerns:
cli.py  →  task_manager.py  →  storage.py  →  models.py
•	Presentation layer (cli.py) — argument parsing and output formatting only, no business logic
•	Service layer (task_manager.py) — all business rules and orchestration live here
•	Persistence layer (storage.py) — isolated JSON I/O, swappable in theory without touching business logic
•	Domain layer (models.py) — plain data + small self-contained behaviours, no dependencies on the other layers
Three additional modules (task_parser.py, task_priority.py, task_list_merge.py) sit outside this pipeline entirely — each depends only on models.py, and each is exercised solely by its own test file. They read as either unfinished features or utilities awaiting integration, not part of the active request flow.

4. Key Components and Responsibilities
Module	Role	Responsibility
cli.py	Entry point / presentation layer	Parses CLI args (argparse), formats output, dispatches to TaskManager. Run via `python cli.py <command>`.
models.py	Domain model	Task class + TaskPriority/TaskStatus enums. No persistence or CLI logic — pure data + small behaviours (mark_as_done, is_overdue).
task_manager.py	Service / business logic layer	TaskManager class — the layer CLI calls into. Owns create/list/update/delete/tag/stats operations.
storage.py	Persistence layer	TaskStorage class + custom JSONEncoder/Decoder. Reads/writes the entire tasks.json file on every change.
task_parser.py	Orphaned module (not wired in)	Parses free-text shorthand (e.g. "Buy milk @shopping !2 #tomorrow") into a Task. Never imported by cli.py or task_manager.py.
task_priority.py	Orphaned module (not wired in)	Calculates a weighted importance score per task (priority + urgency + tags + recency). Never imported elsewhere.
task_list_merge.py	Orphaned module (not wired in)	Merge/conflict-resolution logic for a local vs remote task dict. Looks like groundwork for a future sync feature.
tests/	Test suite	One test file per module, using unittest. Confirms task_parser, task_priority and task_list_merge are exercised only by their own tests.

5. Open Questions for the Team
•	Are task_parser.py, task_priority.py, and task_list_merge.py dead code, planned features, or used by something outside this repo?
•	Is there a plan to expose task_priority.py's scoring in list/stats, since list_tasks doesn't currently sort by importance?
•	Is task_list_merge.py meant for a future multi-device/remote sync feature? What would "remote" be?
•	Why does storage.py rewrite the entire tasks.json file on every single write instead of an incremental approach — known scalability limit?
•	Why does task_manager.py import argparse when it's never used in that file?

6. Verification Exercise
Run the following and inspect the output to confirm the full request path in one trace:
python cli.py create "Test task" -p 3 -t "test"
cat tasks.json
This single trace touches cli.py → task_manager.py → storage.py → models.py, confirming how a Task object is created, passed through the service layer, and serialized via the custom JSON encoder.
