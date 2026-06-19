---
name: eddytor-python-sdk
description: >-
  Best practices for the Eddytor Python SDK (`eddytor-sdk` on PyPI) — connecting,
  querying to pandas/Arrow, ingesting data, and using table handles. Activates for
  the Python SDK, eddytor-sdk, EddytorClient, query to a DataFrame, pandas/pyarrow
  integration, or scripting Eddytor from Python — even without "SDK."
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Eddytor Python SDK best practices

`eddytor-sdk` is the Python client: SQL → pandas/Arrow over Flight SQL, plus table-handle
methods for DML, versioning, and maintenance. Reads stream over Flight SQL; writes/control
go over gRPC + REST under the hood — the SDK hides the wiring.

## Install

```bash
pip install eddytor-sdk            # requires Python >= 3.10
pip install eddytor-sdk[auth]      # extra: interactive device-code login
```

## Connect

```python
from eddytor_sdk import EddytorClient

# API key (headless / servers / CI) — edd_live_… or an OAuth access token
with EddytorClient(api_key="edd_live_xxx", url="http://localhost:8080") as client:
    df = client.query('SELECT * FROM "eddytor"."cfg_…"."…_customers" LIMIT 10')

# Interactive OAuth 2.1 device-code login (humans)
client = EddytorClient.login("http://localhost:8080")
```

URL resolves `url=` arg → `EDDYTOR_API_URL` env → default. Use the **context manager**
(`with …`) so the connection closes cleanly; otherwise call `client.close()`.

## Read

```python
df   = client.query(sql)         # -> pandas.DataFrame
tbl  = client.query_arrow(sql)   # -> pyarrow.Table (no pandas conversion cost)
rows = client.execute(sql)       # -> list[tuple]
```

Prefer **table handles** from discovery — they carry the fully-qualified name so you don't
hand-build identifiers:

```python
tables = client.tables()                       # discover -> list[Table]
customers = client.table("eddytor", "cfg_…", "…_customers")   # or by parts
df = customers.query_all(limit=10)             # also query_all_arrow(), count(where=…)
meta, hist = customers.metadata(), customers.history()
```

## Write

```python
# bulk ingest a pandas DataFrame or pyarrow Table
client.ingest("customers", df, mode="append")   # modes: create | append | replace | create_append

# on a table handle
customers.insert(df)            # append rows
customers.merge(df)             # upsert by primary key
customers.delete(df)            # delete by PK
customers.execute_dml(sql)      # -> affected row count
```

## Versioning, maintenance, lifecycle (table handle)

```python
customers.rollback(version); customers.restore("2026-06-01T00:00:00Z")
customers.diff(10, 12); customers.optimize(); customers.vacuum(dry_run=True)
customers.add_constraints([...]); customers.move(cfg_id, "europe/customers"); customers.drop()
```

## Errors

```python
from eddytor_sdk import (
    EddytorError,            # base
    EddytorConnectionError,  # transport / can't reach server
    EddytorQueryError,       # SQL / guardrail failure
    EddytorAuthError,        # bad/expired credential
    EddytorSchemaError,      # schema mismatch on ingest
)
```

Catch `EddytorError` for a blanket handler; branch on the subclasses when you act
differently (e.g. refresh on `EddytorAuthError`, retry on `EddytorConnectionError`).

## Gotchas

* **Reads are Flight SQL** (default proxy port 8082) — make sure that port is reachable, not
  just the REST URL. See eddytor-flight-sql for endpoint/auth details.
* **API key vs OAuth:** `EddytorClient(api_key=…)` for headless; `.login()` for humans. An
  OAuth access token is short-lived (~15 min) — prefer an `edd_live_…` key for long jobs.
* **Permissions apply:** the SDK sees only what the credential's role (∩ key scope) allows.
* **`query_arrow` avoids the pandas round-trip** — use it for large results or when feeding
  Arrow-native tools; `query` is the convenience path.
* **`ingest` mode matters:** `append` adds, `replace` overwrites, `create`/`create_append`
  create the table first — picking the wrong one can drop or duplicate data.
* Reach `client.rest` / `client.grpc` only for operations the high-level API doesn't cover.

## Guidelines

* Use a `with EddytorClient(...)` block and let it close the connection.
* Discover with `client.tables()` / `client.table(...)` and use the handle's methods rather
  than concatenating SQL identifiers.
* Inject the API key from the environment; never hardcode `edd_live_…` in source.
* `query_arrow` for scale, `query` for convenience; reserve raw `.rest`/`.grpc` for gaps.
