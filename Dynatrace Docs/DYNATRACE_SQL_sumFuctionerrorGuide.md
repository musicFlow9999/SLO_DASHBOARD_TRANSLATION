# DQL sum() Function Error Guide - Array Type Mismatch

## Error Message
```
This parameter of the `sum()` function should be a number or a duration, but was an array
```

## Table of Contents
- [Understanding the Error](#understanding-the-error)
- [Official Documentation Context](#official-documentation-context)
- [Root Causes](#root-causes)
- [Solutions Based on Dynatrace Documentation](#solutions-based-on-dynatrace-documentation)
- [Aggregation Functions in DQL](#aggregation-functions-in-dql)
- [Array Functions in DQL](#array-functions-in-dql)
- [Correct Usage Patterns](#correct-usage-patterns)
- [Data Types and Type Compatibility](#data-types-and-type-compatibility)
- [Best Practices from Documentation](#best-practices-from-documentation)

---

## Understanding the Error

Based on the official Dynatrace documentation, this error occurs because the `sum()` function has specific type requirements that are being violated.

### Official Type Requirements for sum()

According to the Dynatrace aggregation functions documentation:
- **"The sum function allows numeric expressions and duration expressions. If you mix types, the result is null."**
- **"The sum of two numeric expressions results in a double data type"**
- The `sum()` function calculates the sum of a field for a list of records

### What sum() Does NOT Accept
- Arrays (which are a distinct data type in DQL)
- Strings
- Booleans
- Records
- Mixed incompatible types

---

## Official Documentation Context

### From Aggregation Functions Documentation

The Dynatrace documentation states:
- **"Aggregation functions compute results from a list of records"**
- **"You can use the aggregation functions listed on this page with the summarize command"**
- Nine aggregation functions are available with `makeTimeseries`: min, max, sum, avg, count, countIf, percentile, countDistinctExact, and countDistinctApprox

### From Metric Commands Documentation

For the `timeseries` command:
- **"Six aggregation functions are available to use with the timeseries command"**
- **"These functions are: sum - Calculates the sum of the metric in each time slot"**

---

## Root Causes

### 1. Using sum() Outside Aggregation Context

Per the documentation, `sum()` must be used with:
- `summarize` command
- `makeTimeseries` command  
- `timeseries` command

**Documentation Quote**: "You can use the aggregation functions listed on this page with the summarize command"

### 2. Array Field as Input

DQL treats arrays as a distinct data type. From the data types documentation:
- **"A data structure that contains a sequence of values, each identified by index"**
- Arrays require specific array functions, not aggregation functions

### 3. Incorrect Function Selection

The documentation clearly distinguishes between:
- **Aggregation functions** (like `sum()`) - work across records
- **Array functions** (like `arraySum()`) - work within arrays

---

## Solutions Based on Dynatrace Documentation

### Solution 1: Use sum() in Correct Context

#### With summarize Command
From the documentation example:
```dql
fetch logs
| summarize sum(value)
```

**Documentation Quote**: "The below example uses the summarize command and the sum aggregation function to sum the field value"

#### With makeTimeseries Command
```dql
fetch bizevents
| makeTimeseries total = sum(amount), interval: 1h
```

#### With timeseries Command
```dql
timeseries failed = sum(dt.requests.failed)
```

### Solution 2: Use arraySum() for Arrays

From the array functions documentation:
- **"Returns the sum of an array"**
- **"The data type of the returned value is double or long"**

**Official Example**:
```dql
data record(a = array(2, 3, 7, 7, 1))
| fieldsAdd arraySum(a)
// Result: 20
```

### Solution 3: Use Iterative Expressions

The documentation provides this advanced pattern:
```dql
data record(a = array(2, 2)), record(a = array(7, 1))
| summarize sum(a[])
// Result: [9, 3]
```

**Documentation Quote**: "The following example uses 'summarize' with an iterative expression in the 'sum' aggregation function to calculate the element-wise sum of arrays"

### Solution 4: Expand Arrays Before Aggregation

From the structuring commands documentation:
```dql
fetch data
| expand array_field
| summarize total = sum(array_field)
```

**Documentation Quote**: "Expands an array into separate records. This command takes an array, and for each incoming record, produces as many new records as there are elements in the array"

---

## Aggregation Functions in DQL

### Official List from Documentation

| Function | Description | Supported in |
|----------|-------------|--------------|
| `sum()` | Calculates the sum of a field for a list of records | summarize, makeTimeseries, timeseries |
| `avg()` | Calculates the average value | summarize, makeTimeseries, timeseries |
| `min()` | Calculates the minimum value | summarize, makeTimeseries, timeseries |
| `max()` | Calculates the maximum value | summarize, makeTimeseries, timeseries |
| `count()` | Counts the total number of records | summarize, makeTimeseries, timeseries |
| `countIf()` | Counts records matching a condition | summarize, makeTimeseries |

### Type Compatibility Matrix

From the documentation:
- **"If you mix two data types, the result is null, unless you mix data for which combinations are allowed, such as long and double"**
- **"You will also get the null result for operations not covered by a given function, for example the sum() of two boolean expressions"**

---

## Array Functions in DQL

### Official Array Functions from Documentation

| Function | Description | Example |
|----------|-------------|---------|
| `arraySum()` | Returns the sum of an array | `arraySum([2, 3, 7])` → 12 |
| `arrayAvg()` | Returns the average of an array | `arrayAvg([2, 4, 6])` → 4 |
| `arrayMin()` | Returns the smallest number | `arrayMin([5, 2, 8])` → 2 |
| `arrayMax()` | Returns the biggest number | `arrayMax([5, 2, 8])` → 8 |
| `arraySize()` | Returns the number of elements | `arraySize([1, 2, 3])` → 3 |

### Official arraySum() Documentation

**Function Definition**: "Returns the sum of an array. Values that are not numeric are ignored. Returns 0 if there is no matching element."

**Return Type**: "The data type of the returned value is double or long"

---

## Correct Usage Patterns

### Pattern 1: Standard Aggregation (From Documentation)

```dql
fetch dt.service.metrics
| summarize 
    total_requests = sum(request.count),
    total_duration = sum(request.duration)
```

### Pattern 2: Time Series Aggregation

```dql
timeseries cpu_sum = sum(dt.host.cpu.usage), 
           by: dt.entity.host, 
           interval: 1h
```

### Pattern 3: Array Handling

```dql
fetch data
| fieldsAdd array_total = arraySum(measurements)
| summarize avg_total = avg(array_total)
```

### Pattern 4: Conditional Aggregation

From documentation example:
```dql
fetch logs
| summarize {
    errors = countIf(loglevel == "ERROR"),
    warnings = countIf(loglevel == "WARN")
  }, by: {dt.entity.host}
```

---

## Data Types and Type Compatibility

### From Official DQL Data Types Documentation

**Key Quote**: "The Dynatrace Query Language operates with strongly typed data: the functions and operators accept only declared types of data"

### Array Data Type
- **Definition**: "A data structure that contains a sequence of values, each identified by index"
- **Comparison**: "Only the equals operator == can be directly used on arrays"
- Arrays cannot be directly used with numeric aggregation functions

### Type Conversion Rules
From the documentation:
- Numeric types (long, double) can be mixed in sum()
- Duration types can be summed
- Mixing incompatible types results in null
- Arrays require explicit array functions or expansion

---

## Best Practices from Documentation

### 1. Use Appropriate Commands

**Aggregation Commands** (from documentation):
- `summarize` - Groups records and aggregates them
- `makeTimeseries` - Creates timeseries from data stream
- `timeseries` - Combines loading, filtering and aggregating metrics

### 2. Handle Arrays Properly

Three documented approaches:
1. **Use array functions**: `arraySum()`, `arrayAvg()`, etc.
2. **Expand arrays**: Use `expand` command to create records
3. **Iterative expressions**: Use `sum(array[])` in summarize

### 3. Check Data Types

The documentation emphasizes:
- **"DQL operates with strongly typed data"**
- Use type checking functions when needed
- Convert types explicitly when necessary

### 4. Follow Command Pipeline Structure

From the documentation:
- **"All commands are sequenced by a | (pipe)"**
- **"The data flows or is funneled from one command to the next"**
- Aggregation functions belong in specific command contexts

---

## Troubleshooting Based on Documentation

### Step 1: Verify Command Context
Ensure `sum()` is used within:
```dql
| summarize ...
| makeTimeseries ...
timeseries ...  // As starting command
```

### Step 2: Check Field Type
If dealing with arrays, switch to:
```dql
| fieldsAdd result = arraySum(array_field)
```

### Step 3: Consider Expansion
For complex array processing:
```dql
| expand array_field
| summarize total = sum(array_field)
```

### Step 4: Use Iterative Expressions
For element-wise operations:
```dql
| summarize element_sums = sum(arrays[])
```

---

## Summary

Based on official Dynatrace documentation:

1. **The error occurs** because `sum()` expects numbers or durations, not arrays
2. **`sum()` must be used** within `summarize`, `makeTimeseries`, or `timeseries` commands
3. **For array summation**, use `arraySum()` function instead
4. **DQL is strongly typed** - functions accept only declared compatible types
5. **Multiple solutions exist** including array functions, expansion, and iterative expressions

All information in this guide is directly sourced from official Dynatrace DQL documentation, ensuring accuracy and reliability for troubleshooting this specific error.