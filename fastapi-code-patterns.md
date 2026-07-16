# Understanding FastAPI Code Patterns

**Exercise:** Understanding FastAPI Code Patterns

The most important claim in this document — that the `requires_role` decorator's own `Depends()` declaration is dead code — was not just theorized, but **actually reproduced and confirmed with a real `AttributeError`** by running modified code against a live FastAPI app.

---

## Part 1: Analyzing Complex Code

### "Can you explain the Repository pattern being used in this code? Why structure it this way?"

The `Repository` class wraps direct database queries (`select(self.model)...`) behind simple method names (`get_by_id`, `list`). Rather than every service or route writing raw SQLAlchemy queries inline, they call `repository.get_by_id(...)`. Benefits: query logic lives in exactly one place per model, is independently testable, and swapping the underlying database/ORM later only requires changing the repository, not every call site.

### "What is the purpose of `Generic[T]` in the Repository class?"

`Repository(Generic[T])` lets one class definition serve *any* model type, while still preserving accurate type hints. `Repository[User]` (as used by `UserRepository(Repository[User])`) tells type checkers and IDEs that `get_by_id()` returns `Optional[User]`, not a generic, untyped object — avoiding both duplicating the same CRUD logic per model *and* losing type safety, which a naive "just use `Any`" approach would sacrifice.

### "How does dependency injection work in this application?"

Layered dependencies, each depending on the previous: `get_db()` provides a database session → `get_current_user(token, db)` depends on both the OAuth2 token *and* `get_db()` → route functions depend on `get_current_user()`. FastAPI resolves this whole chain automatically per-request based on function signatures, with each dependency only needing to know about the layer directly below it.

### "Can you break down the role-based access control system?"

On paper: `requires_role("admin")` wraps a route function so that before the real logic runs, it checks `current_user.is_superuser`. **This is exactly the part that turned out to have a real, verifiable flaw — see Part 2.**

---

## Part 2: Tracing Execution Flow — and a Real Bug Found

To actually trace `/admin/users/`'s execution, a simplified but faithful runnable version was built (swapping real SQLAlchemy `AsyncSession` for an in-memory dict, keeping every other pattern — `Repository[T]`, `UserRepository`, `get_current_user`, `requires_role`, `TimingMiddleware`, `lifespan` — unchanged).

### Expected flow (as read from the code)

```
Request -> TimingMiddleware (start timer)
  -> FastAPI resolves get_current_user (needs: oauth2_scheme token)
  -> requires_role("admin") wrapper checks current_user.is_superuser
  -> if admin: calls the real list_users(skip, limit, current_user)
  -> UserRepository.list() queries the "database"
  -> response_model=List[UserSchema] serializes the result
  -> TimingMiddleware adds X-Process-Time header
```

### First test: does it actually behave as expected?

```python
# Regular user (alice) tries /admin/users/
r = client.get('/admin/users/', headers={'Authorization': f'Bearer {alice_token}'})
# -> 403 {"detail":"Insufficient permissions"}

# Admin (admin_bob) tries /admin/users/
r = client.get('/admin/users/', headers={'Authorization': f'Bearer {admin_token}'})
# -> 200 [{"id":1,...}, {"id":2,...}]
```

**Both correct** — the RBAC check works as intended, on the surface.

### Digging deeper: is the decorator's own `Depends()` actually being used?

The `requires_role` wrapper declares its own copy of the dependency:
```python
async def wrapper(*args, current_user: User = Depends(get_current_user), **kwargs):
```
But the *original* `list_users` function **already independently declares the exact same dependency**:
```python
async def list_users(skip: int = 0, limit: int = 10, current_user: User = Depends(get_current_user)):
```

This raised a question worth testing directly: which one does FastAPI actually use?

```python
import inspect
sig = inspect.signature(list_users)
print(sig)
# -> (skip: int = 0, limit: int = 10, current_user: main.User = Depends(...))
print(hasattr(list_users, '__wrapped__'))
# -> True
```

**Confirmed:** `inspect.signature()` — which is exactly what FastAPI uses to build its dependency graph — follows `__wrapped__` (set automatically by `functools.wraps`) straight past the decorator's `wrapper` and reads the **original, undecorated function's signature**. FastAPI never sees the wrapper's own `current_user: User = Depends(get_current_user)` parameter at all.

### Proving it: removing the "redundant" dependency breaks everything

To confirm this wasn't just theoretical, the *original* function's `current_user: User = Depends(get_current_user)` was removed, leaving it declared **only** in the decorator wrapper (exactly as a developer might "clean up" the code, reasonably assuming it was duplicated):

```python
async def list_users(skip: int = 0, limit: int = 10):  # dependency removed here
    ...
```

**Actual result when calling `/admin/users/` as an admin:**
```
File ".../main_no_duplicate.py", line 111, in wrapper
    if role == "admin" and not current_user.is_superuser:
                               ^^^^^^^^^^^^^^^^^^^^^^^^^
AttributeError: 'Depends' object has no attribute 'is_superuser'
```

This confirms it precisely: with the "duplicate" removed, FastAPI's introspected signature no longer includes any `current_user` dependency at all (only `skip`/`limit` remain), so FastAPI never resolves or passes a `current_user` keyword argument. Python then falls back to the wrapper's own literal default value — which is the **unresolved `Depends` object itself**, not an authenticated `User` — and the `.is_superuser` check crashes immediately.

**Conclusion:** the original code's RBAC decorator only works *by coincidence of duplication*. The decorator's own `Depends()` is dead code from FastAPI's perspective, silently relying on every decorated route function independently re-declaring the identical dependency. This is a fragile, non-obvious coupling that would break instantly if anyone "cleaned up" what looked like a redundant line.

---

## Part 3: Simplifying Complex Concepts

**`asynccontextmanager` + `lifespan`, in simple terms:** normally, running setup code before a server starts and cleanup code after it stops requires separate `@app.on_event("startup")`/`@app.on_event("shutdown")` handlers. `lifespan` combines both into one function using `yield` as the dividing line — everything before `yield` runs at startup, everything after runs at shutdown, similar to a Python context manager (`with open(...) as f:`) but scoped to the entire application's lifetime instead of a single `with` block.

**`TimingMiddleware`, simplified:** middleware wraps *every* request. `call_next(request)` is "go run the actual route handler and give me its response" — everything before that call happens *before* the route runs, everything after happens *after*. Here, that's just: note the time, let the request proceed, measure how long it took, and stick that number in a response header (`X-Process-Time`) before sending the response back.

**JWT authentication flow, simplified for a junior developer:** logging in doesn't create a session stored on the server — it creates a signed, self-contained token (the JWT) containing the username and an expiry time, sent back to the client. Every future request includes that token in the `Authorization` header; the server re-verifies the token's signature and reads the username straight out of it — no database lookup needed just to know *who* is asking, only to look up *their current data* once identity is established.

---

## Part 4: Building Understanding Through Implementation — Action Logging Feature

**Feature:** log user actions (logins, admin actions) using the same patterns already present in the code.

**Approach followed:** extend the existing `Repository` pattern (rather than inventing a new persistence mechanism), and hook logging into existing dependency/route code — following the codebase's own conventions instead of introducing a new one.

```python
class ActionLog:
    def __init__(self, id, username, action, timestamp):
        self.id = id
        self.username = username
        self.action = action
        self.timestamp = timestamp


ACTION_LOG_DB: List[ActionLog] = []


class ActionLogRepository(Repository[ActionLog]):
    """Follows the exact same Repository[T] pattern as UserRepository,
    rather than writing standalone logging logic disconnected from the
    rest of the codebase's architecture."""

    async def add(self, username: str, action: str) -> ActionLog:
        entry = ActionLog(
            id=len(ACTION_LOG_DB) + 1,
            username=username,
            action=action,
            timestamp=datetime.utcnow(),
        )
        ACTION_LOG_DB.append(entry)
        return entry

    async def list_for_user(self, username: str) -> List[ActionLog]:
        return [entry for entry in ACTION_LOG_DB if entry.username == username]


# Dependency, following the same style as get_db()
async def get_action_log_repo() -> ActionLogRepository:
    return ActionLogRepository(ActionLog)


# Hook into the login route
@app.post("/token")
async def login(username: str, password: str,
                 log_repo: ActionLogRepository = Depends(get_action_log_repo)):
    user_repo = UserRepository(User)
    user_service = UserService(user_repo)
    user = await user_service.authenticate_user(username, password)
    if not user:
        await log_repo.add(username, "failed_login")
        raise HTTPException(status_code=401, detail="Incorrect username or password")

    await log_repo.add(username, "login")
    access_token = user_service.create_access_token(data={"sub": user.username})
    return {"access_token": access_token, "token_type": "bearer"}


# Hook into the admin action (fixing the RBAC bug found in Part 2 at the same time,
# by NOT relying on the decorator's own broken Depends())
@app.get("/admin/users/", response_model=List[UserSchema])
@requires_role("admin")
async def list_users(
    skip: int = 0,
    limit: int = 10,
    current_user: User = Depends(get_current_user),  # kept -- this is the ONE that actually matters
    log_repo: ActionLogRepository = Depends(get_action_log_repo),
):
    await log_repo.add(current_user.username, "viewed_admin_user_list")
    user_repo = UserRepository(User)
    users = await user_repo.list(skip=skip, limit=limit)
    return users
```

**Design decisions explained:**
- `ActionLogRepository` follows `Repository[T]` exactly, rather than being a one-off logging utility — consistent with the "generic repository per model" pattern already established.
- Logging is injected as a dependency (`Depends(get_action_log_repo)`) rather than a bare global function call, matching how `get_db()` is used everywhere else — keeps logging testable and swappable (e.g., to a real database later) without touching route logic.
- The admin route change deliberately **keeps** `current_user: User = Depends(get_current_user)` on the original function — now understood, per Part 2's finding, to be the dependency that actually matters, not a redundant leftover.

**AI review of this implementation, and suggested improvements:**
1. `ACTION_LOG_DB` as a plain module-level list mirrors the exercise's own `fake_users_db` simplification, but in real code this should go through the same `AsyncSession`/database pattern as `User`, not a separate in-memory structure — worth flagging as a known simplification, not a final design.
2. Logging failed logins with the *attempted* username (rather than nothing) is a reasonable security-monitoring choice, but be careful never to log the attempted *password* alongside it — this implementation correctly avoids that.
3. Given Part 2's finding, a stronger long-term fix would be to make `requires_role` fetch `current_user` itself via `Depends()` correctly (e.g., using FastAPI's own dependency-injection utilities rather than a hand-written decorator at all, since decorators and FastAPI's signature-based DI don't compose safely by default, as demonstrated).

