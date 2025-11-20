# PH03 — Testing and Hardening for NIP-119 AND Filters

## Objectives

- Extend the test suite to cover NIP-119 AND-filter semantics thoroughly.
- Use fuzzing and reference comparisons to detect regressions in `scan` and `monitor` paths.
- Validate error handling and resource limits for AND filters under realistic and adversarial workloads.

## Scope

- `test/dumbFilter.pl`
- `test/filterFuzzTest.pl`
- Any additional targeted test scripts required for AND filters.

## Tasks

### 1. Extend dumbFilter.pl to Support AND Semantics

- Add recognition of `&<tag>` keys:
  - Implement handling for `&e`, `&p`, `&t`, etc., in terms of event tags.
  - Enforce the same hex vs raw semantics as `NostrFilter`:
    - `&e`, `&p`: 64-char hex strings, matching the original event JSON tags.
    - Other single-character tags: raw strings.
- Implement overlap rules:
  - For each tag name `x` with both `&x` and `#x`:
    - Remove all values from `#x` that also appear in `&x`.
    - If the resulting `#x` is empty, treat it as if `#x` were absent.
- Implement AND matching:
  - For each `&x`:
    - For each required value `v` in `&x`, the event must contain at least one tag `[x, v, ...]`.
  - For OR tags, keep existing behavior.
- Keep the code simple and explicit; correctness and readability take precedence over micro-optimisations here.

### 2. Extend filterFuzzTest.pl with AND Filters

- Update `genRandomFilterGroup`:
  - Introduce a probability of generating `&t` filters alongside `#t`.
  - Optionally, add `&e` and `&p` generation if it is easy to source valid ids/pubkeys from the test dataset.
  - Ensure that:
    - Generated AND filters respect the necessary formats (hex for ids and pubkeys).
    - The distribution includes:
      - AND-only filters (only `&t`).
      - Mixed AND+OR filters (`&t` + `#t`).
      - Filters combining AND with ids/authors/kinds.
- Ensure we do not unintentionally create invalid/unsupported filters that would only exercise error paths (except where explicitly testing error handling).

### 3. Fuzzing: Scan and Monitor Paths

- Reuse existing fuzz strategy:
  - For `scan` and `scan-limit`:
    - Compare:
      - `./strfry export --reverse | dumbFilter | jq -r .id | sort | sha256sum`
      - vs `./strfry scan --pause 1 --metrics <filterGroup> | jq -r .id | sort | sha256sum`.
  - For `monitor`:
    - Generate monitor commands, feed them to `./strfry monitor`, and compare resulting IDs to `./strfry scan` on the “interest” filter group.
- Add seeds and durations:
  - Run with multiple seeds (via `SEED` env var) for a reasonable period (overnight runs).
  - Document recommended commands and expected behavior in `.memory/plans/nip-119-and-filters` (or test README).

### 4. Targeted Regression and Edge-Case Tests

- Create a small, focused test script or documented sequence of `strfry scan` commands to cover:
  - The NIP-119 example:
    - `{ "kinds": [1], "&t": ["meme", "cat"], "#t": ["black", "white"] }`.
  - Cases with overlapping AND/OR values:
    - `&t: ["a", "b"]`, `#t: ["a", "b", "c"]` → effectively `AND(a,b)` AND `OR(c)`.
  - AND-only filters:
    - `{"&t":["nostr"]}`.
  - Multiple AND-tag keys:
    - `{"&t":["a","b"], "&e":[<id1>,<id2>]}`.
  - Interaction with `since`/`until` and `limit`.
- For each scenario, document:
  - Inputs.
  - Expected result set properties (e.g. “must include event X, must exclude event Y”).

### 5. Error Handling and Limits

- Add tests (or manual scripts) to ensure:
  - Invalid `&` keys:
    - Wrong length (e.g. `"&topic"`).
    - Non-indexable tags.
  - Malformed value arrays (non-array or mixed types) cause clear and consistent errors.
  - Excessive tag-key usage:
    - Filters that exceed the allowed number of distinct tag keys across `#` and `&` are rejected with the expected error.
  - Very large value arrays:
    - Ensure `FilterSetBytes` still enforces size limits and error messages when constructing AND sets.

### 6. Performance and Resource Checks

- During extended fuzz runs, monitor:
  - CPU usage and memory growth.
  - Latency and throughput for `scan` and `monitor`.
- Look for patterns where AND-heavy filters induce significant slowdowns:
  - If found, capture representative filters and include them as separate regression cases.
  - These can inform later optimisation work (e.g. using AND values for index selection).

## Deliverables

- Updated `test/dumbFilter.pl` with full AND support.
- Updated `test/filterFuzzTest.pl` generating AND-tag filters and validating:
  - `scan`, `scan-limit`, and `monitor`.
- Documentation of:
  - How to run AND-filter regression tests.
  - Example commands for operators/maintainers.

## Exit Criteria

- Fuzz tests run with AND filters enabled for a sustained period on a real dataset without mismatches or crashes.
- Targeted regression tests pass and cover the main semantics and edge cases.
- No observed performance regressions beyond the expected behavior for more expressive filters (and any tradeoffs are documented).
- Ready to proceed to PH04 (documentation and rollout).

