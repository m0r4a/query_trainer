# Tempo Query Patterns

This document is a reference for the TraceQL exercises in the trainer dashboard. It covers span selection, attribute filtering, duration and status conditions, structural operators for parent-child relationships, aggregation functions, and the practical patterns for correlating traces with Prometheus metrics and Loki logs.

---

## How Tempo Differs from Prometheus and Loki

Tempo stores distributed traces. A trace is a tree of spans, where each span represents a unit of work in the system: handling an HTTP request, publishing a message, executing a database query. Spans carry attributes (key-value metadata), a status (OK, ERROR, UNSET), a duration, and structural relationships (parent-child).

TraceQL is a query language for selecting spans and computing aggregates over them. It is not a time series language. There are no `rate()` or `sum by ()` functions in the same sense as PromQL. Instead, TraceQL has span pipeline operators that filter and transform spans, and metric functions that aggregate across matched spans.

The key mental shift from Prometheus and Loki: **you are querying individual events, not aggregated series.** Each result is a span or a trace, not a time series data point.

---

## Span Selector Basics

Every TraceQL query starts with a span selector `{}`. An empty selector matches every span in the search window.

**All spans:**

```traceql
{}
```

**Spans from App A:**

```traceql
{resource.service.name="app-a"}
```

**Spans from App B:**

```traceql
{resource.service.name="app-b"}
```

**Spans with error status:**

```traceql
{status=error}
```

**Spans with a specific span name:**

```traceql
{name="POST /api/messages"}
```

**Spans with a specific HTTP route attribute:**

```traceql
{span.http.route="/api/messages"}
```

Attributes in TraceQL are accessed with a prefix:

- `resource.` prefix accesses resource-level attributes (set once per SDK instance, like `service.name`, `service.version`)
- `span.` prefix accesses span-level attributes (set per span, like `http.method`, `db.operation`)
- `status`, `name`, `duration` are intrinsic span fields with no prefix

---

## Attribute Filtering

**HTTP 5xx errors in App A:**

```traceql
{resource.service.name="app-a", span.http.status_code>=500}
```

**NATS publish spans:**

```traceql
{span.messaging.operation="publish", span.messaging.system="nats"}
```

**Database INSERT spans:**

```traceql
{span.db.operation="INSERT", span.db.system="postgresql"}
```

**Spans with a specific error type:**

```traceql
{span.exception.type="context deadline exceeded"}
```

**Spans with any exception recorded:**

```traceql
{span.exception.message != nil}
```

`nil` checks are how you test for attribute presence in TraceQL. `!= nil` means the attribute exists and has a value.

**Spans from a specific service version:**

```traceql
{resource.service.version="0.2.0"}
```

---

## Duration Conditions

Duration conditions filter spans by how long they took. Duration is expressed as a literal with a unit suffix: `ms` for milliseconds, `s` for seconds, `m` for minutes.

**Spans slower than 500ms:**

```traceql
{duration > 500ms}
```

**Spans slower than 500ms in the App B processing pipeline:**

```traceql
{resource.service.name="app-b", duration > 500ms}
```

**Database spans slower than 100ms:**

```traceql
{span.db.system="postgresql", duration > 100ms}
```

**Spans between 100ms and 1s:**

```traceql
{duration > 100ms && duration < 1s}
```

**The slowest spans in App A (rate-based, using metrics functions):**

```traceql
{resource.service.name="app-a"} | histogram_over_time(duration)
```

---

## Status Conditions

Span status follows the OTel specification: `ok`, `error`, or `unset`. In this project, any span that catches an unhandled error sets its status to `error` and records an exception event.

**All error spans:**

```traceql
{status=error}
```

**Error spans from App B only:**

```traceql
{resource.service.name="app-b", status=error}
```

**Error spans in database operations:**

```traceql
{span.db.system="postgresql", status=error}
```

**Error spans with their exception messages:**

```traceql
{status=error, span.exception.message != nil}
```

---

## Structural Operators

Structural operators express relationships between spans in a trace tree. This is the feature that makes TraceQL distinctly more powerful than just filtering spans individually.

### Child (`>`)

Matches a parent span directly followed by a child span. Used to find traces where a specific type of operation caused a specific downstream effect.

**HTTP handler spans that have an error in a direct child database span:**

```traceql
{name="POST /api/messages"} > {span.db.system="postgresql", status=error}
```

This selects entire traces where a `POST /api/messages` span is the direct parent of a failing database span. You are not just finding database errors in isolation; you are finding the upstream request that triggered them.

**NATS consume spans that have a slow child database span:**

```traceql
{span.messaging.operation="receive"} > {span.db.system="postgresql", duration > 200ms}
```

### Descendant (`>>`)

Matches any ancestor followed by any descendant at any depth in the span tree, not just direct parent-child.

**HTTP handler spans where any downstream span errored:**

```traceql
{name="POST /api/messages"} >> {status=error}
```

**Traces where App A published a message that eventually caused an error in App B:**

```traceql
{resource.service.name="app-a", span.messaging.operation="publish"}
>> {resource.service.name="app-b", status=error}
```

This is the trace-level expression of "a publish from App A caused a downstream failure in App B." Without structural operators you would have to find these traces manually by looking up trace IDs.

### Sibling (`&&` within a trace)

In TraceQL, `&&` within a single span selector means both conditions apply to the same span. To express "two spans that are siblings," you use `{condition1} && {condition2}` at the trace level with the `&&` operator between span groups.

---

## Aggregation and Metrics

TraceQL supports aggregate functions that compute statistics over matched spans. These are not stored time series; they are computed on demand over the traces that match your search window.

**Count of error spans by service:**

```traceql
{status=error} | count() by (resource.service.name)
```

**p99 duration of HTTP handler spans:**

```traceql
{name="POST /api/messages"} | quantile_over_time(duration, 0.99)
```

**Rate of error spans over time (Tempo metrics generator required):**

```traceql
{status=error} | rate()
```

The metrics generator is a Tempo component that pre-aggregates trace data into Prometheus-compatible metrics. When enabled, it produces metrics like `traces_spanmetrics_calls_total` and `traces_spanmetrics_duration_seconds` which can be queried in Prometheus. This closes the loop between trace data and the metrics trainer section.

**Histogram of database query durations:**

```traceql
{span.db.system="postgresql"} | histogram_over_time(duration)
```

---

## Correlation with Loki

In Grafana, Tempo and Loki are linked through the `trace_id` field. The workflow:

**From Tempo to Loki:** When viewing a trace in the Tempo panel, each span that has associated logs shows a "Logs" link. Clicking it opens a Loki query pre-filled with the trace ID. The Loki datasource must be configured with a derived field rule that matches `trace_id` in log line content.

**From Loki to Tempo:** When viewing log lines in the Loki panel, any line containing a field matching the derived field rule for `trace_id` renders that value as a link. Clicking it opens the trace in Tempo.

The prerequisite for this to work:
1. The app must inject the W3C trace context into every log line (both apps in this project do this via the OTel SDK)
2. The Loki datasource in Grafana must have a derived field configured: field name `trace_id`, regex `"trace_id":"(\w+)"`, internal link pointing to the Tempo datasource

This correlation is the most important concept in the Tempo section of the trainer. A trace tells you what happened structurally and how long each step took. The correlated logs tell you what the application was thinking at each step. Together they are substantially more useful than either alone.

---

## Correlation with Prometheus

If the Tempo metrics generator is enabled and writing span metrics to Prometheus, you can create Grafana panels that link from a metric anomaly directly to a trace search.

The pattern is:

1. Prometheus panel shows p99 latency spike at 14:23
2. You navigate to Tempo Explore with a time range around 14:23
3. Query: `{resource.service.name="app-a", duration > 500ms}` scoped to that time window
4. Inspect the returned traces to find the common factor (a specific route, a specific downstream call)

This workflow is called exemplar-based correlation when Prometheus exemplars are enabled. An exemplar is a sample data point that carries a trace ID alongside its metric value. Prometheus can store exemplars emitted by the OTel SDK, and Grafana renders them as dots on the time series panel. Clicking a dot opens the associated trace in Tempo directly.

For exemplars to work, the OTel SDK must be configured to emit exemplars, and Prometheus must have `--enable-feature=exemplar-storage` set.

---

## Available Span Attributes Reference

**Resource attributes (resource.)** — set once per service instance:

| Attribute | Example Value |
|---|---|
| `resource.service.name` | `app-a`, `app-b` |
| `resource.service.version` | `0.1.0` |
| `resource.deployment.environment` | `local` |

**Span attributes for App A HTTP spans (span.):**

| Attribute | Example Value |
|---|---|
| `span.http.method` | `POST` |
| `span.http.route` | `/api/messages` |
| `span.http.status_code` | `200`, `500` |
| `span.http.request_content_length` | `128` |

**Span attributes for App A NATS publish spans (span.):**

| Attribute | Example Value |
|---|---|
| `span.messaging.system` | `nats` |
| `span.messaging.destination` | `events.created` |
| `span.messaging.operation` | `publish` |

**Span attributes for App B NATS consume spans (span.):**

| Attribute | Example Value |
|---|---|
| `span.messaging.system` | `nats` |
| `span.messaging.destination` | `events.created` |
| `span.messaging.operation` | `receive` |
| `span.messaging.message.id` | NATS message sequence |

**Span attributes for App B database spans (span.):**

| Attribute | Example Value |
|---|---|
| `span.db.system` | `postgresql` |
| `span.db.operation` | `INSERT` |
| `span.db.sql.table` | `messages` |
| `span.db.statement` | sanitized SQL string |

**Error span event attributes:**

| Attribute | Notes |
|---|---|
| `span.exception.type` | Go error type or interface name |
| `span.exception.message` | Error string |
| `span.exception.stacktrace` | Goroutine stack at error site |
