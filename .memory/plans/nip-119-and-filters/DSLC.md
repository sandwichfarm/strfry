# NIP-119 AND Filters — Development & Specification Lifecycle (DSLC)

## Purpose

- Define a clear lifecycle for introducing and maintaining NIP-119 AND-filter support in strfry.
- Ensure that specification, design, implementation, testing, and documentation stay aligned over time.

## Stages

1. **Spec Assimilation**
   - Source of truth: ../../Develop/nips/91.md (NIP-119 AND Operator in Filters).
   - Capture local interpretation and clarifications in:
     - `000-overview.md`
     - `PH01-analysis-and-design.md`
   - Outcomes:
     - Clear semantics for `&<tag>` in the context of strfry’s filter model.
     - Explicit rules for precedence (`AND` over `OR`) and set overlap.

2. **Design Finalisation**
   - Documents:
     - `PH01-analysis-and-design.md`
     - Updates to `000-overview.md` as needed.
   - Activities:
     - Decide on `NostrFilter` data structures (e.g. `tagsAnd`).
     - Pin down index selection rules and limits.
     - Define test strategy (reference implementation + fuzz tests).
   - Exit condition:
     - No open design questions; each decision is traceable to code and tests planned in PH02/PH03.

3. **Implementation**

   - Documents:
     - `PH02-engine-and-index-changes.md`
   - Activities:
     - Implement `&<tag>` parsing, AND matching, and OR/AND reconciliation in `src/filters.h`.
     - Adjust `DBScan` index selection and `indexOnlyScans` behavior.
     - Ensure `Subscription` and related structures account for new fields.
   - Guardrails:
     - Keep changes local and focused (filters, DBQuery, minimal touch to other components).
     - Preserve backward-compatible behavior for existing filters.

4. **Verification & Hardening**

   - Documents:
     - `PH03-testing-and-hardening.md`
   - Activities:
     - Extend `test/dumbFilter.pl` to mirror NIP-119 semantics.
     - Extend `test/filterFuzzTest.pl` to generate AND-tag filters and validate:
       - `scan`, `scan-limit`, and `monitor`.
     - Add targeted regression tests for key scenarios and edge cases.
     - Run extended fuzz and monitor for performance and correctness issues.
   - Exit condition:
     - Fuzz tests and regression tests pass, including overnight runs on realistic datasets.

5. **Documentation & Rollout**

   - Documents:
     - `PH04-documentation-and-rollout.md`
   - Activities:
     - Update README/docs and CHANGES with AND-filter support and examples.
     - Decide on NIP-11 advertising and configuration.
     - Plan and execute staged rollout and rollback strategies.
   - Exit condition:
     - Feature is documented, advertised (where appropriate), and deployed with a clear recovery path.

6. **Maintenance & Evolution**

   - Ongoing responsibilities:
     - Re-run fuzz tests periodically or after significant query-engine changes.
     - Update documentation if NIP-119 evolves or new related NIPs appear.
     - Monitor for performance regressions in AND-heavy workloads.
   - Change management:
     - Any substantial changes to semantics or limits should:
       - Update `000-overview.md` and relevant PH documents.
       - Add or adjust regression tests.

## Controls and Checkpoints

- **Design Review**
  - Before starting PH02:
    - Confirm that PH01 decisions are recorded and accepted.
    - Ensure that no untracked assumptions remain.
- **Implementation Review**
  - After PH02:
    - Code walkthrough focusing on:
      - `NostrFilter` changes.
      - `DBScan` changes.
      - Error-handling consistency.
- **Test Review**
  - After PH03:
    - Validate that test coverage matches the plan:
      - AND/OR interactions.
      - Edge cases (overlaps, limits, invalid filters).
- **Release Review**
  - Before completing PH04:
    - Confirm that documentation matches the actual behavior.
    - Ensure operators have clear instructions for verifying AND-filter support.

## Traceability

- Requirements ↔ Design:
  - Each NIP-119 requirement is mapped into:
    - A design decision in `PH01-analysis-and-design.md`.
    - A high-level summary in `000-overview.md`.
- Design ↔ Implementation:
  - `PH02-engine-and-index-changes.md` lists the exact files and changes implementing each design point.
- Implementation ↔ Tests:
  - `PH03-testing-and-hardening.md` ties specific behaviors to new or extended tests in `test/dumbFilter.pl` and `test/filterFuzzTest.pl`.
- Implementation ↔ Documentation:
  - `PH04-documentation-and-rollout.md` ensures that user-facing docs mirror actual semantics.

## Exit Criteria for the DSLC

- All phases (PH01–PH04) completed with corresponding documents updated.
- Fuzz and regression tests stable, with no unresolved AND-filter issues.
- Documentation and release notes accurately describe behavior and constraints.
- The system is prepared for future adjustments with a clear path to:
  - Refine index usage for AND-only filters.
  - Adjust limits or semantics if NIP-119 evolves.

