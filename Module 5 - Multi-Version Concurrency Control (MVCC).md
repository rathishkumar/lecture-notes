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

> *"Readers never block Writers, and Writers never block Readers."*

## 2. The Concept: Snapshots

Instead of overwriting a record immediately, MVCC keeps **multiple versions** of the data alive at the same time.

### The Analogy: The Shared Document

Imagine a Google Doc, but with a twist:

1. **User A** opens the document at 9:00 AM. They see "Version 1".
2. **User B** opens the document at 9:05 AM and starts typing. They are creating "Version 2".
3. **User A** continues to read. They **do not** see User B's typing. They still see the clean "Version 1" snapshot from 9:00 AM.

User A is reading the **past** (consistent state), while User B is writing the **future**.

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
   * 2. Create a new row with the new data (`xmin` = Current ID).

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



## 5. Implementation Example

Let's look at how PostgreSQL handles this using the hidden system columns.

### Setup

```sql

-- Create a simple table
CREATE TABLE accounts (
    id INT PRIMARY KEY,
    balance INT
);

-- Insert initial data (Assume Transaction ID = 10)
INSERT INTO accounts VALUES (1, 100);

```


### The Workflow

**Step 1: Update the Data (Transaction 20)**
We update the balance from 100 to 200.

```sql

BEGIN; -- TxID 20
UPDATE accounts SET balance = 200 WHERE id = 1;
COMMIT;

```


**Internal State of DB:**
The database now contains **two** physical rows for ID 1.

| Row Address | ID | Balance | xmin (Born)  | xmax (Died)  | Status     |
| :---------- | :- | :------ | :----------- | :----------- | :--------- |
| 0x01        | 1  | 100     | 10           | **20** | Dead (Old) |
| 0x02        | 1  | 200     | **20** | NULL         | Live (New) |

**Step 2: A New Reader Arrives (Transaction 30)**
Transaction 30 starts.

* It ignores Row 0x01 because `xmax` (20) is less than 30 (it knows it died).
* It reads Row 0x02 because `xmin` (20) is less than 30 (it knows it was born).
* **Result:** Reads 200.

**Step 3: An Old Reader (Time Travel)**
Imagine a long-running report (Transaction 15) is still running.

* It looks at Row 0x02: `xmin` is 20. 20 > 15. "This is from the future." **Ignore.**
* It looks at Row 0x01: `xmax` is 20. 20 > 15. "This deletion happened in the future." **Valid.**
* **Result:** Reads 100.


## 6. The Cost of MVCC: Garbage Collection

Since updates create new copies, the database grows indefinitely if we don't clean up.

### The Problem: Bloat

If we update a row 1,000 times, we have 1,000 versions. 999 of them are useless once all old transactions have finished. This is called  **Bloat** .

### The Solution: VACUUM

The database runs a background process (called `VACUUM` in PostgreSQL or `Purge` in Oracle) to physically remove dead versions that are no longer visible to *any* active transaction.

## 7. Comparison: 2PL vs. MVCC

| Feature               | Two-Phase Locking (2PL)     | MVCC                                         |
| :-------------------- | :-------------------------- | :------------------------------------------- |
| **Concurrency** | Low (Readers block Writers) | **High** (Readers/Writers coexist)     |
| **Storage**     | Low (Update in place)       | **High** (Keeps multiple versions)     |
| **Maintenance** | Low                         | **High** (Needs Vacuum/Purge)          |
| **Read Speed**  | Fast (Reads latest)         | Slower (Must traverse version chain)         |
| **Rollback**    | Complex (Needs Undo Log)    | **Instant** (Just ignore new versions) |

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
