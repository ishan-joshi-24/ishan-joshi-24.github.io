---
layout: build
title: "An Embedding Cron That Doesn't Re-Embed What Didn't Change"
description: "Six thousand concepts, twelve directions, three phases. The whole design is about not doing work. source_id tracking, incremental processing, and the cache-bypass flag that saved Redis."
---

The first version of our typeahead embedded every concept on every cron tick. About 6,000 concepts. Every six hours. We were spending real money re-embedding strings that hadn't changed in months.

This is the rewrite.

## The three phases

```
phase 1: source-derived A (high churn)
phase 2: source-derived B (medium churn)
phase 3: curated base + composite (low churn, full regen)
```

### Why three and not one

Different change rates, different generation logic.

- Phase 1 sources change constantly. Hundreds of edits a day.
- Phase 2 sources change less. Once written, mostly stick.
- Phase 3 is a curated taxonomy. Changes only on deliberate releases. Maybe once a month.

Bundling them into one phase means either running expensive composite generation on every change (waste) or running cheap incremental work on a slow schedule (laggy). Splitting them lets each phase run on its own cadence.

### Incremental tracking via `source_id`

`source_id` is the load-bearing column. Every concept generated from a source row stores its row ID alongside the embedding. So when phase 1 runs:

```sql
SELECT id FROM sources
WHERE updated_at > $last_run
   OR id NOT IN (
     SELECT source_id FROM concept_embeddings
     WHERE source_type = 'A' AND source_id IS NOT NULL
   )
```

The "new or changed" set is tens of rows after a busy day, sometimes hundreds. We embed those. We delete embeddings whose `source_id` no longer exists in the source table.

### Phase 3 is full-regen

Base concepts are curated lists. Composite concepts are derived by combining base concepts along 12 directions. The composite set is a deterministic function over the base set. When base doesn't change, composite doesn't change, and we skip phase 3 entirely. When base does change, we don't know which composites moved without re-deriving them all, so we re-derive. Cheap because it's rare. Few minutes, costs cents.

## The batch size that took two iterations

OpenAI's embedding endpoint accepts up to 2048 inputs per call. Bigger batches are cheaper per concept.

First version, 2000 at a time, crashed Node on out-of-memory. The response is megabytes of JSON, `JSON.parse` allocates more, the TypeORM bulk insert builds another array. Peak around 1.5GB, hit v8 limit.

Second version, 500 at a time, stable. Slightly more API calls, well under the rate limit. Memory peaks around 200MB.

The lesson: the API limit and your process memory limit are different limits. Batch to whichever is tighter.

## The `skipCache` flag

Our LLM service wraps OpenAI calls with a Redis cache. For the query path that's wonderful, repeat searches hit, latency drops.

For the cron path the cache is poison. The cron embeds 6,000 strings, that's 6,000 cache writes with 1-hour TTLs. Redis fills with embedding payloads nobody will read, evicting actually-useful data.

So `generateEmbeddingsBatch` got a `skipCache` parameter:

```ts
async generateEmbeddingsBatch(
  inputs: string[],
  opts: { model?: string; skipCache?: boolean } = {},
) {
  const cached = opts.skipCache
    ? new Map()
    : await this.cache.getMany(inputs.map(this.cacheKey));

  const uncached = inputs.filter(i => !cached.has(this.cacheKey(i)));
  const fresh = await this.callOpenAIEmbed(uncached, opts.model);

  if (!opts.skipCache) await this.cache.setMany(/* ... */);

  return inputs.map(i => cached.get(this.cacheKey(i)) ?? fresh.get(i));
}
```

Cron passes `skipCache: true`. Query path doesn't. Same function, two callers, different cache semantics.

## What it looks like in production

```
[phase 1] 47 new concepts, 12 stale deleted, $0.0019
[phase 2] 8 new concepts, 0 stale, $0.0003
[phase 3] skipped (taxonomy unchanged)
total runtime: 23.4s
```

When phase 3 runs:

```
[phase 3] 5,847 concepts regenerated across 12 directions, $0.23
total runtime: 4m 12s
```

Quarter of a dollar to fully rebuild. Single-digit milli-dollars per incremental run. Bill is cents per day.

## What I'd revisit

Embedding model migration. Right now if we swap `text-embedding-3-small` for something else, we regenerate every concept and rebuild HNSW. Vectors from different models aren't comparable. The fix is tracking model-name alongside each embedding, running two embeddings during migration, falling back to the old model when the new is missing. Not built yet because we haven't needed it.
