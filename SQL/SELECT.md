# SELECT — Reading Data (PostgreSQL)

Everything for pulling data out: filtering, joining, aggregating, subqueries, CTEs, window functions.
PostgreSQL syntax; ~95% is standard SQL and works anywhere.

> **Running schema used in every example** (same across [CREATE](CREATE.md) / [UPDATE](UPDATE.md)):
> `users(id, name, email, country, created_at)`
> `orders(id, user_id, total, status, created_at)`
> `products(id, name, price, stock)`
> `order_items(id, order_id, product_id, qty, unit_price)`

- [Basics](#basics)
- [WHERE — filtering](#where--filtering)
- [Pattern matching & NULL](#pattern-matching--null)
- [ORDER BY / LIMIT / pagination](#order-by--limit--pagination)
- [Aggregates & GROUP BY](#aggregates--group-by)
- [JOINs](#joins)
- [Subqueries](#subqueries)
- [CTEs (WITH)](#ctes-with)
- [Window functions](#window-functions)
- [CASE / COALESCE / conditionals](#case--coalesce--conditionals)
- [Set operations](#set-operations)
- [Dates & times](#dates--times)
- [JSON / arrays (Postgres)](#json--arrays-postgres)
- [Performance: EXPLAIN](#performance-explain)

---

## Basics

```sql
SELECT * FROM users;                          -- everything (avoid in prod queries)
SELECT id, name, email FROM users;            -- only the columns you need
SELECT name AS full_name FROM users;          -- alias a column
SELECT DISTINCT country FROM users;           -- unique values
SELECT count(*) FROM users;                   -- how many rows
SELECT now(), current_date, current_user;     -- no FROM needed for expressions
```

> Order of evaluation (NOT the order you write it):
> `FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT`
> That's why you **can't use a SELECT alias in WHERE** (alias doesn't exist yet) but **can in ORDER BY**.

---

## WHERE — filtering

```sql
SELECT * FROM orders WHERE status = 'paid';
SELECT * FROM orders WHERE total > 100;
SELECT * FROM orders WHERE total BETWEEN 50 AND 200;        -- inclusive both ends
SELECT * FROM users  WHERE country IN ('US', 'DE', 'EG');   -- multiple matches
SELECT * FROM users  WHERE country NOT IN ('US', 'CA');

-- combine with AND / OR — parenthesize OR or it bites you
SELECT * FROM orders
WHERE status = 'paid'
  AND (total > 500 OR user_id = 42);
```

> **Gotcha — case sensitivity:** `=` is case-sensitive. `'paid' <> 'Paid'`.
> Use `lower(col) = 'paid'` or `ILIKE` for case-insensitive matches.

---

## Pattern matching & NULL

```sql
SELECT * FROM users WHERE email LIKE '%@gmail.com';   -- % = any chars, _ = one char
SELECT * FROM users WHERE name ILIKE 'ahm%';          -- ILIKE = case-insensitive (Postgres)
SELECT * FROM users WHERE email ~ '^[a-z]+@';         -- ~ = regex match (Postgres)

-- NULL is NOT a value — never use = with it
SELECT * FROM orders WHERE status IS NULL;            -- correct
SELECT * FROM orders WHERE status IS NOT NULL;
-- WHERE status = NULL  ← always returns 0 rows. Classic trap.
```

> **Gotcha:** `country NOT IN ('US', NULL)` returns **zero rows** — any NULL in the list poisons the
> whole `NOT IN`. Filter NULLs out first, or use `NOT EXISTS`.

---

## ORDER BY / LIMIT / pagination

```sql
SELECT * FROM orders ORDER BY created_at DESC;          -- newest first
SELECT * FROM orders ORDER BY status ASC, total DESC;   -- multi-key sort
SELECT * FROM users  ORDER BY name NULLS LAST;          -- where NULLs land (Postgres)

SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;             -- top 10
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10 OFFSET 20;   -- page 3 (rows 21–30)
```

> **Gotcha — OFFSET pagination gets slow** on big tables (DB still scans + discards the skipped rows).
> For deep pages use **keyset pagination** instead:
> ```sql
> SELECT * FROM orders WHERE created_at < '2026-01-01 00:00:00'
> ORDER BY created_at DESC LIMIT 10;   -- pass the last seen value as the cursor
> ```

---

## Aggregates & GROUP BY

```sql
SELECT count(*), sum(total), avg(total), min(total), max(total) FROM orders;

-- one row per group
SELECT status, count(*) AS n, sum(total) AS revenue
FROM orders
GROUP BY status;

-- filter on the aggregate with HAVING (WHERE can't see aggregates)
SELECT user_id, sum(total) AS spent
FROM orders
WHERE status = 'paid'        -- WHERE filters rows BEFORE grouping
GROUP BY user_id
HAVING sum(total) > 1000     -- HAVING filters groups AFTER aggregation
ORDER BY spent DESC;

-- count distinct, and count of non-null
SELECT count(DISTINCT user_id) AS unique_buyers FROM orders;
SELECT count(email) FROM users;   -- counts NON-NULL emails only (count(*) counts all rows)
```

> **Rule:** every column in SELECT must be either inside an aggregate **or** in the GROUP BY.
> Postgres errors loudly here; MySQL silently picks a random row (bug factory).

```sql
-- Postgres niceties:
SELECT string_agg(name, ', ') FROM users;                  -- concat group into one string
SELECT array_agg(id) FROM orders GROUP BY user_id;         -- collect group into an array
SELECT status, count(*) FROM orders GROUP BY ROLLUP(status); -- subtotals + grand total
SELECT count(*) FILTER (WHERE status = 'paid') FROM orders;  -- conditional aggregate (clean!)
```

---

## JOINs

```sql
-- INNER: only rows with a match on both sides
SELECT o.id, u.name, o.total
FROM orders o
JOIN users u ON u.id = o.user_id;

-- LEFT: every order, even if the user row is missing (user cols = NULL)
SELECT u.name, o.id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;

-- find users with NO orders (anti-join pattern)
SELECT u.*
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;

-- multi-table join: order → items → product
SELECT o.id, p.name, oi.qty, oi.unit_price
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p     ON p.id = oi.product_id;

-- self join (e.g. employees → managers, same table aliased twice)
SELECT e.name AS employee, m.name AS manager
FROM users e
JOIN users m ON m.id = e.manager_id;
```

| Join | Keeps |
|------|-------|
| `INNER JOIN` | rows matching on both sides |
| `LEFT JOIN`  | all left rows + matches (NULLs where none) |
| `RIGHT JOIN` | all right rows + matches |
| `FULL JOIN`  | everything from both, matched where possible |
| `CROSS JOIN` | every combination (Cartesian product) |

> **Gotcha:** putting a right-table condition in `WHERE` silently turns a LEFT JOIN into an INNER
> JOIN (NULLs fail the test). Keep it in the `ON` clause: `LEFT JOIN orders o ON o.user_id = u.id AND o.status = 'paid'`.

---

## Subqueries

```sql
-- scalar subquery (returns one value)
SELECT name FROM users
WHERE id = (SELECT user_id FROM orders ORDER BY total DESC LIMIT 1);

-- IN subquery (returns a column)
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE total > 500);

-- EXISTS — usually faster than IN, and NULL-safe (prefer this)
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- correlated subquery in SELECT (runs per outer row — watch cost)
SELECT u.name,
       (SELECT count(*) FROM orders o WHERE o.user_id = u.id) AS order_count
FROM users u;

-- subquery as a derived table (must be aliased)
SELECT country, avg(spent)
FROM (SELECT user_id, country, sum(total) AS spent
      FROM orders JOIN users ON users.id = orders.user_id
      GROUP BY user_id, country) t
GROUP BY country;
```

---

## CTEs (WITH)

Named subqueries up front — readable, reusable, and the building block for recursion.

```sql
WITH paid_orders AS (
    SELECT * FROM orders WHERE status = 'paid'
),
big_spenders AS (
    SELECT user_id, sum(total) AS spent
    FROM paid_orders GROUP BY user_id HAVING sum(total) > 1000
)
SELECT u.name, b.spent
FROM big_spenders b JOIN users u ON u.id = b.user_id
ORDER BY b.spent DESC;
```

```sql
-- recursive CTE: walk a tree (category → parent_id) or generate a series
WITH RECURSIVE tree AS (
    SELECT id, name, parent_id FROM categories WHERE parent_id IS NULL  -- anchor
    UNION ALL
    SELECT c.id, c.name, c.parent_id                                    -- recursive step
    FROM categories c JOIN tree t ON c.parent_id = t.id
)
SELECT * FROM tree;
```

> Postgres can also **write** through a CTE (`WITH x AS (DELETE ... RETURNING *) ...`) — see [UPDATE](UPDATE.md).

---

## Window functions

Aggregate **without collapsing rows** — you keep every row and add a computed column over a "window".

```sql
SELECT id, user_id, total,
       row_number() OVER (PARTITION BY user_id ORDER BY created_at) AS nth_order,
       rank()       OVER (ORDER BY total DESC)                      AS overall_rank,
       sum(total)   OVER (PARTITION BY user_id)                     AS user_lifetime,
       sum(total)   OVER (PARTITION BY user_id ORDER BY created_at) AS running_total,
       lag(total)   OVER (PARTITION BY user_id ORDER BY created_at) AS prev_order_total
FROM orders;
```

| Function | Use |
|----------|-----|
| `row_number()` | unique 1,2,3… (great for "latest per group" / dedupe) |
| `rank()` / `dense_rank()` | ranking with ties (rank skips, dense doesn't) |
| `lag()` / `lead()` | previous / next row's value (deltas, trends) |
| `sum/avg() OVER (… ORDER BY)` | running total / moving average |
| `ntile(4)` | bucket rows into quartiles |

```sql
-- "latest order per user" — the canonical window pattern
SELECT * FROM (
    SELECT *, row_number() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
) t
WHERE rn = 1;
```

---

## CASE / COALESCE / conditionals

```sql
SELECT name,
       CASE WHEN total >= 1000 THEN 'VIP'
            WHEN total >= 100  THEN 'regular'
            ELSE 'new'
       END AS tier
FROM orders JOIN users ON users.id = orders.user_id;

SELECT coalesce(nickname, name, 'anonymous') FROM users;  -- first non-NULL
SELECT nullif(total, 0) FROM orders;                      -- NULL if total=0 (avoid /0)
SELECT greatest(a, b, c), least(a, b, c) FROM t;          -- row-wise max/min
```

---

## Set operations

```sql
SELECT email FROM users
UNION        SELECT email FROM newsletter;   -- combine + dedupe
-- UNION ALL → keep duplicates (faster, no sort)
-- INTERSECT → rows in BOTH
-- EXCEPT    → in first, not in second
```

> Each side must have the **same column count and compatible types**, in the same order.

---

## Dates & times

```sql
SELECT now();                                   -- timestamp with tz
SELECT current_date, current_time;
SELECT created_at::date FROM orders;            -- cast timestamp → date
SELECT date_trunc('month', created_at) FROM orders;          -- floor to month
SELECT extract(year FROM created_at) FROM orders;            -- pull a part out
SELECT now() - interval '7 days';                            -- date math
SELECT * FROM orders WHERE created_at >= now() - interval '30 days';  -- last 30 days
SELECT age(now(), created_at) FROM users;                    -- human-readable diff
SELECT to_char(created_at, 'YYYY-MM-DD HH24:MI') FROM orders;-- format to string
```

> **Gotcha:** `WHERE created_at::date = '2026-06-29'` can't use an index on `created_at`.
> Prefer a range: `WHERE created_at >= '2026-06-29' AND created_at < '2026-06-30'`.

---

## JSON / arrays (Postgres)

```sql
-- jsonb column "meta"
SELECT meta->>'plan'          FROM users;   -- ->>  get value as text
SELECT meta->'address'->>'city' FROM users; -- ->   get value as json, chain it
SELECT * FROM users WHERE meta @> '{"plan":"pro"}';   -- contains (uses GIN index)
SELECT jsonb_array_elements(meta->'tags') FROM users; -- explode array to rows

-- native arrays
SELECT * FROM products WHERE 'sale' = ANY(tags);   -- value in array
SELECT array_length(tags, 1) FROM products;
SELECT unnest(tags) FROM products;                 -- array → rows
```

---

## Performance: EXPLAIN

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 42;            -- planner's estimate (no run)
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42;    -- actually runs + real timings
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;                      -- + cache/disk page hits
```

> Read it bottom-up. **`Seq Scan`** on a big table in a hot query = you probably want an index
> (see [CREATE › Indexes](CREATE.md#indexes)). **`Index Scan`** good. Watch for row-estimate vs
> actual being wildly off → stale stats, run `ANALYZE <table>;`.
