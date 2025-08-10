# Dynatrace Query Language (DQL) - Comprehensive Guide

## Table of Contents
1. [Overview](#overview)
2. [Key Concepts](#key-concepts)
3. [Basic Syntax](#basic-syntax)
4. [Data Types](#data-types)
5. [Core Commands](#core-commands)
6. [Functions](#functions)
7. [Operators](#operators)
8. [Query Structure and Pipeline](#query-structure-and-pipeline)
9. [Common Use Cases](#common-use-cases)
10. [Best Practices](#best-practices)
11. [Examples](#examples)
12. [Learning Resources](#learning-resources)

## Overview

Dynatrace Query Language (DQL) is a powerful tool to explore your data and discover patterns, identify anomalies and outliers, create statistical modeling, and more based on data stored in Dynatrace Grail storage. DQL serves as the unified language for querying all observability data in Dynatrace, including logs, metrics, events, entities, and business data.

### Key Features
- **Flexible Schema**: DQL offers maximum flexibility because it is built for processing arbitrary event data, requiring no up-front description of the input data's schema contrary to relational databases like SQL tables
- **Pipeline-based Processing**: Uses pipe (`|`) operators to chain commands for data transformation
- **Strongly Typed**: The Dynatrace Query Language operates with strongly typed data: functions and operators accept only declared types of data
- **Grail Integration**: Direct access to all data stored in Dynatrace's Grail data lake

## Key Concepts

### Pipeline Architecture
A DQL query contains at least one or more commands, each of which returns tabular output containing records (lines or rows) and fields (columns). All commands are sequenced by a | (pipe). The data flows or is funneled from one command to the next.

### Data Processing Flow
1. **Fetch**: Load data from specified sources
2. **Filter**: Reduce records based on conditions
3. **Transform**: Add, modify, or parse fields
4. **Aggregate**: Summarize and group data
5. **Sort/Limit**: Order and restrict results

## Basic Syntax

### Query Structure
```dql
fetch [data_source]
| filter [condition]
| fields [field_list]
| summarize [aggregation] by [grouping]
| sort [field] [asc/desc]
| limit [number]
```

### Field Naming Rules
Field names referenced in a DQL statement must match the a-zA-Z0-9_. character set. Field names can not start with a number. Fields containing a character outside of a-zA-Z0-9_. must be escaped using the ` (Backtick) symbol.

### Time Ranges
```dql
// Using relative time
fetch logs from: now()-2h to: now()

// Using absolute time  
fetch logs from: "2024-01-01T00:00:00" to: "2024-01-02T00:00:00"
```

## Data Types

DQL supports the following strongly-typed data types:

### Basic Types
- **Boolean**: `true`, `false`
- **Long**: 64-bit signed integers (-2^63 to 2^63-1)
- **Double**: 64-bit floating-point numbers
- **String**: Text data with Unicode support
- **Binary**: Byte arrays

### Complex Types
- **Array**: A data structure that contains a sequence of values, each identified by index
- **Record**: A set of key-value pair data whose value can be any DQL data type
- **Duration**: Time intervals with specific units
- **Timestamp**: Point-in-time values with nanosecond precision
- **Timeframe**: A specific time frame with a start time and an end time as timestamps with nanosecond precision

### Type Conversion
```dql
// Boolean conversion
toBoolean("true")    // Returns true
toBoolean(1)         // Returns true  
toBoolean(0)         // Returns false

// Numeric conversion
toLong("123")        // Returns 123
toDouble("3.14")     // Returns 3.14
toString(123)        // Returns "123"
```

## Core Commands

### Data Loading Commands
- **`fetch`**: Loads data from the specified resource
- **`lookup`**: Loads data from the specified resource. It's used with lookup data
- **`data`**: Generates sample data during query runtime

### Filtering Commands
- **`filter`**: Reduces the number of records in a list by keeping only those records that match the specified condition
- **`filterOut`**: Removes records that match a specific condition
- **`search`**: Searches for records that match the specified search condition

### Field Operations
- **`fields`**: Keeps only specified fields
- **`fieldsAdd`**: Evaluates an expression and appends or replaces a field
- **`fieldsKeep`**: Keeps selected fields
- **`fieldsRemove`**: Removes specified fields from records

### Data Processing
- **`parse`**: Extracts structured data from text fields
- **`summarize`**: Groups and aggregates data
- **`sort`**: Orders records by specified fields
- **`limit`**: Restricts the number of returned records
- **`distinct`**: Removes duplicates from a list of records

### Time Series Operations
- **`makeTimeseries`**: Combines loading, filtering and aggregating metrics data into a time series output

## Functions

### Aggregation Functions
Aggregation functions compute results from a list of records:

- **`count()`**: Counts the total number of records
- **`sum(field)`**: Calculates total of numeric field
- **`avg(field)`**: Calculates the average value of a field for a list of records
- **`min(field)`**: Finds minimum value
- **`max(field)`**: Calculates the maximum value of a field for a list of records
- **`countDistinct(field)`**: Calculates the cardinality of unique values of a field for a list of records
- **`collectArray(field)`**: Collects the values of the provided field into an array

### String Functions
String functions allow you to create expressions that manipulate text strings in a variety of ways:

- **`contains(string, substring)`**: Searches the string expression for a substring
- **`startsWith(string, prefix)`**: Tests if string starts with prefix
- **`endsWith(string, suffix)`**: Checks if a string expression ends with a suffix
- **`concat(str1, str2, ...)`**: Concatenates the expressions into a single string
- **`substring(string, start, length)`**: Extracts portion of string
- **`upper(string)`**: Converts a string to uppercase
- **`lower(string)`**: Converts a string to lowercase

### Date/Time Functions
- **`now()`**: Current timestamp
- **`formatTimestamp(timestamp, format)`**: Formats timestamp as string
- **`parseTimestamp(string, format)`**: Converts string to timestamp
- **`duration(value, unit)`**: Creates duration values

### Conversion Functions
Conversion and casting functions convert the expression or value from one data type to another type:

- **`toString(value)`**: Converts to string
- **`toLong(value)`**: Returns long value if the value is long, otherwise, null
- **`toDouble(value)`**: Returns double value if the value is double, otherwise, returns null
- **`toBoolean(value)`**: Returns boolean value if the value is boolean, otherwise, returns null

### Conditional Functions
- **`if(condition, trueValue, falseValue)`**: Conditional expression
- **`ifNull(value, defaultValue)`**: Returns default if value is null
- **`coalesce(value1, value2, ...)`**: Returns first non-null value

## Operators

### Comparison Operators
Equals - Yields true if both operands are not null and equal to each other. Otherwise, false:

- **`==`**: Equal to
- **`!=`**: Not equals - Yields null, if one of the operands is null, or if the operands are not equal to each other
- **`<`**, **`<=`**, **`>`**, **`>=`**: Numeric comparisons

### Logical Operators
- **`and`**: Logical and (multiplication) - Yields true if both operands are true
- **`or`**: Logical or (addition) - Yields true if one of the operands is true, regardless of the other operand
- **`not`**: Logical negation

### Pattern Matching
- **`~`**: You can use the ~ operator in expressions to match the value of fields against a given search string. The performed comparison is case-insensitive and supports pattern matching using wildcards

### Array Operations
- **`in`**: The in comparison operator evaluates the occurrence of a value returned by the left side's expression within a list of values returned by the right side's DQL subquery

### Time Alignment
- **`@`**: The @ operator aligns a timestamp to the provided time unit. It rounds down the timestamp to the beginning of the time unit

## Query Structure and Pipeline

### Basic Pipeline Flow
```dql
fetch logs                          // Load data
| filter loglevel == "ERROR"       // Filter records
| fields timestamp, content        // Select fields
| parse content, "STATUS:status"   // Parse structured data
| summarize count() by status      // Aggregate
| sort count desc                  // Order results
| limit 10                         // Restrict output
```

### Complex Example
The following DQL query uses seven pipeline steps to get from raw log data to an aggregated table showing performance statistics for task execution:

```dql
fetch logs from: now()-1h
| filter endsWith(dt.system.bucket, "pgi.log")
| parse content, "LD IPADDR:ip ':' LONG:payload SPACE LD 'HTTP_STATUS' SPACE INT:http_status LD (EOL| EOS)"
| summarize 
    total_payload = sum(payload),
    failed_requests = countIf(http_status >= 400),
    successful_requests = countIf(http_status < 400)
    by ip, host.name
| sort total_payload desc
| limit 100
```

## Common Use Cases

### Log Analysis
```dql
// Find error patterns in logs
fetch logs
| filter loglevel == "ERROR"
| parse content, "ERROR SPACE LD:error_type SPACE LD"
| summarize error_count = count() by error_type
| sort error_count desc
```

### Performance Monitoring
```dql
// Analyze response times by endpoint
fetch events
| filter event.type == "HTTP_REQUEST"
| summarize 
    avg_response_time = avg(response_time),
    request_count = count()
    by endpoint
| sort avg_response_time desc
```

### Business Analytics
This example calculates the number of booking.process.started events. Intentionally only business days and hours (Mon-Fri, 8:00 AM to 5:00 PM) are accepted by the aggregation:

```dql
fetch events
| filter event.type == "booking.process.started"
| fieldsAdd 
    hour = formatTimestamp(timestamp, format:"hh"),
    day_of_week = formatTimestamp(timestamp, format:"EE")
| filterOut (day_of_week == "Sat" or day_of_week == "Sun") 
    or (toLong(hour) <= 08 or toLong(hour) >= 17)
| summarize bookings = count()
```

### Time Series Analysis
DQL provides dedicated commands such as makeTimeseries to aggregate a list of raw event records into a chartable timeseries:

```dql
fetch logs
| filter loglevel in {"ERROR", "WARN", "INFO"}
| makeTimeseries count(), by: {loglevel}, interval: 5m
```

## Best Practices

### Performance Optimization
Reduce the number of processed records by filtering the data using, for example, the filter or filterOut commands:

1. **Filter Early**: Apply filters immediately after fetch to reduce data volume
2. **Select Fields Early**: Select the amount of processed data by selecting fields early using the fields, fieldsKeep, or fieldsRemove commands
3. **Sort Late**: Sorting right after fetch and then continuing the query will reduce the query performance

### Recommended Command Order
Recommended order of commands for aggregating queries:

1. **Filter**: Reduce record count
2. **Field Selection**: Limit processed data
3. **Transformation**: Parse, add fields, etc.
4. **Aggregation**: Summarize data
5. **Sort**: Order final results
6. **Limit**: Restrict output size

### Query Structure Best Practices
```dql
// Good: Filter early, sort late
fetch logs
| filter timestamp >= now()-1h              // Filter first
| filter loglevel == "ERROR"               // Additional filters
| fields timestamp, content, host.name     // Select needed fields
| parse content, "ERROR:error_code"        // Transform data
| summarize count() by error_code, host.name // Aggregate
| sort count desc                          // Sort results
| limit 50                                 // Limit output

// Poor: Sort early, inefficient
fetch logs
| sort timestamp desc                      // Don't sort early
| filter loglevel == "ERROR"              // Should filter first
```

### Entity Queries
In simple queries that only filter entities, you can use classicEntitySelector or native DQL filters:

```dql
// Simple entity filtering
fetch entities
| filter in(id, classicEntitySelector("type(cloud_application_instance), entityName.startsWith(pod)"))

// For relationship queries, use classicEntitySelector
fetch entities  
| filter in(id, classicEntitySelector("type(host),toRelationship.isProcessOf(type(PROCESS_GROUP_INSTANCE),softwareTechnologies(JAVA))"))
```

## Examples

### Basic Queries

#### Simple Log Search
```dql
fetch logs
| filter contains(content, "HTTP")
| fields timestamp, content, loglevel
| sort timestamp desc
| limit 100
```

#### Count Events by Type
```dql
fetch events
| summarize event_count = count() by event.type
| sort event_count desc
```

### Intermediate Queries

#### Parse and Analyze Log Data
```dql
fetch logs
| filter contains(content, "HTTP")
| parse content, "SPACE INT:status_code SPACE"
| summarize 
    total_requests = count(),
    error_rate = countIf(status_code >= 400) * 100.0 / count()
    by host.name
| sort error_rate desc
```

#### Time-based Analysis
```dql
fetch logs
| filter loglevel == "ERROR"
| fieldsAdd hour = formatTimestamp(timestamp, format:"HH")
| summarize error_count = count() by hour
| sort hour asc
```

### Advanced Queries

#### Multi-step Data Processing
```dql
fetch logs from: now()-24h
| filter dt.system.bucket == "application.log"
| parse content, "LD 'user=' LD:user_id SPACE 'action=' LD:action SPACE 'duration=' LONG:duration"
| filterOut isNull(user_id)
| summarize 
    total_actions = count(),
    avg_duration = avg(duration),
    unique_users = countDistinct(user_id)
    by action
| fieldsAdd efficiency_score = total_actions / avg_duration
| sort efficiency_score desc
| limit 20
```

#### Cross-data Analysis
```dql
fetch events
| filter event.type == "user.session.started"
| lookup [
    fetch entities
    | filter entity.type == "APPLICATION"
    | fields entity.id, display_name
], sourceField:app_id, lookupField:entity.id
| summarize sessions = count() by display_name
| sort sessions desc
```

## Learning Resources

### Official Documentation
- **Main DQL Guide**: Read how to use DQL queries to get started or explore the language
- **Language Reference**: Complete syntax and function documentation
- **Interactive Tutorials**: You can learn DQL through hands-on experience with interactive tutorials in the Learn DQL App

### Community Resources
- **Dynatrace Community Forum**: DQL discussions and examples
- **Video Tutorials**: Practical DQL examples and use cases
- **Best Practices Documentation**: Performance optimization guides

### Getting Started
1. **Start Simple**: The best way to learn DQL is to start with some basic queries
2. **Use the Learn DQL App**: You can use the app, if you are a customer with access to Dynatrace SaaS environment or if you are a registered member of the Dynatrace Community
3. **Practice with Real Data**: Begin with basic fetch and filter operations
4. **Build Complexity Gradually**: Add parsing, aggregation, and advanced functions
5. **Study Examples**: Learn from documented use cases and community examples

### Trial Access
You can also sign up for a 15 day free trial to try out the app to access Dynatrace and practice DQL queries.

---

**Note**: This guide covers the core concepts and capabilities of DQL. For the most current documentation and detailed function references, always consult the official Dynatrace documentation at [docs.dynatrace.com](https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-query-language).