# Isolation Levels Overview

## Objectives

- Explain why different isolation levels exist
- Define isolation anomalies
- Show how each isolation level addresses (or does not address) anamolies
- Choose minimal level for a business invariant

---

### **Key Concepts**

**Isolation**  = rules about what a transaction can "see" while others change data

**Levels (PostgreSQL):**

1. **READ COMMITTED:** each statement sees the latest committed data
2. **REPEATABLE READ:** one snapshot for entire transaction (snapshot isolation)
3. **SERIALIZABLE:** snapshot + conflict detection (abort if unsafe interleaving/ordering)

---

### Scenario: Movie Seat Booking

Two users:

* Session A
* Session A

Show S1 has seats:

* Some house (reserved) seats (kept aside for VIP / guest pass)
  * Policy: at least one house seat must remain free until release)
* Users list available seats, then reserve
* No double booking of the same seat

---

### Data Model

````sql
CREATE DATABASE txn_lab;

\c txn_lab;

CREATE TABLE shows(
  id SERIAL PRIMARY KEY,
  code TEXT UNIQUE
);

CREATE TABLE seats(
  id SERIAL PRIMARY KEY,
  show_id INT REFERENCES shows(id),
  seat_no TEXT,
  house BOOLEAN DEFAULT FALSE,      -- house seat (kept aside)
  reserved BOOLEAN NOT NULL DEFAULT FALSE,
  UNIQUE(show_id, seat_no)
);

- Populate movie and seat details
INSERT INTO shows(code) VALUES ('S1');
INSERT INTO seats(show_id, seat_no, house)
SELECT 1,'A1',TRUE UNION ALL
SELECT 1,'A2',TRUE UNION ALL
SELECT 1,'A3',FALSE UNION ALL
SELECT 1,'A4',FALSE;
````

---

## Isolation Anomalies

| Anomaly                         | Definition                                                                                  | Example (Session A vs Session B)                                                                  | Prevented By                                                                |
| ------------------------------- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| Dirty Read                      | Reading uncommitted changes from another transaction                                        | Session A sees Session B’s tentative reservation before COMMIT                                   | Not possible in PostgreSQL (MVCC)                                           |
| Non-Repeatable Read             | Same row/set read twice within one transaction, value differs                               | Session A lists free seats (A1–A4); after Session B reserves A2, Session A lists again: A1,A3,A4 | REPEATABLE READ / SERIALIZABLE                                              |
| Phantom Read                    | Predicate result changes (rows added/removed matching a filter)                             | Session A counts free house seats (2). Session B reserves A2. Session A counts again: 1           | REPEATABLE READ (snapshot consistency) / SERIALIZABLE                       |
| Lost Update                     | Both read same value, derive new value, one overwrites                                      | Free_count goes 4→3 instead of 4→2                                                              | Use atomic UPDATE or SELECT FOR UPDATE (not fixed by isolation level alone) |
| Write Skew                      | Two transactions read same invariant set, update different rows; together violate invariant | Session A reserves A1; Session B reserves A2; invariant “at least one house seat free” breaks   | SERIALIZABLE (abort + retry) or explicit locking                            |
| Read Skew (Snapshot vs current) | Transaction sees a mix of old and new states across multiple reads                          | Session A sees A2 free early, later uses stale assumption                                         | REPEATABLE READ / SERIALIZABLE mitigate by freezing view                    |

**Note:** Double booking (same seat) prevented by `UNIQUE(show_id, seat_no)` regardless of isolation level.

---

### Snapshot Differences

**READ COMMITTED** (Non-Repeatable Read)

```
Session A                          | Session B
-----------------------------------+-----------------------------
BEGIN                              |
SELECT free seats -> A1,A2,A3,A4   |
                                   | UPDATE A2 reserved=TRUE; COMMIT
SELECT free seats -> A1,A3,A4      |
COMMIT                             |
```

**Demo:**

```sql
-- Session A
BEGIN;
SELECT seat_no FROM seats WHERE show_id = 1 AND reserved = FALSE ORDER BY seat_no;

/***
 seat_no
---------
 A1
 A2
 A3
 A4
(4 rows)
***/

-- wait

-- Session B
UPDATE seats SET reserved = TRUE WHERE show_id = 1 AND seat_no = 'A2';


-- Session A
SELECT seat_no FROM seats WHERE show_id=1 AND reserved=FALSE ORDER BY seat_no;

/***
 seat_no
---------
 A1
 A3
 A4
(3 rows)
***/

```

---

**REPEATABLE READ** (Stale Snapshot)

```
Session A (RR)                     | Session B
-----------------------------------+-----------------------------
BEGIN ISOLATION LEVEL RR           |
SELECT free seats -> A1,A2,A3,A4   |
                                   | UPDATE A2 reserved=TRUE; COMMIT
SELECT free seats -> A1,A2,A3,A4   | (Session A still sees A2 free)
UPDATE A2 reserved=TRUE (0 rows)   |
COMMIT  
```

**Demo:**

```sql
-- Reset
UPDATE seats SET reserved = FALSE WHERE show_id = 1;

-- Validate
SELECT * FROM seats;

-- Session A
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT seat_no FROM seats WHERE reserved = FALSE AND show_id = 1 ORDER BY seat_no;
/***
 seat_no
---------
 A1
 A2
 A3
 A4
(4 rows)
***/
-- wait

-- Session B
UPDATE seats SET reserved = TRUE WHERE show_id = 1 AND seat_no = 'A2'; COMMIT;

--Session A
SELECT seat_no FROM seats WHERE reserved = FALSE AND show_id = 1 ORDER BY seat_no;
/***
 seat_no
---------
 A1
 A2
 A3
 A4
(4 rows)
***/

UPDATE seats SET reserved = TRUE WHERE show_id = 1 AND seat_no = 'A2';

-- ERROR: could not serialize access due to concurrent update

COMMIT;
-- ROLLBACK

```

---

**Lost Update** (Counter - Total Free Seats)

Lost Update = both transactions read the same base value, compute a new value locally, then write-last write overwrites the first.

**Create counter table (materialized free seats):**

```sql
CREATE TABLE show_stats(
  show_id INT PRIMARY KEY REFERENCES shows(id),
  free_count INT,
  version INT DEFAULT 0
);

INSERT INTO show_stats(show_id, free_count)
SELECT 1, COUNT(*) FROM seats WHERE show_id=1 AND reserved=FALSE;

```

**Initial:**

```
SELECT * FROM show_stats; -- free_count = 4
```

**Snapshot difference:**

```
Session A                            | Session B
-------------------------------------+--------------------------------
BEGIN                                | BEGIN
SELECT free_count -> 4               | SELECT free_count -> 4
-- compute new = 3                   | -- compute new = 3
UPDATE show_stats SET free_count=3   | UPDATE show_stats SET free_count=3
COMMIT                               | COMMIT (overwrites Session A's change)
Final free_count = 3 (should be 2)
```

**Demo:**

```sql
-- Session A
BEGIN;
SELECT free_count FROM show_stats WHERE show_id=1; -- 4
UPDATE show_stats SET free_count=3 WHERE show_id=1;
-- (Do not commit yet)

-- Session B
BEGIN;
SELECT free_count FROM show_stats WHERE show_id=1; -- still 4 (Session A not committed)
UPDATE show_stats SET free_count=3 WHERE show_id=1;
COMMIT; -- Session B commits first

-- Session A
COMMIT;

SELECT free_count FROM show_stats WHERE show_id=1; -- 3 (lost one decrement)
```

Why isolation levels don’t fix it:

* **REPEATABLE READ** keeps snapshot stable but both still base off “4”.
* **SERIALIZABLE** may allow it if each only writes and doesn’t read updated value after the other commits-no invariant violation unless defined.

**Preventive Approaches**

1. Atomic in-place update:

   ```sql
   UPDATE show_stats
   SET free_count = free_count - 1
   WHERE show_id=1 AND free_count > 0;
   ```

   **Safe:** Each reservation uses a single statement; no lost update because subtraction happens directly in the database.
2. Pessimistic locking:

   ```sql
   BEGIN;
   SELECT free_count FROM show_stats WHERE show_id=1 FOR UPDATE;
   UPDATE show_stats SET free_count = free_count - 1 WHERE show_id=1 AND free_count > 0;
   COMMIT;
   ```

   Prevents concurrent overwrite by forcing serialization on the row.
3. Optimistic version check:

   ```sql
   BEGIN;
   SELECT free_count, version FROM show_stats WHERE show_id=1;
   -- Suppose free_count=4, version=0
   UPDATE show_stats
   SET free_count = free_count - 1,
       version = version + 1
   WHERE show_id=1 AND version = 0;
   -- Check rows affected = 1 else retry
   COMMIT;
   ```

   If a concurrent transaction already updated (version changed), this update affects 0 rows; client retries with fresh read.

---

**Write Skew** (REPEATABLE READ)

```
Session A (RR)                         | Session B (RR)
-----------------------------------+-----------------------------
BEGIN                              | BEGIN
SELECT house free -> A1,A2         | SELECT house free -> A1,A2
UPDATE A1 reserved=TRUE            | UPDATE A2 reserved=TRUE
COMMIT                             | COMMIT
Result: A1,A2 both reserved (invariant broken)
```

**Demo:**

```sql
UPDATE seats SET reserved=FALSE WHERE show_id=1;

-- Session A:
 SELECT seat_no FROM seats WHERE house = TRUE and show_id = 1 and reserved = FALSE;

/***
 seat_no
---------
 A1
 A2
(2 rows)
***/

UPDATE seats SET reserved = TRUE WHERE show_id = 1 AND seat_no = 'A1';

-- wait


-- Session B
BEGIN ISOLATION LEVEL REPEATABLE READ;

SELECT seat_no FROM seats WHERE house = TRUE AND show_id = 1 AND reserved = FALSE;

/***
 seat_no
---------
 A1
 A2
(2 rows)
***/

UPDATE seats SET reserved = TRUE WHERE show_id = 1 AND seat_no = 'A2'; COMMIT;

-- Session A
COMMIT;

SELECT seat_no FROM seats WHERE house = TRUE AND show_id = 1 AND reserved = FALSE;

/***
 seat_no
---------
 A1
 A2
(2 rows)

-- Violates the policy
***/
```

---

**SERIALIZABLE**

```
Session A (SERIALIZABLE)               | Session B (SERIALIZABLE)
-----------------------------------+-----------------------------
BEGIN                              | BEGIN
SELECT house free -> A1,A2         | SELECT house free -> A1,A2
UPDATE A1 reserved=TRUE            | UPDATE A2 reserved=TRUE
COMMIT (OK)                        | COMMIT → ERROR 40001 (abort)
Retry Session B → sees updated state; invariant preserved
```

**Demo:**

```sql
-- Reset

UPDATE seats SET reserved = FALSE WHERE show_id = 1;

-- Session A
BEGIN ISOLATION LEVEL SERIALIZABLE;

SELECT seat_no FROM seats WHERE show_id = 1 AND house = TRUE AND reserved = FALSE;
/***
 seat_no
---------
 A1
 A2
(2 rows)
***/

UPDATE seats SET reserved = TRUE WHERE show_id = 1 AND seat_no = 'A1';


-- Session B
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT seat_no FROM seats WHERE show_id = 1 AND house = TRUE AND reserved = FALSE;

/***
 seat_no
---------
 A1
 A2
(2 rows)
***/

UPDATE seats SET reserved = TRUE WHERE show_id = 1 AND seat_no = 'A2';
COMMIT;

-- Session A
COMMIT;
/***
ERROR:  could not serialize access due to read/write dependencies among transactions
DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
HINT:  The transaction might succeed if retried.
***/
```

---

### Choosing Minimal Mechanism

| Goal                           | Mechanism                            |
| ------------------------------ | ------------------------------------ |
| List seats                     | READ COMMITTED                       |
| Stable multi-read availability | REPEATABLE READ                      |
| House seat invariant           | SERIALIZABLE or guard row            |
| Avoid lost update on counter   | Atomic UPDATE / FOR UPDATE / version |
| Double booking prevention      | UNIQUE constraint                    |

---

### Summary

* Lost update requires coordination (atomic, lock, version).
* Write skew prevented by SERIALIZABLE if predicate read occurs.
* Non-repeatable + phantom solved by REPEATABLE READ snapshot.
* SERIALIZABLE adds logical safety (abort cycles).
* Retry on 40001 is normal.

---

### Reflection

1. What creates lost update? Separate read-modify-write.
2. Why atomic UPDATE solves it? Single evaluation at commit time.
3. Why write skew needs invariant read? To form dependency edges.
4. Why SERIALIZABLE aborts? No valid serial order preserving both decisions.

---
