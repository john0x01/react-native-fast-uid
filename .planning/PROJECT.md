# react-native-fast-uid

## What This Is

A React Native open-source library that generates **UUIDv7** (RFC 9562) and **ULID** identifiers via a pure C++17/JSI binding — no bridge crossings, no async, no native APIs beyond platform CSPRNG access. New-architecture only. Built for offline-first sync layers, log buffers, mutation queues, and content-addressed stores that need time-sortable IDs on hot paths.

## Core Value

Synchronous, time-sortable, RFC-9562-compliant ID generation from JS, the UI worklet runtime, or anywhere a JSI runtime exists — fast enough that the cost is invisible in the hottest paths a React Native app has.

## Requirements

### Validated

(None yet — ship to validate)

### Active

#### Generation API
- [ ] **API-01**: `uuidv7()` returns a single RFC-9562-compliant UUIDv7 as `jsi::String` (UTF-8/ASCII)
- [ ] **API-02**: `uuidv7.batch(n)` returns `jsi::Array<jsi::String>` of `n` UUIDv7s
- [ ] **API-03**: `ulid()` returns a single ULID (Crockford base32) as `jsi::String`
- [ ] **API-04**: `ulid.batch(n)` returns `jsi::Array<jsi::String>` of `n` ULIDs
- [ ] **API-05**: `installOnRuntime(runtime?)` installs bindings on default JS runtime AND optionally on a Reanimated UI runtime via the same install function

#### Native Implementation
- [ ] **NATIVE-01**: iOS uses `SecRandomCopyBytes` (Security framework) as CSPRNG
- [ ] **NATIVE-02**: Android uses `getrandom(2)` syscall with `/dev/urandom` fallback; handles getrandom blocking until kernel pool initialized
- [ ] **NATIVE-03**: Sub-millisecond monotonic counter for same-timestamp collisions; spec-compliant random for new ms (RFC 9562 Method 2; see §6.2)
- [ ] **NATIVE-04**: Mutex around shared monotonic counter state — JS thread and Reanimated UI thread can both call generators
- [ ] **NATIVE-05**: ASCII string allocation via `jsi::String::createFromAscii`
- [ ] **NATIVE-06**: No global state — mutex + counter live inside install closure
- [ ] **NATIVE-07**: No exceptions across the JSI boundary except `jsi::JSError`

#### Build & Packaging
- [ ] **BUILD-01**: Bootstrapped from `create-react-native-library` (JSI/C++ template, NOT turbo-module)
- [ ] **BUILD-02**: iOS build via CocoaPods with `s.requires_arc = true`, `s.compiler_flags = "-std=c++17"`
- [ ] **BUILD-03**: Android build via CMake + NDK; produces `.so` per ABI
- [ ] **BUILD-04**: New-architecture only — old arch explicitly unsupported
- [ ] **BUILD-05**: C++ release builds compile with `-Wall -Wextra -Werror`

#### Quality & Compliance
- [ ] **QA-01**: Generated UUIDv7s pass an independent RFC-9562 validator in the test suite
- [ ] **QA-02**: Monotonicity test — generate 10M IDs in batch; assert lex-sort matches insertion order; runs in CI
- [ ] **QA-03**: Worklet test — generate IDs from a Reanimated worklet; verify they're valid UUIDs
- [ ] **QA-04**: TypeScript strict mode, zero `any` types
- [ ] **QA-05**: Every exported TS symbol has TSDoc
- [ ] **QA-06**: Biome (TS) + clang-format (C++) applied; no `console.log` in shipped code

#### CI
- [ ] **CI-01**: GitHub Actions runs TS check, lint, JS tests
- [ ] **CI-02**: GitHub Actions runs iOS pod install + xcodebuild
- [ ] **CI-03**: GitHub Actions runs Android `assembleDebug` + `assembleRelease` (release matters — release-mode optimizations affect benchmark numbers)

#### Example App
- [ ] **EX-01**: `example/` demonstrates single-call generation
- [ ] **EX-02**: `example/` demonstrates batch generation
- [ ] **EX-03**: `example/` verifies monotonic ordering — generates 1M IDs and asserts they sort in insertion order
- [ ] **EX-04**: `example/` demonstrates Reanimated worklet usage

#### Docs & Release
- [ ] **DOC-01**: README's first screen has install command, three-line minimal example, supported platforms, micro-benchmark numbers vs `uuid` and `react-native-uuid` (sub-30s comprehension)
- [ ] **DOC-02**: `docs/benchmarks.md` documents reproducible methodology — device specs, RN version, Hermes vs JSC, sample sizes; numbers in README must match the benchmark script
- [ ] **DOC-03**: `CHANGELOG.md`, `LICENSE` (MIT), `CONTRIBUTING.md` present
- [ ] **REL-01**: Published to npm at version `0.1.0`
- [ ] **REL-02**: GitHub release `v0.1.0` tagged
- [ ] **REL-03**: Submitted to react-native-directory

### Out of Scope

- **UUIDv1, v3, v4, v5** — covered by `react-native-uuid` and `crypto.randomUUID()`; this lib is v7 + ULID only
- **`crypto.getRandomValues` polyfill** — `react-native-get-random-values` already does this
- **Async API** — all generation is synchronous JSI; async would defeat the purpose
- **Old-architecture support** — new-arch only; reduces surface area and matches RN's direction
- **Timestamp injection / mocking in public API** — always uses system clock; test-mode injection is internal-only
- **String parsing / validation utilities** — generation only; parsing is a separate concern
- **UUIDv6** — deprecated draft
- **UUIDv8 / custom variant generation** — out of scope for v0.1
- **Collision detection or registry** — that's a dataset concern, not a library concern

## Context

- **JSI warm-up project.** Author is a senior RN engineer studying new-architecture internals. Library is intentionally small enough to ship in a long weekend but exercises every JSI fundamental: HostObject install, `jsi::Function::createFromHostFunction`, string vs ArrayBuffer return, Reanimated runtime registration. Precursor to a planned simdjson wrapper.
- **Real OSS gap.** No popular native UUIDv7/ULID library for RN exists. `react-native-uuid` is pure JS and v1/v3/v4/v5 only. `uuidv7` is decent JS but not native and has no batch API. `react-native-get-random-values` polyfills `crypto.getRandomValues` for v4 only. Clean greenfield.
- **Reference libraries to study.** `react-native-mmkv` (gold-standard JSI lib structure — HostObject, install hook, CMake), `react-native-quick-crypto` (larger JSI lib for vendoring + CMake patterns), `react-native-reanimated` worklet runtime install (runtime-agnostic install pattern), `simdjson` (future-proof: this lib is the precursor), reference C++ UUIDv7 implementations (e.g. `mind-uuid-cpp`) for monotonicity edge cases.
- **Specs.** RFC 9562 (UUIDv7 canonical), Crockford base32 (ULID alphabet).

## Constraints

- **Tech stack**: C++17 only — avoid C++20+ features (NDK and Xcode toolchain support varies)
- **Tech stack**: TypeScript strict, function exports (no class wrappers); pnpm package manager
- **Architecture**: New React Native architecture only — old-arch unsupported by design
- **Performance**: Synchronous JSI; no bridge crossings; ASCII string allocation; mutex contention must not dominate batch generation
- **Compatibility**: Both Hermes and JSC; both default JS runtime and Reanimated UI runtime
- **Platforms**: iOS (Security framework / `SecRandomCopyBytes`) and Android (NDK / `getrandom(2)` + `/dev/urandom` fallback)
- **Style — C++**: Header-only where possible; `.cpp` only for install entry point; clang-format; warnings = errors in release; no exceptions across JSI boundary except `jsi::JSError`; no global state (mutex + counter live in install closure); comments explain WHY (RFC-spec choices) not WHAT
- **Style — TS**: Strict, zero `any`, function exports, TSDoc on every exported symbol, no `console.log` in shipped code
- **Style — Git**: Conventional commits; **no Claude `Co-Authored-By` trailer** (working agreement)
- **Tooling**: Biome (TS) + clang-format (C++)
- **Compliance**: RFC 9562 — generated IDs must validate against an independent RFC-9562 checker in the test suite
- **License**: MIT

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Template: `create-react-native-library` JSI/C++ variant (not turbo-module) | Pure JSI install; no codegen spec needed for this surface | — Pending |
| Plain JSI function bindings via `jsi::Function::createFromHostFunction` (not HostObject) | No state to hold; HostObject overkill | — Pending |
| Output type: `jsi::String` (UTF-8/ASCII) | Trivial JS interop; no ArrayBuffer parsing on the JS side | — Pending |
| Batch return: `jsi::Array<jsi::String>` | Simpler than typed array; profile before optimizing further | — Pending |
| Monotonicity: sub-ms counter for same-timestamp collisions; fresh random per new ms | RFC 9562 leaves room (Method 2, §6.2) — exploit it for monotonic guarantee | — Pending |
| Thread safety: mutex around shared monotonic counter | JS thread and Reanimated UI thread can both call generators | — Pending |
| Worklet compatibility: same install function callable on default JS runtime AND Reanimated UI runtime | Runtime-agnostic install pattern; transfers to future libs | — Pending |
| iOS CSPRNG: `SecRandomCopyBytes` | Apple's blessed CSPRNG | — Pending |
| Android CSPRNG: `getrandom(2)` + `/dev/urandom` fallback | Linux CSPRNG; `getrandom` blocks until kernel pool initialized — handle | — Pending |
| String allocation: `jsi::String::createFromAscii` | Faster than `createFromUtf8` for ASCII-only output | — Pending |
| Lint/format: Biome (TS) + clang-format (C++) | Two tools, but C++ needs its own | — Pending |
| New-architecture only; old-arch unsupported | Reduces surface area; matches RN's direction | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-05-04 after initialization*
