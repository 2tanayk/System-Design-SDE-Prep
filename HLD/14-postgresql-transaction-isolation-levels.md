# PostgreSQL Transaction Isolation — MVCC, Locks, Anomalies, and Experiments

Hands-on study notes based on two-session experiments in psql using a phone flash-sale store. The focus is PostgreSQL behavior: locks, MVCC snapshots, anomalies, isolation levels, practical fixes, and interview reasoning.

## 1. The big picture

Many transactions can run at the same time against the same data. Transaction isolation controls how much concurrent transactions can see and how their operations can affect one another.

PostgreSQL is easiest to reason about using two layers:

1. **Plain reads (SELECT):** ask which snapshot the statement or transaction sees. In the MVCC row-update scenarios covered here, ordinary reads do not wait for row-level write locks.
2. **Writes and locking reads (UPDATE, DELETE, SELECT ... FOR UPDATE):** may need a conflicting lock on a row. If another transaction holds it, they wait or may eventually fail.

### Wait versus abort

- **Wait:** another transaction holds a conflicting lock on the row or resource you need.
- **Abort:** a deadlock is detected, a timeout occurs, or PostgreSQL detects that the transaction's snapshot is no longer safe to use (for example, a concurrent update at Repeatable Read or a serialization failure).

The supplied study model characterizes PostgreSQL as more optimistic: transactions can proceed using snapshots and PostgreSQL aborts transactions when it detects certain conflicts. It characterizes MySQL InnoDB as more pessimistic because it uses locking more heavily in comparable situations. The exact behavior still depends on the isolation level and statement type.

## 2. Locks from scratch

### Shared and exclusive locks

| Lock | Meaning | Compatibility |
|---|---|---|
| Shared (S) | "I'm reading; don't change this" | Multiple shared locks can coexist |
| Exclusive (X) | "I'm changing this" | Conflicts with other requests for S or X |

Compatibility matrix:

| Held by another transaction → / Your request ↓ | S | X |
|---|---|---|
| S | OK | Wait |
| X | Wait | Wait |

Important points:

- Write locks normally last until COMMIT or ROLLBACK, not merely until the UPDATE statement ends.
- Longer transactions therefore make other transactions wait longer.
- An exclusive row lock blocks operations that request a conflicting lock. An ordinary PostgreSQL MVCC SELECT can read an older visible row version instead of waiting for the row lock.
- Common scopes include row locks, table locks, and (in some database engines) gap or range locks.
- **Deadlock:** transactions form a circular wait. For example, T1 holds A and wants B, while T2 holds B and wants A. The database detects the cycle and aborts one transaction.

### Textbook lock model for generic interview questions

Use this table when an interviewer asks for the standard lock-based explanation without naming a database. Do not use it to predict ordinary SELECT behavior in PostgreSQL.

| Isolation level | Read takes | Read lock held until | Write takes | Write lock held until |
|---|---|---|---|---|
| Read Uncommitted | Nothing | N/A | X | End of transaction |
| Read Committed | S | Immediately after the read | X | End of transaction |
| Repeatable Read | S | End of transaction | X | End of transaction |
| Serializable | S + range lock | End of transaction | X | End of transaction |

In this simplified model, a lock conflict causes waiting; a circular wait causes a deadlock and one transaction is aborted.

## 3. MVCC: snapshots and row versions

PostgreSQL uses Multi-Version Concurrency Control (MVCC).

### What is a snapshot?

A snapshot is **not a copy of the database**. It is metadata that tells PostgreSQL which transaction IDs count as committed and visible from the current transaction's point of view.

A snapshot includes:

- **xmin:** transaction IDs below this boundary are considered finished for snapshot visibility.
- **xmax:** transaction IDs at or above this boundary had not started when the snapshot was taken.
- **In-progress transaction list:** transactions running at snapshot time; their changes are invisible to that snapshot.

The important idea is that visibility is determined by the snapshot, not by checking the wall-clock time at which another transaction eventually commits.

A transaction that was in progress when your snapshot was taken stays invisible to that snapshot even if it commits later.

### Row-version stamps

A physical row version has transaction metadata:

- **xmin:** the transaction ID that created this row version.
- **xmax:** the transaction ID that deleted or replaced this row version; zero means it has not been marked as deleted/replaced.

An UPDATE does not normally edit a tuple in place. It marks the old row version with an xmax and creates a new row version. Old versions must remain available to transactions whose snapshots can still see them. VACUUM can reclaim obsolete versions once they are no longer needed.

A row version is visible when its creator is visible as committed to the snapshot and its deleting/replacing transaction is not visible as committed to that snapshot.

### Snapshot lifetime by isolation level

| Isolation level | Snapshot taken | Snapshot lifetime |
|---|---|---|
| Read Committed | At the beginning of each statement | One statement |
| Repeatable Read | At the first query, not merely at BEGIN | Whole transaction |
| Serializable | Same transaction-level snapshot behavior as Repeatable Read, plus conflict tracking | Whole transaction |

**Important gotcha:** BEGIN ISOLATION LEVEL REPEATABLE READ does not, by itself, pin the transaction's view. Run a SELECT to establish the snapshot.

### Inspect snapshots and row versions

    SELECT pg_current_snapshot();

    SELECT ctid, xmin, xmax, *
    FROM products;

Useful observations:

- At Repeatable Read, the snapshot value stays the same for the transaction even after other transactions commit.
- At Read Committed, each new statement gets a new snapshot.
- After another transaction updates a row, a new physical tuple version appears. Inspecting ctid, xmin, and xmax can help reveal this.

### Cost of long-running transactions

A long-running transaction can keep old row versions relevant to its snapshot. VACUUM cannot reclaim versions that may still be needed, so old versions can accumulate and contribute to table bloat. Keep transactions short.

## 4. What happens when UPDATE encounters a concurrent update at Repeatable Read?

A useful mental model:

- Reads use the transaction's snapshot.
- Writes must also check the current physical row and whether it changed after that snapshot.

When an UPDATE reaches a row:

1. Find the row using the transaction's snapshot.
2. Inspect the current row version and its transaction stamps.
3. Decide what to do:
   - If the row has not been changed since the snapshot, create the new version.
   - If another transaction is currently updating it but has not committed, wait.
   - If another transaction updated and committed it after this transaction's snapshot, abort with a concurrent-update serialization error.

The typical PostgreSQL error is:

    ERROR: could not serialize access due to concurrent update

SQLSTATE: 40001

Why abort? The old version has already been replaced, but the transaction's snapshot cannot see the new version. PostgreSQL cannot safely apply the write using the stale view.

This is commonly described as **first committer wins**.

### Contrast with Read Committed

At Read Committed, after waiting for a concurrent updater to finish, PostgreSQL rechecks the latest row version and re-evaluates the WHERE condition. If it still matches, the UPDATE can proceed against that newer version rather than aborting for the stale transaction snapshot.

## 5. Concurrency anomalies

| Anomaly | Meaning | Phone-store example | PostgreSQL behavior / prevention |
|---|---|---|---|
| Dirty read | Read another transaction's uncommitted data | See stock = 0 from a checkout that later rolls back | PostgreSQL never exposes true dirty reads; Read Uncommitted behaves as Read Committed |
| Dirty write | Overwrite another transaction's uncommitted change | Two checkouts write the same product row | Conflicting write locks make the second writer wait |
| Non-repeatable read | Read the same row twice and get different values | Price is 100, then 80 | Possible at Read Committed; prevented at Repeatable Read |
| Phantom | Repeat a predicate query and get a different set of rows | Order count changes from 0 to 1 | Possible at Read Committed; prevented by PostgreSQL Repeatable Read snapshots |
| Lost update | Two read-modify-write cycles overwrite one another | Two buyers both read stock 10 and both write 9 | Possible at Read Committed; use atomic update, locking, optimistic versioning, or Repeatable Read with retry |
| Read skew | Related reads observe values from different committed states | Auditor reads stock 10, checkout commits stock 9 and sold 1, auditor then reads sold 1: total appears as 11 | Possible at Read Committed; prevented by Repeatable Read or by reading related data in one statement |
| Write skew | Transactions read the same rule/state but write different rows, jointly violating the rule | Two tabs both see that a customer has two orders under a maximum of three, then each inserts one | Possible at Read Committed and Repeatable Read; use Serializable with retry, a guard-row lock, or a suitable constraint |
| Deadlock | Transactions wait in a cycle | Two transactions update products in opposite order | Can happen at any isolation level; use consistent lock ordering and retry the victim |

**Read skew** is the same underlying mechanism as non-repeatable read, but it involves related values or multiple rows rather than reading one row twice.

### PostgreSQL isolation-level behavior

| Behavior | Read Committed | Repeatable Read | Serializable |
|---|---|---|---|
| Snapshot | Per statement | Per transaction | Per transaction |
| Ordinary MVCC reads wait on row-level UPDATE locks? | No | No | No |
| Concurrent update of a row being written | Wait, then re-read/recheck the latest version | Wait; abort if the other transaction committed after this snapshot (40001) | Same stale-update protection, plus serialization checks |
| Non-repeatable reads | Possible | Prevented | Prevented |
| Phantoms | Possible | Prevented in PostgreSQL | Prevented |
| Read skew | Possible | Prevented | Prevented |
| Lost update in a read-then-write cycle | Possible | Prevented by aborting a stale writer | Prevented by aborting conflicting transactions |
| Write skew | Possible | Possible | Prevented by aborting a transaction |
| Application responsibility | Atomic updates or locks where required | Retry SQLSTATE 40001 | Retry SQLSTATE 40001 |

PostgreSQL treats READ UNCOMMITTED as READ COMMITTED.

**Important PostgreSQL distinction:** textbook descriptions often say phantoms can occur at Repeatable Read and that shared locks prevent write skew. PostgreSQL's Repeatable Read uses a stable MVCC snapshot, so it prevents phantoms, but it can still allow write skew when transactions read the same predicate and insert or update different rows.

## 6. Hands-on lab setup

Use two separate psql terminals connected to the same database. A third terminal is useful for observing lock waits.

### Create the database and schema

Run the database creation command from a connection where you have permission to create databases:

    CREATE DATABASE iso_lab;

Connect both experiment terminals to iso_lab:

    psql -d iso_lab

Create the tables:

    CREATE TABLE products (
        id INT PRIMARY KEY,
        name TEXT,
        price INT,
        stock INT
    );

    CREATE TABLE orders (
        id SERIAL PRIMARY KEY,
        customer TEXT,
        product_id INT,
        qty INT
    );

    ALTER TABLE products
    ADD COLUMN version INT NOT NULL DEFAULT 0;

### Reset between experiments

Run only when no experiment transactions are open:

    TRUNCATE products, orders RESTART IDENTITY;

    INSERT INTO products (id, name, price, stock)
    VALUES
        (1, 'Phone', 100, 10),
        (2, 'Case', 10, 100);

### Experiment rules

- Explicitly write BEGIN ISOLATION LEVEL ... for each transaction.
- Verify the setting with SHOW transaction_isolation.
- Statements outside an explicit transaction block autocommit.
- After an error, run ROLLBACK. In psql, an error leaves the transaction failed until it is rolled back.
- BEGIN and START TRANSACTION are equivalent ways to start a transaction in PostgreSQL.
- Predict the result before executing each statement.
- Use terminal A and terminal B exactly as separate sessions; do not run both sides in one psql session.

### Watch lock waits from a third terminal

    SELECT
        pid,
        state,
        wait_event_type,
        wait_event,
        pg_blocking_pids(pid) AS blocked_by,
        left(query, 45) AS query
    FROM pg_stat_activity
    WHERE datname = current_database()
      AND pid <> pg_backend_pid();

A blocked session commonly shows wait_event_type = Lock.

## 7. Experiments

### Experiment 1 — Non-repeatable read: price changes during checkout

Reset the data first.

| Terminal A | Terminal B |
|---|---|
| BEGIN ISOLATION LEVEL READ COMMITTED; | |
| SELECT price FROM products WHERE id = 1; returns 100 | |
| | UPDATE products SET price = 80 WHERE id = 1; autocommits |
| SELECT price FROM products WHERE id = 1; returns 80 | |

Repeat with terminal A using Repeatable Read. It sees 100 both times inside the transaction. After A commits and starts a new transaction, it sees 80.

**Why:** Read Committed takes a snapshot for each statement; Repeatable Read keeps the transaction's snapshot.

### Experiment 2 — Phantom: order count changes during a report

Reset the data first.

| Terminal A | Terminal B |
|---|---|
| BEGIN ISOLATION LEVEL READ COMMITTED; | |
| SELECT count(*) FROM orders WHERE product_id = 1; returns 0 | |
| | INSERT INTO orders (customer, product_id, qty) VALUES ('Sam', 1, 1); autocommits |
| SELECT count(*) FROM orders WHERE product_id = 1; returns 1 | |

Repeat with A at Repeatable Read. The count stays 0 in PostgreSQL because the transaction keeps its original snapshot.

The textbook lock model often teaches that phantoms can survive Repeatable Read and require Serializable range locking. PostgreSQL's snapshot-based Repeatable Read prevents this phantom in the experiment.

### Experiment 2.5 — Inspect row versions and snapshots

Inspect physical row metadata:

    SELECT ctid, xmin, xmax, id, price
    FROM products
    ORDER BY id;

In another terminal, update the Phone row:

    UPDATE products
    SET price = 80
    WHERE id = 1;

Inspect again:

    SELECT ctid, xmin, xmax, id, price
    FROM products
    ORDER BY id;

At Repeatable Read, after another transaction updates the row:

- The snapshot value in the observing transaction remains unchanged.
- The transaction can still see the old row version.
- That old physical version may now have a non-zero xmax because another transaction marked it as replaced.
- If the updating transaction was in progress when the snapshot was created, its transaction ID is in the snapshot's in-progress list. The observing transaction continues to treat its changes as invisible even if it commits later.

Optional low-level inspection, requiring the pageinspect extension:

    CREATE EXTENSION pageinspect;

    SELECT lp, t_xmin, t_xmax, t_ctid
    FROM heap_page_items(get_raw_page('products', 0));

### Experiment 3 — Lost update: two buyers decrement one stock counter

Reset the data first. Initial Phone stock is 10.

**Bug at Read Committed:**

1. A reads stock = 10.
2. B reads stock = 10.
3. A writes stock = 9 and commits.
4. B writes stock = 9 and commits.
5. Final stock is 9, although two purchases occurred. One decrement was lost.

Compare the fixes:

| Fix | How it works | B's behavior | Final stock |
|---|---|---|---|
| Atomic update | Database performs the decrement in one statement | Waits if needed, then applies decrement to latest row | 8 |
| SELECT ... FOR UPDATE | Acquire row lock when reading before deciding | B's locking read waits, then reads 9 | 8 |
| Repeatable Read | Both transactions pin snapshots first | Stale writer gets 40001 and must retry using a fresh transaction | 8 after retry |
| Optimistic locking | UPDATE checks a version value | Second writer affects zero rows, rereads, retries | 8; version becomes 2 |

**Fix A — atomic update**

    UPDATE products
    SET stock = stock - 1
    WHERE id = 1;

The database applies the arithmetic as one atomic statement. If another transaction is updating the row, the second updater waits and applies its decrement to the current row version.

**Fix B — lock before reading and deciding**

    BEGIN;

    SELECT stock
    FROM products
    WHERE id = 1
    FOR UPDATE;

    -- Application validates stock, then performs the update.

    UPDATE products
    SET stock = stock - 1
    WHERE id = 1;

    COMMIT;

A competing transaction's SELECT ... FOR UPDATE waits for the lock, then obtains the current row after the first transaction completes.

**Fix C — Repeatable Read with retry**

Both transactions read using a transaction-level snapshot. If one commits a change to the same row after the other's snapshot was taken, the stale writer receives 40001. Roll back and retry the whole business operation in a new transaction.

**Fix D — optimistic locking with a version column**

For a transaction that previously read stock = 10 and version = 0:

    UPDATE products
    SET stock = 9,
        version = version + 1
    WHERE id = 1
      AND version = 0;

If another transaction already changed the row and incremented the version, this update affects zero rows. The application detects the conflict, rereads the current state, and retries.

Use:
- Atomic UPDATE for simple arithmetic.
- SELECT ... FOR UPDATE for read-decide-write workflows with significant contention.
- Optimistic locking when conflicts are relatively rare.
- Repeatable Read plus retry when transaction-wide snapshot semantics are useful.

### Experiment 4 — Read skew

Invariant: stock + units sold = 10.

At Read Committed:

1. Auditor reads stock = 10.
2. A checkout commits stock = 9 and one additional unit sold.
3. Auditor reads units sold = 1.
4. Auditor calculates 10 + 1 = 11, mixing values from two different committed states.

At Repeatable Read, the auditor sees one transaction snapshot: stock = 10 and units sold = 0, so the total remains 10.

Reading related values in a single SQL statement can also ensure they are read from one statement snapshot.

This experiment is described here as a walkthrough rather than a full separate setup.

### Experiment 5 — Write skew: maximum three orders per customer

Initial setup after reset:

    INSERT INTO orders (customer, product_id, qty)
    VALUES
        ('Riya', 1, 1),
        ('Riya', 1, 1);

Riya has two orders. Two application tabs independently check that the customer is below a maximum of three orders.

**Bug at Repeatable Read:**

| Transaction A | Transaction B |
|---|---|
| BEGIN ISOLATION LEVEL REPEATABLE READ; | |
| SELECT count(*) ... WHERE customer = 'Riya'; returns 2 | |
| | BEGIN ISOLATION LEVEL REPEATABLE READ; |
| | SELECT count(*) ... WHERE customer = 'Riya'; returns 2 |
| INSERT one Riya order | |
| | INSERT one Riya order |
| COMMIT | |
| | COMMIT |

Both transactions can commit, leaving four orders and breaking the maximum-three rule.

**Why PostgreSQL Repeatable Read does not prevent this:** each transaction inserts a new row. The two transactions do not update the same existing row, so the lost-update protection for concurrent updates does not trigger. Both snapshots were valid when read.

#### Fix 1 — Serializable with retry

Run the same transaction logic at SERIALIZABLE. PostgreSQL detects the dangerous read/write dependency pattern and aborts one transaction with SQLSTATE 40001. After rollback and retry, the transaction sees the updated count of three and refuses the additional order.

The error may be raised at a statement or at COMMIT. Either transaction can be chosen as the victim.

#### Fix 2 — guard row at Read Committed

Every purchase first locks the same existing product row:

    BEGIN ISOLATION LEVEL READ COMMITTED;

    SELECT *
    FROM products
    WHERE id = 1
    FOR UPDATE;

    SELECT count(*)
    FROM orders
    WHERE customer = 'Riya';

    -- Insert only if the count is below 3.

    COMMIT;

The second transaction waits for the guard-row lock. After the first commits, it obtains the lock and checks the count again, now seeing three.

This works only if every code path that can affect the rule takes the same guard lock. You cannot lock a row that does not exist yet, which is why the guard is an existing row that all relevant operations must pass through.

#### Fix 3 — database constraint

Use a database constraint when the business rule can be expressed declaratively, such as UNIQUE, CHECK, or EXCLUDE USING gist.

A simple maximum count such as "at most three rows per customer" is not expressible as an ordinary row-level CHECK or UNIQUE constraint. It needs another design, such as a guard row/counter or Serializable transaction logic.

### Experiment 6 — Deadlock: opposite lock order

Reset the data first.

| Terminal A | Terminal B |
|---|---|
| BEGIN; | |
| UPDATE products SET stock = stock - 1 WHERE id = 1; holds Phone row | |
| | BEGIN; |
| | UPDATE products SET stock = stock - 1 WHERE id = 2; holds Case row |
| UPDATE ... WHERE id = 2; waits | |
| | UPDATE ... WHERE id = 1; creates a circular wait |

After PostgreSQL's deadlock detection, one transaction receives:

    ERROR: deadlock detected

SQLSTATE: 40P01

Notes:

- PostgreSQL waits for deadlock_timeout before checking for a cycle; the default is commonly one second.
- Which transaction is chosen as the victim depends on timing.
- The victim's transaction is rolled back and its locks are released, allowing the survivor to continue.
- Deadlocks can occur at any isolation level. They are caused by conflicting lock acquisition order, not by a particular isolation level.

**Fix: acquire locks in a consistent order.**

For example, always update product IDs in ascending order. For a multi-row locking read:

    SELECT *
    FROM products
    WHERE id IN (1, 2)
    ORDER BY id
    FOR UPDATE;

Both transactions now attempt to acquire locks in the same order, reducing the chance of a circular wait.

## 8. Serializable in PostgreSQL: SSI and retries

### Mental model

PostgreSQL Serializable can be thought of as:

**Repeatable Read + a watcher that tracks read/write dependencies.**

It records what transactions read and detects dangerous overlaps where transactions can each miss another transaction's writes. PostgreSQL cancels a transaction when the concurrent execution cannot be safely explained by a serial order.

The transaction that loses receives SQLSTATE 40001 and must be retried. Serializable does not simply mean that every reader takes a blocking shared lock.

### SSI: Serializable Snapshot Isolation

PostgreSQL uses Serializable Snapshot Isolation (SSI):

- Reads use snapshots.
- Read tracking markers, visible as SIReadLock entries, are not ordinary blocking locks.
- They do not block writers.
- PostgreSQL tracks read/write dependencies and aborts a transaction when it detects a dangerous structure that could produce a non-serializable result.

Inspect predicate-lock tracking:

    SELECT pid, relation::regclass, locktype, mode
    FROM pg_locks
    WHERE mode = 'SIReadLock';

### Why write skew is detected

If two transactions each read a count of two and each insert a third row, they have both made decisions using a snapshot that missed the other's write. Their overlapping read/write dependencies can create a schedule that cannot be explained by a safe serial order.

PostgreSQL aborts one transaction. When retried, it sees the other transaction's committed row and can enforce the limit.

### False positives and indexes

Serializable predicate tracking may be conservative.

For example, two unrelated customers (Riya and Sam) can sometimes conflict in the system's tracking. Without a useful index on the customer filter, PostgreSQL may scan a broad part of the table and use a broader read marker. It may not be able to prove that the transactions are unrelated, so it can abort one conservatively.

An index can make predicate tracking more precise in some cases, reducing false conflicts, but it cannot guarantee that false positives disappear entirely.

### Who gets cancelled?

The exact victim depends on the dependency pattern and timing. A committed transaction is not cancelled; PostgreSQL cancels an active transaction when needed to preserve serializability. Do not rely on a particular transaction always being the victim.

### Rules for application engineers

1. Start the transaction explicitly at SERIALIZABLE.
2. Retry the **whole transaction** on SQLSTATE 40001. A deadlock error, 40P01, is also commonly retried.
3. Cap retries (for example, 3–5) and add a small randomized delay between attempts.
4. A serialization error can be reported at COMMIT, not only at an individual statement.
5. Re-run all business reads and decisions on every retry using a fresh transaction.
6. Make transaction work safe to execute again. Do not send emails, charge cards, or call external APIs inside a transaction that may be retried; perform such side effects after a successful commit or use an appropriate outbox/idempotency design.
7. Keep transactions short. Do not hold them open while waiting for user input or slow external calls.
8. Index columns used in selective predicates to reduce unnecessarily broad predicate tracking where possible.
9. For read-only reports, PostgreSQL supports SERIALIZABLE READ ONLY DEFERRABLE. It waits for a safe snapshot and the resulting transaction can run without serialization failure.
10. Serializable has CPU and memory overhead. Use it where correctness requires it rather than for every transaction.
11. Prefer a constraint for a rule that can be expressed declaratively, and an atomic update for a simple counter. Serializable is useful for business rules spanning multiple rows when explicit locking is difficult.

### Retry pseudocode

    for attempt in 1..5:
        BEGIN ISOLATION LEVEL SERIALIZABLE

        run all business reads and decisions again
        perform database writes

        try COMMIT
        if SQLSTATE is 40001 or 40P01:
            ROLLBACK
            sleep for a small random delay (for example, 10–100 ms)
            continue with a fresh transaction
        otherwise:
            return success

    report failure after the retry limit

The retry must create a fresh transaction and repeat the entire unit of business logic. Retrying only the failed SQL statement is not sufficient.

## 9. Theory: optional exam and theory-heavy interview material

### Serializability

A schedule is **serializable** when its result is equivalent to the result of some serial (one-transaction-at-a-time) order.

### Conflicting operations and precedence graphs

Operations conflict when they belong to different transactions, access the same item, and at least one is a write. The conflict types are:

- Read–Write (R-W)
- Write–Read (W-R)
- Write–Write (W-W)

A precedence graph has one node per transaction. Add an edge Ti → Tj when an operation of Ti conflicts with and occurs before an operation of Tj.

A cycle means the schedule is not conflict-serializable. The write-skew example can create a cycle such as A → B → A.

### View serializability

View serializability is broader than conflict serializability. It considers whether a schedule has the same reads and final writes as some serial schedule. The difference involves blind writes. Testing view serializability is NP-complete, which is why databases generally do not enforce it directly.

### How databases enforce serializability

- **Strict two-phase locking (2PL):** acquire locks as needed and retain them until commit. This is the textbook lock-based model and can cause deadlocks.
- **SSI:** PostgreSQL's approach uses snapshots and read/write dependency tracking. It detects dangerous structures and aborts transactions rather than making all reads block.

### Phantoms and predicates

Row-level conflict tracking cannot lock a row that does not exist yet. Preventing changes to a queried set therefore requires predicate or range-level reasoning. This is why phantom behavior is an important part of isolation theory.

## 10. Fix toolbox

| Fix | How it works | Best fit |
|---|---|---|
| Atomic UPDATE | Database performs read-modify-write in one statement | Counters and simple arithmetic |
| SELECT ... FOR UPDATE | Acquires a row lock before the application makes its decision | Read-decide-write workflows, especially with high contention |
| Guard row | Locks a shared existing row that all relevant operations must use | Rules spanning rows that may not exist yet |
| Optimistic locking | WHERE version = expected_value makes a conflicting update affect zero rows | Relatively rare conflicts |
| Repeatable Read + retry | PostgreSQL aborts a writer using a stale snapshot | Transaction-level stable reads with concurrent-update protection |
| Serializable + retry | PostgreSQL detects unsafe overlapping transactions | Business rules spanning multiple rows |
| Constraint (UNIQUE, EXCLUDE, etc.) | Database directly enforces an expressible invariant | Rules that can be declared as constraints |
| Consistent lock order | Makes transactions acquire resources in the same order | Deadlock prevention |
| NOWAIT / SKIP LOCKED | Fail immediately or skip rows that are already locked | Work queues and workflows that should avoid waiting |

## 11. PostgreSQL vs. MySQL InnoDB — quick contrast

The supplied study notes use this simplified comparison:

| Behavior | PostgreSQL | MySQL InnoDB |
|---|---|---|
| General approach | MVCC snapshots with optimistic conflict detection in the cases covered | More lock-oriented/pessimistic behavior in comparable cases |
| Ordinary plain reads | Snapshot reads; do not wait for row-level write locks | Snapshot reads below Serializable; do not wait for row-level write locks |
| Serializable | SSI: tracks dependencies and can abort with 40001 | Plain SELECT becomes a shared-locking read; deadlocks can occur |
| Repeatable Read phantoms | Prevented by PostgreSQL's transaction snapshot | Prevented for locking reads through next-key/gap locking |
| Concurrent update after snapshot | Can abort with a serialization/concurrent-update error | Update commonly waits and then applies to the latest row |
| Read Uncommitted | Behaves as Read Committed | A distinct isolation level is supported |
| Lock wait timeout | lock_timeout is configurable and off by default | innodb_lock_wait_timeout; commonly defaults to 50 seconds, with statement rollback behavior |
| Common retry errors | 40001 (serialization failure), 40P01 (deadlock) | 1213 (deadlock), 1205 (lock wait timeout) |

Use database-specific behavior when the interviewer names PostgreSQL or MySQL. The textbook lock model is useful for generic explanations, but it does not accurately predict all PostgreSQL MVCC behavior.

## 12. Interview cheat sheet

### When the database is not specified

Start by stating the assumption:

"Assuming the standard lock-based model, unless you want a specific database."

Then walk through the schedule:

1. What lock does each operation request?
2. Does another transaction hold a conflicting lock?
3. Does the operation wait?
4. Does the wait create a circular dependency (deadlock)?
5. What is the final state, and which anomaly occurred?

### When PostgreSQL or MVCC is specified

Switch to the snapshot model:

- Ordinary SELECTs use MVCC snapshots and do not wait for row-level UPDATE locks.
- Read Committed takes a snapshot per statement.
- Repeatable Read and Serializable use a transaction-level snapshot.
- PostgreSQL can abort a transaction rather than wait through a stale-snapshot conflict.

### Standard answers

- Dirty reads: PostgreSQL does not expose them; Read Uncommitted is treated as Read Committed.
- Non-repeatable reads: possible at Read Committed; prevented at Repeatable Read.
- Phantoms: possible at Read Committed; PostgreSQL prevents them at Repeatable Read.
- Lost update: possible with a read-then-write workflow at Read Committed; use atomic updates, locks, version checks, or Repeatable Read plus retry.
- Write skew: possible at PostgreSQL Repeatable Read; use Serializable plus retry, a guard row, or a suitable constraint.
- Deadlocks: can happen at any isolation level; consistent lock ordering helps prevent them.
- Serializable in one sentence: "The result must be equivalent to some serial order. PostgreSQL uses SSI and may abort conflicting transactions, so the application must retry the whole transaction."
- Wait versus abort: lock conflict usually means wait; deadlock, timeout, or serialization/snapshot conflict can mean abort.
- Why Repeatable Read still allows write skew: both transactions can read the same snapshot and insert or update different rows, so there is no same-row write conflict to trigger the lost-update protection.

## 13. Results table — fill from your own runs

The following are the expected outcomes from the supplied experiments. Replace them with what you actually observe, and note where a statement waited.

| Scenario | Read Committed | Repeatable Read | Serializable |
|---|---|---|---|
| Non-repeatable read | Can happen | Prevented | Prevented |
| Phantom | Can happen | Prevented in PostgreSQL | Prevented |
| Lost update (read-then-write) | Can happen | Stale writer aborts with 40001 | Conflicting transaction aborts with 40001 |
| Read skew | Can happen | Prevented | Prevented |
| Write skew | Can happen | Can happen | Transaction aborts with 40001 |
| Deadlock from opposite lock order | One transaction aborts with 40P01 | One transaction aborts with 40P01 | One transaction aborts with 40P01 |

## 14. One-line takeaways

- **Reads use your snapshot. Writes check the snapshot against reality.**
- Snapshot lifetime is the key difference between Read Committed and Repeatable Read.
- Write skew requires protecting the rule being decided on: use a guard lock, Serializable, or a suitable constraint.
- PostgreSQL Repeatable Read and Serializable can both require a retry on 40001.
- Deadlocks come from lock ordering, not from the isolation level itself.
- When reasoning about PostgreSQL, distinguish ordinary MVCC SELECTs from locking reads and writes.
