# PH01 — Analysis and Design for NIP-119 AND Filters

## Objectives

- Fully pin down how NIP-119 AND filters map onto strfry’s current filter and index architecture.
- Decide on data structures, invariants, and error semantics for `&<tag>` across all relevant components.
- Identify edge cases and explicitly document the intended behavior.

## Scope

- `NostrFilter` construction and matching (src/filters.h).
- DB scanning and index selection (src/DBQuery.h).
- Active monitor behavior (src/ActiveMonitors.h).
- Reference filter implementation used by tests (test/dumbFilter.pl).
- Fuzz testing strategy for AND filters (test/filterFuzzTest.pl).

## Tasks

### 1. NIP-119 Clarification and Local Conventions

- Confirm that NIP-119 (spec file ../../Develop/nips/91.md) is the authoritative reference for AND semantics, regardless of local naming as “NIP-91”.
- Clarify “indexable tags” in strfry:
  - Events index tags whose name is a single character and whose value size is ≤ `MAX_INDEXED_TAG_VAL_SIZE` (src/events.cpp, src/constants.h).
  - Valid `&` keys must refer only to such tags.
- Decide on:
  - Whether we support `&` for all single-character tags or only a subset (e.g. `e`, `p`, `t`, `d`).
  - Exact error messages for unsupported `&` keys (e.g. `"unindexed AND tag filter"` or unified with the existing `"unindexed tag filter"` path).

### 2. Data Model and Invariants

- Extend `NostrFilter` with a dedicated AND-tag structure:
  - `flat_hash_map<char, FilterSetBytes> tagsAnd;` (name to be finalised before coding).
- Define invariants:
  - A `FilterSetBytes` remains immutable after construction, storing unique, sorted values.
  - For a given tag name `x`:
    - OR set `tags[x]` contains only values not present in AND set `tagsAnd[x]`.
    - If both maps are present and OR set becomes empty after reconciliation, `tags` does **not** contain `x`.
  - `numMajorFields` is computed on the union of keys across `tags` and `tagsAnd`.
  - `indexOnlyScans` is **false** whenever `tagsAnd` is non-empty.
- Ensure that tag-count limits are applied to the combined tag-key set:
  - Decide whether `tags.size() + tagsAnd.size()` or the union of distinct tag names is the limiting factor (recommended: union count).

### 3. Parsing and Validation Design

- Define parsing rules for `NostrFilter` constructor:
  - `'#'` keys:
    - Behavior unchanged; still parse to `tags`.
  - `'&'` keys:
    - Require exactly two characters (e.g. `"&t"`).
    - Enforce the same length constraints as `#` for the corresponding tag:
      - `p`, `e`: 64-char hex → 32-byte binary via `from_hex`.
      - Others: plain strings up to `MAX_INDEXED_TAG_VAL_SIZE`.
    - On violation, throw a clear error (consistent with existing code style).
  - Overlap reconciliation:
    - After parsing all keys, reconcile OR vs AND per tag name:
      - Conceptually, `O_x := O_x \ A_x`.
      - If `O_x` becomes empty, drop that entry from `tags`.
    - Avoid converting the entire `FilterSetBytes` back to a vector of strings unless necessary; consider adding helper methods if warranted.
  - Empty arrays:
    - For `ids`/`authors`/`kinds` and `#<tag>`: current behavior is to treat them as `neverMatch`.
    - For `&<tag>`:
      - Treat empty arrays the same way (filter can never match).
      - Ensure we still short-circuit the filter out of the group in `NostrFilterGroup` constructor.

### 4. Matching Semantics Design

- Update `NostrFilter::doesMatch(PackedEventView ev)`:
  - Order:
    1. `neverMatch` check.
    2. Time bounds (`since`, `until`).
    3. ids/authors/kinds checks.
    4. **AND-tag checks** (new).
    5. OR-tag checks (existing loop over `tags`).
  - AND-tag algorithm options:
    - **Simple approach (preferred for first implementation)**:
      - For each `(tagName, filterSet)` in `tagsAnd`:
        - For each required value `v` in `filterSet`:
          - Scan the event’s tags via `ev.foreachTag` and ensure at least one `[tagName, v]` exists.
          - If any `v` is missing, return `false`.
      - Complexity: O(number_of_AND_values × total_tags_in_event), which is acceptable given existing limits on tag counts.
    - Note for later optimisation:
      - We may add helper methods to `FilterSetBytes` or caching inside `doesMatch` if profiling shows the simple approach is too slow.
  - OR-tag algorithm:
    - Keep the existing `tags` loop unchanged.
    - Because overlaps have already been removed from OR sets at parse time, spec’s “ignore in OR” rule is satisfied.

### 5. Index Selection and Scan Behavior

- Review `DBScan` constructor logic:
  - Current order: `ids` → `tags` → `authors × kinds` → `authors` → `kinds` → `created_at`.
  - Today, tag-based scans are only triggered when `f.tags.size() > 0`.
- Design decisions:
  - **Index-only behavior**:
    - Any presence of `tagsAnd` must force `indexOnly = false`.
    - This ensures we always load full events to enforce AND semantics, even if tag indices are used to narrow candidate sets.
  - Tag index selection when AND is present:
    - If there are OR tags (`f.tags.size() > 0`):
      - Reuse existing logic to choose the best tag key and value set (fewest OR values).
    - If there are only AND tags, and no OR tags:
      - Option A (simpler initial implementation):
        - Do not use the tag index. Fall back to `created_at` or another field.
      - Option B (more efficient, to consider in later phase):
        - Allow using `tagsAnd` sets to seed cursors against `dbi_Event__tag`, with `indexOnly = false`.
  - Explicitly document which option we will implement first in PH02 and what the upgrade path is if we later want to make AND-only queries more index friendly.

### 6. Monitor and Auxiliary Component Behavior

- `ActiveMonitors`:
  - Continue to rely on `f->doesMatch` as the correctness gate.
  - Decide whether to:
    - Only index OR tags in `allTags`, leaving AND-only filters in `allOthers`.
    - Or also index AND tag values as coarse hints (with an explicit comment noting that AND semantics are only enforced by `doesMatch`).
  - Document any tradeoffs and final choice so future maintainers understand the reasoning.
- Negentropy and `Subscription`:
  - Verify that AND-tag filters are properly carried around and that any equality comparisons on `NostrFilter` objects remain valid or are updated as necessary.

### 7. Testing Strategy Design

- Extend `test/dumbFilter.pl`:
  - Add support for `&<tag>` keys with exactly the same semantics as `NostrFilter`.
  - Implement overlap removal between `&` and `#` with clear, readable Perl code.
- Extend `test/filterFuzzTest.pl`:
  - Introduce random generation of `&t` filters, and optionally `&e`/`&p` where data is available in the test dataset.
  - Ensure generated filters respect basic invariants (e.g. valid hex for `e`/`p`).
  - Verify that fuzz tests for `scan`, `scan-limit`, and `monitor` modes still cross-check `strfry` against `dumbFilter`.
- Add a small set of fixed regression tests:
  - Exact reproduction of the spec example.
  - Cases with overlapping `#` and `&` values.
  - Filters that use AND-only tags.
  - Filters that mix AND with other major fields (ids, authors, kinds).

## Deliverables

- A refined, unambiguous design for:
  - `NostrFilter` data model changes.
  - Matching semantics, including AND precedence and OR interaction.
  - Index selection strategy when AND tags are present.
  - Test and fuzz coverage additions.
- Updated version of 000-overview.md with any clarified decisions.
- Green light to begin implementation in PH02.

## Exit Criteria

- All design questions above are answered and recorded in this document (or referenced sub-docs).
- There is a clear mapping from each design decision to:
  - A concrete code change in PH02.
  - One or more tests to be added or extended in PH03.
- No known ambiguities remain about how AND filters should behave in strfry.

