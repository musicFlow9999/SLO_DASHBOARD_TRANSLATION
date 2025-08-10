# DQL Null Handling Functions

## Overview

Dynatrace Query Language (DQL) provides several functions to handle null values in your queries. While DQL doesn't have an `IFNULL` function like SQL, it offers equivalent functionality through the `coalesce()` and `default()` functions.

## Functions

### coalesce()

Returns the first non-null argument from a list of expressions.

#### Syntax
```dql
coalesce(expression1, expression2, ..., expressionN)
```

#### Parameters
- `expression1, expression2, ..., expressionN` - One or more expressions to evaluate

#### Returns
- The first non-null value from the arguments
- `null` if all arguments are null

#### Examples

**Basic usage:**
```dql
| fieldsAdd result = coalesce(field1, field2, "fallback_value")
```

**With timeseries data:**
```dql
timeseries { requests = sum(dt.service.request.count) }
| join [
    timeseries { errors = sum(dt.service.request.count) },
    filter: { failed == true }
  ],
  kind: leftOuter,
  prefix: "err."
| fieldsAdd safe_errors = coalesce(err.errors, repeat(0.0, arraySize(requests)))
```

### default()

Provides a default value when an expression evaluates to null.

#### Syntax
```dql
default(expression, defaultValue)
```

#### Parameters
- `expression` - The expression to evaluate
- `defaultValue` - The value to return if the expression is null

#### Returns
- The expression value if not null
- The default value if the expression is null

#### Examples

**Basic usage:**
```dql
| fieldsAdd safe_value = default(nullable_field, 0)
```

**With arrays in timeseries:**
```dql
| fieldsAdd failed_requests = default(err.failed, repeat(0.0, arraySize(total)))
```

## Common Use Cases

### 1. Handling Left Outer Joins

When performing left outer joins in timeseries queries, null values appear when there's no matching data in the joined dataset.

```dql
timeseries { total = sum(dt.service.request.count) },
  by: { dt.entity.service }
| join [
    timeseries { failed = sum(dt.service.request.count) },
      by: { dt.entity.service },
      filter: { failed == true }
  ],
  kind: leftOuter,
  on: { dt.entity.service, timeframe, interval },
  prefix: "err."
| fieldsAdd err.failed = coalesce(err.failed, repeat(0.0, arraySize(total)))
| fieldsAdd success_rate = ((total - err.failed) / total) * 100
```

### 2. Replacing Null Values in Fields

Replace null values with a meaningful default for display or calculation purposes.

```dql
fetch logs
| fieldsAdd category = coalesce(log.category, "uncategorized")
| fieldsAdd severity = default(log.severity, "info")
```

### 3. Aggregations with Default Values

Use the `default` parameter directly in timeseries aggregations to fill empty time slots.

```dql
timeseries {
  requests = sum(dt.service.request.count, default: 0),
  errors = sum(dt.service.request.count, filter: failed == true, default: 0)
}
```

### 4. Chaining Multiple Fallbacks

Use `coalesce()` to implement a fallback chain for multiple potential null sources.

```dql
| fieldsAdd display_name = coalesce(
    custom_name,
    entity_name,
    technical_id,
    "Unknown"
  )
```

## Best Practices

### 1. Choose the Right Function

- **Use `coalesce()`** when:
  - You have multiple potential sources for a value
  - You want to implement a fallback chain
  - You need to handle nulls from different fields

- **Use `default()`** when:
  - You have a single expression that might be null
  - You want to provide a simple fallback value
  - Working with timeseries aggregations (use the `default:` parameter)

### 2. Handle Array Sizes in Timeseries

When replacing null arrays in timeseries data, ensure the replacement array matches the original size:

```dql
| fieldsAdd safe_array = coalesce(
    nullable_array,
    repeat(0.0, arraySize(reference_array))
  )
```

### 3. Use nonempty Parameter

Prevent empty results in timeseries queries by using the `nonempty:true` parameter:

```dql
timeseries {
  metric = sum(dt.host.cpu.usage, default: 0)
},
nonempty: true
```

### 4. Combine with Conditional Logic

Use with the `if()` function for more complex null handling:

```dql
| fieldsAdd result = if(
    isNotNull(value),
    value * 100,
    else: 0
  )
```

## Performance Considerations

1. **Early null handling**: Apply `coalesce()` or `default()` as early as possible in your query pipeline to avoid propagating nulls through multiple operations.

2. **Aggregation defaults**: Use the `default:` parameter in aggregation functions rather than post-processing with `coalesce()` when possible:
   ```dql
   # Preferred
   timeseries { count = sum(metric, default: 0) }
   
   # Less efficient
   timeseries { count = sum(metric) }
   | fieldsAdd count = coalesce(count, 0)
   ```

3. **Minimize array operations**: When working with timeseries arrays, use `repeat()` efficiently by calculating array sizes once and reusing them.

## Related Functions

- `isNull()` - Tests if a value is null
- `isNotNull()` - Tests if a value is not null
- `if()` - Conditional evaluation with null handling
- `isTrueOrNull()` - Returns true if the condition is true or null

## Additional Resources

- [DQL Conditional Functions Documentation](https://docs.dynatrace.com/docs/platform/grail/dynatrace-query-language/functions/conditional-functions)
- [DQL Operators Documentation](https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-query-language/operators)
- [Timeseries Command Reference](https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-query-language/commands/metric-commands)