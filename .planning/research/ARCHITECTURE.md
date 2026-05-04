# Architecture Research — react-native-fast-uid

**Domain:** Pure-JSI / C++17 React Native library (UUIDv7 + ULID generation, new-architecture only)
**Researched:** 2026-05-04
**Confidence:** MEDIUM

> **Tool-availability note:** WebSearch, WebFetch, and network-using Bash were permission-denied. Architecture is drawn from prior knowledge of `mrousavy/react-native-mmkv` (v3.x), `margelo/react-native-quick-crypto`, `callstack/react-native-builder-bob` (the `create-react-native-library` JSI template), and `software-mansion/react-native-reanimated` worklet-runtime install patterns. Treat exact filenames and Reanimated API names as MEDIUM confidence; the structural pattern is HIGH confidence.

---

## System Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                        JavaScript Layer (src/)                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  index.ts  — typed re-exports, install gate, install on rt     │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                  ▲                                    │
│                  globalThis.__FastUid_uuidv7 / __FastUid_ulid         │
│                                  │                                    │
├──────────────────────────────────┼────────────────────────────────────┤
│            JSI Binding Layer (cpp/) — runtime-agnostic                │
│  ┌────────────────────┐  ┌────────────────────┐  ┌───────────────┐   │
│  │  install.cpp       │  │  bindings.cpp      │  │  ascii alloc  │   │
│  │  installBindings() │→ │  uuidv7/ulid host  │→ │  helpers      │   │
│  │  (state in closure)│  │  functions + batch │  │               │   │
│  └────────────────────┘  └─────────┬──────────┘  └───────┬───────┘   │
│                                    │                     │           │
├────────────────────────────────────┼─────────────────────┼───────────┤
│              Generator Core (cpp/core/) — pure C++, no JSI            │
│  ┌────────────────────┐  ┌────────────────────┐  ┌───────────────┐   │
│  │  generator.h/.cpp  │  │  uuidv7.h/.cpp     │  │  ulid.h/.cpp  │   │
│  │  state holder      │  │  encode 16 bytes → │  │  Crockford    │   │
│  │  (mutex+counter+   │  │  36-char hex+dash  │  │  base32       │   │
│  │   last-ms+rand)    │  │                    │  │  26-char      │   │
│  └────────────────────┘  └────────────────────┘  └───────────────┘   │
│                                  ▲                                    │
├──────────────────────────────────┼────────────────────────────────────┤
│              Platform CSPRNG Adapter (cpp/platform/)                  │
│  ┌────────────────────────────────┐  ┌──────────────────────────┐    │
│  │  rng_apple.mm                  │  │  rng_android.cpp         │    │
│  │  SecRandomCopyBytes            │  │  getrandom(2) +          │    │
│  │                                │  │  /dev/urandom fallback   │    │
│  └────────────────────────────────┘  └──────────────────────────┘    │
├──────────────────────────────────────────────────────────────────────┤
│              Native Glue (ios/ + android/) — install trigger          │
│  ┌────────────────────────────────┐  ┌──────────────────────────┐    │
│  │  ios/FastUid.mm                │  │  android/.../JNI.cpp +   │    │
│  │  TurboModule shim →            │  │  CMakeLists.txt          │    │
│  │  installBindings(rt)           │  │  installBindings(rt)     │    │
│  └────────────────────────────────┘  └──────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|------------------------|
| **Generator core** (`cpp/core/`) | Produce 16 raw bytes (RFC 9562 §5.7 layout for UUIDv7; ULID layout for ULID). Hold mutex, sub-ms counter, last-ms, refill-random-tail-on-new-ms. Encode bytes → ASCII into a stack buffer. **No JSI types referenced.** | Header-only `.h` plus one `.cpp` per format. Compiles in a free-standing C++ test target with no RN headers. |
| **CSPRNG adapter** (`cpp/platform/`) | `void fill_random(uint8_t* out, size_t n)`; iOS impl uses `SecRandomCopyBytes`, Android impl wraps `getrandom(2)` with `/dev/urandom` fallback. Throws `std::runtime_error` on irrecoverable failure. | Two `.cpp`/`.mm` files selected by build system (NOT preprocessor macros). |
| **JSI binding layer** (`cpp/bindings/`) | Translate JS args ↔ generator core. Wrap native exceptions in `jsi::JSError`. ASCII alloc via `jsi::String::createFromAscii`. Build `jsi::Array` for batches. Owns no state. | One `.cpp` exposing `createUuidv7Function`, `createUlidFunction`, batch variants — each returns `jsi::Function`. |
| **Install entry** (`cpp/install.cpp`) | Single C++ function called by ALL platforms and ALL runtimes: `void installBindings(jsi::Runtime& rt)`. Heap-allocates `GeneratorState`, captured by lambdas via `std::shared_ptr`. Sets four properties on `rt.global()`. | One `.cpp`. **No global state** — fresh state per call. |
| **iOS native glue** (`ios/FastUid.mm`) | Thin Objective-C++ TurboModule shim. New-arch: implements TurboModule's `installJSIBindings` method (or equivalent), grabs `jsi::Runtime&`, calls C++ `installBindings(rt)`. | `.mm` file, < 30 lines. Conventional `react-native-mmkv` v3 pattern. |
| **Android native glue** (`android/.../JNI.cpp` + `FastUidModule.kt`) | Kotlin TurboModule exposes `installJSIBindings(): Boolean`. JNI receives `jsi::Runtime*` from `CallInvokerHolder` (new-arch) and calls C++ `installBindings(rt)`. CMakeLists.txt builds `cpp/**` + `JNI.cpp` into `libreact-native-fast-uid.so`. | fbjni-style JNI, links `react_nativemodule_core`, `-llog`, `-landroid`. |
| **TS export layer** (`src/index.ts`) | Imports native module, invokes `installJSIBindings()` once at module load, re-exports `globalThis.__FastUid_*` as typed functions. Provides `installOnRuntime()` for Reanimated UI runtime. Strict types, TSDoc on every export. | One `index.ts` + `types.ts`; no runtime logic beyond install gating. |

## Recommended Project Structure

```
react-native-fast-uid/
├── cpp/                                   # ALL C++17 sources, single tree
│   ├── core/
│   │   ├── generator.h                    # State struct + sub-ms monotonicity logic
│   │   ├── generator.cpp                  # RFC 9562 §6.2 Method 2 implementation
│   │   ├── uuidv7.h                       # encode_uuidv7(uint8_t[16], char out[36])
│   │   ├── uuidv7.cpp
│   │   ├── ulid.h                         # encode_ulid(uint8_t[16], char out[26])
│   │   └── ulid.cpp
│   ├── platform/
│   │   ├── rng.h                          # void fill_random(uint8_t*, size_t)
│   │   ├── rng_apple.mm                   # SecRandomCopyBytes (iOS-only)
│   │   └── rng_android.cpp                # getrandom + /dev/urandom (Android-only)
│   ├── bindings/
│   │   ├── bindings.h
│   │   └── bindings.cpp                   # createUuidv7Function, batch variants, etc.
│   ├── install.h
│   └── install.cpp                        # The runtime-agnostic install entry
├── ios/
│   ├── FastUid.h
│   └── FastUid.mm                         # TurboModule shim → installBindings(rt)
├── android/
│   ├── build.gradle                       # cmake { path = "src/main/cpp/CMakeLists.txt" }
│   └── src/main/
│       ├── cpp/
│       │   ├── CMakeLists.txt             # globs ../../../../cpp/**/*.cpp + JNI.cpp
│       │   └── JNI.cpp                    # JNIEXPORT installJSIBindings → installBindings(rt)
│       └── java/com/fastuid/
│           └── FastUidModule.kt           # TurboModule, exposes installJSIBindings(): Boolean
├── src/
│   ├── index.ts                           # Public API: uuidv7, ulid, batch, installOnRuntime
│   ├── NativeFastUid.ts                   # TurboModule codegen spec
│   └── types.ts                           # Branded string types if desired
├── example/                               # New-arch RN app, demonstrates worklet usage
├── react-native-fast-uid.podspec          # source_files include "../cpp/**", "ios/**"
├── package.json                           # codegenConfig.name = "RNFastUidSpec"
├── tsconfig.json
├── biome.json
├── .clang-format
└── README.md
```

### Structure Rationale

- **Single `cpp/` tree at repo root, NOT under `ios/` or `android/`.** Both platforms compile from the same sources. iOS podspec globs `"../cpp/**/*"`; Android `CMakeLists.txt` globs `../../../../cpp/**/*.cpp`. Matches `react-native-mmkv`, `react-native-quick-crypto`, `react-native-vision-camera`. Avoids drift between platforms.
- **`cpp/core/` has zero JSI dependency.** Math-heavy generator code compiles in a tiny standalone CMake target without any RN headers — fast iteration during Phase 2, RFC-9562 conformance tests run in seconds.
- **`cpp/platform/` isolates the only platform-divergent code.** Selection at build time via source globs / podspec filtering, not preprocessor macros.
- **`cpp/install.cpp` is the seam.** Only entry called by iOS / Android glue. Runtime-agnostic by design.
- **`src/NativeFastUid.ts` is the codegen spec.** Single method: `installJSIBindings(): boolean;`. Codegen produces `RNFastUidSpec` C++ class consumed by the iOS/Android shims.
- **No `lib/` checked in.** `react-native-builder-bob` handles TS → JS at publish.

## Architectural Patterns

### Pattern 1: Runtime-Agnostic Install Closure

**What:** `installBindings(jsi::Runtime& rt)` takes a runtime by reference and installs host functions on it. No assumption about which runtime — JS runtime, Reanimated UI runtime, or any future JSI runtime. Generator state is heap-allocated inside `installBindings` and captured by `std::shared_ptr` in each lambda.

**When to use:** Any pure-JSI library supporting both default JS and Reanimated UI runtime. Same pattern `react-native-mmkv` adopted in v3.

**Trade-offs:**
- Pro: One install function, two call sites (JS thread at module load, UI thread via worklet). Zero duplication.
- Pro: Per-runtime state isolation falls out automatically (NATIVE-06 satisfied).
- Con: Cross-runtime monotonicity is NOT preserved — JS-thread `uuidv7()` and worklet `uuidv7()` in the same wall-clock ms may sort either order. Document as expected behavior; per-runtime monotonicity is preserved.

```cpp
// cpp/install.cpp
void installBindings(jsi::Runtime& rt) {
  // Idempotency guard
  if (rt.global().hasProperty(rt, "__FastUid_uuidv7")) return;

  auto state = std::make_shared<GeneratorState>();

  auto uuidv7Fn = createUuidv7Function(rt, state);
  auto ulidFn   = createUlidFunction(rt, state);
  uuidv7Fn.setProperty(rt, "batch", createUuidv7BatchFunction(rt, state));
  ulidFn.setProperty(rt,   "batch", createUlidBatchFunction(rt,   state));

  rt.global().setProperty(rt, "__FastUid_uuidv7", uuidv7Fn);
  rt.global().setProperty(rt, "__FastUid_ulid",   ulidFn);
}
```

### Pattern 2: State-via-shared_ptr-Capture (no globals, no statics)

**What:** Mutex, monotonic counter, last-ms, and the random tail live in a `GeneratorState` struct heap-allocated inside `installBindings`. Wrapped in `std::shared_ptr` and captured by every host-function lambda. Runtime destruction → lambdas GC'd → shared_ptr count drops → state freed.

**When to use:** Whenever NATIVE-06 ("no global state") applies — multi-runtime install, hot reload, per-test teardown.

```cpp
// cpp/core/generator.h
struct GeneratorState {
  std::mutex mu;
  uint64_t last_ms = 0;
  uint16_t sub_ms_counter = 0;
  uint8_t  rand_tail[10];
};
```

### Pattern 3: Two-Layer Error Translation

**What:** Generator core throws `std::runtime_error` with a sanitized message. JSI binding layer catches `std::exception` and rethrows as `jsi::JSError`. Platform RNG layer translates platform error codes (`OSStatus`, `errno`) into a category string but discards raw values from the JS-facing message — they're logged via native loggers (`os_log` / `__android_log_print`) for debugging.

**Error message contract:**

| Native condition | JS-visible message | Logged to native log |
|---|---|---|
| `SecRandomCopyBytes` returns non-zero | `"fast-uid: CSPRNG unavailable"` | `OSStatus` value |
| `getrandom` exhausts retries (EINTR) and `/dev/urandom` open/read fails | `"fast-uid: CSPRNG unavailable"` | `errno` |
| `batch(n)` with n < 0 or n > 1<<24 | `"fast-uid: batch size out of range"` | n/a |
| `batch(n)` with non-integer | `"fast-uid: batch size must be an integer"` | n/a |

### Pattern 4: Mutex Inside the Generator, Not the Binding

**What:** `GeneratorState::mu` is acquired inside `generate_uuidv7(state, out)` / `generate_ulid(state, out)` — NOT in the host-function lambda. Binding layer is mutex-unaware.

**Trade-offs:**
- Pro: Generator core is testable for concurrency correctness without JSI.
- Pro: Batch generation acquires the lock ONCE for the entire batch — addresses "mutex contention must not dominate batch generation". For `n=1000`: 1 lock acquisition vs. 1000.
- Pro: JSI string allocation happens AFTER lock release.

```cpp
jsi::Array createBatch(jsi::Runtime& rt, std::shared_ptr<GeneratorState> state, size_t n) {
  std::vector<std::array<char, 36>> out(n);
  generate_uuidv7_batch(*state, out);  // ONE lock for the whole batch
  jsi::Array arr(rt, n);
  for (size_t i = 0; i < n; ++i) {
    arr.setValueAtIndex(rt, i, jsi::String::createFromAscii(rt, out[i].data(), 36));
  }
  return arr;
}
```

## Data Flow

### Single-ID Request Flow

```
JS:  const id = uuidv7();
  ↓
HostFunction lambda  (captures shared_ptr<GeneratorState>)
  ↓
generate_uuidv7(state, char buf[37])
  ↓ (acquires state.mu)
  ├─ now_ms = system_clock::now()
  ├─ if (now_ms == state.last_ms): state.sub_ms_counter++
  │                                 if overflow: spin-wait to next ms
  ├─ if (now_ms != state.last_ms):  state.sub_ms_counter = 0
  │                                  fill_random(state.rand_tail, 10)
  ├─ assemble 16 bytes per RFC 9562 §5.7
  └─ encode_uuidv7_hex(bytes, buf)
  ↓ (releases state.mu)
buf = "0190b8c0-..."
  ↓
jsi::String::createFromAscii(rt, buf, 36)  ← OUTSIDE the lock
  ↓
JS: id is now a string
```

### Batch Request Flow

```
JS:  const ids = uuidv7.batch(1000);
  ↓
HostFunction batch lambda
  ↓
generate_uuidv7_batch(state, n=1000)
  ↓ (acquires state.mu ONCE)
  ├─ for i in 0..1000: [same per-ID logic, lock already held]
  ↓ (releases state.mu)
1000 char[36] outputs
  ↓
jsi::Array of 1000 jsi::String (per-string ASCII alloc)
  ↓
JS: ids is now string[]
```

### Install Flow (JS Runtime — once at module load)

```
App starts
  ↓
import 'react-native-fast-uid'  (in src/index.ts)
  ↓
NativeFastUid.installJSIBindings()  ← TurboModule call
  ↓
iOS: -[FastUid installJSIBindings]   ──→  fastuid::installBindings(rt)
Android: JNI ..._installJSIBindings  ──→  fastuid::installBindings(rt)
  ↓
rt.global().setProperty('__FastUid_uuidv7', ...)  + ulid + batches
  ↓
src/index.ts re-exports globals as typed functions
  ↓
App code calls uuidv7() ← actually globalThis.__FastUid_uuidv7
```

### Install Flow (Reanimated UI Runtime — on demand)

```
JS: import { installOnRuntime } from 'react-native-fast-uid';
JS: import { runOnUI } from 'react-native-reanimated';
  ↓
runOnUI(() => {
  'worklet';
  installOnRuntime();
})()
  ↓
A pre-registered host function on the UI runtime calls fastuid::installBindings(uiRt)
  ↓
UI runtime's global gets __FastUid_uuidv7 etc.
  ↓
Subsequent worklets call uuidv7() / ulid() synchronously
```

> **Phase 1 verification:** The exact API surface for installing onto Reanimated's UI runtime evolved between Reanimated 3.x and 4.x. The current canonical path uses a host function registered on the UI runtime that bridges to the C++ install entry. Implementation should verify against the current Reanimated version against `react-native-mmkv` v3's commit history (the first major library to add Reanimated worklet support).

### State Management

```
Per Runtime:
  [GeneratorState (heap, shared_ptr)]
    ├─ std::mutex mu          (synchronizes the next 3 fields)
    ├─ uint64_t last_ms
    ├─ uint16_t sub_ms_counter
    └─ uint8_t  rand_tail[10]

  Captured by 4 host-function lambdas:
    uuidv7,  uuidv7.batch,  ulid,  ulid.batch
    (all sharing the same state via shared_ptr copies)

  Lifetime: tied to runtime — runtime destroyed → lambdas GC'd
            → shared_ptr count zero → state destroyed.
```

## Build Order Across Phases

```
Phase 1: Project bootstrap
  └─ create-react-native-library JSI/C++ template
  └─ tsconfig, biome, clang-format, podspec, gradle/CMake skeletons
  └─ Validate skeleton builds on iOS + Android example apps
       ↓
Phase 2: Generator core (cpp/core/, cpp/platform/)
  └─ rng.h interface + iOS/Android impls
  └─ generator state struct + sub-ms monotonicity logic (RFC 9562 §6.2 Method 2)
  └─ uuidv7.h/cpp encode (hex + dashes)
  └─ ulid.h/cpp encode (Crockford base32)
  └─ Standalone C++ test target — runs RFC-9562 validator (QA-01)
       ↓
Phase 3: JSI bindings (cpp/bindings/, cpp/install.cpp)
  └─ createUuidv7Function, createUlidFunction, batch variants
  └─ installBindings(rt) with idempotency guard
  └─ Error translation layer (std::exception → jsi::JSError)
       ↓
Phase 4: iOS native glue
  └─ FastUid.mm TurboModule shim
  └─ Podspec source globs cpp/** and ios/**
  └─ xcodebuild on example app green
       ↓
Phase 5: Android native glue
  └─ JNI.cpp installJSIBindings entry
  └─ CMakeLists.txt globs cpp/**
  └─ FastUidModule.kt TurboModule
  └─ assembleDebug + assembleRelease green
       ↓
Phase 6: TS API + worklet install
  └─ src/index.ts public surface
  └─ src/NativeFastUid.ts codegen spec
  └─ installOnRuntime() worklet path verified against Reanimated
       ↓
Phase 7: Example app + tests
  └─ Single-call demo, batch 1M monotonicity check, worklet demo
       ↓
Phase 8: Docs, CI, release
  └─ README, benchmarks.md, CHANGELOG, LICENSE, CONTRIBUTING
  └─ GitHub Actions: TS check, lint, iOS pod+xcodebuild, Android assembleDebug+Release
  └─ npm publish 0.1.0, GitHub release, react-native-directory submission
```

**Why this order:**
- Generator core (Phase 2) is the highest-risk component (RFC-9562 conformance + monotonicity). Putting it first means the math is right before any JSI integration noise.
- JSI bindings (Phase 3) depend on generator core but are platform-agnostic — testable independently of iOS/Android setup.
- Platforms 4+5 *could* parallelize, but iOS first validates the pipeline on the simpler platform.
- Worklet install (Phase 6) requires the C++ install entry to be runtime-agnostic — that's a Phase 3 design property exercised here.

## Threading Model

### Contract

> **`installBindings(rt)` may be called from any thread, but only on a runtime that is not currently executing JS.** Conventional call sites: (a) JS thread at module load on the default runtime, (b) UI thread at worklet-init time on the Reanimated UI runtime.
>
> **Each `uuidv7()` / `ulid()` / batch call is safe from any thread the runtime considers itself to be on at that moment.** The lambda runs on the runtime's calling thread; `GeneratorState::mu` serializes concurrent calls at the generator level.

### Concrete Threading Scenarios

| Scenario | Thread(s) | Mutex behavior | Correctness |
|---|---|---|---|
| JS thread calls `uuidv7()` 1000x sequentially | JS thread only | 1000 uncontested lock acquisitions | Monotonic, correct |
| JS calls `uuidv7.batch(1000)` | JS thread only | 1 lock for the whole batch | Monotonic, correct, optimal |
| Worklet (UI thread) calls `uuidv7()` after `installOnRuntime()` | UI thread on UI runtime | Lock on UI-runtime's state (different state object) | Monotonic within UI stream |
| JS calls AND worklet calls in same wall-clock ms | Two threads, two runtimes, two states | Each runtime locks its own mutex independently | Each stream monotonic; cross-stream order undefined (documented behavior) |
| RN frees a runtime mid-call | Runtime's owning thread | `lock_guard` releases as it unwinds; lambda's shared_ptr keeps state alive | No use-after-free |

### Mutex Contention Analysis

- **Single-ID call:** `lock_guard` ~10-20ns uncontested. Generator body ~50-200ns. Mutex < 30% of total — acceptable.
- **Batch call:** Lock once, generate N IDs in critical section. Cost amortizes to negligible.
- **Two-runtime workload:** Zero contention across runtimes.

### Reentrancy

Host-function lambdas MUST NOT call back into JS while holding the state mutex. Current design doesn't — generators only do byte ops + ASCII formatting under lock; JSI string alloc happens after release.

## Anti-Patterns

### Anti-Pattern 1: Putting C++ sources under `ios/` or `android/`
**Why wrong:** Sources drift. iOS gets a fix Android never sees. Symlinks break on Windows.
**Instead:** Single `cpp/` tree at repo root. Both podspec and CMake glob into it.

### Anti-Pattern 2: Exposing JSI types in the generator core
**Why wrong:** Couples generator to JSI headers. Standalone testing now requires full RN build.
**Instead:** Generator takes `char[]` output buffer, returns `void` (or throws). JSI layer wraps with `createFromAscii`.

### Anti-Pattern 3: Static / global generator state
**Why wrong:** Violates NATIVE-06. Cross-runtime state collisions. Hot reload leaks.
**Instead:** State allocated inside `installBindings`, captured via `shared_ptr`.

### Anti-Pattern 4: Locking around the JSI string allocation
**Why wrong:** Mutex held during JSI alloc. JSI alloc can be slow (heap, GC pressure). Other threads spin on irrelevant work.
**Instead:** Lock inside generator only. JSI alloc after release.

### Anti-Pattern 5: Throwing native exceptions across the JSI boundary
**Why wrong:** RN's JSI binding doesn't translate arbitrary native exceptions cleanly. Hermes: opaque crash. Violates NATIVE-07.
**Instead:** `try { ... } catch (const std::exception& e) { throw jsi::JSError(rt, e.what()); }` at every host-function outermost layer.

### Anti-Pattern 6: Calling `installBindings` more than once on the same runtime
**Why wrong:** Two `GeneratorState` objects bound to the same runtime; old lambdas still reachable. Monotonicity broken across the seam.
**Instead:** Idempotency guard: `if (rt.global().hasProperty(rt, "__FastUid_uuidv7")) return;`. Document the contract.

## Integration Points

### External Services

| Service | Integration | Notes |
|---|---|---|
| iOS `Security.framework` (`SecRandomCopyBytes`) | Linked via podspec `s.frameworks = "Security"` | Returns `errSecSuccess` (0) on success. Sanitize errors. |
| Android `getrandom(2)` | Direct via `<sys/random.h>`; available since Android API 28. Below that, fall back to `/dev/urandom`. | Handle EINTR with retry budget. Do NOT use `GRND_NONBLOCK` — we WANT blocking until kernel pool initialized for cryptographic quality. |
| RN TurboModule infra (new-arch) | iOS: `RCTTurboModule` protocol; Android: Kotlin `@ReactModule` + JNI | Codegen spec is intentionally minimal: `installJSIBindings(): boolean`. |
| Reanimated worklet runtime | Optional peer dep; `installOnRuntime` only works if Reanimated is installed | Verify against current Reanimated version in Phase 1; the worklet-runtime install pattern shifted across 3.x → 4.x. |

### Internal Boundaries

| Boundary | Communication | Notes |
|---|---|---|
| `cpp/core/` ↔ `cpp/platform/` | Synchronous `fill_random` call | Core depends on platform; not the reverse. |
| `cpp/bindings/` ↔ `cpp/core/` | Synchronous calls passing `GeneratorState&` | Bindings own no state. |
| `cpp/install.cpp` ↔ `cpp/bindings/` | Bindings export factory functions returning `jsi::Function` | Install is the only thing touching `rt.global()`. |
| `ios/FastUid.mm` ↔ `cpp/install.cpp` | Single C call: `fastuid::installBindings(rt)` | iOS shim < 30 lines. |
| `android/.../JNI.cpp` ↔ `cpp/install.cpp` | Same single C call | JNI shim < 50 lines. |
| `src/index.ts` ↔ `globalThis.__FastUid_*` | Read globals, re-export typed | Underscore-prefixed names are private. |

## Surfaced Gaps Not Yet Decided (for Phase 1 design review)

1. **Idempotency of `installBindings`.** Should the C++ entry guard against double-install? Recommend YES via `rt.global().hasProperty` check.
2. **Batch size cap.** What's max `n`? Recommend `1 << 24` (16M) with `jsi::JSError` for over-cap.
3. **Worklet install API name.** `installOnRuntime()` (recommended — runtime-agnostic naming) vs. `installForWorklets()` vs. `installOnUIRuntime()`.
4. **Reanimated peer-dep version range.** Verify which Reanimated versions stably expose the runtime-install API. Likely floor: 3.6+; confirm in Phase 1.
5. **TurboModule codegen spec name.** `NativeFastUid` (proposed) → produces `RNFastUidSpec`. Verify against codegen conventions.
6. **CMakeLists.txt location.** `android/CMakeLists.txt` (root) vs. `android/src/main/cpp/CMakeLists.txt`. Recommend the latter.
7. **Standalone C++ test harness.** Recommend `cpp/tests/` with its own `CMakeLists.txt`, runnable via `cmake -B build && cmake --build build && ./build/fastuid_tests`.
8. **CSPRNG error logging.** Recommend logging raw `OSStatus` / `errno` via `os_log` / `__android_log_print` at error level; sanitize what crosses to JS.
9. **Spin-wait on counter overflow.** Sub-ms counter is 12 bits = 4096 values per ms. If exceeded, spin until next ms (or extend into rand tail bits per RFC §6.2 alternative). Decide which fallback.
10. **Hermes vs. JSC string-creation cost.** `createFromAscii` performance differs slightly between engines; document expected difference in benchmarks.

## new-arch Gotchas the JSI Template Handles vs. Leaves to Author

**Handled by template:**
- Codegen wiring (`codegenConfig` in package.json)
- iOS podspec with proper compiler flags + ARC
- Android Gradle + CMake skeleton
- `NativeXxx.ts` codegen spec scaffold
- TurboModule registration boilerplate (Kotlin + Obj-C)
- Example app pre-configured for new-arch

**Left to author:**
- The actual `installBindings(rt)` implementation
- Mutex / state design
- Error translation across the JSI boundary
- Reanimated UI-runtime install support (template generally targets only the default runtime)
- ASCII vs UTF-8 string allocation choice
- CSPRNG selection and platform-specific calls
- Idempotency guards against double-install
- Standalone C++ test target (template doesn't include one)
- CMake source globbing strategy for shared `cpp/` tree
- fbjni vs raw JNI choice on Android (template historically uses fbjni)

## Roadmap Implications

- **Phase 1 must include a verification subtask:** Run `create-react-native-library` JSI template, compare to proposed structure, note deltas. Verify Reanimated version-floor + current worklet-runtime install API.
- **Phase 2 (Generator core) should ship with a standalone C++ test target** — not just Jest tests. RFC-9562 conformance (QA-01) and 10M monotonicity test (QA-02) are vastly faster in native and don't need RN involvement.
- **Phase 3 should include the `installBindings` idempotency guard** as part of the install-entry implementation, not as a separate task.
- **Phase 6 (TS API + worklet install)** is the riskiest phase aside from Phase 2 — Reanimated's runtime-install API is the area with the most ecosystem churn. Build slack for verification.
- **Phase 7's example app worklet test (QA-03)** is the integration test for Pattern 1 — if the runtime-agnostic install design is wrong, this is where it breaks. Treat it as a gate, not a checkbox.

## Open Questions

- Reanimated 3.6+ vs 4.x: which is the floor version, and what's the current canonical worklet-install API call?
- Does `create-react-native-library` JSI template currently support new-arch-only builds out of the box, or is some podspec/gradle stripping needed?
- For the standalone C++ test harness, which test framework? (Recommend Catch2 single-header — zero infrastructure cost.)
- ASCII allocation: is `jsi::String::createFromAscii(rt, ptr, len)` available on both Hermes and JSC at the RN versions targeted? (Believed yes since RN 0.71+, but verify.)
