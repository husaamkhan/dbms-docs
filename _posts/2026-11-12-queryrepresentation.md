---
title: "Query Representation"
date: 2026-11-06
---

## Contents

- [Purpose](#purpose)
- [Representation](#representation)
  - [Queries](#queries)
  - [Expressions](#expressions)
    - [Values](#values)
    - [Comparison Operators](#comparison-operators)
- [Examples](#examples)
- [Memory Management](#memory-management)

## Purpose
The query representation provides a common structure for representing queries. It supports the project's eventual transition from Relational Algebra to SQL and allows for an easier execution engine implementation.

Since the project initially began as an assignment focused on developing a Relational Algebra frontend, Relational Algebra will continue to be used while developing a prototype of the database system. As the project expands, both Relational Algebra and SQL will be able to use the same query executor by translating their respective parse trees into a shared query representation.

The query representation also explicitly defines the operations required to execute a query and how they are composed. This eliminates the need for the executor to interpret frontend-specific instructions or determine the sequence of operations required for execution, allowing it to focus solely on executing the provided operations.

## Representation
The representation consists of a Query structure and an Expression structure. Queries describe the relational operations that make up a query, while expressions describe conditions and values used by those operations.

### Queries
Relational operations are represented using the Query structure, with the QueryType enumeration identifying the operation being represented.

The following operations are supported:

| Operation | Description |
|---|---|
| `QUERY_TABLE` | References a base relation. |
| `QUERY_PROJECT` | Selects specific columns from an input relation. |
| `QUERY_FILTER` | Filters an input relation based on a condition. |
| `QUERY_JOIN` | Combines two relations based on a join condition. |
| `QUERY_CROSS_JOIN` | Produces the Cartesian product of two relations. |
| `QUERY_UNION` | Combines the tuples of two relations. |
| `QUERY_INTERSECT` | Returns tuples common to two relations. |
| `QUERY_EXCEPT` | Returns tuples present in the left relation but not the right. |

The Query structure's anonymous union stores the data associated with that operation. Each operation stores the information required to describe its inputs and any additional parameters.

Queries can reference other queries as their inputs, allowing multiple relational operations to be composed into a single recursive structure.  

### Expressions

Expressions are represented using the Expression structure, with the ExpressionType enumeration identifying the type of expression.

The following expression types are supported:

| Expression | Description |
|---|---|
| `EXPR_COLUMN` | References a column by name. |
| `EXPR_LITERAL` | Represents a literal value. |
| `EXPR_COMPARISON` | Compares two expressions using a comparison operator. |
| `EXPR_AND` | Performs a logical AND between two expressions. |
| `EXPR_OR` | Performs a logical OR between two expressions. |
| `EXPR_NOT` | Negates an expression. |

Like the Query structure, Expression structures also use an anonymous union to store associated data, and can recursively contain other expressions. This allows complex conditions to be represented through combinations of comparisons and logical operators.

#### Values
The Value structure represents a literal value used within an expression. Its type field identifies the value's data type, while its union stores the corresponding value.

The ValueType enumeration currently supports:

- `VALUE_FLOAT`: A floating-point number, stored as a double.
- `VALUE_INT`: An integer, stored as a long long.
- `VALUE_STR`: A string, stored as a character pointer.

#### Comparison Operators
The ComparisonOperator enumeration identifies the comparison performed by an EXPR_COMPARISON expression.

The supported operators are:

| Operator | Meaning |
|---|---|
| `COMP_EQUAL` | Equal to |
| `COMP_NOT_EQUAL` | Not equal to |
| `COMP_LESS` | Less than |
| `COMP_GREATER` | Greater than |
| `COMP_LESS_EQUAL` | Less than or equal to |
| `COMP_GREATER_EQUAL` | Greater than or equal to |

### Examples
The following examples illustrate how relational operations and expressions are composed into query representations.


#### Projection
Consider the following query:
```
project[Name, Age](Students)
```

The corresponding representation is:
```
QUERY_PROJECT
├── input: QUERY_TABLE
│   └── name: "Students"
└── column_list:
    ├── "Name"
    └── "Age"
```

#### Filtering

Consider the following query:
```
select[Age > 18](Students)
```

The corresponding representation is:
```
QUERY_FILTER
├── input: QUERY_TABLE
│   └── name: "Students"
└── condition: EXPR_COMPARISON
    ├── operator: COMP_GREATER
    ├── left: EXPR_COLUMN
    │   └── column_name: "Age"
    └── right: EXPR_LITERAL
        └── value:
            ├── type: VALUE_INT
            └── integer: 18
```

#### Combining Operations
Consider the following query:
```
project[Name](select[Age > 18](Students))
```

The corresponding representation is:
```
QUERY_PROJECT
├── input: QUERY_FILTER
│   ├── input: QUERY_TABLE
│   │   └── name: "Students"
│   └── condition: EXPR_COMPARISON
│       ├── operator: COMP_GREATER
│       ├── left: EXPR_COLUMN
│       │   └── column_name: "Age"
│       └── right: EXPR_LITERAL
│           └── value:
│               ├── type: VALUE_INT
│               └── integer: 18
└── column_list:
    └── "Name"
```

#### Compound Conditions
Consider the following query:
```
select[Age > 18 and Grade >= 80](Students)
```

The corresponding representation is:
```
QUERY_FILTER
├── input: QUERY_TABLE
│   └── name: "Students"
└── condition: EXPR_AND
    ├── left: EXPR_COMPARISON
    │   ├── operator: COMP_GREATER
    │   ├── left: EXPR_COLUMN
    │   │   └── column_name: "Age"
    │   └── right: EXPR_LITERAL
    │       └── value:
    │           ├── type: VALUE_INT
    │           └── integer: 18
    └── right: EXPR_COMPARISON
        ├── operator: COMP_GREATER_EQUAL
        ├── left: EXPR_COLUMN
        │   └── column_name: "Grade"
        └── right: EXPR_LITERAL
            └── value:
                ├── type: VALUE_INT
                └── integer: 80
```

## Memory Management

The query representation uses arena allocation to simplify memory management. Query representations are composed of recursively nested structures that may reference numerous other queries, expressions, and values. Releasing these structures individually would require recursively traversing the representation and freeing each allocation.

An arena allows the memory associated with a representation to be released together. Its lifetime can be tied to query execution, with the arena being reset or destroyed once the representation is no longer needed.
