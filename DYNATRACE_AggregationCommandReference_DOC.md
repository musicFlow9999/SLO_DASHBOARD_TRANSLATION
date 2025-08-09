# DQL Aggregation Commands Reference

## Overview

Dynatrace Query Language (DQL) provides powerful aggregation commands for analyzing and transforming data. This document covers the three main aggregation commands: `fieldsSummary`, `makeTimeseries`, and `summarize`.

---

## 1. fieldsSummary Command

### Description
The `fieldsSummary` command calculates the cardinality of field values that the specified fields have.

### Syntax
```dql
fieldsSummary field, … [, topValues] [, extrapolateSamples]
```

### Parameters

| Parameter | Type | Description | Required |
|-----------|------|-------------|----------|
| `field` | field identifier | A field identifier | Required |
| `topValues` | positive long | The number of top N values to be returned | Optional |
| `extrapolateSamples` | boolean | Flag indicating if the cardinality shall be multiplied with a possible sampling rate | Optional |

### Examples

#### Basic Example
Calculate the cardinality of values for the field `host`:

```dql
data record(host = "host-a"),
     record(host = "host-a"),
     record(host = "host-b")
| fieldsSummary host
```

**Result:**
- Total cardinality: 3
- Unique values: 3
- host-a (count: 2)
- host-b (count: 1)

#### Advanced Example with Sampling
Fetch logs with a sampling ratio of 10,000 and calculate cardinality with top 10 values:

```dql
fetch logs, samplingRatio: 10000
| fieldsSummary dt.entity.host, topValues: 10, extrapolateSamples: true
```

---

## 2. makeTimeseries Command

### Description
Creates timeseries from the data in the stream. The `makeTimeseries` command provides a convenient way to chart raw non-metric data (such as events or logs) over time.

### Syntax
```dql
makeTimeseries [by: { [expression, …] }] [, interval] [, bins] [, from] [, to] 
               [, timeframe] [, time ,] [, spread ,] [, nonempty ,] 
               [, scalar:] aggregation, …
```

### Parameters

| Parameter | Type | Description | Required |
|-----------|------|-------------|----------|
| `aggregation` | aggregation function | The aggregation function to create the series | Required |
| `time` | expression | Expression providing the timestamp for a record | Optional |
| `from` | timestamp, duration | Start timestamp of the series | Optional |
| `to` | timestamp, duration | End timestamp of the series | Optional |
| `timeframe` | timeframe | The timeframe of the series | Optional |
| `bins` | positive integer | Number of buckets (12-1,500) | Optional |
| `interval` | positive duration | Length of a bucket in the series | Optional |
| `by` | list of expressions | Expressions to split the series by | Optional |
| `spread` | timeframe | Timeframe for bucket calculation | Optional |
| `nonempty` | boolean | Produces empty series when there is no data | Optional |
| `scalar` | boolean | Calculate single scalar value spanning whole timeframe | Optional |

### Supported Aggregation Functions

The following aggregation functions are available with `makeTimeseries`:

#### Function Signatures
```dql
sum(expression [, default] [, rate])
avg(expression [, default] [, rate])
min(expression [, default] [, rate])
max(expression [, default] [, rate])
count([default] [, rate])
countIf(expression [, default] [, rate])
countDistinctExact(expression [, default] [, rate])
countDistinctApprox(expression [, precision] [, default] [, rate])
percentile(expression, percentile [, default] [, rate])
start()
end()
```

#### Aggregation Function Parameters

| Parameter | Type | Description | Required |
|-----------|------|-------------|----------|
| `expression` | expression | The expression to create series for | Required |
| `default` | number | Default value for gaps/empty bins (default: null) | Optional |
| `rate` | duration | Duration to adjust bin values: (binValue / interval) * rate | Optional |
| `percentile` | double, long | n-th percentile (0-100) | Required for percentile() |
| `precision` | long | Precision level (3-16, default: 14) | Optional for countDistinctApprox() |

### Key Behaviors

1. **Interval vs Bins**: The `interval` and `bins` parameters are mutually exclusive
2. **Time Series Alignment**: Produces homogenous time series with identical start/end timestamps
3. **Missing Data**: Empty time slots filled with `null` or specified `default` value
4. **No Records**: Returns empty result unless `nonempty: true` is specified

### Examples

#### Basic Count Over Time
```dql
data record(timestamp = toTimestamp("2019-08-01T09:30:00.000-0400")),
     record(timestamp = toTimestamp("2019-08-01T09:31:00.000-0400")),
     record(timestamp = toTimestamp("2019-08-01T09:31:30.000-0400")),
     record(timestamp = toTimestamp("2019-08-01T09:32:00.000-0400"))
| makeTimeseries count(),
    from: toTimestamp("2019-08-01T09:30:00.000-0400"),
    to: toTimestamp("2019-08-01T09:33:00.000-0400")
```

#### Error Logs Over Time
```dql
fetch logs
| filter dt.entity.host == "HOST-15FE58391F97B7AA" and loglevel == "ERROR"
| makeTimeseries count()
```

#### Business Events with Fixed Interval
```dql
fetch bizevents, from: now() - 24h
| filter accountId == 7
| filter in(event.category, array("/broker-service/v1/trade/long/buy", "/v1/trade/buy"))
| makeTimeseries count(default: 0), interval: 30m
```

#### Advanced Multi-Metric Analysis
```dql
fetch bizevents, from: now() - 7d
| filter in(event.type, array("com.easytrade.long-sell", "com.easytrade.quick-sell"))
| makeTimeseries {
    count(),
    high_volume = countIf(amount >= 100 and amount <= 10000),
    max(price)
  },
  by: { accountId },
  interval: 1d
```

#### Span Duration Percentiles
```dql
fetch spans
| filter request.is_root_span == true
| filter endpoint.name == "GET /api/cart"
| makeTimeseries {
    avg = avg(duration),
    p50 = median(duration),
    p90 = percentile(duration, 90)
  }
```

#### Scalar Aggregation
```dql
fetch logs
| makeTimeseries countDistinct(dt.entity.host, scalar: true),
    by: {k8s.cluster.name}
```

#### Handling Empty Results
Without `nonempty`:
```dql
fetch logs
| filter status >= 500
| makeTimeseries count = count(default: 0), interval: 30m
# Returns: No records if no matching data
```

With `nonempty`:
```dql
fetch logs
| filter status >= 500
| makeTimeseries count = count(default: 0), interval: 30m, nonempty: true
# Returns: Time series with 0 values
```

---

## 3. summarize Command

### Description
Groups together records that have the same values for a given field and aggregates them.

### Syntax
```dql
summarize [field =] aggregation, ... [, by: {[field =] expression, ...}]
```

### Parameters

| Parameter | Type | Description | Required |
|-----------|------|-------------|----------|
| `expression` | Various types* | An expression to group by | Required |
| `aggregation` | Various types* | An aggregation function | Required |

*Types: array, boolean, counter, double, duration, ip, long, record, string, timeframe, timestamp

### Key Behaviors

1. **Empty Input with No By Clause**: Returns single record as result
2. **Non-existent Fields**: Adds `null` group to result
3. **Multiple Aggregations**: Supports multiple aggregation functions in single command

### Examples

#### Simple Sum
```dql
data record(value = 2),
     record(value = 3),
     record(value = 7),
     record(value = 7),
     record(value = 1)
| summarize sum(value)
# Result: 20
```

#### Group By with Aliases
```dql
data record(value = 2, cat = "a"),
     record(value = 3, cat = "b"),
     record(value = 7, cat = "a"),
     record(value = 7, cat = "b"),
     record(value = 1, cat = "b")
| summarize sum = sum(value),
    by: { category = cat }
```

**Result:**
- category: a, sum: 9
- category: b, sum: 11

#### Empty Input Handling
```dql
data record(value = 2),
     record(value = 3),
     record(value = 7),
     record(value = 7),
     record(value = 1)
| filter value > 7
| summarize count(), sum(value), collectArray(value), takeAny(value)
# Result: 0, null, [], null
```

#### Handling Missing Fields
```dql
data record(value = 2),
     record(value = 3, category = "b"),
     record(value = 7, category = "a"),
     record(value = 7, category = "b"),
     record(value = 1)
| summarize sum(value),
    by: { category }
```

**Result:**
- category: a, sum: 7
- category: b, sum: 10
- category: null, sum: 3

#### Join Alternative Using Summarize
```dql
data record(key = "a", value = 1),
     record(key = "b", value = 2),
     record(key = "c", value = 4),
     record(key = "b", amount = 10),
     record(key = "c", amount = 20),
     record(key = "c", amount = 40),
     record(key = "d", amount = 50)
| summarize {
    value = takeAny(value),
    amount = arrayRemoveNulls(collectArray(amount))
  },
  by: { key }
| expand amount
```

#### Iterative Expression with Arrays
```dql
data record(a = array(2, 2)),
     record(a = array(7, 1))
| summarize sum(a[])
# Result: [9, 3] (element-wise sum)
```

#### Log Level Analysis
```dql
fetch logs
| summarize {
    errors = countIf(loglevel == "ERROR"),
    warnings = countIf(loglevel == "WARN"),
    severe = countIf(loglevel == "DEBUG")
  },
  by: {
    dt.entity.host,
    dt.entity.process_group
  }
```

---

## Best Practices

### 1. Choose the Right Command
- **fieldsSummary**: Use for field cardinality analysis and top value discovery
- **makeTimeseries**: Use for time-based visualizations and trending
- **summarize**: Use for general aggregations and grouping operations

### 2. Performance Optimization
- Use `bins` parameter wisely (12-1,500 range) for makeTimeseries
- Consider sampling with `extrapolateSamples` for large datasets
- Use `scalar: true` for single-value calculations across timeframes

### 3. Data Quality
- Handle missing data with `default` parameters
- Use `nonempty: true` to ensure consistent chart output
- Consider `arrayRemoveNulls()` when collecting arrays

### 4. Time Series Considerations
- Interval and bins parameters are mutually exclusive
- Time series are automatically aligned and homogenized
- For spans, `makeTimeseries` uses `start_time` field automatically

## Common Use Cases

### 1. Log Analysis
- Error rate trending over time
- Log level distribution by host/service
- Cardinality analysis of error sources

### 2. Business Intelligence
- Transaction volume analysis
- Performance percentiles over time
- Revenue aggregation by category

### 3. Infrastructure Monitoring
- Resource utilization trending
- Service availability metrics
- Host/container performance analysis

### 4. Application Performance
- Response time percentiles
- Request rate analysis
- Error rate by endpoint

---

## Related Documentation
- [DQL Aggregation Functions](https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-query-language/functions/aggregation-functions)
- [DQL Commands Reference](https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-query-language/commands)
- [DQL Overview](https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-query-language)