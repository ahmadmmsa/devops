# INSERT / UPDATE / DELETE — Writing Data (PostgreSQL)

DML mutations + transactions. Same running schema as [SELECT](SELECT.md) / [CREATE](CREATE.md).
This is the dangerous half of SQL — **the golden rule lives at the bottom; read it.**

- [INSERT](#insert)
- [UPDATE](#update)
- [DELETE](#delete)
- [UPSERT — INSERT … ON CONFLICT](#upsert--insert--on-conflict)
- [RETURNING](#returning)
- [Write through a CTE](#write-through-a-cte)
- [Transactions](#transactions)
- [Locking](#locking)
- [Bulk load (COPY)](#bulk-load-copy)
- [⚠️ Safety rules](#-safety-rules)

---

## INSERT

```sql
-- single row (always name the columns — survives schema changes)
INSERT INTO users (name, email, country)
VALUES ('Ahmad', 'ahmad@example.com', 'EG');

-- multi-row (one statement, one round trip — much faster than N inserts)
INSERT INTO users (name, email, country) VALUES
    ('Sara',  'sara@example.com',  'DE'),
    ('John',  'john@example.com',  'US'),
    ('Lina',  'lina@example.com',  'EG');

-- insert from a query
INSERT INTO us_users (id, name, email)
SELECT id, name, email FROM users WHERE country = 'US';

-- let defaults/identity fill themselves
INSERT INTO orders (user_id, total) VALUES (1, 250.00);   -- id, status, created_at auto-filled
INSERT INTO orders DEFAULT VALUES;                         -- all defaults
```

> **Don't insert into a `GENERATED ALWAYS AS IDENTITY` column.** Let the DB assign it. If you truly
> must (data import), use `OVERRIDING SYSTEM VALUE`.

---

## UPDATE

```sql
UPDATE orders SET status = 'shipped' WHERE id = 42;        -- ALWAYS scope with WHERE
UPDATE orders SET status = 'paid', total = total * 1.1 WHERE id = 42;  -- multiple cols, self-ref math

-- update from another table (Postgres FROM-join syntax)
UPDATE orders o
SET    total = o.total * 0.9                               -- 10% off
FROM   users u
WHERE  u.id = o.user_id AND u.country = 'EG';

-- conditional update
UPDATE products
SET stock = stock - 1
WHERE id = 7 AND stock > 0;                                -- guard against going negative
```

> **Gotcha:** `UPDATE orders SET status='shipped';` with **no WHERE** updates *every row*. There is
> no "are you sure?". See [Safety rules](#-safety-rules).

---

## DELETE

```sql
DELETE FROM orders WHERE id = 42;
DELETE FROM orders WHERE status = 'cancelled' AND created_at < now() - interval '90 days';

-- delete using another table
DELETE FROM orders o
USING users u
WHERE o.user_id = u.id AND u.is_active = false;

DELETE FROM orders;            -- deletes ALL rows (row-by-row, fires triggers, can rollback)
-- to empty a table fast, prefer TRUNCATE (see CREATE.md)
```

> `DELETE` honors foreign keys: it'll fail (or cascade) depending on the child's `ON DELETE` rule.
> See [CREATE › Foreign keys](CREATE.md#foreign-keys--on-delete).

---

## UPSERT — INSERT … ON CONFLICT

Insert, but if it collides with a unique/PK constraint, update instead. The clean way to "create or update".

```sql
-- on conflict: update the existing row (needs a UNIQUE/PK on email)
INSERT INTO users (email, name, country)
VALUES ('ahmad@example.com', 'Ahmad M.', 'EG')
ON CONFLICT (email)
DO UPDATE SET name = EXCLUDED.name,            -- EXCLUDED = the row you tried to insert
              country = EXCLUDED.country;

-- on conflict: do nothing (idempotent insert — great for seed data / queues)
INSERT INTO users (email, name) VALUES ('sara@example.com', 'Sara')
ON CONFLICT (email) DO NOTHING;

-- counter / merge pattern
INSERT INTO product_views (product_id, views) VALUES (7, 1)
ON CONFLICT (product_id) DO UPDATE SET views = product_views.views + 1;
```

> `ON CONFLICT` needs a real **unique constraint or index** on the named column(s) to detect the
> conflict. PG15+ also has standard `MERGE`, but `ON CONFLICT` is simpler for the common case.

---

## RETURNING

Postgres lets any write hand back the affected rows — no separate SELECT round-trip.

```sql
INSERT INTO users (name, email) VALUES ('Ahmad', 'a@x.com')
RETURNING id, created_at;                      -- get the generated id immediately

UPDATE orders SET status = 'paid' WHERE id = 42
RETURNING id, status, total;

DELETE FROM orders WHERE status = 'cancelled'
RETURNING *;                                   -- see exactly what you deleted (audit/sanity)
```

---

## Write through a CTE

Chain writes in one atomic statement — e.g. archive then delete.

```sql
WITH moved AS (
    DELETE FROM orders
    WHERE created_at < now() - interval '1 year'
    RETURNING *
)
INSERT INTO orders_archive SELECT * FROM moved;   -- delete + archive, all-or-nothing
```

---

## Transactions

A group of statements that **all** commit or **all** roll back. Use them for any multi-step change
that must stay consistent (the classic: move money between two accounts).

```sql
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;                          -- both happened, atomically

BEGIN;
    DELETE FROM orders WHERE user_id = 99;
    -- ...realize that was wrong...
ROLLBACK;                        -- nothing happened; table untouched

-- savepoints: partial rollback inside a big transaction
BEGIN;
    INSERT INTO users (name,email) VALUES ('A','a@x.com');
    SAVEPOINT sp1;
    INSERT INTO users (name,email) VALUES ('B','bad');   -- fails
    ROLLBACK TO sp1;                                      -- undo just B, keep A
    INSERT INTO users (name,email) VALUES ('C','c@x.com');
COMMIT;
```

> **Gotcha (Postgres):** once any statement errors inside a transaction, the whole tx enters an
> aborted state — every later statement returns *"current transaction is aborted"* until you
> `ROLLBACK` (or `ROLLBACK TO` a savepoint). It will **not** silently continue.

**Isolation levels** (how much concurrent transactions can see of each other):

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;   -- strictest: behaves as if txns ran one-at-a-time
-- READ COMMITTED (default) → REPEATABLE READ → SERIALIZABLE
```

---

## Locking

```sql
-- lock rows you're about to update so a concurrent txn can't change them under you
BEGIN;
    SELECT * FROM products WHERE id = 7 FOR UPDATE;   -- row lock until commit/rollback
    UPDATE products SET stock = stock - 1 WHERE id = 7;
COMMIT;

SELECT * FROM products WHERE id = 7 FOR UPDATE SKIP LOCKED;   -- job-queue pattern: grab next free row
SELECT * FROM products WHERE id = 7 FOR UPDATE NOWAIT;        -- error instead of waiting
```

> **Deadlock:** two txns lock rows in opposite order and wait on each other. Postgres detects it and
> kills one with a deadlock error. Avoid it by always locking rows **in a consistent order** (e.g.
> ascending id).

---

## Bulk load (COPY)

Orders of magnitude faster than looping INSERTs for big imports.

```sql
-- server-side (file must be readable by the postgres OS user)
COPY users (name, email, country) FROM '/var/lib/postgresql/import.csv' WITH (FORMAT csv, HEADER);
COPY (SELECT * FROM orders WHERE status='paid') TO '/tmp/paid.csv' WITH (FORMAT csv, HEADER);
```
```bash
# client-side via psql \copy (uses YOUR file perms, works over a remote connection)
psql -d shop -c "\copy users(name,email,country) FROM 'import.csv' WITH (FORMAT csv, HEADER)"
```

> For migrating whole databases from MSSQL / MySQL / SQLite into Postgres, see [pgloader.md](pgloader.md).

---

## ⚠️ Safety rules

> **The golden rule: never run a bare `UPDATE`/`DELETE`.** A missing `WHERE` hits every row, and
> there's no undo outside a transaction.

1. **SELECT first, then mutate.** Write `SELECT * FROM orders WHERE …;`, eyeball the rows, then swap
   `SELECT *` for `DELETE` / `UPDATE … SET` keeping the *exact same* `WHERE`.
2. **Wrap risky changes in a transaction** so you can bail:
   ```sql
   BEGIN;
   DELETE FROM orders WHERE status = 'cancelled';
   -- check the row count it reports; if wrong:
   ROLLBACK;   -- else: COMMIT;
   ```
3. **Use `RETURNING`** to see exactly what changed.
4. **psql guardrail** — make psql refuse a non-transaction-wrapped destructive mistake by stopping on
   the first error and not autocommitting:
   ```
   \set ON_ERROR_STOP on
   \set AUTOCOMMIT off
   ```
5. **Back up before bulk surgery:** `pg_dump shop > shop.sql` (or dump just the table:
   `pg_dump -t orders shop > orders.sql`).
