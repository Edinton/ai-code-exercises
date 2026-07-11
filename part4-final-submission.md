# Task Manager (Python) — Final Submission Summary


**Exercise:** Knowing Where to Start
**Part 4:** Practical Application, Reflection & Submission

---

## 1. Initial vs. Final Understanding

**Initial understanding (before AI-assisted exploration):**
I assumed the seven top-level modules formed one connected pipeline, that `TaskStatus` enforced a strict linear workflow (todo → in_progress → review → done), and that `TaskPriority` and the computed importance score were roughly the same idea. I also wasn't sure whether `storage.py` used JSON, a database, or plain text.

**Final understanding (after Parts 1–3):**
The active pipeline is actually just `cli.py → task_manager.py → storage.py → models.py`; three modules (`task_parser.py`, `task_priority.py`, `task_list_merge.py`) are fully-built but orphaned — never imported anywhere in the live app, only exercised by their own tests. `TaskStatus` is a fixed label with no enforced transitions at all — any status can follow any other. Priority (static, user-set) and the importance score (computed, blending priority + urgency + status + tags + recency) are genuinely distinct concepts that happen to overlap in vocabulary (`URGENT` priority vs. `"urgent"` tag), which is a real source of confusion if unexamined. Persistence is a single flat `tasks.json` file, rewritten in full on every change, via a custom JSON encoder/decoder since `Task` isn't natively serializable.

---

## 2. Most Valuable Insights from Each Prompt

- **Project Structure prompt (Part 1):** The single highest-value insight was catching that half the codebase isn't wired in. That's invisible from file names or folder layout alone — only tracing imports revealed it, and it reframed every later part of the exercise (e.g. "orphaned module" became a recurring pattern to check for).
- **Feature Location prompt (Part 2):** Taught a transferable question — "does this module already know how to serialize/transform data, or does it just consume already-prepared data?" — which is what correctly separated where new logic (CSV export) belongs from where existing logic (JSON storage) should stay untouched.
- **Domain Understanding prompt (Part 3):** Surfaced business rules that aren't obvious from reading code top-to-bottom, like "DONE always wins in a sync conflict regardless of timestamp" — a deliberate one-way-door decision, not a bug — and the fact that "overdue" is a live computed predicate, not a stored state.

---

## 3. Approach to Implementing the New Business Rule

For "tasks overdue more than 7 days should be marked abandoned unless high priority":

1. Add an `ABANDONED` value to `TaskStatus` in `models.py` (doesn't exist yet — confirmed by re-checking the enum before assuming this was a simple logic change).
2. Add a new method in `task_manager.py` (e.g. `apply_abandonment_rule()`) that checks days-overdue and priority per task, sets status, and persists via existing `storage.save()`.
3. Decide how the rule is triggered — this app has no scheduler or background process, so it would need to run either on every CLI invocation or via an explicit new subcommand — flagged as a question for the team rather than assumed.
4. No changes needed to `storage.py` or `models.py`'s core fields; the existing save/update mechanisms are reusable as-is.
5. Before writing code, resolved the ambiguity of whether "high priority" includes `URGENT` — since the rule's wording is genuinely unclear, this needs a team decision, not a guess baked silently into the implementation.

---

## 4. Strategies for Approaching Unfamiliar Code in the Future

- **Trace imports before trusting file names.** A file's name describes intent, not necessarily reality — grep `^import|^from` across the codebase early to see what's actually wired together versus what's aspirational or abandoned.
- **Separate "static/stored" concepts from "computed/derived" ones** when reading domain models (e.g. `priority` vs. `importance score`; `status` vs. `is_overdue()`). Conflating the two leads to wrong assumptions about what a code change will actually affect.
- **Watch for words that are reused with different meanings** (e.g. `URGENT` priority vs. `"urgent"` tag) — these are common, quiet sources of bugs in codebases with informal/undocumented conventions.
- **Write down assumptions as explicit open questions before implementing**, rather than resolving ambiguity silently in code — especially for business rules with vague wording, since a wrong silent assumption is far more expensive to unwind later than asking upfront.
- **Use AI prompts as a second pass, not a first pass.** Forming an independent hypothesis first (even a wrong one) made the AI's corrections far more memorable and useful than asking a fresh question with no prior guess to compare against.
