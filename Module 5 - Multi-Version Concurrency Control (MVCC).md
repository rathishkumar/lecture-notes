# Module 5 - Multi-Version Concurrency Control (MVCC)

### Concurrency with History

**Goal:** Understand how modern databases allow Readers and Writers to operate simultaneously without blocking each other. Learn about Snapshots, Version Chains, Visibility Rules, and the concept of Vacuuming.

## 1. Introduction

In Module 3 (2PL), we learned that to maintain consistency, we often have to lock data.

* If Transaction A is **Reading**, Transaction B cannot **Write**.
* If Transaction A is **Writing**, Transaction B cannot **Read**.

**The Problem:**
In high-traffic systems (like Amazon or Facebook), reads happen constantly. If every read blocked a write, the system would grind to a halt.

**The Solution: MVCC**
Multi-Version Concurrency Control is the industry standard (used by PostgreSQL, Oracle, MySQL InnoDB, SQL Server). The core philosophy is:

> *"Readers never block Writers,*
>
> *Writers never block Readers."*

## 2. The Concept: Snapshots

Instead of overwriting a record immediately, MVCC keeps **multiple versions** of the data alive at the same time.

### The Analogy: The Shared Document

Imagine a Google Doc, but with a twist:

1. **User A** opens the document at 9:00 AM. They see "Version 1".
2. **User B** opens the document at 9:05 AM and starts typing. They are creating "Version 2".
3. **User A** continues to read. They **do not** see User B's typing. They still see the clean "Version 1" snapshot from 9:00 AM.

User A is reading the **past** (consistent state), while User B is writing the **future**.

The DBMS maintains **multple physical versions** of a single logical object in the database:

* When txn writes to an object, the DBMS creates new version of the object
* When txn reads an object, it reads the newest version, that existed when txn started
* Use timestamp to determine visibility

> Multi-versioning without garbage collection allows the DBMS to support ***time-travel*** queries.

## 3. How MVCC Works (The Mechanics)

To implement this, the database adds hidden columns to every row to track the "lifespan" of a version.

### The Hidden Columns (PostgreSQL Style)

* **`xmin` (Created By):** The Transaction ID that created this version.
* **`xmax` (Deleted By):** The Transaction ID that deleted (or updated) this version.

### The Operations

1. **INSERT:**

   * Create a new row version.
   * Set `xmin` = Current Transaction ID.
   * Set `xmax` = NULL (It is currently alive).
2. **DELETE:**

   * **Do not remove the row physically.**
   * Set `xmax` = Current Transaction ID.
   * The row is now "dead" to any transaction starting *after* this one, but "alive" for older transactions.
3. **UPDATE:**

   * MVCC treats an UPDATE as a **DELETE + INSERT**.
   * 1. Mark the old row as deleted (`xmax` = Current ID).
     2. Create a new row with the new data (`xmin` = Current ID).

### Demo

```sql

CREATE TABLE txn_demo ( id INT PRIMARY KEY, val INT NOT NULL) ;
INSERT INTO txn_demo VALUES (1, 100), (2, 200)

```

Viewing version details:

```sql

SELECT ctid, xmin, xmax, * FROM txn_demo;

 ctid  | xmin | xmax | id | val
-------+------+------+----+-----
 (0,1) |  960 |    0 |  1 | 100
 (0,2) |  960 |    0 |  2 | 200
(2 rows)

```

Session A:

```sql

BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

SELECT ctid, xmin, xmax, * FROM txn_demo;

 ctid  | xmin | xmax | id | val
-------+------+------+----+-----
 (0,1) |  960 |    0 |  1 | 100
 (0,2) |  960 |    0 |  2 | 200
(2 rows)

-- update on session B and run the select again

SELECT ctid, xmin, xmax, * FROM txn_demo;

SELECT ctid, xmin, xmax, * FROM txn_demo;
 ctid  | xmin | xmax | id | val
-------+------+------+----+-----
 (0,2) |  960 |    0 |  2 | 200
 (0,3) |  961 |    0 |  1 | 101
(2 rows)



```

Session B

```sql

BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

SELECT ctid, xmin, xmax, * FROM txn_demo;

 ctid  | xmin | xmax | id | val
-------+------+------+----+-----
 (0,1) |  960 |    0 |  1 | 100
 (0,2) |  960 |    0 |  2 | 200
(2 rows)

UPDATE txn_demo SET val = val + 1 WHERE id = 1 RETURNING txid_current();
 txid_current
--------------
          961
(1 row)


SELECT ctid, xmin, xmax, * FROM txn_demo;
 ctid  | xmin | xmax | id | val
-------+------+------+----+-----
 (0,2) |  960 |    0 |  2 | 200
 (0,3) |  961 |    0 |  1 | 101
(2 rows)


```

![1764589007748](image/Module5-Multi-VersionConcurrencyControl(MVCC)/1764589007748.png)

### Transaction Status Table

* This table keeps track of transaction status
* Eg: txn id 961 Aborted & txn id 962 Committed, etc.

![1764591348368](image/Module5-Multi-VersionConcurrencyControl(MVCC)/1764591348368.png)

## 4. Visibility Rules: Who sees what?

When a transaction $T$ tries to read a row, the database runs a visibility check on every version of that row.

**A row version is visible to Transaction $T$ if:**

1. **Creation Check:** The transaction that created it (`xmin`) committed *before* $T$ started.
2. **Deletion Check:** The transaction that deleted it (`xmax`) had *not* committed when $T$ started (or it hasn't been deleted yet).

**Visual Representation:**

```plaintext
Time  |  Transaction 100 (Reader)   |  Transaction 101 (Writer)
------+-----------------------------+---------------------------
 t1   |  Starts (Snapshot taken)    |
 t2   |  Reads Row A (Value: 50)    |
 t3   |                             |  Starts
 t4   |                             |  Updates Row A to 99
 t5   |                             |  Commits
 t6   |  Reads Row A again          |
      |  -> SEES 50 (Old Version)   |
      |  (Because T101 is "future") |
```

## 5. Version Storage

**Approach 1: Append-Only Storage**

* New versions are appended to same table space
* PostgreSQL use this approach

**Approach 2: Time-Travel Storage**

* Old versions are copied to separate table storage

**Approach 3: Delta Storage**

* The original values of the modified attributes are copied into a separate delta record space.
* Most common version

## 6. Version Chain Ordering

![1764590904640](image/Module5-Multi-VersionConcurrencyControl(MVCC)/1764590904640.png)

## 7. The Cost of MVCC: Garbage Collection

Since updates create new copies, the database grows indefinitely if we don't clean up.

### The Problem: Bloat

If we update a row 1,000 times, we have 1,000 versions. 999 of them are useless once all old transactions have finished. This is called  **Bloat**.

### The Solution: VACUUM

The database runs a background process (called `VACUUM` in PostgreSQL or `Purge` in Oracle) to physically remove dead versions that are no longer visible to *any* active transaction.

## 8. Glossary

* **Snapshot Isolation:** An isolation level where a transaction sees data exactly as it was at the start of the transaction.
* **Write Skew:** A specific anomaly in MVCC where two transactions read overlapping data, make disjoint updates, and both commit, violating logic (e.g., two doctors going off-call simultaneously).
* **Vacuum:** The process of reclaiming storage occupied by dead row versions.
* **Dirty Read:** Reading a version created by a transaction that hasn't committed yet (MVCC naturally prevents this).

## 9. Key Takeaways

1. **Performance:** MVCC is excellent for read-heavy workloads because reads are never blocked by locks.
2. **Space vs. Time:** MVCC trades storage space (keeping old versions) for concurrency (speed).
3. **Updates are Inserts:** In MVCC, an `UPDATE `is actually a `DELETE `of the old version and an `INSERT` of the new one.
4. **Cleanup is Critical:** Without a proper Garbage Collection process (Vacuum), an MVCC database will run out of disk space and slow down.

End of Module 5
