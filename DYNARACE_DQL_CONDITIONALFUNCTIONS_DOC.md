# DQL Conditional Functions - README

## Overview

This document provides a comprehensive guide to conditional functions in Dynatrace Query Language (DQL). Conditional functions allow you to implement logic-based data transformations and handle null values effectively in your queries.

## Table of Contents
- [Available Functions](#available-functions)
- [Function Details](#function-details)
  - [coalesce()](#coalesce)
  - [if()](#if)
- [Use Cases](#use-cases)
- [Best Practices](#best-practices)
- [Advanced Examples](#advanced-examples)
- [Performance Considerations](#performance-considerations)

## Available Functions

DQL provides two primary conditional functions:

| Function | Purpose | Return Type |
|----------|---------|-------------|
| `coalesce()` | Returns the first non-null argument | Varies based on input |
| `if()` | Evaluates a condition and returns different values based on the result | Varies based on then/else parameters |

## Function Details

### coalesce()

Returns the first non-null argument from a list of expressions. If all arguments are null, it returns null.

#### Syntax
```dql
coalesce(expression1, expression2, ..., expressionN)
```

#### Parameters
- **expression**: One or more expressions of compatible types (array, boolean, double, duration, ip, long, record, string, timeframe, timestamp)
- All expressions should be of compatible types
- At least one expression is required

#### Basic Example
```dql
data record(a = "a", b = "b", c = "c"),
     record(b = "b", c = "c"),
     record(c = "c"),
     record()
| fieldsAdd first_non_null = coalesce(a, b, c)
```

#### Result
| a    | b    | c    | first_non_null |
|------|------|------|----------------|
| a    | b    | c    | a              |
| null | b    | c    | b              |
| null | null | c    | c              |
| null | null | null | null           |

### if()

Evaluates a boolean condition and returns different values based on whether the condition is true or false.

#### Syntax
```dql
if(condition, then [, else])
```

#### Parameters
- **condition** (required): Boolean expression to evaluate
- **then** (required): Value to return if condition is true
- **else** (optional): Value to return if condition is false or null
  - If omitted and condition is false/null, returns null

#### Supported Return Types
- array
- boolean
- double
- duration
- ip
- long
- record
- string
- timeframe
- timestamp

#### Basic Example
```dql
data record(a = 10),
     record(a = 20)
| fieldsAdd 
    simple_check = if(a < 15, "small"),
    with_else = if(a < 15, "small", else: "large")
```

#### Result
| a  | simple_check | with_else |
|----|-------------|-----------|
| 10 | small       | small     |
| 20 | null        | large     |

## Use Cases

### 1. Handling Missing Data
Use `coalesce()` to provide fallback values for potentially null fields:

```dql
fetch logs
| fieldsAdd message = coalesce(
    custom_message,
    error_message,
    warning_message,
    "No message available"
  )
```

### 2. Categorizing Metrics
Use `if()` to create categories based on numeric thresholds:

```dql
fetch dt.metrics.builtin
| fieldsAdd severity = if(
    value > 90, 
    "CRITICAL",
    else: if(value > 70, "WARNING", else: "NORMAL")
  )
```

### 3. Data Normalization
Combine both functions to normalize inconsistent data:

```dql
fetch spans
| fieldsAdd normalized_status = coalesce(
    if(status_code >= 500, "ERROR"),
    if(status_code >= 400, "CLIENT_ERROR"),
    if(status_code >= 200, "SUCCESS"),
    "UNKNOWN"
  )
```

### 4. Conditional Aggregations
Use conditional logic within aggregations:

```dql
fetch logs
| summarize 
    error_count = countIf(loglevel == "ERROR"),
    total_count = count()
| fieldsAdd error_rate = if(
    total_count > 0,
    error_count * 100.0 / total_count,
    else: 0
  )
```

## Best Practices

### 1. Type Consistency
Ensure all branches of conditional expressions return compatible types:

```dql
// ✅ Good: Both branches return strings
if(condition, "yes", else: "no")

// ❌ Bad: Mixed types (string and number)
if(condition, "yes", else: 0)
```

### 2. Null Handling
Always consider null cases in your conditions:

```dql
// Handle potential null values explicitly
| fieldsAdd category = if(
    isNull(value), 
    "UNKNOWN",
    else: if(value > 100, "HIGH", else: "LOW")
  )
```

### 3. Nested Conditions
For complex logic, use nested if statements with clear formatting:

```dql
| fieldsAdd priority = if(
    severity == "CRITICAL" AND environment == "PRODUCTION",
    "P1",
    else: if(
      severity == "CRITICAL" OR environment == "PRODUCTION",
      "P2",
      else: if(
        severity == "WARNING",
        "P3",
        else: "P4"
      )
    )
  )
```

### 4. Performance Optimization
Use `coalesce()` instead of multiple nested `if()` statements when checking for null:

```dql
// ✅ Efficient
coalesce(field1, field2, field3, "default")

// ❌ Less efficient
if(isNotNull(field1), field1, 
  else: if(isNotNull(field2), field2,
    else: if(isNotNull(field3), field3, else: "default")
  )
)
```

## Advanced Examples

### Dynamic Threshold Calculation
```dql
fetch dt.metrics.cpu.usage
| fieldsAdd adaptive_threshold = if(
    hour(timestamp) >= 9 AND hour(timestamp) <= 17,
    85,  // Business hours threshold
    else: 95  // Off-hours threshold
  )
| fieldsAdd alert_status = if(
    value > adaptive_threshold,
    "ALERT",
    else: "OK"
  )
```

### Complex Field Mapping
```dql
fetch logs
| fieldsAdd normalized_level = coalesce(
    if(contains(content, "ERROR"), "ERROR"),
    if(contains(content, "WARN"), "WARNING"),
    if(contains(content, "INFO"), "INFO"),
    if(loglevel != null, loglevel),
    "UNKNOWN"
  )
```

### Service Health Scoring
```dql
fetch dt.service.metrics
| fieldsAdd health_score = 
    if(error_rate > 5, 0, else:
      if(error_rate > 1, 25, else:
        if(response_time > 1000, 50, else:
          if(response_time > 500, 75, else: 100)
        )
      )
    )
| fieldsAdd health_status = 
    if(health_score >= 80, "HEALTHY", else:
      if(health_score >= 50, "DEGRADED", else: "UNHEALTHY")
    )
```

### Multi-Field Validation
```dql
fetch bizevents
| fieldsAdd validation_status = if(
    isNotNull(user_id) AND isNotNull(transaction_id),
    coalesce(
      if(amount <= 0, "INVALID_AMOUNT"),
      if(timestamp > now(), "FUTURE_DATE"),
      "VALID"
    ),
    else: "MISSING_REQUIRED_FIELDS"
  )
```

## Performance Considerations

### 1. Evaluation Order
- Conditions in `if()` statements are evaluated lazily
- Only the necessary branch is computed
- Place most likely conditions first in nested structures

### 2. Complex Expressions
- Avoid redundant calculations by storing intermediate results:

```dql
// Store calculation result once
| fieldsAdd calc_value = complex_calculation()
| fieldsAdd category = if(calc_value > 100, "HIGH", else: "LOW")
```

### 3. Coalesce Optimization
- `coalesce()` stops evaluating at the first non-null value
- Order arguments from most likely to least likely to be non-null

### 4. Memory Usage
- Conditional functions don't significantly impact memory
- However, complex nested conditions may affect query readability and maintenance

## Common Patterns

### Default Value Pattern
```dql
| fieldsAdd safe_value = coalesce(potentially_null_field, 0)
```

### Ternary Operation Pattern
```dql
| fieldsAdd result = if(condition, true_value, else: false_value)
```

### Switch-Case Pattern
```dql
| fieldsAdd category = coalesce(
    if(value == "A", "Category 1"),
    if(value == "B", "Category 2"),
    if(value == "C", "Category 3"),
    "Other"
  )
```

### Guard Clause Pattern
```dql
| fieldsAdd safe_division = if(
    denominator != 0,
    numerator / denominator,
    else: null
  )
```

## Troubleshooting

### Common Issues

1. **Type Mismatch Errors**
   - Ensure all branches return compatible types
   - Use explicit type casting when necessary

2. **Unexpected Null Results**
   - Check if condition evaluates to null
   - Provide explicit else clause

3. **Complex Logic Errors**
   - Break down complex conditions into multiple fields
   - Use intermediate calculations for debugging

### Debugging Tips

1. Add intermediate fields to inspect condition results:
```dql
| fieldsAdd 
    debug_condition = (value > 100),
    result = if(value > 100, "HIGH", else: "LOW")
```

2. Use coalesce to handle edge cases:
```dql
| fieldsAdd safe_result = coalesce(
    complex_calculation(),
    "CALCULATION_FAILED"
  )
```

## Related Functions

- `isNull()` / `isNotNull()` - Check for null values
- `contains()` - String matching for conditions
- `in()` - Check if value exists in a list
- `between()` - Range checking
- `case()` - Alternative conditional structure (if available)

## Summary

Conditional functions in DQL provide powerful tools for data transformation and logic implementation:

- **`coalesce()`** - Ideal for null handling and providing default values
- **`if()`** - Essential for implementing business logic and data categorization

These functions are fundamental for creating robust, production-ready queries that handle edge cases and implement complex business rules effectively.

## References

- [Official Dynatrace DQL Documentation](https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-query-language)
- [DQL Functions Reference](https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-query-language/functions)
- [DQL Best Practices Guide](https://docs.dynatrace.com/docs/platform/grail/dynatrace-query-language)