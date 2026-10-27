# Database Migration and Schema Version Control

## 1. Two related meanings of database migration

“Database migration” can refer to two related but different activities:

1. **Data/platform migration:** Move existing data or a database workload from a source system to a target system.
2. **Schema migration:** Change database structure over time as application requirements evolve, while keeping changes version-controlled and deployable.

A food-delivery system may use both: moving its PostgreSQL database to a new managed service, and separately evolving its schema by adding columns, constraints, tables, or indexes.

## 2. Data migration — move data from source to target

A data migration moves data from a source system to a target system. Example: moving a production PostgreSQL database to a new PostgreSQL service.

The core challenge is that the application may continue accepting writes while existing data is being copied. Copying a snapshot alone can miss new rows or updates that occur during the copy.

### Online migration: initial load + ongoing changes

A common approach is:

1. Prepare the target schema, permissions, capacity, and configuration.
2. Copy existing data in an initial bulk load or **backfill**.
3. Capture subsequent changes from the source using a suitable change-capture mechanism.
4. Apply those changes to the target until it catches up.
5. Validate target data and behavior.
6. Establish a cutover boundary and switch the application to the target.
7. Monitor and retain a rollback plan that accounts for writes after cutover.

**Change Data Capture (CDC)** captures inserts, updates, and deletes so they can be propagated elsewhere. CDC may be based on database logs or other mechanisms. PostgreSQL logical replication and other database migration tools offer different ways to replicate ongoing changes.

~~~text
Source database
    |\
    | \ initial data copy
    |  \------------------------> Target database
    |
    +---- ongoing change capture -> Target catches up
                                      |
                                      v
                                  Validate
                                      |
                                      v
                               Controlled cutover
                                      |
                                      v
                              Application uses target
~~~

### Why copying once is unsafe during live traffic

If the initial copy captures orders 1 through 100,000, but order 100,001 is created while the copy is running, the target may not contain that new order. Ongoing change capture or a carefully controlled write pause is needed to account for such changes before cutover.

### Validation

Validation should match the risk and data model. It can include:
- Row counts and aggregate checks.
- Checksums or sampled record comparisons.
- Referential integrity.
- Important queries and application workflows.
- Indexes, constraints, permissions, sequences/identifiers, and configuration.
- Data consistency at the chosen cutover point.

### Cutover and rollback

**Cutover** is the controlled switch from source to target. The procedure must define which system owns writes and how in-flight transactions and final changes are handled.

Do not assume rollback means merely pointing the application back at the old database. If new writes occurred on the target after cutover, switching back without reconciling them can lose or fork data. A rollback plan must account for post-cutover writes and for whether the old system has remained synchronized.

## 3. Offline vs. online migration

### Offline migration

The workload is paused or writes are stopped while data is copied, validated, and switched over.

**Choose it when** the dataset and maintenance window make downtime acceptable.

Advantages: less concurrent-change complexity and a simpler consistency boundary.

Tradeoff: the application may be unavailable or unable to accept writes during the migration.

### Online migration

The application remains available during most of the copy, with ongoing changes captured and applied to the target.

**Choose it when** downtime must be minimized.

Tradeoffs:
- More moving pieces to operate.
- CDC/replication lag must be monitored.
- Target validation and final cutover need careful coordination.
- The process is not necessarily zero-downtime; a brief coordinated switch may still be required.

## 4. Why dual writing needs care

**Dual writing** means the application independently writes the same business change to both source and target databases.

For example, an order may be written successfully to the source and then fail on the target. The stores now disagree. The reverse order, retries, and timeouts create similar ambiguity.

Dual writing can be used in specific migration designs, but it requires explicit handling of partial failure, idempotency, ordering, and reconciliation. Initial copying plus change capture and a controlled cutover may be safer than adding ad hoc writes to both databases in application code.

## 5. Schema migration and schema version control

A **schema** describes database structure: tables, columns, types, indexes, constraints, and related objects.

**Schema migration** means applying incremental, controlled changes to that structure as application requirements evolve. **Schema version control** means keeping those changes as versioned artifacts alongside application code.

Two management styles:

- **State-based:** Describe the desired final schema; a tool calculates the difference between the current and desired state.
- **Change-based / migration-based:** Define ordered scripts that move the schema from version to version.

Change-based migrations make the change history explicit and are commonly used with tools such as Flyway or Liquibase.

## 6. Anatomy and execution of a migration

A migration file is a versioned step in an ordered sequence.

- **Up migration:** Applies the forward change, such as creating a table, adding a column, or creating an index.
- **Down migration / rollback:** Defines how to reverse the corresponding change when that is safe and supported.
- **Migration tracking table:** A tool-maintained table (often named something like schema migrations) records applied migration identifiers and metadata such as timestamps or checksums.

A typical lifecycle:

1. The migration engine connects to the database and examines its tracking table.
2. It compares available migration scripts with recorded applied migrations.
3. It executes unapplied migrations in the tool's defined order, usually by version or timestamp.
4. It records successful execution and migration metadata.

A migration tool should prevent a successfully applied version from being run again as if it were new. Keep released migration files immutable; add a new migration for a subsequent change instead of silently editing a script that has already run in shared environments.

### Idempotency and determinism

- **Idempotent operation:** Repeating it has the same intended effect as performing it once.
- **Deterministic migration:** Given the expected starting state and inputs, it produces a predictable result.

Design migrations to be safe and predictable, and make operations idempotent when the database and migration design support that approach. However, do not assume every migration can safely be rerun; migration tracking and recovery procedures remain important. A half-completed migration needs the tool-specific recovery procedure, not blindly rerunning statements.

### Transactional migrations

Where supported, changes can be enclosed in a transaction so that a failure rolls back the migration rather than leaving partially applied schema changes.

However, not all DDL can run inside a transaction. For example, PostgreSQL's `CREATE INDEX CONCURRENTLY` cannot run inside a transaction block. Some database engines also have DDL operations that cause implicit commits. Migration design must respect the actual database's rules.

## 7. Expand-contract: evolve schemas without breaking live application instances

A dangerous deployment is to remove or rename a column while older application instances still depend on it. During a rolling deployment, old and new application versions may run simultaneously.

The **expand-contract** (also called parallel-change) pattern makes the schema change in compatible stages.

Example: change the field used for a customer's delivery instructions.

### Phase 1 — Expand

Add the new column while leaving the old one in place. Avoid immediately removing structures that currently deployed code still needs.

### Phase 2 — Transition

Deploy application code that is compatible with both schemas. Depending on the change, it may read the old field, write both fields, or read/write the new field with a fallback strategy.

### Phase 3 — Backfill and verify

Copy existing data from the old field into the new field. Verify coverage and handle changes that occur during the backfill so the new field does not become stale.

### Phase 4 — Switch fully

Deploy code that reads and writes the new field as the authoritative representation. Confirm that no active application version still depends on the old field.

### Phase 5 — Contract

Only after old code is no longer in use and the new structure has been verified, remove the old column in a separate migration.

~~~text
Expand -> Transition code -> Backfill/verify -> Switch fully -> Contract
Add new  Support old/new   Copy existing data  Stop relying on  Remove old
structure versions         and verify           old structure    structure
~~~

This pattern is useful for low-downtime changes because schema and application releases need not happen atomically. The key is backward and forward compatibility during the transition.

## 8. Deployment strategies for migrations

### Migrations on application startup

A service instance can run migrations during startup. This can be convenient in small deployments, but becomes risky when horizontal scaling starts several instances together. Multiple instances may attempt to apply the same migration at the same time, and an instance may become ready before required migration work is safely completed.

Migration tools may use locking to coordinate runners, but deployment design still needs to be explicit.

### Dedicated migration pipeline

A CI/CD pipeline runs a dedicated, controlled migration step, typically once per release, before deploying code that depends on the new schema.

Advantages:
- Separates schema changes from normal application startup.
- Reduces races caused by many replicas trying to migrate at once.
- Makes migration results and failure handling more visible in deployment logs.

Tradeoff: release orchestration must ensure that the schema change is compatible with application versions currently running.

### Decouple schema and code releases

Deploy schema changes that are backward-compatible with the currently running application before deploying code that depends on them. Remove old schema elements only after all active application instances have moved off them.

## 9. Practical example: Add a delivery-instructions field

Suppose the application needs a new optional field on a user's address.

1. **Expand:** Add a nullable `delivery_instructions` column without breaking old code.
2. Deploy code that can handle the column's presence and absence.
3. Backfill existing data if there is a meaningful old source field.
4. Switch API reads and writes to the new column when all required versions support it.
5. Validate production data and deployment state.
6. **Contract only if needed:** Remove obsolete columns in a later migration after no deployed code relies on them.

For a new column with no old equivalent, a backfill may not be needed. Not every schema change needs every phase, but compatibility and rollout order must be considered explicitly.

## 10. Checklist

### Data/platform migration
1. Are source and target schemas, versions, extensions, permissions, and capacity compatible?
2. How is the initial data copied?
3. How are writes during the copy captured?
4. How is the target validated?
5. What defines the cutover point and write ownership?
6. How will post-cutover writes be handled if rollback is needed?

### Schema migration
1. Is the migration versioned and committed with code?
2. Is it safe and predictable for the expected starting schema?
3. Can it be transactional, or does a DDL command require special treatment?
4. Is it compatible with the application versions currently running?
5. Is a backfill required, and how will it handle concurrent updates?
6. Is migration execution coordinated rather than raced by many application instances?
7. Are old schema elements removed only after no deployed code needs them?

## 11. Summary card

- **Data migration:** Copy existing data, capture ongoing changes if the source remains live, validate the target, and cut over with a plan for rollback and post-cutover writes.
- **Schema migration:** Treat schema changes as versioned, ordered artifacts; migration tools record which versions have been applied.
- **Safety:** Released migrations should be immutable; use transactions where supported, and understand operations that cannot run transactionally.
- **Zero/low downtime:** Use expand-contract to keep old and new application versions compatible while schema and data transition in stages.
- **Deployment:** A controlled migration pipeline often avoids startup races, especially with multiple application instances.
- **Interview one-liner:** “For data migration, I'd combine a consistent initial copy with ongoing change capture, validation, and controlled cutover. For schema evolution, I'd version migrations, maintain compatibility with running code, and use expand-contract for changes that must be deployed with minimal downtime.”
