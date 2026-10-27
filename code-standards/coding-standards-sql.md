---
id: coding-standards-sql
scope: scoped
severity: warning
audience: all
applies-to:
  - "**/*.sql"
schemaVersion: 1
---

# Coding standards: SQL

The SQL specifics on top of the coding standards for all languages (`coding-standards-all-languages`), which are not repeated here. These rules apply to every hand-written SQL statement: queries, views, stored procedures, integration and data platform notebooks. SQL embedded in C# is covered through the C# standard's raw SQL rule. Schema design and migrations are out of scope.

## SQL Philosophy

- **Explicit over implicit.** Every column, alias, join type and grouping is written out. A query never relies on defaults or on what the engine happens to infer.
- **One shape for every query.** Consistent casing, layout and naming make a query readable at a glance and keep pull request diffs small.

## Casing

- **Reserved keywords and functions are uppercase** (`SELECT`, `FROM`, `AS`, `COUNT`, `SUM`).
- **Literals and data types are uppercase** (`TRUE`, `FALSE`, `NULL`, `INT`, `DATE`).
- **Aliases are `PascalCase`.** Never lowercase, uppercase, `camelCase` or `snake_case`.

## Naming and aliases

- **Names are descriptive and consistent.** Only letters and digits, never starting with a digit, and never a reserved keyword (`TABLE`, `COLUMN`, `ROW`).
- **Every alias uses `AS`.**
- **Aliases have no quotes and no spaces.**
- **Every column expression has an alias.** An aggregate or calculation never appears without one.
- **Table aliases are unique within a query and always used.** An alias that is never referenced is removed.
- **No `SELECT *`.** Every column is listed, also in a `UNION ALL` and in a common table expression.

## Layout

- **Root keywords align on the left, each on its own line** (`SELECT`, `FROM`, `WHERE`, `GROUP BY`). The "rivers" pattern, where keywords are right-aligned around a gutter, is not used.
- **One argument per line, with a leading comma** followed by a space.
- **Indentation is one level per nesting step.**
- **String values are in single quotes, and a list of values is in parentheses.**
- **A multi-line parenthesis opens at the end of a line, its content is indented one level, and it closes on its own line** aligned with the start of the line that opened it.

## Joins

- **The shortest join keyword.** `JOIN` rather than `INNER JOIN`, `LEFT JOIN` rather than `LEFT OUTER JOIN`.
- **Every column is referenced through its table alias** when a query reads more than one table, so a column that later appears in both tables never becomes ambiguous.
- **A filter on the joined table belongs in the join condition,** not in the `WHERE` clause. The exception is an anti-join, where `WHERE <joined alias>.<key> IS NULL` after a `LEFT JOIN` keeps only the rows without a match.
- **Each join starts on its own line,** one level deeper than `FROM`; `ON` and every following `AND` start on their own line, one level deeper than the join.

## Grouping and combining

- **No `DISTINCT` and no `UNION` to remove duplicates.** Deduplication is explicit with `GROUP BY`, or handled before the data is combined.
- **`GROUP BY` lists every column explicitly,** one per line. Implicit grouping (`GROUP BY ALL`) is not used.
- **`UNION ALL` combines result sets,** with every column of both sides written out and an empty line before and after the `UNION ALL`. `UNION` is not used, because its silent deduplication makes the result unpredictable.

## Conditional logic

- **`CASE` is a last resort for translations.** A value is translated through an enumeration or mapping table in the source system; `CASE` is only used when no such table exists and the translation cannot be added to an existing one.
- **Each `WHEN` and the `ELSE` are on their own line,** one level deeper than `CASE`, and `END AS` is on its own line at the level of `CASE`. Both the searched form (`CASE WHEN x = 1`) and the simple form (`CASE x WHEN 1`) are allowed.

## Common table expressions

- **Common table expressions over nested queries.** A subquery in `FROM` is only used when a common table expression is not possible.
- **Each common table expression starts on its own line,** with an empty line between two of them and the separating comma leading the next one.

## Comments

- **A comment that is needed** uses `--` for a short note on one line, and `/* */` for an explanation over several lines.
