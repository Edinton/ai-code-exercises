# Design Pattern Implementation Challenge — Factory Pattern (Python)

**Exercise:** Design Pattern Implementation Challenge

**Pattern chosen:** Factory Pattern — Database Connection Manager (Python)

All tests in this document were actually written and executed (via `unittest`), including a direct behavior-preservation comparison against the original code — final result: **12/12 tests passing.**

---

## Original Code

```python
class DatabaseConnection:
    def __init__(self, db_type, host, port, username, password, database,
                 use_ssl=False, connection_timeout=30, retry_attempts=3,
                 pool_size=5, charset='utf8'):
        self.db_type = db_type
        self.host = host
        self.port = port
        self.username = username
        self.password = password
        self.database = database
        self.use_ssl = use_ssl
        self.connection_timeout = connection_timeout
        self.retry_attempts = retry_attempts
        self.pool_size = pool_size
        self.charset = charset
        self.connection = None

    def connect(self):
        print(f"Connecting to {self.db_type} database...")

        if self.db_type == 'mysql':
            connection_string = f"mysql://{self.username}:{self.password}@{self.host}:{self.port}/{self.database}"
            connection_string += f"?charset={self.charset}"
            connection_string += f"&connectionTimeout={self.connection_timeout}"
            if self.use_ssl:
                connection_string += "&useSSL=true"
            print(f"MySQL Connection: {connection_string}")

        elif self.db_type == 'postgresql':
            connection_string = f"postgresql://{self.username}:{self.password}@{self.host}:{self.port}/{self.database}"
            if self.use_ssl:
                connection_string += "?sslmode=require"
            print(f"PostgreSQL Connection: {connection_string}")

        elif self.db_type == 'mongodb':
            connection_string = f"mongodb://{self.username}:{self.password}@{self.host}:{self.port}/{self.database}"
            connection_string += f"?retryAttempts={self.retry_attempts}"
            connection_string += f"&poolSize={self.pool_size}"
            if self.use_ssl:
                connection_string += "&ssl=true"
            print(f"MongoDB Connection: {connection_string}")

        elif self.db_type == 'redis':
            print(f"Redis Connection: {self.host}:{self.port}/{self.database}")

        else:
            raise ValueError(f"Unsupported database type: {self.db_type}")

        print("Connection successful!")
        return self.connection
```

---

## Prompt 1 — Pattern Opportunity Identification

**Structures/problems identified:**
- A single `__init__` with 11 parameters, most only relevant to *some* database types (`charset` only matters for MySQL; `pool_size`/`retry_attempts` only for MongoDB).
- A single `connect()` method with a growing `if/elif` chain, one branch per database type.
- The class conflates generic connection configuration with database-specific connection logic — every new type means editing this one method.

**Pattern suggested:** **Factory Pattern** — object creation is genuinely complex (many parameters, only some relevant per type) and the concrete type isn't known until runtime (`db_type` is a string).

**Benefits:** callers no longer need to know which of 11 parameters matter for their chosen type; adding a new database type becomes "add a class," not "grow a method"; each type's logic becomes independently testable.

**Drawbacks/challenges:** more files/classes overall; existing callers instantiating `DatabaseConnection` directly would need to migrate to the new factory API.

**Priority:** High impact, moderate effort — the current design gets worse with every new type added, and the refactor is mostly mechanical extraction, not new logic.

---

## Prompt 2 — Pattern Implementation Guidance

**Proposed approach:** separate class per database type + a `DatabaseConnectionFactory.create(db_type, **kwargs)` entry point.

**AI's refinement:** all subclasses should implement a common interface/abstract base class so calling code can treat any connection type uniformly (`connection.connect()`) without knowing the concrete type — otherwise the factory would return incompatible types, defeating part of the purpose.

**Step-by-step plan followed:**
1. Define abstract base `DatabaseConnection` with an abstract `connect()`.
2. Create one concrete subclass per type, each with only its own relevant extra parameters.
3. Move each `elif` branch's logic into that type's own implementation.
4. Create `DatabaseConnectionFactory.create(db_type, **kwargs)`, raising the same `ValueError` for unsupported types.
5. Update calling code to use the factory instead of instantiating directly.

---

## Prompt 3 — Multi-Pattern Architecture Transformation (Applied)

Beyond the base Factory refactor, **Template Method** was layered in per the architecture-transformation guidance: the base class's `connect()` now defines the fixed shape (log start → build connection string → log it → log success), while each subclass only supplies `build_connection_string()` and a `db_type_label`. This directly addressed the follow-up concern raised: no shared validation/construction shape, and no way to test string-building independently of the print side effects.

**Gradual approach used:** extract `build_connection_string()` first (behavior-preserving), then unify the print statements into the base class's `connect()` — exactly the order recommended to keep each step independently testable and revertable. The Strategy-based retry/pooling extension discussed as a *future* option was intentionally **not implemented**, since only one type (MongoDB) currently needs that behavior — introducing Strategy now would be premature abstraction, per the risk/mitigation discussion.

---

## Refactored Implementation

```python
from abc import ABC, abstractmethod


class DatabaseConnection(ABC):
    """
    Common base for all database connection types.
    Implements Template Method for connect(): the overall shape is fixed
    here, while each subclass only supplies build_connection_string().
    """

    def __init__(self, host, port, username, password, database,
                 use_ssl=False, connection_timeout=30):
        self.host = host
        self.port = port
        self.username = username
        self.password = password
        self.database = database
        self.use_ssl = use_ssl
        self.connection_timeout = connection_timeout
        self.connection = None

    @property
    @abstractmethod
    def db_type_label(self):
        raise NotImplementedError

    @abstractmethod
    def build_connection_string(self):
        raise NotImplementedError

    def connect(self):
        print(f"Connecting to {self.db_type_label} database...")
        connection_string = self.build_connection_string()
        if connection_string is not None:
            print(connection_string)
        print("Connection successful!")
        return self.connection


class MySQLConnection(DatabaseConnection):
    def __init__(self, *args, charset='utf8', **kwargs):
        super().__init__(*args, **kwargs)
        self.charset = charset

    @property
    def db_type_label(self):
        return "mysql"

    def build_connection_string(self):
        connection_string = (
            f"mysql://{self.username}:{self.password}@{self.host}:{self.port}/{self.database}"
            f"?charset={self.charset}&connectionTimeout={self.connection_timeout}"
        )
        if self.use_ssl:
            connection_string += "&useSSL=true"
        return f"MySQL Connection: {connection_string}"


class PostgreSQLConnection(DatabaseConnection):
    @property
    def db_type_label(self):
        return "postgresql"

    def build_connection_string(self):
        connection_string = f"postgresql://{self.username}:{self.password}@{self.host}:{self.port}/{self.database}"
        if self.use_ssl:
            connection_string += "?sslmode=require"
        return f"PostgreSQL Connection: {connection_string}"


class MongoDBConnection(DatabaseConnection):
    def __init__(self, *args, retry_attempts=3, pool_size=5, **kwargs):
        super().__init__(*args, **kwargs)
        self.retry_attempts = retry_attempts
        self.pool_size = pool_size

    @property
    def db_type_label(self):
        return "mongodb"

    def build_connection_string(self):
        connection_string = (
            f"mongodb://{self.username}:{self.password}@{self.host}:{self.port}/{self.database}"
            f"?retryAttempts={self.retry_attempts}&poolSize={self.pool_size}"
        )
        if self.use_ssl:
            connection_string += "&ssl=true"
        return f"MongoDB Connection: {connection_string}"


class RedisConnection(DatabaseConnection):
    @property
    def db_type_label(self):
        return "redis"

    def build_connection_string(self):
        return f"Redis Connection: {self.host}:{self.port}/{self.database}"


class DatabaseConnectionFactory:
    _connection_types = {
        'mysql': MySQLConnection,
        'postgresql': PostgreSQLConnection,
        'mongodb': MongoDBConnection,
        'redis': RedisConnection,
    }

    @classmethod
    def create(cls, db_type, **kwargs):
        connection_class = cls._connection_types.get(db_type)
        if connection_class is None:
            raise ValueError(f"Unsupported database type: {db_type}")
        return connection_class(**kwargs)
```

---

## Tests Verifying Behavior Preservation

Every test below actually ran against **both** the original and refactored implementations, capturing stdout and comparing directly — not just theorized.

```python
def capture_original(db_type, **kwargs):
    conn = OriginalDatabaseConnection(db_type=db_type, **kwargs)
    buf = io.StringIO()
    with contextlib.redirect_stdout(buf):
        conn.connect()
    return buf.getvalue()

def capture_refactored(db_type, **kwargs):
    conn = DatabaseConnectionFactory.create(db_type, **kwargs)
    buf = io.StringIO()
    with contextlib.redirect_stdout(buf):
        conn.connect()
    return buf.getvalue()

class TestBehaviorPreservation(unittest.TestCase):
    def test_mysql_with_ssl_matches(self):
        kwargs = dict(host='localhost', port=3306, username='db_user',
                      password='password123', database='app_db', use_ssl=True)
        self.assertEqual(capture_original('mysql', **kwargs),
                          capture_refactored('mysql', **kwargs))

    # ... (postgresql, mongodb, redis -- with and without SSL/custom params --
    #      all follow the identical pattern, plus:)

    def test_unsupported_type_raises_same_error(self):
        kwargs = dict(host='h', port=1, username='u', password='p', database='d')
        with self.assertRaises(ValueError) as orig_ctx:
            OriginalDatabaseConnection(db_type='oracle', **kwargs).connect()
        with self.assertRaises(ValueError) as new_ctx:
            DatabaseConnectionFactory.create('oracle', **kwargs)
        self.assertEqual(str(orig_ctx.exception), str(new_ctx.exception))
```

**New tests specific to the pattern itself** (capabilities the original code structurally couldn't support):

```python
class TestFactorySpecificBehavior(unittest.TestCase):
    def test_factory_returns_correct_concrete_type(self):
        # Confirms the factory returns the right subclass per db_type

    def test_mysql_specific_param_not_accepted_by_other_types(self):
        # charset is MySQL-only; passing it to RedisConnection raises
        # TypeError -- proves parameters are now correctly scoped per type,
        # unlike the original's one-size-fits-all constructor

    def test_all_connection_types_share_common_interface(self):
        # Every concrete type exposes a callable connect(), usable
        # polymorphically without the caller knowing the concrete type

    def test_build_connection_string_independently_testable(self):
        # New capability: test string construction WITHOUT triggering the
        # print/log side effects -- impossible to isolate in the original
        conn = MySQLConnection(host='localhost', port=3306, username='u',
                                password='p', database='d', use_ssl=True, charset='utf16')
        result = conn.build_connection_string()
        assert 'charset=utf16' in result
        assert 'useSSL=true' in result
```

### Test Run Result

```
Ran 12 tests in 0.002s

OK
```

All 12 tests pass: 8 behavior-preservation tests (every database type, with/without SSL, plus the error case for unsupported types) confirming byte-for-byte identical output to the original, and 4 new tests confirming the pattern's own guarantees (correct type dispatch, properly scoped parameters, shared interface, and independently testable string construction).

---

## Benefits Gained from the Pattern Implementation

1. **Object creation is now centralized and declarative** — `DatabaseConnectionFactory.create('mysql', ...)` reads as "give me a MySQL connection," rather than requiring the caller to understand which of 11 constructor parameters are relevant.
2. **Parameters are now correctly scoped per type** — proven directly by `test_mysql_specific_param_not_accepted_by_other_types`: passing `charset` to a `RedisConnection` now raises a `TypeError`, whereas the original silently accepted (and ignored) irrelevant parameters for any type.
3. **The `if/elif` chain is gone, replaced by polymorphism** — adding a new database type means adding one new class and one dictionary entry, not editing an ever-growing method.
4. **Connection-string construction is now independently testable** — `test_build_connection_string_independently_testable` verifies string content directly, without needing to capture or parse print output, a testing capability the original monolithic `connect()` didn't allow.
5. **A consistent, extensible shape (Template Method) governs all connection types** — the "log start → build string → log it → log success" flow lives in exactly one place (the base class), rather than being independently reimplemented (and potentially drifting) across four separate branches.
6. **Deliberately avoided premature abstraction** — the Strategy pattern discussed for cross-cutting retry/pooling concerns was consciously not implemented yet, since only one database type currently needs it; documented as a future option rather than built preemptively.

