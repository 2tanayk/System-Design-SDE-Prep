# Transaction Isolation and Its Levels — PostgreSQL + Spring @Transactional

The focus of this topic is **transaction isolation**: what one transaction is allowed to observe while other transactions are running concurrently, and which anomalies each isolation level prevents.

Locks, MVCC snapshots, and PostgreSQL's Serializable Snapshot Isolation (SSI) are supporting mechanisms. They matter because they explain the actual behavior, but the main subject is the four isolation levels and how to choose them in application code.

The examples use PostgreSQL and a small phone-store database. The final sections map the concepts to Spring's @Transactional annotation.

## 1. What does transaction isolation solve?

Imagine two customers trying to buy the last phone.

The database initially contains:

    product_id = 1
    stock = 1

Both transactions read stock = 1. If both independently decide that the item is available and write stock = 0, both may believe they successfully bought the last phone.

The problem is not that either transaction was individually invalid. The problem is that the concurrent execution allowed them to interfere in a way that broke a business rule.

**Isolation levels define the guarantees transactions receive when they overlap.** Stronger isolation prevents more anomalies, but can introduce more coordination, waiting, or transaction aborts that the application must handle.

## 2. The anomalies isolation levels protect against

| Anomaly | What happens | Store example |
|---|---|---|
| Dirty read | A transaction reads another transaction's uncommitted change | Checkout temporarily writes stock = 0; another transaction reads it; checkout rolls back |
| Non-repeatable read | The same row is read twice and returns different values | A transaction reads price = 100, another commits price = 80, then the first reads 80 |
| Phantom read | Repeating a predicate query returns a different set of rows | Order count for a product changes from 0 to 1 after another transaction inserts an order |
| Serialization anomaly | Concurrent transactions produce a result that could not occur in any serial, one-at-a-time execution | Two transactions each see two orders and each insert one, violating a maximum of three |
| Lost update | Two read-modify-write operations overwrite each other's result | Two buyers read stock = 10 and both write 9, losing one decrement |
| Read skew | Related values are read from different committed states | Auditor reads stock = 10, checkout commits stock = 9 and sold = 1, auditor then reads sold = 1 and calculates 11 |
| Write skew | Transactions read the same condition but write different rows, jointly violating a rule | Two tabs both see two orders under a maximum of three, then both insert another order |

The first three are the classic phenomena used by the SQL standard to describe isolation levels. Lost update, read skew, and write skew are useful practical anomalies to recognize when reasoning about application transactions.

## 3. The four isolation levels

The standard levels, from weakest to strongest, are:

    READ UNCOMMITTED
          |
          v
    READ COMMITTED
          |
          v
    REPEATABLE READ
          |
          v
    SERIALIZABLE

The higher levels provide stronger guarantees, but don't assume every database implements them using the same internal mechanism.

### 3.1 Read Uncommitted

**Guarantee:** the weakest level. In the standard model, a transaction may read changes that another transaction has not committed yet.

That creates the possibility of a dirty read.

Example:

1. Transaction A changes stock from 10 to 0 but has not committed.
2. Transaction B reads stock = 0.
3. Transaction A rolls back, so stock returns to 10.
4. Transaction B made a decision using a value that never committed.

This can lead to decisions based on data that disappears.

**PostgreSQL-specific behavior:** PostgreSQL accepts READ UNCOMMITTED in the SQL command and JDBC isolation API, but treats it as READ COMMITTED. It does not expose true dirty reads.

Practical takeaway: know the standard definition for interviews, but remember that selecting READ_UNCOMMITTED on PostgreSQL does not give you dirty reads.

### 3.2 Read Committed

**Guarantee:** a statement sees only data committed before that statement began. Dirty reads are prevented, but repeated statements inside one transaction can see newer committed data.

PostgreSQL uses Read Committed by default unless the database/session configuration is changed.

#### Example: non-repeatable read

Initial price is 100.

| Transaction A | Transaction B |
|---|---|
| BEGIN ISOLATION LEVEL READ COMMITTED; | |
| SELECT price ... returns 100 | |
| | UPDATE products SET price = 80 WHERE id = 1; COMMIT; |
| SELECT price ... returns 80 | |

The same transaction read the same row twice and got different values.

Why? PostgreSQL takes a new snapshot at the start of each statement under Read Committed.

#### Other anomalies possible at Read Committed

- **Phantom:** repeating a count or predicate query can see rows inserted by another committed transaction.
- **Read skew:** separate statements can observe different committed states of related rows.
- **Lost update:** two transactions can both read an old value and later overwrite each other's read-modify-write result.
- **Write skew:** two transactions can make decisions based on the same condition and modify different rows, jointly breaking an invariant.

#### Concurrent UPDATE nuance

If two transactions update the same row, the second updater normally waits for the first. After the first commits, PostgreSQL rechecks the latest row version and the WHERE condition before applying the second update.

That wait does not automatically prevent a lost update when both transactions previously read the same value and later write a calculated replacement. For simple arithmetic, prefer an atomic statement such as:

    UPDATE products
    SET stock = stock - 1
    WHERE id = 1;

### 3.3 Repeatable Read

**Guarantee:** the transaction sees a stable snapshot. Re-reading a row does not reveal another transaction's later committed changes.

In PostgreSQL, the snapshot is established at the first query or data-modification statement, not merely at BEGIN. All statements in the transaction use that transaction-level snapshot.

#### Example: non-repeatable read is prevented

Initial price is 100.

| Transaction A | Transaction B |
|---|---|
| BEGIN ISOLATION LEVEL REPEATABLE READ; | |
| SELECT price ... returns 100 (snapshot established) | |
| | UPDATE products SET price = 80 WHERE id = 1; COMMIT; |
| SELECT price ... still returns 100 | |

After A commits and starts a new transaction, it can see 80.

#### PostgreSQL-specific phantom behavior

The SQL standard's minimum definition permits phantoms at Repeatable Read. PostgreSQL provides a stronger snapshot-based implementation: predicate queries also see the same transaction snapshot, so the classic phantom experiment is prevented at PostgreSQL Repeatable Read.

#### Concurrent update can abort

If another transaction updates and commits a row after your Repeatable Read snapshot was established, and your transaction later tries to update that same row, PostgreSQL waits for the concurrent updater if necessary and then aborts the stale transaction with SQLSTATE 40001:

    ERROR: could not serialize access due to concurrent update

The transaction must be rolled back and retried from the beginning using a fresh snapshot.

This prevents the stale writer from silently applying its decision to a row version that did not exist in its snapshot.

#### Important limitation: write skew is still possible

Suppose the rule is "a customer may have at most three orders." Two transactions both count two existing orders using their Repeatable Read snapshots. Each inserts a different new row. Because they write different rows, neither necessarily encounters a same-row concurrent-update conflict. Both may commit, leaving four orders.

So PostgreSQL Repeatable Read prevents non-repeatable reads and phantoms, but it does **not** guarantee full serializability. Write skew remains possible.

### 3.4 Serializable

**Guarantee:** the result of successfully committed concurrent Serializable transactions must be equivalent to some execution in which those transactions ran one at a time in a serial order.

This is the strongest standard isolation level. It is intended to prevent serialization anomalies, including write skew.

#### Example: maximum three orders per customer

Initial state: Riya has two orders.

At Repeatable Read:
1. Transaction A counts two.
2. Transaction B counts two.
3. A inserts one order.
4. B inserts another order.
5. Both may commit, leaving four.

At Serializable:
1. A and B both count two.
2. Both attempt their inserts.
3. PostgreSQL detects that the overlapping read/write dependencies could produce a result that is not equivalent to a serial order.
4. One transaction is aborted with SQLSTATE 40001.
5. The application retries the entire aborted transaction. It now sees three orders and refuses the fourth.

#### How PostgreSQL implements it

PostgreSQL uses Serializable Snapshot Isolation (SSI). It tracks read/write dependencies using predicate-lock metadata (visible as SIReadLock in pg_locks). These are not ordinary blocking locks: they do not make writers wait. PostgreSQL uses the dependency information to detect dangerous concurrent patterns and abort a transaction when needed.

This is supporting implementation detail; the application-level guarantee is serializable execution or a serialization failure.

#### Application responsibility

A serialization failure can occur during a statement or at COMMIT. When SQLSTATE 40001 occurs:

- Roll back the failed transaction.
- Retry the **whole transaction**, not just the last SQL statement.
- Start with a fresh transaction and repeat the business reads and decisions.
- Cap retries (for example, 3–5) and add a small randomized delay.
- Keep the transaction short.
- Do not perform irreversible external side effects inside a transaction that may be retried. Send emails or call external APIs after a successful commit, or use an outbox/idempotency design.

Serializable can have overhead and may abort transactions. Use it where a business invariant genuinely needs it rather than enabling it indiscriminately.

## 4. Compare the levels

### Standard SQL model vs. PostgreSQL

| Isolation level | Dirty read | Non-repeatable read | Phantom read | Serialization anomaly |
|---|---|---|---|---|
| Read Uncommitted — SQL model | Possible | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible | Possible |
| Repeatable Read — SQL minimum | Prevented | Prevented | Possible | Possible |
| Repeatable Read — PostgreSQL | Prevented | Prevented | Prevented | Possible (write skew) |
| Serializable | Prevented | Prevented | Prevented | Prevented |

PostgreSQL-specific reminders:
- READ UNCOMMITTED behaves as READ COMMITTED.
- PostgreSQL Repeatable Read prevents phantoms, which is stronger than the SQL standard's minimum for that level.
- PostgreSQL Repeatable Read can still allow write skew.
- Serializable prevents serialization anomalies by detecting unsafe overlaps and aborting a transaction when required.

### PostgreSQL behavior at a glance

| Behavior | Read Committed | Repeatable Read | Serializable |
|---|---|---|---|
| Snapshot | Per statement | Per transaction | Per transaction |
| Non-repeatable reads | Possible | Prevented | Prevented |
| Phantoms | Possible | Prevented in PostgreSQL | Prevented |
| Read skew | Possible | Prevented | Prevented |
| Lost update in a read-then-write workflow | Possible | Stale writer aborts | Conflicting transaction may abort |
| Write skew | Possible | Possible | Prevented by aborting a transaction |
| Typical application handling | Atomic updates/locks where required | Retry 40001 on concurrent-update conflict | Retry whole transaction on 40001 |

## 5. Direct mapping to Spring @Transactional

This is the application-facing connection for a Spring Boot developer.

Spring exposes transaction isolation through the Isolation enum:

- Isolation.DEFAULT
- Isolation.READ_UNCOMMITTED
- Isolation.READ_COMMITTED
- Isolation.REPEATABLE_READ
- Isolation.SERIALIZABLE

Example:

    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Isolation;
    import org.springframework.transaction.annotation.Transactional;

    @Service
    public class OrderService {

        @Transactional(isolation = Isolation.READ_COMMITTED)
        public void placeOrder() {
            // Read and update order-related data.
        }

        @Transactional(isolation = Isolation.REPEATABLE_READ)
        public void generateConsistentReport() {
            // Multiple reads share a stable transaction snapshot.
        }

        @Transactional(isolation = Isolation.SERIALIZABLE)
        public void enforceComplexBusinessRule() {
            // PostgreSQL may abort with SQLSTATE 40001.
            // The application must retry the whole transaction.
        }
    }

### What does DEFAULT mean?

Spring's @Transactional isolation defaults to Isolation.DEFAULT. This means Spring uses the underlying datastore's default isolation level rather than forcing a specific level.

For PostgreSQL, the default is Read Committed unless configured otherwise.

### Important propagation interaction

Spring's isolation setting applies when Spring starts a new transaction, such as with REQUIRED when no transaction already exists, or with REQUIRES_NEW.

If a method using REQUIRED joins an existing transaction, it normally inherits the existing transaction's isolation. Its own isolation declaration does not silently upgrade the already-running transaction. Spring can be configured to reject incompatible existing transaction characteristics with validateExistingTransaction.

Therefore, do not assume that putting @Transactional(isolation = Isolation.SERIALIZABLE) on an inner method automatically makes an already-running outer transaction Serializable.

### Practical guidance

- Start with the database default (commonly PostgreSQL Read Committed).
- Use Repeatable Read when a transaction needs a stable view across multiple statements.
- Use Serializable when correctness depends on a multi-row invariant that weaker isolation and simpler locking/constraints do not adequately protect.
- Handle 40001 at the service boundary by retrying the complete transaction.
- Remember that @Transactional is applied through Spring's transaction infrastructure; the call must pass through the Spring proxy for the annotation to take effect.

Isolation is only one part of @Transactional. Propagation, rollback rules, timeout, readOnly, and transaction boundaries are separate concerns and should not be confused with isolation.

## 6. Small PostgreSQL mechanisms that explain the behavior

These are supporting concepts, not the primary topic.

### MVCC and snapshots

PostgreSQL uses Multi-Version Concurrency Control (MVCC). UPDATE creates a new row version rather than simply overwriting the old version in place. A snapshot determines which versions a transaction can see.

At a high level:
- Read Committed takes a new snapshot for each statement.
- Repeatable Read and Serializable use a transaction-level snapshot established by the first query or data-modification statement.
- Long-running transactions can keep old row versions relevant, delaying their cleanup and contributing to table bloat.

Useful inspection commands:

    SHOW transaction_isolation;

    SELECT pg_current_snapshot();

    SELECT ctid, xmin, xmax, id, price
    FROM products
    ORDER BY id;

### Locks and waiting

Writes and locking reads can wait for conflicting row locks. Ordinary MVCC SELECTs do not normally wait for row-level UPDATE locks. Write locks are generally held until COMMIT or ROLLBACK.

A deadlock is a circular wait (A holds one row and wants another; B holds the second and wants the first). PostgreSQL aborts one transaction with SQLSTATE 40P01. A consistent lock-acquisition order helps prevent this. Deadlocks are possible at any isolation level; they are not themselves an isolation level.

### Wait versus abort

- Lock conflict: usually wait.
- Deadlock: abort one transaction (40P01).
- PostgreSQL Repeatable Read stale concurrent update: abort (40001).
- Serializable dangerous dependency pattern: abort a transaction (40001).

## 7. Hands-on PostgreSQL lab

Use two psql sessions connected to the same database. Predict each result before executing it.

### Setup once

    CREATE DATABASE iso_lab;

Connect both terminals:

    psql -d iso_lab

Create tables:

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

Reset between experiments (with no open transactions):

    TRUNCATE products, orders RESTART IDENTITY;

    INSERT INTO products (id, name, price, stock)
    VALUES
        (1, 'Phone', 100, 10),
        (2, 'Case', 10, 100);

Always set the isolation level explicitly:

    BEGIN ISOLATION LEVEL READ COMMITTED;
    SHOW transaction_isolation;

Statements outside an explicit transaction block autocommit. After an error, run ROLLBACK before continuing.

### Experiment A — Read Committed: non-repeatable read

Terminal A:

    BEGIN ISOLATION LEVEL READ COMMITTED;
    SELECT price FROM products WHERE id = 1; -- 100

Terminal B:

    UPDATE products SET price = 80 WHERE id = 1;

Terminal A:

    SELECT price FROM products WHERE id = 1; -- 80
    COMMIT;

Repeat with A at REPEATABLE READ. It should see 100 both times in that transaction. After A commits and starts a new transaction, it sees 80.

### Experiment B — Read Committed vs. PostgreSQL Repeatable Read: phantom

Reset orders first.

Terminal A:

    BEGIN ISOLATION LEVEL READ COMMITTED;
    SELECT count(*) FROM orders WHERE product_id = 1; -- 0

Terminal B:

    INSERT INTO orders (customer, product_id, qty)
    VALUES ('Sam', 1, 1);

Terminal A:

    SELECT count(*) FROM orders WHERE product_id = 1; -- 1
    COMMIT;

Repeat with A at REPEATABLE READ. Its count stays 0 because it retains its transaction snapshot.

### Experiment C — Lost update and the fixes

Reset products first. Initial Phone stock is 10.

At Read Committed:
1. A reads stock = 10.
2. B reads stock = 10.
3. A writes stock = 9 and commits.
4. B writes stock = 9 and commits.
5. Final stock = 9, although two buyers attempted to decrement it.

Compare these fixes.

**Atomic update:**

    UPDATE products
    SET stock = stock - 1
    WHERE id = 1;

Run once per purchase; PostgreSQL applies each decrement to the current row value.

**Pessimistic row lock:**

    BEGIN;

    SELECT stock
    FROM products
    WHERE id = 1
    FOR UPDATE;

    -- Validate stock, then update.

    UPDATE products
    SET stock = stock - 1
    WHERE id = 1;

    COMMIT;

A competing locking read waits, then sees the current row after the first transaction finishes.

**Repeatable Read:**
Both sessions read stock = 10 using their transaction snapshots. One updates and commits. The other attempts to update the stale row and receives 40001; it must roll back and retry from the beginning.

**Optimistic version column:**

    UPDATE products
    SET stock = 9,
        version = version + 1
    WHERE id = 1
      AND version = 0;

If the version has already changed, the update affects zero rows. The application detects the conflict, rereads and retries.

### Experiment D — Write skew: maximum three orders

Initial state:

    INSERT INTO orders (customer, product_id, qty)
    VALUES ('Riya', 1, 1), ('Riya', 1, 1);

At Repeatable Read, run two transactions:

    BEGIN ISOLATION LEVEL REPEATABLE READ;
    SELECT count(*) FROM orders WHERE customer = 'Riya'; -- 2

Both sessions perform the count before either inserts. Then both insert one Riya order and commit. Both may succeed, leaving four rows.

Repeat with SERIALIZABLE. PostgreSQL should abort one transaction with 40001 because the concurrent execution cannot be serialized. Roll back and retry the whole failed transaction; it sees three and should refuse the fourth order.

This experiment demonstrates why Repeatable Read alone does not prevent write skew.

## 8. Choosing an isolation level in practice

| Requirement | Starting point |
|---|---|
| Typical CRUD transaction; each statement can use the latest committed data | Read Committed |
| Multiple reads must see one stable transaction snapshot | Repeatable Read |
| A business invariant spans multiple rows and must behave as if transactions ran one at a time | Serializable, with retry handling |
| Simple decrement/increment of one row | Atomic UPDATE may be simpler than raising isolation |
| Read-decide-write on a known row under contention | SELECT ... FOR UPDATE may be suitable |
| Rule can be expressed as a database constraint | Prefer the constraint where practical |

Do not automatically choose Serializable for every method. Pick the weakest level that correctly protects the business invariant, and use atomic statements, explicit locks, optimistic versioning, or constraints when they are a better fit.

## 9. Interview summary

- **Read Uncommitted:** standard model allows dirty reads; PostgreSQL treats it as Read Committed.
- **Read Committed:** each statement gets a fresh snapshot; non-repeatable reads, phantoms, read skew, lost updates, and write skew can occur.
- **Repeatable Read:** PostgreSQL keeps a transaction-level snapshot; prevents non-repeatable reads, phantoms, and read skew, but write skew can still occur.
- **Serializable:** successful transactions behave as though they ran in some serial order; PostgreSQL may abort a transaction with 40001, so the application retries the entire transaction.
- **Spring mapping:** @Transactional(isolation = Isolation.X) selects the isolation when Spring starts a new transaction; DEFAULT delegates to the database's configured default.
- Locks, MVCC and SSI explain how PostgreSQL implements these guarantees; they are supporting mechanisms, not separate isolation levels.

## References

- [PostgreSQL 18 — Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL 18 — SET TRANSACTION](https://www.postgresql.org/docs/current/sql-set-transaction.html)
- [Spring Framework — Using @Transactional](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
- [Spring Framework — Isolation enum](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Isolation.html)
