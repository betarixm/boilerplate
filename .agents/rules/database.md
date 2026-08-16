# Database Rules

## Scope

These rules apply to web applications that own a relational database.

## Engine Selection

Use SQLite for local development and automated tests.

Use PostgreSQL for production.

Use PostgreSQL for staging and any other environment that must reproduce production behavior.

SQLite keeps local setup free of external services. PostgreSQL provides the concurrency, types, and operational behavior production requires.

## Configuration

Select the engine through configuration, not through code branches.

Read the connection string from a single environment variable, conventionally `DATABASE_URL`.

```text
DATABASE_URL=sqlite:///./dev.db          # development
DATABASE_URL=postgresql://.../app        # production
```

Do not hardcode a connection string, and do not commit production credentials.

Provide a working development default so a fresh checkout runs without additional setup.

Do not commit the development database file. Ignore it in `.gitignore`.

## Portability

Write schema and queries that run on both engines.

Use a query layer or ORM that targets both, and manage schema changes with migrations that are applied to both.

Avoid engine-specific features unless they are isolated behind an interface with an implementation for each engine.

Account for the differences that break portability most often:

* SQLite has dynamic typing and no native `BOOLEAN`, `UUID`, `JSONB`, or array types
* SQLite `LIKE` is case-insensitive for ASCII by default; PostgreSQL `LIKE` is case-sensitive
* SQLite serializes writes and enforces foreign keys only when `PRAGMA foreign_keys = ON`
* PostgreSQL enforces stricter type coercion, `GROUP BY` rules, and transaction isolation
* Sequences, `RETURNING`, upsert syntax, and full-text search differ between the engines

Do not rely on behavior that differs between the engines when correctness depends on it.

## Verification

Run the test suite against the development engine for fast local feedback.

Run migrations and the test suite against PostgreSQL in CI before any change reaches production.

A change is not complete until it passes against PostgreSQL.

## Existing Projects

Follow the engine and tooling already established by the project.

Do not migrate an existing project to this arrangement solely to conform to these preferences.
