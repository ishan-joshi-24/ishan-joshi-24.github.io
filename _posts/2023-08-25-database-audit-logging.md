---
layout: post
title: "Database Audit Logging: One Table vs Many, Transactions, and Trade-offs"
date: 2023-08-25
---

Audit logging seems simple: record what changed, when, and who did it. But the moment you start implementing it, you hit design decisions that matter. A lot.

Should you log to one table or many? Should the audit insert be in the same transaction as the data change? Can you afford the performance cost? These aren't questions with obvious answers.

Let me walk through the actual trade-offs.

## The Problem You're Solving

Audit logging exists for compliance, debugging, and accountability. You need to know:
- What changed
- When it changed
- Who changed it
- What the old and new values were

Simple. Until you start building it.

## Approach 1: Single Audit Table

Most people start here. One table that logs everything.

```sql
CREATE TABLE audit_log (
  id BIGINT PRIMARY KEY,
  table_name VARCHAR(255),
  record_id BIGINT,
  action VARCHAR(20), -- INSERT, UPDATE, DELETE
  old_values JSONB,
  new_values JSONB,
  changed_by VARCHAR(255),
  changed_at TIMESTAMP,
  ip_address VARCHAR(50)
);
```

Every change goes to this table.

```sql
-- User updates their email
UPDATE users SET email = 'new@example.com' WHERE id = 123;

-- Log it
INSERT INTO audit_log
  (table_name, record_id, action, old_values, new_values, changed_by, changed_at)
VALUES
  ('users', 123, 'UPDATE',
   '{"email": "old@example.com"}',
   '{"email": "new@example.com"}',
   'john', NOW());
```

### Advantages:
- Simple schema. One place for all audit data.
- Easy to query. "Show me all changes to any table" is one query.
- Consistent. Same structure everywhere.

### Disadvantages:
- Performance bottleneck. This table gets huge fast.
- Every UPDATE anywhere in your system writes here too.
- Queries are slower. You're querying a massive table with millions of rows.
- JSONB parsing. You're storing flexible data. Queries like "find all email changes" require parsing JSON.
- Hard to enforce schema. What fields should the audit log have for users vs products vs orders?

### The Query Problem

```sql
-- Find all email changes
SELECT * FROM audit_log
WHERE table_name = 'users'
  AND new_values->>'email' IS DISTINCT FROM old_values->>'email';
```

This is slow. You're scanning the entire table, parsing JSON on every row.

Compare to:

```sql
-- If you had a users_audit table with email columns
SELECT * FROM users_audit
WHERE old_email IS DISTINCT FROM new_email;
```

Much faster. No JSON parsing. Can add an index on the email columns.

## Approach 2: Separate Audit Table Per Domain Table

One audit table per domain table.

```sql
CREATE TABLE users_audit (
  id BIGINT PRIMARY KEY,
  user_id BIGINT,
  action VARCHAR(20),
  old_email VARCHAR(255),
  new_email VARCHAR(255),
  old_name VARCHAR(255),
  new_name VARCHAR(255),
  changed_by VARCHAR(255),
  changed_at TIMESTAMP,
  ip_address VARCHAR(50)
);

CREATE TABLE orders_audit (
  id BIGINT PRIMARY KEY,
  order_id BIGINT,
  action VARCHAR(20),
  old_status VARCHAR(50),
  new_status VARCHAR(50),
  old_total DECIMAL(10,2),
  new_total DECIMAL(10,2),
  changed_by VARCHAR(255),
  changed_at TIMESTAMP,
  ip_address VARCHAR(50)
);
```

### Advantages:
- Fast queries. No JSON parsing. Can index specific columns.
- Clear schema. Everyone knows what gets audited for users vs orders.
- Better performance. Smaller tables, focused queries.
- Type safety. old_email is VARCHAR, not a JSON string.

### Disadvantages:
- More schema maintenance. Need a separate table for each entity.
- Harder to answer "show me all changes across the system."
- Schema changes are painful. If you add a field to users, you also add it to users_audit.
- Code duplication. You write insert logic per table.
- Flexibility loss. JSONB can handle any change. Fixed columns can't.

### The Query Advantage

```sql
-- Find all email changes to specific users
SELECT user_id, old_email, new_email, changed_at
FROM users_audit
WHERE changed_at > NOW() - INTERVAL '7 days'
  AND old_email != new_email
ORDER BY changed_at DESC;
```

This is fast. Indexed. No JSON parsing.

## The Transaction Question

Should the audit insert be in the same transaction as the data change?

### Option A: Same Transaction

```sql
BEGIN;
  UPDATE users SET email = 'new@example.com' WHERE id = 123;
  INSERT INTO audit_log (...);
COMMIT;
```

**Advantages:**
- Guaranteed consistency. Either both succeed or both fail.
- No orphaned records. You can never have a change without an audit log.
- Simple logic. One transaction, done.

**Disadvantages:**
- Performance impact. The audit insert is in the critical path.
- The audit table becomes a bottleneck. Every data change waits for the audit log to finish.
- If the audit table is slow, your entire system is slow.
- Lock contention. The audit table lock is held longer.

### Option B: Async / Separate Transaction

```sql
BEGIN;
  UPDATE users SET email = 'new@example.com' WHERE id = 123;
COMMIT;

-- Later, asynchronously
BEGIN;
  INSERT INTO audit_log (...);
COMMIT;
```

Or queue the audit insert to be processed later.

**Advantages:**
- Fast. The data change completes before the audit insert.
- The audit table can't slow down your main application.
- You can batch inserts. Collect multiple audit entries, insert them together.

**Disadvantages:**
- Consistency risk. If the async audit fails, the data is changed but not logged.
- Complexity. You need a queue, worker process, error handling.
- Compliance issues. Some regulations require immediate audit logging.
- Debugging is harder. If something goes wrong, the audit log might not have it.

## The Practical Reality

Most people use a hybrid:

```sql
BEGIN;
  UPDATE users SET email = 'new@example.com' WHERE id = 123;
  -- Cheap insert into audit table (same transaction, but it's fast)
  INSERT INTO users_audit (...) VALUES (...);
COMMIT;
```

The audit table is optimized for fast inserts (no complex logic, no foreign keys, no expensive indexes). So it doesn't slow down the main transaction.

This gives you:
- Consistency (same transaction)
- Performance (audit table is cheap)
- Auditability (guaranteed logging)

## What Actually Gets Logged?

This is another decision nobody talks about.

### Full before/after
```json
{
  "old": { "email": "old@example.com", "name": "John", "age": 30 },
  "new": { "email": "new@example.com", "name": "John", "age": 30 }
}
```

**Pros:** Complete history. You can reconstruct any state.
**Cons:** Storage bloat. Logging a user with 50 fields means 50*2 fields in audit.

### Only changed fields
```json
{
  "email": { "old": "old@example.com", "new": "new@example.com" }
}
```

**Pros:** Compact. Only what actually changed.
**Cons:** Harder to answer "what was this record's state on 2023-08-20?"

### Just the new state
```json
{
  "email": "new@example.com",
  "name": "John",
  "age": 30
}
```

**Pros:** Simplest. Just insert the new values.
**Cons:** You've lost the old values. Can't see what changed.

## Triggers vs Application Code

You can implement this in the database (triggers) or the application.

### Triggers

```sql
CREATE TRIGGER users_audit_trigger
AFTER UPDATE ON users
FOR EACH ROW
BEGIN
  INSERT INTO users_audit (user_id, action, old_email, new_email, ...)
  VALUES (NEW.id, 'UPDATE', OLD.email, NEW.email, ...);
END;
```

**Pros:**
- Guaranteed. Every change is logged, even from scripts.
- Database-level enforcement.

**Cons:**
- Hard to test. Trigger logic is hidden.
- Doesn't know who made the change (no app context).
- Performance. Triggers run for every change.
- Debugging nightmare. Slow queries? Could be the trigger.

### Application Code

```javascript
// In your application
async function updateUserEmail(userId, newEmail) {
  await db.updateUser(userId, { email: newEmail });
  await db.auditLog('users', userId, 'UPDATE', { email: newEmail });
}
```

**Pros:**
- Visible. Easy to test and debug.
- Context available. You know who made the change, why, etc.
- Control. You decide what to log.

**Cons:**
- Inconsistent. Direct SQL queries bypass your audit logic.
- Maintenance burden. Every place that modifies data needs audit code.

## The Immutability Problem

Some systems treat audit logs as immutable. Once written, they can't be changed. This is important for compliance.

```sql
CREATE TABLE audit_log (
  ...
) WITH (fillfactor=100); -- PostgreSQL: optimize for inserts, no updates

-- No UPDATE or DELETE allowed on audit_log
REVOKE UPDATE, DELETE ON audit_log FROM app_user;
```

This prevents accidental (or malicious) modification of the audit trail.

But it also means:
- You can't fix mistakes (typos, wrong data)
- Space bloat. You can't clean up old data.
- You need separate archival strategies.

## Real-World Trade-off Example

Let's say you're building e-commerce. Orders are critical.

**Option 1: Single audit_log table**
- Pros: One place for everything
- Cons: When you log 1000 orders/minute, this table becomes a contention point
- Result: Your order processing slows down

**Option 2: Separate orders_audit table**
- Pros: Fast. Focused schema.
- Cons: More tables to manage
- Result: Order processing stays fast, audit logs are queryable

**Better Option: Separate orders_audit + Async**
```sql
BEGIN;
  UPDATE orders SET status = 'shipped' WHERE id = 123;
  -- Insert into orders_audit in the same transaction (it's cheap)
  INSERT INTO orders_audit (...) VALUES (...);
COMMIT;

-- Meanwhile, async process periodically batches and compresses old audit entries
-- Writes to archive, deletes from live table to keep it small
```

This gives you:
- Fast inserts (small, focused table)
- Consistency (same transaction)
- Performance (batched async cleanup)
- Queryability (dedicated schema)

## The Key Decisions

1. **One table or many?** One for simplicity, many for performance. Most production systems use many.

2. **Same transaction?** Yes, unless the audit table becomes a bottleneck. Make the insert cheap (minimal schema, minimal indexes).

3. **Trigger or app code?** App code for visibility and control. Triggers for guaranteed logging. Often both.

4. **What to log?** Changed fields is a good middle ground. Full before/after if storage isn't an issue.

5. **Async cleanup?** If your audit table grows unbounded, you'll have problems. Implement archival.

## The Insight

Audit logging isn't hard because of the concept. It's hard because of the trade-offs:
- Consistency vs performance
- Simplicity vs queryability
- Immutability vs space
- Guaranteed logging vs system speed

There's no "right" answer. It depends on your system's constraints. What matters is understanding the trade-offs and making intentional choices.

Most failures happen because people implement audit logging as an afterthought, using a single table for everything, in the same transaction as the data change, without thinking about performance. Then when the audit table has 100 million rows, every update slows down.

Think about audit logging from the beginning. Design for your scale.
