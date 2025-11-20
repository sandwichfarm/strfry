# NIP-119 AND Filters in strfry — High-Level Overview

## Purpose

- Implement NIP-119 (“AND Operator in Filters”, referred to here as NIP-91 in some local docs) in strfry’s query engine.
- Allow relays and tools built on strfry to express “must have all of these tag values” constraints within a single tag key, in addition to existing OR semantics.
- Preserve current performance characteristics and safety properties (bounded filter complexity, predictable index usage, and resilience against adversarial filters).

## NIP-119 Summary

Specification (../../Develop/nips/91.md):

- New filter keys: `&<tag>` for indexable tags (e.g. `&t`, `&p`, `&e`, `&d`).
- Semantics:
  - `&t: ["meme", "cat"]` means: event must have tag `t` with **both** `meme` and `cat` present among its tag values.
  - `#t` continues to mean: event must have tag `t` with **any** of the listed values (OR semantics).
  - When both `&t` and `#t` are present:
    - AND takes precedence over OR.
    - Values that appear in `&t` are **ignored** in `#t` (behave as if removed from the OR set).
    - Example:
      - Filters: `{ "kinds":[1], "&t":["meme","cat"], "#t":["black","white"] }`
      - Match: kind 1 events whose `t` tags include **both** `meme` and `cat`, **and** at least one of `black` or `white`.

## Current strfry Filter Model (Baseline)

- `NostrFilter` (src/filters.h:86) represents one filter object:
  - `ids`, `authors`, `kinds` as `FilterSetBytes` / `FilterSetUint`.
  - `tags: flat_hash_map<char, FilterSetBytes>` for `#<tag>` filters (single-character, indexable tags only).
  - `since`, `until`, `limit`, `indexOnlyScans`, `neverMatch`.
- `NostrFilter::doesMatch(PackedEventView ev)`:
  - Enforces `since/until`.
  - Applies `ids`, `authors`, `kinds`.
  - For each `#<tag>` entry:
    - Event must have at least one tag with that tag name and a value in the corresponding `FilterSetBytes` (OR within each tag key; AND across different tag keys).
- `NostrFilterGroup` is an OR of `NostrFilter`s (REQ array-of-filters semantics).
- Indexing (src/DBQuery.h, src/events.cpp):
  - Events are indexed by id, pubkey, kind, created_at, and tag (`dbi_Event__tag` keys are `[tagName][tagVal][created_at]`).
  - `DBScan` chooses a primary index:
    - Prefer `ids`, else tags, else compact `authors x kinds`, else authors, else kinds, else created_at.
  - `indexOnlyScans` allows some queries to be served from the index without reloading full events:
    - True when there is at most one “major” field, or exactly `authors`+`kinds`.
    - For pure tag filters (`#t` only), `indexOnlyScans` is true and correctness relies entirely on the tag index.
- Testing:
  - `test/dumbFilter.pl` is a simple reference implementation of filter semantics (no AND support yet).
  - `test/filterFuzzTest.pl` generates random filters (ids/authors/kinds/#e/#p/#t) and cross-checks:
    - `./strfry export | dumbFilter` vs `./strfry scan` and `./strfry monitor`.

This baseline only supports OR semantics for tag values; AND within a tag key is not yet expressible.

## Target Semantics for AND Filters in strfry

### New Filter Keys

- `&<tag>` where `<tag>` is a single printable character whose tag values are indexed by strfry:
  - Supported examples: `&e`, `&p`, `&t`, `&d` (and any other one-character tags that are indexed).
  - Any `&` key whose length is not 2 or whose tag is not indexable should be rejected as an “unindexed AND tag filter”.

### Matching Rules

Given a filter object with:

- OR tags: `#x: [values...]`.
- AND tags: `&x: [values...]`.

For a given tag name `x`:

- Let `A_x` be the set of values from `&x`.
- Let `O_x` be the set of values from `#x` **excluding** any values in `A_x`.

Then an event matches if:

- For every tag name `x` in `A_x`:
  - The event includes at least one tag `[x, v, ...]` for **each** `v` in `A_x` (AND within the tag).
- For every tag name `x` where `O_x` is non-empty:
  - The event includes at least one tag `[x, v, ...]` with `v` in `O_x` (OR within that tag).
- Existing AND across different tag keys is preserved:
  - e.g. having both `&t` and `#e` still means the event must satisfy both tag constraints.
- If, after removing overlaps with `A_x`, `O_x` becomes empty:
  - `#x` contributes no additional constraint (no “empty array means never match” here; we treat it as if `#x` were absent).

### Precedence and Grouping

- AND takes precedence over OR **within a tag key**:
  - Conceptually: `(&t: ["a","b"]) AND ( #t: ["c","d"] )` where the OR set has had `["a","b"]` removed.
- At the filter object level:
  - All major fields (`ids`, `authors`, `kinds`, time bounds, tags/AND-tags) are combined with AND.
- At the filter group level:
  - Filters in the group remain OR’ed as today.

### Error Handling and Limits

- Invalid shapes:
  - `&` keys with non-array values should be rejected as bad filters.
  - `&` keys with empty arrays: treated analogously to empty `ids`/`authors`/`kinds` arrays and result in a `neverMatch` filter.
- Tag-key count limits:
  - Current limit: `if (tags.size() > 3) throw herr("too many tags in filter");`.
  - With AND tags, the **combined** number of distinct tag keys across `#` and `&` will remain capped (target: <= 3); details to be pinned down in the implementation phase.

## High-Level Design Direction

### Data Model Extensions

- Extend `NostrFilter` to represent AND-tag constraints explicitly:
  - Introduce `flat_hash_map<char, FilterSetBytes> tagsAnd;` (name to be finalised).
  - Keep existing `tags` map for OR semantics (`#` keys) to minimise disruption.
  - Treat the union of keys in `tags` and `tagsAnd` as “tag major fields” for:
    - Calculating `numMajorFields`.
    - Enforcing tag-count limits.
    - Deciding whether `indexOnlyScans` can be true.

### Parsing Behavior

- In `NostrFilter` constructor:
  - For keys starting with `'#'`:
    - Continue to parse into `tags` as today, but record the raw values so they can be reconciled with `&` values later.
  - For keys starting with `'&'`:
    - Treat exactly the same value element types and length limits as for `'#'`:
      - `&p`, `&e`: hex-decoding to 32-byte values.
      - Other one-character tags: raw UTF-8 up to `MAX_INDEXED_TAG_VAL_SIZE`.
    - Store results in `tagsAnd`.
  - After all keys are processed:
    - For each tag name present in both `tags` and `tagsAnd`, remove from the OR set any values that exist in the AND set.
    - If the OR set becomes empty after this reconciliation, drop that tag from `tags` entirely (do not mark the filter as `neverMatch` just for this case).
  - Update `numMajorFields` and `indexOnlyScans`:
    - Tag “major field” count is based on the union of `tags` and `tagsAnd` keys.
    - If any AND tags are present, `indexOnlyScans` must be **false** (we need full events to enforce AND semantics).

### Matching Behavior

- `NostrFilter::doesMatch` will be extended to:
  - Check AND tags first:
    - For each `(tagName, filterSet)` in `tagsAnd`:
      - For each value in `filterSet`, assert that there exists at least one tag in the event with that `tagName` and value.
      - A simple and robust approach is to loop over required values and scan the event tags each time; given bounded tag counts, this is acceptable and keeps the implementation straightforward.
      - If any required value is missing, return `false`.
  - Then check OR tags (using the existing loop over `tags`), with the OR sets already cleaned of AND values.
  - All existing checks (`since`, `until`, `ids`, `authors`, `kinds`) remain unchanged.

### DB Scanning and Index Selection

- `DBScan` will remain responsible for choosing a primary index and generating candidate events:
  - For tag-driven queries:
    - Today: `else if (f.tags.size())` chooses `dbi_Event__tag`.
    - With AND tags:
      - If there are any OR tags (`f.tags.size() > 0`), we continue to use the tag index seeded by whichever OR tag key has the fewest values.
      - If there are only AND tags and no OR tags:
        - We may extend `DBScan` to treat `tagsAnd` as eligible for selecting the tag index (using their values to build search keys), or fall back to another index (e.g. created_at) if we want to avoid complexity.
      - In all cases where AND tags are present:
        - `indexOnly` must be false, so each candidate event is re-checked via `NostrFilter::doesMatch`.
  - This preserves correctness while still benefiting from tag indices for narrowing candidate sets where possible.

### Interaction with ActiveMonitors and Other Components

- `ActiveMonitors` uses `NostrFilter`’s fields to register interest sets:
  - When a filter has `#` tags only:
    - It registers the tag + value combinations in `allTags` and uses `f->doesMatch` as the final guard.
  - With AND tags:
    - We will keep using `f->doesMatch` as the authority on correctness.
    - Indexing of tags for monitors can remain best-effort:
      - Filters that only use `&` (and no `#`) may end up in `allOthers` or may register their AND tag values into `allTags` purely as a coarse routing hint.
      - The final AND semantics are always enforced by `f->doesMatch`.
- Negentropy, `Subscription`, and other components that treat tags as a coarse “where clause” will continue to rely on `NostrFilter` as the canonical evaluator.

## Acceptance Criteria (End State)

- **Protocol correctness**
  - `&<tag>` filters behave exactly as described in NIP-119, including:
    - AND precedence over OR within a tag key.
    - Ignoring AND values from the OR set.
    - Correct behavior when both `&` and `#` are present for the same tag name.
- **Backwards compatibility**
  - Existing filters using only `ids`, `authors`, `kinds`, `#<tag>`, `since`, `until`, `limit` behave exactly as before (no regressions in results).
- **Performance and safety**
  - Tag-based filter complexity remains bounded (combined tag key count limit enforced).
  - Queries using AND tags do not cause unbounded scan behavior or pathological performance regressions in realistic workloads.
- **Testing**
  - `dumbFilter` implements the same AND semantics as `NostrFilter`.
  - `filterFuzzTest` is extended to generate AND-tag filters and continues to pass for:
    - `scan`, `scan-limit`, and `monitor` modes.
- **Documentation**
  - strfry’s user-facing documentation describes:
    - Supported AND syntax (`&<tag>`).
    - Precise semantics and examples.
    - Limitations (e.g. only indexable single-character tags, tag-count limits).

This document defines *what* we want from NIP-119 support in strfry and how it should conceptually fit into the existing query engine. The phase documents in this directory spell out *how* we will implement, harden, and ship it.

