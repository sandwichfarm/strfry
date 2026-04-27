# External Integrations

**Analysis Date:** 2026-04-27

## APIs & External Services

**Nostr Protocol:**
- Nostr relay implementation (no external API calls)
  - Implements NIPs: 1, 2, 4, 9, 11, 22, 28, 40, 70, 77
  - Clients connect via WebSocket and send JSON-RPC style messages
  - EVENT, REQ, CLOSE message types handled by `src/apps/relay/RelayReqWorker.cpp`, `RelayIngester.cpp`

**NIP-50 Full-Text Search (Feature Branch):**
- Integrated search backend (LMDB-based, no external service)
  - Search queries execute against local inverted index
  - BM25 scoring algorithm with configurable k1=1.2, b=0.75
  - Configuration: `relay.search.enabled`, `relay.search.backend` (supports "lmdb", "noop")
  - Search index tables: `SearchIndex` (inverted postings), `SearchDocMeta` (document metadata)

**negentropy Set Reconciliation:**
- Protocol for efficient sync with clients and other relays
  - Implementation: `external/negentropy/` (vendored)
  - LMDB backend via `negentropy::storage::BTreeLMDB`
  - Enabled by default: `relay.negentropy.enabled = true`
  - Storage table: `NegentropyFilter` for cached filters
  - Used in `src/NegentropyFilterCache.h` and negentropy worker threads

**Write Policy Plugins:**
- Optional external executable for custom event validation
  - Config: `relay.writePolicy.plugin` (path to executable)
  - Timeout: `relay.writePolicy.timeoutSeconds` (default 10s)
  - Called by ingester before committing events
  - No built-in plugin; users provide custom scripts

## Data Storage

**Databases:**
- **LMDB (Lightning Memory-Mapped Database)**
  - Type: Embedded key-value store with memory-mapped I/O
  - Storage path: Configurable via `db` config (default: `./strfry-db/`)
  - Connection: Per-thread LMDB transactions via lmdbxx C++ wrapper
  - ORM/Client: Custom lmdbxx wrapper (`golpe/external/lmdbxx/include/`)
  - No external database required; all data stored locally on filesystem

**Database Schema:**
```
Tables (structured via golpe.yaml -> FlatBuffers):
  - Event: Indexed nostr events with multiple indices (created_at, id, pubkey, kind, tags, etc.)
  - EventPayload: Raw JSON + compressed event data (zstd)
  - Meta: Schema versioning and metadata
  - CompressionDictionary: zstd dictionary for event compression
  - NegentropyFilter: Cached negentropy reconciliation state
  - SearchState: NIP-50 search indexing progress
  
Raw Tables (no structure):
  - SearchIndex: Inverted index postings [token] -> [levId:48|tf:16]
  - SearchDocMeta: BM25 document metadata [levId] -> [docLen:16|kind:16|reserved:32]
```

**Configuration:**
- LMDB parameters tunable: `dbParams.maxreaders` (256), `dbParams.mapsize` (10TB default)
- Event retention: configurable age limits (`events.rejectEventsOlderThanSeconds`, `events.ephemeralEventsLifetimeSeconds`)

**File Storage:**
- All data stored locally in LMDB database directory
- No external file storage (S3, etc.)
- Optional zstd-compressed event payloads stored in EventPayload table

**Caching:**
- **In-Memory Caches:**
  - Filter/subscription cache in `ActiveMonitors` (in-memory state for active REQs)
  - Kind matching compiled once and cached in `LmdbSearchProvider`
  - Negentropy filter cache in `NegentropyFilterCache` (LMDB-backed)

## Authentication & Identity

**Auth Provider:**
- None for relay operation (Nostr is cryptographically self-authenticating)
- Clients prove identity via Schnorr signatures on events (verified with libsecp256k1)
- Admin contact pubkey configured via `relay.info.pubkey` (for NIP-11 relay info, not authentication)

**Event Verification:**
```cpp
// Signature verification in src/events.cpp
bool verifySig(secp256k1_context* ctx, std::string_view sig, std::string_view hash, std::string_view pubkey)
// Uses Schnorr signatures (BIP-340)
```

**IP/Access Control:**
- Real IP detection from reverse proxy header: `relay.realIpHeader` (e.g., "x-real-ip")
- No built-in authentication; operators implement via plugins or reverse proxy

## Monitoring & Observability

**Error Tracking:**
- None detected (no Sentry, error tracking services)
- Errors logged locally via loguru to stdout/stderr

**Logs:**
```
Framework: loguru header-only library (golpe/external/loguru/)
Log Levels:
  - LI: Info
  - LW: Warning
  - LE: Error
  - LD: Debug (optional)

Log Output: stdout/stderr (can be configured to files)

Key Logged Events:
  - Event validation/rejection (src/events.cpp)
  - REQ query performance (relay.logging.dbScanPerf)
  - Invalid events (relay.logging.invalidEvents)
  - Search provider status (cmd_relay.cpp)
  - Config reloads (hot-reload monitoring)
```

**Metrics:**
- **Prometheus metrics endpoint** (IP:port/metrics)
  - Tracks: Client/relay message counts by verb, events indexed by kind, search queries, etc.
  - Implementation: `src/PrometheusMetrics.h`
  - Format: Prometheus-compatible text format

## CI/CD & Deployment

**Hosting:**
- Linux-based (Arch, Ubuntu, etc. tested)
- Docker support provided via `sample-docker-compose.yaml`
- No cloud-specific integrations (not AWS, GCP, Azure native)

**Container Deployment:**
```yaml
# sample-docker-compose.yaml
services:
  strfry-nostr-relay:
    build: .
    volumes:
      - /path/to/strfry.conf:/etc/strfry.conf
      - /path/to/strfry-db:/app/strfry-db
    ports:
      - "7777:7777"
```

**CI Pipeline:**
- None detected in repository
- Build: `make` or `make build` (uses golpe/rules.mk)
- Clean: `make clean`
- Test: `make test-subid` (signature verification tests)

**Zero Downtime Restart:**
- Graceful shutdown support with pending request draining
- Configuration hot-reload without restart (via file watchers)
- Process restart without losing connections (documented in README.md)

## Environment Configuration

**Required Environment Variables:**
- None mandatory for basic operation
- System-level: `PERL5LIB=golpe/vendor/` (for code generation)

**Optional Environment Variables:**
- `BIN` - Binary name (default: `strfry`)
- `APPS` - Apps to build (default: `dbutils relay mesh`)
- `OPT` - Compiler optimization flags (default: `-O3 -g`)
- Build: Set via Makefile or command line: `make OPT="-O2 -g" BIN=my-relay`

**Secrets Location:**
- No .env file detected
- No sensitive API keys needed (Nostr is symmetric/self-signing)
- Configuration via `strfry.conf` (plain text TOML, should protect from unauthorized access)

## Webhooks & Callbacks

**Incoming:**
- WebSocket connections only (Nostr protocol)
  - Endpoint: `ws://bind:port/` (default `ws://127.0.0.1:7777/`)
  - HTTP fallback for NIP-11 relay info: `GET http://bind:port/` (returns JSON)

**Outgoing:**
- **Write Policy Plugin Execution:**
  - Relay spawns external process specified in `relay.writePolicy.plugin`
  - IPC: Process stdin/stdout communication (implementation in writer thread)
  - Timeout: 10 seconds (configurable)
  - Use case: Custom event filtering/validation before acceptance

**Streaming/Syncing:**
- **Relay-to-Relay Sync (mesh module):**
  - Commands: `strfry mesh stream` (downstream), `strfry mesh sync` (upstream)
  - No WebSocket to external services; instead, relay-to-relay connections via strfry's own Nostr protocol
  - Configuration: Relay target address in command arguments
  - Implementation: `src/apps/mesh/cmd_stream.cpp`, `cmd_sync.cpp`

## Search System (NIP-50 - Feature Branch)

**Search Index Architecture:**
- **Tokenizer:** Normalizes text, splits into searchable tokens
  - Implementation: `src/search/Tokenizer.h`
  - Lowercasing, stemming (TBD), stop words

- **Indexing Pipeline:**
  - Catch-up indexer thread (auto-started if LMDB backend enabled)
  - Indexes new events asynchronously via `LmdbSearchProvider::runCatchupIndexer()`
  - Tracks progress in `SearchState` table: `lastIndexedLevId`, `indexVersion`
  - Only indexes kinds matching `relay.search.indexedKinds` config

- **Search Query Execution:**
  - Query parsing: `search/SearchRunner.h`
  - Filter support: by kind, author (pubkey), time range (since/until)
  - Limit: default 100, configurable, max 500 per filter
  - Config: `relay.search.maxQueryTerms` (max 6)

- **Ranking & Scoring:**
  - BM25 algorithm (Okapi BM25)
  - Document field: event message content
  - Parameters: k1=1.2, b=0.75 (configurable in `relay.search.bm25`)
  - Recency boosting: `relay.search.recencyBoostPercent` (0-100, default 1%)
  - Overfetch strategy: `relay.search.overfetchFactor` (default 5x)
  - Candidate ranking modes: order or weighted
  - Ranking order: configurable (e.g., "terms-tf-recency", "tf-recency-terms")

- **Performance Tuning:**
  - Max postings per token: `relay.search.maxPostingsPerToken` (100k default)
  - Max candidate docs: `relay.search.maxCandidateDocs` (1000 default)
  - Query time budget: `relay.queryTimesliceBudgetMicroseconds` (10000)

---

*Integration audit: 2026-04-27*
