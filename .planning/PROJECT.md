# strfry NIP-50 Search

## What This Is

Finishing and shipping NIP-50 (full-text search) support upstream into [hoytech/strfry](https://github.com/hoytech/strfry) — the most popular nostr relay implementation. Work lives on `feature/nip-50` (this fork) and is already proposed as [PR #160](https://github.com/hoytech/strfry/pull/160), where it has been validated end-to-end on a 20M+ event production relay. The milestone is to get that PR mergeable: resolve the conflicts that have accumulated against master, close the remaining known bugs with credible evidence, and add the tests + benchmark results that make a clean upstream merge realistic.

## Core Value

**The PR merges into hoytech/strfry.** Everything else — knobs, tokenizer extensions, alternate backends — is subordinate to that single outcome.

## Requirements

### Validated

<!-- Inferred from existing code on feature/nip-50 (codebase map confirms). These already work. -->

- ✓ `SearchProvider` abstraction — pluggable backend interface (`src/search/SearchProvider.h`) — existing
- ✓ LMDB-backed search index — `src/search/LmdbSearchProvider.h` with posting lists, doc-meta, persisted state — existing
- ✓ Noop search provider — testing/disabled-mode backend (`src/search/NoopSearchProvider.h`) — existing
- ✓ Tokenizer (ASCII + Unicode word boundaries, lowercased) — `src/search/Tokenizer.h` — existing
- ✓ NIP-50 query path wired through `RelayReqWorker` / `DBQuery` — existing
- ✓ Live indexer hook (events indexed on write via txn hook) — existing
- ✓ Catch-up indexer (`SearchRunner.h`) with batched progress, `lastProcessedLevId` advance fix, dup-detection on re-encounter — existing (recently fixed)
- ✓ `KindMatcher` — single/range/wildcard/exclusion syntax for `indexedKinds` — existing
- ✓ DB-utils: `cmd_search_index_stats`, `cmd_search_reindex`, `cmd_search_set_state` — existing
- ✓ Full ranking config surface — `maxQueryTerms`, `maxPostingsPerToken`, `maxCandidateDocs`, `recencyBoostPercent`, `overfetchFactor`, `candidateRanking`, `candidateRankMode`, weighted `rankWeight*` — existing, used in production
- ✓ Production validation on Lee Salminen's ~20M event relay (per PR thread) — existing

### Active

<!-- Building toward these to make PR #160 mergeable. -->

- [ ] **Resolve merge conflicts** — squash to clean topical commits and rebase onto `hoytech/master`; PR currently `mergeStateStatus: DIRTY`
- [ ] **`cmd_search_destroy_index`** — explicit command to drop the search index (called out by PR comment 2026-03-02; needed for clean reindex)
- [ ] **Verify resume / interrupt correctness** — kill mid-index → restart → assert no `MDB_KEYEXIST`, correct `lastIndexedLevId` advance, no stuck-batch logging
- [ ] **Indexer fuzz test** — Perl-style harness (analog of `test/filterFuzzTest.pl`) that randomises events and asserts index contents match expected postings; covers resume by injecting interrupts
- [ ] **Bench results posted to PR** — exercise `bench/scenarios/*.yml` to produce concrete index-build and query-latency numbers under representative load
- [ ] **Docs / CHANGES entry** — README NIP-50 section, config-block doc comments, `CHANGES` line tying the work to the PR
- [ ] **Address hoytech's review** — once he leaves a line-level review, fold his feedback in (placeholder requirement; scope filled when feedback arrives)

### Out of Scope

- **External search backends (Meilisearch, Tantivy, etc.)** — abstraction supports it, milestone ships LMDB only; introducing a second backend bloats the PR and slows merge
- **CJK segmentation / stemmer / accent folding** — current ASCII-plus-Unicode-boundary tokenizer is "good enough"; deeper language work is a future milestone
- **NIP-91 AND-tag filters (`&<tag>`)** — separate work captured in `PR.md`; not bundled into this PR (keeps merge surface minimal)
- **New ranking knobs** — the existing weighted ranking surface is already broad and field-tested; adding more would invite bikeshedding
- **Reformatting the existing PR commits beyond the squash-rebase** — don't rewrite working code for style; minimise diff surface vs upstream

## Context

**The product:** strfry is hoytech's C++ nostr relay — single-binary, LMDB-backed, custom code-gen build system (`golpe`). NIP-50 is the nostr search extension; relays that implement it advertise `search` in NIP-11 and accept `search:` in REQ filters.

**Where the work stands:**
- Branch: `feature/nip-50` (this fork, sandwichfarm/strfry).
- Upstream PR: [hoytech/strfry#160](https://github.com/hoytech/strfry/pull/160), opened 2025-11-12, +2430/-22 across 34 files. Status `OPEN`, mergeable `CONFLICTING`.
- Hoytech (PR owner) commented 2026-02-27: *"This is very impressive, thank you! ... broadly like it's on the right track."* No formal line-level review yet.
- Production validation: Lee Salminen ran the branch on a ~20M event relay; query performance "nearly instant"; original indexer stalls/loops were fixed in `feature/nip-50-indexertweaks` and merged back.
- Recent fixes (this fork): infinite-loop on missing events, batch-end persist, always-log batch progress, dup-detection on `MDB_APPENDDUP`. Reportedly resolved but not 100% verified by author.

**Why now:** PR has been open ~5.5 months, picking up production users on the fork. Each week of upstream drift adds conflicts. The window for a tidy upstream merge is narrowing.

**Codebase context:** See `.planning/codebase/` — STACK.md, ARCHITECTURE.md, STRUCTURE.md, CONCERNS.md cover the C++/LMDB/golpe baseline plus the NIP-50 in-flight files.

## Constraints

- **Tech stack**: C++17, LMDB (single-writer), `golpe` codegen — same as the rest of strfry. No new languages, runtimes, or top-level deps.
- **Maintainer style**: hoytech's code is intentionally minimal — manual memory layouts, hand-rolled LMDB cursors, no STL-heavy abstractions. New code must look like it belongs.
- **Backwards compatibility**: search is gated behind `relay.search.enabled`; default off. Must be invisible to operators who don't opt in.
- **Branch hygiene**: squash-rebase strategy for this PR; preserve attribution where possible (Lee's perf observations, prior commits).
- **Performance**: indexing must not stall on large DBs (Lee's 20M-event datapoint is the floor); search queries must remain "nearly instant" at that scale.
- **Test harness**: Perl-based (`test/*.pl`) — new tests should match that style, not introduce a new framework.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Target upstream PR #160, not the fork | Maximum impact + community adoption; fork already runs in production | — Pending (depends on merge) |
| Ship LMDB backend only | Smaller PR surface = higher merge probability; abstraction preserves future external backends | — Pending |
| Tokenizer stays where it is (ASCII + Unicode boundaries) | "Good enough"; CJK / stemming would balloon scope | — Pending |
| Squash + rebase to clean topical commits | Easier upstream review than 30+ messy iteration commits | — Pending |
| Verify indexer correctness via Perl-style fuzz test | Matches existing `filterFuzzTest.pl` precedent; no new framework | — Pending |
| Add destroy-index command | Explicitly requested in PR thread; needed for clean reindex workflow | — Pending |
| Add bench results to PR comment | Concrete perf evidence reduces hoytech's review burden | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-27 after initialization*
