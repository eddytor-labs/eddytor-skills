---
name: eddytor-observability
description: >-
  Enable and reason about Eddytor's telemetry — OpenTelemetry traces and metrics
  pushed over OTLP to your collector, plus structured logs. Activates for
  telemetry, OpenTelemetry, OTLP, metrics, tracing, logs, log level, monitoring,
  or wiring a collector. Push-only: there is no /metrics scrape endpoint.
license: CC-BY-NC-4.0
metadata:
  author: eddytor
  version: "1.0"
---

# Eddytor Observability (OpenTelemetry + logs)

Every Eddytor binary initializes tracing at boot via `init_tracing(service_name)`.
Telemetry has three parts: **logs** (always on, to stdout), **OTLP traces**, and
**OTLP metrics**. Traces and metrics are **opt-in and push-only** — the binaries
dial *out* to your collector. There is **no `/metrics` endpoint** and nothing for
Prometheus or a browser to scrape inbound.

## Default procedure

1. **Pick a collector** that accepts OTLP over **HTTP** (e.g. an OpenTelemetry
   Collector with the `otlp` receiver). Note its HTTP endpoint URL.
2. **Set the endpoint** on the server and engine, via either the env var (wins)
   or config — see examples. Both traces *and* metrics turn on together from this
   single endpoint; they share it.
3. **Set the log level** if the default is too noisy/quiet — `telemetry.log_level`
   in config, or `RUST_LOG` to override at runtime.
4. **Verify** the collector is receiving spans/metrics after some requests. If no
   endpoint is set, no OTel layer is registered at all (zero overhead) — confirm
   the var actually reached the process.

## Tool usage examples

### Enable OTLP (env var — takes precedence)

```bash
# Server and engine both export when this is set and non-empty
export OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4318"
```

Empty / whitespace-only counts as unset. With nothing set, OTel is fully off.

### Enable OTLP (config)

```toml
# eddytor.toml
[telemetry]
otlp_endpoint = "http://otel-collector:4318"
log_level = "info,sqlx=warn"
```

Equivalent env override for the config key:

```bash
export EDDYTOR__TELEMETRY__OTLP_ENDPOINT="http://otel-collector:4318"
```

### Set the log level

```bash
# RUST_LOG overrides telemetry.log_level at runtime; per-target filters allowed
export RUST_LOG="debug,sqlx=warn"
```

```toml
# or in config (lower precedence than RUST_LOG)
[telemetry]
log_level = "info"
```

The built-in default filter (when neither is set) is roughly `info` with a few
noisy targets quieted:
`info,datafusion_datasource=error,delta_kernel=warn,poem_mcpserver=warn,sqlx::postgres::notice=warn`.

## Gotchas

* **There is no `/metrics` scrape endpoint.** Do not point Prometheus at the
  binaries or tell anyone to curl `/metrics` — it does not exist. Export is
  push-only OTLP; if you want Prometheus, scrape your *collector*, not Eddytor.
* **One endpoint drives both signals.** Setting `OTEL_EXPORTER_OTLP_ENDPOINT` (or
  `telemetry.otlp_endpoint`) turns on **traces and metrics together**. You cannot
  enable one without the other; metrics reuse the trace endpoint.
* **Env wins over config.** `OTEL_EXPORTER_OTLP_ENDPOINT` beats
  `telemetry.otlp_endpoint`; likewise `RUST_LOG` beats `telemetry.log_level`.
* **Exporter is HTTP/OTLP** — give a collector that speaks OTLP/HTTP (the `:4318`
  family), not a gRPC-only endpoint.
* **Log format switches on build.** JSON when the binary is built with the
  `kubernetes` feature, human-readable text otherwise. You change the format by
  choosing the image edition, not at runtime.
* **Set the endpoint on every binary you want observed.** Server and engine are
  separate processes; each reads its own env/config.
* **A bad endpoint fails open, quietly.** If the exporter can't build, the binary
  logs to stderr and keeps running without that signal — check the collector
  actually received data rather than assuming success.

## Guidelines

* **Metrics emitted:** per-request count (`http.server.request.count`) and
  duration (`http.server.request.duration`, seconds), recorded by the
  `RequestMetrics` middleware on the outer route chains of **both server and
  engine**, so they cover **REST and gRPC**.
* **Labels are deliberately low-cardinality:** `http.route` (the matched route
  *pattern*, e.g. `/api/tables/:id` — never the raw URI with a UUID),
  `http.request.method`, `http.response.status_code`, and `rpc.grpc.status_code`
  (set when a `grpc-status` response header is present, since gRPC errors surface
  there, not in the HTTP status).
* **Distributed traces stitch across services:** W3C TraceContext propagates
  server → engine over gRPC metadata, so a request's spans link end-to-end.
* **Graceful shutdown flushes telemetry.** On SIGTERM / ctrl-c the binaries call
  `shutdown_telemetry()` to flush buffered spans and the final metrics interval —
  give pods a real termination grace period so the last batch isn't dropped.
* **Leave it off if you don't run a collector.** No endpoint = no OTel layer and
  no overhead; logs to stdout still work on their own.
