---
title: "Writing a Crappy SQL Database from Scratch"
date: 2024-11-10
author: "agao"
ShowToc: true
tags: ["database", "sql"]
---

Surprisingly, I've never had to write much SQL (i.e. any) in any of my previous jobs or current job before. So I thought it would be fun to get my hands dirty and try implementing a subset of SQL with an in-memory backend. It mostly follows from [this blog post](https://notes.eatonphil.com/database-basics.html).

But, I also wasn't super interested in writing my own SQL parser, so I decided to use a fork of Vitess' [sqlparser library](https://github.com/xwb1989/sqlparser). This meant that I could focus on implementing the fun parts, instead of worrying about the boring parts like lexing, binding power, and whatnot.

Another alternative I considered was [`go-mysql-server`](https://github.com/dolthub/go-mysql-server?tab=readme-ov-file), which allows you to write your own custom backend implementation, and again ignore all the parsing and execution logic.

> Here I'm only noting specific things I discovered while implementing my database, rather than being a step-by-step tutorial on how to write a SQL database from scratch. For a more comprehensive experience, check out Phil Eaton's [series of blogs](https://notes.eatonphil.com/tags/databases.html).

> I also relied heavily on the [`octosql`](https://github.com/cube2222/octosql) implementation for inspiration on how to handle the parsed outputs from `sqlparser`.

Like always, the source code can be found [here](https://github.com/algao1/crumbs/tree/master/badsql).

## Data Representation

I ended up sticking with the format chosen in the blog which was a standard row-based format, with each row comprising of multiple 'cells' -- representing the actual values.

```go
type Cell interface {
    AsText() string
    AsInt() int32
    AsBool() bool
}
```

One alternative would be to go with a [columnar format](https://airbyte.com/data-engineering-resources/columnar-storage) which might improve performance, but is harder to reason about for the purposes of implementing a SQL database.

As mentioned earlier, we'll be using an in-memory backend, so each `MemoryCell` is essentially just a slice of bytes, with some helper functions to convert to the desired type.

```go
type MemoryCell []byte

func (c MemoryCell) AsInt() int32 {
    var i int32
    err := binary.Read(bytes.NewBuffer(c), binary.BigEndian, &i)
    if err != nil {
        panic(err)
    }
    return i
}

func (c MemoryCell) AsText() string {
    return string(c)
}

func (c MemoryCell) AsBool() bool {
    return len(c) != 0
}
```

We can likewise swap this out for a number of different backends, like a B-tree or even a CSV file.

Continuing on, we define a `table` struct that maps to a SQL table, complete with fields for column names, column types, and the rows of data. Our memory backend is then just a collection of tables.

```go
type table struct {
    columns     []string
    columnTypes []ColumnType
    rows        [][]MemoryCell
}

type MemoryBackend struct {
    tables map[string]*table
}
```

## Create & Insert

Creating a table is relatively simple, we just need to get the table name and ensure that it's unique. Then we need to iterate over the query and parse the column names and types.

```go
func (b *MemoryBackend) CreateTable(stmt *sqlparser.DDL) error {
    // ...
    t := table{
        columns:     make([]string, 0),
        columnTypes: make([]ColumnType, 0),
        rows:        make([][]MemoryCell, 0),
    }
    for _, col := range stmt.TableSpec.Columns {
        columnType, err := parseColumnType(col.Type)
        if err != nil {
            return fmt.Errorf("unable to parse column type: %w", err)
        }
        t.columns = append(t.columns, col.Name.CompliantName())
        t.columnTypes = append(t.columnTypes, columnType)
    }
    b.tables[tableName] = &t
    return nil
}
```

It's a similar story with inserting a row, except we also need to check that each parsed column type matches the registered column types, and the number of parameters match.

```go
func (b *MemoryBackend) Insert(stmt *sqlparser.Insert) error {
    // ...
    for rowIndex, row := range rows {
        var cells []MemoryCell
        for i, val := range row {
            v, _, ct, err := table.evaluateCell(rowIndex, val)
            if err != nil {
                return fmt.Errorf("unable to evaluate cell: %w", err)
            }
            if ct != table.columnTypes[i] {
                return fmt.Errorf("mismatched types: %s != %s", ct, table.columnTypes[i])
            }
            cells = append(cells, v)
        }

        if len(cells) != len(table.columns) {
            return fmt.Errorf("mismatched number of fields: %d != %d", len(cells), len(table.columns))
        }
        table.rows = append(table.rows, cells)
    }
    return nil
}
```

Now our SQL database should return an error whenever we insert a row with the incorrect types or an incorrect number of parameters.

```
>> CREATE TABLE users (id INT, name TEXT);
>> INSERT INTO users VALUES (1, 'Alice');
>> INSERT INTO users VALUES (1, 3, 'Alice');
error: mismatched types: int != text
>> INSERT INTO users VALUES (1);
error: mismatched number of fields: 1 != 2
```

## Query

With the `CREATE` and `INSERT` statements implemented, now we just need some way to query our data, and we can do that by implementing the `SELECT` statement.

Let's start with a very simple select statement -- getting the `id` and `name` columns from the `users` table.

```
>> SELECT id, name FROM users;
+-------+----+
| NAME  | ID |
+-------+----+
| Alice |  1 |
| Bob   |  2 |
| Eva   |  3 |
+-------+----+
```

We just need to figure out what columns need to be returned (`id`, `name`), and then iterate over the rows in the table and _selecting_ only those columns. However, what if we want to conditionally select rows? Maybe we like to only return rows with `id = 3`?

We would need to support a new keyword, the `WHERE` keyword.

```
>> SELECT id, name FROM users WHERE id + 2 = 3;
+-------+----+
| NAME  | ID |
+-------+----+
| Alice |  1 |
+-------+----+
```

The `WHERE` filter would need to be able to support arbitrary expressions, and so we need a general mechanism for evaluating an expression for a given row. We will do something similar to a **tree-walk interpreter**, by traversing the SQL abstract syntax tree (AST) and evaluating as we go along, bottom up.

Using the example above, we need to evaluate the expression `id + 2 = 3` against every row, and filter. In the base case, we need to either return the value (int, text, etc.) if the expression is a `SQLVal` or resolve the value and return it for `ColName`.

```go
func (t *table) evaluateCell(rowIndex int, stmt sqlparser.Expr) (MemoryCell, string, ColumnType, error) {
    switch stmt := stmt.(type) {
    case *sqlparser.SQLVal:
        // Return a MemoryCell with the correct value.
    case *sqlparser.ColName:
        for i, colName := range t.columns {
            if colName == sqlparser.String(stmt) {
                return t.rows[rowIndex][i], colName, t.columnTypes[i], nil
            }
        }
    case *sqlparser.ComparisonExpr:
        return t.evaluateInfixComparisonCell(rowIndex, *stmt)
    case *sqlparser.BinaryExpr:
        return t.evaluateInfixOperationCell(rowIndex, *stmt)
    }
    return nil, "", 0, fmt.Errorf("unsupported expression: %s", stmt)
}
```

Now we need to support infix operations like `+` and the infix comparisons like `=` each within their own helper function. Let's take a look at the infix operation case.

```go
func (t *table) evaluateInfixOperationCell(rowIndex int, stmt sqlparser.BinaryExpr) (MemoryCell, string, ColumnType, error) {
    l, ln, lt, err := t.evaluateCell(rowIndex, stmt.Left)
    if err != nil {
        return nil, "", 0, err
    }
    r, rn, rt, err := t.evaluateCell(rowIndex, stmt.Right)
    if err != nil {
        return nil, "", 0, err
    }

    colName := ln
    if ln == unknownColumn {
        colName = rn
    }

    switch stmt.Operator {
    case sqlparser.PlusStr:
        if lt != rt {
            return nil, "", 0, ErrInvalidOperands
        }
        switch lt {
        case TextType:
            return MemoryCell([]byte(string(l) + string(r))), colName, TextType, nil
        case IntType:
            return MemoryCell(intToBytes(l.AsInt() + r.AsInt())), colName, IntType, nil
        }
        return nil, "", 0, ErrInvalidOperands
    }
    return nil, "", 0, ErrInvalidCell
}
```

There's nothing really special here other than propagating the column name if either left or right is unknown (which happens in the case of colName + val), making sure that the left and right types match, and implementing the actual `+` operation for `TextType` and `IntType`.

> The same approach applies for supporting keywords like `AND` and `OR`.

But that's about it! I have plans to continue working on and improving this, but don't quite feel like writing it all at the moment.
