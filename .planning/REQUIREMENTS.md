# Requirements: react-native-fast-uid

**Defined:** 2026-05-04
**Core Value:** Synchronous, time-sortable, RFC-9562-compliant ID generation from JS, the UI worklet runtime, or anywhere a JSI runtime exists — fast enough that the cost is invisible in the hottest paths a React Native app has.

## v0.1 Requirements

Initial public release (`v0.1.0` on npm). Each requirement maps to exactly one roadmap phase.

### Generation API

- [ ] **API-01**: `uuidv7()` returns a single RFC-9562-compliant UUIDv7 as a 36-char hyphenated **lowercase** ASCII string
- [ ] **API-02**: `uuidv7.batch(n)` returns `string[]` of `n` UUIDv7s; throws `FastUidError(code: "BATCH_TOO_LARGE")` for `n > MAX_BATCH`; returns `[]` for `n === 0`
- [ ] **API-03**: `ulid()` returns a single ULID as a 26-char Crockford base32 **uppercase** ASCII string (alphabet excludes `I`, `L`, `O`, `U`)
- [ ] **API-04**: `ulid.batch(n)` returns `string[]` of `n` ULIDs; same bounds + error semantics as API-02
- [ ] **API-05**: `installOnRuntime(runtime?)` installs bindings on the default JS runtime AND optionally on a Reanimated UI runtime; second call on the same runtime is a no-op (idempotent via sentinel)
- [ ] **API-06**: All four generators are tree-shakeable via named exports — no `export default` object
- [ ] **API-07**: TS exports include opt-in branded types `Uuid` and `Ulid` (`string & { readonly __uuid: unique symbol }`); functions return plain `string` for ergonomic interop

### Native Implementation

- [ ] **NATIVE-01**: iOS uses `SecRandomCopyBytes` (Security framework); return code is checked; `errSecSuccess != 0` raises `FastUidError(code: "CSPRNG_FAILED")` via `jsi::JSError`
- [ ] **NATIVE-02**: Android uses `getrandom(2)` syscall (raw `syscall(__NR_getrandom, ...)` for NDK API-level portability) with EINTR retry loop; falls back to `/dev/urandom` (cached fd via `open(O_CLOEXEC)`) on `EAGAIN`/`GRND_NONBLOCK` failure or pre-API-28 unavailability; both-source failure raises `FastUidError(code: "CSPRNG_FAILED")`
- [ ] **NATIVE-03**: Sub-millisecond monotonic counter implements RFC 9562 §6.2 Method 2: `last_ms` clamped to `max(last_ms, system_clock_ms())` (handles NTP/DST clock-going-backwards); same-ms increments counter; new-ms reseeds counter to a random value with the **high bit clear**; counter overflow advances `last_ms` by 1 ms (does NOT wrap)
- [ ] **NATIVE-04**: A single `std::mutex` serializes counter state; held only over arithmetic — JSI string allocation happens **outside** the lock; batch generation acquires the lock **once per batch**, not per ID
- [ ] **NATIVE-05**: ASCII string allocation via `jsi::String::createFromAscii`; formatter writes only into the ASCII-safe alphabet (UUID hex+dash, Crockford base32); debug-mode invariant asserts every output byte is in range
- [ ] **NATIVE-06**: No per-call global state. The monotonic counter struct is the deliberate exception — a process-static `std::shared_ptr<GeneratorState>` so monotonicity holds across runtime tear-down/recreate cycles (Fast Refresh). The exception is documented in code comments
- [ ] **NATIVE-07**: No exceptions cross the JSI boundary except `jsi::JSError`; every host function is wrapped in `try { } catch (jsi::JSError&) { throw; } catch (std::exception& e) { throw jsi::JSError(rt, ...); } catch (...) { throw jsi::JSError(rt, "INTERNAL"); }`
- [ ] **NATIVE-08**: Batch size cap `MAX_BATCH` (suggested `1 << 17` ≈ 131k); request beyond cap throws `FastUidError(code: "BATCH_TOO_LARGE")` synchronously, before any allocation
- [ ] **NATIVE-09**: Errors expose a stable code-based shape (`FastUidError` with `code` ∈ `"CSPRNG_FAILED" | "RUNTIME_NOT_INSTALLED" | "BATCH_TOO_LARGE" | "INTERNAL"`); user-facing messages do NOT leak `errno` / `OSStatus` — those are logged via `os_log` / `__android_log_print` at error level only

### Build & Packaging

- [ ] **BUILD-01**: Bootstrapped from `create-react-native-library` JSI/C++ template (NOT turbo-module template)
- [ ] **BUILD-02**: iOS podspec sets `s.compiler_flags = '-std=c++17'`, `s.requires_arc = true`, `s.pod_target_xcconfig = { 'CLANG_CXX_LANGUAGE_STANDARD' => 'c++17', 'CLANG_CXX_LIBRARY' => 'libc++' }`, links `Security` framework
- [ ] **BUILD-03**: Android build via CMake (`set(CMAKE_CXX_STANDARD 17)`, `set(CMAKE_CXX_STANDARD_REQUIRED ON)`, `set(CMAKE_CXX_EXTENSIONS OFF)`) + NDK r26+; produces `.so` per ABI for `arm64-v8a, armeabi-v7a, x86, x86_64`
- [ ] **BUILD-04**: New-architecture only — `peerDependencies.react-native >= 0.76`; old-arch explicitly unsupported with a clear init-time error message; no Bridge code paths
- [ ] **BUILD-05**: Internal C++ release builds compile with `-Wall -Wextra -Werror` via an opt-in `WARNINGS_AS_ERRORS` CMake option (default OFF); `-Werror` does NOT propagate to consumer pod builds
- [ ] **BUILD-06**: No Reanimated headers included in C++; worklet install path receives the `jsi::Runtime*` from JS-land. Reanimated declared as `peerDependenciesMeta.optional` so consumers without it still install
- [ ] **BUILD-07**: Single `cpp/` tree at repo root; iOS podspec source globs `"../cpp/**/*"`; Android `CMakeLists.txt` globs `cpp/**/*.cpp` — no per-platform source duplication
- [ ] **BUILD-08**: Android ships `consumer-rules.pro` (or `consumerProguardFiles`) keeping JSI-registration symbols so consumer release builds with R8 don't strip them — no `UnsatisfiedLinkError`

### Quality & Compliance

- [ ] **QA-01**: An independent RFC-9562 validator runs in the test suite — generated UUIDv7s pass; RFC test vectors from the appendix pass
- [ ] **QA-02**: 10M-monotonicity stress test runs in CI: generate 10M IDs in a single batch, assert lex-sort matches insertion order; includes a clock-fault-injection variant (mocked clock stepping backwards) and a counter-near-overflow variant
- [ ] **QA-03**: Worklet test — IDs generated from a Reanimated worklet (without any prior JS-thread call) are valid UUIDv7s; runs in the example app and is enforceable in CI
- [ ] **QA-04**: TypeScript strict mode enabled; zero `any` types in shipped code OR test files (`tsc --strict --noEmit` on full repo)
- [ ] **QA-05**: Every exported TS symbol has TSDoc
- [ ] **QA-06**: Biome (TS) + clang-format (C++) applied; no `console.log` in shipped code (Biome lint rule + grep before publish)
- [ ] **QA-07**: Concurrency stress test — JS thread + Reanimated worklet generate IDs in parallel for ≥30 seconds; no crash, no duplicates within a runtime, each runtime's stream is monotonic
- [ ] **QA-08**: Standalone C++ test harness in `cpp/tests/` (recommended: Catch2 single-header) running RFC-9562 conformance + monotonicity tests without RN — fast iteration during Phase 2

### CI

- [ ] **CI-01**: GitHub Actions runs TypeScript check (`tsc --noEmit`), Biome lint, Jest tests on every push/PR
- [ ] **CI-02**: GitHub Actions runs iOS `pod install` + `xcodebuild` against the example app on `macos-15` (Xcode 16) runner
- [ ] **CI-03**: GitHub Actions runs Android `assembleDebug` + `assembleRelease` (release matters — release-mode optimizations affect benchmark numbers); pod cache key includes `Podfile.lock` hash to prevent stale-cache replay
- [ ] **CI-04**: arm64 CI job (Apple Silicon GH Actions runner Android emulator OR equivalent) catches alignment/ABI bugs only real arm64 devices see
- [ ] **CI-05**: Consumer-app smoke build — separate CI job creates a fresh RN app, installs the published tarball (`npm pack`), builds for both platforms, runs a `uuidv7()` call asserting valid output. Catches build/packaging pitfalls that pass on the library's own CI but fail at users
- [ ] **CI-06**: Benchmark job runs in **release mode only** (Pitfall 15); fails if numbers regress >20% from baseline

### Example App

- [ ] **EX-01**: `example/` demonstrates single-call generation (`uuidv7()`, `ulid()`)
- [ ] **EX-02**: `example/` demonstrates batch generation (`uuidv7.batch(n)`, `ulid.batch(n)`)
- [ ] **EX-03**: `example/` verifies monotonic ordering — generates 1M IDs and asserts they sort in insertion order
- [ ] **EX-04**: `example/` demonstrates Reanimated worklet usage — IDs generated inside a worklet without `runOnJS`

### Docs & Release

- [ ] **DOC-01**: README's first screen has install command, three-line minimal example, supported platforms (iOS/Android, new-arch only, Hermes/JSC), micro-benchmark numbers vs `uuid` and `react-native-uuid` measured in **release builds** (sub-30s comprehension test)
- [ ] **DOC-02**: `docs/benchmarks.md` documents reproducible methodology — device specs, RN version, Hermes/JSC, sample sizes, p50/p95/p99 (not just mean), exact bench script committed; release-mode-only; numbers in README must be regenerable from the benchmark script
- [ ] **DOC-03**: `CHANGELOG.md`, `LICENSE` (MIT), `CONTRIBUTING.md` present
- [ ] **DOC-04**: README includes a migration table — from `uuid.v4()`, `crypto.randomUUID()`, `react-native-uuid` — making the drop-in story explicit
- [ ] **DOC-05**: README documents the error-format contract (`FastUidError` with `code`) so consumers can pattern-match without breaking on message changes
- [ ] **DOC-06**: README documents Expo behavior — Expo Go is not supported (JSI native module), Expo Dev Client works
- [ ] **REL-01**: Published to npm at version `0.1.0`; `files` whitelist set; `prepublishOnly` runs build + typecheck + tests; published with `npm publish --provenance --access public` from GitHub Actions
- [ ] **REL-02**: GitHub release `v0.1.0` tagged; release notes generated from conventional commits via `release-it` + `@release-it/conventional-changelog`
- [ ] **REL-03**: Submitted to react-native-directory with `newArchitecture: true`, `expoGo: false`, `devClient: true` (verify schema at submission); Reanimated declared as optional peer

## v0.2+ Requirements (deferred)

Acknowledged but not in current roadmap. Triggers (≥3 issues asking, etc.) advance items to `v0.x+1` Active.

### Bytes API

- **BYTES-01**: `uuidv7.bytes(): Uint8Array` returning the raw 16-byte UUIDv7
- **BYTES-02**: `ulid.bytes(): Uint8Array` returning the raw 16-byte ULID
- **BYTES-03**: `uuidv7.batchBytes(n)` / `ulid.batchBytes(n)` returning a packed `Uint8Array` of `n × 16` bytes

### Vendoring

- **VENDOR-01**: README documents `cpp/core/` as header-only-vendorable into other JSI libraries

### Platform Expansion

- **PLAT-01**: macOS support (verify `SecRandomCopyBytes` + JSI parity)
- **PLAT-02**: Windows / Web platforms (only with user demand + tested matrix)

## Out of Scope

Explicit exclusions. Anti-features from research belong here with reasoning to prevent re-adding.

| Feature | Reason |
|---------|--------|
| UUID v1, v3, v4, v5 | Covered by `react-native-uuid` and `crypto.randomUUID()`; this lib is v7 + ULID only |
| `crypto.getRandomValues` polyfill | Owned by `react-native-get-random-values`; the native CSPRNG path replaces it |
| Async API (`await uuidv7()`) | Defeats the JSI purpose — sync is the whole point |
| Old-architecture support | New-arch only; reduces surface area; matches RN's direction |
| Timestamp injection / mocking in public API | Always uses system clock; test-mode injection is internal-only |
| String parsing / validation utilities | Generation only; parsing is a separate concern (defer to a future companion package) |
| UUIDv6 | Deprecated draft |
| UUIDv8 / custom variant generation | Out of scope for v0.1; conflicts with "pure generator" framing |
| Collision detection or registry | Dataset concern, not a library concern |
| `Math.random()` fallback when CSPRNG fails | Silent CVE; RFC 9562 §6.9 mandates cryptographic random; **never** fall back |
| Returning `null` on CSPRNG failure | `string \| null` infects every caller; `FastUidError` throw is correct fail-loud behavior |
| Stripe-style prefixes (`uuidv7({ prefix: 'usr_' })`) | A prefixed UUID isn't a UUID; breaks Postgres `uuid` columns and validators; `` `usr_${uuidv7()}` `` in user code is correct |
| `uuidv7.timestamp(uuid)` introspection | Parsing concern; out of scope (DOC-04 links the bit layout for users who need it) |
| Custom epoch | RFC 9562 §5.7 fixes the epoch as Unix ms; custom epoch breaks interop |
| Lowercase ULID / RFC 4648 base32 alphabet | ULID spec is Crockford uppercase; lowercase doesn't round-trip canonical parsers; RFC 4648 is a different alphabet |
| Pre-allocated buffer for `batch(n, buffer)` | Forces JS-side encoding — slower than `createFromAscii` directly; `string[]` is correct |
| Custom alphabet for ULID | ULID spec is Crockford-only; use `nanoid` for custom alphabets |
| Configurable monotonicity strictness | Method 2 has no measurable cost vs non-monotonic; always-on is the correct default |
| HostObject install | No state to hold at the host-object level; `createFromHostFunction` is the right primitive |
| Bridge / TurboModule codegen as the public API | Pure JSI install is the design; codegen spec is internal plumbing only |

## Traceability

Empty initially; populated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| API-01      | TBD   | Pending |
| API-02      | TBD   | Pending |
| API-03      | TBD   | Pending |
| API-04      | TBD   | Pending |
| API-05      | TBD   | Pending |
| API-06      | TBD   | Pending |
| API-07      | TBD   | Pending |
| NATIVE-01   | TBD   | Pending |
| NATIVE-02   | TBD   | Pending |
| NATIVE-03   | TBD   | Pending |
| NATIVE-04   | TBD   | Pending |
| NATIVE-05   | TBD   | Pending |
| NATIVE-06   | TBD   | Pending |
| NATIVE-07   | TBD   | Pending |
| NATIVE-08   | TBD   | Pending |
| NATIVE-09   | TBD   | Pending |
| BUILD-01    | TBD   | Pending |
| BUILD-02    | TBD   | Pending |
| BUILD-03    | TBD   | Pending |
| BUILD-04    | TBD   | Pending |
| BUILD-05    | TBD   | Pending |
| BUILD-06    | TBD   | Pending |
| BUILD-07    | TBD   | Pending |
| BUILD-08    | TBD   | Pending |
| QA-01       | TBD   | Pending |
| QA-02       | TBD   | Pending |
| QA-03       | TBD   | Pending |
| QA-04       | TBD   | Pending |
| QA-05       | TBD   | Pending |
| QA-06       | TBD   | Pending |
| QA-07       | TBD   | Pending |
| QA-08       | TBD   | Pending |
| CI-01       | TBD   | Pending |
| CI-02       | TBD   | Pending |
| CI-03       | TBD   | Pending |
| CI-04       | TBD   | Pending |
| CI-05       | TBD   | Pending |
| CI-06       | TBD   | Pending |
| EX-01       | TBD   | Pending |
| EX-02       | TBD   | Pending |
| EX-03       | TBD   | Pending |
| EX-04       | TBD   | Pending |
| DOC-01      | TBD   | Pending |
| DOC-02      | TBD   | Pending |
| DOC-03      | TBD   | Pending |
| DOC-04      | TBD   | Pending |
| DOC-05      | TBD   | Pending |
| DOC-06      | TBD   | Pending |
| REL-01      | TBD   | Pending |
| REL-02      | TBD   | Pending |
| REL-03      | TBD   | Pending |

**Coverage:**
- v0.1 requirements: 50 total
- Mapped to phases: 0 (filled by roadmap)
- Unmapped: 50 ⚠️ (resolves when roadmap is created)

---
*Requirements defined: 2026-05-04*
*Last updated: 2026-05-04 after initial definition*
