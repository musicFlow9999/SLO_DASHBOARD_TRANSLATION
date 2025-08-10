# DQL Array Operations and Default Value Handling Guide

## Overview

Dynatrace Query Language (DQL) provides powerful array operations and default value handling mechanisms for timeseries calculations. While DQL doesn't have a `repeat` function like some other query languages, it offers several elegant alternatives through scalar-to-array broadcasting, default functions, and array manipulation techniques.

This guide focuses on practical approaches for handling missing data, creating default arrays, and performing calculations like availability percentages where you need to handle potential null values in joined timeseries data.

---

## The Challenge: No `repeat` Function

Unlike some query languages, DQL doesn't have a `repeat(value, count)` function to create arrays with repeated values. However, DQL provides several alternative approaches that are often more elegant and performant.

### Common Scenario: Availability Calculation

When calculating availability percentages from service metrics, you often need to handle cases where:
- Some services have no failed requests (null values after a left outer join)
- You need to substitute zeros for missing failure data
- Array operations require matching dimensions

---

## Solution Approaches

### 1. Scalar-to-Array Broadcasting (Recommended)

DQL automatically handles scalar-to-array operations by broadcasting the scalar value to match array dimensions. This is the most idiomatic approach in DQL.

```dql
// Availability % = (total - failed) / total * 100
timeseries { total = sum(dt.service.request.count) },
  by: { dt.entity.service },
  filter: { in(dt.entity.service, array($services)) }
| join [
    timeseries { failed = sum(dt.service.request.count, default: 0.0) },
      by: { dt.entity.service },
      filter: { failed == true and in(dt.entity.service, array($services)) },
      nonempty: true
  ],
  kind: leftOuter,
  on: { dt.entity.service, timeframe, interval },
  prefix: "err."
| fieldsAdd availability_pct = 
    (total[] - default(err.failed[], 0.0)) / total[] * 100
| fieldsAdd service = entityName(dt.entity.service)
| fields service, availability_pct, timeframe, interval
```

**Key Points:**
- `default(err.failed[], 0.0)` - The scalar `0.0` is automatically broadcast to array dimensions
- No explicit array creation needed
- Clean and readable syntax

### 2. Using `coalesce()` Function

The `coalesce()` function returns the first non-null value from its arguments, making it perfect for handling missing data.

```dql
| fieldsAdd availability_pct = 
    (total[] - coalesce(err.failed[], 0.0)) / total[] * 100
```

**Benefits:**
- Similar to `default()` but more explicit about null handling
- Works with both scalar and array values
- Part of the standard SQL-like function set

### 3. Array Multiplication for Zero Arrays

When you need an array of zeros with the same dimensions as another array:

```dql
| fieldsAdd zeros = total[] * 0.0  // Creates array of zeros
| fieldsAdd availability_pct = 
    (total[] - default(err.failed[], zeros[])) / total[] * 100
```

**Use Cases:**
- When you need explicit zero arrays for multiple operations
- Complex calculations requiring dimensional matching

### 4. Conditional Logic with `if()`

For more complex scenarios with conditional logic:

```dql
| fieldsAdd availability_pct = 
    if(isNotNull(err.failed[]), 
       (total[] - err.failed[]) / total[] * 100,
       100.0)  // Assume 100% availability if no failures
```

### 5. Default Values in Aggregation

Set defaults directly in the timeseries aggregation:

```dql
timeseries { 
    total = sum(dt.service.request.count, default: 0.0),
    failed = sum(dt.service.request.count, default: 0.0, filter: { failed == true })
  },
  by: { dt.entity.service },
  nonempty: true
```

---

## Working with Joins and Default Values

### Left Outer Join Pattern

When performing left outer joins with timeseries data, handle missing values from the right side:

```dql
timeseries total = sum(dt.service.request.count, default: 0), 
  nonempty: true,
  by: { dt.entity.service }
| fieldsAdd key = record(dt.entity.service)  // Create join key
| join [
    timeseries failed = sum(dt.service.request.count, default: 0),
      nonempty: true,
      by: { dt.entity.service },
      filter: { failed == true }
    | fieldsAdd key = record(dt.entity.service)
  ],
  kind: leftOuter,
  on: { key },
  fields: { failed }
| fieldsAdd failed = coalesce(failed, total[] * 0)  // Handle nulls
```

### Important Parameters

#### `nonempty: true`
- Ensures timeseries returns results even when no data exists
- Prevents "There are no records" messages
- Creates timeseries with default values

#### `default: value`
- Fills empty time slots with specified value instead of null
- Applied during aggregation
- Works with all aggregation functions

#### `union: true`
- In timeseries: captures all entities even if missing from some columns
- Equivalent to SQL OUTER JOIN behavior
- Useful for comprehensive reporting

---

## Array Function Reference

### Essential Array Functions for Default Handling

| Function | Description | Example |
|----------|-------------|---------|
| `default(expr, value)` | Returns value if expr is null | `default(failed[], 0.0)` |
| `coalesce(expr1, expr2, ...)` | Returns first non-null value | `coalesce(failed[], 0.0)` |
| `arrayRemoveNulls(array)` | Removes null elements from array | `arrayRemoveNulls(collectArray(value))` |
| `isNull(expr)` | Tests if expression is null | `if(isNull(failed), 0, failed)` |
| `isNotNull(expr)` | Tests if expression is not null | `filter isNotNull(value)` |

### Array Manipulation Functions

| Function | Description | Example |
|----------|-------------|---------|
| `arraySize(array)` | Returns number of elements | `arraySize(total)` |
| `arraySum(array)` | Sum of array elements | `arraySum(values[])` |
| `arrayAvg(array)` | Average of array elements | `arrayAvg(response_times[])` |
| `arrayMin(array)` | Minimum value in array | `arrayMin(measurements[])` |
| `arrayMax(array)` | Maximum value in array | `arrayMax(measurements[])` |

---

## Best Practices

### 1. Choose the Right Approach

- **Simple null replacement**: Use `default()` or `coalesce()` with scalar values
- **Complex conditions**: Use `if()` statements with appropriate logic
- **Join operations**: Set defaults in aggregation and use `nonempty: true`
- **Array operations**: Leverage scalar broadcasting instead of creating arrays

### 2. Performance Optimization

```dql
// Good: Scalar broadcasting (efficient)
fieldsAdd result = array[] - default(nullableArray[], 0.0)

// Less optimal: Creating unnecessary arrays
fieldsAdd zeros = array[] * 0.0,
         result = array[] - default(nullableArray[], zeros[])
```

### 3. Handle Edge Cases

```dql
// Prevent division by zero
| fieldsAdd availability_pct = 
    if(total[] > 0,
       (total[] - default(failed[], 0.0)) / total[] * 100,
       100.0)  // or null, depending on business logic
```

### 4. Use Appropriate Join Types

```dql
// For availability: leftOuter keeps all services
join [...], kind: leftOuter

// For comparisons: inner join for only matching data
join [...], kind: inner

// For complete picture: outer join for all data
join [...], kind: outer
```

---

## Complete Example: Service Availability Dashboard

Here's a comprehensive example combining multiple techniques:

```dql
// Calculate service availability with proper null handling
timeseries { 
    total_requests = sum(dt.service.request.count, default: 0) 
  },
  by: { dt.entity.service, dt.entity.environment },
  filter: { in(dt.entity.service, array($services)) },
  interval: 5m,
  from: now() - 1h
| join [
    timeseries { 
        failed_requests = sum(dt.service.request.count, default: 0) 
      },
      by: { dt.entity.service, dt.entity.environment },
      filter: { 
        failed == true and 
        in(dt.entity.service, array($services)) 
      },
      interval: 5m,
      from: now() - 1h,
      nonempty: true
  ],
  kind: leftOuter,
  on: { dt.entity.service, dt.entity.environment, timeframe, interval },
  prefix: "failure."
| fieldsAdd 
    // Handle nulls with scalar broadcasting
    failed = default(failure.failed_requests[], 0.0),
    // Calculate availability with protection against division by zero
    availability_pct = if(total_requests[] > 0,
                         (total_requests[] - failed[]) / total_requests[] * 100,
                         100.0),
    // Add metadata
    service_name = entityName(dt.entity.service),
    environment = entityName(dt.entity.environment, type: "dt.entity.environment")
| fieldsAdd
    // Calculate average availability across time series
    avg_availability = arrayAvg(availability_pct[]),
    // Find minimum availability (worst point)
    min_availability = arrayMin(availability_pct[]),
    // Check if SLA is met (e.g., 99.9%)
    sla_met = if(min_availability >= 99.9, "✓ Met", "✗ Not Met")
| fields service_name, environment, availability_pct, 
         avg_availability, min_availability, sla_met, 
         timeframe, interval
| sort avg_availability desc
```

---

## Common Pitfalls and Solutions

### Pitfall 1: Array Type Mismatches in Joins

**Problem:**
```dql
// Error: Incompatible data types
| join [...], on: { array_field }
```

**Solution:**
```dql
// Use scalar fields or create proper join keys
| fieldsAdd key = record(field1, field2)
| join [...], on: { key }
```

### Pitfall 2: Empty Result Sets

**Problem:**
```dql
// Returns "There are no records"
timeseries count = sum(metric)
| filter condition_with_no_matches
```

**Solution:**
```dql
timeseries count = sum(metric, default: 0), nonempty: true
| filter condition_with_no_matches
```

### Pitfall 3: Null Propagation in Calculations

**Problem:**
```dql
// null - 5 = null (not -5)
| fieldsAdd result = nullableField[] - 5
```

**Solution:**
```dql
| fieldsAdd result = default(nullableField[], 0) - 5
```

---

## Advanced Techniques

### Creating Dynamic Default Arrays

When you need defaults based on other array values:

```dql
// Create baseline from historical average
| fieldsAdd historical_avg = arrayAvg(last_week_values[])
| fieldsAdd current = default(today_values[], historical_avg)
```

### Handling Sparse Data in Timeseries

For irregular data points:

```dql
fetch logs
| makeTimeseries {
    event_count = count(default: 0),
    error_count = countIf(level == "ERROR", default: 0)
  },
  interval: 5m,
  nonempty: true
| fieldsAdd error_rate = if(event_count[] > 0,
                            error_count[] / event_count[] * 100,
                            0.0)
```

### Combining Multiple Data Sources

When merging data with different granularities:

```dql
timeseries fine_grain = avg(metric1), interval: 1m
| append [
    timeseries coarse_grain = avg(metric2), interval: 5m
    | fieldsAdd interpolated = coarse_grain[] // DQL handles alignment
]
| fieldsAdd combined = coalesce(fine_grain[], interpolated[])
```

---

## Conclusion

While DQL doesn't have a `repeat` function, its scalar-to-array broadcasting and comprehensive default handling functions provide more elegant solutions for most use cases. The key is understanding:

1. **Scalar broadcasting** automatically handles dimension matching
2. **Default functions** (`default()`, `coalesce()`) elegantly handle nulls
3. **Array operations** work seamlessly with mixed scalar/array inputs
4. **Proper use of parameters** (`nonempty`, `default`, `union`) prevents empty results

These patterns make DQL queries more maintainable and often more performant than explicit array creation would be.

---

## References

- [DQL Array Functions Documentation](https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-query-language/functions/array-functions)
- [DQL Aggregation Commands](https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-query-language/commands/aggregation-commands)
- [DQL Timeseries Command](https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-query-language/commands/metric-commands)
- [DQL Join Operations](https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-query-language/commands/correlation-and-join-commands)