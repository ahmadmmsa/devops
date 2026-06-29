# CREATE / DDL — Defining Structure (PostgreSQL)

Data Definition Language: building and altering the schema — databases, tables, types, constraints,
indexes, views. Same running schema as [SELECT](SELECT.md) / [UPDATE](UPDATE.md).

- [Databases & schemas](#databases--schemas)
- [CREATE TABLE](#create-table)
- [Data types (Postgres)](#data-types-postgres)
- [Constraints](#constraints)
- [Foreign keys & ON DELETE](#foreign-keys--on-delete)
- [ALTER TABLE](#alter-table)
- [Indexes](#indexes)
- [Views & materialized views](#views--materialized-views)
- [DROP / TRUNCATE](#drop--truncate)
- [Roles & grants](#roles--grants)

---

## Databases & schemas

```sql
CREATE DATABASE shop;                       -- run from `postgres` db / psql, not inside a tx
CREATE DATABASE shop OWNER appuser ENCODING 'UTF8';
\c shop                                      -- psql: connect to it
DROP DATABASE shop;                          -- gone. no undo. no warning.

CREATE SCHEMA billing;                       -- namespace inside a database
CREATE TABLE billing.invoices (...);         -- qualify with schema.table
SET search_path TO billing, public;          -- default schema lookup order
```

> `CREATE DATABASE` cannot run inside a transaction block or while anyone is connected to the target.

---

## CREATE TABLE

```sql
CREATE TABLE users (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- modern auto-id (SQL standard)
    name        text NOT NULL,
    email       text UNIQUE NOT NULL,
    country     char(2),
    is_active   boolean NOT NULL DEFAULT true,
    meta        jsonb DEFAULT '{}',
    created_at  timestamptz NOT NULL DEFAULT now()
);
```

```sql
-- older but everywhere: serial = auto-incrementing int (Postgres shorthand)
CREATE TABLE products (
    id         serial PRIMARY KEY,           -- serial = int + sequence; bigserial = bigint
    name       text NOT NULL,
    price      numeric(10,2) NOT NULL CHECK (price >= 0),  -- exact money: 10 digits, 2 decimal
    stock      int NOT NULL DEFAULT 0,
    tags       text[]                         -- native array column
);

-- safe re-runs
CREATE TABLE IF NOT EXISTS products (...);

-- clone structure / data
CREATE TABLE products_backup (LIKE products INCLUDING ALL);   -- structure + constraints + indexes
CREATE TABLE us_users AS SELECT * FROM users WHERE country = 'US';  -- structure + data, no constraints
CREATE TEMP TABLE scratch (...);              -- dropped at end of session
```

> Prefer `GENERATED ALWAYS AS IDENTITY` over `serial` in new schemas: it's SQL-standard, and
> blocks accidental manual inserts into the id column. Use `bigint`/`bigserial` for ids that grow —
> an `int` PK caps at ~2.1 billion and overflowing it in prod is a famously bad day.

---

## Data types (Postgres)

| Type | Use | Notes |
|------|-----|-------|
| `int` / `bigint` | whole numbers | `int` = ±2.1B; `bigint` for ids/counters |
| `numeric(p,s)` | **money, exact** | never use `float` for money — rounding errors |
| `real` / `double precision` | scientific, approximate | fast, lossy |
| `text` | strings | no length penalty; prefer over `varchar(n)` unless you need the cap |
| `varchar(n)` / `char(n)` | bounded / fixed strings | `char` pads with spaces (rarely what you want) |
| `boolean` | true/false | accepts `'t'`/`'f'`, `1`/`0` |
| `timestamptz` | a moment in time | **use this, not `timestamp`** — stores UTC, handles tz |
| `date` / `time` | calendar date / wall clock | |
| `uuid` | distributed ids | `gen_random_uuid()` (needs `pgcrypto` or PG13+) |
| `jsonb` | structured/semi-structured | binary, indexable (GIN), faster than `json` |
| `text[]` / `int[]` | arrays | `'{a,b,c}'` literal |
| `inet` / `cidr` / `macaddr` | network addrs | native IP types — homelab friendly |

> **Rule of thumb:** `text` for strings, `numeric` for money, `timestamptz` for time, `bigint` for ids.

---

## Constraints

```sql
CREATE TABLE orders (
    id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- unique + not null + indexed
    user_id   bigint NOT NULL,
    total     numeric(12,2) NOT NULL CHECK (total >= 0),        -- value rule
    status    text NOT NULL DEFAULT 'pending'
              CHECK (status IN ('pending','paid','shipped','cancelled')),  -- enum-like
    code      text UNIQUE,                                      -- no duplicates (NULLs allowed/ignored)
    created_at timestamptz NOT NULL DEFAULT now(),

    -- table-level (needed for multi-column constraints)
    CONSTRAINT uq_user_code UNIQUE (user_id, code)             -- name it → easier to drop later
);
```

| Constraint | Guarantees |
|------------|-----------|
| `PRIMARY KEY` | unique + NOT NULL, one per table, auto-indexed |
| `UNIQUE` | no duplicates (NULLs don't count as equal) |
| `NOT NULL` | value required |
| `CHECK (expr)` | row must satisfy expr |
| `DEFAULT val` | value when none supplied |
| `FOREIGN KEY` | value must exist in another table (see below) |

---

## Foreign keys & ON DELETE

```sql
CREATE TABLE orders (
    id       bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id  bigint NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    ...
);

CREATE TABLE order_items (
    id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id   bigint NOT NULL REFERENCES orders(id)   ON DELETE CASCADE,    -- delete items with order
    product_id bigint NOT NULL REFERENCES products(id) ON DELETE RESTRICT,   -- block deleting a sold product
    qty        int NOT NULL CHECK (qty > 0),
    unit_price numeric(10,2) NOT NULL
);
```

| `ON DELETE` action | When parent row is deleted… |
|--------------------|------------------------------|
| `CASCADE` | delete the child rows too |
| `RESTRICT` / `NO ACTION` | block the delete if children exist (default) |
| `SET NULL` | set child FK to NULL (column must be nullable) |
| `SET DEFAULT` | set child FK to its default |

> **Gotcha:** Postgres does **not** auto-index foreign key columns. A FK without an index makes
> the child-side joins and parent deletes slow. Add `CREATE INDEX ON orders(user_id);` yourself.

---

## ALTER TABLE

```sql
ALTER TABLE users ADD COLUMN phone text;
ALTER TABLE users ADD COLUMN age int NOT NULL DEFAULT 0;   -- fast in PG11+ (no full rewrite)
ALTER TABLE users DROP COLUMN phone;
ALTER TABLE users RENAME COLUMN name TO full_name;
ALTER TABLE users RENAME TO customers;

ALTER TABLE users ALTER COLUMN email SET NOT NULL;
ALTER TABLE users ALTER COLUMN country DROP NOT NULL;
ALTER TABLE orders ALTER COLUMN total TYPE numeric(14,2);   -- change type (may rewrite + lock)
ALTER TABLE users ALTER COLUMN created_at SET DEFAULT now();

-- constraints
ALTER TABLE orders ADD CONSTRAINT chk_total CHECK (total >= 0);
ALTER TABLE orders ADD CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id);
ALTER TABLE orders DROP CONSTRAINT chk_total;
```

> **Gotcha — locks:** most `ALTER TABLE` takes an `ACCESS EXCLUSIVE` lock (blocks all reads+writes)
> for its duration. Adding a `NOT NULL` column with a non-volatile default is fast (metadata-only)
> since PG11; a `TYPE` change usually rewrites the whole table. On big prod tables, do these in a
> maintenance window, and validate new constraints in two steps: `ADD ... NOT VALID;` then
> `VALIDATE CONSTRAINT;` (the validate scan doesn't block writes).

---

## Indexes

```sql
CREATE INDEX idx_orders_user ON orders(user_id);              -- speed up lookups/joins on user_id
CREATE UNIQUE INDEX idx_users_email ON users(lower(email));   -- case-insensitive uniqueness
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC); -- composite (column order matters!)
CREATE INDEX idx_orders_paid ON orders(created_at) WHERE status = 'paid'; -- partial: smaller, hot subset
CREATE INDEX idx_users_meta ON users USING GIN (meta);       -- jsonb / array / full-text containment
CREATE INDEX idx_users_lower_name ON users(lower(name));     -- expression index (match the query!)

-- build WITHOUT locking writes (do this on prod) — cannot run inside a transaction
CREATE INDEX CONCURRENTLY idx_orders_status ON orders(status);

DROP INDEX idx_orders_status;
DROP INDEX CONCURRENTLY idx_orders_status;   -- non-blocking drop
\d orders                                     -- psql: list a table's indexes & constraints
```

> **Composite index column order:** put the column you filter by **equality** first, range/sort last.
> `(user_id, created_at)` serves `WHERE user_id=? ORDER BY created_at` perfectly; the reverse order won't.
>
> **Cost:** every index speeds reads but slows every INSERT/UPDATE/DELETE and eats disk. Index the
> columns in your `WHERE`/`JOIN`/`ORDER BY`, not everything. Verify it's used with `EXPLAIN ANALYZE`.

---

## Views & materialized views

```sql
-- VIEW: a saved query, runs fresh every time (no stored data)
CREATE VIEW paid_orders AS
SELECT * FROM orders WHERE status = 'paid';

CREATE OR REPLACE VIEW user_revenue AS
SELECT u.id, u.name, coalesce(sum(o.total),0) AS revenue
FROM users u LEFT JOIN orders o ON o.user_id = u.id AND o.status='paid'
GROUP BY u.id, u.name;

SELECT * FROM user_revenue WHERE revenue > 1000;   -- query it like a table

-- MATERIALIZED VIEW: stores the result on disk (fast reads, stale until refreshed)
CREATE MATERIALIZED VIEW daily_sales AS
SELECT date_trunc('day', created_at) AS day, sum(total) AS revenue
FROM orders GROUP BY 1;

REFRESH MATERIALIZED VIEW daily_sales;                       -- recompute (locks reads)
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales;          -- non-blocking (needs a UNIQUE index on it)
```

> Use a plain **view** for convenience/security (hide columns). Use a **materialized view** for
> expensive aggregates you read often and can tolerate being slightly stale (refresh on a cron).

---

## DROP / TRUNCATE

```sql
DROP TABLE products;                  -- structure + data, gone
DROP TABLE IF EXISTS products;        -- no error if absent
DROP TABLE orders CASCADE;            -- also drops dependent FKs/views — be careful
TRUNCATE orders;                      -- empty fast (no row-by-row delete, no triggers by default)
TRUNCATE orders RESTART IDENTITY;     -- also reset the id sequence to 1
TRUNCATE orders, order_items CASCADE; -- truncate FK-linked tables together
```

> `TRUNCATE` vs `DELETE`: `TRUNCATE` is near-instant (deallocates pages, resets nothing FK-checked),
> can't be filtered with `WHERE`, and **can't be rolled back in MySQL** — but in **Postgres it IS
> transactional**, so `BEGIN; TRUNCATE …; ROLLBACK;` is safe. Still: it's a sledgehammer.

---

## Roles & grants

```sql
CREATE ROLE appuser LOGIN PASSWORD 'secret';        -- a login role (= "user")
CREATE ROLE readonly;                                -- a group role (no login)

GRANT CONNECT ON DATABASE shop TO appuser;
GRANT USAGE ON SCHEMA public TO appuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO appuser;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
GRANT readonly TO appuser;                            -- appuser inherits readonly's grants

-- make grants apply to FUTURE tables too (the part everyone forgets)
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO readonly;

REVOKE INSERT ON orders FROM appuser;
\du                                                   -- psql: list roles
```

> See [Linux/PostgreSQL.md](../Linux/PostgreSQL.md) and [Linux/HA-PostgreSQL.md](../Linux/HA-PostgreSQL.md)
> for server install, `pg_hba.conf` auth, and replication.
