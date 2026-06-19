---
name: eddytor-flight-sql
description: >-
  Best practices for high-performance bulk reads from Eddytor over Apache Arrow
  Flight SQL. Activates for Flight SQL, ADBC, Arrow, bulk export, large result
  sets, BI tools, connecting Tableau/Power BI/pandas, or "the query is too big for
  REST" — even without the phrase "Flight SQL."
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Flight SQL best practices

Flight SQL is Eddytor's **high-throughput read path** — columnar Arrow batches over gRPC,
for large result sets, exports, and BI/ADBC clients. It is **read-only**: writes go
through the gRPC DML path / CLI / SDK (see the gotchas).

## Endpoints

* **Self-host (Docker Compose):** `http://localhost:8082` — the server proxies Flight SQL
  to the engine.
* **Kubernetes / behind the server:** the server exposes Flight on port **8082** (proxying
  the engine's **50052**); reach it at your server host on 8082 (or via a port-forward:
  `kubectl -n eddytor port-forward svc/eddytor-server 8082:8082`).
* **Eddytor Cloud:** `https://flight.eddytor.com:443`.

The engine's 50052 is internal — always go through the server's 8082 proxy, which enforces
auth.

## Connecting

Authenticate with the **same credentials as the REST API**: an OAuth bearer access token or
an `edd_live_…` API key, sent as the Flight `authorization: Bearer <token>` header. Any
ADBC Flight SQL driver works (Python, Go, JDBC, BI tools).

```python
# pip install adbc-driver-flightsql pyarrow
from adbc_driver_flightsql import dbapi, DatabaseOptions

conn = dbapi.connect(
    "grpc+tls://flight.eddytor.com:443",   # self-host: grpc://localhost:8082
    db_kwargs={DatabaseOptions.AUTHORIZATION_HEADER.value: "Bearer edd_live_…"},
)
tbl = conn.cursor().execute(
    'SELECT * FROM "eddytor"."cfg_…"."…_customers" WHERE status = \'active\''
).fetch_arrow_table()
```

The simplest path is often the CLI, which wraps Flight SQL: `eddytor query "SELECT …" --limit N`
(set `eddytor config set-flight-url` first). The Python SDK (`eddytor-sdk`) also reads via
Flight and returns `pyarrow.Table`.

## When to use Flight SQL vs REST vs MCP

* **Flight SQL** — bulk/analytical reads, full-table exports, feeding pandas/BI. Streams
  Arrow, no JSON-serialization overhead.
* **REST** (`/api/v1`) — control plane and small reads from apps (see eddytor-rest-api).
* **MCP** — LLM/agent access (the `query_rows` / `execute_sql` tools).

## Gotchas

* **Read-only.** Flight SQL does not write — table DML (insert/update/merge/delete) is the
  gRPC path. Use the CLI (`insert`/`merge`), the SDK, or MCP write tools instead.
* **Go through 8082, not 50052.** The engine's 50052 is internal and must **never** be
  exposed to the network — the server's 8082 proxy is where the bearer token is validated.
  Keep the engine on a ClusterIP / private network behind the server.
* **Same auth + permissions as REST** — a token/key only sees what its role (∩ key scope)
  allows; expired access tokens fail the handshake (re-auth or use an API key).
* **Quote SQL identifiers** per component: `"catalog"."schema"."table"` (double quotes), and
  fully qualify them — get exact names from `list_tables` / `eddytor get tables`.
* TLS: Cloud is `grpc+tls://…:443`; a plaintext self-host proxy is `grpc://…:8082`. Match
  the scheme to how the edge is terminated.

## Guidelines

* Reach for Flight SQL once result sets are large enough that REST JSON pagination hurts.
* Prefer the CLI or `eddytor-sdk` over hand-rolling an ADBC connection unless you need a
  specific BI/driver integration.
* Keep credentials out of code — inject the bearer token/API key from the environment.
