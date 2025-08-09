# DQL sum() Function Error Guide - Array Type Mismatch

## Error Message
```
This parameter of the `sum()` function should be a number or a duration, but was an array
```

## Table of Contents
- [Understanding the Error](#understanding-the-error)
- [Root Causes](#root-causes)
- [Common Scenarios](#common-scenarios)
- [Solutions](#solutions)
- [Correct Usage Patterns](#correct-usage-patterns)
- [Working with Arrays](#working-with-arrays)
- [Best Practices](#best-practices)
- [Troubleshooting Checklist](#troubleshooting-checklist)

---

## Understanding the Error

This error occurs when the `sum()` function receives an array data type instead of the expected number or duration value. The `sum()` function in DQL is designed to aggregate single numeric values, not arrays.

### What sum() Expects
- **Valid Input Types:**
  - `number` (double, long)
  - `duration`
  
- **Invalid Input Types:**
  - `array`
  - `string`
  - `boolean`
  - `record`
  - `timestamp`

---

## Root Causes

### 1. Incorrect Context Usage
Using `sum()` outside of aggregation contexts:
```dql
// ❌ INCORRECT - sum() used in fieldsAdd
fetch metrics
| fieldsAdd total = sum(value)  // Error: sum() is an aggregation function
```

### 2. Array Field Input
Attempting to sum a field that contains arrays:
```dql
// ❌ INCORRECT - field contains array values
fetch data
| summarize total = sum(array_field)  // Error if array_field is [1,2,3]
```

### 3. Conditional Functions Returning Arrays
Using conditional functions that return array types:
```dql
// ❌ INCORRECT - if() returns an array
fetch logs
| summarize total = sum(
    if(condition, [1, 2, 3], else: [4, 5, 6])  // Returns array
  )
```

### 4. Nested Array Structures
Working with nested data structures:
```dql
// ❌ INCORRECT - accessing nested arrays
fetch records
| summarize total = sum(nested.array.values)  // If values is an array
```

---

## Common Scenarios

### Scenario 1: Misusing sum() in Field Transformation
**Problem:**
```dql
fetch dt.metrics
| fieldsAdd running_total = sum(value)  // ❌ Wrong context
```

**Solution:**
```dql
fetch dt.metrics
| summarize total = sum(value)  // ✅ Correct context
```

### Scenario 2: Summing Multi-Value Fields
**Problem:**
```dql
fetch logs
| parse content, "VALUES: [" -> "]" as values
| summarize total = sum(values)  // ❌ values might be parsed as array
```

**Solution:**
```dql
fetch logs
| parse content, "VALUES: [" -> "]" as values_string
| fieldsAdd values_array = split(values_string, ",")
| fieldsAdd sum_values = arraySum(values_array)  // ✅ Use arraySum()
```

### Scenario 3: Conditional Logic with Mixed Types
**Problem:**
```dql
fetch metrics
| summarize total = sum(
    if(isNull(single_value), backup_array, else: single_value)
  )  // ❌ Mixed types
```

**Solution:**
```dql
fetch metrics
| summarize total = sum(
    if(isNull(single_value), arraySum(backup_array), else: single_value)
  )  // ✅ Ensure consistent types
```

---

## Solutions

### Solution 1: Use Correct Aggregation Context

#### For Simple Aggregation
```dql
// ✅ CORRECT - Using sum() in summarize
fetch dt.metrics
| filter metric == "cpu.usage"
| summarize total_cpu = sum(value)
```

#### For Time Series Aggregation
```dql
// ✅ CORRECT - Using sum() in makeTimeseries
fetch dt.metrics
| makeTimeseries total = sum(value), interval: 5m
```

### Solution 2: Handle Arrays with arraySum()

#### Direct Array Summation
```dql
// ✅ CORRECT - Sum elements within an array
fetch data
| fieldsAdd array_total = arraySum(array_field)
```

#### Conditional Array Handling
```dql
// ✅ CORRECT - Handle both single values and arrays
fetch data
| fieldsAdd safe_sum = if(
    isArray(field),
    arraySum(field),
    else: field
  )
```

### Solution 3: Transform Arrays Before Aggregation

#### Expand Arrays First
```dql
// ✅ CORRECT - Expand array then sum
fetch data
| expand array_field
| summarize total = sum(array_field)
```

#### Extract Specific Array Elements
```dql
// ✅ CORRECT - Extract element before summing
fetch data
| fieldsAdd first_value = array_field[0]
| summarize total = sum(first_value)
```

---

## Correct Usage Patterns

### Pattern 1: Standard Aggregation
```dql
fetch dt.service.metrics
| filter service.name == "payment-service"
| summarize 
    total_requests = sum(request.count),
    total_duration = sum(request.duration)
```

### Pattern 2: Conditional Aggregation
```dql
fetch logs
| summarize 
    error_count = sum(if(level == "ERROR", 1, else: 0)),
    warning_count = sum(if(level == "WARNING", 1, else: 0))
```

### Pattern 3: Grouped Aggregation
```dql
fetch dt.metrics
| summarize total = sum(value), by: {service, environment}
| sort total desc
```

### Pattern 4: Time-based Aggregation
```dql
fetch dt.metrics
| makeTimeseries 
    hourly_sum = sum(value), 
    interval: 1h, 
    by: {metric.name}
```

---

## Working with Arrays

### Array Functions Reference

| Function | Purpose | Example |
|----------|---------|---------|
| `arraySum()` | Sum all elements in an array | `arraySum([1, 2, 3])` → 6 |
| `arrayAvg()` | Average of array elements | `arrayAvg([2, 4, 6])` → 4 |
| `arrayMin()` | Minimum value in array | `arrayMin([5, 2, 8])` → 2 |
| `arrayMax()` | Maximum value in array | `arrayMax([5, 2, 8])` → 8 |
| `arraySize()` | Count of array elements | `arraySize([1, 2, 3])` → 3 |

### Array Processing Examples

#### Example 1: Summing Nested Arrays
```dql
fetch custom.metrics
| fieldsAdd measurements = parseJson(data).measurements
| fieldsAdd total_measurements = arraySum(measurements)
| summarize avg_total = avg(total_measurements)
```

#### Example 2: Conditional Array Processing
```dql
fetch events
| fieldsAdd scores = if(
    event_type == "test",
    parseJson(data).test_scores,
    else: parseJson(data).practice_scores
  )
| fieldsAdd total_score = arraySum(scores)
```

#### Example 3: Array to Single Value Transformation
```dql
fetch metrics
| fieldsAdd values_array = split(values_string, ",")
| fieldsAdd numeric_array = arrayMap(values_array, toDouble(_))
| fieldsAdd sum_of_values = arraySum(numeric_array)
| summarize total = sum(sum_of_values)
```

---

## Best Practices

### 1. Type Checking
Always verify data types before aggregation:
```dql
fetch data
| fieldsAdd 
    is_array = isArray(field),
    is_number = isNumber(field)
| fieldsAdd safe_value = if(
    is_array,
    arraySum(field),
    else: if(is_number, field, else: 0)
  )
| summarize total = sum(safe_value)
```

### 2. Defensive Coding
Handle potential type mismatches:
```dql
fetch metrics
| fieldsAdd normalized_value = coalesce(
    if(isNumber(value), value),
    if(isArray(value), arraySum(value)),
    0
  )
| summarize total = sum(normalized_value)
```

### 3. Clear Data Transformation Pipeline
Structure your queries for clarity:
```dql
fetch raw_data
// Step 1: Parse and extract
| fieldsAdd parsed_data = parseJson(json_field)
// Step 2: Transform arrays
| fieldsAdd values = arraySum(parsed_data.values)
// Step 3: Aggregate
| summarize total = sum(values)
```

### 4. Use Appropriate Functions
Choose the right function for your data structure:

| Data Structure | Use Function | Context |
|---------------|--------------|---------|
| Single values | `sum()` | In `summarize` or `makeTimeseries` |
| Array field | `arraySum()` | In `fieldsAdd` |
| Mixed types | Conditional logic + appropriate function | Depends on type |

---

## Troubleshooting Checklist

### Step-by-Step Debugging

#### 1. Identify the Field Type
```dql
fetch your_data
| limit 1
| fieldsAdd 
    field_type = type(your_field),
    is_array = isArray(your_field),
    is_number = isNumber(your_field)
```

#### 2. Inspect Data Structure
```dql
fetch your_data
| limit 5
| fields your_field
// Examine the actual data structure
```

#### 3. Test Transformation
```dql
fetch your_data
| limit 10
| fieldsAdd test_transform = if(
    isArray(your_field),
    arraySum(your_field),
    else: your_field
  )
| fields your_field, test_transform
```

#### 4. Validate Aggregation
```dql
fetch your_data
| // Apply your transformation
| summarize test_sum = sum(transformed_field)
| fieldsAdd validation = isNotNull(test_sum)
```

### Common Fixes Quick Reference

| Error Scenario | Quick Fix |
|---------------|-----------|
| Array in sum() | Replace with `arraySum()` |
| sum() in fieldsAdd | Move to `summarize` |
| Mixed types in conditional | Ensure all branches return same type |
| Nested arrays | Use `expand` or array indexing |
| String that looks like array | Parse then use `arraySum()` |

---

## Related Functions and Concepts

### Aggregation Functions
- `avg()` - Average of values
- `min()` - Minimum value
- `max()` - Maximum value
- `count()` - Count of records
- `stddev()` - Standard deviation

### Array Functions
- `arraySum()` - Sum array elements
- `arrayAvg()` - Average of array
- `arraySize()` - Array length
- `arrayMap()` - Transform array elements
- `arrayFilter()` - Filter array elements

### Type Checking Functions
- `isArray()` - Check if value is array
- `isNumber()` - Check if value is number
- `isNull()` - Check if value is null
- `type()` - Get value type

---

## Summary

The "array type mismatch" error with `sum()` is typically resolved by:

1. **Using the correct context** - `sum()` belongs in `summarize` or `makeTimeseries`
2. **Handling arrays properly** - Use `arraySum()` for array fields
3. **Ensuring type consistency** - All conditional branches should return the same type
4. **Transforming data appropriately** - Convert arrays to single values before aggregation

Remember: `sum()` aggregates across records, while `arraySum()` sums within an array field.