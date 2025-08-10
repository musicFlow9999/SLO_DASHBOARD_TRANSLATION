# Dynatrace Pattern Language (DPL) - Comprehensive Documentation

## Overview

Dynatrace Pattern Language (DPL) is a pattern language that allows you to describe patterns using matchers, where a matcher is a mini-pattern that matches a certain type of data. 

### Key Concepts
- **Matcher**: A mini-pattern that matches a certain type of data
  - Example: `INTEGER` (or `INT`) matches integer numbers
  - Example: `IPADDR` matches IPv4 or IPv6 addresses
- Matchers are available to handle all kinds of data types

### Use Cases for DPL
DPL can be used to:
- Parse and extract data from various sources
- Define patterns for log processing
- Create structured data from unstructured text
- Build custom parsing rules for different data formats

### Tool Support
For instant feedback on the effectiveness and coverage of your patterns for your specific use case, use **DPL Architect** (`/docs/discover-dynatrace/references/dynatrace-pattern-language/dpl-architect`).

## Pattern Interpretation

### Basic Rules
- Patterns are interpreted **from left to right**
- The following are ignored in patterns:
  - Extra whitespaces
  - Line breaks
  - Comments

### Pattern Writing Examples

#### One-liner Format
```dpl
INT ' ' IPADDR:ip EOL
```

#### Explanatory Format (with comments)
```dpl
/* this pattern expects an integer number and an IP address
separated by single space in each line */
INT         //an integer
' '         //followed by single space
IPADDR:ip   //followed by IPv4 or IPv6 address, extracted as a new field, `ip`
EOL         //line is terminated with line feed character
```

## Data Extraction Principles

### Selective Data Extraction
- Not all data elements in the input are necessarily needed for analysis
- Field separators or end-of-record markers are useful only for parsing
- All matchers in a defined pattern must match
- Only a subset of matchers may extract (parse) data

### Export Names
A matcher will extract data **only when it has been assigned an export name**:
- Export name is an arbitrary name of your choice
- It becomes the name of the field used in query statements
- Syntax: `MATCHER:export_name`

## Pattern Structure

### Components of a DPL Pattern
A DPL pattern consists of one or more matcher expressions that can be:
- Separated by whitespace
- Separated by commas
- Separated by newlines

For a complete reference, see the **DPL Grammar page** (`/docs/discover-dynatrace/references/dynatrace-pattern-language/log-processing-grammar`).

### Types of Matcher Expressions
A matcher expression can be any of the following:
1. Basic matchers
2. Grouped matchers
3. Configured matchers
4. Modified matchers

### Grouping Matcher Expressions
Matcher expressions can be grouped using various techniques to create more complex patterns.

## Matcher Expression Syntax

### Element Order (Right to Left)
The correct order for a matcher expression is:
```
[lookaround] MATCHER_EXPR ['(' configuration ')'] [quantifier] [mod_optional] [':'export_name]
```

**Important Notes:**
- Square brackets indicate optional elements
- Whitespace characters and newlines are allowed between elements
- Placing elements in a different order will cause a syntax error
- Example of incorrect order: placing `mod_optional` after `export_name`

### Matcher Expression Components

#### 1. Configuration
- Some matchers allow configuration to specify their behavior
- Example: A timestamp needs an expected format definition
- Syntax: `MATCHER(configuration)`
- Link: `/docs/discover-dynatrace/references/dynatrace-pattern-language/log-processing-modifiers`

#### 2. Quantifiers
- Most matchers and groupings can have quantifiers
- Tells the engine how many times to try matching
- Link: `/docs/discover-dynatrace/references/dynatrace-pattern-language/log-processing-modifiers`

#### 3. Optional Modifiers
- All matchers and groupings can be declared optional
- If element is missing, engine outputs NULL and continues
- Link: `/docs/discover-dynatrace/references/dynatrace-pattern-language/log-processing-modifiers`

#### 4. Export Names
- All matchers and groupings can be assigned an export name
- This is the name of the field exposed to the query layer
- A matcher without an export name still matches but isn't visible in queries
- Link: `/docs/discover-dynatrace/references/dynatrace-pattern-language/log-processing-modifiers`

#### 5. Lookaround
- All matchers and groupings can "look around" (backward or forward)
- Mainly used for decision-making and conditional branching
- Link: `/docs/discover-dynatrace/references/dynatrace-pattern-language/log-processing-modifiers`

## Step-by-Step Pattern Matching Example

### Sample Data
Comma-separated record terminated with line feed character:
```
1,alice,192.168.1.12,bob,10.6.24.183,mallory,192.168.1.3
```

### Pattern Expression
```dpl
INT:seq
','
LD:uname
','
IPADDR:user_ip
EOL
```

Where:
- `INT:seq` - Integer matcher with export name "seq"
- `','` - Literal comma matcher
- `LD:uname` - Letter/digit matcher with export name "uname"
- `IPADDR:user_ip` - IP address matcher with export name "user_ip"
- `EOL` - End of line matcher

### Matching Process

#### Step 1: Match Integer
- Engine starts at first byte of input data
- Finds `1` which matches `INT:seq`
- Moves to next byte

#### Step 2: Match Comma
- Finds comma, which doesn't match integer
- `INT:seq` matcher completes, converting `1` to integer
- Next matcher `','` is selected and matches current position

#### Step 3: Match Username
- Data pointer moves to first letter of `alice`
- Matcher `[a-zA-Z0-9]*:uname` with quantifier `*` consumes variable bytes
- Continues matching until non-matching byte (comma after `alice`)

#### Step 4: Match Second Comma
- Matcher `','` matches the comma after username

#### Step 5: Match IP Address
- Data pointer at beginning of `192.168.1.12`
- `IPADDR:user_ip` matches and advances 12 bytes

#### Step 6: Match End of Line
- Engine matches newline character with `EOL`

#### Step 7: Continue Process
- Data pointer advances to next byte
- Pattern iterator resets
- Cycle continues with first matcher against current data position

### Error Handling
If the engine encounters unmatched data:
1. Resets the pattern iterator
2. Marks the byte as unmatched
3. Moves to the next byte
4. Continues until match found or no more data

### Expected Output
The structured data output would be:
```
seq: 1
uname: alice
user_ip: 192.168.1.12

seq: 2
uname: bob
user_ip: 10.6.24.18

seq: 3
uname: mallory
user_ip: 192.168.1.3
```

## Best Practices

### Pattern Design
1. **Start Simple**: Begin with basic matchers and add complexity as needed
2. **Use Comments**: Document complex patterns for maintainability
3. **Test Incrementally**: Use DPL Architect to test patterns with sample data
4. **Export Selectively**: Only export fields needed for queries

### Performance Considerations
1. **Minimize Backtracking**: Design patterns to match efficiently
2. **Use Specific Matchers**: Prefer specific matchers over generic ones when possible
3. **Optimize Quantifiers**: Be careful with greedy quantifiers that might consume too much

### Common Patterns

#### Log File Parsing
```dpl
TIMESTAMP('yyyy-MM-dd HH:mm:ss'):timestamp
' ['
LD:level
'] '
DATA:message
EOL
```

#### CSV File Processing
```dpl
(DATA:field ',')*
DATA:last_field
EOL
```

#### IP Address and Port
```dpl
IPADDR:ip
':'
INT:port
```

## Related Documentation

- **DPL Grammar**: `/docs/discover-dynatrace/references/dynatrace-pattern-language/log-processing-grammar`
- **DPL Architect**: `/docs/discover-dynatrace/references/dynatrace-pattern-language/dpl-architect`
- **Modifiers Reference**: `/docs/discover-dynatrace/references/dynatrace-pattern-language/log-processing-modifiers`

## Summary

Dynatrace Pattern Language provides a powerful and flexible way to:
- Parse structured and semi-structured data
- Extract relevant fields for analysis
- Handle various data formats with a single pattern language
- Create reusable parsing patterns for log processing

The key to effective DPL usage is understanding:
- Matcher types and their capabilities
- Proper syntax and element ordering
- When to extract vs. just match data
- How the pattern matching engine processes input sequentially

With these concepts, you can create efficient patterns for parsing virtually any text-based data format.