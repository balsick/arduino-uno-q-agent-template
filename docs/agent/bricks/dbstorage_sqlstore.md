# `dbstorage_sqlstore` brick

Thread-safe SQLite storage with a simple `dict`-based API. Source: `https://github.com/arduino/app-bricks-py/tree/main/src/arduino/app_bricks/dbstorage_sqlstore`.

## Install

User must add the **Database - SQL** brick from the App Lab GUI.

The DB file lives under `/app/data/<name>.db` on the board (auto-created). It persists across app restarts.

## Python API

```python
from arduino.app_bricks.dbstorage_sqlstore import SQLStore

db = SQLStore("scores.db")   # opens (or creates) /app/data/scores.db
```

| Method | Signature | Purpose |
| --- | --- | --- |
| `start` | `() -> None` | Open the connection. Called lazily by every other method too. |
| `stop` | `() -> None` | Close the connection. |
| `create_table` | `(table: str, columns: dict[str, str])` | Explicit `CREATE TABLE IF NOT EXISTS`. Column types are raw SQL (`"INTEGER"`, `"REAL"`, `"TEXT"`, `"BLOB"`, `"INTEGER PRIMARY KEY"`, …). |
| `store` | `(table: str, data: dict[str, Any], create_table: bool = True)` | Insert one row. Auto-creates the table by inferring types from `data` (`int→INTEGER`, `float→REAL`, `str→TEXT`, `bytes→BLOB`). |
| `read` | `(table, columns=None, condition=None, order_by=None, limit=-1) -> list[dict]` | `SELECT`. Returns `[]` if the table doesn't exist yet. |
| `update` | `(table, data: dict, condition: str = "")` | `UPDATE … SET col=? …`. Empty `condition` updates all rows. |
| `delete` | `(table, condition: str = "")` | `DELETE FROM …`. Empty `condition` deletes everything. |
| `execute_sql` | `(sql: str, args: tuple \| None = None) -> list[dict] \| None` | Raw SQL. Returns rows for queries, `None` for DML. |
| `create_or_replace_table` | `(table, columns, force_drop_table=False)` | Best-effort schema migration: adds/removes columns to match `columns`. If a non-trivial change is needed (type change, indexed column), set `force_drop_table=True` to drop & recreate (destroys data). |

`store/read/update/delete` raise `DBStorageSQLStoreError` on failure. Rows come back as plain `dict[str, Any]` (named access).

## Minimal example

```python
from arduino.app_utils import App
from arduino.app_bricks.dbstorage_sqlstore import SQLStore

db = SQLStore("scores.db")
db.create_table("scores", {
    "id":     "INTEGER PRIMARY KEY",
    "name":   "TEXT",
    "score":  "INTEGER",
    "ts":     "TEXT",
})

# Insert
db.store("scores", {"name": "ALICE", "score": 1240, "ts": "2026-05-13T10:00:00Z"})

# Read top 5
top = db.read("scores", order_by="score DESC", limit=5)
for row in top:
    print(row["name"], row["score"])

# Update
db.update("scores", {"score": 1500}, condition="name = 'ALICE'")

# Raw SQL with params
db.execute_sql("DELETE FROM scores WHERE score < ?", (100,))

App.run()
```

## Gotchas

- DB path is fixed to `/app/data/<name>.db`. On the host you can `adb pull /app/data/<name>.db` to inspect with `sqlite3`.
- `condition` and `order_by` are **interpolated as raw SQL** (no parametrization). Never feed user input directly — use `execute_sql(sql, args)` with `?` placeholders, or sanitize aggressively.
- `store()` builds the schema from the **first** row it sees. If a later row contains a new field, you'll get a SQL error — use `create_table` up front when the shape matters.
- `read()` returns `[]` when the table doesn't exist (no exception), but raises on every other SQLite error.
- The brick is thread-safe (one `RLock` per `SQLStore`), so it's fine to call from `RemoteSensor` / `WebUI` callbacks running on different threads.
- Schema changes via `create_or_replace_table` can't change a column's type in-place; you need `force_drop_table=True` (data loss) for that.
