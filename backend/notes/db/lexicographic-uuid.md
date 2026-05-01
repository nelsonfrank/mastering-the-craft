## ULID (Universally Unique Lexicographically Sortable Identifier)

ULID is an alternative to UUID designed to be **sortable**, **readable**, and **unique**. It looks like this:

```
01ARZ3NDEKTSV4RRFFQ69G5FAV
```

It's 128 bits (same as UUID) but encoded as a 26-character Base32 string.

### Structure

```
 01ARZ3NDEK        TSV4RRFFQ69G5FAV
|----------|      |----------------|
 48-bit timestamp  80-bit randomness
 (milliseconds)
```

---

## Why ULID is Better Than UUID (in many cases)

### 1. **Lexicographic Sortability**

UUIDs are random and sort chaotically. ULIDs embed a millisecond-precision timestamp in the first 48 bits, so they sort chronologically by default — great for databases, logs, and event streams.

### 2. **Better Database Performance**

Random UUIDs (v4) cause **index fragmentation** in B-tree indexes (used by PostgreSQL, MySQL, etc.) because new rows insert at random positions. ULIDs insert near the end (since they're time-ordered), leading to:

- Fewer page splits
- Better cache utilization
- Faster range queries

### 3. **Human Readability**

- UUID: `550e8400-e29b-41d4-a716-446655440000` — hard to reason about
- ULID: `01ARZ3NDEKTSV4RRFFQ69G5FAV` — shorter, no hyphens, Crockford Base32 (avoids ambiguous chars like `I`, `L`, `O`, `U`)

### 4. **No Coordination Needed**

Like UUID v4, ULIDs are generated client-side with no central authority — but with the added bonus of natural ordering.

### 5. **Monotonicity Within a Millisecond**

If multiple ULIDs are generated in the same millisecond, the random component is **incremented**, preserving strict ordering even at high throughput.

---

## Quick Comparison

| Feature                | UUID v4 | ULID    |
| ---------------------- | ------- | ------- |
| Sortable               | ❌      | ✅      |
| Time-encoded           | ❌      | ✅      |
| DB index friendly      | ❌      | ✅      |
| Human readable         | ❌      | ✅      |
| No coordination needed | ✅      | ✅      |
| Widely supported       | ✅      | Growing |

---

## When to Stick With UUID

- You need **maximum compatibility** (UUIDs are natively supported in most DBs, ORMs, and APIs)
- You're using **UUID v7**, which also has timestamp ordering and largely closes the gap with ULID
- Your system already uses UUIDs and migration cost isn't worth it

---

**Bottom line:** ULID solves UUID's biggest practical pain points — random ordering and DB performance — while keeping the same collision-resistance and decentralized generation. If you're starting a new project, ULID (or UUID v7) is generally the better choice.
