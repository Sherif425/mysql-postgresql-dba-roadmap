I have a question about atomicity (A from ACID) , as I understood, it is consider the transaction as a whole unit like atom that couldn't be divided so either all queries in the transaction succeeded all or it failed and it rolled back even if just one query failed before committing, my question is : are all queries part of a transaction that starts with begin and end with commit ? should all developers do their queries just in a transaction ?

Your understanding of atomicity is right: a transaction is all-or-nothing. The answer to your two questions has some nuances that matter a lot for a DBA.

## 1. Is every query part of a transaction?

**Yes, always.** A database never runs a statement outside a transaction. If you don't write `BEGIN`, the database runs in **autocommit mode**: it wraps each statement in its own invisible transaction and commits it immediately.

```sql
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- with autocommit this is really:  BEGIN; UPDATE ...; COMMIT;
```

So even a single statement is atomic. An `UPDATE` that changes 1 million rows either changes all of them or none, even if the server crashes halfway through.

- **MySQL:** `autocommit=1` by default (check with `SELECT @@autocommit;`).
- **PostgreSQL:** always autocommits each statement unless you run `BEGIN`.

## 2. "If one query fails, everything rolls back": the engines differ here

This is the part most people get wrong.

| What happens when one statement inside BEGIN…COMMIT fails | MySQL (InnoDB) | PostgreSQL |
| --- | --- | --- |
| The failed statement | Rolled back | Rolled back |
| The earlier statements in the transaction | **Still pending.** The transaction stays open, and the application can still `COMMIT` them | **The whole transaction is marked as failed.** Every further command is refused until you `ROLLBACK` |
| Who decides to roll back everything | **The application** must call `ROLLBACK` | The database forces it |

Try it yourself in two Docker containers:

```sql
-- MySQL
CREATE TABLE t (id INT PRIMARY KEY);
BEGIN;
INSERT INTO t VALUES (1);
INSERT INTO t VALUES (1);   -- ERROR: duplicate key
COMMIT;                     -- succeeds! row 1 is saved
SELECT * FROM t;            -- shows 1

-- PostgreSQL
CREATE TABLE t (id INT PRIMARY KEY);
BEGIN;
INSERT INTO t VALUES (1);
INSERT INTO t VALUES (1);   -- ERROR: duplicate key
SELECT 1;                   -- ERROR: current transaction is aborted, commands ignored until end of transaction block
COMMIT;                     -- PostgreSQL answers ROLLBACK
SELECT * FROM t;            -- empty
```

So with MySQL, the application code must catch errors and call `ROLLBACK`; that's why Part 2's procedure example has an `EXIT HANDLER … ROLLBACK`. A **deadlock** is the exception: there, MySQL rolls back the whole transaction itself.

**Another difference: DDL** (`CREATE`, `ALTER`, `DROP`).
- **PostgreSQL** DDL is transactional: you can `BEGIN; ALTER TABLE …; ROLLBACK;` and the change disappears. That makes schema migrations much safer.
- **MySQL** DDL causes an **implicit commit**: running `ALTER TABLE` inside your transaction silently commits everything before it, and the `ALTER` can't be rolled back.

## 3. Should developers always use explicit transactions?

**No. Use them when several statements must succeed or fail together**, meaning when they form one business action:
- a money transfer: debit + credit;
- an order: insert the order + its order lines + reduce stock;
- a report that needs one consistent snapshot across several queries (`BEGIN` with REPEATABLE READ).

A single, independent statement doesn't need `BEGIN`; autocommit already makes it atomic.

**The DBA's rule: keep transactions as short as possible.** An open transaction holds locks and keeps old row versions alive:
- in MySQL, the undo "history list" grows and everything slows down;
- in PostgreSQL, `VACUUM` can't clean up, so tables bloat.

Never keep a transaction open while waiting for a user, a slow API call or a file upload. The worst case is a session that ran `BEGIN` and then did nothing, which PostgreSQL shows as `idle in transaction`. Many production incidents start exactly like that. That's why the roadmap mentions `idle_in_transaction_session_timeout` (PostgreSQL) and finding long transactions as a core DBA skill (sections 1.5 and 1.7).

**For your week-01 notes,** write three sentences:
1. Every statement runs in a transaction (autocommit).
2. On an error, MySQL keeps the transaction open, while PostgreSQL aborts it.
3. Why long transactions are dangerous.

That's already deeper than most "ACID" explanations.