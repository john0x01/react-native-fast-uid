# Pitfalls Research

**Domain:** React Native pure-JSI/C++17 library — UUIDv7 + ULID generation; Reanimated worklet runtime support; new-architecture only; iOS + Android.
**Researched:** 2026-05-04
**Confidence:** MEDIUM-HIGH (training-data + ecosystem knowledge; live web/Context7 lookup was unavailable in this run; spot-verify primary sources before locking)

> Scope note: every pitfall below is specific to *this* library's surface area (synchronous `createFromHostFunction` bindings, ASCII string allocation on hot paths, dual-runtime install for default JS + Reanimated UI, platform CSPRNG, RFC-9562 Method-2 monotonicity, mutex-protected counter, new-arch only, Hermes + JSC). Generic C++/RN advice is excluded.

---

## Critical Pitfalls

### Pitfall 1: Capturing `jsi::Runtime&` by reference in a `HostFunction` that may outlive the runtime

**What goes wrong:** A `jsi::HostFunctionType` captures the install-time `jsi::Runtime&` (or anything tied to it — `PropNameID`, cached `jsi::String`, cached `jsi::Function`) in its lambda capture list. On reload (Fast Refresh / dev reload / runtime tear-down), the original runtime is destroyed but the host function value can briefly survive in a stray reference. Result: dangling reference, use-after-free, or "runtime mismatch" assertion in Hermes debug builds.

**Why it happens:** `HostFunctionType` is `jsi::Value(jsi::Runtime& rt, ...)` — the runtime is *passed in at call time*. Authors new to JSI ignore the parameter and reuse a captured `Runtime&` from install. It "works" in tests because the runtime hasn't been torn down yet.

**How to avoid:**
- The `jsi::Runtime&` parameter passed to your host function is the *only* runtime you may use. Never capture a runtime reference, never store one in a struct member.
- Capture *only* POD: shared mutex/counter state via `std::shared_ptr`, atomic ints, `std::array<uint8_t, N>`. Never capture `jsi::Value`, `jsi::String`, `jsi::Function`, or `PropNameID` across calls — all are runtime-bound.
- Construct `PropNameID::forAscii(rt, "...")` *inside* the host function on each call.
- Run a Fast Refresh / reload cycle in the example app — surfaces the bug fast.

**Warning signs:** Crash on second JS reload that didn't happen on first run; ASan / Hermes debug "use after free" on `Runtime::~Runtime`; cryptic `facebook::jsi::JSError` after Reanimated re-installs.

**Phase to address:** **C++ install pattern** (the very first JSI binding). Lock the rule via code-review checklist: "no `jsi::Runtime&` capture, no `jsi::*` value capture across calls."

---

### Pitfall 2: Treating Reanimated UI runtime as "the same JS context with a different name"

**What goes wrong:** Author installs bindings on the default RN JS runtime, expects them to appear automatically inside `runOnUI(() => { uuidv7() })`. They don't. Worklets execute on a *separate* `jsi::Runtime` — typically a *separate Hermes instance* with its own heap, globals, host-function table. The worklet sees `ReferenceError: uuidv7 is not defined`. Authors then "fix" via worklet-shareable closures — which crashes because `jsi::Function` isn't shareable across runtimes.

**Why it happens:** The "JS thread / UI thread" mental model from old-arch suggested one shared context. Worklet runtime is genuinely different (Reanimated 3+ allocates with `jsi::makeWorkletRuntime`). Hermes supports multiple instances per process, fully isolated.

**How to avoid:**
- Design the install function to accept `jsi::Runtime&` and be called *once per runtime*: once on default JS runtime, once on each worklet runtime.
- Expose via TS API as `installOnRuntime()` (default JS install) plus a Reanimated-aware variant that calls Reanimated's `runOnRuntime(workletRuntime, () => { /* native install */ })`.
- All runtime-bound state lives *per runtime*. Counter/mutex *can* be process-global because they're POD.
- Verify with QA-03: generate IDs from a Reanimated worklet **without** any JS-thread call first.

**Warning signs:** `uuidv7 is not a function` only inside `useAnimatedStyle`/`useAnimatedReaction`/`runOnUI`; crash deep inside Reanimated `WorkletRuntime`; Hermes assertion `Runtime mismatch in JSIValue`.

**Phase to address:** **Reanimated runtime install + QA-03 worklet test** — make this an early end-to-end feature, before perf work.

---

### Pitfall 3: Capturing `jsi::Value` / `jsi::String` / `jsi::Function` across host-function invocations

**What goes wrong:** Author caches a `jsi::String` of `"-"` or a `jsi::Function` reference between calls. On the next call (sometimes after GC), the cached value is invalid — Hermes' GC has moved or freed the underlying object. Crash with hard-to-debug Hermes internal error.

**Why it happens:** JSI values are *not* GC-rooted by C++ ownership. Hermes uses moving GC; JSC differs and may *appear* to tolerate the bug, hiding it until Hermes runs.

**How to avoid:**
- Build every JSI value fresh inside the host function from POD. Path: 16 bytes → 36 ASCII chars in `char[37]` stack buffer → `jsi::String::createFromAscii(rt, buf, 36)`. No caching.
- For batch: build `jsi::Array<jsi::String>` inside one call via `setValueAtIndex`, return it. Never persist across calls.

**Warning signs:** Random crash after GC pressure; tests pass on JSC, fail on Hermes; crash inside `hermes::vm::HermesValue`.

**Phase to address:** **C++ core / API-01 + API-02**. Ban caching of JSI values in C++ style guide.

---

### Pitfall 4: Install-hook timing — installing too early, or being re-invoked on Fast Refresh

**What goes wrong:**
1. **Too early:** Calling install before runtime is fully bootstrapped — `JSI runtime is null` or `cannot get global object`.
2. **Re-install on Fast Refresh:** Each JS reload re-runs the install module. If the C++ side blindly overwrites globals, it allocates new closures, leaks old `HostFunctionType` shared state — *N* counter instances disagreeing.

**Why it happens:** RN new-arch JSI bootstrap has non-trivial ordering. Fast Refresh re-runs JS modules; authors don't realize C++ has no idempotency.

**How to avoid:**
- Trigger install *from JS* (TurboModule call). Guarantees the runtime exists.
- Make install idempotent: check `rt.global().hasProperty(rt, "__fastUidInstalled__")` — if present, return immediately.
- Counter state in `static std::shared_ptr<MonotonicState>` constructed once; subsequent installs reuse same state via sentinel.
- Test: hammer Cmd-R (iOS) / RR (Android) ten times, then call `uuidv7()` — should still work.

**Warning signs:** `JSI` errors in `RCTBridgeModule install` logs; memory growth across reloads; monotonicity test fails after a reload sequence.

**Phase to address:** **API-05 (`installOnRuntime`)** + dev-loop smoke test before benchmarks.

---

### Pitfall 5: Hermes vs JSC `createFromAscii` lifetime/copy-semantics confusion

**What goes wrong:** `createFromAscii` *copies* bytes into the runtime — but the pitfall is the *assumption*: passing a non-ASCII byte to `createFromAscii` is undefined behavior in release and asserts in Hermes debug. Confusing it with `createFromUtf8` ships latent bugs.

**Why it happens:** Hermes optimizes ASCII strings (1 byte/char) vs UTF-16. UUIDs/ULIDs are pure ASCII (`0-9a-f-` / Crockford `0-9A-Z` minus `I,L,O,U`).

**How to avoid:**
- Hand-roll the formatter to write only into the ASCII-safe alphabet. Static_assert + debug-mode runtime assert that every output byte is in range.
- Run tests under Hermes debug mode — `createFromAscii` will `abort()` on a non-ASCII byte.
- Use `createFromAscii` deliberately; never migrate to `createFromUtf8` without a benchmark.

**Warning signs:** Hermes debug abort with "non-ASCII byte"; garbled output in batch; benchmark numbers wildly different between Hermes and JSC.

**Phase to address:** **NATIVE-05** (ASCII string allocation). Add debug-only ASCII alphabet invariant in formatter unit tests.

---

### Pitfall 6: Mutex on the hot path destroying batch performance

**What goes wrong:** Naive impl: `std::mutex` locked once *per ID* inside batch loop. 10k lock/unlock pairs per batch, 10k cache-line bounces. Single-thread micro-benchmark looks fine; under realistic two-thread load, lock contention dominates.

**Why it happens:** Per-ID overhead from a mutex (uncontended ~10-30ns; contended 100ns–μs with context switch) is comparable to or larger than UUID generation work itself.

**How to avoid:**
- **Lock once per batch.** Inside `batch(n)`: acquire mutex, snapshot `now_ms` and counter starting position, advance counter `n` times *locally*, release mutex, format strings *outside* the lock.
- Single-call `uuidv7()`: single lock/unlock is fine.
- Run a 2-thread contention benchmark in CI (one thread `batch(1000)`, another `uuidv7()` in tight loop) — verify p99 stays in budget.
- Avoid false sharing: `alignas(64)` on counter struct.

**Warning signs:** Single-thread fast, two-thread half throughput; Reanimated frame drops during JS-thread batch; `perf` shows >20% in `pthread_mutex_lock`.

**Phase to address:** **NATIVE-04 (mutex)** + **benchmark phase**. Two-thread contention bench must exist before "fast" claim.

---

### Pitfall 7: UI-thread (worklet) blocking on the JS-thread's mutex causes frame drops

**What goes wrong:** JS thread mid-batch holds mutex. UI thread calls `uuidv7()` from worklet during `useAnimatedReaction`. UI thread blocks on `mutex.lock()` for ms. Frame deadline (16.67ms @60Hz, 8.33ms @120Hz) missed.

**How to avoid:**
- **Bound the critical section.** Lock-once-per-batch pattern: lock → read clock → reserve counter range → unlock → format outside. 100k batch holds lock for ~μs, not ms.
- Alternative: `std::atomic<uint64_t>` packed `(timestamp_ms, counter)` CAS-loop. Lock-free; UI thread never blocks. Adds complexity. Decide based on bench data; ship mutex first.
- Document: "if user wants worklet-safety, don't run >10k batch on JS thread while UI animations need IDs."

**Phase to address:** **NATIVE-04 critical-section design + EX-04 (worklet example)**.

---

### Pitfall 8: `getrandom(2)` blocking on Android early-boot or returning EINTR

**What goes wrong:**
1. **Blocks** on cold boot until kernel CSPRNG pool seeded — app startup hang.
2. **Returns -1 / EINTR** if a signal arrives mid-syscall — naive `if (getrandom(...) != n) abort();` triggers false-positive crash.
3. **libc symbol missing** below NDK API 28 in some configs — need raw `syscall(SYS_getrandom, ...)`.

**How to avoid:**
- EINTR retry loop: `while ((n = getrandom(buf, len, 0)) == -1 && errno == EINTR);`
- For early-boot: pass `GRND_NONBLOCK`, on `EAGAIN` fall back to `/dev/urandom` (always available; RFC 9562 explicitly allows "any source of unique values").
- Open `/dev/urandom` once via `open(O_CLOEXEC)`, cache fd, `read()` per call with EINTR retry.
- Use `syscall(__NR_getrandom, buf, len, GRND_NONBLOCK)` directly for portability.
- Test on fresh emulator boot.

**Warning signs:** App hangs at startup on cold boot; EINTR crash logs; linker error `undefined reference to getrandom`.

**Phase to address:** **NATIVE-02 (Android CSPRNG)** — lock in EINTR loop + `/dev/urandom` fallback + raw syscall path *before* benchmarking.

---

### Pitfall 9: `SecRandomCopyBytes` failure mode silently ignored

**What goes wrong:** `SecRandomCopyBytes` returns `errSecSuccess` (0) or non-zero. If failure is ignored, the buffer is undefined → UUID contains uninitialized stack/heap garbage — potentially leaking memory contents into IDs (RFC 9562 §6.9 requires cryptographic randomness).

**How to avoid:**
- Always check return: `if (SecRandomCopyBytes(...) != errSecSuccess) { throw jsi::JSError(rt, "csprng failed"); }` — throw inside host function so it crosses the JSI boundary as JS exception (NATIVE-07).
- Zero the buffer before calling — defense in depth.

**Warning signs:** IDs containing recognizable patterns (string fragments, repeating bytes); static-analyzer warnings about ignored return.

**Phase to address:** **NATIVE-01 + NATIVE-07**. Test the error path: mock CSPRNG to fail; assert `jsi::JSError` raised.

---

### Pitfall 10: RFC 9562 Method-2 monotonicity misimplementation

**What goes wrong:** Method 2 (sub-ms counter, RFC 9562 §6.2) is subtle. Spec-violating bugs:

1. **Counter reset to fully-random value on new ms:** Two IDs in adjacent ms can sort out of order. Spec: pick from a *limited* upper-bit range.
2. **Counter overflow:** RFC says advance the timestamp by 1ms. Naive impls wrap → next ID smaller than previous (catastrophic).
3. **Clock going backwards (NTP, DST, manual change):** `std::chrono::system_clock` is *not* monotonic. RFC requires detection: freeze timestamp at last-seen-max + bump counter.
4. **Method 1/2/3 confusion:** Mixing Method 1's "fully random" with Method 2's counter structure produces bugs.

**How to avoid:**
- Strict Method 2:
  - State `(last_ms, counter)` under mutex.
  - On call: `now_ms = max(last_ms, system_clock_ms())` — software monotonic clamp.
  - If `now_ms == last_ms`: `counter++`. If overflow: `last_ms = now_ms + 1`, reseed counter to random with **high bit clear**.
  - If `now_ms > last_ms`: `last_ms = now_ms`; reseed counter to random with **high bit clear**.
- "High bit clear" leaves room for `counter++` on same ms without overflow.
- QA-01: independent RFC 9562 validator + RFC appendix test vectors.
- QA-02: 10M-monotonicity stress test.
- Use `system_clock` (Unix epoch) + software monotonic clamp; **not** `steady_clock` (unspecified epoch).

**Warning signs:** 10M monotonicity test fails (often after counter overflow or NTP step); independent validator reports IDs out of order.

**Phase to address:** **NATIVE-03** — *highest-risk correctness item*. Test-first: write QA-02 (10M stress) and clock-fault-injection test *before* implementation.

---

### Pitfall 11: Counter state surviving (or not) across JS reload — monotonicity breaks at module boundary

**What goes wrong:** Dev-mode reload: install hook re-runs. If C++ counter state is owned by install closure and recreated on every install, `last_ms` resets — next ID could have earlier `last_ms` than previously-issued, violating monotonicity *across the reload boundary*.

**How to avoid:**
- Counter state is `static std::shared_ptr<MonotonicState>` at file scope (or anonymous namespace). First install constructs it; subsequent installs reuse via sentinel check (Pitfall 4). State persists.
- This is *the one* exception to NATIVE-06 ("no global state"): the counter is a process-singleton because monotonicity is process-level, not per-runtime.
- Document the exception in code.
- Test: full app reload + first ID after reload has greater `(timestamp, counter)` than last ID before.

**Phase to address:** **NATIVE-03 + NATIVE-06 reconciliation**. Make explicit: NATIVE-06 means "no per-call global state, no double-install state," but the counter is process-level by design.

---

### Pitfall 12: C++ exceptions escaping the JSI boundary

**What goes wrong:** `std::bad_alloc`, `std::system_error`, or any non-`jsi::JSError` leaks out of host function → undefined behavior → typically `std::terminate` and app crash.

**How to avoid:**
- Top-level wrapper at every host function:
  ```cpp
  try { ... }
  catch (const jsi::JSError&) { throw; }
  catch (const std::exception& e) { throw jsi::JSError(rt, e.what()); }
  catch (...) { throw jsi::JSError(rt, "unknown native error"); }
  ```
- Validate batch size early (`n <= MAX_BATCH`) — throw `jsi::JSError("batch too large")` rather than risking OOM.

**Warning signs:** `terminate called after throwing an instance of std::bad_alloc` with no JS handler; native crash, not JS exception.

**Phase to address:** **C++ install pattern (very early)**. Make the wrapper a macro/template applied to every host function.

---

### Pitfall 13: Hermes string interning + GC pressure under batch allocation

**What goes wrong:** 1M batch allocates 1M `jsi::String` + Array of 1M slots. Hermes young-gen heap fills, GC mid-batch. 1-10ms pauses; "first batch slow, subsequent fast" because heap warms up.

**How to avoid:**
- Cap batch size — `batch(n)` rejects `n > MAX_BATCH` (suggested 100k) with `jsi::JSError`. Document loop pattern for larger jobs.
- Benchmark warm + steady-state separately; document methodology in DOC-02.
- Avoid creating Object/Array instances inside per-ID loop other than strings.

**Warning signs:** `batch(1000000)` works once, second call OOMs; debug-vs-release benchmark numbers wildly different.

**Phase to address:** **API-02 (batch) + DOC-02**. Add batch-size cap before performance work.

---

### Pitfall 14: Build-time pitfalls — podspec missing `-std=c++17`, NDK ABI mismatch, Reanimated headers transitively breaking build

**What goes wrong (sub-modes):**
1. **iOS:** Missing `-std=c++17` in `compiler_flags` *and* `pod_target_xcconfig.CLANG_CXX_LANGUAGE_STANDARD` → C++17 features fail to compile in consumer projects.
2. **Android:** `set(CMAKE_CXX_STANDARD 14)` from a template; failing to filter NDK ABIs to `arm64-v8a, armeabi-v7a, x86, x86_64`.
3. **Reanimated transitive headers:** Including `<reanimated/...>` requires consumer to have Reanimated. Solution: don't include Reanimated headers in C++; accept `jsi::Runtime*` from JS-land via `installOnRuntime(runtimePtr)`.
4. **Hermes header inclusion:** `<hermes/hermes.h>` is *not* part of public RN headers. Use only `<jsi/jsi.h>`.
5. **`-Werror` leaking:** If in `s.compiler_flags`, applies to consumer's pod build — benign warning blows up consumer's CI. Scope `-Werror` to internal builds only.

**How to avoid:**
- iOS: `s.compiler_flags = '-std=c++17'`; `s.pod_target_xcconfig = { 'CLANG_CXX_LANGUAGE_STANDARD' => 'c++17', 'CLANG_CXX_LIBRARY' => 'libc++' }`.
- Android: `set(CMAKE_CXX_STANDARD 17)`, `set(CMAKE_CXX_STANDARD_REQUIRED ON)`, `set(CMAKE_CXX_EXTENSIONS OFF)`. Build for all four ABIs in CI.
- Don't include Reanimated headers in C++.
- Scope `-Werror` to internal CI matrix only — `WARNINGS_AS_ERRORS` CMake option default OFF.
- CI smoke job: fresh RN app + `npm install` your tarball + build both platforms.

**Phase to address:** **BUILD-01 through BUILD-05** + dedicated **consumer-app smoke build CI job**.

---

### Pitfall 15: Release-mode benchmark divergence — measuring the wrong thing in CI

**What goes wrong:** Benchmarks run via `assembleDebug`. Hermes optimizations, DCE, inlining radically different vs release. README numbers (DOC-01) are debug — 5–50× slower than release. Users cite slow numbers; competitors look better.

**How to avoid:**
- Benchmarks **must** run against release builds. `bench` script invokes `assembleRelease` (Android) and Release scheme (iOS).
- CI captures release benchmark numbers; fail job if regression >20% from baseline.
- DOC-02 explicitly states "release-mode, Hermes, device specs X" — DOC-01 numbers must match.
- Disable React DevTools / Flipper / Metro overlay during bench runs.

**Phase to address:** **CI-03 + DOC-02 + DOC-01**. Lock release-mode benchmarking before any number is published.

---

### Pitfall 16: GitHub Actions emulator differences — arm64 vs x86_64 + iOS pod cache corruption

**What goes wrong:**
1. **Android x86_64 vs arm64:** Default GH Actions Linux runners are x86_64. Many JSI/C++ bugs manifest only on arm64 (alignment, stack-size). Users on real arm64 devices crash; CI never sees.
2. **iOS pod cache:** If cache key doesn't include lockfile hash, stale pod replays — build "passes" with old code; real failure ships.
3. **R8/ProGuard stripping:** Release minification strips C++ symbols. `consumerProguardRules` not shipped → consumer's release build strips JNI registration → `UnsatisfiedLinkError`.

**How to avoid:**
- Apple Silicon GH Actions runners now support arm64 emulation — run at least one CI job on arm64.
- iOS pod cache: `key: pods-${{ hashFiles('**/Podfile.lock') }}`. Run `pod install --repo-update` deterministically.
- Ship `consumer-rules.pro` with `-keep class com.fastuid.** { *; }` in the AAR.
- CI smoke: install published tarball into fresh RN app; release build; assert `uuidv7()` returns valid output.

**Phase to address:** **CI-01 through CI-03**. Add arm64 CI job and lock pod cache keys before stamping a release.

---

### Pitfall 17: npm publish — accidentally shipping `example/`, missing `files`, no provenance

**What goes wrong:**
1. **Shipping `example/`:** Default `npm publish` includes everything not `.gitignore`d. Bloats package; consumers' bundlers pick up the example's `metro.config.js`.
2. **Missing `files` whitelist:** Ships test files, `.planning/`, `.cursor/`, etc.
3. **Missing prebuilt JS:** Without `prepublishOnly: tsc`, ships only `src/` — consumers using Babel-only can't compile.
4. **No npm provenance:** `npm publish --provenance` (GH Actions OIDC) marks package as built from verifiable source — supply-chain protection.

**How to avoid:**
- Explicit `"files": ["src", "lib", "android", "ios", "cpp", "*.podspec", "README.md", "CHANGELOG.md", "LICENSE"]`.
- `prepublishOnly` script that runs build + typecheck + tests.
- `npm pack && tar -tzf *.tgz | head -50` as publish-checklist step. Eyeball the contents.
- Publish with `npm publish --provenance --access public` from GH Actions.
- Defense-in-depth `.npmignore` for `.planning/`, `.cursor/`, `.github/`, `example/`.

**Phase to address:** **REL-01**. Pre-publish checklist with `npm pack` inspection.

---

### Pitfall 18: API design — error semantics and TS typing baked in too early

**What goes wrong:**
1. **Error semantics:** First version throws `Error("uuidv7: csprng failed")`. v0.2 wants to differentiate error types — consumers' `e.message.includes("csprng")` breaks.
2. **Branded TS types:** Forcing `string & { __brand: "Uuidv7" }` breaks ergonomics; hard to remove.
3. **Batch return shape:** Locking `string[]` means future "fastpath" using `Uint8Array` can't replace it without new function name.
4. **Leaking platform errors:** Exposing `errno`/`OSStatus` in messages tells attackers about platform internals.

**How to avoid:**
- Define error codes from the start: `class FastUidError extends Error { code: "CSPRNG_FAILED" | "RUNTIME_NOT_INSTALLED" | "BATCH_TOO_LARGE" | "INTERNAL"; }`. Consumers match on `.code`.
- Avoid branded types in v0.1 — add later if community asks.
- Document v0.x may break; reserve guarantees for v1.0.
- Never include `errno`/`OSStatus` in user-facing messages. Log internally only.
- Reserve names: `uuidv7.batch(n)`. Future fast path → `uuidv7.batchBytes(n, buffer)` rather than overload.

**Phase to address:** **API-01 through API-04 design**. Lock the error model before implementation.

---

### Pitfall 19: react-native-directory submission — missing metadata, broken example, undeclared peer deps

**What goes wrong:**
1. Marking `expo: true` without testing — pure-JSI requires custom dev client; not Expo Go-compatible.
2. `newArchitecture: false` by default accidentally.
3. Unstated peer dep on Reanimated.
4. No example app screenshot.

**How to avoid:**
- `newArchitecture: true`. Verify the current schema.
- For Expo: `expoGo: false`, `devClient: true` (verify schema at submission).
- `peerDependencies`: react, react-native. `peerDependenciesMeta`: `react-native-reanimated: { optional: true }`.
- Add example app screenshot to README.

**Phase to address:** **REL-03**. Verify directory schema at submission time, not project start.

---

### Pitfall 20: Worklet thread + mutex contention causing deadlock-like behavior

**What goes wrong:** Subtle dual-runtime scenario: host fn on JS runtime takes mutex; host fn on worklet UI runtime tries to take same mutex. JS thread + RuntimeExecutor's worklet thread on different OS threads. `std::mutex::lock()` is well-defined — UI thread blocks. **Not a deadlock, but a frame-budget violation.** Worse: if user calls `runOnJS(uuidv7)` from a worklet that holds something, can construct real deadlock with separate mutex.

**How to avoid:**
- Use process-static `std::mutex` (not runtime-bound). Mutex lifetime = process lifetime.
- Critical sections short — bound by arithmetic only (Pitfall 7 reinforcement).
- Document explicitly: "Holding the mutex across `runOnJS`/`runOnUI` calls is not supported."
- CI test: fire `uuidv7()` from JS and Reanimated worklet simultaneously for 30s; assert no crash, no duplicates.

**Phase to address:** **NATIVE-04 + QA-03**.

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Mutex per-ID inside batch | Single-thread fine, bad multi-thread p99 | Lock-once-per-batch: snapshot counter range, format outside | >1k IDs/batch under any contention |
| Cross-runtime mutex contention from worklet | Frame drops during JS-thread batch | Bound critical section to arithmetic only; consider lock-free CAS for v0.2 | Concurrent UI animation + JS-thread batch |
| `createFromUtf8` on hot path | 2–3× slower than ASCII | Always `createFromAscii` for UUID/ULID alphabets | Inverted: always |
| Hermes GC pressure under unbounded batch | First batch fast, second slow, ms-pauses | Cap batch size ~100k; document loop pattern | `batch(n)` with n > ~500k |
| Debug-mode benchmarking in CI | README numbers don't match user-observed | Release-mode benchmarks only; CI gates on regression | Always misleading |
| Poll clock per ID | `clock_gettime` ~20ns vDSO; per-1M-batch adds up | Snapshot clock once per batch entry | >100k IDs/batch |
| Counter overflow advancing timestamp catastrophically | Monotonicity holds but UUIDs drift seconds into future | Reseed counter high-bit-clear; advance ms by 1 only when truly out of bits | Bursts ~10⁹ IDs/ms (synthetic) |
| False sharing of counter struct | Mysterious slowdown in production | `alignas(64)` on counter struct; isolate from other writeable state | High-core-count under contention |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Use `rand()` / `std::mt19937` instead of CSPRNG | Predictable IDs; UUID birthday-paradox protection lost | Platform CSPRNG only |
| Reuse a 16-byte CSPRNG read across multiple UUIDs | Correlated random bits between adjacent IDs | Fresh CSPRNG read per UUIDv7 (or per fresh-ms in Method 2 reseed) |
| Leak `errno`/`OSStatus` in error messages | Tells attacker about platform internals | Generic messages: "csprng failed" |
| Skip CSPRNG return-code check | Buffer contains uninitialized memory → leaks heap into IDs | Always check return; throw on failure; zero-init buffer defensively |
| Use `jsi::Value::asNumber` for batch size without bound check | `batch(Number.MAX_SAFE_INTEGER)` allocates 16PB → OOM crash | Validate `0 < n <= MAX_BATCH`; throw `jsi::JSError` |
| Embed device-specific entropy (MAC, IMEI) in IDs | Privacy leak; cross-app correlation | UUIDv7 = timestamp + random only; never add identifiers |
| `console.log` generated IDs in dev mode | IDs become correlation keys for crash logs | No `console.log` in shipped code (QA-06) |
| Make timestamps mockable via public API | Allows attacker to forge time-sortable IDs | Test-mode injection internal-only |

---

## "Looks Done But Isn't" Checklist

- [ ] **`uuidv7()` works on JS thread** — also from Reanimated `runOnUI` worklet without prior JS-thread call (QA-03).
- [ ] **Monotonicity test passes at 10M** — not just 10k (QA-02).
- [ ] **Release build benchmarks** — not debug-mode numbers.
- [ ] **CSPRNG return code checked** — inject CSPRNG-failure mock; assert `jsi::JSError`.
- [ ] **Idempotent install** — reload 10× and check monotonicity holds across.
- [ ] **No exception escapes JSI boundary** — inject `std::bad_alloc`; assert JS Error, not crash.
- [ ] **Both Hermes and JSC tested** — JSC swap-in.
- [ ] **arm64 + x86_64 both green in CI**.
- [ ] **`pod install` succeeds in fresh consumer app** — smoke-build CI from npm tarball.
- [ ] **Released package contents** — `npm pack && tar -tzf *.tgz` before publish.
- [ ] **react-native-directory metadata accurate** — `expoGo: false`, new-arch flag, Reanimated peer-dep.
- [ ] **README first-screen comprehension** — install/example/platforms/numbers — sub-30s reader test.
- [ ] **TS strict, no `any`** — `tsc --strict --noEmit` on full repo including tests.
- [ ] **No `console.log` in shipped code** — biome lint rule + grep before publish.
- [ ] **Worklet smoke test (QA-03) automated**.
- [ ] **Counter state persists across reload**.
- [ ] **Batch size cap** — `batch(Number.MAX_SAFE_INTEGER)` throws `jsi::JSError`, not OOM.
- [ ] **No Reanimated headers in C++** — build succeeds without Reanimated installed in a consumer.

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| 1. Capturing `jsi::Runtime&` | C++ install pattern (first JSI work) | Code-review checklist; Fast-Refresh smoke |
| 2. Worklet runtime is separate | Reanimated install (API-05) — schedule **early** | QA-03 worklet test without prior JS call |
| 3. Caching `jsi::Value` across calls | C++ core (NATIVE-05) | Hermes-debug abort test; cross-runtime test |
| 4. Install timing + Fast Refresh | API-05 install hook | Reload-loop smoke in dev mode |
| 5. `createFromAscii` lifetime/UTF-8 | NATIVE-05 | Hermes-debug formatter unit test |
| 6. Mutex per-ID destroying batch | NATIVE-04 + benchmark phase | 2-thread contention bench in CI |
| 7. UI-thread blocking on mutex | NATIVE-04 + EX-04 | Worklet + JS-thread parallel test |
| 8. `getrandom` EINTR / blocking | NATIVE-02 | Cold-boot emulator; EINTR-injection unit |
| 9. `SecRandomCopyBytes` return code | NATIVE-01 + NATIVE-07 | Mock-failure asserts `jsi::JSError` |
| 10. RFC-9562 Method-2 misimplementation | NATIVE-03 monotonic counter | QA-01 + QA-02 + clock-step injection |
| 11. Counter state lost on reload | NATIVE-03 + NATIVE-06 reconciliation | Reload-spanning monotonicity |
| 12. C++ exception escapes JSI | C++ core install pattern | Inject `std::bad_alloc`; assert JS Error |
| 13. Hermes GC pressure under batch | API-02 + DOC-02 | Batch-size cap; release-mode bench N×100k |
| 14. Build/packaging misconfig | BUILD-01–05 + consumer smoke build | Fresh-RN-app CI install smoke |
| 15. Release-mode benchmark divergence | CI-03 + DOC-01 + DOC-02 | Release bench in CI; gate on regression |
| 16. CI emulator + cache differences | CI-01–03 | arm64 + x86_64 matrix; lockfile cache key |
| 17. npm publish — files, provenance | REL-01 | `npm pack` inspection; provenance flag |
| 18. API design — error codes, branded types | API-01–04 design | Error-code enum; no branded types in v0.1 |
| 19. react-native-directory submission | REL-03 | Schema validation; `expo: true` claim verified |
| 20. Worklet + mutex contention | NATIVE-04 + QA-03 | Concurrent worklet + JS-thread stress |

---

## Recovery Strategies

| Pitfall | Recovery Cost | Steps |
|---------|---------------|-------|
| Captured `jsi::Runtime&` | LOW (dev) / HIGH (post-release) | Refactor host fns to per-call rt param; full search; patch release |
| Worklet runtime not supported | HIGH | API redesign accepting `jsi::Runtime*`; new install entry; semver-major bump if shipped |
| Mutex per-ID killing batch perf | MEDIUM | Refactor batch to lock-once; bench before/after; patch |
| `getrandom` EINTR crash | LOW | Add retry loop; patch within hours |
| `SecRandomCopyBytes` failure ignored | LOW | Add return check + `jsi::JSError`; patch |
| Method-2 monotonicity bug | HIGH (correctness regression) | Fix algorithm; re-run 10M test; ship patch; advise users on existing IDs may sort wrong against new |
| C++ exception escape | MEDIUM | Add try/catch wrapper to all host fns; patch |
| Released package shipping `example/` | LOW (cosmetic) | Add `files` whitelist; `npm unpublish` <24h or `npm deprecate` + new version |
| Release-mode benchmarks misreported | MEDIUM (trust) | Update README + DOC-02; explain in CHANGELOG |
| Reanimated peer dep undeclared | LOW | Move to `peerDependenciesMeta.optional`; document |

---

## Roadmap Implications

Three pitfalls dictate phase ordering:

1. **Reanimated dual-runtime install must be early.** Pitfall 2 forces install-API design before optimization — discovering the worklet-runtime constraint after building the JS-runtime path means an API redesign.
2. **Method-2 monotonicity must be test-first.** Pitfall 10 is the highest correctness risk; QA-01 and QA-02 should be written *before* NATIVE-03 implementation.
3. **Mutex critical-section design must precede benchmarks.** Pitfalls 6 and 7 mean published benchmark numbers depend on the lock-once-per-batch design existing first; never benchmark a per-ID-locking prototype.

Suggested phase ordering signal (phase names illustrative, reconcile with roadmap):

1. C++ install pattern (lifetime/exception/idempotency rules)
2. Single-call `uuidv7()` end-to-end (catches Pitfalls 1, 3, 4, 5, 9, 12)
3. Method-2 monotonic counter test-first (Pitfalls 10, 11) → unlocks correctness confidence
4. Dual-runtime install (Pitfall 2) → unlocks worklet path
5. Batch API + mutex critical-section design (Pitfalls 6, 7, 13) → unlocks benchmarks
6. Build/packaging hardening (Pitfall 14) → unlocks consumer-side smoke
7. CI hardening (Pitfalls 15, 16) → unlocks reliable benchmarks
8. Publish (Pitfalls 17, 18, 19) → ship

---

## Sources & Verification Notes

**Primary sources to verify against (recommended pre-NATIVE-03 work):**
- **RFC 9562** (UUIDv7 canonical, Oct 2024): https://datatracker.ietf.org/doc/html/rfc9562 — especially §5.7 (UUIDv7 layout) and §6.2 (Monotonicity, Methods 1/2/3). Test vectors in appendix.
- **React Native JSI headers**: `react-native/ReactCommon/jsi/jsi/jsi.h` — `HostFunctionType`, `createFromHostFunction`, `createFromAscii` semantics.
- **react-native-mmkv source**: `mrousavy/react-native-mmkv` — gold-standard JSI install pattern; verify dual-runtime handling in 2.x+ versions.
- **react-native-reanimated source**: `software-mansion/react-native-reanimated` — `createWorkletRuntime` (Reanimated 3+), `runOnRuntime`, `WorkletRuntime` C++ class.
- **react-native-quick-crypto**: `margelo/react-native-quick-crypto` — large-scale JSI lib for vendoring/CMake patterns; CSPRNG handling.
- **Hermes documentation**: https://hermesengine.dev — string optimization, GC behavior. Source at `facebook/hermes`.
- **Apple Security framework**: `SecRandomCopyBytes` reference. https://developer.apple.com/documentation/security/1399291-secrandomcopybytes
- **Linux man pages**: `getrandom(2)`, `urandom(4)` — EINTR semantics, blocking behavior, `GRND_NONBLOCK`.
- **NDK getrandom availability**: Bionic libc; verify symbol availability at minSdk 24+.
- **CocoaPods spec reference**: `compiler_flags`, `pod_target_xcconfig` — verify Xcode-version-correct values.
- **react-native-directory schema**: https://github.com/react-native-community/directory — verify schema at submission.
