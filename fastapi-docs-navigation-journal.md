# Documentation Navigation for FastAPI — Exercise Journal



## Part 1: Documentation Summarization — A Reading Roadmap


**Recommended reading order for someone new to FastAPI:**

1. **Tutorial — First Steps** — the minimal "Hello World" app; establishes the core mental model (path operations, decorators).
2. **Tutorial — Path Parameters** and **Query Parameters** — how type hints drive automatic parsing/validation for the two most common input sources.
3. **Tutorial — Request Body** — introduces Pydantic models as the way to validate structured input, the single most important conceptual leap from "Flask-style" thinking.
4. **Tutorial — Dependencies** — FastAPI's dependency injection system; this is the mechanism nearly every non-trivial app (auth, DB sessions, shared logic) is built on.
5. **Tutorial — Security** (First Steps → OAuth2 with Password and JWT) — the standard, docs-recommended way to add authentication.
6. **Tutorial — Handling Errors** — `HTTPException` and custom exception handlers for consistent error responses.
7. **Advanced — Background Tasks** and **Advanced — WebSockets** — once the core model is solid, these cover async/real-time patterns as needed.

**5 most important sections for building a REST API quickly** (if time is limited): First Steps, Request Body (Pydantic), Path/Query Parameters, Dependencies, and Security (OAuth2/JWT) — these five cover the shape of nearly every CRUD + auth API.

**A real finding from fetching the live docs:** the current documentation recommends running a FastAPI app with the `fastapi dev` CLI command (from the `fastapi-cli` package, installed via `pip install "fastapi[standard]"`), rather than the `uvicorn main:app --reload` command used in our earlier FastAPI exercise. Both work — `fastapi dev` is a friendlier wrapper around uvicorn with better default output — but it's a good example of why re-checking docs periodically matters even for something as basic as "how do I run this."

---

## Part 2: Documentation Deep Dive — Dependency Injection & Security

**Feature chosen:** `Depends()` — specifically because it's the piece every other advanced feature (auth, DB sessions, shared validation) sits on top of.

### What the docs actually say (fetched from `/tutorial/dependencies/`)

A **dependency** is just a function (or callable class) that FastAPI calls before your path operation function, using the same parameter-declaration system as path operations themselves — meaning a dependency can *also* declare path parameters, query parameters, and even its own sub-dependencies. FastAPI resolves the whole chain, caching each dependency's result per-request by default so a dependency used in multiple places in the same request only runs once.

**Why `Depends()` matters practically:**
- **Shared logic without inheritance or mixins.** A dependency like `get_current_user` can be reused across every protected route without any class hierarchy — just a function reference.
- **Testability.** Because dependencies are just callables, FastAPI's `app.dependency_overrides` lets you swap a real dependency (e.g. a DB session) for a fake one in tests — the docs specifically frame this as a first-class use case, not an afterthought.
- **Automatic documentation.** Whatever a dependency declares (e.g. a required header) shows up in the OpenAPI schema at the *route* level, even though the parameter is defined once in the dependency function, not repeated per-route.

### When to use `Depends()` vs. when not to
Based on the docs and confirmed while building Part 4: use it for anything that (a) needs to run before the route logic, (b) is shared across more than one route, or (c) represents "external" state like auth or a DB session. Don't reach for it for pure business logic specific to one route — that's just a regular function call inside the route body; wrapping it in `Depends()` adds indirection without buying anything.

### Security docs (fetched from `/tutorial/security/oauth2-jwt/`)
The current docs' recommended JWT pattern uses:
- `pwdlib` (with the Argon2 hasher) for password hashing — **not** `passlib`, which is what most existing tutorials/blog posts still show.
- `pyjwt` for encoding/decoding tokens.
- `OAuth2PasswordBearer` to extract the bearer token from the `Authorization` header, and `OAuth2PasswordRequestForm` to accept the standard OAuth2 password-flow form fields (`username`/`password`) at a `/token` endpoint.
- A `get_current_user` dependency that decodes the token and raises `401` on any failure, so every protected route just declares `Depends(get_current_user)` and gets consistent auth behavior for free.

This exact pattern is what I used to build Part 4's auth — see below for the fully tested implementation.

---

## Part 3: Concept-to-Code Translation

I ran the exercise's own Part 3 reference example before doing anything else with it — and it failed immediately:

```
NameError: name 'Header' is not defined
```

`Header` is used in `verify_api_key` but never imported — a bug in the exercise's own reference code. I fixed that, and while fixing it also applied two more updates the current Pydantic v2 environment requires:
- `class Config: schema_extra = {...}` → `model_config = ConfigDict(json_schema_extra={...})`
- `user.dict()` → `user.model_dump()`

I also adopted the `Annotated[Type, Depends(...)]` dependency-declaration style throughout, since that's what the current docs actually recommend (the exercise's code imported `Annotated` but never used it — likely left over from an earlier documented pattern that used bare default-value `Depends(...)`).

### All 5 concepts — tested end-to-end

**1. Dependency Injection** (chained dependencies: `verify_api_key` → `get_current_user` → route)
```
valid key:   200 {'message': 'Hello, Demo User!', ...}
invalid key: 403 {'detail': 'Invalid API key'}
missing key: 422 {'detail': ..., 'errors': [{'loc': ['header', 'x-api-key'], 'msg': 'Field required', ...}]}
```

**2. Pydantic request/response validation** (password silently excluded via `response_model`)
```
valid:   200 {'username': 'johndoe', 'email': 'john@example.com', 'full_name': None, 'id': 1001}
invalid: 422 — 3 separate field errors (username too short, invalid email, password too short) returned together
```

**3. Background tasks** — confirmed the background function actually executed (printed "Sending welcome email to new@example.com...") after the response was already prepared, demonstrating the "don't block the response" behavior the docs describe, not just returning a 200 and hoping.

**4. Path/query/header/cookie parameters together**
```
200 {'item_id': 42, 'query_param': 'test', 'limit': 5, 'user_agent': 'pytest-client', 'session': 'abc123'}
```
All four parameter sources resolved correctly from a single request in one test.

**5. Custom exception handling**
```
not found (id=404): 404, custom header X-Error-Code: PRODUCT_NOT_FOUND present
bad id (id=-1):      400, "Product ID must be a positive integer"
valid (id=5):        200, full product data
```

*(Full corrected, tested code in `part3/reference.py`.)*

---

## Part 4: Comprehensive Documentation Challenge — Blog API

Built a complete blog API: JWT-based registration/login, post CRUD with **owner-only** edit/delete enforcement, comments, and search — directly applying the patterns confirmed in Parts 1–2 rather than guessing.

### How the documentation informed each implementation choice

| Feature | Documentation informing it | What I applied |
|---|---|---|
| Password storage | Security → OAuth2 JWT tutorial | `pwdlib` with Argon2 (`PasswordHash.recommended()`), not a rolled-my-own hash or plaintext |
| Login endpoint | Same tutorial | `OAuth2PasswordRequestForm` at `POST /token`, form-encoded (not JSON) — this is a real gotcha: it's easy to assume login should accept JSON like every other endpoint here, but the OAuth2 password flow specifically expects form data |
| Route protection | Dependencies tutorial | A single `get_current_user` dependency reused across every write endpoint (create/update/delete post, create comment) — one place to get auth logic right, not duplicated per-route |
| Ownership enforcement | Not directly in the docs (an application-level decision) | Compared `post["author"]` against the authenticated user inside each route, after confirming via the dependencies docs that `Depends()` gives you the *user*, not authorization decisions — those are still the route's job |
| Response shaping | Request Body / `response_model` tutorial | `UserResponse` never includes `hashed_password`, enforced structurally by the response model rather than by remembering to strip a field manually |
| Search | Query Parameters tutorial | A plain `Query(min_length=1)` parameter with a simple case-insensitive substring match — the docs don't prescribe a specific search implementation, so this was an application-level design choice informed by "basic search" in the requirements, not over-engineered into something like full-text search |

### Project structure
```
part4/
├── app/
│   ├── main.py
│   ├── db.py                 # in-memory data store
│   ├── auth.py                # password hashing + JWT + get_current_user
│   ├── models/
│   │   ├── user.py
│   │   ├── post.py
│   │   └── comment.py
│   └── routes/
│       ├── auth.py            # /register, /token
│       ├── posts.py           # CRUD + /posts/search
│       └── comments.py        # nested under /posts/{post_id}/comments
└── requirements.txt
```

### Full end-to-end test run (17 scenarios, all passed on first run)

| # | Action | Result |
|---|---|---|
| 1 | Register alice | `201` |
| 2 | Register bob | `201` |
| 3 | Register alice again (duplicate) | `400` — "Username already registered" |
| 4 | Login alice (form data) | `200` — real JWT issued |
| 5 | Login alice, wrong password | `401` |
| 6 | Login bob | `200` |
| 7 | Alice creates a post | `201` |
| 8 | Bob creates a post | `201` |
| 9 | Create post with no auth token | `401` — "Not authenticated" |
| 10 | List all posts | `200` — 2 posts |
| 11 | Get single post | `200` |
| 12 | **Bob tries to edit Alice's post** | `403` — "You can only edit your own posts" |
| 13 | Alice edits her own post | `200` — succeeds |
| 14 | **Bob tries to delete Alice's post** | `403` |
| 15 | Search `"pydantic"` | `200` — matches Bob's post only |
| 16 | Search `"TYPE HINTS"` (case-insensitive, content match) | `200` — matches Alice's post only |
| 17 | Bob comments on Alice's post | `201` |
| 18 | Comment with no auth | `401` |
| 19 | Comment on nonexistent post | `404` |
| 20 | List comments on Alice's post | `200` — Bob's comment returned |
| 21 | Alice deletes her own post | `204` |
| 22 | Confirm deletion | `404` |

The two ownership-enforcement tests (#12, #14) were the ones I was most careful to verify actually work as intended, since silently allowing cross-user edits would be a serious, easy-to-miss bug in a real blog API — confirmed both correctly reject with `403` while the *same actions by the actual owner* (#13, #21) succeed.

### Auto-generated API surface (confirmed via OpenAPI schema — this is exactly what `/docs` renders)
```
POST    /register                    auth=False
POST    /token                       auth=False
GET     /posts/                      auth=False
POST    /posts/                      auth=True
GET     /posts/search                auth=False
GET     /posts/{post_id}             auth=False
PUT     /posts/{post_id}             auth=True
DELETE  /posts/{post_id}             auth=True
POST    /posts/{post_id}/comments/   auth=True
GET     /posts/{post_id}/comments/   auth=False
GET     /                            auth=False
```
Reads are public, writes require a valid bearer token — exactly as designed, and this authorization requirement is visible automatically in the interactive docs (Swagger UI shows a lock icon on each protected route) without writing any separate documentation.

**To run for real:**
```bash
pip install -r requirements.txt
uvicorn app.main:app --reload
```
Then visit `http://127.0.0.1:8000/docs`, use the "Authorize" button with a token from `POST /token`, and try the protected endpoints interactively.

---

## Key Takeaways

1. **Documentation drifts, even for widely-used frameworks — fetching the live docs instead of relying on memory caught two real, current discrepancies:** the recommended run command changed from `uvicorn` to `fastapi dev`, and the recommended password-hashing library changed from `passlib` to `pwdlib`. Neither would have surfaced from a purely memory-based answer.
2. **Reference/example code — even from a trusted exercise source — benefits from actually running it before trusting it.** The Part 3 example had a real bug (`Header` unimported) that a read-through alone didn't catch; running it did, immediately.
3. **`Depends()`'s real value is chaining + reuse, not just "getting a value into a route."** Building the blog API's auth made this concrete: `get_current_user` depends on `oauth2_scheme`, and every protected route depends on `get_current_user` — one dependency graph, defined once, correctly enforced everywhere it's used.
4. **Authorization is a layer the framework deliberately doesn't own.** FastAPI's dependency system will happily tell you *who* is authenticated, but *whether that user is allowed to do this specific thing* (like editing someone else's post) is application logic you still have to write explicitly — confirmed by testing that this check needed to be added by hand in each route, it wasn't something `Depends()` gave for free.

## Files Included

- `part3/reference.py` — corrected, fully tested version of all 5 concept-to-code examples
- `part4/app/` — the complete blog API (auth, posts, comments, search)
- `part4/requirements.txt`
