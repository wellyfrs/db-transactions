# Database Transactions

This repository contains my notes on database transactions, focusing on the isolation levels of SQL databases.

## Definition

A transaction is a sequence of operations that must be understood as a single atomic (indivisible) operation.

The most trivial example is a financial transaction where you transfer money from your account to another account. That transaction has at least two operations: subtract the amount from your account and add the amount to the other account. It can't be a half transaction, where the amount is subtracted from your account but not added to the other account.

Formally, database transactions are defined by the [SQL-2023](docs/SQL-2023.pdf) standard as follows:

> An SQL-transaction (transaction) is a sequence of executions of SQL-statements that is atomic with respect
to recovery. That is to say: either the execution result is completely successful, or it has no effect on any
SQL-schemas or SQL-data.

The key concept here is the **atomicity** property, which is the first of the ACID properties of database transactions.

## ACID Properties

ACID is an acronym for four properties of database transactions:

- **Atomicity**: transaction operations should all complete or none of them should (all or nothing).
- **Consistency**: changes (effects) must be consistent with the rules of the database (e.g. constraints).
- **Isolation**: changes should not be visible to other transactions until they are committed (confirmed).
- **Durability**: transaction changes must be persisted once committed (confirmed), surviving system failures.

Formally, the atomicity, consistency and durability properties were first defined by Jim Gray in his paper [_The Transaction Concept: Virtues and Limitations_ (1981)](docs/transaction-concept.pdf). The first two versions of the SQL standard, SQL-86 and SQL-89, included only these three properties. The isolation property was added in the SQL-92 standard as concurrent transactions became more common.

## Isolation

When multiple transactions are executed concurrently, which is the case in many applications today, data can be read or written by more than one transaction "at the same time", which can lead to data integrity anomalies known as phenomena.

This is a classical problem in computer science, when you have shared resources that can be accessed by multiple processes, with the difference that in a database system the processes can read and write data.

A common solution for this problem is to use locks. A lock simply prevents other process from accessing a resource while a process is using it. The downside of locks is that they can cause a lot of waiting time and deadlocks (causing transactions to be aborted and hopefully retried), degrading the performance.

> A deadlock is a situation in which two or more competing actions are each waiting for the other to finish, and thus neither ever does.

Therefore, there is a trade-off between performance and data consistency.

That's why the isolation property was, by definition, relaxed with **isolation levels**, from complete isolation (full serializability) to more relaxed levels that allow some phenomena to happen. It's up to the engineer to choose the right isolation level that fits better the use case.

## Isolation levels

The SQL-92 standard defined three phenomena:

- **Dirty Read**: a transaction reads data that has been written by another transaction but not yet committed. If the other transaction rolls back, the data read by the first transaction is invalid.
- **Non-Repeatable Read** (fuzzy read): a transaction reads the same data twice but gets different results because another transaction has updated the data in between.
- **Phantom Read**: a transaction reads a set of rows that satisfy a condition, but another transaction inserts or deletes rows that satisfy the same condition. When the first transaction reads the same set of rows again, it gets a different result.

The SQL standard defines four isolation levels:

- `READ UNCOMMITTED`: the lowest isolation level, on which transactions can read uncommitted changes made by other transactions. It allows dirty reads, non-repeatable reads, and phantom reads.
- `READ COMMITTED`: transactions can only read committed data. It prevents dirty reads but allows non-repeatable reads and phantom reads.
- `REPEATABLE READ`: transactions can read the same data multiple times and get the same result. It prevents dirty reads and non-repeatable reads but allows phantom reads.
- `SERIALIZABLE`: the most strict isolation level, on which transactions are executed in a serial way. It prevents dirty reads, non-repeatable reads, and phantom reads, at the cost of performance.

The table as follows summarizes the isolation levels and the phenomena they prevent:

| Isolation Level    | Dirty Read | Non-Repeatable Read | Phantom Read |
|--------------------|------------|---------------------|--------------|
| `READ UNCOMMITTED` | Yes        | Yes                 | Yes          |
| `READ COMMITTED`   | No         | Yes                 | Yes          |
| `REPEATABLE READ`  | No         | No                  | Yes          |
| `SERIALIZABLE`     | No         | No                  | No           |

## Concurrency Control

The SQL standard doesn't specify how the isolation levels should be implemented. It's up to the database system to implement it in a way that satisfies the standard.

The **two-phase locking (2PL)** is an implementation using locks, which can guarantee serializability as demonstrated in the paper [_The Notions of Consistency and Predicate Locks in a Database System_ (1976)](docs/two-phase-locking.pdf) by C.P. Eswaran, Jim Gray (again), et al.

The **Multi-Version Concurrency Control (MVCC)** is an implementation using versioning. In this approach, the database system creates a new version of a row when it's updated. When a transaction reads a row, it reads the version of the row that was committed at the time the transaction started. This allows readers to access a consistent snapshot of the data without being blocked by writers.

MVCC still requires locks to prevent write-write conflicts, but it reduces the need for other locks. It's used by many database systems, like Oracle and PostgreSQL.

## Other Phenomena

In the paper [_A Critique of ANSI SQL Isolation Levels_ (1995)](docs/critique-of-isolation-levels.pdf), Hal Berenson, Phil Bernstein, Jim Gray (again!), et al. argue that the three phenomena defined by the SQL-92 standard were not enough to describe all the anomalies that can happen in a database system, adding the following:

- **Dirty Write**: a transaction writes data that has been written by another transaction but not yet committed. If the other transaction rolls back, the data written by the first transaction is invalid.
- **Read Skew**: a transaction reads two rows that satisfy a condition, but another transaction updates one of the rows in between. When the first transaction reads the same two rows again, it gets a different result.
- **Write Skew**: a transaction reads a row and a condition, and then updates the row based on the condition. Another transaction updates the row in between. When the first transaction updates the row, the condition is no longer satisfied.
- **Lost Update**: a transaction reads a row, updates the row, and then writes the row. Another transaction reads the row and updates the row in between. When the first transaction writes the row, the update made by the second transaction is lost.

## SQL

Most of the database systems provide a way to group operations into transactions using the `BEGIN TRANSACTION` and `COMMIT`(confirm) statements.

```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;
COMMIT;
```

The `ROLLBACK` statement is used to undo the operations of a transaction.

## Further Readings

Check the SQL-2023 standard and the referenced papers in the [docs](docs/) directory.

- [Postgres Documentation (ch. 13, _Concurrency Control_)](https://www.postgresql.org/docs/current/mvcc.html)
- _Designing Data-Intensive Applications_ by Martin Kleppmann (ch. 7, _Transactions_)
- _High-Performance Java Persistence_ by Vlad Mihalcea (ch. 6, _Transactions_)