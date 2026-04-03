# Loki Query Patterns

This document is a reference for the LogQL exercises in the trainer dashboard. It covers log stream selectors, filter expressions, parser expressions, metric queries derived from logs, and trace correlation.

To provide a realistic parsing challenge, **App A** emits structured JSON logs, while **App B** emits plain text logs separated by pipes (`|`).

---

## How Loki Differs from Prometheus

Loki is not a metrics database. It stores log streams indexed by a small set of labels. The labels attached to a log stream are defined at collection time by the OTel Collector configuration and are kept intentionally small because each unique label combination creates a separate stream. High-cardinality fields (like `trace_id` or `request_id`) are stored in the log line body and parsed at query time, not as stream labels.

This distinction matters for query performance. Filtering on stream labels (the `{...}` selector) is fast because it uses the index. Filtering on parsed fields (using `| json` or `| pattern` followed by field conditions) is slower because it requires scanning log line content. Always narrow the stream with labels first, then parse and filter.

---

## Log Stream Labels

The following labels are available for stream selection in this project. These are the only fields you can use in the `{...}` selector.

| Label | Values |
|---|---|
| `service` | `app-a`, `app-b` |
| `level` | `debug`, `info`, `warn`, `error` |
| `environment` | `local` |
| `version` | semver string, e.g. `0.1.0` |

All other fields are in the log line body and must be extracted with a parser.

---

## Log Line Schema

### App A (JSON Format)
App A writes structured JSON logs. The following fields are present:

| Field | Type | Notes |
|---|---|---|
| `timestamp` | string | RFC 3339 with millisecond precision |
| `level` | string | `debug`, `info`, `warn`, `error` |
| `service` | string | `app-a` |
| `version` | string | Application version |
| `environment` | string | `local` |
| `trace_id` | string | W3C TraceContext trace ID (32 hex chars) |
| `span_id` | string | W3C TraceContext span ID (16 hex chars) |
| `request_id` | string | ULID, unique per HTTP request |
| `message` | string | Human-readable event description |
| `error` | string/null | Error message if applicable |
| `http.method` | string | HTTP handlers only |
| `http.route` | string | HTTP handlers only |
| `http.status_code` | number | HTTP handlers only |
| `duration_ms` | number | HTTP handlers only |

### App B (Pipe-Separated Format)

App B writes plain text logs following a strict positional structure, often ending with key-value pairs for extra context:
`Timestamp | Level | Service | Version | Environment | TraceID | SpanID | RequestID | Message | ExtraContext`

Example:
`2024-01-15T10:30:00Z | INFO | app-b | 0.1.0 | local | 4bf92f3577b3... | 08a3c2b1... | 01HQ4V2... | message processed | db.operation=INSERT db.table=public.messages`

---

## Stream Selection

The `{...}` block is mandatory in every LogQL query. It selects which log streams to include.

**All logs from App A:**

```logql
{service="app-a"}
```

**All error logs from App B:**

```logql
{service="app-b", level="error"}
```

**All logs from both services:**

```logql
{service=~"app-a|app-b"}
```

**All error and warning logs across all services:**

```logql
{level=~"warn|error"}
```

---

## Filter Expressions

Filter expressions narrow results within a selected stream by matching against the raw log line content. They apply before any parsing.

**Lines containing the word "error":**

```logql
{service=~"app-a|app-b"} |= "error"
```

**Lines not containing "healthcheck" (App A):**

```logql
{service="app-a"} != "healthcheck"
```

**Lines matching a regex pattern:**

```logql
{service="app-a"} |~ "status_code.*5[0-9]{2}"
```

**Filters can be chained:**

```logql
{service="app-a"}
|= "error"
!= "connection reset by peer"
```

---

## Parser Expressions

Parsers extract structured fields from the log line body and make them available as labels for filtering and formatting.

### 1. The JSON Parser (App A)

For App A, the `| json` operator extracts all top-level JSON keys automatically. Note that dots in field names are replaced with underscores (`http.method` becomes `http_method`).

**Parse all JSON fields:**

```logql
{service="app-a"} | json
```

**Parse and filter on a specific field:**

```logql
{service="app-a"} | json | http_status_code >= 500
```

**Line format: reshape the output for readability:**

```logql
{service="app-a"} | json | line_format "{{.level}} {{.http_route}} {{.http_status_code}} {{.duration_ms}}ms"
```

**Label format: rename an extracted field:**

```logql
{service="app-a"} | json | label_format status=http_status_code
```

### 2. The Pattern Parser (App B)

For App B, define the structure using `| pattern`. Use `<_>` to discard fields and `<name>` to extract fields as labels.

**Extract the Trace ID and Message:**

```logql
{service="app-b"} | pattern `<_> | <_> | <_> | <_> | <_> | <trace_id> | <_> | <_> | <message> | <_>`
```

**Chaining Parsers (Pattern + Logfmt):**

App B includes key-value pairs at the end. You can extract the end of the string with `pattern`, then parse the pairs with `logfmt`.

```logql
{service="app-b"}
  | pattern `<_> | <_> | <_> | <_> | <_> | <_> | <_> | <_> | <_> | <extra>`
  | logfmt
  | db_operation == "INSERT"
```

---

## Trace Correlation via trace_id

The `trace_id` field in every log line is the W3C TraceContext trace ID, enabling bidirectional correlation with Tempo.

**Find all log lines for a specific trace across both apps:**

```logql
{service=~"app-a|app-b"} |~ "4bf92f3577b34da6a3ce929d0e0e4736"
```

*(Using `|~` is faster than parsing both JSON and Pipe formats when just searching for a specific ID).*

**Format both formats into a single, unified view:**

```logql
{service=~"app-a|app-b"}
  | json
  | pattern `<_> | <_> | <_> | <_> | <_> | <trace_id_b> | <_> | <_> | <message_b> | <_>`
  | label_format trace_id=coalesce(trace_id, trace_id_b), msg=coalesce(message, message_b)
  | line_format "[{{.service}}] Trace: {{.trace_id}} - {{.msg}}"
```

**Find the trace IDs of all failed requests in App A in the last 5 minutes:**

```logql
{service="app-a", level="error"} | json | line_format "trace_id={{.trace_id}} route={{.http_route}} status={{.http_status_code}}"
```

---

## Metric Queries from Logs

LogQL can aggregate log streams into metrics (counts and rates) without requiring a separate metrics instrumentation point.

**Rate of all log lines per second from App A:**
```logql
rate({service="app-a"}[5m])
```

**Log-derived error ratio (App A):**

```logql
sum(rate({service="app-a", level="error"}[5m]))
/ sum(rate({service="app-a"}[5m]))
```

**Count of error lines per service in the last hour:**

```logql
sum by (service) (count_over_time({level="error"}[1h]))
```

**Rate of slow requests (App A - parsed field filter + metric query):**

```logql
sum(
  rate(
    {service="app-a"}
    | json
    | duration_ms > 500
    [5m]
  )
)
```

**Rate of database INSERT operations (App B - Pattern + Logfmt):**

```logql
sum(
  rate(
    {service="app-b"}
    | pattern `<_> | <_> | <_> | <_> | <_> | <_> | <_> | <_> | <_> | <extra>`
    | logfmt
    | db_operation == "INSERT"
    [5m]
  )
)
```

**Count of unique trace IDs with errors (approximate cardinality):**

```logql
count_over_time(
  {service="app-a", level="error"} | json | trace_id != "" [1h]
)
```

---

## Available Log Fields Reference

### App A (Parsed via `| json`)

| Extracted Label | Source Field | Example Value |
|---|---|---|
| `trace_id` | `trace_id` | `4bf92f3577b3...` |
| `span_id` | `span_id` | `00f067aa0ba...` |
| `request_id` | `request_id` | `01HQ4V2WK5...` |
| `http_method` | `http.method` | `POST` |
| `http_route` | `http.route` | `/api/messages` |
| `http_status_code` | `http.status_code` | `200` |
| `duration_ms` | `duration_ms` | `45` |

### App B (Parsed via `| pattern` + `| logfmt`)

| Extracted Label | Extraction Method | Example Value |
|---|---|---|
| `trace_id` | `pattern` (Position 6) | `4bf92f3577b3...` |
| `span_id` | `pattern` (Position 7) | `08a3c2b192e4...` |
| `request_id` | `pattern` (Position 8) | `01HQ4V2WK5...` |
| `message` | `pattern` (Position 9) | `message processed` |
| `db_operation` | `logfmt` on ExtraContext | `INSERT` |
| `db_table` | `logfmt` on ExtraContext | `public.messages` |
