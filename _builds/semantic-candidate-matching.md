---
layout: build
title: "Semantic Search That Survives Production"
description: "Replacing chronological lists with embedding-driven relevance ranking. pgvector, OpenAI embeddings, and the trick to keep your existing SQL filter intact."
---

The default sort for any list of people, documents, or items is `created_at DESC`. Easy to write. Useless. Recency tells you nothing about relevance.

This is what it took to replace that with a real semantic ranker, inside a system with a stubborn five-year-old SQL filter that nobody wanted to rewrite.

## The shape of the answer

```
input, embed, vector top-K per slot, interleave + dedupe, hydrate via existing SQL
```

### Embed the input, not the title

`text-embedding-3-small`. 1536 dimensions. Cheap, fast, plenty of quality for our scale.

The interesting choice is *what* you embed. A short title like "Backend Engineer" returns everyone who ever wrote "backend" or "engineer" anywhere. Per slot, the embedding text fuses title, description, related tags, and optionally a free-text bar input.

The free-text bar is the dangerous part. It steers the embedding. It does *not* reach the SQL filter. That distinction is the difference between an intuitive search bar and a prompt-injection vector into your WHERE clause.

### Top-K with HNSW

```sql
SELECT id FROM embeddings
ORDER BY embedding <=> $1::vector
LIMIT 200
```

`LIMIT 200` because after eligibility pruning we still want a screenful. HNSW because it's the right answer at hundreds of thousands of vectors with sub-10ms recall. IVFFlat builds faster but queries slower and needs cluster tuning. HNSW just works.

### Cache the slot results

Every slot result, the ranked list of 200 IDs, lands in Redis for five minutes:

```
ranked:<scope>:<fnv1a(embedding_text)>
```

FNV-1a is a non-cryptographic 64-bit hash that's fast and stable. Cryptographic hashes are overkill for "same input, same key."

### Interleave across slots

Multiple slots produce multiple ranked lists. Concatenation gives the first slot all the top positions. Score-sum union loses semantics. The right move is round-robin interleave with "strongest position wins" dedup.

```
slot A: [a1, a2, a3, ...]
slot B: [b1, a1, b2, ...]
slot C: [a2, c1, c2, ...]

interleave: a1, b1, a2, then dedup duplicates: a1, b1, a2, c1, ...
```

A candidate's final position is bounded by their best position across slots. The right semantic for "great for at least one of your inputs."

### Hydrate via the existing SQL

The interleaved list is just IDs. We still need name, photo, summary, plus the eligibility filter. The trick that kept this project tractable: the existing eligibility query already does eligibility correctly. Don't rewrite it. Add one option.

```sql
WHERE id IN (:...orderedIds)
ORDER BY FIELD(id, :oid0, :oid1, :oid2, ...)
```

`FIELD()` in MySQL returns the index of a value in a list. It preserves an externally-determined order through a SQL query. Slow if you sort a whole table by it. Fast when the IN clause already narrowed the row set to ~200 rows.

Pagination through `LIMIT/OFFSET` still works on top of FIELD ordering. Eligibility filter does eligibility. Ranker computes order, separately. They compose at the SQL layer.

## In production

- p95 end-to-end with three slots: under 350ms. Most of it is hydration, not vector search.
- Vector search per slot: 5-15ms with HNSW.
- OpenAI cost: trivially low because of the cache.
- Failure mode: OpenAI down, cache holds five minutes. Both miss, fall back to the unranked list. The user sees results, not an error.

## The one thing to take from this

Don't replace your eligibility filter. Wrap it. Compute relevance separately, feed the ordered IDs back as a filter and an order. Eligibility logic stays in one place. Ranking logic stays in one place. They compose.
