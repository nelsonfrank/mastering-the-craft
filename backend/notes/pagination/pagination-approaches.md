# Architectures for Scalable Backend Pagination

Here are the main strategies, ranked from simplest to most scalable:

---

### 1. Offset/Limit Pagination

The most common approach. Skip N rows, return next M.

```sql
SELECT * FROM items ORDER BY id LIMIT 20 OFFSET 100;
```

**Pros:** Simple, supports random page jumps, easy to implement
**Cons:** Slow on large offsets (DB still scans skipped rows), inconsistent if data mutates (rows insert/delete shift pages)

**Best for:** Small-to-medium datasets, admin panels, simple APIs

---

### 2. Cursor-Based (Keyset) Pagination

Use a stable pointer (usually the last seen ID or timestamp) instead of an offset.

```sql
SELECT * FROM items WHERE id > :last_seen_id ORDER BY id LIMIT 20;
```

**Pros:** Consistent performance regardless of dataset size, stable under inserts/deletes, no page drift
**Cons:** No random page jumps, cursor must be passed client-side, more complex with multi-column sort

**Best for:** Feeds, timelines, high-traffic APIs (REST or GraphQL), infinite scroll

---

### 3. Page Token / Opaque Cursor

An extension of cursor-based. Encode cursor state into an opaque token (base64, JWT, or encrypted blob) returned to the client.

```json
{ "data": [...], "next_page_token": "eyJpZCI6IDEwMH0=" }
```

The server decodes the token to reconstruct the query position.

**Pros:** Hides internal DB structure, allows encoding multiple sort fields, flexible
**Cons:** Tokens can expire or become invalid, still no random access

**Best for:** Public APIs, mobile apps, any cursor-based flow where you want a clean contract

---

### 4. Seek Method (Composite Keyset)

Handles multi-column sorting by using a tuple comparison — critical when your cursor isn't unique (e.g. sorting by `created_at`).

```sql
SELECT * FROM items
WHERE (created_at, id) < (:last_created_at, :last_id)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

**Pros:** Correct and efficient with non-unique sort keys
**Cons:** DB support varies (works well in Postgres, trickier in MySQL)

**Best for:** Any cursor pagination with a non-unique sort column

---

### 5. Time-Based Windows

Partition data by fixed time buckets rather than row counts.

```sql
SELECT * FROM events
WHERE created_at BETWEEN :start AND :end
ORDER BY created_at;
```

**Pros:** Natural for time-series data, predictable query shape, good for partitioned tables
**Cons:** Uneven bucket sizes (busy periods return more rows), requires client to handle variable page sizes

**Best for:** Logs, analytics, event streams, time-series data

---

### 6. Deferred Join (Optimization pattern)

Not a pagination strategy per se, but a performance fix for slow offset queries — paginate on the index only, then join.

```sql
SELECT * FROM items
JOIN (SELECT id FROM items ORDER BY id LIMIT 20 OFFSET 10000) AS page USING (id);
```

**Pros:** Dramatically faster than naive offset on large tables (avoids fetching full rows during scan)
**Cons:** Only helps offset pagination; still has the fundamental offset problem at extreme depths

**Best for:** When you're stuck with offset pagination but need to squeeze performance

---

### Quick Decision Guide

| Need                    | Use                     |
| ----------------------- | ----------------------- |
| Simple admin UI         | Offset/Limit            |
| Infinite scroll / feed  | Cursor-based            |
| Public REST/GraphQL API | Page Token              |
| Non-unique sort column  | Seek (Composite Keyset) |
| Time-series / logs      | Time-based Windows      |
| Offset but slow queries | Deferred Join           |

The general industry direction is **cursor-based / page tokens** for anything user-facing at scale, with offset reserved for internal tooling or small datasets.
