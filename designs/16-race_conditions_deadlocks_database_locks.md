# Race Conditions, Deadlocks, and Database Locks

## 1. Race Condition

A **race condition** happens when multiple operations access and modify the same shared data concurrently, and the final result depends on the timing or order in which those operations execute.

### Example: Two simultaneous withdrawals

Suppose a bank account has a balance of `$100`.

Two requests arrive at nearly the same time, and both try to withdraw `$80`.

```js
let balance = 100;

async function withdraw(amount) {
  if (balance >= amount) {
    // Simulate some delay
    await delay(100);

    balance -= amount;
  }
}

withdraw(80);
withdraw(80);
```

A possible execution is:

```text
Initial balance: 100

Request A: checks balance → 100 ✓
Request B: checks balance → 100 ✓

Request A: subtracts 80 → 20
Request B: subtracts 80 → -60
```

The account should not have allowed both withdrawals, but both requests saw the same initial balance.

The important issue is that this sequence:

```text
if (balance >= amount)
balance -= amount
```

is logically one operation, but without synchronization another operation can execute between those steps.

### How to prevent it

Common solutions include:

- Database transactions
- Row-level locks
- Atomic SQL updates
- Optimistic concurrency control
- Application-level mutexes/locks

For example, an atomic database operation might look conceptually like:

```sql
UPDATE accounts
SET balance = balance - 80
WHERE id = 123
  AND balance >= 80;
```

Then the application checks whether one row was actually updated.

---

# 2. Deadlock

A **deadlock** happens when two or more transactions/processes are waiting for resources held by each other, so none of them can continue.

Consider two locks:

```text
Lock A
Lock B
```

Transaction 1:

```text
Acquire A
Acquire B
Do work
Release B
Release A
```

Transaction 2:

```text
Acquire B
Acquire A
Do work
Release A
Release B
```

Now consider this execution:

```text
Transaction 1             Transaction 2
---------------           ---------------
Acquire A ✓
                          Acquire B ✓

Acquire B → WAIT
                          Acquire A → WAIT
```

Now:

```text
Transaction 1 owns A
Transaction 1 waits for B

Transaction 2 owns B
Transaction 2 waits for A
```

Neither transaction can proceed.

This is a **deadlock**.

### Visual representation

```text
Transaction 1
      |
      | owns
      v
    Lock A
      |
      | needed by
      v
Transaction 2
      |
      | owns
      v
    Lock B
      |
      | needed by
      v
Transaction 1
```

It forms a cycle:

```text
T1 → waits for T2 → waits for T1
```

---

# 3. Race Condition vs Deadlock

| | Race Condition | Deadlock |
|---|---|---|
| Main problem | Incorrect/unpredictable result | No progress |
| Typical cause | Concurrent access to shared state | Circular waiting for resources |
| Example | Two withdrawals | Two transactions locking rows in opposite order |
| Result | Wrong data/state | Transactions become stuck |
| Typical solution | Transactions, locks, atomic operations | Consistent lock ordering, deadlock detection, retries |

A simple way to remember:

> **Race condition:** "Who gets there first changes the result."

> **Deadlock:** "Everyone is waiting, so nobody gets there."

---

# 4. Why Databases Need Locks

Databases have many clients executing operations concurrently.

Imagine:

```text
User A → UPDATE accounts ...
User B → UPDATE accounts ...
User C → SELECT ...
User D → UPDATE ...
```

If the database simply allowed everyone to modify data simultaneously without coordination, data could become inconsistent.

Databases therefore use concurrency-control mechanisms.

One important mechanism is **locking**.

A lock essentially says:

> "This transaction currently has a certain kind of access to this piece of data."

Depending on the database engine and isolation level, locks can exist at different granularities and have different purposes.

---

# 5. Shared Locks

A **shared lock** (often called an `S` lock) is generally used for reading.

Multiple transactions can usually hold a shared lock on the same resource at the same time.

For example:

```text
Transaction A:
  SELECT account 123

Transaction B:
  SELECT account 123
```

Conceptually:

```text
          Account 123
          /                S lock      S lock
         |            |
        T1           T2
```

Both transactions can read the data.

However, an exclusive modification generally cannot happen while incompatible shared locks are held.

For example:

```text
T1 → S lock
T2 → S lock

T3 → wants X lock
      ↓
     WAIT
```

The exact behavior depends on the database engine, query, isolation level, and whether the database uses locking or MVCC for that operation.

---

# 6. Exclusive Locks

An **exclusive lock** (often called an `X` lock) is generally associated with modifying data.

For example:

```sql
UPDATE accounts
SET balance = balance - 50
WHERE id = 123;
```

Conceptually:

```text
Transaction 1
     |
     | X lock
     v
Account 123
```

While T1 holds the incompatible exclusive lock, another transaction attempting to modify the same locked resource generally has to wait.

For example:

```text
T1: UPDATE account 123
    ↓
    X lock acquired

T2: UPDATE account 123
    ↓
    wants X lock
    ↓
    WAIT
```

After T1 commits or rolls back:

```text
T1 → COMMIT
      ↓
    lock released
      ↓
T2 → can acquire lock
```

---

# 7. Lock Compatibility

A simplified lock compatibility matrix looks like this:

| Existing lock | New Shared (S) | New Exclusive (X) |
|---|---:|---:|
| Shared (S) | Usually compatible | Not compatible |
| Exclusive (X) | Not compatible | Not compatible |

So:

```text
S + S → ✓
S + X → ✗
X + S → ✗
X + X → ✗
```

This is a simplified model. Real database engines have additional lock modes and rules.

---

# 8. Row-Level Locks

Databases can lock individual rows.

Suppose we have:

```text
accounts

id    name    balance
---------------------
1     Alice   100
2     Bob     200
3     Carol   300
```

Transaction A updates Alice:

```sql
UPDATE accounts
SET balance = balance - 50
WHERE id = 1;
```

Conceptually:

```text
Row 1 → locked by T1

Row 2 → available
Row 3 → available
```

Another transaction can potentially update Bob simultaneously:

```sql
UPDATE accounts
SET balance = balance - 50
WHERE id = 2;
```

Conceptually:

```text
Row 1 → T1
Row 2 → T2
Row 3 → available
```

This is one reason fine-grained row locking can provide good concurrency.

---

# 9. Table-Level Locks

A database can also use locks at a larger granularity, such as an entire table.

Conceptually:

```text
accounts table
┌─────────────────────┐
│ Alice               │
│ Bob                 │
│ Carol               │
│ David               │
└─────────────────────┘
          ↑
       table lock
```

A table-level lock can affect many rows at once.

This can reduce concurrency compared with row-level locking, but table locks can sometimes be useful or necessary depending on the operation and database engine.

---

# 10. Page-Level Locks

Some database engines can also lock a **page**, which is a unit of storage containing multiple rows.

Conceptually:

```text
Database
│
├── Page 1
│     ├── Row 1
│     ├── Row 2
│     └── Row 3
│
├── Page 2
│     ├── Row 4
│     ├── Row 5
│     └── Row 6
│
└── Page 3
```

A page lock may affect several rows even though the transaction only logically cares about one or a few of them.

The exact lock granularity available and commonly used varies by database system.

---

# 11. Intent Locks

Real database locking systems are more sophisticated than simply having `S` and `X`.

Many systems use **intent locks** to coordinate locks at different levels of the hierarchy.

For example:

```text
Database
   ↓
Table
   ↓
Page
   ↓
Row
```

Suppose a transaction wants to acquire an exclusive lock on a row.

The database may use an **intent exclusive (IX)** lock at a higher level to indicate:

> "I intend to acquire an exclusive lock somewhere below this level."

Conceptually:

```text
Table
  |
  | IX
  v
Page
  |
  | IX
  v
Row
  |
  | X
  v
Row 123
```

Intent locks help the database efficiently determine whether higher-level locks conflict with lower-level locks.

Common intent modes include:

- IS — Intent Shared
- IX — Intent Exclusive
- SIX — Shared with Intent Exclusive

The exact implementation depends on the database engine.

---

# 12. Locks During a Transaction

Consider:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 50
WHERE id = 123;

COMMIT;
```

Conceptually:

```text
BEGIN
  ↓
Acquire necessary locks
  ↓
Perform UPDATE
  ↓
Other transactions may have to wait
  ↓
COMMIT
  ↓
Locks are released according to the database's locking rules
```

If the transaction rolls back:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 50
WHERE id = 123;

ROLLBACK;
```

the database reverses the transaction's changes and releases the relevant locks according to its implementation.

---

# 13. Example: Two Transactions Updating the Same Row

Suppose:

```text
accounts
id = 1
balance = 100
```

Transaction A:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 80
WHERE id = 1;
```

Transaction B simultaneously executes:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 80
WHERE id = 1;
```

A simplified locking sequence could be:

```text
T1:
  Acquire X lock on row 1
  Update balance: 100 → 20

T2:
  Wants X lock on row 1
  ↓
  WAIT
```

Then:

```text
T1:
  COMMIT
  ↓
  Lock released

T2:
  Acquires lock
  Performs its operation
```

The exact result and visibility depend on the database's concurrency model and isolation level.

---

# 14. MVCC: An Important Alternative to Traditional Read Locking

Modern relational databases often use **MVCC (Multi-Version Concurrency Control)**.

Instead of making every reader wait for writers, the database can maintain multiple versions of rows.

Conceptually:

```text
Account 123

Version 1:
balance = 100

Version 2:
balance = 50
```

A transaction can read an appropriate version based on its transaction snapshot.

This means a reader may be able to read while another transaction is modifying the current version.

For example:

```text
T1 → UPDATE account
       ↓
    creates/maintains new row version

T2 → SELECT account
       ↓
    reads an appropriate visible version
```

This can significantly improve read/write concurrency.

PostgreSQL, for example, relies heavily on MVCC.

MySQL's InnoDB engine also uses MVCC, together with locking.

---

# 15. MVCC Does Not Mean "No Locks"

This is an important distinction.

It is tempting to think:

> "If the database uses MVCC, there are no locks."

That's not correct.

MVCC can reduce the need for read locks, but databases still use locks for many operations.

For example:

```sql
SELECT *
FROM accounts
WHERE id = 123
FOR UPDATE;
```

The intention is essentially:

> "Read this row, and lock it because I intend to modify it."

Conceptually:

```text
T1:
SELECT ... FOR UPDATE
        ↓
    X-like row lock
        ↓
    other conflicting operations wait
```

This is extremely useful when implementing workflows such as:

```text
Read balance
↓
Validate
↓
Modify balance
↓
Commit
```

---

# 16. Pessimistic vs Optimistic Concurrency

There are two broad approaches to concurrency control.

## Pessimistic concurrency

Assume conflicts are likely.

Lock the resource before modifying it.

Example:

```sql
BEGIN;

SELECT balance
FROM accounts
WHERE id = 123
FOR UPDATE;

-- Validate balance

UPDATE accounts
SET balance = balance - 80
WHERE id = 123;

COMMIT;
```

Conceptually:

```text
Lock first
   ↓
Read
   ↓
Validate
   ↓
Modify
   ↓
Commit
```

This prevents another transaction from simultaneously modifying the protected row in a conflicting way.

## Optimistic concurrency

Assume conflicts are relatively uncommon.

Instead of locking the row for the entire operation, use a version number.

Example:

```text
id    balance    version
-----------------------
123   100        7
```

Application reads:

```text
balance = 100
version = 7
```

Then attempts:

```sql
UPDATE accounts
SET balance = 20,
    version = 8
WHERE id = 123
  AND version = 7;
```

If another transaction already changed the row:

```text
version = 8
```

then the `WHERE version = 7` condition fails.

The application knows that a concurrent modification occurred.

This is commonly called **optimistic locking**.

---

# 17. How a Database Deadlock Happens

Consider two rows:

```text
Account A
Account B
```

Transaction 1:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 10
WHERE id = 'A';

UPDATE accounts
SET balance = balance + 10
WHERE id = 'B';
```

Transaction 2:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 20
WHERE id = 'B';

UPDATE accounts
SET balance = balance + 20
WHERE id = 'A';
```

Possible execution:

```text
T1:
locks A

T2:
locks B

T1:
tries to lock B
→ WAIT

T2:
tries to lock A
→ WAIT
```

Now:

```text
T1 → waiting for B
B  → held by T2

T2 → waiting for A
A  → held by T1
```

That's a deadlock.

---

# 18. How Databases Deal With Deadlocks

A database generally cannot simply allow a deadlock to continue forever.

Many database systems have **deadlock detection**.

The database can detect a cycle such as:

```text
T1 → waiting for T2
T2 → waiting for T1
```

It then chooses a transaction as a **victim**, rolls it back, and allows the other transaction to continue.

Conceptually:

```text
T1 ───── waits for ─────> T2
↑                         │
└──────── waits for ──────┘

          ↓

Database detects cycle

          ↓

Rollback T1

          ↓

T2 continues
```

The application should generally be prepared to retry a transaction that was aborted because of a deadlock.

---

# 19. Preventing Deadlocks

One of the most common strategies is **consistent lock ordering**.

Bad:

```text
T1:
lock A
lock B

T2:
lock B
lock A
```

Better:

```text
T1:
lock A
lock B

T2:
lock A
lock B
```

Both transactions acquire resources in the same order.

Then this circular dependency is much less likely:

```text
T1 owns A → waits for B
T2 owns B → waits for A
```

Other techniques include:

- Keep transactions short.
- Lock only what is necessary.
- Access tables/rows in a consistent order.
- Avoid unnecessary user interaction while a transaction is open.
- Use appropriate indexes so queries don't lock or scan more data than necessary.
- Handle deadlock errors and retry safely when appropriate.

---

# 20. Isolation Levels

Locks and MVCC are also connected to **transaction isolation**.

SQL databases commonly define isolation levels such as:

```text
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

The general idea is:

> How isolated should one transaction be from concurrent transactions?

Higher isolation generally provides stronger consistency guarantees, but can reduce concurrency or increase contention.

## READ UNCOMMITTED

Allows the weakest isolation.

Conceptually, a transaction may be able to observe data that another transaction has not committed yet.

This can allow **dirty reads**.

## READ COMMITTED

A transaction generally sees committed data.

A common behavior is:

```text
T1 modifies row
T2 reads row → sees previously committed version/value

T1 commits

T2 reads again → may see new committed value
```

The exact behavior depends on the database implementation.

## REPEATABLE READ

Provides stronger guarantees around repeated reads within a transaction.

The same logical row generally remains consistent for repeated reads according to the database's isolation semantics.

## SERIALIZABLE

Provides the strongest standard isolation level.

The goal is that concurrent transactions behave as though they had executed one after another.

Conceptually:

```text
Concurrent execution:

T1 ────────┐
           ├── Database makes result equivalent to:
T2 ────────┘

T1 → T2

or

T2 → T1
```

The exact implementation may use locks, predicate/key-range mechanisms, MVCC, serialization failures, or combinations of these.

---

# 21. Important Distinction: Locks vs Isolation Level

These concepts are related but not identical.

**Lock** answers:

> "What resource is currently protected, and who can access it?"

**Isolation level** answers:

> "What concurrency behavior and visibility guarantees does this transaction receive?"

For example:

```text
Transaction
    │
    ├── Isolation level
    │      └── Defines concurrency/visibility guarantees
    │
    └── Locking/MVCC mechanisms
           └── Used by the database to enforce those guarantees
```

The exact relationship depends heavily on the database engine.

---

# 22. A Practical Bank Transfer Example

Suppose we transfer `$50` from Alice to Bob.

```sql
BEGIN;

-- Lock/read Alice
SELECT balance
FROM accounts
WHERE id = 1
FOR UPDATE;

-- Lock/read Bob
SELECT balance
FROM accounts
WHERE id = 2
FOR UPDATE;

UPDATE accounts
SET balance = balance - 50
WHERE id = 1;

UPDATE accounts
SET balance = balance + 50
WHERE id = 2;

COMMIT;
```

Conceptually:

```text
BEGIN
  ↓
Lock Alice
  ↓
Lock Bob
  ↓
Check balances
  ↓
Subtract from Alice
  ↓
Add to Bob
  ↓
COMMIT
  ↓
Release locks
```

The transaction gives us an important property:

```text
Before:

Alice = 100
Bob   = 100

After:

Alice = 50
Bob   = 150
```

We don't want a situation where the application crashes after:

```text
Alice = 50
Bob   = 100
```

A transaction lets the database treat the operation as one atomic unit:

```text
Transfer succeeds completely
        OR
Transfer is rolled back
```

---

# 23. The Most Important Mental Model

When working with databases, think about concurrency like this:

```text
                    DATABASE
                       │
             ┌─────────┴─────────┐
             │                   │
          T1                      T2
             │                   │
        wants data            wants data
             │                   │
             └─────────┬─────────┘
                       ↓
              Concurrency control
                       │
             ┌─────────┴─────────┐
             │                   │
           Locks               MVCC
             │                   │
      ┌──────┴──────┐      multiple versions
      │             │
   Shared       Exclusive
      │             │
     Read         Write
```

The database's job is to coordinate concurrent transactions so that their results satisfy the guarantees provided by the database and the selected isolation level.

---

# 24. Quick Summary

### Race condition

```text
Multiple operations
       ↓
access shared state
       ↓
timing affects result
       ↓
incorrect/unpredictable outcome
```

### Deadlock

```text
T1 owns A → waits for B
T2 owns B → waits for A
       ↓
circular waiting
       ↓
deadlock
```

### Shared lock

```text
Used for compatible reading access
S + S → usually allowed
```

### Exclusive lock

```text
Used for modifications
X + X → conflict
S + X → conflict
```

### Row lock

```text
Protects a row/resource
Good concurrency
```

### Table lock

```text
Protects a larger resource
Potentially less concurrency
```

### Intent lock

```text
Signals intention to acquire lower-level locks
Useful for hierarchical locking
```

### MVCC

```text
Maintains multiple versions/snapshots
Allows many reads without blocking writers
```

### Pessimistic locking

```text
Lock first → then work
```

### Optimistic locking

```text
Work → check version/conflict → commit if still valid
```

### Deadlock prevention

```text
Acquire locks in a consistent order
```

### Transaction

```text
BEGIN
  ↓
Read / Lock
  ↓
Validate
  ↓
Modify
  ↓
COMMIT
```

The key idea is that **race conditions are about incorrect results caused by uncontrolled concurrency, while deadlocks are about circular waiting. Database transactions, locking, MVCC, and isolation levels are the mechanisms databases use to control that concurrency.**
