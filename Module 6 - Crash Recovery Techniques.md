# Module 6 - Crash Recovery Techniques

### Ensuring Durability in a Volatile World

**Goal:** Understand how databases ensure data is never lost, even if the power fails. Learn about the trade-off between performance (RAM) and safety (Disk), and master the two main techniques for crash resilience: Write Ahead Logging (WAL) and Shadow Paging.

## 1. Introduction

In previous modules, we discussed how to handle multiple users at once (Concurrency). Now, we must discuss **Reliability**.

**The Problem**

Databases manipulate data in **RAM (Buffer Pool)** for speed. However, RAM is **volatile**.

* If Transaction A updates a row in memory.
* And the power goes out *before* that memory page is written to the hard drive.

**The data is lost forever.**

Writing to the hard drive (Disk) for every single change is safe, but incredibly slow.

**The Solution:**

We need protocols that allow us to work in fast RAM while guaranteeing **Atomicity** (all or nothing) and **Durability** (survives crashes).

## 2. The Concept: The "Note First" Rule

To solve the speed vs. safety dilemma, most modern systems use a specific philosophy:

> *Never modify the database on disk until you have written a log of what you are about to do.*

This allows the database to write to disk sequentially (fast) via a log, rather than randomly (slow) via data pages.

## 3. Technique 1: Write Ahead Logging (WAL)

WAL is the industry standard (used by PostgreSQL, MySQL InnoDB, SQL Server, Oracle).

### The Mechanics

The database maintains a separate file called the **Log File** on stable storage.

1.**The Setup:**

* **Data Files:** Store the actual tables (Random Access, Slow).
* **Log File:** Stores a history of actions (Sequential Append-Only, Fast).

2.**The Rule:**

* Before the database writes a modified data page (from RAM to Disk), it **must** ensure the log record describing that change is already on the Disk.

3.**The Workflow:**

* **Step 1:** Transaction starts.
* **Step 2:** User updates a record. The DBMS creates a **Log Entry** in RAM.
  * Format: `<TxnID, Object, Old_Value, New_Value>`
* **Step 3:** The Log Entry is flushed to the **Log File on Disk**.
* **Step 4:** Only *after* the log is safe, the DBMS is allowed to say "Commit".
* **Step 5:** The actual Data Page in RAM is modified. It can be written to disk lazily later (even after the transaction finishes).

### Visual Representation

```plaintext

Time | Action                                     | RAM State      | Disk State

-----+--------------------------------------------+----------------+-------------------

 t1  | Txn 100 updates A from 10 to 20            | A=20 (Dirty)   | A=10

 t2  | WAL Record created <T100, A, 10, 20>       | Log Buffer     | Log File (Empty)

 t3  | WAL Flush (Crucial Step)                   |                | Log File has <T100...>

 t4  | Txn 100 Commits                            |                |

 t5  | System Crashes!                            | (Lost)         | A=10, Log=<T100...>
```

**Recovery:** When the system restarts, it sees `A=10` on disk, but the Log says `T100 set A=20`. The database **replays** the log to make `A=20.`

### WAL Example

![1764763230662](image/Module6-CrashRecoveryTechniques/1764763230662.png)

The log buffer is flushed to disk when the buffer is full or if there is a timeout (e.g., 5 ms)

### Checkpoints

> *The DBMS's WAL will grow faster*
>
> *After a crash, the DBMS must replay the entire log, which will take a long time*

The DBMS periodically takes a checkpoint where it flushes all buffers out of disk.

* Hint on how far back it needs to replay the WAL after a crash
* Truncate the WAL upto a certain safe point in time

#### Blocking / Consistent Checkpoint Protocol

* Pause all queries
* Flush all WAL records in memory to disk
* Flush all modified pages in the buffer pool to disk
* Write a `<CHECKPOINT>` entry to WAL and flush to disk
* Resume queries

#### Checkpoint Frequency

> *Frequent checkpointing causes the runtime performance to degrade System spends too much time flushing the buffers*

Waiting a long time:

* The checkpoint will be large and slow
* Makes recovery time much longer

Strategy:

* Checkpoint based on time (every 5 minutes)
* Checkpoint based on WAL size (if WAL file reaches 512MB -> flush)

#### Checkpoint Example

![1764764790338](image/Module6-CrashRecoveryTechniques/1764764790338.png)

## 4. Technique 2: Shadow Paging

Shadow Paging is an alternative technique (used by CouchDB, LMDB, and older systems) that avoids in-place updates entirely.

### The Analogy: The Photocopier

Imagine you want to edit a page in a book.

1. **WAL Approach:** You write notes on a separate notepad (Log), then erase and rewrite the actual book page later.
2. **Shadow Paging Approach:** You photocopy the page. You edit the  *photocopy* . Once you are done, you swap the page in the book with your new photocopy and throw the old one away.

### The Mechanics

The database organizes data into pages and uses a **Page Table** to track where data lives on disk.

1. **Copy-on-Write:**
   * When a transaction wants to modify a page, the DBMS makes a copy of it (the  **Shadow Page** ).
   * The original page (Current) is left untouched.
2. **Modification:**
   * All updates are applied to the Shadow Page.
3. **Commit:**
   * To commit, the DBMS updates the **Root Pointer** to point to the new Shadow Page instead of the old one.
   * This switch is atomic (a single hardware operation).

### Visual Representation

```plaintext

Before Commit:
[ Root ] ----> [ Old Page Table ] ----> [ Page A (Data: 10) ]
                                   \--> [ Shadow Page A (Data: 20) ] (Not active yet)

After Commit:
[ Root ] --/-> [ Old Page Table ]       [ Page A (Data: 10) ] (Now Garbage)
           \-> [ New Page Table ] ----> [ Shadow Page A (Data: 20) ] (Now Active)

```

### Shadow Paging Example

Scenario: Update a row value from `10` to `20` in `Page 5.`

![1764766191214](image/Module6-CrashRecoveryTechniques/1764766191214.png)

## 5. Comparison: WAL vs. Shadow Paging

| Feature                 | Write Ahead Logging (WAL)              | Shadow Paging                                       |
| :---------------------- | :------------------------------------- | :-------------------------------------------------- |
| **Write Speed**   | **Fast** (Sequential log writes) | **Slower** (Random disk writes for new pages) |
| **Storage**       | Low overhead (small logs)              | High overhead (duplicate pages)                     |
| **Recovery**      | Complex (Must replay logs)             | **Instant** (Just keep the old pointer)       |
| **Fragmentation** | Low                                    | High (Data gets scattered)                          |
| **Usage**         | MySQL, Postgres, Oracle                | CouchDB, LMDB, ZFS                                  |

## 6. Glossary

* **Atomicity:** The property that ensures a transaction is treated as a single unit; it either completely succeeds or completely fails.
* **Durability:** The guarantee that once a transaction has been committed, it will remain committed even in the case of a system failure.
* **Dirty Page:** A page in the Buffer Pool (RAM) that has been modified but not yet written to the disk.
* **LSN (Log Sequence Number):** A unique identifier for every entry in the WAL, used to order events.
* **Checkpoint:** A point in time where the DBMS forces all dirty pages to disk to shorten recovery time.

## 7. Key Takeaways

* **RAM is Dangerous:** We cannot trust volatile memory. We must write to disk to guarantee durability.
* **Sequential vs. Random:** WAL is popular because appending to a log file (Sequential I/O) is much faster than finding and updating specific rows (Random I/O).
* **The Golden Rule:** In WAL, the log record *must* hit the disk before the data page does.
* **No Undo Needed (Shadow Paging):** Shadow paging makes recovery easy because we never overwrite the "good" data until the very last moment, but it fragments data on the disk.

---

End of Module 6
