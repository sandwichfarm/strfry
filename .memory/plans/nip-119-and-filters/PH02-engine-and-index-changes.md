# PH02 — Core Engine and Index Changes for NIP-119 AND Filters

## Objectives

- Implement the core `NostrFilter` changes to support `&<tag>` filters.
- Adjust DB scanning/index selection and matching logic to respect AND semantics without regressing existing behavior.
- Ensure errors and limits for AND filters are enforced consistently.

## Scope

- `src/filters.h`:
  - `FilterSetBytes` and `NostrFilter`.
- `src/DBQuery.h`:
  - `DBScan` constructor and index selection.
- Minor wiring to ensure new fields are visible in `Subscription` and related structures (no protocol changes).

## Tasks

### 1. Extend NostrFilter with AND Tags

- Add `tagsAnd` field to `NostrFilter`:
  - `flat_hash_map<char, FilterSetBytes> tagsAnd;`
- Update constructor:
  - Recognise `&` keys: `k.starts_with('&')`.
  - Validate key:
    - Length must be 2; otherwise `throw herr("unindexed AND tag filter");` (or a similar, consistent message).
    - Tag must be a valid, indexable one-character tag.
  - Value handling:
    - Must be an array; empty → set `neverMatch = true`.
    - For `&p` and `&e`: hex-decode 64-character strings to 32-byte values (`FilterSetBytes(v, true, 32, 32)`).
    - For other single-character tags: raw UTF-8 values up to `MAX_INDEXED_TAG_VAL_SIZE` (`FilterSetBytes(v, false, 0, MAX_INDEXED_TAG_VAL_SIZE)`).
  - Maintain the existing path for `#` keys; do not mix parsing logic between `#` and `&` at this stage.

### 2. Reconcile OR and AND Tag Values

- After the initial constructor loop:
  - For each tag key `t` present in both `tags` (OR) and `tagsAnd` (AND):
    - Compute `O_t' = O_t \ A_t`.
    - Rebuild `FilterSetBytes` for `O_t'` or drop the `tags[t]` entry entirely if `O_t'` is empty.
  - Implementation approaches:
    - **Simple rehydrate/rebuild**:
      - Extract values from both `FilterSetBytes` instances using `at(i)` and reconstruct new sets.
      - Given typical filter sizes (small arrays), this is safe and simple.
    - Optionally, add helper methods to `FilterSetBytes` if needed, but avoid overcomplicating until required by profiling.
- Ensure that:
  - `numMajorFields` uses the union of keys across `tags` and `tagsAnd`.
  - The existing limit on tag count (`"too many tags in filter"`) is enforced on the union of tag names.

### 3. Update indexOnlyScans Logic

- In `NostrFilter` constructor:
  - After computing `numMajorFields`, update `indexOnlyScans`:
    - Keep existing logic for the `ids`/`authors`/`kinds` cases.
    - Add a condition: if `tagsAnd` is non-empty, set `indexOnlyScans = false`.
- Add unit-level checks (via assertions or tests) to ensure:
  - A filter that uses any AND tags never results in an index-only scan.

### 4. Implement AND Matching in NostrFilter::doesMatch

- Extend `doesMatch(PackedEventView ev)`:
  - After time and major field checks (ids/authors/kinds) and before OR tags:
    - For each `(tagName, filterSetAnd)` in `tagsAnd`:
      - For each index `i` in `0..filterSetAnd.size()-1`:
        - Retrieve `requiredVal = filterSetAnd.at(i)`.
        - Use `ev.foreachTag` to scan event tags:
          - If a tag with `tagName` and `tagVal == requiredVal` is found, mark this value as satisfied and break.
        - If no such tag is found, return `false`.
  - Leave the existing OR-tag loop over `tags` unchanged.
- Confirm that:
  - AND conditions are enforced even when OR tags are absent.
  - AND conditions are enforced when combining with other major fields and time bounds.

### 5. Adjust DBScan Index Selection

- In `DBScan` constructor (src/DBQuery.h):
  - Maintain the existing priority order between id, tags, authors×kinds, authors, kinds, created_at.
  - Tag-driven selection:
    - Keep the current `else if (f.tags.size())` block as the primary tag-index path.
    - Ensure `indexOnly` is set to `false` whenever `f.tagsAnd` is non-empty.
  - Decide on AND-only behavior:
    - Initial implementation:
      - Do **not** add a separate `else if (f.tagsAnd.size())` path; rely on other indices (authors/kinds/created_at) for AND-only filters.
      - This avoids premature complexity and still ensures correctness via `doesMatch`.
    - Document the possible future extension:
      - A later phase could introduce a `tagsAnd`-driven tag index selection with `indexOnly = false` if profiling suggests this is worthwhile.

### 6. Ensure Compatibility in Other Components

- `Subscription` and any code that copies or stores `NostrFilter`:
  - Ensure that the new `tagsAnd` map is correctly copied/moved (constructor, move semantics).
- `ActiveMonitors`:
  - No changes required for correctness (it already calls `f->doesMatch`).
  - Optionally, consider whether to reference `tagsAnd` for indexing, but this can be deferred.

## Deliverables

- Updated `src/filters.h` with:
  - New `tagsAnd` field.
  - Parsing, reconciliation, and matching logic for AND tags.
  - Appropriate updates to `indexOnlyScans` and tag-count limits.
- Updated `src/DBQuery.h` with:
  - Ensured `indexOnly` disabling when `tagsAnd` is present.
  - No regressions in index selection for existing filter shapes.

## Exit Criteria

- All relevant code compiles and links.
- A minimal manual test (e.g. via `strfry scan` with a hand-crafted filter including `&t`) demonstrates:
  - AND semantics for `&t`.
  - Correct interaction between `&t` and `#t`.
  - No regression for filters that do not use `&`.
- Ready to proceed to PH03 (testing and hardening).

