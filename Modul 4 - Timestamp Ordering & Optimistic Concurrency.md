# Module 4 - Timestamp Ordering & Optimistic Concurrency

### Concurrency without Locks

**Goal:** Understand how databases can ensure serializability *without* using locks, by using Timestamps and Optimistic protocols. Learn the "Read-Validate-Write" phases and how to implement Optimistic Locking in SQL.

---

## 1. Introduction

In Module 3, we studied **Two-Phase Locking (2PL)**. This is a **Pessimistic** approach. It assumes conflict *will* happen, so it locks the door before doing anything.

**The Downside of Locking:**

* Locks cause waiting (blocking).
* Locks can lead to Deadlocks.
* Managing lock tables is expensive for the CPU.

**The Alternative:**
What if we assume conflicts are rare? We can let transactions run freely and only check for problems later. This family of protocols relies on **Time** (Timestamps) rather than **Locks**.

---

## 2. The Concept: Timestamp Ordering (T/O)

In Timestamp Ordering, the database assigns a unique **Timestamp (TS)** to every transaction when it begins.

* **TS(T):** The timestamp of the Transaction.
* **W-TS(X):** The timestamp of the *last transaction* that successfully **wrote** to object X.
* **R-TS(X):** The timestamp of the *last transaction* that successfully **read** object X.

**The Golden Rule:**

> *If a transaction tries to access data from the "future" (written by a younger transaction), it must be aborted and restarted with a new timestamp.*

### The Analogy: The Deli Counter

Imagine a deli where everyone takes a ticket number (Timestamp).

* If you hold ticket #5, you cannot process an order if the clerk has already served ticket #10.
* You are "too late." You must throw away ticket #5, get a new ticket (e.g., #15), and try again.

---

## 3. Basic T/O Protocol Rules

The database checks the timestamp every time a read or write is attempted.

### 1. Read Rule

Transaction **T** wants to Read Object **X**.

* **Check:** Is `W-TS(X) > TS(T)`?
  * *Translation:* Did someone *newer* than me already write this?
  * **Yes:** Abort T (I am reading obsolete data).
  * **No:** Allow Read. Update `R-TS(X)` to `max(R-TS(X), TS(T))`.

### 2. Write Rule

Transaction **T** wants to Write Object **X**.

* **Check:** Is `R-TS(X) > TS(T)` OR `W-TS(X) > TS(T)`?
  * *Translation:* Did someone *newer* than me already read or write this?
  * **Yes:** Abort T (I am trying to overwrite newer history).
  * **No:** Allow Write. Update `W-TS(X)` to `TS(T)`.

---


## 4. Optimistic Concurrency Control (OCC)

OCC is a specific implementation of timestamp-based logic designed for low-conflict environments (like web applications).

**Philosophy:** "Apologize later rather than ask permission."

### The Three Phases of OCC

Unlike 2PL (Grow/Shrink), OCC has three phases:

1. **Read Phase (Work Phase):**
   * Transaction executes, reading values and writing to a **private workspace** (local memory).
   * **No locks** are taken on the database.
2. **Validation Phase:**
   * When the transaction wants to commit, the DBMS checks: "Did anyone change the data I read while I was working?"
   * If conflict detected $\rightarrow$ **Abort**.
3. **Write Phase:**
   * If validation passes, copy changes from private workspace to the real database.

**Visual Representation:**

```plaintext
    T1 Starts                                     T1 Commits
       |                                              |
       v                                              v
       |-------------------|--------------------------|-------|
           Read Phase           Validation Phase      Write Phase
         (Local Changes)        (Check Conflicts)    (Apply to DB)
```

---



## 5. Implementing Optimistic Locking in PostgreSQL

While PostgreSQL uses MVCC internally, developers often implement **Application-Level Optimistic Locking** to handle long-running web forms (e.g., a user editing a profile for 5 minutes).

We use a `version` number to detect conflicts.

### Setup

```sql

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    stock INT,
    version INT DEFAULT 1
);

INSERT INTO products (name, stock) VALUES ('Headphones', 50);

```



### The Workflow

**Step 1: Read the Data (Read Phase)**
User A reads the product. Note the version is  **1** .

```sql

SELECT id, stock, version FROM products WHERE id = 1;
-- Returns: stock=50, version=1

```


*(User A takes 1 minute to think...)*

**Step 2: Concurrent Update (Conflict)**
Meanwhile, User B buys a headphone.

```sql

UPDATE products 
SET stock = 49, version = version + 1 
WHERE id = 1;
-- New version is now 2

```

**Step 3: User A Tries to Update (Validation + Write Phase)**
User A tries to set stock to 45. They pass `version = 1` (the version they saw).

```sql

UPDATE products 
SET stock = 45, 
    version = version + 1 
WHERE id = 1 
  AND version = 1; -- <--- CRITICAL CHECK

```


**Result:**

* **Rows Affected: 0**
* The condition `version = 1` failed because the current version is 2.
* The application detects "0 rows updated" and tells User A: *"Sorry, someone modified this record while you were looking at it. Please refresh."*

---



## 6. Comparison: 2PL vs. T/O vs. OCC

| Feature                       | Two-Phase Locking (2PL)         | Timestamp Ordering (T/O)             | Optimistic (OCC)                   |
| :---------------------------- | :------------------------------ | :----------------------------------- | :--------------------------------- |
| **Approach**            | Pessimistic (Wait)              | Pessimistic/Timestamp                | Optimistic (Validate)              |
| **Conflict Resolution** | Block/Wait                      | Kill (Abort) immediately             | Kill (Abort) at Commit             |
| **Deadlocks?**          | **Yes** (Needs detection) | **No** (Wait-free)             | **No**                       |
| **Throughput**          | Good for high conflict          | Good for distributed systems         | Best for low conflict (Read-heavy) |
| **Starvation?**         | Possible                        | Possible (Long txns keep restarting) | Possible                           |

---

## 7. Glossary

* **Timestamp:** A monotonically increasing number (or system clock time) assigned to a transaction.
* **Thomas Write Rule:** An optimization that ignores outdated writes instead of aborting the transaction.
* **Private Workspace:** A temporary memory area where OCC transactions keep their changes before committing.
* **Validation:** The process of checking if a transaction's local changes conflict with the database state.
* **Starvation:** When a transaction is repeatedly aborted and restarted (common in T/O if a long transaction keeps conflicting with short ones).

---

# 8. Key Takeaways

1. **No Locks Needed:** You can ensure serializability without locks by using timestamps.
2. **Abort vs. Wait:** T/O and OCC prefer to **abort** transactions rather than make them wait. This eliminates deadlocks but increases the retry rate.
3. **Use Case:**
   * Use **2PL** (Standard DBs) when conflicts are frequent.
   * Use **OCC** (Application logic) when conflicts are rare (e.g., "Edit Profile" pages).
1. **Version Column:** The standard way to implement Optimistic Locking in SQL is adding a `version` column and checking it in the `WHERE` clause.

---

End of Module 4
