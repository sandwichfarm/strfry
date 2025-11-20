# PH04 — Documentation and Rollout for NIP-119 AND Filters

## Objectives

- Document NIP-119 AND-filter support in strfry’s user-facing docs.
- Provide guidance for operators and client developers on usage, limitations, and examples.
- Plan a safe rollout, including configuration and NIP-11 advertising if applicable.

## Scope

- strfry documentation (README and docs/).
- Release notes (CHANGES).
- Optional: configuration and NIP-11 metadata updates.

## Tasks

### 1. Update User Documentation

- README.md and/or a dedicated filters section:
  - Describe supported filter fields including `&<tag>`.
  - Provide concrete examples:
    - Simple AND-only filter.
    - Mixed AND+OR filter (spec example).
    - Combining AND with ids/authors/kinds and time bounds.
  - Explain:
    - That AND takes precedence over OR within a tag.
    - That AND values are removed from OR sets for the same tag.
    - That only indexable single-character tags are supported for `&`.
- If needed, add a dedicated filter doc (e.g. `docs/filters.md`):
  - Summarise the filter model:
    - `NostrFilter` AND across fields.
    - `NostrFilterGroup` OR across filters.
  - Include AND-filter semantics as a subsection.

### 2. Configuration and Limits Documentation

- Document any relevant config options that affect AND filters:
  - `maxReqFilterSize`, tag-count limits, `events__maxTagValSize`, etc.
- Clarify how these limits apply when both `#` and `&` are used:
  - Combined tag-key limit.
  - Value size constraints.
- Note any potential performance considerations:
  - AND filters may require reading full events instead of relying solely on indices.
  - Best practices for clients:
    - Keep AND value sets small and focused.
    - Avoid constructing filters that are effectively “full DB scans” with AND-only tags.

### 3. NIP-11 and Feature Advertising

- Decide whether to advertise explicit support for NIP-119/NIP-91 in NIP-11:
  - Check existing NIP-11 representation and `relay.info.nips` config (see CHANGES and docs).
  - If strfry chooses to advertise this NIP:
    - Update default NIP list or example configuration to include the appropriate NIP number.
- Document for operators:
  - How to enable/disable advertising of AND-filter support via configuration.

### 4. Release Notes

- Update CHANGES:
  - Add a concise entry describing:
    - Support for NIP-119 AND filters.
    - Brief note on semantics and any noteworthy limits.
  - Mention any tests or new tools added specifically for AND filters (e.g. fuzz improvements).

### 5. Rollout Plan

- Internal rollout:
  - Run extended fuzz tests (from PH03) on a staging or mirror environment with a realistic dataset.
  - Monitor for:
    - Performance regressions.
    - Unexpected error rates.
- Production rollout steps:
  - Upgrade procedure for existing deployments (no DB migrations expected).
  - Optional: staged rollout:
    - First enable on non-critical relays.
    - Observe behavior and metrics.
    - Gradually roll out to more important relays.
- Rollback strategy:
  - Document how to revert to the previous version if unexpected issues arise.
  - Note that clients relying on AND filters will see reduced capabilities against older versions.

## Deliverables

- Updated README/docs with AND-filter documentation and examples.
- Updated CHANGES with a clear entry for NIP-119 support.
- Optional NIP-11/NIP list updates where appropriate.
- A short runbook or section in docs for operators explaining:
  - What AND filters are.
  - How to test them on their relay.
  - Known caveats and best practices.

## Exit Criteria

- Documentation is up to date and accurately reflects the implementation.
- Release notes clearly visible to downstream users and operators.
- A practical rollout and rollback plan exists and has been sanity-checked against existing deployments.

