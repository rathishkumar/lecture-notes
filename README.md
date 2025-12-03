# Database Transactions, Concurrency Control and Recovery

**Guest Lecture Series – Lecture Notes Repository**

**Welcome!**

This repository contains all lecture notes, diagrams, and examples used in the Database Internals guest lectures. The goal is to help you understand how a database actually works beneath SQL, specifically how it handles many users at once and how it keeps your data safe.

## 🎯 What You’ll Learn

By going through these modules, you’ll understand:

* How databases ensure ACID guarantees
* How transactions run without interfering with each other
* Why anomalies like dirty reads or deadlocks happen
* How modern databases (PostgreSQL, MySQL, etc.) implement locks, timestamps, MVCC, and WAL
* How databases recover safely after a crash

We use a simple, intuitive analogy throughout the course:

A Movie Seat Booking System — to explain concurrency, conflicts, and isolation.

## 📚 Modules Overview

### Part 1: Concurrency Control

| **#** | **Module**                                                                                                    | **Description**                                                                        |
| ----------- | ------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| 01          | [Transaction Fundamentals](Lecture%20Notes/Module%201%20-%20Transaction%20and%20Concurrency%20Control.md)              | What is a transaction? Understanding ACID, transaction lifecycle, and the basics of locking. |
| 02          | [Isolation Levels](Lecture%20Notes/Module%202%20-%20Isolation%20Levels.md)                                             | Dirty reads, non-repeatable reads, phantoms, write skew, and how databases prevent them.     |
| 03          | [Two-Phase Locking (2PL)](Lecture%20Notes/Module%203%20-%20Two%20Phase%20Locking.md)                                   | Shared vs. exclusive locks, growing & shrinking phases, strict 2PL, and deadlocks.           |
| 04          | [Timestamp Ordering &amp;OCC](Lecture%20Notes/Module%204%20-%20Timestamp%20Ordering%20&%20Optimistic%20Concurrency.md) | How databases handle concurrencywithoutlocks using timestamps and optimistic validation.     |
| 05          | [MVCC](Lecture%20Notes/Module%205%20-%20Multi-Version%20Concurrency%20Control%20(MVCC).md)                             | How snapshot isolation works. Version chains, visibility rules, and why vacuuming is needed. |

### Part 2: Crash Recovery

| **#** | **Module**                                                                            | **Description**                                                          |
| ----------- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| 06          | [Crash Recovery Techniques](Lecture%20Notes/Module%206%20-%20Crash%20Recovery%20Techniques.md) | Why recovery is hard. Introducing Write-Ahead Logging (WAL) and Shadow Paging. |
| 07          | [ARIES Recovery](Lecture%20Notes/Module%207%20-%20ARIES%20Recovery.md)                         | The standard recovery algorithm:Analysis → Redo → Undo.                      |

## 🛠️ What’s Inside This Repo

* Lecture Notes for each module
* SQL examples to replicate anomalies and test locking behavior
* Diagrams explaining MVCC snapshots, WAL logs, lock schedules, and recovery phases
* A consistent example throughout the course: Movie Seat Booking

## 🚀 How to Use This Repository

* Start with Module 1 – each module builds on the previous one.
* Try the SQL snippets – run them on a local PostgreSQL instance to observe locks, waits, and anomalies.
* Refer to the diagrams inside the image/ folder whenever you need a visual explanation.
* Move through the modules in order for the best learning experience.

## 🔑 Quick Glossary

* **Atomicity:** A transaction is all-or-nothing
* **Isolation:** Transactions shouldn’t interfere with each other
* **Serializability:** The final result should look like transactions ran one-by-one
* **Deadlock:** Two transactions waiting for each other forever
* **WAL:** Log is written first, data page later
* **LSN:** ID used to order all log records
* **Dirty Page:** Modified in memory but not yet saved to disk

## 🤝 Contributions

Found a typo? Want to fix a diagram or improve an explanation?

Feel free to open a PR  -  contributions are welcome.

Just tell me!
