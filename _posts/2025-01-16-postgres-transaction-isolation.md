---
layout: post
title: "Understanding PostgreSQL Transaction Isolation Levels"
date: 2023-02-15
---

Transaction isolation is one of those things that sounds simple until you try to build a concurrent system. Then it becomes the thing that keeps you up at night wondering why your database is doing something weird.

Let me walk you through how PostgreSQL handles this, starting from the problem, then the solutions, and finally how to actually use them.

## The Problem: Concurrent Transactions

Imagine two customers trying to withdraw money from the same account at the same time.

```
Account balance: $1000

Transaction A: Read balance (1000), deduct 500, write 500
Transaction B: Read balance (1000), deduct 300, write 700
```

Both transactions read the original balance of 1000. A writes 500, then B writes 700. The final balance is 700, but we've lost 500. This is called a "lost update" and it's bad.

The database needs to control how much transactions can see and interfere with each other. This is where isolation levels come in.

## The Four Isolation Levels (In Theory)

There are four standard SQL isolation levels. Think of them as a spectrum from "chaos" to "ultra-safe":

### 1. READ UNCOMMITTED
Transactions can read data that hasn't been committed yet. This is the danger zone.

**Problem: Dirty reads**
```
Transaction A: Writes balance = 500 (not committed)
Transaction B: Reads balance = 500 (reads uncommitted data)
Transaction A: Rolls back, balance is actually 1000
Transaction B: Just read data that never existed
```

This is so risky that PostgreSQL actually doesn't support it. If you ask for READ UNCOMMITTED, PostgreSQL gives you READ COMMITTED instead. It's like asking for a skydive without a parachute and the company just gives you a parachute anyway.

### 2. READ COMMITTED (PostgreSQL default)
Transactions can only read data that's been committed. Once you commit, your changes are visible to everyone.

**Problem: Non-repeatable reads**
```
Transaction A: Read balance = 1000
Transaction B: Changes balance to 500, commits
Transaction A: Read balance again = 500 (different value!)
```

The same query inside a transaction returns different results. The data itself is committed and real, but from your transaction's perspective, the world changed underneath you.

**Problem: Phantom reads**
```
Transaction A: SELECT COUNT(*) WHERE age > 18 → 100
Transaction B: INSERT new person aged 25, commits
Transaction A: SELECT COUNT(*) WHERE age > 18 → 101 (phantom row appeared)
```

New data matching your criteria appears mid-transaction.

### 3. REPEATABLE READ
Once you read something, you'll keep seeing the same version of it for the rest of your transaction, even if someone else changes it.

**How it works:**
- PostgreSQL takes a snapshot of the database at the start of your transaction
- All queries see that snapshot
- Changes made by other transactions after your snapshot started? You can't see them
- You won't see non-repeatable reads or phantoms

**Problem: Still allows some weirdness**
You can still get into situations where the order of operations matters and causes unexpected results.

### 4. SERIALIZABLE
This is the "pretend transactions happen one at a time" level.

PostgreSQL uses something called "Serialization Anomaly Detection" to make this work. It doesn't actually run transactions sequentially (that would be slow). Instead, it detects if the results would be different than if they ran sequentially. If so, it aborts one transaction and tells you to retry.

## How PostgreSQL Actually Implements This

PostgreSQL uses MVCC (Multi-Version Concurrency Control). Instead of locking everything, it keeps multiple versions of data around.

When you start a transaction, PostgreSQL:
1. Records your transaction ID
2. Takes a snapshot of which transactions were active when you started
3. You can only see changes made by transactions that committed before you started
4. You cannot see changes made by transactions that started after you

This is why REPEATABLE READ works naturally in PostgreSQL. You're literally seeing a snapshot from a point in time.

## Seeing It In Action

Let's look at actual examples:

### READ COMMITTED Example

Terminal 1 (Transaction A):
```sql
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT balance FROM accounts WHERE id = 1;
-- balance: 1000
-- (wait for Terminal 2)
SELECT balance FROM accounts WHERE id = 1;
-- balance: 500 (changed!)
COMMIT;
```

Terminal 2 (Transaction B):
```sql
BEGIN;
UPDATE accounts SET balance = 500 WHERE id = 1;
COMMIT;
```

When B commits, A can now see the change. Non-repeatable read.

### REPEATABLE READ Example

Terminal 1:
```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1;
-- balance: 1000
-- (wait for Terminal 2)
SELECT balance FROM accounts WHERE id = 1;
-- balance: 1000 (still the same!)
COMMIT;
```

Terminal 2:
```sql
BEGIN;
UPDATE accounts SET balance = 500 WHERE id = 1;
COMMIT;
```

A never sees the change. It's looking at the snapshot from when it started.

### SERIALIZABLE Example

Terminal 1:
```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT balance FROM accounts WHERE id = 1;
-- balance: 1000
-- (wait for Terminal 2)
SELECT balance FROM accounts WHERE id = 1;
-- balance: 1000
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
-- SERIALIZATION FAILURE
COMMIT;
-- ERROR: could not serialize access due to concurrent update
```

Terminal 2:
```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
UPDATE accounts SET balance = balance - 300 WHERE id = 1;
COMMIT;
-- This succeeds
```

PostgreSQL detected that A and B's transactions conflict. They can't both execute in any serial order without one seeing data the other wrote. So it aborts A.

## When Do You Use Each Level?

**READ COMMITTED (the default):**
- Most web applications
- You're okay with seeing committed data
- You handle non-repeatable reads in your app logic
- Fast, allows maximum concurrency

**REPEATABLE READ:**
- You're doing multiple reads and need consistency
- Complex calculations that span multiple queries
- You want a consistent snapshot
- Good performance in most cases

**SERIALIZABLE:**
- Critical financial operations
- When you absolutely cannot have any anomalies
- Complex multi-step transactions
- You're willing to handle retries
- Slower because PostgreSQL has to do anomaly detection

## The Performance Cost

Higher isolation = more work for the database.

- READ COMMITTED: Fast, minimal overhead
- REPEATABLE READ: Slightly slower, maintains snapshot
- SERIALIZABLE: Slowest, has to detect conflicts

But "slower" is relative. SERIALIZABLE is still fast on modern hardware. Use it when correctness matters more than raw speed.

## Real World Mistake

Here's something I see often:

```sql
-- Developer thinks this is atomic
SELECT balance FROM accounts WHERE id = 1; -- 1000
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
```

If this is in READ COMMITTED (the default), between the SELECT and UPDATE, another transaction could change the balance. The SELECT reads 1000, but then someone else decreases it to 500, and your UPDATE still works on the original 1000 value you read, leading to lost updates.

**Better:**
```sql
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
```

Just do it in one statement. Or use REPEATABLE READ if you need multiple statements.

## The Key Insight

Transaction isolation is about controlling visibility and conflicts. PostgreSQL's MVCC is clever because it doesn't lock data. Instead, it keeps old versions around and lets readers see old data while writers create new versions. This allows reads and writes to happen concurrently without stepping on each other.

The isolation level controls: "How old is the data I can see?" and "When do I fail because of conflicts?"

- READ COMMITTED: See all committed data (might change within transaction)
- REPEATABLE READ: See snapshot from transaction start (won't change within transaction)
- SERIALIZABLE: Pretend you're alone (or fail if you're not)

That's the whole game.
