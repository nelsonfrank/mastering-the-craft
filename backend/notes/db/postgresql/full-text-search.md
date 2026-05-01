# Full-Text Search (FTS)

_Built-in_[cite: 1]

PostgreSQL's native text-search engine using `tsvector` and `tsquery`.[cite: 1]

FTS works by reducing documents to a sorted list of normalized _lexemes_ (dictionary stems), which are then matched against a parsed query.[cite: 1] It understands natural language: "running", "runs", and "ran" all map to the lexeme `run`.[cite: 1]

## Core types

```sql
-- tsvector: a document representation
SELECT to_tsvector('english', 'The quick brown fox jumped over the lazy dog');
-- Result: 'brown':3 'dog':9 'fox':4 'jump':5 'lazi':8 'quick':2
-- Stop words ('the','over') are removed; words are stemmed

-- tsquery: a search query
SELECT to_tsquery('english', 'jumping & fox');
-- Result: 'jump' & 'fox' (both stemmed)

-- plainto_tsquery: converts plain text (no operators needed)
SELECT plainto_tsquery('english', 'quick brown fox');
-- Result: 'quick' & 'brown' & 'fox'

-- phraseto_tsquery: preserves word order
SELECT phraseto_tsquery('english', 'brown fox');
-- Result: 'brown' <-> 'fox' (must be adjacent)
```

## Matching with @@

```sql
SELECT title, body
FROM articles
WHERE to_tsvector('english', title || ' ' || body) @@ to_tsquery('english', 'postgres & search');
```

## tsquery operators

| Operator | Meaning                                 | Example                        |
| :------- | :-------------------------------------- | :----------------------------- | ------ | --------------- |
| `&`      | AND — both must match[cite: 1]          | `'cat' & 'dog'`[cite: 1]       |
| `        | `                                       | OR — either matches[cite: 1]   | `'cat' | 'dog'`[cite: 1] |
| `!`      | NOT — must not match[cite: 1]           | `'cat' & !dog'`[cite: 1]       |
| `<->`    | FOLLOWED BY (adjacent)[cite: 1]         | `'quick' <-> 'brown'`[cite: 1] |
| `<N>`    | FOLLOWED BY within N positions[cite: 1] | `'quick' <2> 'fox'`[cite: 1]   |

## GIN index (required for performance)

```sql
-- Index on the column directly if using a stored tsvector
ALTER TABLE articles ADD COLUMN search_vector tsvector;

UPDATE articles
SET search_vector = to_tsvector('english', coalesce(title,'') || ' ' || coalesce(body,''));

CREATE INDEX idx_articles_fts ON articles USING gin(search_vector);

-- Keep it auto-updated with a trigger
CREATE FUNCTION articles_search_trigger() RETURNS trigger AS $$
BEGIN
  NEW.search_vector := to_tsvector('english', coalesce(NEW.title,'') || ' ' || coalesce(NEW.body,''));
  RETURN NEW;
END; $$ LANGUAGE plpgsql;

CREATE TRIGGER tsvectorupdate
BEFORE INSERT OR UPDATE ON articles
FOR EACH ROW EXECUTE FUNCTION articles_search_trigger();
```

## Supported languages

```sql
SELECT cfgname FROM pg_ts_config;

-- arabic, danish, dutch, english, finnish, french, german,
-- hungarian, indonesian, irish, italian, lithuanian,
-- nepali, norwegian, portuguese, romanian, russian,
-- spanish, swedish, tamil, turkish...
```

- **✓ Advantages:** Native, no extensions.[cite: 1] Understands stemming and stop words.[cite: 1] Language-aware.[cite: 1] Supports phrase and proximity search.[cite: 1] Fast with GIN index.[cite: 1] Free ranking via `ts_rank()`.[cite: 1]
- **✗ Disadvantages:** No typo tolerance.[cite: 1] Exact lexeme matching only.[cite: 1] No partial/prefix matching within FTS itself.[cite: 1] Requires language config to be chosen upfront.[cite: 1] Can miss synonyms without dictionary setup.[cite: 1]

---

# Fuzzy Search

_Extension: pg_trgm_[cite: 1]

Find results even when users make typos, using trigram similarity.[cite: 1]

A _trigram_ is a group of 3 consecutive characters.[cite: 1] "hello" becomes: ` h`, ` he`, `hel`, `ell`, `llo`, `lo `, `o `.[cite: 1] Similarity is the ratio of shared trigrams to total unique trigrams.[cite: 1] Two strings are "similar" if their similarity exceeds a threshold.[cite: 1]

## Setup

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- GIN index for fast trigram search (preferred)
CREATE INDEX idx_products_name_trgm ON products
USING gin(name gin_trgm_ops);

-- GiST alternative (slower to query, faster to update)
CREATE INDEX idx_products_name_gist ON products
USING gist(name gist_trgm_ops);
```

## Similarity functions

```sql
-- similarity(): 0.0 (no match) to 1.0 (identical)
SELECT similarity('postgresql', 'postgresqll'); -- ~0.7
SELECT similarity('cat', 'dog'); -- ~0.0

-- % operator: true if similarity >= pg_trgm.similarity_threshold (default 0.3)
SELECT name FROM products WHERE name % 'postgresl';

-- word_similarity(): partial match — query vs any word in string
SELECT word_similarity('lex', 'Alex Rodriguez'); -- higher than similarity()

-- strict_word_similarity(): even stricter partial
SELECT strict_word_similarity('alex', 'alex rodriguez');
```

## Tuning the threshold

```sql
-- Set per-session (0.0–1.0, lower = more permissive)
SET pg_trgm.similarity_threshold = 0.4;

-- Or per-query with a score filter
SELECT name, similarity(name, 'postgresl') AS score
FROM products
WHERE similarity(name, 'postgresl') > 0.3
ORDER BY score DESC
LIMIT 10;
```

## Levenshtein distance (fuzzystrmatch)

```sql
CREATE EXTENSION IF NOT EXISTS fuzzystrmatch;

-- levenshtein: number of edits to turn one string into another
SELECT levenshtein('kitten', 'sitting'); -- 3

-- Useful for short strings; does NOT use indexes
SELECT name FROM users
WHERE levenshtein(name, 'Jhon') <= 2;
```

> **⚠ Threshold tuning matters.** Too low (e.g., 0.1) and everything matches; too high (0.8) and common typos are missed.[cite: 1] For product names, 0.35–0.45 is a good starting point.[cite: 1] For short strings (<4 chars), trigrams become unreliable — Levenshtein is better.[cite: 1]

- **✓ Advantages:** Handles typos and misspellings naturally.[cite: 1] Also enables fast ILIKE/LIKE and regex queries via the index.[cite: 1] Works on any string — no language config needed.[cite: 1]
- **✗ Disadvantages:** Short strings (<3 chars) have no trigrams — won't index.[cite: 1] Similarity is character-based, not semantic.[cite: 1] Can produce confusing false-positives.[cite: 1] Larger index size than FTS.[cite: 1]

---

# Autocomplete / Prefix Search

_Multiple Approaches_[cite: 1]

Return results as the user types, matching from the beginning of a string.[cite: 1]

## Method 1: LIKE with B-tree index (fastest, simplest)

```sql
-- Works only with text_pattern_ops / varchar_pattern_ops
CREATE INDEX idx_users_name_prefix ON users (name text_pattern_ops);

-- Query: prefix match
SELECT name FROM users
WHERE name LIKE 'Joh%'
LIMIT 10;

-- Or using >= / < range trick (locale-independent, very fast)
SELECT name FROM users
WHERE name >= 'Joh' AND name < 'Joi' -- increment last char
LIMIT 10;
```

## Method 2: Trigram prefix with pg_trgm

```sql
-- The GIN trgm index also accelerates LIKE '%...%' patterns
CREATE INDEX idx_articles_title_trgm ON articles USING gin(title gin_trgm_ops);

-- Prefix: faster with B-tree; trigram: better for mid-string
SELECT title FROM articles WHERE title ILIKE 'quick%';
```

## Method 3: FTS prefix with websearch_to_tsquery

```sql
-- websearch_to_tsquery supports prefix with :*
SELECT title
FROM articles
WHERE search_vector @@ to_tsquery('english', 'quick:\*');
-- Matches: quick, quickly, quickest, quickness...

-- User input "qu" → build prefix query dynamically
SELECT title
FROM articles
WHERE search_vector @@ to_tsquery('english', 'qu:\*');
```

## Method 4: Materialized lookup table

```sql
-- Pre-compute popular completions for very fast autocomplete
CREATE TABLE search_suggestions (
term text PRIMARY KEY,
frequency int DEFAULT 1
);

CREATE INDEX idx_suggestions_prefix ON search_suggestions (term text_pattern_ops);

-- Query
SELECT term
FROM search_suggestions
WHERE term LIKE 'joh%'
ORDER BY frequency DESC
LIMIT 8;
```

> **✓ Best practice:** For a search box, combine prefix LIKE (for speed) with trigram similarity (for typo tolerance) and rank results by a popularity score.[cite: 1]

- **✓ Advantages:** B-tree prefix is extremely fast (<1ms with index).[cite: 1] FTS prefix `:*` is language-aware.[cite: 1] No extra extension for the basic case.[cite: 1]
- **✗ Disadvantages:** Pure prefix can't match mid-string ("ohn" won't find "John").[cite: 1] FTS prefix doesn't handle typos.[cite: 1] For complex autocomplete, external tools (Redis, Elasticsearch) may be faster.[cite: 1]

---

# search_vector Column

_Generated Column Pattern_[cite: 1]

The recommended pattern for production FTS: store a pre-computed `tsvector` column, index it, and keep it current with a trigger.[cite: 1]

Instead of calling `to_tsvector()` on every query (slow, blocks index use), you compute it once on write and store it in a dedicated column.[cite: 1] This is the _search_vector pattern_.[cite: 1]

## Full multi-column weighted setup

```sql
-- 1. Add the column
ALTER TABLE articles
ADD COLUMN search_vector tsvector
GENERATED ALWAYS AS (
setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
setweight(to_tsvector('english', coalesce(subtitle, '')), 'B') ||
setweight(to_tsvector('english', coalesce(body, '')), 'C')
) STORED;

-- 2. Index it
CREATE INDEX idx_articles_search ON articles USING gin(search_vector);

-- 3. Query it
SELECT title, ts_rank(search_vector, query) AS rank
FROM articles,
to_tsquery('english', 'machine & learning') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
```

> **GENERATED ALWAYS AS ... STORED** (PostgreSQL 12+) is the cleanest option — no trigger needed, PostgreSQL handles updates automatically.[cite: 1] For older versions, use a `BEFORE INSERT OR UPDATE` trigger.[cite: 1]

## Trigger-based approach (PG <12 or manual control)

```sql
CREATE OR REPLACE FUNCTION update_search_vector()
RETURNS trigger AS $$
BEGIN
NEW.search_vector :=
setweight(to_tsvector('english', coalesce(NEW.title, '')), 'A') ||
setweight(to_tsvector('english', coalesce(NEW.tags::text, '')), 'B') ||
setweight(to_tsvector('english', coalesce(NEW.body, '')), 'C');
RETURN NEW;
END;
$$
LANGUAGE plpgsql;

CREATE TRIGGER articles_search_vector_update
  BEFORE INSERT OR UPDATE OF title, tags, body
  ON articles
  FOR EACH ROW EXECUTE FUNCTION update_search_vector();
```

## Weight reference

| Weight | Default ts_rank multiplier | Typical use                     |
| :----- | :------------------------- | :------------------------------ |
| `'A'`  | 1.0[cite: 1]               | Title, headline[cite: 1]        |
| `'B'`  | 0.4[cite: 1]               | Subtitle, tags, author[cite: 1] |
| `'C'`  | 0.2[cite: 1]               | Summary, description[cite: 1]   |
| `'D'`  | 0.1[cite: 1]               | Full body text[cite: 1]         |

---

# Synonyms & Custom Dictionaries

_Text Search Configuration_[cite: 1]

Teach PostgreSQL that "car" = "automobile", or that "PostgreSQL" should never be stemmed.[cite: 1]

## The text search configuration stack

```sql
-- Inspect the default config
SELECT * FROM ts_debug('english', 'The cars were running fast');
-- alias | description | token | dictionaries | dictionary | lexemes
-- asciiword | ... | cars  | {english_stem} | english_stem | {car}
-- asciiword | ... | fast  | {english_stem} | english_stem | {fast}
```

## Simple synonym dictionary

```sql
-- 1. Create a .syn file at $SHAREDIR/tsearch_data/mysynonyms.syn
--    Format: one mapping per line
--    automobile car
--    vehicle car
--    auto car
--    PostgreSQL postgres

-- 2. Register the dictionary in PostgreSQL
CREATE TEXT SEARCH DICTIONARY my_synonyms (
  TEMPLATE = synonym,
  SYNONYMS = mysynonyms
);

-- 3. Create a custom config based on english
CREATE TEXT SEARCH CONFIGURATION my_english (
  COPY = english
);

-- 4. Use our synonym dict first, then fall through to stemmer
ALTER TEXT SEARCH CONFIGURATION my_english
  ALTER MAPPING FOR asciiword
  WITH my_synonyms, english_stem;

-- 5. Test it
SELECT to_tsvector('my_english', 'automobile');  -- returns 'car'
SELECT to_tsvector('my_english', 'vehicle');     -- returns 'car'
SELECT to_tsvector('my_english', 'car');         -- returns 'car'
```

## Thesaurus dictionary (phrase synonyms)

```sql
-- Thesaurus handles multi-word synonyms
-- File: $SHAREDIR/tsearch_data/mythesaurus.ths
--   Format:  phrase : replacement  (left=input, right=output)
--   machine learning : ml
--   artificial intelligence : ai
--   deep learning : dl ml

CREATE TEXT SEARCH DICTIONARY my_thesaurus (
  TEMPLATE = thesaurus,
  DICTFILE = mythesaurus,
  DICTIONARY = english_stem    -- fallback dict
);

ALTER TEXT SEARCH CONFIGURATION my_english
  ALTER MAPPING FOR asciiword, hword
  WITH my_thesaurus, my_synonyms, english_stem;

-- Now "machine learning" → 'ml' lexeme
SELECT to_tsvector('my_english', 'machine learning algorithms');
```

## Stop words

```sql
-- View current stop words for English
SELECT * FROM ts_debug('english', 'the and a an');
-- These are dropped (stop words = no lexemes produced)

-- Custom stop words: edit $SHAREDIR/tsearch_data/english.stop
-- Add domain-specific noise words like your company name
```

> **⚠ Reload required.** After changing `.syn`, `.ths`, or `.stop` files on disk, run `SELECT ts_rewrite(...)` or restart the session.[cite: 1] The dictionaries are cached per-backend — a full server reload may be needed for all connections to pick up changes.[cite: 1]

---

# Scoring & Ranking

_Relevance Ranking_[cite: 1]

Sort results by relevance, not just whether they match.[cite: 1]

## ts_rank() — term frequency

```sql
-- Basic: how often does the query appear in the document?
SELECT
  title,
  ts_rank(search_vector, query) AS rank
FROM articles, to_tsquery('english', 'postgres') query
WHERE search_vector @@ query
ORDER BY rank DESC;

-- normalization: controls how document length affects score
-- 0 (default): ignore length
-- 1: divide by 1 + log(length)
-- 2: divide by length
-- 4: divide by mean harmonic distance
-- 8: divide by number of unique words
-- 16: divide by 1 + log(unique words)
-- 32: divide by rank + 1
-- You can combine: 1|2 = divide by length AND log(length)

SELECT title, ts_rank(search_vector, query, 1) AS rank
FROM articles, to_tsquery('english', 'postgres') query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

## ts_rank_cd() — cover density

```sql
-- Considers how close query terms are to each other
-- Better for multi-word queries
SELECT
  title,
  ts_rank_cd(search_vector, query) AS rank
FROM articles, to_tsquery('english', 'machine & learning') query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

## Boosting with business signals

```sql
-- Combine FTS rank with popularity, recency, views, etc.
SELECT
  title,
  ts_rank_cd(search_vector, query) * LOG(1 + view_count) AS boosted_rank
FROM articles, to_tsquery('english', 'postgres') query
WHERE search_vector @@ query
ORDER BY boosted_rank DESC
LIMIT 20;

-- Recency decay: documents from the last 30 days score higher
SELECT
  title,
  ts_rank(search_vector, query) *
    EXP(-0.05 * (EXTRACT(EPOCH FROM (now() - created_at)) / 86400)) AS rank
FROM articles, to_tsquery('english', 'search') query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

## Highlighting matches

```sql
-- ts_headline: wrap matched terms in tags for display
SELECT
  ts_headline(
    'english',
    body,
    to_tsquery('english', 'postgres & search'),
    'MaxWords=35, MinWords=15, MaxFragments=3,
     StartSel=<mark>, StopSel=</mark>'
  ) AS snippet
FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgres & search');
```

> **✓ Tip:** `ts_headline()` is expensive — call it only after filtering.[cite: 1] Never call it without a `WHERE` clause first.[cite: 1]

---

# Combining Approaches

_Production Pattern_[cite: 1]

Real-world search combines FTS + fuzzy + prefix for a seamless user experience.[cite: 1]

## Setup: unified indexes

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS unaccent;

ALTER TABLE products
  ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(brand, '')), 'B') ||
    setweight(to_tsvector('english', coalesce(description, '')), 'D')
  ) STORED;

CREATE INDEX idx_products_fts   ON products USING gin(search_vector);
CREATE INDEX idx_products_trgm  ON products USING gin(name gin_trgm_ops);
CREATE INDEX idx_products_prefix ON products (name text_pattern_ops);
```

## The universal search query

```sql
WITH query AS (
  SELECT
    plainto_tsquery('english', 'wireless headphone')    AS fts_q,
    'wireless headphone'                                AS raw,
    'wireless:* & headphone:*'                          AS prefix_q
),
results AS (
  SELECT
    p.id, p.name, p.brand, p.price,
    -- FTS relevance (0 if no FTS match)
    CASE WHEN search_vector @@ q.fts_q
      THEN ts_rank_cd(search_vector, q.fts_q) ELSE 0
    END AS fts_score,
    -- Fuzzy similarity (always computed)
    greatest(
      similarity(lower(p.name), lower(q.raw)),
      word_similarity(lower(q.raw), lower(p.name))
    ) AS fuzzy_score,
    -- Prefix bonus
    CASE WHEN lower(p.name) LIKE lower(left(q.raw, 10) || '%')
      THEN 0.15 ELSE 0
    END AS prefix_bonus,
    LOG(1 + p.sales_count) AS popularity
  FROM products p, query q
  WHERE
    search_vector @@ q.fts_q          -- FTS match
    OR p.name % q.raw                  -- trigram fuzzy
    OR lower(p.name) LIKE              -- prefix
       lower(left(q.raw, 10) || '%')
)
SELECT
  id, name, brand, price,
  (fts_score * 0.6 + fuzzy_score * 0.3 + prefix_bonus + popularity * 0.1) AS final_score
FROM results
ORDER BY final_score DESC
LIMIT 20;
```

## Two-phase search (fallback strategy)

```sql
-- Phase 1: try exact FTS match
-- Phase 2: if < N results, fall back to fuzzy
-- Implement this at the application layer:

-- Step 1 (fast, accurate)
SELECT * FROM products
WHERE search_vector @@ plainto_tsquery('english', $1)
LIMIT 20;

-- Step 2 (if count < 5, run this instead)
SELECT *, similarity(name, $1) AS score
FROM products
WHERE name % $1
ORDER BY score DESC
LIMIT 20;
```

---

# Limits & Alternatives

_Limitations_[cite: 1]

Know when PostgreSQL search is enough — and when to graduate to a dedicated engine.[cite: 1]

## Hard limits

| Limit                                   | Detail                                                                                                        |
| :-------------------------------------- | :------------------------------------------------------------------------------------------------------------ |
| Max tsvector size                       | 1 MB per row. Documents larger than ~500K words must be chunked.[cite: 1]                                     |
| Max lexemes per tsvector                | 16,383 positions tracked. Past that, positions are dropped (matching still works, ranking degrades).[cite: 1] |
| GIN index size                          | Can be 2–5× the table size. Monitor with `pg_relation_size()`.[cite: 1]                                       |
| ts_headline cost                        | O(n) in document size — don't call it without filtering first.[cite: 1]                                       |
| Trigram on short strings                | <3 characters = 0 trigrams = trigram index cannot be used.[cite: 1]                                           |
| No semantic understanding               | FTS doesn't know "king" ≈ "queen" — you'd need a synonym dict or vector embeddings.[cite: 1]                  |
| No multilingual synonyms out of the box | Each language config is independent; cross-language search requires extra work.[cite: 1]                      |
| Reindexing on dict change               | Changing your text search config requires `REINDEX` of existing data.[cite: 1]                                |

> **Fits well when:** Data is in PG already, scale is <10M rows, query volume is moderate, you need transactional consistency with your search data, or your team lacks ops capacity for another service.[cite: 1]

## Alternatives

| Tool                           | Best for                                              | Advantage over PG                                                               | Cost                                                           |
| :----------------------------- | :---------------------------------------------------- | :------------------------------------------------------------------------------ | :------------------------------------------------------------- |
| **Elasticsearch / OpenSearch** | Large-scale full-text, log search, analytics[cite: 1] | Distributed, relevance tuning, ML ranking, aggregations, nested facets[cite: 1] | Complex ops, data sync, separate infra[cite: 1]                |
| **Typesense**                  | Instant typo-tolerant autocomplete[cite: 1]           | Built for low-latency (<50ms), easy setup, great defaults[cite: 1]              | Limited to search-only; data sync needed[cite: 1]              |
| **Meilisearch**                | Product/document search with UI[cite: 1]              | Developer-friendly, typo tolerance, faceting, ranking rules[cite: 1]            | Single-node only (historically); data sync[cite: 1]            |
| **Algolia**                    | Managed, real-time search at scale[cite: 1]           | 99.999% uptime SLA, instant results, great SDKs[cite: 1]                        | Expensive at scale ($$$), vendor lock-in[cite: 1]              |
| **Redis Search**               | In-memory real-time search[cite: 1]                   | Sub-millisecond, vector search support[cite: 1]                                 | Data must fit in RAM; separate from PG[cite: 1]                |
| **pgvector**                   | Semantic / AI similarity search[cite: 1]              | Stays in PG; embeddings enable "meaning-based" search[cite: 1]                  | Need to generate embeddings; approximate ANN at scale[cite: 1] |

## Extension summary

Extensions mentioned: `pg_trgm`, `fuzzystrmatch`, `unaccent`, `pgvector`, `rum`, `pg_bigm`.[cite: 1]

| Extension       | Purpose                                      | Notes                                                          |
| :-------------- | :------------------------------------------- | :------------------------------------------------------------- |
| `pg_trgm`       | Trigram fuzzy/LIKE search[cite: 1]           | Built-in, just `CREATE EXTENSION`[cite: 1]                     |
| `fuzzystrmatch` | Levenshtein, Soundex, Metaphone[cite: 1]     | Built-in; Soundex/Metaphone for phonetic matching[cite: 1]     |
| `unaccent`      | Strip accents (é→e, ü→u)[cite: 1]            | Use as a mapping in your TS config[cite: 1]                    |
| `pgvector`      | Vector similarity (embeddings)[cite: 1]      | Requires generating vectors from an AI model[cite: 1]          |
| `rum`           | Faster GIN alternative with ranking[cite: 1] | Third-party; faster `ts_rank` at query time[cite: 1]           |
| `pg_bigm`       | 2-gram index (better for CJK)[cite: 1]       | Third-party; better than trigram for Japanese/Chinese[cite: 1] |

---

# Decision Matrix

Pick the right approach for your use case at a glance.[cite: 1]

| Use Case                                  | FTS               | Fuzzy             | Prefix            | pgvector          | External                |
| :---------------------------------------- | :---------------- | :---------------- | :---------------- | :---------------- | :---------------------- |
| Blog / document search[cite: 1]           | ✓ Best[cite: 1]   | Optional[cite: 1] | Optional[cite: 1] | Overkill[cite: 1] | Overkill[cite: 1]       |
| Product catalog search[cite: 1]           | ✓ Core[cite: 1]   | ✓ Add[cite: 1]    | ✓ Add[cite: 1]    | Nice[cite: 1]     | At scale[cite: 1]       |
| User name / autocomplete[cite: 1]         | Partial[cite: 1]  | ✓ Good[cite: 1]   | ✓ Best[cite: 1]   | No[cite: 1]       | Overkill[cite: 1]       |
| Typo-tolerant search[cite: 1]             | No[cite: 1]       | ✓ Best[cite: 1]   | No[cite: 1]       | No[cite: 1]       | Typesense[cite: 1]      |
| Semantic / "meaning" search[cite: 1]      | No[cite: 1]       | No[cite: 1]       | No[cite: 1]       | ✓ Best[cite: 1]   | ES / Algolia[cite: 1]   |
| Phonetic matching ("Jon"="John")[cite: 1] | No[cite: 1]       | Partial[cite: 1]  | No[cite: 1]       | No[cite: 1]       | Soundex ext.[cite: 1]   |
| 10M+ rows, heavy search load[cite: 1]     | Can work[cite: 1] | Slow[cite: 1]     | Fast[cite: 1]     | Partial[cite: 1]  | ES / Typesense[cite: 1] |
| Transactional consistency[cite: 1]        | ✓[cite: 1]        | ✓[cite: 1]        | ✓[cite: 1]        | ✓[cite: 1]        | Extra sync[cite: 1]     |

## Performance cheat sheet

| Method                                | Index type                         | Typical latency (1M rows) | Index overhead       |
| :------------------------------------ | :--------------------------------- | :------------------------ | :------------------- |
| FTS with GIN on tsvector[cite: 1]     | GIN[cite: 1]                       | 1–5 ms[cite: 1]           | Medium[cite: 1]      |
| Prefix LIKE with B-tree[cite: 1]      | B-tree (text_pattern_ops)[cite: 1] | <1 ms[cite: 1]            | Low[cite: 1]         |
| Fuzzy with GIN trgm[cite: 1]          | GIN (gin_trgm_ops)[cite: 1]        | 5–20 ms[cite: 1]          | High (2–4×)[cite: 1] |
| LIKE '%query%' without index[cite: 1] | None (seq scan)[cite: 1]           | 500ms+[cite: 1]           | None[cite: 1]        |
| Levenshtein (no index)[cite: 1]       | None (seq scan)[cite: 1]           | 1–10 s[cite: 1]           | None[cite: 1]        |
| pgvector ANN (HNSW)[cite: 1]          | HNSW[cite: 1]                      | 2–10 ms[cite: 1]          | Very high[cite: 1]   |

> **Rule of thumb:** Start with FTS + a tsvector trigger.[cite: 1] Add `pg_trgm` when users complain about typos.[cite: 1] Add prefix B-tree for autocomplete.[cite: 1] Graduate to Elasticsearch or Typesense only when PG's latency under real load becomes unacceptable.[cite: 1]</N>
