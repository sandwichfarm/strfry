# Codebase Structure

**Analysis Date:** 2026-04-27

## Directory Layout

```
/home/sandwich/Develop/strfry/
├── build/                       # Code generation outputs (golpe)
│   ├── main.cpp                # CLI dispatcher, entry point
│   ├── config.h / config.cpp    # Generated config parsing
│   ├── defaultDb.h              # Generated LMDB schema
│   ├── StrfryTemplates.h        # Generated HTTP response templates
│   └── app_git_version.h        # Generated build metadata
├── src/
│   ├── apps/
│   │   ├── relay/               # Relay server implementation
│   │   │   ├── cmd_relay.cpp    # Entry point for 'strfry relay'
│   │   │   ├── RelayServer.h    # Main coordinator (thread pools, message dispatch)
│   │   │   ├── RelayWebsocket.cpp # WebSocket I/O and connection handling
│   │   │   ├── RelayIngester.cpp  # Message parsing and protocol dispatch
│   │   │   ├── RelayWriter.cpp    # Event storage and search indexing
│   │   │   ├── RelayReqWorker.cpp # Query execution (REQ/COUNT)
│   │   │   ├── RelayReqMonitor.cpp # Live subscription updates
│   │   │   ├── RelayNegentropy.cpp # NIP-77 sync protocol
│   │   │   ├── RelayCron.cpp    # Periodic cleanup (ephemeral, expirations)
│   │   │   ├── RelaySignalHandler.cpp # SIGUSR1, config reload
│   │   │   └── golpe.yaml       # Relay-specific golpe config overrides
│   │   ├── dbutils/             # Database utility commands
│   │   │   ├── cmd_info.cpp     # Display DB version/stats
│   │   │   ├── cmd_export.cpp   # Export events to JSON
│   │   │   ├── cmd_import.cpp   # Import events from JSON
│   │   │   ├── cmd_compact.cpp  # LMDB compaction
│   │   │   ├── cmd_delete.cpp   # Delete by criteria
│   │   │   ├── cmd_scan.cpp     # Scan indices
│   │   │   ├── cmd_dict.cpp     # Compression dictionary ops
│   │   │   ├── cmd_monitor.cpp  # Watch DB changes
│   │   │   ├── cmd_negentropy.cpp # Test negentropy protocol
│   │   │   ├── cmd_search_index_stats.cpp  # Search index metrics
│   │   │   ├── cmd_search_reindex.cpp      # Rebuild search index
│   │   │   └── cmd_search_set_state.cpp    # Set search state
│   │   └── mesh/                # Mesh networking (sync/stream/router)
│   │       ├── cmd_router.cpp   # Network router
│   │       ├── cmd_stream.cpp   # Event stream
│   │       └── cmd_sync.cpp     # Sync command
│   ├── search/                  # NIP-50 full-text search (feature/nip-50)
│   │   ├── SearchProvider.h     # Abstract interface
│   │   ├── SearchProvider.cpp   # Factory function
│   │   ├── LmdbSearchProvider.h # LMDB-backed BM25 implementation
│   │   ├── NoopSearchProvider.h # No-op stub
│   │   ├── SearchRunner.h       # Integration with DBQuery
│   │   ├── Tokenizer.h          # Text tokenization for search
│   │   └── KindMatcher.h        # Kind filtering for indexing
│   ├── tmpls/                   # HTTP response templates (for templar code-gen)
│   ├── ThreadPool.h             # Generic message queue + worker threads
│   ├── RelayServer.h            # Message type definitions
│   ├── QueryScheduler.h         # Query round-robin orchestration
│   ├── DBQuery.h                # Query execution state machine
│   ├── Subscription.h           # REQ subscription metadata
│   ├── filters.h                # NostrFilter parsing and matching
│   ├── events.h                 # Event validation, lookup, write APIs
│   ├── events.cpp               # Event implementation
│   ├── PackedEvent.h            # Binary event format (id, pubkey, tags)
│   ├── Decompressor.h           # Zstd decompression with dictionary cache
│   ├── Decompressor.cpp         # Decompressor implementation
│   ├── WSConnection.h           # WebSocket connection state
│   ├── WriterPipeline.h         # Legacy pipeline (not used in relay)
│   ├── NegentropyFilterCache.h # Cache for negentropy filter state
│   ├── ActiveMonitors.h         # Active query monitoring
│   ├── PluginEventSifter.h     # Write policy plugin interface
│   ├── PrometheusMetrics.h     # Prometheus metrics macros
│   ├── Bytes32.h                # 32-byte hash wrapper
│   ├── jsonParseUtils.h         # JSON parsing helpers
│   ├── constants.h              # Global constants
│   ├── global.h                 # Global variable declarations
│   ├── misc.cpp / misc.h        # Utility functions
│   ├── onAppStartup.cpp         # Startup hooks (schema init, migrations)
│   └── AnsiLogo.h               # Banner text
├── external/                    # Git submodules
│   └── negentropy/              # NIP-77 protocol library
├── golpe/                       # Golpe code generator
│   ├── logging.cpp              # Logging configuration
│   └── (golpe compiler source)
├── golpe.yaml                   # Main schema definition (tables, indices, config)
├── Makefile                     # Build orchestration
├── README.md                    # Documentation
├── docs/                        # Additional documentation
├── scripts/                     # Build/deploy scripts
└── debian/                      # Debian packaging
```

## Directory Purposes

**build/:**
- Purpose: Generated code and build artifacts from golpe
- Contains: `defaultDb::environment` (LMDB schema), config parsing, HTTP templates, git version
- Generated: Yes
- Committed: No (regenerated each build)

**src/apps/relay/:**
- Purpose: Core nostr relay server
- Contains: Thread pools (WebSocket, Ingester, Writer, ReqWorker, ReqMonitor, Negentropy), message dispatch, protocol implementation
- Key files: `RelayServer.h` (coordinator), `cmd_relay.cpp` (entry), RelayWebsocket/Ingester/Writer/ReqWorker (layers)

**src/search/:**
- Purpose: NIP-50 full-text search (in-flight on feature/nip-50)
- Contains: Search provider interface, LMDB BM25 implementation, tokenization, kind filtering
- Key files: `SearchProvider.h` (interface), `LmdbSearchProvider.h` (BM25 implementation)
- Status: Integrated into Writer (indexEvent on write) and ReqWorker (query execution)

**src/apps/dbutils/:**
- Purpose: Offline database maintenance commands
- Contains: Export/import, compaction, info, scanning, search index management
- Entry: Each cmd_*.cpp has cmd_*() function called from main

**external/negentropy/:**
- Purpose: NIP-77 efficient synchronization protocol
- Contains: Negentropy library (C++) for range-based event reconciliation
- Integrated: RelayNegentropy uses both Vector (in-memory) and BTreeLMDB (indexed) storage backends

**golpe/:**
- Purpose: Code generation framework (like flatbuffers for LMDB)
- Contains: Schema compiler, code templates, logging config
- Used: Processes golpe.yaml to generate C++ schema/config files

## Key File Locations

**Entry Points:**
- `build/main.cpp`: CLI dispatcher; loads config, opens DB, calls cmd_*()
- `src/apps/relay/cmd_relay.cpp`: `cmd_relay()` creates RelayServer and calls run()
- `src/apps/dbutils/cmd_*.cpp`: Each utility command has cmd_*() function

**Configuration:**
- `golpe.yaml`: Schema definition (Event table, indices, config fields)
- `src/apps/relay/golpe.yaml`: Relay-specific config overrides
- Generated: `build/config.h`, `build/config.cpp`

**Core Logic:**
- `src/apps/relay/RelayServer.h`: Message type definitions, thread pool declarations
- `src/apps/relay/RelayWebsocket.cpp`: Accept connections, parse JSON, dispatch to ingester
- `src/apps/relay/RelayIngester.cpp`: Parse protocol (EVENT, REQ, COUNT, CLOSE, NEG-*), validate
- `src/apps/relay/RelayWriter.cpp`: Deduplicate, apply write policy, store to LMDB, index for search
- `src/apps/relay/RelayReqWorker.cpp`: Execute queries, scan indices, integrate search provider
- `src/apps/relay/RelayReqMonitor.cpp`: Live subscription updates on DBChange
- `src/apps/relay/RelayNegentropy.cpp`: NIP-77 protocol state machine

**Testing:**
- `test/SubIdTests.cpp`: Unit tests for subscription ID validation
- No full integration test suite in main repo

**Database Layer:**
- `src/events.h`: Event lookup, validation, write APIs
- `src/events.cpp`: Event processing implementation
- `src/filters.h`: NostrFilter parsing (ids, authors, kinds, tags, since, until, limit, search)
- `src/DBQuery.h`: Query state machine, index scanning, candidate collection

**Search (NIP-50):**
- `src/search/SearchProvider.h`: Interface (indexEvent, deleteEvent, query)
- `src/search/LmdbSearchProvider.h`: BM25 scoring with LMDB SearchIndex table
- `src/search/SearchRunner.h`: Integration point in DBQuery
- `src/search/Tokenizer.h`: Tokenization and normalization for indexing/query
- `src/search/KindMatcher.h`: Kind filtering (which kinds to index)

## Naming Conventions

**Files:**
- Headers: `PascalCase.h` (e.g., `RelayServer.h`, `ThreadPool.h`)
- Implementation: `PascalCase.cpp` (e.g., `RelayIngester.cpp`, `events.cpp`)
- CLI commands: `cmd_*.cpp` (e.g., `cmd_relay.cpp`, `cmd_info.cpp`)
- Tests: `*Tests.cpp` (e.g., `SubIdTests.cpp`)
- Config: `golpe.yaml` for schema; generated `config.h`
- Generated: Files in `build/` prefixed with `generated_` or specific names like `defaultDb.h`

**Directories:**
- Layer-specific: `relay/`, `dbutils/`, `mesh/`, `search/`
- Functional: `apps/`, `external/`
- Infrastructure: `golpe/`, `scripts/`, `docs/`

**Code Identifiers:**
- Classes/Structs: `PascalCase` (e.g., `RelayServer`, `ThreadPool`, `DBQuery`)
- Functions: `camelCase` (e.g., `runIngester()`, `addSub()`, `queryScheduler.process()`)
- Member variables: `camelCase` (e.g., `connId`, `subId`, `eventPayload`)
- Enum members: `UPPER_CASE` (e.g., `EventSourceType::IP4`, `EventWriteStatus::Written`)
- Type aliases: `PascalCase` (e.g., `RecipientList`, `NostrFilter`)
- Message types: `Msg*` prefix (e.g., `MsgWebsocket`, `MsgIngester`, `MsgWriter`)
- Thread pool functions: `run*` prefix (e.g., `runWebsocket()`, `runIngester()`, `runWriter()`)
- Thread-pool template: `tp*` prefix (e.g., `tpWebsocket`, `tpIngester`)

## Where to Add New Code

**New Feature (e.g., new NIP protocol):**
- Primary code: Create `src/apps/relay/Relay<NIP>.cpp` for protocol implementation
- Integration: Add layer function to `RelayServer` (e.g., `run<NIP>()`)
- Thread pool: Declare in `RelayServer.h` (e.g., `ThreadPool<Msg<NIP>> tp<NIP>`)
- Message dispatch: Add message type in `RelayServer.h` (e.g., `struct Msg<NIP> : NonCopyable { ... }`)
- Tests: Add `test/<NIP>Tests.cpp`

**New Database Command:**
- Location: `src/apps/dbutils/cmd_<name>.cpp`
- Signature: `void cmd_<name>(const std::vector<std::string> &subArgs)`
- Registration: Declare in `build/main.cpp` (generated from golpe.yaml), add dispatch case in `run()`
- Example: `src/apps/dbutils/cmd_info.cpp` for minimal template

**New Component/Module:**
- Headers: Place in `src/` root (if shared) or `src/<layer>/` (if layer-specific)
- Example: New search backend → `src/search/My<Name>SearchProvider.h`, implement `ISearchProvider` interface
- Example: New filter type → Add to `src/filters.h` as new `FilterSet*` struct

**Utilities/Helpers:**
- Shared helpers: `src/misc.h` for utility functions
- Crypto: Uses secp256k1 library; link in Makefile
- JSON: Uses tao::json; parsed via `tao::json::from_string()`
- LMDB: Interact via `env.txn_ro()`, `env.txn_rw()`, `env.lookup_*()`, `env.dbi_*`

**Search/Index Enhancements:**
- Add to golpe.yaml if new LMDB table/index needed
- Implement in `src/search/LmdbSearchProvider.h` if search-specific
- Integrate into `SearchRunner.h` if query-level integration needed
- Test via `cmd_search_*` dbutils commands

## Special Directories

**build/:**
- Purpose: Generated code from golpe schema
- Generated: Yes (by `make` running golpe compiler)
- Committed: No (gitignored; regenerated each build)
- Files: `defaultDb.h`, `config.h`, `StrfryTemplates.h`, `app_git_version.h`

**external/:**
- Purpose: Git submodules (negentropy, external libraries)
- Generated: No
- Committed: Yes (submodule references)
- Manual: Rarely modified; use `git submodule update --init` to pull

**test/:**
- Purpose: Unit and integration tests
- Generated: No
- Committed: Yes
- Status: Minimal; mainly `SubIdTests.cpp`; integration tests run via `strfry relay` with test harness (out of repo)

**.git/, .github/, debian/, scripts/:**
- Standard support directories (version control, CI/CD, packaging, deployment)

## Feature Branch Notes (feature/nip-50)

**In-Flight Changes:**
- Search provider interface and LMDB implementation in `src/search/`
- Search integration in `RelayWriter::runWriter()` (lines 84–99 of RelayWriter.cpp) — indexes events post-write
- Search integration in `RelayReqWorker::runReqWorker()` (line 10) — wires searchProvider to QueryScheduler
- Search integration in `QueryScheduler` (line 16) — holds searchProvider pointer
- Search integration in `DBQuery` (Search-related logic in query execution)
- Search integration in `RelayWebsocket::runWebsocket()` (lines 53–58 of RelayWebsocket.cpp) — advertises NIP-50 in supportedNips if healthy()
- New LMDB tables in golpe.yaml: `SearchIndex`, `SearchDocMeta`, `SearchState` (lines 103–114)
- New dbutils commands: `cmd_search_index_stats.cpp`, `cmd_search_reindex.cpp`, `cmd_search_set_state.cpp`
- Search provider initialization: `cmd_relay.cpp` line 40 — `searchProvider = makeSearchProvider()`
- Search catch-up indexer thread: `cmd_relay.cpp` lines 78–86 — runs async background indexing

**Search Architecture:**
- Abstract provider pattern allows multiple backends (currently Noop, LMDB BM25)
- Lazy catch-up: Background indexer thread catches up on write lag
- Graceful degradation: Search failure does not fail writes; only affects query results
- Config-driven: `relay.search.enabled`, `relay.search.backend`, `relay.search.indexedKinds` control behavior

---

*Structure analysis: 2026-04-27*
