# Roadmap: react-native-fast-uid

## Overview

Seven phases take this library from empty repo to `v0.1.0` on npm. The ordering is dictated by the three highest-leverage risks: **(1)** RFC-9562 Method-2 monotonicity is the highest correctness risk and must be test-first against a standalone C++ harness before any JSI noise enters; **(2)** Reanimated's UI worklet runtime is a separate `jsi::Runtime` and the runtime-agnostic install design must exist at the FIRST host function, not retrofitted; **(3)** mutex granularity dominates batch performance and lock-once-per-batch must be in place before any benchmark numbers are published. The flow is: scaffold + ratify locked decisions → generator core (math first) → JSI plumbing (idempotency, error wrapping, lock-once-per-batch, ASCII alloc baked in once) → native glue (iOS first, Android second) → TS surface + worklet runtime install (the second-highest risk after monotonicity) → example app + release-mode benchmarks → CI hardening + consumer smoke build + docs + npm publish + directory submission. Phase 1 also locks in the eight ratification gaps from SUMMARY.md, the ten Phase-1 surfaced gaps from ARCHITECTURE.md, and the `[VERIFY]` version-pin reconciliation from STACK.md so that every subsequent phase is executing against a fully-locked design.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Scaffold + ratify locked decisions** — Bootstrap `create-react-native-library` JSI/C++ template, verify `[VERIFY]` version pins, lock the eight SUMMARY.md ratification gaps and ten ARCHITECTURE.md Phase-1 gaps in code-form
- [ ] **Phase 2: Generator core + standalone C++ test harness** — RFC-9562 §6.2 Method-2 monotonic counter, ULID Crockford encode, iOS+Android CSPRNG adapters; test-first against an independent RFC-9562 validator and 10M monotonicity stress in a Catch2 harness with no RN dependency
- [ ] **Phase 3: JSI bindings + runtime-agnostic install + error model** — Host functions for `uuidv7`/`ulid` and their batches, `installBindings(jsi::Runtime&)` with idempotency sentinel, lock-once-per-batch critical section, `FastUidError` code-based error model, exception-translation wrapper applied uniformly
- [ ] **Phase 4: iOS + Android native glue** — TurboModule shims invoking `installBindings`; iOS podspec globbing `cpp/**` with `Security.framework` linkage; Android CMake + JNI + `consumer-rules.pro`; both platforms green in `assembleRelease` / Release scheme
- [ ] **Phase 5: TypeScript API + Reanimated worklet runtime install** — Public surface (`uuidv7`, `uuidv7.batch`, `ulid`, `ulid.batch`, `installOnRuntime`), tree-shakeable named exports, opt-in branded `Uuid`/`Ulid` types, QA-03 worklet test as a gate, QA-07 concurrency stress
- [ ] **Phase 6: Example app + release-mode benchmarks** — Example demonstrates four user workloads (single, batch, monotonicity verification, worklet); reproducible mitata benchmarks vs `uuid`, `react-native-uuid`, `uuidv7` in **release** builds only; `docs/benchmarks.md` documents methodology with p50/p95/p99
- [ ] **Phase 7: CI hardening, consumer smoke build, docs & release** — GitHub Actions matrix (TS/lint/Jest, iOS pod+xcodebuild, Android `assembleDebug`+`assembleRelease`, arm64 job, regression-gated benchmark, **consumer-app smoke build from `npm pack` tarball**), README/CHANGELOG/LICENSE/CONTRIBUTING, `npm publish --provenance`, GitHub release `v0.1.0`, react-native-directory submission

## Phase Details

### Phase 1: Scaffold + ratify locked decisions
**Goal**: A clean repo scaffolded from `create-react-native-library` JSI/C++ template with example app building green on both platforms in Debug AND Release; every `[VERIFY]` version pin reconciled to ground truth; every architectural ratification gap closed with a written decision in PROJECT.md or a code-level invariant in `cpp/`.
**Depends on**: Nothing (first phase)
**Requirements**: BUILD-01, BUILD-02, BUILD-03, BUILD-04, BUILD-05, BUILD-07, QA-04, QA-06
**Success Criteria** (what must be TRUE):
  1. `pnpm install` succeeds from a fresh clone; `pnpm typecheck` (TS strict, zero `any`) and `pnpm lint` (Biome + clang-format) both pass on the empty scaffold
  2. The example app builds and launches on both iOS (`xcodebuild` Release scheme on `macos-15`/Xcode 16) and Android (`./gradlew assembleRelease` with NDK r26+, CMake 3.22, AGP 8.6+, JDK 17) — no Bridge/old-arch code paths anywhere
  3. STACK.md `[VERIFY]` markers are all replaced with concrete pinned versions in `package.json`, `Gemfile`, `android/build.gradle`, and `Podfile.lock` matching what `create-react-native-library` actually scaffolded
  4. The eight SUMMARY.md "Decisions Surfaced for Ratification" (counter overflow strategy, `FastUidError` code shape, `installOnRuntime` idempotency, branded TS types, separate-counters/shared-mutex for ULID, batch cap `1<<17`, `installOnRuntime` API name, process-static counter as documented exception to NATIVE-06) are added to PROJECT.md "Key Decisions" with `Outcome: Locked`
  5. The ten ARCHITECTURE.md "Surfaced Gaps" (idempotency-of-install, batch size cap, worklet install API name, Reanimated peer-dep version, codegen spec name, CMakeLists.txt location, standalone test harness location, CSPRNG error logging policy, counter-overflow fallback, Hermes-vs-JSC string-cost expectation) are answered with concrete decisions written into ARCHITECTURE.md or PROJECT.md
**Plans**: TBD

### Phase 2: Generator core + standalone C++ test harness
**Goal**: A pure-C++17 generator core (RFC-9562 §6.2 Method-2 UUIDv7 + Crockford-base32 ULID) plus iOS/Android CSPRNG adapters, all with **zero JSI dependency**, validated by a Catch2 single-header standalone test harness that runs in seconds without RN. Math is correct before any JSI wiring exists.
**Depends on**: Phase 1
**Requirements**: NATIVE-01, NATIVE-02, NATIVE-03, QA-01, QA-02, QA-08
**Success Criteria** (what must be TRUE):
  1. An independent RFC-9562 validator (not our own parser — circular) confirms 10M generated UUIDv7s are spec-compliant; RFC-9562 appendix test vectors pass
  2. 10M-ID single-batch monotonicity stress (`cpp/tests/`) lex-sorts equal to insertion order; clock-fault-injection variant (mocked clock stepping backwards via NTP/DST simulation) and counter-near-overflow variant both pass — counter overflow advances `last_ms` by 1 ms (does not wrap), reseed-on-new-ms uses **high-bit-clear** random
  3. `cpp/platform/rng_apple.mm` (`SecRandomCopyBytes` with `errSecSuccess` check + zero-init buffer) and `cpp/platform/rng_android.cpp` (raw `syscall(__NR_getrandom, ...)` with EINTR retry loop and `/dev/urandom` cached-fd `open(O_CLOEXEC)` fallback for `EAGAIN`/pre-API-28) both produce 16 cryptographically-random bytes; CSPRNG-failure injection produces a categorized error (raw `errno`/`OSStatus` logged via `os_log`/`__android_log_print`, never returned)
  4. `cpp/core/` builds in a free-standing CMake target without any RN/JSI/Reanimated headers — `cmake -B build && cmake --build build && ./build/fastuid_tests` runs the full suite locally and in CI in under 10 seconds
  5. Standalone tests pass under both `-O0` and `-O2` (Address Sanitizer + Undefined Behavior Sanitizer optionally enabled in a CI matrix slot)
**Plans**: TBD

### Phase 3: JSI bindings + runtime-agnostic install + error model
**Goal**: The four host functions (`uuidv7`, `uuidv7.batch`, `ulid`, `ulid.batch`) wrapped around the generator core via `jsi::Function::createFromHostFunction`, installed by a single runtime-agnostic `installBindings(jsi::Runtime&)` entry that is idempotent, lock-once-per-batch, ASCII-allocating, and exception-safe. Every JSI invariant the project depends on (no captured `Runtime&`, no cached `jsi::Value`, `FastUidError` code-shape, batch-size cap, process-static counter state across reload) lives at this seam — established once, applied uniformly.
**Depends on**: Phase 2
**Requirements**: API-01, API-02, API-03, API-04, NATIVE-04, NATIVE-05, NATIVE-06, NATIVE-07, NATIVE-08, NATIVE-09
**Success Criteria** (what must be TRUE):
  1. Calling `installBindings(rt)` twice on the same runtime is a no-op (sentinel check on `__FastUid_installed__`); the monotonic counter is a process-static `std::shared_ptr<GeneratorState>` so monotonicity holds across Fast Refresh / reload cycles — verified by reload-loop test (10× reload then call generator, IDs strictly greater than last pre-reload ID)
  2. A 1000-ID batch acquires the state mutex **exactly once** (verified by counter on a wrapped mock mutex in a unit test) — `jsi::String::createFromAscii` allocations happen outside the critical section; per-call `uuidv7()` and per-call `ulid()` each acquire exactly one lock
  3. Every host function is wrapped in a `try { } catch (jsi::JSError&) { throw; } catch (std::exception& e) { throw jsi::JSError(rt, ...); } catch (...) { throw jsi::JSError(rt, "INTERNAL"); }` boundary; injecting `std::bad_alloc` produces a JS `FastUidError` with `code: "INTERNAL"` (not a native crash)
  4. `batch(n)` with `n > MAX_BATCH` (1<<17 ≈ 131k) throws `FastUidError(code: "BATCH_TOO_LARGE")` synchronously **before** any allocation; `batch(0)` returns `[]`; `batch(Number.MAX_SAFE_INTEGER)` does not OOM the process
  5. CSPRNG-failure injection at the binding layer surfaces as `FastUidError(code: "CSPRNG_FAILED")` in JS with no `errno`/`OSStatus` value leaked in the user-facing message; the raw value is logged via `os_log`/`__android_log_print` at error level
**Plans**: TBD

### Phase 4: iOS + Android native glue
**Goal**: The C++ install entry called from each platform's TurboModule shim, with podspec/CMake correctly configured to glob the single `cpp/` tree, link `Security.framework` on iOS, ship `consumer-rules.pro` keeping JSI registration symbols on Android. Both platforms produce a green release build of the example app, exercising the full single-call path from JS → host function → generator → ASCII string.
**Depends on**: Phase 3
**Requirements**: BUILD-06, BUILD-08
**Success Criteria** (what must be TRUE):
  1. iOS `FastUid.mm` (< 30 lines) is a thin TurboModule shim invoking `fastuid::installBindings(rt)`; podspec sets `s.compiler_flags = '-std=c++17'` AND `s.pod_target_xcconfig = { 'CLANG_CXX_LANGUAGE_STANDARD' => 'c++17', 'CLANG_CXX_LIBRARY' => 'libc++' }`, links `Security`, globs `"../cpp/**/*"`; example app `xcodebuild` Release scheme builds clean and `uuidv7()` returns a valid UUIDv7 from JS thread
  2. Android `JNI.cpp` + `FastUidModule.kt` invoke the same `installBindings(rt)`; `CMakeLists.txt` (at `android/src/main/cpp/`) globs `../../../../cpp/**/*.cpp` with `set(CMAKE_CXX_STANDARD 17)`, `set(CMAKE_CXX_EXTENSIONS OFF)`; produces `.so` for `arm64-v8a, armeabi-v7a, x86, x86_64`
  3. `./gradlew assembleRelease` on the example app succeeds with R8/ProGuard enabled; `consumer-rules.pro` (or `consumerProguardFiles`) keeps JSI-registration symbols so consumer release builds don't strip them — verified by calling `uuidv7()` in the release-built example app and asserting valid output (no `UnsatisfiedLinkError`)
  4. No Reanimated headers are included in `cpp/`; the example app builds successfully with Reanimated **uninstalled** (Reanimated declared as `peerDependenciesMeta.optional`); `-Werror` does NOT propagate to consumer pod builds (scoped to internal `WARNINGS_AS_ERRORS` CMake option, default OFF)
  5. Both platforms surface CSPRNG failures, batch-size errors, and runtime-not-installed errors as `FastUidError` with the documented `code` values — same shape on iOS and Android
**Plans**: TBD
**UI hint**: yes

### Phase 5: TypeScript API + Reanimated worklet runtime install
**Goal**: The user-facing TypeScript surface — typed re-exports of `globalThis.__FastUid_*`, `installOnRuntime(runtime?)` for the Reanimated UI worklet runtime, opt-in branded `Uuid`/`Ulid` types, tree-shakeable named exports, full TSDoc — plus the QA-03 worklet gate proving that the dual-runtime design from Phase 3 actually delivers IDs inside a Reanimated worklet without any prior JS-thread call. This phase is the second-highest risk after Phase 2 because Reanimated's runtime install API has churned across 3.x → 4.x; if the dual-runtime design is wrong, this is where it breaks.
**Depends on**: Phase 4
**Requirements**: API-05, API-06, API-07, QA-03, QA-05, QA-07
**Success Criteria** (what must be TRUE):
  1. Consumers `import { uuidv7, ulid, installOnRuntime } from 'react-native-fast-uid'` — named exports only (no `export default` object); Metro tree-shaking drops unused exports verified in the example app's release bundle; every exported symbol carries TSDoc
  2. `installOnRuntime()` (no-arg form) installs on the default JS runtime; `installOnRuntime(workletRuntime)` installs on a Reanimated worklet runtime — second call on the same runtime is a no-op (idempotent via Phase 3 sentinel); auto-install at module load runs once on the default runtime
  3. **QA-03 gate**: a Reanimated worklet calls `uuidv7()` inside `useAnimatedReaction` / `runOnUI` (or current Reanimated equivalent) **without any prior JS-thread call to `uuidv7`** — the worklet returns a valid UUIDv7. If this fails, the dual-runtime design is wrong and Phase 3 must be revisited
  4. **QA-07 concurrency stress**: JS thread + Reanimated worklet generate IDs in parallel for ≥30 seconds — no crash, no duplicates within a runtime, each runtime's stream is independently monotonic; cross-runtime ordering is undefined and documented as such
  5. TS strict mode (`tsc --strict --noEmit`) reports zero errors and zero `any` types across `src/` AND test files; opt-in branded types `Uuid` and `Ulid` (`string & { readonly __uuid: unique symbol }`) are exported, but `uuidv7()` and `ulid()` return plain `string` for ergonomic interop
**Plans**: TBD

### Phase 6: Example app + release-mode benchmarks
**Goal**: An example app demonstrating the four user workloads (single-call, batch, 1M-monotonicity verification, Reanimated worklet) plus a reproducible mitata benchmark suite that compares this library against `uuid`, `react-native-uuid`, and `uuidv7` in **release builds only**. Numbers in the README's first screen will come from this benchmark run; publishing release-mode-locked numbers from a Phase-3-ratified lock-once-per-batch implementation establishes the right baseline before any user sees a number.
**Depends on**: Phase 5
**Requirements**: EX-01, EX-02, EX-03, EX-04, DOC-02
**Success Criteria** (what must be TRUE):
  1. `example/` app has four demo screens: single-call (`uuidv7()`, `ulid()`), batch (`uuidv7.batch(n)`, `ulid.batch(n)`), 1M-monotonicity verification (generates 1M IDs, asserts `[...ids].sort()` deep-equals `ids`), Reanimated worklet generation (IDs created inside a worklet without `runOnJS`, displayed via `useDerivedValue`)
  2. Benchmarks run inside the example app via a benchmark screen (mitata, `^1.0.x`); comparison baselines (`uuid@^9`, `react-native-uuid@^2`, `uuidv7@^1`) are pinned in the bench script and printed in the output table
  3. **All published numbers are release-mode**: bench script invokes `Release` scheme on iOS and `assembleRelease` on Android; Debug-mode runs are clearly labeled "for development only — do not cite"
  4. `docs/benchmarks.md` documents reproducible methodology — device model, OS version, RN version, JS engine (Hermes/JSC), sample size, p50/p95/p99 (not just mean), exact bench script committed; numbers in `README.md` are regenerable by running the bench script and match within tolerance
  5. The 1M-monotonicity demo (EX-03) runs in the example app in under 5 seconds on a current iPhone/Pixel and reports zero out-of-order IDs
**Plans**: TBD
**UI hint**: yes

### Phase 7: CI hardening, consumer smoke build, docs & release
**Goal**: Ship `v0.1.0` to npm with confidence. GitHub Actions enforces TS+lint+Jest, iOS+Android builds (Debug AND Release), arm64 emulator, release-mode benchmark regression-gated at >20%, AND a consumer-app smoke build that installs the `npm pack` tarball into a fresh RN app — the single highest-leverage CI job because it catches build/packaging pitfalls only consumer projects surface. Docs deliver sub-30s comprehension. Release uses `npm publish --provenance` from GH Actions OIDC. Directory submission with verified schema closes the loop.
**Depends on**: Phase 6
**Requirements**: CI-01, CI-02, CI-03, CI-04, CI-05, CI-06, DOC-01, DOC-03, DOC-04, DOC-05, DOC-06, REL-01, REL-02, REL-03
**Success Criteria** (what must be TRUE):
  1. GitHub Actions on every push/PR runs: TypeScript check (`tsc --noEmit`), Biome lint, Jest tests, iOS `pod install` + `xcodebuild` (`macos-15`/Xcode 16), Android `assembleDebug` AND `assembleRelease` (release matters — release optimizations affect benchmark numbers), arm64 emulator job, mitata release-mode benchmark regression-gated at >20% from baseline; pod cache key includes `Podfile.lock` hash to prevent stale-cache replay
  2. **Consumer-app smoke build (CI-05) is green**: a separate CI job creates a fresh RN app, runs `npm pack` on this repo, installs the tarball into the fresh app, builds for both platforms in Release mode, runs `uuidv7()` and asserts a valid UUIDv7 string is returned — catches build/packaging pitfalls (R8 stripping, missing `consumer-rules.pro`, podspec misconfig, transitive Reanimated headers) that pass on the library's own CI but fail at users
  3. README first-screen comprehension passes the sub-30s test: install command, three-line minimal example, supported platforms (iOS+Android, new-arch only, Hermes/JSC), release-mode micro-benchmark numbers vs `uuid` and `react-native-uuid` (matching `docs/benchmarks.md`), drop-in migration table from `uuid.v4()` / `crypto.randomUUID()` / `react-native-uuid`, `FastUidError` code-shape contract, Expo Go (unsupported) vs Expo Dev Client (supported) matrix
  4. `CHANGELOG.md` (release-it + conventional-changelog generated), `LICENSE` (MIT), `CONTRIBUTING.md` are present and accurate; `package.json` has `files` whitelist, `prepublishOnly` running build+typecheck+tests, `peerDependenciesMeta.react-native-reanimated.optional: true`, `engines.node >= 20`
  5. `npm publish --provenance --access public` from GitHub Actions ships `0.1.0`; `npm pack && tar -tzf` inspection confirms no `example/`, `.planning/`, or test files leak; `git tag v0.1.0` + GitHub release published; react-native-directory PR submitted with `newArchitecture: true`, `expoGo: false`, `devClient: true` (schema verified at submission time)
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5 → 6 → 7

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Scaffold + ratify locked decisions | 0/TBD | Not started | - |
| 2. Generator core + standalone C++ test harness | 0/TBD | Not started | - |
| 3. JSI bindings + runtime-agnostic install + error model | 0/TBD | Not started | - |
| 4. iOS + Android native glue | 0/TBD | Not started | - |
| 5. TypeScript API + Reanimated worklet runtime install | 0/TBD | Not started | - |
| 6. Example app + release-mode benchmarks | 0/TBD | Not started | - |
| 7. CI hardening, consumer smoke build, docs & release | 0/TBD | Not started | - |

---
*Roadmap created: 2026-05-04*
*Granularity: standard (5–8 phases)*
*Coverage: 51 / 51 v0.1 requirements mapped — no orphans*
