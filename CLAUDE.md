# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Go library that parses RSQL (RESTful Service Query Language) strings and converts them into database query formats. RSQL is based on FIQL and provides a URI-friendly syntax for filtering API entries.

The library supports both MongoDB (JSON format) and MySQL (SQL WHERE clause format) out of the box. The core architecture is designed to be extensible to other database formats.

## Development Commands

### Running Tests
```bash
go test -v
```

### Running a Specific Test
```bash
go test -v -run TestName
```

For example:
```bash
go test -v -run TestParser_ProcessMongo
```

### Running Tests with Coverage
```bash
go test -v -cover
```

## Core Architecture

### Parser Design

The library uses a **recursive descent parser** that processes RSQL strings in stages:

1. **OR splitting** (`,` separator) - Top-level parsing splits by OR operators
2. **AND splitting** (`;` separator) - Each OR block is split by AND operators
3. **Parentheses handling** - Nested expressions are recursively processed
4. **Operation parsing** - Individual comparisons are extracted and formatted

#### Key Components

**Parser struct** (rsql.go:32-38):
- `operators`: Array of operator definitions with custom formatters
- `andFormatter`: Function to combine AND operations
- `orFormatter`: Function to combine OR operations
- `keyTransformers`: Functions to transform field names (e.g., case conversion)

**Operator struct** (rsql.go:23-30):
- `Operator`: String pattern matching regex `([!=])[^=()]*=`
- `Formatter`: Function that takes (key, value) and returns formatted query string

### Parsing Flow

The `Process()` function (rsql.go:197-284) executes this flow:
1. Split input by `,` (OR) using `findORs()`
2. For each OR block, split by `;` (AND) using `findANDs()`
3. For each AND block, check for outer parentheses using `findOuterParentheses()`
4. Recursively process parenthetical content
5. Parse individual operations by extracting operator, key, and value
6. Apply key transformers
7. Validate against allowed/forbidden keys
8. Format using operator's formatter function
9. Combine results using AND/OR formatters

### Special Character Handling

The library handles escaped RSQL special characters:
- `\(`, `\)`, `\,`, `\;`, `\=` are encoded to prevent parser confusion
- Uses URL-encoded representations: `%5C%28`, `%5C%29`, etc.
- See `specialEncode` map (rsql.go:12-18)

### Parentheses Detection Algorithm

`findOuterParentheses()` (rsql.go:380-440) distinguishes between:
- **Grouping parentheses**: Used for logical grouping like `(a==1,b==2)`
- **List parentheses**: Used in operators like `=in=(1,2,3)`

It tracks whether parentheses appear within an operation (between `=` or `!` and the next `,` or `;`) to determine their purpose.

## Supported Operators

### Built-in MongoDB Operators

Defined in `Mongo()` function (rsql.go:61-141):

- `==`: Equal (formats as `{ "key": value }`)
- `!=`: Not equal (uses `$ne`)
- `=gt=`: Greater than (uses `$gt`)
- `=ge=`: Greater than or equal (uses `$gte`)
- `=lt=`: Less than (uses `$lt`)
- `=le=`: Less than or equal (uses `$lte`)
- `=in=`: In list (uses `$in`, removes parentheses)
- `=out=`: Not in list (uses `$nin`, removes parentheses)

### Built-in MySQL Operators

Defined in `MySQL()` function (rsql.go:143-224):

- `==`: Equal (formats as `key = value`)
- `!=`: Not equal (formats as `key != value`)
- `=gt=`: Greater than (formats as `key > value`)
- `=ge=`: Greater than or equal (formats as `key >= value`)
- `=lt=`: Less than (formats as `key < value`)
- `=le=`: Less than or equal (formats as `key <= value`)
- `=in=`: In list (formats as `key IN (value)`, removes parentheses)
- `=out=`: Not in list (formats as `key NOT IN (value)`, removes parentheses)

### Extending with Custom Operators

Custom operators can be added via `WithOperators()`:
- Must match regex pattern `([!=])[^=()]*=`
- Provide a formatter function that receives (key, value) and returns the query string
- Can handle list operations by inspecting value for parentheses

Example from README: `=ex=` for exists, `=all=` for MongoDB $all operator

## Key Features

### Key Transformers

Applied via `WithKeyTransformers()` - functions that transform field names before formatting.
Common use case: Converting between naming conventions (camelCase ↔ snake_case, uppercase/lowercase).
Applied in order during operation parsing (rsql.go:254-257).

### Field Validation

Two mechanisms via `ProcessOptions`:
- `SetAllowedKeys()`: Whitelist of acceptable field names
- `SetForbiddenKeys()`: Blacklist of prohibited field names

Validation occurs after key transformation (rsql.go:259-264).

## Testing Strategy

Tests in `rsql_test.go` use table-driven patterns with these categories:

1. **Operator tests**: Verify each operator formats correctly
2. **Logical combination tests**: AND/OR precedence and nesting
3. **Parentheses tests**: Grouping vs. list parentheses
4. **Custom operator tests**: Extensibility validation
5. **Process options tests**: Key validation and transformation
6. **Helper function tests**: `findParts()`, `findORs()`, `findANDs()`, `findOuterParentheses()`

Test structure uses:
```go
tests := []struct {
    name    string
    s       string  // RSQL input
    want    string  // Expected output
    wantErr bool
    // ... test-specific fields
}
```

## Important Implementation Notes

1. **No go.mod**: This project doesn't use Go modules (appears to be pre-modules or intentionally avoiding them)

2. **Regex limitations**: The operator regex `([!=])[^=()]*=` requires operators to start with `!` or `=` and end with `=`

3. **Database support**: The library supports both MongoDB and MySQL out of the box:
   - `rsql.Mongo()`: Produces MongoDB JSON query strings (e.g., `{ "$or": [ ... ] }`)
   - `rsql.MySQL()`: Produces SQL WHERE clause strings (e.g., `(a = 1 OR b = 2)`)
   - Additional databases can be supported by creating similar formatter functions

4. **No validation of values**: The parser doesn't validate that values are valid for their types (e.g., numeric operators with string values). This is left to the database layer.

5. **String-based output**: The parser returns formatted query strings, not structured objects. This is intentional for flexibility but means query syntax errors might not be caught until execution.
