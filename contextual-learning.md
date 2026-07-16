# Contextual Learning with FastAPI

**Exercise:** Contextual Learning with FastAPI

**Comparison baseline:** Flask (no prior Flask/Django experience — Flask used as the general reference point)

The JWT auth app in Part 3 was actually installed, run, and tested end-to-end against a live server — not just read as sample code.

---

## Part 1: Framework Comparison

**"Compare FastAPI and Flask: what concepts are similar and what's fundamentally different?"**

*Similar:* both lightweight compared to Django's "batteries included" approach; both use route decorators; both are unopinionated about project structure.

*Fundamentally different:*
- **Async-first vs. sync-first:** FastAPI is ASGI/async from the ground up; Flask is WSGI/sync by default.
- **Validation is structural, not manual:** Flask requires manually parsing `request.json`/`request.form`; FastAPI infers validation directly from type hints via Pydantic.
- **Automatic documentation:** FastAPI generates interactive OpenAPI docs (`/docs`) with zero extra config; Flask needs a separate library for this.
- **Dependency injection is built-in:** FastAPI's `Depends()` is core to the framework; Flask has no first-class equivalent.

**"How does FastAPI's dependency injection system compare to Django's middleware?"**

Django middleware runs globally for every request, defined once in settings. FastAPI's `Depends()` is per-route, composable, and explicit — each endpoint declares exactly which dependencies it needs, and different endpoints can require entirely different chains. Middleware is "this runs for everything, always"; dependencies are "this specific route needs this specific thing."

**"If I'm used to Flask's Blueprint system, what's the equivalent in FastAPI?"**

`APIRouter` — group related routes in a separate module, then mount them with `app.include_router(router, prefix="/users")`, directly analogous to `app.register_blueprint(bp, url_prefix="/users")`.

**"How is FastAPI's request validation different from typical form validation in Django?"**

Django form validation is an explicit, separate step you call yourself (`form.is_valid()`). FastAPI's validation happens automatically the moment a request arrives, purely from type hints and Pydantic models — no separate "call validate" step; a mismatched request gets a 422 before your function body even runs.

### Translation Table

| Concept (Flask/Django) | FastAPI equivalent |
|---|---|
| `@app.route('/x', methods=['GET'])` | `@app.get('/x')` |
| `request.json` + manual validation | Pydantic `BaseModel` as a typed parameter |
| Flask Blueprint | `APIRouter` |
| Django Middleware | `Depends()` (per-route, not global) |
| Django `forms.Form` | Pydantic model, validated automatically |
| Flask-RESTX/Swagger setup | Automatic `/docs` (zero config) |
| WSGI sync view functions | `async def` route handlers (optional, but idiomatic) |

---

## Part 2: Understanding Design Choices

**Why Pydantic instead of a custom validation system?** Pydantic already solved "validate and parse data from Python type hints" as a mature, independent library — reusing it avoided reinventing validation and let FastAPI benefit from Pydantic's own ecosystem (editor autocomplete, static type checking) for free.

**What problem does automatic API documentation solve?** Hand-written API docs historically drifted out of sync with the actual implementation. Since FastAPI derives docs from the same type hints that drive validation, documentation can't drift — it's generated from the code itself, not a separate artifact maintained in parallel.

**Why type hints so extensively?** One annotation drives three things: request validation, response serialization, and automatic docs — rather than separate boilerplate for each concern.

**What motivated the async-first approach vs. Flask's sync approach?** FastAPI was built after Python's `async`/`await` matured, specifically for high-concurrency I/O-bound workloads. Flask predates mature async Python and stays sync-first for backward compatibility, with async support added later as an option rather than the default.

**Design philosophy summary:** FastAPI's core belief is that a single, correctly-typed piece of code should drive validation, serialization, documentation, and (via `Depends()`) authorization simultaneously — rather than treating these as four separate concerns each needing their own boilerplate. Structuring an application "the FastAPI way" means treating Pydantic models and type hints as the primary design tool from the start, not something added after the code already works.

---

## Part 3: Applied Contextual Learning — JWT Authentication

### Setup

```bash
pip install fastapi uvicorn "python-jose[cryptography]" "passlib[bcrypt]" python-multipart
```

### Real Issue Encountered and Fixed

The app failed on the very first login attempt with a `500 Internal Server Error`. The server log revealed:
```
AttributeError: module 'bcrypt' has no attribute '__about__'
...
ValueError: password cannot be longer than 72 bytes, truncate manually if necessary
```

**Root cause:** a known compatibility break between `passlib` and `bcrypt >= 4.0`, which removed the `__about__` attribute passlib uses internally to detect the bcrypt version. This is a genuine, real-world dependency-pinning issue — not something in the exercise's sample code itself, but something anyone actually running this exact code today would hit.

**Fix applied:**
```bash
pip install "bcrypt==4.0.1"
```

### Verified End-to-End Test Results (after the fix)

```
=== 1. Login with correct credentials ===
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...","token_type":"bearer"}

=== 2. Login with WRONG password ===
{"detail":"Incorrect username or password"}
HTTP status: 401

=== 3. Access protected /users/me/ WITH valid token ===
{"username":"johndoe","email":"johndoe@example.com","full_name":"John Doe","disabled":false}
HTTP status: 200

=== 4. Access protected /users/me/ WITHOUT token ===
{"detail":"Not authenticated"}
HTTP status: 401

=== 5. Access protected /users/me/ WITH garbage token ===
{"detail":"Could not validate credentials"}
HTTP status: 401

=== 6. Access /users/me/items/ WITH valid token ===
[{"item_id":"Foo","owner":"johndoe"}]
HTTP status: 200
```

Every case behaves exactly as designed: valid credentials issue a working JWT, wrong credentials are rejected, and protected routes correctly distinguish between no token, an invalid token, and a genuinely valid one.

### Reflection Questions

**How does FastAPI's approach to authentication compare to frameworks you've used before?**
Coming with no direct Flask/Django auth experience, the most notable thing is how much is handled *implicitly* through the type system: `OAuth2PasswordBearer` and `OAuth2PasswordRequestForm` aren't just utility functions — they're integrated into FastAPI's automatic docs (the `/docs` page shows a working "Authorize" button) and into the dependency graph, rather than being separate middleware or decorators bolted onto routes.

**What advantages does FastAPI's dependency injection system provide for authentication?**
`get_current_user` and `get_current_active_user` are ordinary `async def` functions, composed via `Depends()` — `get_current_active_user` itself depends on `get_current_user`, forming a chain. Any route needing an authenticated, active user just declares `current_user: User = Depends(get_current_active_user)` as a parameter — the authentication logic is testable and reusable independent of any specific route, and adding auth to a new route is a one-line parameter addition, not wrapping the function body in a decorator or manually checking headers.

**How does type hinting in FastAPI make security implementation clearer compared to other frameworks?**
The function signature `async def read_users_me(current_user: User = Depends(get_current_active_user))` documents, in one line, exactly what this route requires (an authenticated active user) and exactly what type it receives — readable without needing to trace through separate middleware configuration or decorator chains defined elsewhere.

**What patterns from other frameworks can you identify in the JWT implementation?**
The `fake_users_db` dictionary standing in for a real database mirrors a common pattern in any framework's tutorials/examples — a placeholder data layer meant to be swapped for a real ORM query later. The `authenticate_user` → `create_access_token` → return-token flow is a standard OAuth2 "password flow" shape that isn't FastAPI-specific at all; what's FastAPI-specific is only the *mechanism* (dependency injection) used to enforce it on protected routes.

---

## Part 4: Mental Model Translation

**"If I think of Django's middleware, views, and models as the core architectural components, what would be the corresponding mental model for FastAPI applications?"**

| Familiar concept | FastAPI equivalent | Key philosophical difference |
|---|---|---|
| Django/Flask **views** (route handlers) | FastAPI **path operation functions** (`@app.get`, `@app.post`) | Largely the same idea — a function that handles a request and returns a response |
| Django **models** | Pydantic **`BaseModel`** classes | Django models describe *persisted* data (tied to an ORM/database table); Pydantic models describe *shape and validation*, with no inherent database connection — a FastAPI app typically has both a Pydantic model (API shape) and a separate ORM model (storage shape) |
| Django **middleware** | FastAPI **`Depends()`** (mostly) + genuine ASGI middleware (for truly global concerns like CORS) | Middleware is global and implicit; `Depends()` is per-route and explicit — most "auth" and "get current user" logic that would be middleware in Django is a dependency in FastAPI |
| Flask **Blueprints** | FastAPI **`APIRouter`** | Nearly identical purpose: grouping related routes into a mountable, prefixed module |
| Django **forms** | Pydantic models as **request body parameters** | Django forms are explicitly instantiated and validated (`form.is_valid()`); FastAPI validation is automatic and implicit at the framework level |
| Django **admin/serializers** (DRF) | **`response_model=`** parameter on route decorators | Both control what shape of data actually gets sent back to the client, but FastAPI's version is declared directly on the route, not a separate serializer class layered on top |

**Differences in approach/philosophy noted:** Django and Flask both treat "validate the incoming data" as a distinct, explicit step the developer writes and calls. FastAPI treats validation as an automatic *consequence* of how you declared your function signature — the mental shift is trusting that declaring a type hint on a parameter is itself the validation logic, rather than looking for a separate validation call anywhere in the route body.

**Updated mental model after this exercise:** FastAPI routes should be thought of less like "a function that handles a request" (the Flask/Django framing) and more like "a typed function signature that the framework calls after already guaranteeing the input matches" — shifting where trust in the data's shape is established, from inside the function body to the function's declared parameters themselves.
