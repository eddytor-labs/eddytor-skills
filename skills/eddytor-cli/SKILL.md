---
name: eddytor-cli
description: >-
  Best practices for the `eddytor` command-line interface — auth, configuration,
  the verb-first command grammar, JSON output for scripting/CI, and bulk queries.
  Activates for the eddytor CLI, terminal usage, scripting Eddytor, CI pipelines,
  `eddytor login`, `eddytor get`, or `--json` — even without the word "CLI."
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Eddytor CLI best practices

`eddytor` is the terminal/scripting interface. It talks to the REST API for control
operations and to Flight SQL for bulk reads. Commands follow a **verb-first grammar**:
`eddytor <verb> <resource> [args] [--json]`.

## Setup & auth

```bash
eddytor config set-api-url https://eddytor.example.com   # your server (default https://app.eddytor.com)
eddytor login                                            # OAuth 2.1 device-code flow (browser)
eddytor doctor                                           # verify connectivity, auth, and Flight SQL
```

* **Interactive / human:** `eddytor login` (device code), `eddytor logout` clears stored creds.
* **Headless / CI:** use an API key instead of `login` — `eddytor config set-key edd_live_…`
  (mint with `eddytoradm create-api-key`, see eddytor-access-control). Or pass it via the
  config file the CLI writes.
* **Bulk reads** need the Flight URL too: `eddytor config set-flight-url <url>` (self-host
  Docker Compose: `http://localhost:8082`). `eddytor config show` prints the resolved config.

## The verb-first grammar

```bash
eddytor get tables                         # list (singular alias: `get table`)
eddytor describe table <catalog> <schema> <name>
eddytor create table ...                   # create/constraint/storage/api-key
eddytor set domain <catalog> <schema> <table> <column> fixed -v a,b,c
eddytor insert rows ... / eddytor merge rows ...   # DML from a file
eddytor query "SELECT * FROM \`cat\`.\`sch\`.\`tbl\` WHERE status = 'active'" --limit 100
```

* Table-scoped commands take a **positional triple** `catalog schema table` (and domain
  commands a fourth `column`) — not a single dotted string.
* Verbs: `get` `describe` `create` `delete` `set` `link`/`unlink` `insert` `merge`
  `optimize` `vacuum` `rollback` `restore` `diff` `rename` `move` `profile` `validate`
  `upload` `download` `infer-schema` `query` `install`.
* `get` resources include: tables, history, domain, domain-values, storage, objects,
  api-keys, audit-log, org, users, sso, providers, skills.

## Scripting & CI

* Add `--json` (global flag) for machine-readable output — pipe to `jq`. It's accepted
  everywhere but a no-op on a few side-effecting commands (login/logout/config/insert/
  merge/link/install), which warn `--json has no effect`.
* Use an **API key** (`config set-key`), not `login`, in non-interactive environments.
* Check the **exit code** — non-zero means failure; errors print to stderr.
* `eddytor doctor` is the first triage step when auth or Flight isn't working.

## Gotchas

* A control command failing with auth errors usually means a stale/short-lived token —
  re-`login`, or set an API key for long-running jobs.
* `query` needs `flight_url` set and reachable; `eddytor doctor` checks it specifically.
* Row DML (`insert`/`merge`) goes over gRPC under the hood — guardrail violations come
  back as structured write errors (unique/PK → conflict, domain/check/not-null →
  unprocessable), not generic failures.
* `--insecure` skips TLS verification — only for local/self-signed dev, never production.

## Guidelines

* Human at a terminal → `login`. Pipeline/CI → API key + `--json`.
* Use `eddytor query` (Flight SQL) for large result sets; control-plane verbs go over REST.
* Fully qualify tables as `catalog schema table` positionals; get exact names from `get tables`.
