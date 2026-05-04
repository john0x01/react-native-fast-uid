# Project Research Summary

**Project:** react-native-fast-uid
**Domain:** React Native pure-JSI / C++17 library — UUIDv7 (RFC 9562) + ULID generation; Reanimated worklet runtime support; new-architecture only
**Researched:** 2026-05-04
**Confidence:** MEDIUM-HIGH (architectural choices HIGH; specific version pins LOW pending registry verification)

## Executive Summary

`react-native-fast-uid` ships native-grade, time-sortable identifier generation (UUIDv7 + ULID) for React Native via pure JSI host functions in C++17. The architectural decisions in PROJECT.md Section "Key Decisions" are the right ones — research validated each (`createFromHostFunction` over HostObject, `createFromAscii` for ASCII alphabets, `SecRandomCopyBytes` / `getrandom(2)` + `/dev/urandom` fallback, sub-ms monotonic counter under mutex, new-arch only) — and surfaces no scope gaps in the locked v0.1 surface. The clean greenfield opportunity is real: no popular RN library currently offers native UUIDv7+ULID, and **batch generation + worklet-runtime install are differentiators no competitor matches**.

The recommended approach is the well-established mmkv/quick-crypto pattern: a single `cpp/` tree at the repo root, a runtime-agnostic `installBindings(jsi::Runtime&)` function called once per runtime (default JS at module-load via TurboModule, Reanimated UI runtime via `installOnRuntime`), generator state heap-allocated and captured by `std::shared_ptr` in lambdas. This pattern transfers directly to the planned simdjson follow-up library — the JSI install plumbing, error-translation wrapper, ASCII allocation, and dual-runtime support are all reused verbatim.

Three pitfalls dominate the risk profile and dictate phase ordering: **(1) RFC 9562 Method-2 monotonicity is the highest correctness risk** (subtle counter reseed/overflow rules; must be test-first with QA-01/QA-02 written before implementation); **(2) Reanimated's UI worklet runtime is a fully separate `jsi::Runtime`** (the dual-runtime install must be designed in early, not retrofitted); **(3) Mutex granularity dominates batch performance** (lock-once-per-batch is mandatory, not optional). Address these three early and the rest of the build is execution.

## Key Findings

### Recommended Stack

The 2026 standard for pure-JSI RN libraries is well-established and largely scaffolded by `create-react-native-library`. Architectural choices are high-confidence; exact version pins must be re-verified at scaffold time (network access was unavailable during research).

**Core technologies:**
- **`create-react-native-library` (JSI/C++ template, NOT Turbo Module)**: scaffolds the codegen-minimal native-module + cpp/ tree
- **React Native ≥ 0.76**: new-arch-default release; floor for "new-arch only" libraries
- **C++17** (locked): NDK r26 + Xcode 15+ both ship cleanly; C++20 toolchain support uneven
- **Hermes** (primary) + **JSC** (fallback): JSI-engine-agnostic; matters for CI matrix not code branches
- **Reanimated ≥ 3.16** (`peerDependenciesMeta.optional`): exposes the worklet runtime install API surface
- **react-native-builder-bob**: still the canonical library build orchestrator in 2026
- **Biome (TS) + clang-format (C++)** (locked): single tool per language
- **pnpm** (locked) + **Node ≥ 20 LTS**
- **mitata** (recommended) for benchmarks — better percentile statistics than tinybench
- **Jest 29** with `react-native/jest-preset` (Jest 30 status uncertain at cutoff — verify)
- **release-it + @release-it/conventional-changelog** (single-package; not changesets)
- **CI**: GH Actions with `macos-15`/Xcode 16 for iOS, JDK 17 + NDK r26 + CMake 3.22 + Gradle 8.10 for Android

See [STACK.md](STACK.md) for full version table, verification checklist, and "what NOT to use" list.

### Expected Features

The PROJECT.md v0.1 surface (`uuidv7()`, `uuidv7.batch(n)`, `ulid()`, `ulid.batch(n)`, `installOnRuntime(runtime?)`) is exactly right — no scope gaps. The competitive position is strong: every other RN library is either pure-JS (slow), v4-only (no time-sortable), or polyfill-only (no batch).

**Must have (table stakes — confirmed by research):**
- Canonical 36-char hyphenated **lowercase** UUIDv7 (RFC 9562 §4)
- Canonical 26-char Crockford base32 **uppercase** ULID (spec mandates uppercase)
- Strict per-process monotonicity within same ms (the whole point of v7 over v4)
- CSPRNG-backed randomness only (no `Math.random()` fallback ever — silent CVE)
- Hermes + JSC parity, iOS + Android parity, sync API
- Works in **release** builds, not just debug (R8/ProGuard footguns)
- TS strict types, MIT, README first-screen comprehension
- Reasonable behavior on duplicate `installOnRuntime` (recommend idempotent)

**Should have (differentiators — competitive edge):**
- **Batch API** (`batch(n)` returning `string[]`) — no other RN UUID lib has this; saves N JSI hops for sync engines
- **Worklet-runtime install** — no competitor offers; enables IDs in gestures/animations without `runOnJS`
- **Strict cross-runtime monotonicity** within the same process
- **New-arch-only** with explicit messaging (positive signal in 2026)
- **Tree-shakeable named exports** (no default object)
- **Reproducible benchmark script** (most RN libs ship un-reproducible marketing numbers)
- **Branded TS types `Uuid`/`Ulid`** (opt-in, ~6 lines, zero runtime cost)
- **Drop-in migration story** from `uuid.v4()` / `crypto.randomUUID()` / `react-native-uuid`

**Anti-features (validated PROJECT.md exclusions; refuse on request):**
- v1/v3/v4/v5 (covered elsewhere) · `getRandomValues` polyfill · async API · old-arch · timestamp injection · parsing/validation · v6/v8/custom · collision registry · `Math.random` fallback · `null` on CSPRNG fail · prefixed UUIDs · custom epoch · lowercase ULID / RFC 4648 base32 · pre-allocated batch buffer · custom ULID alphabet · configurable monotonicity strictness

**Defer (v0.2+ if evidence accrues):**
- `uuidv7.bytes()` / `ulid.bytes()` returning `Uint8Array` (trigger: ≥3 issues asking)
- Header-only vendoring docs (trigger: a user asks)
- macOS / Windows / Web platforms (trigger: user demand + tested)

See [FEATURES.md](FEATURES.md) for full categorization, dependency graph, and competitor matrix.

### Architecture Approach

Single `cpp/` tree at the repo root (matching mmkv/quick-crypto/vision-camera). Both iOS podspec and Android CMake glob into the shared tree — no platform drift. Generator core has zero JSI dependency (compilable in a standalone test target). Platform-divergent code (CSPRNG) lives in `cpp/platform/` with build-time selection (no preprocessor macros). The runtime-agnostic install entry `installBindings(jsi::Runtime&)` is the single seam called by ALL platforms and ALL runtimes.

**Major components:**
1. **Generator core** (`cpp/core/`) — RFC-9562 §6.2 Method 2 monotonicity, mutex+counter state, ASCII formatting; pure C++, no JSI types
2. **CSPRNG adapter** (`cpp/platform/`) — `rng_apple.mm` (`SecRandomCopyBytes`), `rng_android.cpp` (`getrandom(2)` + `/dev/urandom` fallback)
3. **JSI binding layer** (`cpp/bindings/`) — host functions wrapping generator core; ASCII alloc; native-exception → `jsi::JSError` translation
4. **Install entry** (`cpp/install.cpp`) — runtime-agnostic; idempotency guard; state via `std::shared_ptr` captured in lambdas
5. **Native glue** (`ios/FastUid.mm` + `android/.../JNI.cpp` + `FastUidModule.kt`) — TurboModule shims that call C++ install entry from each platform
6. **TS export layer** (`src/index.ts` + `src/NativeFastUid.ts`) — auto-install gate at module-load, typed re-exports of `globalThis.__FastUid_*`, `installOnRuntime()` for Reanimated UI runtime
7. **Standalone C++ test harness** (`cpp/tests/`) — RFC-9562 conformance + 10M monotonicity stress test, runs in seconds without RN

The threading model: each runtime has its own `GeneratorState` heap-allocated inside its install call; mutex acquired *inside* the generator (not the binding); batch acquires lock *once* and formats strings outside the critical section. JS thread + worklet UI thread access different `GeneratorState` objects (different runtimes) — zero cross-runtime contention.

See [ARCHITECTURE.md](ARCHITECTURE.md) for file layout, data-flow diagrams, threading-scenario table, and surfaced gaps for Phase 1 ratification.

### Critical Pitfalls

Top five highest-leverage to address early. The full set is in [PITFALLS.md](PITFALLS.md) (20 pitfalls, performance traps, security mistakes, and a "looks done but isn't" checklist).

1. **RFC 9562 Method-2 monotonicity misimplementation** (Pitfall 10) — counter reseed-on-new-ms must use *high-bit-clear random* (not fully random); counter overflow must *advance the timestamp by 1ms* (not wrap); software clamp `now_ms = max(last_ms, system_clock_ms())` is required because `system_clock` is non-monotonic (NTP, DST). **Test-first**: write QA-01 (independent RFC validator) and QA-02 (10M stress) BEFORE implementation. Use `system_clock`, never `steady_clock` (unspecified epoch).

2. **Reanimated UI runtime is a separate `jsi::Runtime`** (Pitfall 2) — bindings installed on default JS runtime do *not* appear in worklets. Design `installBindings(jsi::Runtime&)` runtime-agnostic from day one; expose `installOnRuntime()` to be called once per runtime. **Verify with QA-03** — generate IDs from a Reanimated worklet *without* a prior JS-thread call. Discovering this constraint late forces an API redesign.

3. **Mutex per-ID destroying batch performance** (Pitfalls 6 + 7) — naive impl locks per ID inside batch loop; lock contention dominates at 1k+ batches and causes worklet frame drops. **Lock-once-per-batch**: snapshot counter range under lock, format ASCII *outside* the critical section. Run a 2-thread contention bench in CI before claiming "fast."

4. **Counter state must persist across JS reload** (Pitfall 11) — Fast Refresh re-runs install. If state is recreated, monotonicity breaks across the reload boundary. **Process-static `std::shared_ptr<GeneratorState>` is the deliberate exception to NATIVE-06** ("no global state"); document the exception in code. The sentinel-based idempotent install (Pitfall 4) reuses the same state on re-install.

5. **C++ exceptions escaping the JSI boundary** (Pitfall 12 + NATIVE-07) — `std::bad_alloc` / `std::system_error` / any non-`jsi::JSError` causes `std::terminate`. **Wrap every host function** with `try { } catch (jsi::JSError&) { throw; } catch (std::exception&) { throw jsi::JSError(...); } catch (...) { throw jsi::JSError(...); }`. Validate batch size early (`n <= MAX_BATCH ~100k`) and throw before allocation.

Honorable mentions to bake into early CI:
- **Release-mode benchmarking only** (Pitfall 15) — debug numbers are 5–50× slower than release; debug-mode README numbers will permanently break trust
- **`getrandom` EINTR retry + `/dev/urandom` fallback** (Pitfall 8) — handle cold-boot blocking and signal interruption
- **`SecRandomCopyBytes` return-code check** (Pitfall 9) — ignored failures leak uninitialized memory into IDs
- **Build hygiene** (Pitfall 14) — `-std=c++17` in BOTH `compiler_flags` AND `pod_target_xcconfig`; `-Werror` scoped to internal CI only (never to consumer pod build); no Reanimated headers in C++

## Implications for Roadmap

Granularity = **standard** (5–8 phases, 3–5 plans each per `.planning/config.json`). Suggested phase structure follows the dependency chain: scaffold → C++ core (highest-risk math first) → JSI bindings → platforms → TS API + worklet → example/tests → docs/CI/release. Phases 4 + 5 are independent and parallelizable.

### Phase 1: Scaffold + toolchain
**Rationale:** Greenfield; nothing exists. Lock toolchain pins (RN floor, NDK, CocoaPods, Xcode, Reanimated) before anything else — research flagged all version pins as `[VERIFY]` and the scaffold itself is the source of truth.
**Delivers:** `create-react-native-library` JSI/C++ template scaffolded; pnpm + Biome + clang-format + tsconfig + podspec + Gradle/CMake skeletons; example app builds clean on iOS + Android in both Debug and Release.
**Addresses:** BUILD-01 through BUILD-05; the Verification Checklist in STACK.md (registry pin verification, CRNL scaffold inspection).
**Avoids:** Pitfall 14 (build/packaging misconfig surfacing only at consumer-install time).

### Phase 2: Generator core (RFC-9562 + ULID + monotonicity, C++ only)
**Rationale:** Highest correctness risk in the project (Pitfall 10). Test-first against RFC test vectors before any JSI integration noise. Standalone C++ test harness (Catch2 single-header recommended) makes RFC conformance + 10M monotonicity test runnable in seconds without RN.
**Delivers:** `cpp/core/` (generator state, Method-2 counter with high-bit-clear reseed + overflow→ms-advance + software monotonic clamp), `cpp/platform/` (`SecRandomCopyBytes` + `getrandom` with EINTR retry and `/dev/urandom` fallback), `cpp/tests/` standalone harness running QA-01 (independent RFC-9562 validator) + QA-02 (10M monotonicity stress) + clock-fault-injection test.
**Addresses:** NATIVE-01, NATIVE-02, NATIVE-03, QA-01, QA-02.
**Avoids:** Pitfalls 8 (`getrandom` EINTR), 9 (`SecRandomCopyBytes` return code), 10 (Method-2 misimpl).

### Phase 3: JSI bindings + runtime-agnostic install entry
**Rationale:** With the math correct, wrap it for JSI. Idempotency, exception translation, and ASCII allocation are all design properties that need to exist at the FIRST host function so they're applied uniformly to all four (uuidv7, uuidv7.batch, ulid, ulid.batch). Process-static `shared_ptr<GeneratorState>` reconciles NATIVE-06 with the cross-reload monotonicity requirement.
**Delivers:** `cpp/bindings/` host-function factories; `cpp/install.cpp` with idempotency sentinel + process-static counter state; exception wrapper macro/template; lock-once-per-batch critical section; batch size cap.
**Addresses:** NATIVE-04 (mutex), NATIVE-05 (`createFromAscii`), NATIVE-06 (no per-call globals), NATIVE-07 (no exceptions across JSI), API-01 through API-04.
**Avoids:** Pitfalls 1 (capturing `Runtime&`), 3 (caching `jsi::Value`), 4 (install timing), 5 (`createFromAscii` semantics), 6 (per-ID lock), 11 (counter state lost on reload), 12 (C++ exceptions escape), 13 (unbounded batch GC pressure), 18 (error code design).

### Phase 4: iOS + Android native glue
**Rationale:** Platform-specific TurboModule shims that invoke the C++ install entry. iOS first (no NDK/CMake/JNI complexity) validates the new-arch install pipeline; Android follows the same pattern via JNI + CMake. Could parallelize per `.planning/config.json` `parallelization: true`, but iOS-first is the safer sequence.
**Delivers:** iOS `FastUid.mm` (TurboModule shim, < 30 lines); Android `FastUidModule.kt` + `JNI.cpp` + `CMakeLists.txt` (globs `cpp/**`); release build of example app green on both platforms.
**Addresses:** BUILD-02, BUILD-03; bridges generator/bindings to RN.

### Phase 5: TypeScript API + worklet runtime install
**Rationale:** Public surface that consumers actually import. The worklet path is the second-highest risk after Method-2 monotonicity — Reanimated's runtime install API has churned across 3.x → 4.x. Verify against current Reanimated source. QA-03 worklet test is a *gate*, not a checkbox: if dual-runtime design is wrong, this is where it breaks.
**Delivers:** `src/NativeFastUid.ts` codegen spec, `src/index.ts` typed re-exports + `installOnRuntime()`, `src/types.ts` opt-in branded `Uuid`/`Ulid`, auto-install at module load; QA-03 worklet test in example app.
**Addresses:** API-05, QA-03, QA-04, QA-05.
**Avoids:** Pitfalls 2 (worklet runtime separate), 20 (worklet + mutex contention).

### Phase 6: Example app, monotonicity demo, benchmarks
**Rationale:** Example demonstrates the four user workloads (single, batch, monotonicity verification, worklet). Benchmarks must run in **release mode** (Pitfall 15) — locked in this phase. mitata for stats; comparison baselines pinned in the bench script.
**Delivers:** `example/` with single-call demo, batch demo, 1M monotonicity check (EX-03), Reanimated worklet demo (EX-04); `docs/benchmarks.md` (DOC-02) with reproducible methodology — device, RN version, engine, sample size, p50/p95/p99; release-mode benchmark numbers vs `uuid`, `react-native-uuid`, `uuidv7`.
**Addresses:** EX-01 through EX-04, DOC-02.
**Avoids:** Pitfall 15 (debug-mode benchmark divergence).

### Phase 7: CI hardening + consumer smoke build
**Rationale:** CI is what makes the v0.1 ship reliable for users on different toolchains. Release-mode Android (CI-03) is non-negotiable; arm64 + x86_64 matrix catches alignment bugs only real devices see; pod cache key includes lockfile hash. **Consumer-app smoke job** — install your tarball into a fresh RN app, build both platforms — catches the build/packaging pitfalls that *only* surface in consumer projects.
**Delivers:** GitHub Actions workflows for TS check + lint + Jest, iOS `pod install` + `xcodebuild`, Android `assembleDebug` + `assembleRelease`, arm64 emulator job, npm-tarball-into-fresh-app smoke build.
**Addresses:** CI-01, CI-02, CI-03.
**Avoids:** Pitfalls 14 (build misconfig surfacing only in consumer apps), 16 (CI emulator + cache differences).

### Phase 8: Docs, release, react-native-directory submission
**Rationale:** Final phase — README first-screen comprehension, migration table, error-format docs, CHANGELOG, LICENSE, CONTRIBUTING. npm provenance via GH Actions OIDC. `npm pack && tar -tzf` inspection before publish. Directory metadata: `expoGo: false`, new-arch flag, Reanimated optional peer-dep.
**Delivers:** README (DOC-01) with install + 3-line example + platforms + release-mode benchmark numbers; CHANGELOG, LICENSE (MIT), CONTRIBUTING; `files` whitelist; `prepublishOnly` script; `npm publish --provenance`; GitHub release `v0.1.0`; react-native-directory PR.
**Addresses:** DOC-01, DOC-03, REL-01, REL-02, REL-03.
**Avoids:** Pitfalls 17 (npm publish), 18 (API design irreversibility), 19 (react-native-directory metadata).

### Phase Ordering Rationale

- **Math first, plumbing second.** Generator core (Phase 2) is the highest correctness risk; isolating it in a standalone C++ test harness lets RFC-9562 conformance + 10M monotonicity be verified in seconds before any RN integration noise enters.
- **JSI install plumbing as one phase.** Idempotency, exception wrapping, ASCII alloc, and lock-once-per-batch are properties that must exist at the *first* host function, not retrofitted. Phase 3 establishes them all uniformly.
- **iOS before Android.** iOS Phase 4 sequence has no NDK/CMake/JNI complexity; validating the new-arch TurboModule install on iOS first reduces variables when Android comes online.
- **Worklet path is its own phase.** The single highest-risk integration after Phase 2; deserves dedicated focus and the QA-03 gate.
- **Benchmarks AFTER lock-once-per-batch is in place.** Publishing release-mode numbers from a per-ID-locking prototype would establish a permanent baseline 5–50× slower than reality (Pitfalls 6 + 7 + 15 compound).
- **CI hardening before publish.** Consumer-app smoke build is the single highest-leverage CI job — it catches the build/packaging pitfalls that pass on the library's own CI but fail at users.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 1 (Scaffold):** verify all `[VERIFY]` version pins from STACK.md against current registry; inspect the actual `create-react-native-library` JSI/C++ scaffold output and reconcile to ARCHITECTURE.md's proposed structure
- **Phase 5 (TS API + worklet install):** Reanimated worklet runtime install API has churned across 3.x → 4.x; verify the canonical install path against current Reanimated source (mmkv v3 commit history is the reference)

Phases with standard patterns (skip research-phase):
- **Phase 2 (Generator core):** RFC 9562 §5.7 + §6.2 are the spec; no further research needed beyond the spec itself
- **Phase 3 (JSI bindings):** mmkv/quick-crypto patterns are well-established; ARCHITECTURE.md captures them
- **Phase 4 (Native glue):** standard TurboModule patterns
- **Phase 6 (Example/benchmarks):** mitata + standard mmkv-style benchmark methodology
- **Phase 7 (CI):** standard GH Actions matrix; CRNL scaffolds workflows
- **Phase 8 (Docs/release):** standard `release-it` + npm provenance patterns

## Decisions Surfaced for Ratification (from FEATURES + ARCHITECTURE — gaps in PROJECT.md)

These are *not* scope changes — they are clarifications/small additions that need locking before code:

1. **Counter overflow strategy:** advance timestamp by 1 ms (recommended; matches ULID spec + LiosK's `uuidv7`). Add to PROJECT.md Key Decisions.
2. **CSPRNG failure error message format:** typed `FastUidError` with `code: "CSPRNG_FAILED" | "RUNTIME_NOT_INSTALLED" | "BATCH_TOO_LARGE" | "INTERNAL"` from day one (irreversible after publish per Pitfall 18). Document in README error-format section.
3. **`installOnRuntime` idempotency:** second call is a no-op via sentinel check (recommended). Add to Key Decisions.
4. **Branded TS types `Uuid`/`Ulid`:** opt-in alongside plain `string` returns (recommended); ~6 lines, zero runtime cost.
5. **ULID counter — shared or separate from UUIDv7?** Separate counters, same mutex (recommended; sharing the mutex avoids a second lock; separate counters keep spec semantics clean).
6. **Batch size cap:** `1 << 17` (~131k) recommended — bounds Hermes GC pressure (Pitfall 13) without hurting realistic workloads.
7. **`installOnRuntime` API name** vs `installForWorklets`/`installOnUIRuntime`: `installOnRuntime` (runtime-agnostic naming, matches API-05 PROJECT.md).
8. **Counter state — process-static exception to NATIVE-06:** monotonic counter is a process-singleton; document the deliberate exception in code AND in PROJECT.md.

## JSI Patterns That Transfer to the Planned simdjson Library

Author context: this is a JSI warm-up before a planned simdjson wrapper. Patterns deliberately exercised here that transfer 1:1:

| Pattern | Where it lives here | Transfers to simdjson as |
|---------|---------------------|---------------------------|
| `installBindings(jsi::Runtime&)` runtime-agnostic install | `cpp/install.cpp` | Same install entry, different host functions |
| Single `cpp/` tree, podspec + CMake glob | Repo layout | Identical layout |
| `std::shared_ptr<State>` captured in lambdas; runtime-bound state lifetime | NATIVE-06 + Pitfall 1 | Identical (state holds parsed-doc handles instead of monotonic counter) |
| `try/catch` exception-translation wrapper at every host function | NATIVE-07 + Pitfall 12 | Identical |
| `jsi::String::createFromAscii` for ASCII output paths | NATIVE-05 + Pitfall 5 | Same when emitting ASCII (numbers, keys); `createFromUtf8` for true Unicode strings |
| Reanimated UI runtime install via `installOnRuntime()` | API-05 + Pitfall 2 | Identical |
| Idempotency sentinel | Pitfall 4 | Identical |
| Standalone `cpp/tests/` harness with Catch2 single-header | Phase 2 | Identical (parses RFC-8259 test vectors instead of UUID validators) |
| Lock-once-per-batch critical section pattern | NATIVE-04 + Pitfall 6 | Applies to streaming-parse buffer reuse |
| `consumer-rules.pro` shipped in AAR | Pitfall 14 + 16 | Identical |
| Release-mode-only benchmark methodology | Pitfall 15 + DOC-02 | Identical |

Building these correctly here means the simdjson follow-up is largely "swap the algorithm." That is the real return on this weekend's investment.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack — architectural choices (template, tool category, what NOT to use) | HIGH | Locked decisions in PROJECT.md; CRNL scaffolds for years; no replacement has emerged |
| Stack — exact version pins | LOW | Network access denied during research; all `[VERIFY]` flags must be resolved at scaffold |
| Features — locked v0.1 scope | HIGH | Validated against `uuid` v9+, `uuidv7`, `react-native-uuid`, `crypto.randomUUID` ecosystem; no scope gaps |
| Features — irreversible decisions (error model, return types, batch shape) | MEDIUM-HIGH | Reasoning solid; needs explicit ratification before code (see "Decisions Surfaced" above) |
| Architecture — file layout + component boundaries + data flow | HIGH | Consensus pattern across mmkv, quick-crypto, vision-camera |
| Architecture — Reanimated worklet install API surface | MEDIUM | API has churned 3.x → 4.x; verify against current source in Phase 5 |
| Pitfalls — JSI lifetime, monotonicity, mutex, CSPRNG | HIGH | Domain-grounded; well-documented mistakes in training data |
| Pitfalls — react-native-directory schema, GH Actions runner specifics | MEDIUM | Schema/runner availability evolves; verify at submission/CI-setup time |

**Overall confidence:** MEDIUM-HIGH. Architectural and design recommendations are stable regardless of patch-version drift. Specific version pins must be re-verified at Phase 1 scaffold time.

### Gaps to Address During Planning/Execution

- **Version pin verification.** All `[VERIFY]` markers in STACK.md must be resolved by running `npm view` queries and inspecting actual `create-react-native-library` scaffold output before committing `package.json`. (Phase 1)
- **Reanimated worklet install API surface.** Verify against current Reanimated source which version-floor stably exposes the runtime-install API and what the canonical install call looks like. Likely floor: 3.6+; confirm in Phase 5.
- **`uuid` npm v11+ Method-2 behavior.** Confirm whether they bump-timestamp or throw on counter saturation — affects whether our behavior is "matches `uuid` npm" or differentiated. (Phase 2)
- **`getrandom` libc symbol vs raw syscall** at chosen `minSdk`. Verify availability across NDK API levels targeted (likely 24+). (Phase 2)
- **`createFromAscii` availability** on both Hermes and JSC at the RN versions targeted. Believed yes since RN 0.71+; verify. (Phase 3)
- **Metro tree-shaking aggressiveness** for `uuidv7.batch` namespace pattern. Verify in example app build that unused exports drop. (Phase 5)
- **react-native-directory submission schema** at submission time — schema evolves; field names and Expo-related fields need verification at REL-03. (Phase 8)
- **GitHub Actions Apple Silicon arm64 runner availability** for Android arm64 emulator coverage. Schema/runner availability changes monthly. (Phase 7)

## Sources

### Primary (HIGH confidence)
- **PROJECT.md** — locked architectural decisions; the contract this research validated against
- **RFC 9562** (UUIDv7 canonical, Oct 2024) — §5.7 layout, §6.2 monotonicity (Methods 1/2/3) — to spot-verify before NATIVE-03
- **Crockford base32 spec** — ULID alphabet (Crockford uppercase, excludes I/L/O/U)

### Secondary (MEDIUM confidence — training data, cutoff Jan 2026)
- **`react-native-mmkv` v3.x source** (mrousavy) — gold-standard JSI install pattern, dual-runtime support reference
- **`react-native-quick-crypto` source** (margelo) — large-scale JSI lib for vendoring/CMake patterns; CSPRNG handling
- **`react-native-reanimated` 3.x source** (software-mansion) — `createWorkletRuntime`, `runOnRuntime`, worklet runtime C++ class
- **`create-react-native-library` JSI/C++ template** (callstack) — what the scaffold actually generates
- **React Native release notes** — RN 0.76 new-arch-default release; iOS 15.1 floor; JSC deprecation/community fork
- **Hermes documentation** — string optimization, GC behavior; `createFromAscii` semantics
- **Apple `SecRandomCopyBytes` reference** — return codes and failure modes
- **Linux `getrandom(2)` / `urandom(4)` man pages** — EINTR semantics, blocking behavior, `GRND_NONBLOCK`

### Tertiary (LOW confidence — needs validation at execution time)
- **All exact version pins** — npm registry verification required at Phase 1 scaffold
- **react-native-directory submission schema** — verify field names at REL-03
- **GH Actions runner availability** (macos-15/Apple Silicon arm64 Android emulator) — verify at CI-01 setup
- **NDK getrandom libc-wrapper API-level availability** — verify at NATIVE-02

---
*Research completed: 2026-05-04*
*Ready for roadmap: yes*
