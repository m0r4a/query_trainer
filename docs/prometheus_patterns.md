# Prometheus Query Patterns

This document is a reference for the PromQL exercises in the trainer dashboard. It covers the four metric types, how to identify them, and the canonical query patterns for each. All examples use metrics available in this project.

---

## Identifying Metric Types in the Metrics Browser

Before writing any query, you need to know the type of the metric. There are two ways to find this:

**In Grafana Explore:** Open the metrics browser, select a metric name, and look at the `TYPE` comment in the raw metadata returned. It will read `# TYPE metric_name counter` or `# TYPE metric_name histogram`, etc.

**In Prometheus UI:** Navigate to `/api/v1/metadata?metric=metric_name`. The response includes the type field.

**By naming convention (unreliable but useful as a first signal):**
- Names ending in `_total` are almost always counters
- Names ending in `_seconds`, `_bytes`, `_ratio` can be counters, gauges, or histograms depending on context
- Names ending in `_bucket`, `_sum`, `_count` in a group of three indicate a histogram family
- Names with no suffix that represent current state (active connections, queue depth) are gauges

The naming convention is not enforced by Prometheus. Always verify with metadata.

---

There's a [really good video](https://www.youtube.com/watch?v=fhx0ehppMGM) on which Julius (prometheus co-funder) gives an overview of all the metrics.

## Counter

A counter is a monotonically increasing value that resets to zero on process restart. It represents the total accumulated count of something since the process started.


**You never query the raw counter value.** A raw counter value is almost meaningless because it depends on how long the process has been running. You always apply `rate()` or `increase()` to make it useful.

`rate(counter[window])` returns the per-second rate of increase averaged over the window. Use this for dashboards and alerting.

`increase(counter[window])` returns the total increase over the window. Use this when you want a human-readable count ("how many errors in the last hour") rather than a rate.

[Here](https://www.youtube.com/watch?v=7uy_yovtyqw&pp=ygUScHJvbWV0aGV1cyBjb3VudGVy) is a video with a longer explanation.

### Patterns

**Per-second rate for a specific scope:**
*Generic Pattern:*
```promql
rate(<metric_name>{<label_key>="<label_value>"}[<time_window>])
```
*Applied Example (Per-second request rate for the entire service):*
```promql
rate(http_server_request_count_total{service="app-a"}[5m])
```

**Per-second rate broken down by a specific dimension:**
*Generic Pattern:*
```promql
sum by (<dimension_label>) (rate(<metric_name>{<label_key>="<label_value>"}[<time_window>]))
```
*Applied Example (Per-second rate broken down by route):*
```promql
sum by (route) (rate(http_server_request_count_total{service="app-a"}[5m]))
```

**Ratio as a percentage (Errors vs Total):**
*Generic Pattern:*
```promql
sum(rate(<metric_name>{<status_label>="<error_condition>"}[<time_window>]))
/ sum(rate(<metric_name>[<time_window>]))
* 100
```
*Applied Example (Error ratio as a percentage):*
```promql
sum(rate(http_server_request_count_total{service="app-a", status_code=~"5.."}[5m]))
/ sum(rate(http_server_request_count_total{service="app-a"}[5m]))
* 100
```
This is the single most important counter pattern. The numerator is the error rate, the denominator is the total rate, and the result is the percentage of requests that are errors. It applies to any counter that has a label distinguishing success from failure.

**Total absolute count over a time period (not a rate):**
*Generic Pattern:*
```promql
increase(<metric_name>{<label_key>="<label_value>"}[<time_window>])
```
*Applied Example (Total messages published in the last hour):*
```promql
increase(messaging_publish_count_total{service="app-a", status="success"}[1h])
```

**Top N entities by volume:**
*Generic Pattern:*
```promql
topk(<N>, sum by (<dimension_label>) (rate(<metric_name>[<time_window>])))
```
*Applied Example (Top routes by request volume):*
```promql
topk(5, sum by (route) (rate(http_server_request_count_total{service="app-a"}[5m])))
```

**Throughput comparison (Output vs Input):**
*Generic Pattern:*
```promql
sum(rate(<output_metric_name>[<time_window>]))
/ sum(rate(<input_metric_name>[<time_window>]))
```
*Applied Example (NATS messages consumed vs published):*
```promql
sum(rate(messaging_consume_count_total{status="success"}[5m]))
/ sum(rate(messaging_publish_count_total{status="success"}[5m]))
```
A ratio below 1.0 means the consumer is falling behind the producer. This is the counter-based signal that complements the `messaging_consumer_lag` gauge.

**Rate spike detection (current rate vs historical baseline):**
*Generic Pattern:*
```promql
rate(<metric_name>[<short_window>])
> rate(<metric_name>[<long_window>] offset <historical_offset>) * <multiplier>
```
*Applied Example (Current rate vs baseline 5 minutes ago):*
```promql
rate(http_server_request_count_total[5m])
> rate(http_server_request_count_total[1h] offset 5m) * 2
```

---

## Gauge

A gauge is a value that can go up or down arbitrarily. It represents a current state: how many connections are open right now, how much memory is in use, how many messages are pending.

**You query the raw gauge value.** You do not apply `rate()` to a gauge. What you do apply are aggregation functions (`sum`, `avg`, `max`, `min`) and window functions (`max_over_time`, `avg_over_time`) depending on what you want to express.

### Patterns

**Current absolute value:**

*Generic Pattern:*
```promql
<metric_name>{<label_key>="<label_value>"}
```

*Applied Example (Current active database connections):*
```promql
db_connections_active{service="app-b"}
```

*Applied Example (Current consumer lag):*
```promql
messaging_consumer_lag{service="app-b"}
```

This is the operational signal for the queue. If this value grows over time, App B cannot keep up with App A.

**Maximum value observed over a time window:**

*Generic Pattern:*
```promql
max_over_time(<metric_name>{<label_key>="<label_value>"}[<time_window>])
```

*Applied Example (Maximum consumer lag observed in the last 30 minutes):*
```promql
max_over_time(messaging_consumer_lag{service="app-b"}[30m])
```

**Aggregating state by a dimension:**

*Generic Pattern:*
```promql
sum by (<dimension_label>) (<metric_name>)
```

*Applied Example (PostgreSQL active connections by state):*
```promql
sum by (state) (pg_stat_activity_count)
```

**Unit conversion (e.g., Bytes to Megabytes):**

*Generic Pattern:*
```promql
<metric_name_in_bytes> / 1024 / 1024
```

*Applied Example (PostgreSQL database size in megabytes):*
```promql
pg_database_size_bytes / 1024 / 1024
```

**Absolute change in value compared to the past:**

*Generic Pattern:*
```promql
<metric_name> - <metric_name> offset <time_window>
```

*Applied Example (Change in consumer lag compared to one hour ago):*
```promql
messaging_consumer_lag{service="app-b"}
- messaging_consumer_lag{service="app-b"} offset 1h
```

A positive value means lag has grown. A negative value means the consumer has caught up. Zero means steady state.

**Percentage change compared to the past:**

*Generic Pattern:*
```promql
(
  <metric_name> - <metric_name> offset <time_window>
)
/ (<metric_name> offset <time_window>)
* 100
```

*Applied Example (Percentage change in consumer lag vs one hour ago):*
```promql
(
  messaging_consumer_lag{service="app-b"}
  - messaging_consumer_lag{service="app-b"} offset 1h
)
/ (messaging_consumer_lag{service="app-b"} offset 1h)
* 100
```

---

## Histogram


A histogram measures the distribution of a value across observations. Instead of recording one number per event, it records which bucket the observation fell into. A histogram metric is actually a family of three series:

- `metric_bucket{le="<upper_bound>"}` — counter of observations that fell at or below each boundary
- `metric_sum` — total sum of all observed values (counter)
- `metric_count` — total number of observations (counter)

**You query histograms with `histogram_quantile()`.** This function calculates a percentile from the bucket series. It requires `rate()` applied to the bucket series first, because buckets are counters.

The most important insight about histograms: percentile accuracy depends on where your bucket boundaries are. If all your observations cluster between 100ms and 200ms but your buckets jump from 100ms to 500ms, the p99 estimate will be imprecise. In this project, histogram buckets are configured at instrumentation time to cover the expected range of values.

Histograms are kind of difficult to understand at first if you haven't work with them before, [here](https://www.youtube.com/watch?v=yYbXak-1hew&pp=ugMICgJlcxABGAHKBRVwcm9tZXRoZXVzIGhpc3RvZ3JhbXM%3D) is a video with an in depth explanation.

### Patterns

**Calculate a specific percentile (e.g., p99):**

*Generic Pattern:*
```promql
histogram_quantile(<percentile_0_to_1>,
  sum by (le) (rate(<metric_name>_bucket[<time_window>]))
)
```

*Applied Example (p99 HTTP request latency):*
```promql
histogram_quantile(0.99,
  sum by (le) (rate(http_server_request_duration_seconds_bucket{service="app-a"}[5m]))
)
```

**Calculate percentile broken down by a dimension:**

*Generic Pattern:*
```promql
histogram_quantile(<percentile_0_to_1>,
  sum by (le, <dimension_label>) (rate(<metric_name>_bucket[<time_window>]))
)
```

*Applied Example (p99 latency broken down by route):*
```promql
histogram_quantile(0.99,
  sum by (le, route) (rate(http_server_request_duration_seconds_bucket{service="app-a"}[5m]))
)
```

Breaking down by a label requires including that label in the `sum by` clause alongside `le`. If you omit `le` from the `sum by`, `histogram_quantile` has no bucket information to work with.

**Calculate the true mean (average) from a histogram:**

*Generic Pattern:*
```promql
sum(rate(<metric_name>_sum[<time_window>]))
/ sum(rate(<metric_name>_count[<time_window>]))
```

*Applied Example (Average request latency):*
```promql
sum(rate(http_server_request_duration_seconds_sum{service="app-a"}[5m]))
/ sum(rate(http_server_request_duration_seconds_count{service="app-a"}[5m]))
```

This gives the mean latency. Note that the mean is often misleading for latency because a small number of very slow requests can dominate it. Use p99 for SLO work and the mean as supplementary context.

---

## Ratio and Comparison Patterns

These patterns combine multiple metrics and are independent of metric type.

**Success rate percentage:**

*Generic Pattern:*
```promql
sum(rate(<metric_name>{<status_label>="<success_value>"}[<time_window>]))
/ sum(rate(<metric_name>[<time_window>]))
* 100
```

*Applied Example (End-to-end pipeline success rate):*
```promql
sum(rate(messaging_consume_count_total{status="success"}[5m]))
/ sum(rate(messaging_consume_count_total[5m]))
* 100
```

*Applied Example (Database write success rate):*
```promql
sum(rate(db_query_duration_seconds_count{status="success"}[5m]))
/ sum(rate(db_query_duration_seconds_count[5m]))
* 100
```

**Ratio comparing two specific states (e.g., Errors to Successes):**

*Generic Pattern:*
```promql
sum(rate(<metric_name>{<status_label>="<error_value>"}[<time_window>]))
/ sum(rate(<metric_name>{<status_label>="<success_value>"}[<time_window>]))
```

*Applied Example (Ratio of failed publishes to successful publishes):*
```promql
sum(rate(messaging_publish_count_total{status="error"}[5m]))
/ sum(rate(messaging_publish_count_total{status="success"}[5m]))
```

A value above 0 here warrants investigation. Combined with NATS exporter metrics, this helps distinguish between application-level publish errors and NATS infrastructure problems.

---

## Alerting Patterns

These are not dashboard visualizations but expressions suitable for Prometheus alerting rules or Grafana alert thresholds.

**Missing data / Service is down:**

*Generic Pattern:*
```promql
absent(<metric_name>{<label_key>="<label_value>"})
```

*Applied Example (Service is down):*
```promql
absent(http_server_request_count_total{service="app-a"})
```
`absent()` returns a value of 1 when the given series does not exist. It fires when the app stops emitting metrics entirely, which `rate() > threshold` style alerts cannot detect.

**Threshold breach based on a percentage:**

*Generic Pattern:*
```promql
sum(rate(<metric_name>{<status_label>=~"<error_regex>"}[<time_window>]))
/ sum(rate(<metric_name>[<time_window>]))
> <decimal_threshold>
```

*Applied Example (Error rate exceeds 5% for more than 2 minutes):*
```promql
sum(rate(http_server_request_count_total{status_code=~"5.."}[5m]))
/ sum(rate(http_server_request_count_total[5m]))
> 0.05
```

**Continuous degradation over time:**

*Generic Pattern:*
```promql
<metric_name> > <metric_name> offset <time_window>
```

*Applied Example (Consumer lag growing higher than 30 minutes ago):*
```promql
messaging_consumer_lag > messaging_consumer_lag offset 30m
```

**Latency SLA breach:**

*Generic Pattern:*
```promql
histogram_quantile(<percentile>,
  sum by (le) (rate(<metric_name>_bucket[<time_window>]))
) > <latency_threshold_in_seconds>
```

*Applied Example (p99 latency exceeds 500ms):*
```promql
histogram_quantile(0.99,
  sum by (le) (rate(http_server_request_duration_seconds_bucket[5m]))
) > 0.5
```

---

## Available Metrics Reference

### App A (service="app-a")

| Metric | Type | Key Labels |
|---|---|---|
| `http_server_request_count_total` | Counter | `method`, `route`, `status_code` |
| `http_server_request_duration_seconds` | Histogram | `method`, `route`, `status_code` |
| `messaging_publish_count_total` | Counter | `subject`, `status` |
| `messaging_publish_duration_seconds` | Histogram | `subject`, `status` |

### App B (service="app-b")

| Metric | Type | Key Labels |
|---|---|---|
| `messaging_consume_count_total` | Counter | `subject`, `status` |
| `messaging_process_duration_seconds` | Histogram | `subject`, `status` |
| `messaging_consumer_lag` | Gauge | `subject` |
| `db_query_duration_seconds` | Histogram | `operation`, `table`, `status` |
| `db_connections_active` | Gauge | (none) |
| `db_connections_idle` | Gauge | (none) |

### NATS JetStream (nats_exporter)

| Metric | Type | Notes |
|---|---|---|
| `nats_core_mem_bytes` | Gauge | Process memory |
| `nats_core_msgs_total` | Counter | Messages routed |
| `nats_core_bytes_total` | Counter | Bytes routed |
| `nats_jetstream_consumer_num_pending` | Gauge | Per-consumer lag |
| `nats_jetstream_stream_total_messages` | Gauge | Messages stored in stream |
| `nats_jetstream_api_total` | Counter | API calls |

### PostgreSQL (postgres_exporter)

| Metric | Type | Notes |
|---|---|---|
| `pg_stat_activity_count` | Gauge | By `state` label |
| `pg_database_size_bytes` | Gauge | By `datname` label |
| `pg_stat_user_tables_n_live_tup` | Gauge | By `relname` label |
| `pg_locks_count` | Gauge | By `mode` label |
| `pg_stat_bgwriter_buffers_alloc_total` | Counter | Buffer allocations |
