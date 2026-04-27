# Technology Stack

**Analysis Date:** 2026-04-27

## Languages

**Primary:**
- C++20 - Entire application core (`src/`, `src/apps/`)
- Perl - Build code generation and testing (`golpe/gen-*.pl`, test scripts)

**Secondary:**
- Flatbuffers Schema Language - Data serialization definitions (`golpe.yaml`, generates `*.fbs`)
- TOML/YAML - Configuration files (`golpe.yaml` for build config, `strfry.conf` for runtime)

## Runtime

**Environment:**
- Linux (primary deployment target)
- Docker support via `sample-docker-compose.yaml`
- Nix environment support via `shell.nix`

**Build System:**
- golpe (vendored custom build system in `./golpe/` directory) - Generates code from templates and config
  - Uses Perl-based generators: `gen-main.cpp.pl`, `gen-config.pl`, `gen-golpe.h.pl`, `gen-fbs.pl`
  - Template system via `templar.pl` for HTML/template generation (`src/tmpls/`)
  - FlatBuffers schema generation from `golpe.yaml`

**Compiler & Flags:**
- Standard: `-std=c++20`
- Optimization: `-O3 -g` (configurable via `OPT` variable in `Makefile`)
- Linking: LTO enabled (`-flto`)
- Compilation via standard Makefile pattern (see `golpe/rules.mk`)

## Frameworks

**Core Frameworks:**
- **uWebSockets v0.14.x** (`golpe/external/uWebSockets/`) - WebSocket server for Nostr protocol
  - Provides HTTP/WebSocket server with SSL/TLS support
  - Permessage-deflate compression support
  - Location: `src/WSConnection.h`, `src/apps/relay/RelayServer.h`

**Data Storage & Serialization:**
- **LMDB** (Lightning Memory-Mapped Database) - Primary data store
  - Header-only wrapper via lmdbxx (`golpe/external/lmdbxx/include/`)
  - Databases defined in `golpe.yaml` as structured tables with indices
  - Tables: `Event`, `EventPayload`, `Meta`, `CompressionDictionary`, `SearchIndex`, `SearchDocMeta`, `NegentropyFilter`, `SearchState`
  - Multi-indexed event table with comparators for efficient querying by id, pubkey, kind, tags, expiration, deletion, replacement

- **FlatBuffers** - Event data serialization
  - Config-driven schema generation from `golpe.yaml`
  - Generated code in `build/defaultDb.schema.fbs` and compiled headers in `build/golpe.h`
  - Enables zero-copy access to packed event data via `PackedEventView` pattern

**Cryptography & Signatures:**
- **libsecp256k1** - Schnorr signature verification for Nostr events
  - Used in `src/events.cpp` for `verifySig()` and event verification
  - Linked via `-lsecp256k1` in `Makefile`

**Compression:**
- **zstd (Zstandard)** - Event payload compression
  - Dictionary-based compression for efficient storage
  - Header: `src/Decompressor.h`, implementation: `src/Decompressor.cpp`
  - Uses LMDB raw table `EventPayload` with compression type prefixes
  - Configuration table: `CompressionDictionary`

**JSON Processing:**
- **nlohmann/json** (TAO JSON) - JSON parsing and manipulation
  - Header-only library in `golpe/external/json/include/`
  - Used throughout for Nostr event JSON handling
  - tao::json namespace includes extended JSON features

**Configuration Parsing:**
- **config.h library** (`golpe/external/config/include/`) - Configuration file parsing
  - Generates `build/config.cpp` and `build/config.h` from `golpe.yaml` + `strfry.conf`
  - Runtime hot-reload support via file watchers

**Parsing & Grammars:**
- **PEGTL (Parsing Expression Grammar Template Library)** - Grammar-based parsing
  - Used for filter validation and query parsing
  - Header in `golpe/external/PEGTL/include/`

**Logging:**
- **loguru** - Structured logging
  - Header: `golpe/external/loguru/`
  - Log levels: LI (Info), LW (Warning), LE (Error)
  - Used throughout for debug/perf logging

**Utilities:**
- **hoytech-cpp library** (`golpe/external/hoytech-cpp/`) - C++ utilities
  - Includes: hex encoding/decoding, error handling, timers, protected queues, file change monitoring
  - Used for file watching, time operations, hex conversions
  - File monitoring: `hoytech/file_change_monitor.h` for config reload

- **parallel-hashmap** (`golpe/external/parallel-hashmap/`) - Hash table implementations
  - High-performance hash maps and sets

- **docopt.cpp** (`golpe/external/docopt.cpp/`) - Command-line argument parsing

## Key Dependencies

**Critical (Linked):**
- `-lsecp256k1` - Signature verification (required for event validation)
- `-lzstd` - Compression (used for on-disk event compression)
- `-llmdb` - Database engine (conditional, required by features.db)
- `-lcrypto -lssl` - OpenSSL (conditional, required by features.ssl, uWebSockets TLS)
- `-lz` - zlib (conditional, required by uWebSockets)
- `-ldl` - Dynamic linking support
- `-lpthread` - Thread support

**Header-Only/Built-in:**
- loguru - Logging framework
- json (nlohmann TAO JSON) - JSON handling
- config library - Config parsing
- hoytech-cpp utilities - General utilities
- parallel-hashmap - Hash tables
- docopt.cpp - CLI argument parsing

**External Services/Protocols:**
- **negentropy** (`external/negentropy/`) - Set reconciliation protocol
  - Implements `negentropy::storage::BTreeLMDB` for efficient sync
  - Protocol for efficient event set comparison with clients/relays
  - Used in `src/NegentropyFilterCache.h` and negentropy threads

## Feature Flags (from golpe.yaml)

```
features:
    ssl: true              # TLS support for WebSocket
    config: true           # Runtime configuration system
    onAppStartup: true     # Startup hooks
    onPreStartup: true     # Pre-startup hooks
    db: true               # LMDB database backend
    customLMDBSetup: true  # Custom database initialization
    websockets: true       # WebSocket server
    templar: true          # Template system for HTTP responses
```

## Configuration Files

**Build Configuration:**
- `golpe.yaml` - Golden config file defining LMDB schema, indices, config options, feature flags
- `Makefile` - Entry point for builds (compiles via `golpe/rules.mk`)
- `golpe/rules.mk` - Core build rules (C++ compilation, linking, code generation)

**Runtime Configuration:**
- `strfry.conf` - Default configuration file (TOML/key-value format)
  - Defines relay name/info, database paths, event validation rules, thread pools, search settings
  - Hot-reloadable via file watchers in `cmd_relay.cpp`
  - Config sections: db, events, relay (info, websocket params, compression, search, filters)

**Environment Configuration:**
- `shell.nix` - Nix development environment with dependencies: perl, lmdb, zstd, secp256k1, flatbuffers, zlib, openssl, libuv
- No `.env` file detected - environment variables loaded from system

## Platform Requirements

**Development:**
- C++20 compiler (GCC or Clang)
- GNU Make
- Perl 5 (for golpe code generation)
- LMDB development headers
- secp256k1 library
- zstd compression library
- OpenSSL/libcrypto development headers
- On Nix: Use `shell.nix` which provides all dependencies

**Production:**
- Linux kernel (tested on production systems)
- 2-10TB+ free disk space (LMDB mmap configurable, default 10TB virtual)
- 256+ file descriptors/sockets available
- Network interface for WebSocket binding (default 127.0.0.1:7777)
- Libraries: liblmdb, libsecp256k1, libzstd, libcrypto, libssl
- Optional: reverse proxy (nginx, etc) for external exposure

**Build Output:**
- `strfry` binary - Main relay executable
- `dbutils` binary - Database utility tool (scan, delete, export, import)
- `relay` subcommand - Run relay server
- `mesh` subcommand - Mesh networking (router, stream, sync commands)

---

*Stack analysis: 2026-04-27*
