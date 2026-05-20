---
layout: build
title: "An AI Agent Backend With a Tiny Vocabulary"
description: "External AI agents act on a platform via a deliberately small API. The vocabulary is the security boundary. Propose-don't-commit, scoped keys, full audit log."
---

There are AI agents in our ecosystem that don't have a user account. They have an API key. They watch an inbox, decide an action is needed, call the backend, and act.

The classic case: a user emails "can we move my booking to next Tuesday?" An agent reads it, finds the booking by email, calls the reschedule endpoint with the new time. The user gets a confirmation. A human never touched it.

## The problem

Email is a terrible action surface. Free-text, easily forged, often ambiguous, and the agent has to interpret it before acting. Blast radius of a misinterpretation is real.

So the backend's job:

1. Give the agent just enough to do its job.
2. Make every action auditable.
3. Make destructive actions hard or impossible.

The actions an agent currently has:

- `GET /agents/bookings?email=...` find bookings by email
- `POST /agents/bookings/:id/reschedule` propose a new time (creates pending, doesn't commit)
- `GET /agents/users?email=...` find a user by email
- `POST /agents/notifications/acknowledge` send a templated acknowledgement

That's it. No "send free-text email", no "edit user profile", no "cancel a booking". The vocabulary is the security boundary.

## API key auth

```ts
export const agentAuthMiddleware: RequestHandler = async (req, res, next) => {
  const key = req.header('X-Agent-Key');
  if (!key) return res.status(401).json({ error: 'missing key' });

  const agent = await agentKeysDao.findByKeyHash(hashKey(key));
  if (!agent || !agent.is_active) return res.status(401).json({ error: 'invalid' });
  if (agent.expires_at && agent.expires_at < new Date()) {
    return res.status(401).json({ error: 'expired' });
  }

  req.agent = { id: agent.id, name: agent.name, scopes: agent.scopes };
  next();
};
```

We store the *hash* of the key, not the key. If the database is dumped, the keys aren't usable.

Each agent has an explicit `scopes` array. Reschedule requires `bookings:reschedule`. User lookup requires `users:read`. Rotation is changing the secret. Scope changes are a separate operation.

## Propose, don't commit

The most important design choice: the reschedule endpoint doesn't actually reschedule. It creates a `pending_reschedule` row with a 24-hour expiry and emails the owner:

> The user has requested to move their booking from Wednesday 3PM to Tuesday 2PM. [Approve] [Decline]

A human still makes the final call. The agent is a proposer, not a committer.

Why? The cost of a wrong action is high. The cost of a confirmation email is zero. Pareto-best: agent decides, human confirms.

This generalizes. *Agents should propose; humans should commit, until the agent has a track record.* When the agent's accuracy is measured over thousands of proposals, you can let it commit on its own. Until then, the human-in-the-loop is doing real work.

## Defensive checks in the endpoint

```ts
router.post('/bookings/:id/reschedule', requireScope('bookings:reschedule'),
  async (req, res) => {
    const { proposed_time, user_email, reason } = req.body;
    const booking = await bookingsDao.findById(req.params.id);

    if (!booking) return res.status(404).json({ error: 'not found' });
    if (booking.user_email !== user_email) {
      return res.status(403).json({ error: 'user mismatch' });
    }

    const pending = await pendingDao.create({
      booking_id: booking.id,
      proposed_time, reason,
      proposed_by_agent: req.agent.id,
      expires_at: addHours(new Date(), 24),
    });

    await notifyOwner(booking, pending);
    return res.json({ pending_id: pending.id, status: 'awaiting_confirmation' });
  }
);
```

- `user_email` is required as a check, not just for lookup. If it doesn't match the booking, we reject. Mismatch is a strong signal the agent has the wrong booking.
- `expires_at` is 24h. Stale proposals don't haunt the database.
- `proposed_by_agent` is logged. Every action is traceable.

## Rate limiting

Each key gets 60 req/min, 1,000 req/hr. Generous because agents are not high-frequency callers. Not infinite. A runaway agent (loop bug rescheduling the same booking a thousand times) hits the limit and stops.

Redis sliding window. 429 with `Retry-After`. The agent's queue re-tries. No data lost.

## Audit log

```sql
CREATE TABLE agent_action_log (
  id BIGSERIAL PRIMARY KEY,
  agent_id INT NOT NULL,
  action VARCHAR(64) NOT NULL,
  resource_type VARCHAR(32),
  resource_id BIGINT,
  request_payload JSONB,
  response_status INT,
  occurred_at TIMESTAMP DEFAULT NOW()
);
```

Indexed on `(agent_id, occurred_at DESC)` for "what did this agent do today" and on `(resource_type, resource_id)` for "what was done to this thing". 90-day retention.

## The dry-run mode I wish I'd added earlier

`?dry_run=true` on every mutating endpoint runs validation and returns what *would* happen without committing:

```json
{
  "would_create": {
    "type": "pending_reschedule",
    "booking_id": 12345,
    "proposed_time": "2026-04-15T14:00:00Z",
    "would_email_owner": true
  },
  "validation_passed": true
}
```

Cuts agent integration debug time roughly in half. Also lets agents preview a destructive action and confirm with the user before committing.

## What's deliberately out of scope

Things agents can't do:

- Send free-text email.
- Edit user profiles.
- Cancel bookings outright.
- Approve or reject any irreversible action.

Each has been requested. The answer is "not yet". We expand scope based on per-action acceptance rate. When proposals from a class stabilize above 90% acceptance for 30+ days, that's the data we'd want before unlocking commit.

## Two principles that made this safe to ship

1. **The vocabulary is the security boundary.** A small explicit set of endpoints, each with its own scope, beats a generic "agent can act" endpoint with internal switches. Adding a capability is a new endpoint and a new scope, which forces deliberateness.
2. **Propose, don't commit.** Until the agent has a track record, the human is in the loop.

The third thing: log everything. The audit table is the difference between "we trust this agent" and "we trust this agent because we can show you exactly what it did last Tuesday." The former is faith. The latter is engineering.
