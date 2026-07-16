# Getting Started with FastAPI — Exercise Journal

**Environment:** Python 3.12, FastAPI 0.139.0, Pydantic 2.13.4 — all code actually installed, built, and tested (using FastAPI's `TestClient`, which exercises the app directly without needing a persistent server process).

---

## Part 1: FastAPI Fundamentals

**What FastAPI is:** a modern Python web framework for building APIs, built on Starlette (async web layer) and Pydantic (data validation). It's designed around Python type hints — annotate a function parameter with a type, and FastAPI uses that to validate requests, serialize responses, and auto-generate interactive documentation.

**Comparison:**

| | FastAPI | Flask | Django |
|---|---|---|---|
| Async support | Native (`async def` throughout) | Bolted on, not the default | Supported since 3.1, not the default mental model |
| Data validation | Built-in via Pydantic + type hints | Manual, or via extensions (Marshmallow) | Via Django Forms / DRF serializers |
| Auto docs | Yes — Swagger UI + ReDoc, generated from code | No, needs an extension | DRF adds this; not core Django |
| Best for | APIs specifically | Small apps/APIs, very flexible | Full web apps (admin, ORM, templates) |
| Learning curve | Low-to-moderate — type hints do a lot of the work | Very low to start | Higher — more built-in conventions |

**Key advantages:** automatic request validation with clear error messages (from Pydantic); auto-generated interactive docs that stay in sync with the code because they're generated *from* it; native async support; editor autocomplete/type-checking benefits from using real Python type hints instead of framework-specific config objects.

**Glossary:**
- **Path operation** — a function decorated with `@app.get`, `@app.post`, etc., handling requests to a specific URL path + HTTP method.
- **Path parameter** — a variable embedded in the URL path, e.g. `/items/{item_id}`.
- **Query parameter** — an optional parameter after `?` in the URL, e.g. `/search?q=test`.
- **Request body** — JSON data sent in POST/PUT, typically validated against a Pydantic model.
- **Pydantic model** — a class (subclassing `BaseModel`) defining the shape and validation rules for data.
- **Dependency injection** — FastAPI's `Depends()` system for sharing reusable logic (auth, DB sessions) across path operations.
- **`response_model`** — declares the response shape, used both for output validation and docs generation.
- **ASGI** — Asynchronous Server Gateway Interface; FastAPI is an ASGI framework, `uvicorn` is the ASGI server that runs it.

---

## Part 2: First API — Hello World

### Code (`main.py`)
```python
from fastapi import FastAPI

app = FastAPI(
    title="My First FastAPI App",
    description="A simple API built with FastAPI",
    version="0.1.0"
)

@app.get("/")
async def root():
    """Root endpoint that returns a simple greeting message"""
    return {"message": "Hello World from FastAPI!"}

@app.get("/items/{item_id}")
async def read_item(item_id: int):
    """Endpoint that returns information about a specific item"""
    return {"item_id": item_id, "message": f"You requested item {item_id}"}

@app.get("/search/")
async def search_items(q: str = None, skip: int = 0, limit: int = 10):
    """Endpoint that searches for items based on query parameters"""
    return {
        "query": q,
        "skip": skip,
        "limit": limit,
        "message": f"Searching for '{q}' (skipping {skip}, limiting to {limit})"
    }
```

### Testing (actually run)

| Request | Result |
|---|---|
| `GET /` | `200` — `{'message': 'Hello World from FastAPI!'}` |
| `GET /items/42` | `200` — `{'item_id': 42, 'message': 'You requested item 42'}` |
| `GET /search/?q=test&skip=5&limit=20` | `200` — `{'query': 'test', 'skip': 5, 'limit': 20, 'message': "Searching for 'test' (skipping 5, limiting to 20)"}` |
| `GET /items/notanumber` | `422` — `{'detail': [{'type': 'int_parsing', 'loc': ['path', 'item_id'], 'msg': 'Input should be a valid integer, unable to parse string as an integer', 'input': 'notanumber'}]}` |

**The key "aha" moment:** the last test required zero manual validation code. `item_id: int` in the function signature was enough for FastAPI to reject a non-numeric path value automatically with a clear, structured error — this is the core value proposition of the framework in one test.

Also confirmed the auto-generated OpenAPI schema (what powers `/docs`) correctly discovered all 3 routes with zero separate documentation effort:
```
Title: My First FastAPI App
Paths discovered automatically:
  /              ['get']
  /items/{item_id}  ['get']
  /search/       ['get']
```

**To run for real** (outside this sandbox):
```bash
pip install fastapi uvicorn
uvicorn main:app --reload
```
Then visit `http://127.0.0.1:8000/docs` for the interactive Swagger UI.

---

## Part 3: Enhanced API — Structure, Pydantic Validation, Error Handling

### Project structure
```
part3/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── models/
│   │   ├── __init__.py
│   │   └── item.py
│   ├── routes/
│   │   ├── __init__.py
│   │   └── items.py
│   └── utils/
│       ├── __init__.py
│       └── exceptions.py
```

### A real finding: the exercise's example code targets Pydantic v1

The installed environment has **Pydantic 2.13.4**. The exercise's sample code used Pydantic v1-style config:
```python
class Config:
    schema_extra = {...}
```
This is deprecated in Pydantic v2. I updated it to the current syntax:
```python
model_config = ConfigDict(json_schema_extra={...})
```
Similarly, `item.dict()` (Pydantic v1) was updated to `item.model_dump()` (Pydantic v2) in the create-item route. This is a good example of why "use this if you don't like what the prompts gave you" reference code still needs a quick version-compatibility check — frameworks move fast enough that even a written reference can drift from what `pip install` gives you today.

### Testing (actually run)

| Request | Result |
|---|---|
| `GET /` | `200` — welcome message with docs/endpoints pointers |
| `POST /items/` `{"name": "Laptop", "description": "Dev machine", "price": 1299.99, "tags": [...]}` | `201` — item created with `id: 1` |
| `POST /items/` `{"name": "Mouse", "price": 25.50}` (no tags) | `201` — `tags` defaults to `[]`, `description` defaults to `None` |
| `POST /items/` `{"name": "Broken", "price": -5}` | `422` — custom handler: `"Input should be greater than 0"`, correctly caught by `Field(..., gt=0)` |
| `POST /items/` `{"price": 10}` (missing `name`) | `422` — custom handler: `"Field required"` |
| `GET /items/1` | `200` — returns the created item |
| `GET /items/999` | `404` — **custom `ItemNotFoundError` handler**: `{"detail": "Item with ID 999 not found"}` |
| `GET /items/?tag=electronics` | `200` — correctly filters to only the Laptop |
| `GET /items/` (no filter) | `200` — both items returned |

Both custom exception handlers (`ItemNotFoundError` → `404`, `RequestValidationError` → structured `422`) fired correctly and exactly as designed — confirming the "organize into multiple files/modules" pattern (models / routes / utils, wired together in `main.py` via `include_router` and `add_exception_handlers`) works end-to-end, not just in theory.

One more deprecation surfaced during testing: `status.HTTP_422_UNPROCESSABLE_ENTITY` triggered a `StarletteDeprecationWarning` — current Starlette prefers `HTTP_422_UNPROCESSABLE_CONTENT`. Applied that fix in Part 4's version.

---

## Part 4: Exercise Challenge — To-Do List API

### Requirements delivered
- Create a to-do item with title, description, due date
- List all items, with optional filtering by completion status
- Mark an item as completed
- Delete an item
- Validation + custom error handling
- Auto-generated docs (verified via the OpenAPI schema)

### Project structure
```
part4/
├── app/
│   ├── main.py
│   ├── models/todo.py
│   ├── routes/todos.py
│   └── utils/exceptions.py
└── requirements.txt
```

### Models (`app/models/todo.py`)
```python
from pydantic import BaseModel, Field
from datetime import date
from typing import Optional


class TodoBase(BaseModel):
    title: str = Field(..., min_length=1, max_length=200, description="Short title for the task")
    description: Optional[str] = Field(None, max_length=2000, description="Optional longer description")
    due_date: Optional[date] = Field(None, description="Date the task is due, if any (YYYY-MM-DD)")


class TodoCreate(TodoBase):
    pass


class TodoResponse(TodoBase):
    id: int
    completed: bool = False
```

### Routes (`app/routes/todos.py`)
```python
from fastapi import APIRouter, Path, Query, status
from typing import List, Optional

from ..models.todo import TodoCreate, TodoResponse
from ..utils.exceptions import TodoNotFoundError

router = APIRouter(prefix="/todos", tags=["todos"])

fake_todos_db = {}
todo_counter = 0


@router.post("/", response_model=TodoResponse, status_code=status.HTTP_201_CREATED)
async def create_todo(todo: TodoCreate):
    global todo_counter
    todo_counter += 1
    new_todo = {**todo.model_dump(), "id": todo_counter, "completed": False}
    fake_todos_db[todo_counter] = new_todo
    return new_todo


@router.get("/", response_model=List[TodoResponse])
async def list_todos(
    completed: Optional[bool] = Query(
        None, description="Filter by status: true for completed, false for pending, omit for all"
    )
):
    todos = list(fake_todos_db.values())
    if completed is not None:
        todos = [t for t in todos if t["completed"] == completed]
    return todos


@router.get("/{todo_id}", response_model=TodoResponse)
async def get_todo(todo_id: int = Path(..., gt=0)):
    if todo_id not in fake_todos_db:
        raise TodoNotFoundError(todo_id)
    return fake_todos_db[todo_id]


@router.patch("/{todo_id}/complete", response_model=TodoResponse)
async def complete_todo(todo_id: int = Path(..., gt=0)):
    if todo_id not in fake_todos_db:
        raise TodoNotFoundError(todo_id)
    fake_todos_db[todo_id]["completed"] = True
    return fake_todos_db[todo_id]


@router.delete("/{todo_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_todo(todo_id: int = Path(..., gt=0)):
    if todo_id not in fake_todos_db:
        raise TodoNotFoundError(todo_id)
    del fake_todos_db[todo_id]
    return None
```

### Full end-to-end test run

| # | Request | Result |
|---|---|---|
| 1 | `POST /todos/` — create "Write GenAI journal" with due date | `201`, `id: 1`, `completed: false` |
| 2 | `POST /todos/` — create "Buy groceries" (no due date) | `201`, `id: 2`, `due_date: null` |
| 3 | `POST /todos/` — create "Review PR" | `201`, `id: 3` |
| 4 | `POST /todos/` — empty title | `422` — `"String should have at least 1 character"` |
| 5 | `POST /todos/` — malformed due date (`"not-a-date"`) | `422` — `"invalid character in year"` |
| 6 | `GET /todos/` | `200` — all 3 items |
| 7 | `PATCH /todos/2/complete` | `200` — item 2 now `completed: true` |
| 8 | `GET /todos/?completed=true` | `200` — only item 2 |
| 9 | `GET /todos/?completed=false` | `200` — items 1 and 3 |
| 10 | `GET /todos/2` | `200` — item 2, completed |
| 11 | `GET /todos/999` | `404` — `"To-do item with ID 999 not found"` |
| 12 | `PATCH /todos/999/complete` | `404` — same message, confirming the not-found check applies consistently across endpoints |
| 13 | `DELETE /todos/3` | `204`, empty body |
| 14 | `GET /todos/3` (post-delete) | `404` — confirms deletion took effect |
| 15 | `DELETE /todos/3` again | `404` — deleting an already-deleted item fails cleanly instead of silently no-op'ing |
| 16 | `GET /todos/` (final) | `200` — items 1 and 2 only |

All 16 cases behaved exactly as intended on the first full run.

### Auto-generated documentation (verified via OpenAPI schema)
```
POST    /todos/                   -> Create Todo
GET     /todos/                   -> List Todos
GET     /todos/{todo_id}          -> Get Todo
DELETE  /todos/{todo_id}          -> Delete Todo
PATCH   /todos/{todo_id}/complete -> Complete Todo
GET     /                         -> Root
```
This is exactly what renders at `/docs` as an interactive Swagger UI — no separate documentation was written; it's derived entirely from the route decorators, Pydantic models, and docstrings above.

**To run for real:**
```bash
pip install -r requirements.txt
uvicorn app.main:app --reload
```
Then visit `http://127.0.0.1:8000/docs`.

---

## Key Takeaways

1. **Type hints aren't just documentation in FastAPI — they're the validation engine.** `item_id: int` alone rejected bad input with a structured `422` and no manual `if`/`try` code. This is the single clearest illustration of what makes FastAPI different from Flask, more valuable than any comparison table.
2. **Reference/example code needs a version sanity-check, even from a trusted source.** The exercise's own Part 3 sample used Pydantic v1 syntax that's deprecated under the Pydantic v2 actually installed — a good reminder that "copy this working example" still benefits from actually running it before trusting it.
3. **Custom exception handlers + `response_model` together give you both correctness and documentation for free.** Every error case (missing field, out-of-range value, not-found ID) produced a clean, predictable JSON shape — and all of that shape shows up automatically in `/docs`, without writing a separate API reference.
4. **Testing via `TestClient` instead of a live server was the right call in this environment** — background server processes didn't survive between sandboxed tool calls, but `TestClient` exercises the exact same ASGI app in-process, which is also genuinely the recommended way to write automated tests for a FastAPI app in real projects (pairs naturally with `pytest`).

## Files Included

- `part2/main.py` — Hello World API
- `part3/app/` — enhanced API with Pydantic models, routing, custom exception handling
- `part4/app/` — the to-do list CRUD API (the exercise challenge)
- `part4/requirements.txt`
