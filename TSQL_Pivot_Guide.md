# Writing a Pivot Table in T-SQL — Step-by-Step Guide

## Summary

The `PIVOT` operator turns row values into columns. There are two flavors:

- **Static PIVOT** — you hard-code the column names. Simplest; best when the columns are fixed.
- **Dynamic PIVOT** — you build the column list at runtime with dynamic SQL. Needed when the values can change or grow.
- **Conditional aggregation** — a `CASE`-based alternative that produces the same result without the `PIVOT` operator. More verbose, but clearer and more flexible.

All examples below use the same sales-by-month sample table.

---

## The sample table and data

A `Sales` table with one row per transaction:

```sql
CREATE TABLE Sales (
    Salesperson VARCHAR(50),
    SaleMonth   VARCHAR(10),   -- 'Jan', 'Feb', 'Mar', ...
    Amount      DECIMAL(10,2)
);
```

Insert some sample values:

```sql
INSERT INTO Sales (Salesperson, SaleMonth, Amount) VALUES
    ('Alice', 'Jan', 1200.00),
    ('Alice', 'Jan',  300.00),   -- two Jan rows for Alice -> they get summed
    ('Alice', 'Feb',  950.00),
    ('Alice', 'Mar', 1100.00),
    ('Bob',   'Jan',  800.00),
    ('Bob',   'Feb', 1500.00),
    ('Bob',   'Mar',  600.00),
    ('Carol', 'Jan',  450.00),
    ('Carol', 'Feb',  725.00),
    ('Carol', 'Mar', 1320.00);
```

Goal: one row per salesperson, one **column per month**, with total sales in each cell.

Expected result:

| Salesperson | Jan  | Feb  | Mar  |
|-------------|------|------|------|
| Alice       | 1500 | 950  | 1100 |
| Bob         | 800  | 1500 | 600  |
| Carol       | 450  | 725  | 1320 |

(Note Alice's two January rows are summed into 1500.)

---

## Part 1 — Static PIVOT (columns known ahead of time)

**Step 1 — Pick your three roles.** Every pivot needs you to identify:

- The **row identifier** (what stays as rows) → `Salesperson`
- The **column source** (values that become column headers) → `SaleMonth`
- The **value to aggregate** (what fills the cells) → `Amount`

**Step 2 — Choose an aggregate.** `PIVOT` *always* aggregates, even if there's one value per cell. Common choices: `SUM`, `COUNT`, `MAX`, `AVG`.

**Step 3 — Write a clean source subquery.** Feed `PIVOT` only the three columns you need. Any extra column gets treated as an implicit row grouping and can fragment your results.

**Step 4 — Apply the `PIVOT` clause** with the aggregate and the explicit list of column values:

```sql
SELECT Salesperson, [Jan], [Feb], [Mar]
FROM (
    SELECT Salesperson, SaleMonth, Amount
    FROM Sales
) AS src
PIVOT (
    SUM(Amount)                              -- aggregate + value column
    FOR SaleMonth IN ([Jan], [Feb], [Mar])   -- column source + the values to spread
) AS pvt;
```

**Step 5 — Read it back.** The `IN (...)` list must match the actual values in `SaleMonth`, and the same names must appear in the outer `SELECT`. Anything not listed is simply dropped.

- **Pros:** simple, readable, fast.
- **Cons:** you must edit the query whenever a new month/value appears.

---

## Part 2 — Dynamic PIVOT (columns vary or grow over time)

When you don't know the months in advance (or they keep growing), generate the `IN (...)` list at runtime and execute the whole statement as a string.

**Step 1 — Build the column list from the data.** Use `STRING_AGG` (SQL Server 2017+) over the distinct values, wrapping each in `QUOTENAME` so brackets/escaping are handled safely:

```sql
DECLARE @cols NVARCHAR(MAX);

SELECT @cols = STRING_AGG(QUOTENAME(SaleMonth), ',')
FROM (SELECT DISTINCT SaleMonth FROM Sales) AS m;
-- @cols now holds something like: [Jan],[Feb],[Mar]
```

**Step 2 — Assemble the full statement as text.** Drop `@cols` into the same structure as the static version, in both the `SELECT` list and the `IN (...)` clause:

```sql
DECLARE @sql NVARCHAR(MAX);

SET @sql = N'
    SELECT Salesperson, ' + @cols + N'
    FROM (
        SELECT Salesperson, SaleMonth, Amount
        FROM Sales
    ) AS src
    PIVOT (
        SUM(Amount)
        FOR SaleMonth IN (' + @cols + N')
    ) AS pvt;';
```

**Step 3 — Execute it safely.** Use `sp_executesql` rather than raw `EXEC` — it's the standard, more secure way to run dynamic SQL:

```sql
EXEC sp_executesql @sql;
```

- **Pros:** adapts automatically as new values appear — no query edits.
- **Cons:** harder to read/debug; built from text, so guard against SQL injection (the `QUOTENAME` step is your main protection — never concatenate raw user input).

---

## Part 3 — The alternative: manual conditional aggregation

You can produce the exact same result without `PIVOT` at all, using `GROUP BY` plus a `CASE` expression inside each aggregate. This is often called "conditional aggregation" or a "crosstab."

**Step 1 — Group by the row identifier.** This is what `PIVOT` does implicitly; here you do it explicitly.

**Step 2 — Create one aggregated column per value** using a `CASE` that only lets the matching rows through; everything else becomes `NULL` and is ignored by `SUM`.

```sql
SELECT
    Salesperson,
    SUM(CASE WHEN SaleMonth = 'Jan' THEN Amount END) AS [Jan],
    SUM(CASE WHEN SaleMonth = 'Feb' THEN Amount END) AS [Feb],
    SUM(CASE WHEN SaleMonth = 'Mar' THEN Amount END) AS [Mar]
FROM Sales
GROUP BY Salesperson;
```

**Step 3 — Optionally wrap with `ISNULL`/`COALESCE`** to show `0` instead of `NULL` for empty cells:

```sql
SELECT
    Salesperson,
    ISNULL(SUM(CASE WHEN SaleMonth = 'Jan' THEN Amount END), 0) AS [Jan],
    ISNULL(SUM(CASE WHEN SaleMonth = 'Feb' THEN Amount END), 0) AS [Feb],
    ISNULL(SUM(CASE WHEN SaleMonth = 'Mar' THEN Amount END), 0) AS [Mar]
FROM Sales
GROUP BY Salesperson;
```

**Why you might prefer this over `PIVOT`:**

- You can mix **multiple aggregates** in one query (e.g. a `SUM` column *and* a `COUNT` column for the same month) — `PIVOT` can't.
- The logic is plain SQL, so it's easier to read, debug, and port to other databases.
- The `CASE` condition can be richer than a simple equality (e.g. ranges, multiple values).

**The trade-off:** it's more verbose, and for a dynamic column set you still need dynamic SQL to generate the `CASE` lines (see Part 5).

### Variant A — multiple aggregates side by side

A capability `PIVOT` simply doesn't have: emit a total *and* a deal count for each month in the same query.

```sql
SELECT
    Salesperson,
    SUM(CASE WHEN SaleMonth = 'Jan' THEN Amount END)  AS [Jan_Total],
    COUNT(CASE WHEN SaleMonth = 'Jan' THEN 1 END)     AS [Jan_Deals],
    SUM(CASE WHEN SaleMonth = 'Feb' THEN Amount END)  AS [Feb_Total],
    COUNT(CASE WHEN SaleMonth = 'Feb' THEN 1 END)     AS [Feb_Deals],
    SUM(CASE WHEN SaleMonth = 'Mar' THEN Amount END)  AS [Mar_Total],
    COUNT(CASE WHEN SaleMonth = 'Mar' THEN 1 END)     AS [Mar_Deals]
FROM Sales
GROUP BY Salesperson;
```

### Variant B — `IIF` shorthand (SQL Server 2012+)

`IIF` is a terser equivalent of a two-branch `CASE`:

```sql
SELECT
    Salesperson,
    SUM(IIF(SaleMonth = 'Jan', Amount, NULL)) AS [Jan],
    SUM(IIF(SaleMonth = 'Feb', Amount, NULL)) AS [Feb],
    SUM(IIF(SaleMonth = 'Mar', Amount, NULL)) AS [Mar]
FROM Sales
GROUP BY Salesperson;
```

### Variant C — `MAX` to pick the value instead of totaling

If each cell holds at most one row and you want the raw value rather than a sum, use `MAX` — it reads as "pick the value":

```sql
SELECT
    Salesperson,
    MAX(CASE WHEN SaleMonth = 'Jan' THEN Amount END) AS [Jan],
    MAX(CASE WHEN SaleMonth = 'Feb' THEN Amount END) AS [Feb],
    MAX(CASE WHEN SaleMonth = 'Mar' THEN Amount END) AS [Mar]
FROM Sales
GROUP BY Salesperson;
```

---

## Part 4 — Zero-filling empty cells with `ISNULL`

A cell with no matching rows is the aggregate of an empty set, which returns `NULL`. To display `0` instead, wrap each aggregate in `ISNULL` (or the ANSI-standard `COALESCE`):

```sql
SELECT
    Salesperson,
    ISNULL(SUM(CASE WHEN SaleMonth = 'Jan' THEN Amount END), 0) AS [Jan],
    ISNULL(SUM(CASE WHEN SaleMonth = 'Feb' THEN Amount END), 0) AS [Feb],
    ISNULL(SUM(CASE WHEN SaleMonth = 'Mar' THEN Amount END), 0) AS [Mar]
FROM Sales
GROUP BY Salesperson;
```

- **`ISNULL(expr, 0)`** is T-SQL-specific and slightly faster; **`COALESCE(expr, 0)`** is portable across databases and accepts more than two arguments.
- Apply it **outside** the aggregate (`ISNULL(SUM(...), 0)`), not inside — wrapping `Amount` itself would change the math by treating absent rows as real zeros in averages and counts.
- The same wrapping works on a static `PIVOT` too: wrap each pivoted column in the outer `SELECT`, e.g. `ISNULL([Jan], 0) AS [Jan]`.

---

## Part 5 — Dynamic conditional aggregation (auto-generate the `CASE` columns)

When the months aren't known ahead of time, you can build the `CASE` column lines from whatever values actually exist, then run the assembled statement — the conditional-aggregation equivalent of a dynamic `PIVOT`.

**Step 1 — Generate one `CASE` line per distinct value.** Use `STRING_AGG` to stitch the lines together, `QUOTENAME` for the column alias, and `QUOTENAME(..., '''')` to safely single-quote the literal inside the `CASE`:

```sql
DECLARE @cols NVARCHAR(MAX);

SELECT @cols = STRING_AGG(
    'ISNULL(SUM(CASE WHEN SaleMonth = '
        + QUOTENAME(SaleMonth, '''')          -- safely quoted literal: 'Jan'
        + ' THEN Amount END), 0) AS '
        + QUOTENAME(SaleMonth),               -- safely bracketed alias: [Jan]
    ',' + CHAR(13) + CHAR(10) + '    '         -- comma + newline + indent
) WITHIN GROUP (ORDER BY SaleMonth)            -- keeps column order stable
FROM (SELECT DISTINCT SaleMonth FROM Sales) AS m;
```

**Step 2 — Drop the generated lines into a full statement:**

```sql
DECLARE @sql NVARCHAR(MAX);

SET @sql = N'
SELECT
    Salesperson,
    ' + @cols + N'
FROM Sales
GROUP BY Salesperson;';
```

**Step 3 — Execute it safely** with `sp_executesql`:

```sql
EXEC sp_executesql @sql;
```

**Step 4 — Inspect before running (optional but recommended).** While developing, `PRINT @sql;` (or `SELECT @sql;`) to see the generated SQL. It should look like the hand-written Part 4 query:

```sql
SELECT
    Salesperson,
    ISNULL(SUM(CASE WHEN SaleMonth = 'Feb' THEN Amount END), 0) AS [Feb],
    ISNULL(SUM(CASE WHEN SaleMonth = 'Jan' THEN Amount END), 0) AS [Jan],
    ISNULL(SUM(CASE WHEN SaleMonth = 'Mar' THEN Amount END), 0) AS [Mar]
FROM Sales
GROUP BY Salesperson;
```

- **Why generate `CASE` lines instead of using dynamic `PIVOT`?** Because this approach keeps every advantage of conditional aggregation — you can extend the generated template to emit multiple aggregates per value, custom conditions, or zero-filling, none of which dynamic `PIVOT` supports.
- **Safety:** `QUOTENAME` on both the literal and the alias is what protects against SQL injection. Never concatenate raw user input without it.
- **Column order:** `WITHIN GROUP (ORDER BY ...)` controls the left-to-right order. For real months, order by a month number rather than the text name so you don't get `Apr, Aug, Dec, ...` alphabetically.

---

## Key things to remember

- **The aggregate is mandatory** — `PIVOT` can't return raw values, only aggregated ones.
- **Trim your source to exactly three columns** in the subquery; stray columns silently change your grouping.
- **Static vs dynamic is purely about the column list** — the core syntax is identical; dynamic just generates the list instead of typing it.
- **Sort order isn't guaranteed** by `STRING_AGG`; add `ORDER BY` / `WITHIN GROUP (ORDER BY ...)` (or a month-number lookup) if column order matters.
- **Conditional aggregation is the flexible fallback** — reach for it when you need multiple aggregates, complex conditions, zero-filling, or cross-database portability.
- **`ISNULL`/`COALESCE` goes outside the aggregate** — wrapping the source value instead would distort averages and counts.
- **Anything static can be made dynamic** — both `PIVOT` and conditional aggregation can have their column lists generated from the data, guarded by `QUOTENAME`.
