# PostgreSQL Query Optimization — Complete Book Summary

**Book:** *PostgreSQL Query Optimization: The Ultimate Guide to Building Efficient Queries*
**Authors:** Henrietta Dombrovskaya, Boris Novikov, Anna Bailliekova
**Publisher:** Apress, 2021

---

## Core Philosophy

> "Think like a database." — The book's central mantra.

The book rejects the "write first, optimize later" approach. Instead, **optimization is an integrated part of query development** — you should write queries correctly from the start. SQL is declarative: you specify *what* you want, not *how* to get it. Two queries yielding the same result can have drastically different execution times because the database engine chooses different execution paths. Understanding how the engine works lets you write queries that naturally guide the optimizer toward efficient plans.

**PostgreSQL has no optimizer hints by design.** The core team believes in investing in a query planner capable of choosing the best execution path without hints. This makes it even more critical to write SQL declaratively, allowing the optimizer to do its job.

---

## Part I: Theoretical Foundations (Chapters 1–4)

### Chapter 1 — Why Optimize?

**Declarative vs. Imperative Thinking:** SQL is declarative — you describe the desired result, not the steps. The danger is writing SQL that is *imperative by nature*, where you hard-code the sequence of operations (nested CTEs building results step by step) instead of letting the optimizer choose the best order.

**Key example from the book:** A query finding frequent flyers departing from Chicago written as nested CTEs (imperative style) locks the optimizer into a fixed execution order. The same query written as a flat `JOIN` with `WHERE` clauses (declarative style) allows the optimizer to choose the best join order based on data distribution.

**Optimization Goals — Use SMART goals:**

| Characteristic | Bad Example | Good Example |
|---|---|---|
| Specific | "All pages should respond fast" | "Each function should complete before system timeout" |
| Measurable | "Customers shouldn't wait too long" | "Registration page response ≤ 4 seconds" |
| Achievable | "Refresh time should never increase" | "Refresh grows at most logarithmically with data volume" |
| Result-based | "Run as fast as possible" | "Refresh time short enough to avoid lock waits" |
| Time-bound | "Optimize as many reports as we can" | "All financial reports under 30 seconds by month-end" |

**Optimization starts from requirements and design.** Database design choices (one table vs. two tables for phone numbers) directly impact query performance. Optimization is a continuous process — nothing is optimized forever.

### Chapter 2 — Theory: Relational Operations

**Query processing pipeline:**
1. **Compile** SQL → logical plan (high-level operations, still declarative)
2. **Optimize** logical plan → physical execution plan (algorithms chosen, order possibly changed)
3. **Execute** the plan → return results

**Core relational operations:**
- **Filter (restriction):** Keeps rows satisfying a condition (`WHERE`)
- **Project:** Selects specific columns (`SELECT column_list`)
- **Product (Cartesian):** All pairs of rows from two tables
- **Join:** Product + filter — but no sane optimizer computes it that way

**Critical equivalence rules** that enable optimization:
- **Commutativity:** `JOIN(R,S) = JOIN(S,R)` — join order doesn't affect results
- **Associativity:** `JOIN(R, JOIN(S,T)) = JOIN(JOIN(R,S), T)` — can regroup joins
- **Distributivity:** `JOIN(R, UNION(S,T)) = UNION(JOIN(R,S), JOIN(R,T))`

These rules mean the optimizer has a huge space of equivalent expressions to choose from, allowing it to find efficient plans. **Temporary tables and forced ordering destroy this flexibility.**

**Think in sets.** Relational operations accept tables and return tables. Results can be passed directly to the next operation without intermediate storage.

### Chapter 3 — Algorithms

**Cost model:** The optimizer uses I/O operations (disk blocks read/written) and CPU cycles to estimate costs. The lower the combined cost, the better the plan.

**Data Access Algorithms:**

| Algorithm | When Best | Cost Behavior |
|---|---|---|
| **Full scan (Seq Scan)** | High selectivity (need most rows) | Linear — reads all blocks |
| **Index-based access** | Low selectivity (need few rows) | Grows with selectivity; worse than full scan at ~2–5% on HDD, higher on SSD |
| **Index-only scan** | All needed columns are in the index | Lowest cost if applicable |

**The crossover point:** For small selectivity, index access wins. For large selectivity, full scan wins. The position depends on hardware. The optimizer estimates both selectivity and the crossover point.

**Index Structures:**
- An index is: (1) redundant, (2) invisible to the application, (3) designed to speed up data selection
- **B-tree:** Most common. Supports equality, range, and ordering. Depth = `log(N)/log(f)` where f = pointers per block. An index with 6–7 levels can accommodate billions of records.
- **Hash:** Better for equality-only. Useless for range queries.
- **Bitmaps:** Used internally by PostgreSQL to combine multiple index scans using AND/OR on block bitmaps.
- **R-tree / GiST:** For spatial/multidimensional data.

**Join Algorithms:**

| Algorithm | Best For | Cost |
|---|---|---|
| **Nested Loops** | Small tables, index-based inner loop | `size(R) × size(S)` |
| **Hash Join** | Large tables, equi-joins | `size(R) + size(S) + output` |
| **Sort-Merge** | Pre-sorted inputs, equi-joins | `sort cost + size(R) + size(S)` |

**No algorithm is universally best.** Nested loops excel for small index-based joins; hash join and sort-merge excel for large tables.

### Chapter 4 — Understanding Execution Plans

**Reading execution plans:**
- Use `EXPLAIN` (estimates only) or `EXPLAIN ANALYZE` (actual execution)
- Plans are trees: execution starts from leaves (rightmost offset), ends at root
- Each node shows: algorithm name, estimated cost (startup..total), expected rows, average row width
- All numbers are **approximate** — actual values may differ

**How optimization works:**
1. **Query rewrite** — eliminates subqueries, substitutes views with their source code
2. **Cost-based optimization** — determines possible orders, algorithms, compares costs, selects optimal plan

**Why the optimizer can be wrong:**
- Cost formulas are approximations (assume uniform distribution)
- Histograms help but can't cover intermediate results
- Heuristics may cut out the optimal plan
- The plan space for N tables is enormous (N! possible join orders × algorithm choices per join)

**Critical insight:** Nearly identical queries can produce very different plans based on filtering values. A filter selecting `iso_country='US'` (many airports) gets a sequential scan, while `iso_country='CZ'` (one airport) gets an index scan. The optimizer uses histograms of value distribution.

---

## Part II: Short Queries and Indexes (Chapter 5)

### Defining "Short"

> A query is **short** when the number of rows needed to compute its output is small, no matter how large the involved tables are. Short queries may read every row from small tables but only a small percentage from large tables.

**It's not about SQL length or result set size.** A 4-line Cartesian product is long. A 15-line join returning 3 rows is short. A `SELECT avg(...)` over all bookings producing one row is long.

**Optimization goal for short queries:** Apply the most restrictive selection criteria first. Ensure all intermediate results remain small. No full scans of large tables.

### Index Selectivity

Lower selectivity = fewer rows per index value = faster search. **The best indexes are unique indexes.** Foreign keys do NOT automatically create indexes — you must create them explicitly.

Don't create indexes on columns with very few distinct values (e.g., `aircraft_code` with only 12 values). The optimizer would never use it.

### Column Transformations Kill Indexes

B-tree indexes cannot be used when the column value is transformed:
- `WHERE lower(last_name) = 'daniels'` — **won't use** index on `last_name`
- `WHERE scheduled_departure::date = '2020-10-14'` — **won't use** index on `scheduled_departure`
- `WHERE coalesce(actual_departure, scheduled_departure) BETWEEN ...` — **won't use** either index

**Solutions:**
1. **Rewrite the condition:** Use `BETWEEN '2020-08-17' AND '2020-08-19'` instead of `::date`
2. **Create a functional index:** `CREATE INDEX ON account (lower(last_name))`
3. **Create a pattern index** for `LIKE`: `CREATE INDEX ON account (lower(last_name) text_pattern_ops)`

**Extremely common mistake:** `WHERE update_ts::date = CURRENT_DATE` — blocks index usage. Correct: `WHERE update_ts >= CURRENT_DATE`

### Compound Indexes

An index on `(X, Y, Z)` supports searches on: `X`, `X+Y`, `X+Y+Z`, even `X+Z` — but **NOT** `Y` alone, `Z` alone, or `Y+Z`.

Column order matters. Put the most restrictive column first. Compound indexes provide:
- **Lower selectivity** than individual indexes (184 ORD→JFK flights vs. 12,922 from ORD)
- **Index-only scans** when all needed columns are in the index

### Covering Indexes (PostgreSQL 11+)

`INCLUDE` adds columns to the index for retrieval only (not as search keys):
```sql
CREATE INDEX ON flight (departure_airport, arrival_airport, scheduled_departure)
INCLUDE (scheduled_arrival);
```

### Excessive Selection Criteria

When the true filtering condition spans multiple tables and can't be indexed, add a redundant but indexable filter. Example: An exception report looking for delayed flights has conditions spanning `flight` and `boarding_pass` tables. Adding `AND bp.update_ts >= '2020-08-16'` (a business-driven time limit) enabled index use and reduced execution from **2 minutes 44 seconds → 200 milliseconds**.

### Partial Indexes

Build indexes on subsets of data:
```sql
CREATE INDEX flight_canceled ON flight(flight_id) WHERE status='Canceled';
```
Useful when values are unevenly distributed. A column with 3 values where one is rare benefits from a partial index on the rare value.

### When PostgreSQL Ignores Your Index

The optimizer may choose sequential scan when:
- The table is small enough to fit in memory
- The index selectivity is too high (too many matching rows)
- `LIMIT` is present and order is unspecified (it's faster to scan sequentially and stop early)

**Let PostgreSQL do its job.** The optimizer dynamically adjusts based on data statistics. Different parameter values for the same query structure can produce completely different plans.

---

## Part III: Long Queries and Full Scans (Chapters 6–7)

### Defining "Long"

> A query is **long** when query selectivity is high for at least one large table — almost all rows contribute to the output, even when the output size is small.

### Key Principles for Long Queries

1. **Indexes are NOT needed** — full scans are preferred. Index access on a large fraction of rows is MORE expensive than a sequential scan.
2. **Hash joins are preferred** over nested loops for large tables.
3. **Most restrictive joins should execute first** — even without indexes, join order matters.
4. **Avoid multiple table scans.**
5. **Reduce result size at the earliest possible stage.**

### Semi-joins and Anti-joins

**Semi-join** (`EXISTS`, `IN`): Returns rows from R where at least one matching row exists in S. **Never increases** result set size — often the most restrictive join.

```sql
-- Preferred syntax (guarantees SEMI JOIN in plan):
SELECT * FROM flight f WHERE EXISTS
  (SELECT flight_id FROM booking_leg WHERE flight_id=f.flight_id)
```

**Anti-join** (`NOT EXISTS`, `NOT IN`): Returns rows from R where NO matching row exists in S.

```sql
-- Preferred syntax (guarantees ANTI JOIN in plan):
SELECT * FROM flight f WHERE NOT EXISTS
  (SELECT flight_id FROM booking_leg WHERE flight_id=f.flight_id)
```

### Controlling Join Order: `join_collapse_limit`

Default = 8. If tables in a join exceed this, the optimizer executes joins in the listed order without cost-based optimization. The number of possible plans for N tables is N! — at 10 tables, that's 3 million plans; at 20, it exceeds integer limits.

- Set `join_collapse_limit = 1` to force the order you specify in the `FROM` clause
- This is a **session-level** parameter

### Grouping Strategy

**Filter First, Group Last (most cases):** Push filtering conditions inside `GROUP BY`. Modern PostgreSQL does this automatically for constant filters but NOT for non-constant criteria (subquery filters).

**Group First, Select Last (special cases):** When grouping reduces the intermediate dataset significantly. Example: Count passengers per flight first, then join to get city names — reduced execution from 7+ minutes to 2.5 minutes.

### SET Operations

- `EXCEPT` instead of `NOT IN` / `NOT EXISTS` — can be faster (2× in one example)
- `INTERSECT` instead of `IN` / `EXISTS`
- `UNION ALL` instead of complex `OR` criteria — improves code maintainability

### Avoiding Multiple Scans (EAV Tables)

**Anti-pattern:** Joining an EAV table N times to retrieve N attributes = N full scans.

**Solution:** Join once, use `CASE` statements in `SELECT`:
```sql
SELECT passenger_id,
  max(CASE WHEN custom_field_name='passport_num' THEN custom_field_value END) AS passport_num,
  max(CASE WHEN custom_field_name='passport_exp_date' THEN custom_field_value END) AS passport_exp_date
FROM custom_field GROUP BY 1
```

**Further optimization:** Pull EAV values into a subquery BEFORE joining to other tables.

### Chapter 7 — Additional Techniques

**Temporary Tables — Usually Harmful:**
- Lose source table indexes and statistics
- Consume tempdb space competing with joins/sorts
- Block the optimizer from rewriting queries
- Lock in suboptimal join order

**CTEs — Better than Temp Tables:**
- PostgreSQL 12+: CTEs used once are automatically inlined (optimization fence removed)
- CTEs used multiple times: still materialized (old behavior)
- Use `MATERIALIZED` / `NOT MATERIALIZED` keywords to override
- Tables in CTEs don't count against `join_collapse_limit`

**Views — Use with Caution:**
- The optimizer substitutes views with their source SQL (views are NOT tables)
- Simple constant filters pushed inside views correctly
- Non-constant criteria (joins to other tables) **cannot** be pushed inside — causes full computation
- Column transformations inside views are invisible to users, causing unexpected slowness
- **Best use:** Security layer or reporting entity to ensure correct joins/business logic

**Materialized Views:**
- Store query results + query definition
- Behave like tables (indexes allowed, optimizer won't expand them)
- Cannot refresh incrementally in PostgreSQL — full truncate + re-insert
- `REFRESH MATERIALIZED VIEW CONCURRENTLY` — requires unique index, doesn't block reads
- Good when: data changes rarely, many reads per refresh, multiple queries benefit

**Partitioning:**
- Range partitioning most common (e.g., by month)
- Partition pruning: if query filters on partitioning key, only relevant partitions scanned
- **Very useful for long queries** with full scans
- **Limited benefit for short queries** — B-tree depth reduction is typically one level
- Enables fast `DROP PARTITION` instead of slow bulk `DELETE`

**Parallelism:**
- Beneficial for bulk scans and hash joins (long queries)
- Negligible speed-up for short queries
- Cannot fix poor design: parallelism gives at best linear improvement; nested loop cost is quadratic

---

## Part IV: DML, Design, and Application Architecture (Chapters 8–10)

### Chapter 8 — Optimizing Data Modification

**Writes are deferred:** Modifications happen in memory first; WAL records are forced to disk only on commit. This makes individual DML appear fast.

**PostgreSQL never updates in place.** A new row version is inserted; old version is marked outdated. `VACUUM` reclaims dead tuple space.

**Mass updates:** Can significantly slow subsequent SELECTs due to dead tuples reducing active rows per block. Requires aggressive vacuuming.

**HOT (Heap-Only Tuples):** If the new row version fits in the same block AND no indexed columns changed, no index update is needed. Use `fillfactor` (default 100, lower = more free space per block) for frequently updated tables.

**Index overhead on inserts:** Only ~1% per extra index according to multiple PostgreSQL experts. Don't fear creating indexes for OLTP.

**Foreign keys and triggers:** Each INSERT/UPDATE triggers implicit SELECTs to validate referential integrity. Small lookup parent tables = negligible cost. Large parent tables = noticeable overhead.

### Chapter 9 — Design Matters

**Design directly impacts performance.** Poor design cannot always be compensated by better queries or indexes.

**One-table vs. two-table design:** Depends on access patterns. Phone numbers in a separate table = fast search by any phone. Phone columns in account table = simpler reporting.

**EAV, Key-Value, and JSON models:**
- Provide flexibility at the cost of performance, type safety, and referential integrity
- `custom_field_value` as text prevents type checking, proper indexing, and constraint enforcement
- JSON columns: Use only when data is consumed as a whole object. Parse searchable attributes into separate columns.

**Normalization:** Primary purpose is data integrity, not performance. However, normalization CAN improve performance (e.g., selecting distinct values from a normalized lookup vs. `DISTINCT` on a denormalized column).

**Surrogate Keys:**
- Useful when no natural key exists
- Can hide data quality issues (same real-world object stored multiple times)
- Cause extra joins (airport codes as natural keys eliminate joins to the airport table)
- Use natural keys when they exist and are stable (airport codes, ISO codes)

### Chapter 10 — Application Development and Performance

**The Shopping List Problem:** Applications often execute thousands of tiny queries sequentially (N+1 problem) instead of one efficient query returning all needed data. A form with 100 fields triggering 16,000 queries could be served by a single query in 200ms.

**Solutions that DON'T work:**
- More powerful computers (both app and DB are in wait state 99% of time)
- Higher network bandwidth (roundtrip count matters, not message size)
- Distributed servers (may improve throughput, not response time for sequential calls)

**ORM is a major contributor:** Maps objects 1:1 to tables, generates separate queries per object. To process N objects: N+1 queries (shopping list pattern). Hides implementation details — developers don't know a simple `.IsActive` attribute triggers a complex query.

**The real solution:** Transfer collections of complex objects in a single database call. PostgreSQL supports this through custom types, functions returning sets, and JSON/JSONB.

---

## Part V: Functions, Dynamic SQL, and NORM (Chapters 11–13)

### Chapter 11 — Functions

**Functions are NOT compiled in PostgreSQL** — they are interpreted. `CREATE FUNCTION` only checks trivial syntax. Column/table existence is NOT validated until execution. Errors in conditional branches may go undiscovered until that branch runs.

**Functions as optimization fences:** The optimizer knows nothing about what happens inside a function. Each invocation = separate execution. Using a scalar function in a `SELECT` list on 10,000 rows = 10,000 separate function executions.

**Performance comparison from the book:**
- Inline SQL calculating passengers per flight: **900ms**
- Same calculation via `num_passengers(flight_id)` in SELECT list: **3.5 seconds**

**When functions HELP performance:** Not for individual queries, but for **process optimization** — replacing hundreds of application roundtrips with a single function call returning a complex object.

**User-defined composite types:** Allow functions to return structured records matching application objects. Nested structures (booking → booking_legs[] → boarding_passes[]) are possible but lose field names/types when nested.

**Security:** `SECURITY DEFINER` functions execute with creator's permissions — useful for giving controlled data access to power users without granting broad table permissions.

### Chapter 12 — Dynamic SQL

**Why dynamic SQL works better in PostgreSQL:** Since functions are interpreted, each `EXECUTE` of dynamic SQL gets a fresh plan optimized for the actual parameter values. This is BETTER than a prepared statement when parameter values have very different selectivities.

**Use cases:**
- OLTP: Building WHERE clauses dynamically based on which search criteria the user provides
- OLAP: Parameterizing table/column names, aggregation functions, time periods
- Aiding the optimizer when it chooses wrong plans due to parameter sniffing

**SQL injection prevention:** Always use `quote_literal()`, `quote_ident()`, and the `format()` function.

### Chapter 13 — NORM (Not an ORM)

**NORM** is a contract-driven approach: application and database developers agree on complex object structures (types), then:
1. Database functions return complete objects (including nested structures) as JSON-over-text
2. The `array_transport()` function converts any array of user-defined types to JSON text transportable through JDBC
3. Search functions use dynamic SQL for flexible criteria
4. DML functions accept JSON objects and parse them into appropriate INSERT/UPDATE/DELETE operations

**Performance gains:** 10–50× improvement over traditional ORM in application controller response time. Eliminates the N+1 problem entirely.

---

## Part VI: Advanced Indexing and the Ultimate Algorithm (Chapters 14–15)

### Chapter 14 — Complex Filtering and Search

**Full Text Search:** Documents → `ts_vector` (list of terms). Queries → `ts_query`. Match operator `@@`. Works without indexes but benefits from GIN or GiST indexes.

**GIN (Generalized Inverted) indexes:** For each term, stores a list of documents containing it. Efficient for full text search and arrays. Also supports JSONB searching with `@>` and `@@` operators.

**GiST indexes:** For spatial/multidimensional data. Supports range queries and nearest-neighbor queries.

**BRIN (Block Range Index):** For very large tables with naturally ordered data. Stores min/max per block range. Much smaller than B-tree but effective only when data ordering matches query patterns.

**JSONB indexing:** GIN indexes support `@>`, `@@`, and jsonpath queries. But:
- Still 2–2.5× slower than B-tree searches via NORM functions
- Don't support date/time searches, LIKE, or transformed attributes
- JSON structure supports only one hierarchy — can't update cross-hierarchy data without rebuilding

### Chapter 15 — The Ultimate Optimization Algorithm

```
Step 1: Is the query SHORT or LONG?
        (Gather requirements. Ask the business about data scope.)
        │
        ├── SHORT → Step 2
        │   ├── 2.1: Find most restrictive criteria
        │   ├── 2.2: Check/create indexes for those criteria
        │   │        (compound? covering? index-only scan?)
        │   ├── 2.3: Add excessive selection criteria if needed
        │   └── 2.4: Build query incrementally — most restrictive first,
        │            add one join at a time, check plan each time
        │
        └── LONG → Step 3: Can you use incremental refresh?
                   │
                   ├── YES → Step 4: Treat incremental as SHORT query
                   │         (time of update = most restrictive criterion)
                   │
                   └── NO → Step 5: Full long query optimization
                             ├── Find most restrictive join/semi-join/anti-join
                             ├── Execute it first
                             ├── Add tables one by one, check plan each time
                             ├── Don't scan large tables multiple times
                             └── Postpone GROUP BY to last step
                                 (unless early grouping reduces intermediate size)
```

**Additional considerations:**
- Parameterized queries may need dynamic SQL when different parameter values change the most restrictive criterion
- Functions don't improve individual query speed but dramatically improve process performance
- Database design changes may be needed (indexes, schema)
- Application interaction patterns (NORM vs. ORM) often matter more than individual query tuning

---

## Quick Reference: Key Rules from the Book

| Rule | Chapter |
|---|---|
| Write declarative SQL — don't hard-code operation order | 1 |
| `::date` on a timestamp blocks index usage — use `BETWEEN` instead | 5 |
| `lower()`, `coalesce()`, any function on a column blocks its index | 5 |
| Compound index `(X,Y,Z)` works for X, XY, XYZ — not Y alone | 5 |
| Foreign keys do NOT auto-create indexes | 5 |
| For long queries: indexes harmful, full scans preferred | 6 |
| `EXISTS` guarantees SEMI JOIN; `IN` may be rewritten as regular JOIN | 6 |
| Filter before grouping; group before joining (when it reduces size) | 6 |
| Temp tables block optimizer rewrites — prefer CTEs (PG 12+) | 7 |
| Views are NOT tables — conditions may not push through | 7 |
| PostgreSQL never updates in place — VACUUM is essential | 8 |
| Functions are opaque to the optimizer — each call = separate execution | 11 |
| Dynamic SQL gets a fresh plan per EXECUTE — better for varied parameters | 12 |
| NORM (functions returning complex objects) = 10–50× over traditional ORM | 13 |
| GIN indexes for JSONB are 2–2.5× slower than B-tree via NORM functions | 14 |

---

*Summary based exclusively on the content of "PostgreSQL Query Optimization" by Dombrovskaya, Novikov, and Bailliekova (Apress, 2021). Uses the postgres_air airline booking database for all examples.*
