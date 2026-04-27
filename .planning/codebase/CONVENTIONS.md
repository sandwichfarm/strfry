# Coding Conventions

**Analysis Date:** 2026-04-27

## Naming Patterns

**Files:**
- PascalCase for class/type headers: `WSConnection.h`, `Decompressor.h`, `Subscription.h`
- Directories use lowercase: `src/search/`, `src/apps/relay/`
- Mixed: headers follow class names (`Bytes32.h`, `PrometheusMetrics.h`)

**Functions:**
- camelCase for functions: `renderIP()`, `parseUint64()`, `getDBVersion()`, `makeSearchProvider()`
- Member functions follow camelCase: `sv()`, `str()`, `decompress()`, `getDict()`
- Inline helper lambdas common in function bodies: `auto digitChar = [](char c){ ... }`

**Variables:**
- camelCase for local/member variables: `currWs`, `hubGroup`, `hubTrigger`, `dctx`, `dictId`
- SCREAMING_SNAKE_CASE for constants: `MAX_U64`, `SHA256_DIGEST_LENGTH`, `SO_KEEPALIVE`, `MAX_INDEXED_TAG_VAL_SIZE`
- Underscore prefix for private members rare; visibility controlled by access modifiers
- Member variables grouped by public/private sections with explicit comments

**Types:**
- PascalCase for classes/structs: `WSConnection`, `DictionaryBroker`, `Decompressor`, `SubId`
- Scoped enums (enum class) with PascalCase names: `enum class EventSourceType`, `enum class EventWriteStatus`

## Code Style

**Formatting:**
- Tool: Clang-format (config in `golpe/external/json/.clang-format`)
- Key settings:
  - IndentWidth: 3 spaces
  - TabWidth: 8 (but UseTab: Never)
  - ColumnLimit: 0 (no line length limit enforced)
  - PointerAlignment: Left (pointers/refs attached to type)
  - BraceWrapping: After class/function/namespace/struct/enum/extern block (custom)
  - AllowShortFunctionsOnASingleLine: Empty (only empty functions)
  - AlwaysBreakTemplateDeclarations: Yes
  - BreakBeforeBinaryOperators: All

**Linting:**
- No centralized linting config found in main repo (clang-tidy configs exist in `golpe/external/` subdependencies)
- Code appears manually reviewed rather than enforced by CI lint step

## Import Organization

**Order:**
1. System headers (`#include <stdint.h>`, `#include <string>`, `#include <algorithm>`)
2. External libraries (`#include <secp256k1_schnorrsig.h>`, `#include <lmdb.h>`, `#include <zstd.h>`)
3. Local project headers (`#include "golpe.h"`, `#include "events.h"`)
4. Submodule dependencies (`#include "flatbuffers/flatbuffers.h"`)

**Example from `events.cpp`:**
```cpp
#include <openssl/sha.h>
#include <negentropy.h>

#include "events.h"
#include "jsonParseUtils.h"
#include "search/SearchProvider.h"
```

**Path Aliases:**
- No CMake/Bazel aliases observed
- Relative includes: `#include "Bytes32.h"`, `#include "search/SearchProvider.h"`
- Subdir includes: `#include "golpe.h"` from `build/golpe.h`
- External includes use absolute paths: `#include <parallel_hashmap/phmap.h>`

## Error Handling

**Patterns:**
- Exceptions via `herr` macro (alias for `hoytech::error`): `throw herr("message")`, `throw herr("prefix: ", value)`
- Variable-argument error messages: `throw herr("tag val too large: ", tagVal.size())`
- Error messages descriptive, include context (field name, size, actual value)
- No error codes; pure exception-based error flow
- Example from `events.cpp`:
  ```cpp
  if (id.size() != 32) throw herr("unexpected id size");
  if (sig.size() != 64 || hash.size() != 32 || pubkey.size() != 32) 
      throw herr("verify sig: bad input size");
  ```
- `noexcept` not used; functions propagate exceptions up

**Scope:**
- Validation happens early in function entry (defensive checks)
- State-changing operations only after all validation passes
- Example: `parseAndVerifyEvent()` validates shape, signature, timestamp before building JSON

## Logging

**Framework:** Loguru (header-only library)

**Setup:** Included as `#define` macros in `golpe.h`:
```cpp
#define LE LOG_S(ERROR)      // Error level
#define LW LOG_S(WARNING)    // Warning level  
#define LI LOG_S(INFO)       // Info level
```

**Patterns:**
- Stream-based: `LI << "message: " << value << " other: " << other`
- Info level for startup: `LI << "Connected to " << url << " (" << remoteAddr << ")"`
- Warnings for config/migration: `LW << "Unknown search backend: " << backend << ", falling back to lmdb"`
- Errors for unrecoverable states: `LE << "Database version too old: " << ver`
- Thread naming helper: `setThreadName(value)` wrapper around loguru

**Example from `onAppStartup.cpp`:**
```cpp
LE << "Database version too old: " << ver << ". Expected version " << CURR_DB_VERSION;
LE << "You should 'strfry export' your events, delete (or move) the DB files, and 'strfry import' them";
throw herr("aborting: DB too old");
```

## Comments

**When to Comment:**
- Sparse; code is generally self-documenting
- Non-obvious algorithms: See `renderSize()` for unit conversion do-while pattern
- Intent clarification: "Prepend virtual d-tag", "Append virtual d-tag" before special tag handling
- Thread-safety notes in comments: "Should only be called from the websocket thread"
- Memory lifetime notes: "Return result only valid until one of: ..."

**JSDoc/TSDoc:**
- Not used; C++ style
- Function signatures document parameters implicitly through types (`std::string_view`, const refs)

## Function Design

**Size:**
- Most functions 10-50 lines
- Large functions broken into logical stages with comments
- Example: `nostrJsonToPackedEvent()` in `events.cpp` has clear extraction, validation, and builder stages

**Parameters:**
- Prefer `std::string_view` for read-only string data (no copy)
- Pass large structs/objects by const reference
- Use `lmdb::txn &` for database transactions (always passed by reference)
- Output parameters when multiple returns needed (e.g., `std::string &packedStr, std::string &jsonStr`)

**Return Values:**
- Value returns for small types (uint64_t, bool, Bytes32)
- `std::string_view` for borrowed data with lifetime constraints
- `std::unique_ptr` for owned objects: `std::make_unique<ISearchProvider>`
- `std::optional` for nullable lookups: `auto s = env.lookup_Meta(txn, 1); if (s) { ... }`
- Example: `lookupEventById()` returns `std::optional<defaultDb::environment::View_Event>`

## Module Design

**Exports:**
- Headers declare public API; implementations in `.cpp`
- Example: `events.h` declares `nostrJsonToPackedEvent()`, `verifyNostrEvent()`, etc.
- Functions are exported via forward declarations in headers
- No namespace wrapping (global scope, or scoped via file)

**Barrel Files:**
- None observed
- Each feature area has own header(s): `search/SearchProvider.h`, `search/LmdbSearchProvider.h`

**Struct vs Class:**
- Structs used for data containers with inline logic: `SubId`, `DictionaryBroker`, `Decompressor`
- Classes with private state and access control: `WSConnection : NonCopyable`
- Explicitly deleted copy constructors for non-copyable types via `struct NonCopyable` base

**Example NonCopyable pattern from `golpe.h`:**
```cpp
struct NonCopyable {
    NonCopyable & operator=(const NonCopyable&) = delete;
    NonCopyable(const NonCopyable&) = delete;
    NonCopyable(NonCopyable&&) = default;
    NonCopyable() = default;
};
```

## Type Conventions

**Smart Pointers:**
- `std::unique_ptr` for exclusive ownership: `std::make_unique<NoopSearchProvider>()`
- Raw pointers for non-owning references: `uWS::WebSocket<uWS::CLIENT> *currWs = nullptr`
- Lambdas with captures for callbacks: `[&](uint64_t delay = 0){ ... }`

**Collections:**
- `std::vector<T>` for sequences
- `flat_hash_map` (parallel_hashmap) for O(1) lookups
- `phmap::btree_*` for ordered data
- `std::string` for owned text; `std::string_view` for borrowed

**Const Correctness:**
- Methods: `const` on read-only members (e.g., `sv() const`, `str() const`)
- Parameters: const references `const std::string &`, const views `std::string_view`
- Return values: const or value; no const pointers in public API

---

*Convention analysis: 2026-04-27*
