---
layout: post
title: "Every Postgres Write Happens Twice. The Second One Is the Only One That Matters."
date: 2026-05-07
description: "The Write-Ahead Log was built for crash recovery. It ended up solving six other problems for free."
---

Every write Postgres makes gets written twice.

The first write is for speed. The second is the only one that actually matters.

This is the **Write-Ahead Log**. WAL.

Here's how it works.

When you commit a transaction, Postgres doesn't immediately update the data files on disk. That's slow. Instead it writes your change to the WAL first, a sequential, append-only log, and calls `fsync()`. Once that hits disk, your app gets "success."

The actual data files get updated later, asynchronously, by a background process called the checkpointer.

So at any given moment, your data files might be out of date. Postgres doesn't care. Because if the server crashes, it replays the WAL on restart and reconstructs exactly where things should be. The WAL is the source of truth. The data files are just a cache.

That's what WAL was built for. **Crash recovery.**

But one engineering decision, *write intent before action*, ended up solving a lot more:

- **Atomicity.** Uncommitted WAL records have no commit marker. Crash mid-transaction? Those records are ignored on recovery. All or nothing, for free.
- **Replication.** Replicas just stream the WAL from the primary and replay it. No full data syncs needed.
- **Point-in-time recovery.** The WAL is a complete history of every change ever made. Want your database restored to exactly 3:41pm yesterday? Replay the log to that timestamp.
- **Change data capture.** Tools like Debezium sit and read the WAL in real time. Every insert, update, delete becomes an event stream. No polling, no triggers.

One log. Seven problems solved.
