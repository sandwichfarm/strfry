# Testing Patterns

**Analysis Date:** 2026-04-27

## Test Framework

**Runner:**
- Perl-based test scripts for integration testing (no C++ unit test framework like gtest/catch2)
- C++ unit tests compiled to binaries: `test/SubIdTests.cpp` → `build/subid_tests`
- Shell scripts for test orchestration: `test/writeTest.pl`, `test/filterFuzzTest.pl`, `test/syncTest.pl`

**Run Commands:**
```bash
make test-subid                    # Run C++ unit tests
perl test/writeTest.pl             # Test event writing, replacements, deletions
perl test/filterFuzzTest.pl scan-limit   # Fuzz test query engine with limit
perl test/filterFuzzTest.pl scan         # Fuzz test query engine without limit
perl test/filterFuzzTest.pl monitor      # Fuzz test monitor engine
perl test/runSyncTests.pl          # Run sync tests
perl test/syncTest.pl              # Individual sync test
```

**Build Integration:**
- Makefile target: `test-subid` in `/home/sandwich/Develop/strfry/Makefile`
- Compilation: `$(CXX) $(CXXFLAGS) $(INCS) $< -o $@`
- Depends on `build/golpe.h` (generated)

## Test File Organization

**Location:**
- C++ tests: `test/*.cpp`
- Perl integration tests: `test/*.pl` (executable, top-level in test dir)
- Test configs: `test/cfgs/` (subdirectory for config files)

**Naming:**
- C++ test files: Suffixed with `Tests.cpp` or similar explicit convention: `SubIdTests.cpp`
- Perl test scripts: Descriptive verb-noun: `writeTest.pl`, `filterFuzzTest.pl`, `syncTest.pl`, `runSyncTests.pl`
- No `*.test.*` or `*.spec.*` suffix pattern

## Test Structure

**C++ Unit Test Pattern (`test/SubIdTests.cpp`):**

```cpp
namespace {
void expectSuccess(std::string_view name, std::string subId) {
    try {
        SubId s(subId);
        if (s.sv() != std::string_view(subId)) {
            std::cerr << name << ": round-trip mismatch\n";
            std::exit(EXIT_FAILURE);
        }
    } catch (const std::exception &e) {
        std::cerr << name << ": expected success but threw: " << e.what() << "\n";
        std::exit(EXIT_FAILURE);
    }
}

void expectFailure(std::string_view name, std::string subId) {
    try {
        SubId s(subId);
        std::cerr << name << ": expected failure but constructed successfully\n";
        std::exit(EXIT_FAILURE);
    } catch (const std::exception &) {
    }
}
} // namespace

int main() {
    expectSuccess("max length", std::string(64, 'a'));
    expectFailure("too long", std::string(65, 'a'));
    expectFailure("empty", std::string());
    // ... more cases
    std::cout << "SubId tests passed\n";
    return EXIT_SUCCESS;
}
```

**Patterns:**
- No test framework; raw C++ with exception handling
- Helper functions with descriptive names for setup/assertion
- Anonymous namespace to hide test helpers
- Direct exit on failure with stderr message (not assert-based)
- stdout success message at end
- Test case as function call in main

**Perl Integration Test Pattern (`test/writeTest.pl`):**

```perl
use strict;
use Carp;
$SIG{ __DIE__ } = \&Carp::confess;
use Data::Dumper;
use JSON::XS;

my $ids = [
    { sec => '...', pub => '...' },
    { sec => '...', pub => '...' },
];

doTest({
    desc => "Basic insert",
    events => [
        qq{--sec $ids->[0]->{sec} --content "hi" --kind 1 },
        qq{--sec $ids->[0]->{sec} --content "hi 2" --kind 1 },
    ],
    verify => [ 0, 1, ],
});

doTest({
    desc => "Replacement, newer timestamp",
    events => [
        qq{--sec $ids->[0]->{sec} --content "hi" --kind 10000 --created-at 5000 },
        qq{--sec $ids->[0]->{sec} --content "hi 2" --kind 10000 --created-at 5001 },
        qq{--sec $ids->[0]->{sec} --content "hi" --kind 10000 --created-at 5000 },
    ],
    verify => [ 1, ],
});
```

**Patterns:**
- Hash-based test declaration with descriptive fields: `desc`, `events`, `verify`, `assertIds`
- Event input as nostril CLI command strings: `--sec`, `--content`, `--kind`, `--created-at`, `--tag`
- Verification by index into events array (which survive/which are replaced)
- Test assertions via `doTest()` helper function (implementation not shown but called at bottom)

**Fuzz Test Pattern (`test/filterFuzzTest.pl`):**

```perl
my $kinds = [qw/1 7 4 42 0 30 3 6/];
my $pubkeys = [qw{ 887645fef0ce0c3c1218d2f5d8e6132a19304cdc57cd20281d082f38cfea0072 ... }];
my $ids = [qw{ 25e5c82273a271cb1a840d0060391a0bf4965cafeb029d5ab55350b418953fbb ... }];
my $topics = [qw{ bitcoin nos nostr ... }];

sub genRandomFilterGroup {
    my $useLimit = shift;
    my $numFilters = $useLimit ? 1 : (rand()*10)+1;
    my @filters;
    for (1..$numFilters) {
        my $f = {};
        while (!keys %$f) {
            if (rand() < .15) {
                $f->{ids} = [];
                for (1..(rand()*10)) {
                    push @{$f->{ids}}, $ids->[int(rand() * @$ids)];
                }
            }
            if (rand() < .3) {
                $f->{authors} = [];
                for (1..(rand()*5)) {
                    push @{$f->{authors}}, $pubkeys->[int(rand() * @$pubkeys)];
                }
            }
            # ... more conditions
        }
    }
}
```

**Patterns:**
- Pre-defined test data pools (kinds, pubkeys, ids, topics)
- Random generation within bounds
- Deterministic via seed (changeable via `SEED` env var)
- Runs forever (overnight stress test)
- Tests query engine with and without limits
- Tests monitor engine separately

## Mocking

**Framework:** None explicit; Perl tests use actual process execution

**Patterns:**
- Real relay instance spawned in test: `perl test/writeTest.pl` starts relay, sends events via WebSocket/REST
- nostril CLI used to generate valid signed Nostr events
- Database state checked via queries and export

**What to Mock:**
- Not applicable; integration tests use real components

**What NOT to Mock:**
- Event signing (uses real secp256k1 via nostril)
- Database operations (uses real LMDB)
- WebSocket connections (uses real relay instance)
- Event parsing and validation (tested against actual codebase)

## Fixtures and Factories

**Test Data:**
- Hardcoded pubkeys in `filterFuzzTest.pl` (~35 known keys)
- Hardcoded event IDs in `filterFuzzTest.pl` (~40 known IDs)
- Hardcoded topics in `filterFuzzTest.pl` (~10 topics)
- Generated on-the-fly in tests via random selection and nostril generation

**Location:**
- `test/cfgs/` directory for configuration files (e.g., relay configs for different test scenarios)
- Hard-coded data in Perl scripts themselves
- No separate fixture files or factories observed

**Example from `writeTest.pl`:**
```perl
my $ids = [
    {
        sec => 'c1eee22f68dc218d98263cfecb350db6fc6b3e836b47423b66c62af7ae3e32bb',
        pub => '003ba9b2c5bd8afeed41a4ce362a8b7fc3ab59c25b6a1359cae9093f296dac01',
    },
    {
        sec => 'a0b459d9ff90e30dc9d1749b34c4401dfe80ac2617c7732925ff994e8d5203ff',
        pub => 'cc49e2a58373abc226eee84bee9ba954615aa2ef1563c4f955a74c4606a3b1fa',
    },
];
```

## Coverage

**Requirements:** None enforced in CI

**View Coverage:** Not applicable (no code coverage tooling configured)

## Test Types

**Unit Tests:**
- Scope: Single type/component validation
- Example: `SubIdTests.cpp` tests `SubId` struct construction with various inputs
- Approach: Direct instantiation, exception-based validation
- Executes: Single binary test, runs in-process

**Integration Tests:**
- Scope: Event writing, replacement, deletion logic; filter matching; sync protocol
- Example: `writeTest.pl` creates relay instance, sends events, verifies DB state changes
- Approach: Real relay process, WebSocket communication, nostril event generation
- Pre-condition: Events injected via nostril with valid signatures and timestamps

**E2E Tests:**
- Scope: Filter matching correctness (fuzz tests), monitor subscription correctness
- Framework: Perl-based deterministic random generation
- Command: `perl test/filterFuzzTest.pl scan`, `perl test/filterFuzzTest.pl monitor`
- Coverage: Query engine, monitor engine under stress with random filter combinations
- Determinism: Seeded random (can be controlled via `SEED` env var)

**Requirement for Fuzz Tests:**
- Reads from "well populated DB"
- Setup: `zstdcat ../nostr-dumps/nostr-wellorder-early-500k-v1.jsonl.zst | ./strfry import`
- Runs indefinitely (overnight stress test mode)

## Common Patterns

**Async Testing:**
- Not explicitly tested; relay runs in background during Perl tests
- WebSocket connections are synchronous in test context (wait for response)
- No async/await patterns; blocking I/O in test scripts

**Error Testing:**
```cpp
void expectFailure(std::string_view name, std::string subId) {
    try {
        SubId s(subId);
        std::cerr << name << ": expected failure but constructed successfully\n";
        std::exit(EXIT_FAILURE);
    } catch (const std::exception &) {
        // Expected
    }
}
```

Pattern: Try-catch, exit on unexpected success, ignore caught exception.

## Test Execution Notes

**From `test/README.md`:**
- Tests run from project root, not test/ directory
- `writeTest.pl` requires `nostril` tool in PATH (event generation)
- Fuzz tests need populated DB for coverage (wellorder 500k dataset recommended)
- No special compiler flags; uses default CXXFLAGS from build system
- CI runs tests indirectly: `make -j4` in GitHub Actions ubuntu.yml; no explicit `make test` target

**CI Integration:**
- GitHub Actions in `.github/workflows/ubuntu.yml`
- Build step: `git submodule update --init && make setup-golpe && make -j4`
- No automated test execution in CI (only build verification)
- Manual test execution would require: dependencies (nostril), pre-populated DB, test env setup

---

*Testing analysis: 2026-04-27*
