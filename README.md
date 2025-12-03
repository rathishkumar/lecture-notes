# Database Transactions, Concurrency Control and Recovery

This repository explores how databases handle **Concurrency** (multiple users) and **Recovery** (crashes). This is intended for Computer Science students and those who are interested in the database internals.

## 📚 Module Breakdown

The course is divided into two logical parts. It is recommended to follow them in order, as the recovery techniques often depend on understanding the transaction models established in the first part.

### Part 1: Concurrency Control

*How do we allow multiple users to access the database simultaneously without corrupting data?*



### Part 2: Crash Recovery

*If failure, how does the database ensure no committed data is lost?*


## 🛠️ Prerequisites

* Basic understanding of SQL (SELECT, INSERT, UPDATE).

## 🚀 Getting Started

* Start with **Module 1** to understand the vocabulary (ACID, Schedules).
* Use the provided SQL snippets in the modules to test locking behavior in a local PostgreSQL instance.
* Refer to the  folder for visual diagrams of locking schedules and log flows.

*Created for the Database Internals Guest Lecture Series.*
