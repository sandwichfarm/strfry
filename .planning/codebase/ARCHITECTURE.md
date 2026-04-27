# Architecture

**Analysis Date:** 2026-04-27

## Pattern Overview

**Overall:** Multi-threaded message-passing relay server using modular thread pools and dedicated processing stages for input (ingestion), storage (writing), and querying.

**Key Characteristics:**
- Synchronous request/response via thread-pool message queues (no async/await)
- LMDB-backed persistent storage with single writer thread
- NIP-50 full-text search via pluggable search providers (Noop or LMDB)
- Negentropy protocol support for efficient event synchronization
- Golpe code-generation framework for schema and configuration
- Event validation at ingestion; write filtering via plugins

## Layers

**WebSocket I/O Layer:**
- Purpose: Accept client connections, parse JSON/binary messages, dispatch to ingester thread pool
- Location: `src/apps/relay/RelayWebsocket.cpp`, `src/WSConnection.h`, `src/RelayServer.h` (runWebsocket)
- Contains: Connection management, HTTP upgrade handling, NIP-11 server info, binary compression/decompression
- Depends on: uWebSockets, uS::Hub, Decompressor
- Used by: Ingester receives raw client messages

**Ingester Layer (Protocol Parsing):**
- Purpose: Parse client messages (EVENT, REQ, COUNT, CLOSE, NEG-*), validate structure, dispatch to writer or query layers
- Location: `src/apps/relay/RelayIngester.cpp`
- Contains: Event validation (signature, timestamp, size), filter parsing, subscription ID validation
- Depends on: secp256k1 for crypto, FilterValidator, Decompressor for compression handling
- Used by: Writer receives validated events; ReqWorker receives subscriptions; Negentropy receives sync requests

**Writer Layer (Event Persistence):**
- Purpose: Apply write policies, deduplicate events, store to LMDB, trigger search indexing
- Location: `src/apps/relay/RelayWriter.cpp`
- Contains: Write policy plugins (acceptEvent), deduplication, event serialization to LMDB EventPayload table
- Depends on: PluginEventSifter, NegentropyFilterCache, SearchProvider
- Used by: Broadcasts written events to active subscriptions; updates search index

**ReqWorker Layer (Query Execution):**
- Purpose: Execute REQ/COUNT queries incrementally, scan LMDB indices, apply NIP-50 search if requested
- Location: `src/apps/relay/RelayReqWorker.cpp`
- Contains: QueryScheduler, DBQuery for scan orchestration, integration point with SearchProvider
- Depends on: QueryScheduler, DBQuery, SearchProvider, Decompressor
- Used by: Sends event batches back to clients; passes completed subscriptions to ReqMonitor

**ReqMonitor Layer (Live Subscription Updates):**
- Purpose: Keep REQ subscriptions alive, deliver new events matching ongoing subscriptions
- Location: `src/apps/relay/RelayReqMonitor.cpp`
- Contains: Active subscription tracking, event delivery when new events pass filter criteria
- Depends on: Subscription state, DBChange notification (when new events written)
- Used by: Receives DBChange messages from Writer, broadcasts to matching subscriptions

**Negentropy Layer (Synchronization Protocol):**
- Purpose: Handle NIP-77 negentropy protocol for efficient event range synchronization
- Location: `src/apps/relay/RelayNegentropy.cpp`
- Contains: Negentropy storage adapters (Vector for in-memory, BTreeLMDB for indexed), stateful/stateless views
- Depends on: negentropy library, LMDB BTree storage
- Used by: Clients query for missing events within a range; relay returns efficient diff

**Cron/Signal Handler (Housekeeping):**
- Purpose: Periodic cleanup (ephemeral events, expirations), signal handling
- Location: `src/apps/relay/RelayCron.cpp`, `src/apps/relay/RelaySignalHandler.cpp`
- Contains: Deletion of expired events, config reload monitoring
- Depends on: Event deletion API, signal masks
- Used by: Standalone thread pools, no external dependencies on other layers

**Search Provider (NIP-50):**
- Purpose: Abstract interface for full-text search backends (Noop, LMDB)
- Location: `src/search/SearchProvider.h`, `src/search/LmdbSearchProvider.h`
- Contains: Token indexing, BM25 scoring, kind filtering, query execution
- Depends on: Tokenizer, KindMatcher, LMDB SearchIndex table
- Used by: ReqWorker queries via SearchRunner; Writer indexes new events

## Data Flow

**Event Ingestion Flow:**

1. WebSocket client sends `["EVENT", {...}]` JSON
2. RelayWebsocket dispatches MsgIngester::ClientMessage to Ingester thread pool
3. Ingester parses JSON, validates signature (secp256k1), checks timestamp/size
4. Ingester creates PackedEvent (binary summary: id, pubkey, created_at, kind, tags)
5. Ingester dispatches MsgWriter::AddEvent to Writer thread pool (1 thread, serial)
6. Writer applies write policy plugin filter, deduplicates against Event__id index
7. Writer commits EventToWrite batch to LMDB in RW transaction: stores Event record + EventPayload
8. Writer indexes event in search tables if SearchProvider enabled (SearchIndex, SearchDocMeta)
9. Writer dispatches MsgReqMonitor::DBChange to notify subscriptions
10. ReqMonitor matches new event against live subscriptions, sends EOSE or individual events to clients

**Query Flow (REQ):**

1. Client sends `["REQ", "sub-id", {...filter}]`
2. Ingester validates filter, creates Subscription, dispatches MsgReqWorker::NewSub
3. ReqWorker creates DBQuery, adds to QueryScheduler::running queue
4. ReqWorker.process() iterates through DBQuery in time-sliced chunks (queryTimesliceBudgetMicroseconds)
5. DBQuery scans LMDB indices (Event__id, Event__tag, Event__pubkey, etc.) via DBScan cursors
6. For each matching levId, ReqWorker calls onEvent callback to send event to client
7. If search needed (q filter), SearchProvider::query() returns hits scored by BM25
8. When DBQuery.complete, ReqWorker dispatches EOSE message
9. ReqWorker moves active subscription to ReqMonitor for live updates
10. ReqMonitor watches for new events; resends subscription to ReqWorker if DBChange occurs

**Search Query Flow (NIP-50):**

1. Client sends REQ with "search" filter: `["REQ", "sub-id", {"search": "bitcoin"}]`
2. Ingester parses and validates filter, passes to ReqWorker
3. ReqWorker creates DBQuery with searchProvider set
4. During DBQuery.process(), calls searchProvider->query(SearchQuery) if search filter present
5. LmdbSearchProvider tokenizes query, looks up SearchIndex table for posting lists
6. For each token, reads postings (levId:48 | tf:16), calculates BM25 score
7. Merges posting lists, ranks by BM25, applies limit
8. Returns SearchHit array (levId + score)
9. ReqWorker sends matched events sorted by score to client
10. Events decoded from EventPayload (with optional zstd decompression)

**Negentropy Sync Flow (NIP-77):**

1. Client sends `["NEG-OPEN", "sub-id", filter_json, negentropy_msg]`
2. Ingester decompresses negentropy_msg, dispatches MsgNegentropy::NegOpen
3. Negentropy thread creates negentropy sync session
4. Client sends `["NEG-MSG", "sub-id", negentropy_msg]` with continued protocol exchange
5. Negentropy layer builds efficient range tree (BTreeLMDB for full index, Vector for subset)
6. Negentropy computes minimal diff, sends back optimized payload
7. Client reconstructs missing events; sync completes when both sides have same hash

**State Management:**

- **Thread-local LMDB transactions:** Each worker thread opens RO or RW txn, keeps it alive for message batch
- **Connection state:** RelayServer tracks connId → Connection (websocket ptr, stats, IP)
- **Subscription state:** QueryScheduler.conns[connId][subId] → DBQuery (active) or ReqMonitor (live)
- **Search index state:** SearchState table tracks lastIndexedLevId and indexVersion for catch-up indexing
- **Negentropy state:** NegentropyViews stores in-memory (Vector) or stateless (BTreeLMDB-backed) views per subscription

## Key Abstractions

**RelayServer:**
- Purpose: Central coordinator; owns all thread pools and global state
- Examples: `src/apps/relay/RelayServer.h`
- Pattern: Singleton struct with member thread pools and dispatch methods; each run*() method pulls from inbox queue and processes messages

**ThreadPool<M>:**
- Purpose: Lock-free work distribution to N threads by key hash
- Examples: `src/ThreadPool.h`
- Pattern: Template generic over message type; dispatch(key) hashes to thread; threads call pop_all() to batch messages

**Message Types (MsgWebsocket, MsgIngester, MsgWriter, etc.):**
- Purpose: Type-safe queue messages using std::variant for message union
- Examples: `src/apps/relay/RelayServer.h` (lines 24–147)
- Pattern: Each layer has a Msg* struct with nested message variants (Send, CloseConn, etc.); dispatcher creates variant and pushes to queue

**DBQuery:**
- Purpose: Stateful query execution with resumable cursor position
- Examples: `src/DBQuery.h`
- Pattern: Holds NostrFilter, scan cursors per index, event queue; process() method does time-sliced work, stores resume position if incomplete

**QueryScheduler:**
- Purpose: Queue and round-robin execution of DBQuery objects
- Examples: `src/QueryScheduler.h`
- Pattern: running deque holds active queries; process() runs one query until time budget exhausted, re-queues if incomplete

**ISearchProvider (Strategy Pattern):**
- Purpose: Abstract interface for pluggable search backends
- Examples: `src/search/SearchProvider.h`, `src/search/LmdbSearchProvider.h`, `src/search/NoopSearchProvider.h`
- Pattern: virtual methods indexEvent, deleteEvent, query; LmdbSearchProvider implements BM25 with LMDB tables; NoopSearchProvider is no-op stub

**PackedEvent/PackedEventView:**
- Purpose: Compact binary representation of nostr event (id, pubkey, created_at, kind, tags)
- Examples: `src/PackedEvent.h`
- Pattern: Builder pattern (PackedEventBuilder) for creation; View pattern (PackedEventView) for reading; used as key for deduplication and indexing

**Decompressor:**
- Purpose: Transparent zstd decompression for EventPayload with dictionary
- Examples: `src/Decompressor.h`
- Pattern: Stateful cache of recent dictionaries; decodeEventPayload() checks compression type byte, decompresses if needed

**Golpe (Code Generation):**
- Purpose: Schema and config code generation from YAML
- Examples: `golpe.yaml`
- Pattern: Defines LMDB tables, indices, comparators, config fields; generates `defaultDb::environment` with typed DBI accessors and config struct

## Entry Points

**Main Entry Point:**
- Location: `build/main.cpp`
- Triggers: strfry CLI invocation with subcommand (relay, dbutils cmds, mesh cmds)
- Responsibilities: Parse args, load config, open LMDB env, dispatch to cmd_*() functions

**Relay Entry Point:**
- Location: `src/apps/relay/cmd_relay.cpp::cmd_relay()`
- Triggers: `strfry relay` command
- Responsibilities: Create RelayServer, call run(); initialize thread pools and signal handling

**RelayServer::run():**
- Location: `src/apps/relay/cmd_relay.cpp`
- Triggers: From cmd_relay()
- Responsibilities: Initialize search provider, start all thread pools (WebSocket, Ingester, Writer, ReqWorker, ReqMonitor, Negentropy), start cron and signal handler threads, block on config file monitor

**DBUtils Entry Points:**
- cmd_info, cmd_export, cmd_import, cmd_compact, cmd_delete, cmd_scan, etc. in `src/apps/dbutils/`
- Each operates directly on LMDB environment for maintenance

## Error Handling

**Strategy:** Exceptions propagate; caught at ingester and writer layers; errors sent to client as NOTICE or OK(false)

**Patterns:**
- **Event validation errors:** Caught in ingesterProcessEvent(); returned as sendOKResponse(connId, eventId, false, reason)
- **Filter parsing errors:** Caught in ingesterProcessReq(); returned as sendClosedError(connId, subId, reason) or sendNoticeError
- **Write errors:** Caught in runWriter(); logged, sendOKResponse() to each affected client
- **Search indexing errors:** Caught in Writer event indexing loop; logged but does not fail write; allows graceful degradation

## Cross-Cutting Concerns

**Logging:** loguru (LI, LW, LE macros); verbose levels configurable per subsystem

**Validation:** FilterValidator for filter syntax; Event signature/timestamp validation in ingester; config reloading with syntax check

**Authentication:** None native; write policy plugins can reject by IP/event content; NIP-11 info endpoint provides relay metadata

**Performance:** Time-sliced query execution (queryTimesliceBudgetMicroseconds) prevents blocking; batched message processing; LMDB mmap for fast index scans; zstd compression for payload storage

---

*Architecture analysis: 2026-04-27*
