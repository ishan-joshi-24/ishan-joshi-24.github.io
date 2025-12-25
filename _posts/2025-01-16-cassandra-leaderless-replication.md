---
layout: post
title: "Understanding Leaderless Replication in Cassandra"
date: 2023-06-10
---

Most databases have a leader. One node decides what the truth is, replicates to followers, and everyone else reads that gospel. It's simple. It works. But it has a single point of failure.

Cassandra said "what if no one is the leader?" and built a system where every node is equal. No coordinator. No single source of truth. Just nodes talking to each other, each one deciding independently if they believe you.

This sounds chaotic. Let me explain how it actually works.

## The Problem With Leaders

Let's say you have a database with a leader and two followers.

```
Write comes in → Leader processes it → Sends to Follower A and B
```

What if the leader crashes before it replicates to the followers? The followers don't have the data. You either lose that write, or you promote a follower to leader and hope it had the data.

What if the leader is slow? Everything backs up. It's a bottleneck.

What if a network partition happens and the leader can't reach the followers? The followers think the leader is dead and promote one of themselves. Now you have two leaders and they disagree about what the data is.

Cassandra asks: "What if we just didn't have a leader?"

## The Leaderless Model

In Cassandra, every node can accept writes. There is no master. When you write to Cassandra, you pick a node (or Cassandra picks one for you). That node becomes the "coordinator" for this request, but it's not a permanent leader. It's just the node you happened to talk to.

```
Client writes to Node A
Node A is the coordinator for this request
Node A writes to itself
Node A writes to Node B and Node C (the replicas)
Client gets ack when enough replicas confirm
```

Each node has the same authority. No one is special.

## The Key Insight: Quorum Writes

Here's where it gets interesting. Cassandra doesn't wait for all replicas to confirm a write. Instead, it uses quorums.

Let's say you have a replication factor of 3 (three copies of the data).

**Write quorum: 2 out of 3 must acknowledge**
```
Write comes in → Coordinator sends to 3 nodes → 2 respond → Ack to client
```

Even if one node is slow or down, the write is considered successful.

**Read quorum: 2 out of 3 must respond**
```
Read comes in → Coordinator asks 3 nodes → 2 respond → Return data
```

The trick: if you write to 2 nodes and read from 2 nodes, at least one of those read nodes must have seen your write. They overlap.

This is called the "quorum intersection property." With W writes and R reads, if W + R > N (replication factor), you're guaranteed to see your own writes.

Common configurations:
- W=1, R=3: Write to 1, read from all 3. Fast writes, slow reads.
- W=2, R=2: Write to 2, read from 2. Balanced.
- W=3, R=1: Write to all 3, read from 1. Slow writes, fast reads.

## What If There Are Conflicts?

Since multiple nodes can accept writes independently, what happens if two clients write different values to the same key at the same time?

```
Client A writes value "apple" to Node A at 10:00:00.001
Client B writes value "banana" to Node B at 10:00:00.002
```

Both nodes accept their writes. Both send replicas to other nodes. Now different nodes have different values for the same key.

This is a conflict, and Cassandra can't know which one is "correct" without human input. So it uses a tiebreaker: **timestamps**.

Each write includes a timestamp (usually the client's clock, but could be server time). When Cassandra encounters two values for the same key, it keeps the one with the higher timestamp and discards the other. This is called "last-write-wins."

```
Node A has: apple (timestamp: 10:00:00.001)
Node B has: banana (timestamp: 10:00:00.002)
When they sync up, banana wins because 10:00:00.002 > 10:00:00.001
```

Is this correct? Maybe not. But it's deterministic, and all nodes will eventually agree on the same value.

## Eventual Consistency

After writes succeed, the replicas might not have the data immediately. But eventually, through gossip (nodes talking to each other), everyone gets the data. This is eventual consistency.

```
Time 0: Write succeeds on Node A and B
Time 0: Client gets ack
Time 0.001: Node C still doesn't have the data
Time 0.05: Node C gets the data through gossip
```

If a client reads from Node C at time 0.01 (before gossip), they might get a stale value or no value at all.

This is fundamentally different from leader-based systems where once the leader confirms a write, it's safe to read from any node.

## How Cassandra Keeps Things Consistent Anyway

Cassandra has mechanisms to fix inconsistencies after they happen:

### 1. Read Repair

When you read from a quorum, Cassandra checks if all nodes have the same value.

```
Read request to Node A, B, C
Node A: apple (timestamp 100)
Node B: apple (timestamp 100)
Node C: banana (timestamp 80) ← outdated
Return "apple" to client
In the background, fix Node C to also have apple
```

Cassandra discovers and fixes inconsistencies on reads.

### 2. Anti-Entropy / Merkle Trees

Nodes periodically compare data structures (Merkle trees) to find differences.

```
Node A tree: [data hashes]
Node B tree: [data hashes]
Compare → Find Node B is missing some data
Node B requests the missing data from Node A
```

This catches inconsistencies even if they're never read.

### 3. Hinted Handoff

If a write fails to reach a replica (the node is down), the coordinator keeps the write locally with a note: "This is for Node C."

Later, when Node C comes back online, the coordinator sends it the write: "Here's the data you missed."

## Seeing It In Action

Here's what actually happens when you write to Cassandra:

```
Client sends: INSERT INTO users (id, name) VALUES (1, 'Alice')
to any node (let's say Node A)

Node A becomes the coordinator for this request
Node A calculates which nodes should have this data
(based on the partition key and replication factor)
Let's say that's Node A, B, C

Node A sends the write to A, B, C in parallel
Node A: write confirmed at time 0ms
Node B: write confirmed at time 5ms
Node C: write confirmed at time 50ms (slow network)

Cassandra waits for 2 acks (quorum W=2)
At time 5ms, both A and B have confirmed
Cassandra returns success to the client

Node C is still processing the write
But the client doesn't care. It's been committed.
```

Now if a client reads:

```
Client asks: SELECT name FROM users WHERE id = 1

Node A is chosen as coordinator
Coordinator asks Node A, B, C
Node A: 'Alice'
Node B: 'Alice'
Node C: [no response yet, still syncing]

Quorum read (R=2): Got 2 responses
Return 'Alice' to client

In the background:
Send 'Alice' to Node C to repair the inconsistency
```

## The Trade-offs

**Advantages of leaderless replication:**
- No single point of failure. Kill any node, the system keeps working.
- No bottleneck at the leader. All nodes can accept writes in parallel.
- Scales better. Distribute load across all nodes.
- High availability. You can tolerate multiple node failures.

**Disadvantages:**
- Eventual consistency. Reads might return stale data.
- Complex conflict resolution. Last-write-wins might not be what you want.
- Tuning required. You have to pick the right W, R, N values.
- Harder to reason about. Multiple nodes means more edge cases.
- Higher latency for reads. You wait for a quorum instead of just the fastest node.

## Quorum Math

This is the critical part to understand. The quorum intersection property guarantees consistency:

If you write to W nodes and read from R nodes, you see your own writes if:

**W + R > N**

Where N is the replication factor.

Examples:
- N=3, W=2, R=2: 2 + 2 > 3 ✓ Consistent
- N=3, W=1, R=1: 1 + 1 = 2, not > 3 ✗ Not consistent (might miss writes)
- N=3, W=3, R=1: 3 + 1 > 3 ✓ Consistent
- N=3, W=1, R=3: 1 + 3 > 3 ✓ Consistent

If the condition is violated, you can get stale reads. Cassandra will write anyway (because you told it to use W=1), but you might read old data even after writing.

## Why This Matters

Leaderless replication is why Cassandra can be deployed globally. You can have:
- 3 nodes in US
- 3 nodes in EU
- 3 nodes in Asia
- Total replication factor: 9

A write succeeds after 2 acks (could be 2 US nodes). The EU and Asia nodes sync up eventually. No single leader to bottleneck across continents.

Compare this to a leader-based system: the leader is in one location, every write must go through it, and everyone else waits.

## The Catch: Conflicts Are Your Problem

Here's what you need to remember: Cassandra doesn't prevent conflicts. It just handles them consistently.

```
Client A in New York writes: status = "online"
Client B in Tokyo writes: status = "offline"
Both hit different nodes at the same time

The timestamps are:
New York client: 10:00:00.001
Tokyo client: 10:00:00.002

Cassandra will pick "offline" because it has the higher timestamp
New York client thinks they set it to "online" but it ended up "offline"
```

This is called a "lost write" and it's a trade-off you accept for the availability benefits.

Some applications use application-level conflict resolution (like CRDTs - conflict-free replicated data types) or distributed consensus if they absolutely need linearizability.

## The Key Insight

Leaderless replication trades consistency for availability. You get:
- No leader = no bottleneck
- Multiple nodes accepting writes = high availability
- But: eventual consistency, conflicts resolved by timestamp, complex to reason about

It's why Cassandra is used for massive, distributed systems where availability matters more than immediate consistency. It's also why you shouldn't use it for your bank account.

The beauty of it is that it's all deterministic. Every node will eventually agree on the state because they follow the same rules: last-write-wins, quorum reads, gossip protocol. It's chaos that converges to order.
