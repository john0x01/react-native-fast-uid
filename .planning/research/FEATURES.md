# Features Research — react-native-fast-uid

**Project:** react-native-fast-uid (UUIDv7 + ULID, JSI/C++17, RN new-arch)
**Researched:** 2026-05-04
**Confidence:** MEDIUM

> **Research Constraints:** External research tools were unavailable in this environment (`brave_search` / `firecrawl` / `exa_search` disabled in config; WebSearch / WebFetch / Bash denied). Findings derived from training-data familiarity (cutoff Jan 2026) with `uuid` (uuidjs/uuid v9+), `uuidv7` (LiosK), `react-native-uuid`, `react-native-mmkv`, `react-native-quick-crypto`, `react-native-reanimated` install patterns, RFC 9562, and the ULID spec. Confidence per claim is annotated; LOW-confidence items should be re-validated before code lock-in (especially error-surface decisions, which are irreversible).

---

## Feature Landscape

### Table Stakes (users churn / file issues if missing)

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| `uuidv7()` returning canonical 36-char hyphenated **lowercase** string | RFC 9562 §4 canonical form; every parser/DB column expects it. Postgres `uuid`, ORMs, regex validators all assume lowercase. | LOW | Already in API-01. Hardcode lowercase. **HIGH confidence.** |
| `ulid()` returning canonical 26-char Crockford base32 **uppercase** | ULID spec mandates Crockford uppercase; every reference impl (oklog/ulid, ulid npm, python-ulid) emits uppercase. | LOW | Already in API-03. **HIGH confidence.** |
| Strict per-process monotonicity within same ms | UUIDv7's whole pitch over v4 is lex-sortability. ULID spec §Monotonicity makes this explicit. | MEDIUM | Already in NATIVE-03. Hard part is mutex'd state across JS + worklet (NATIVE-04). **HIGH.** |
| CSPRNG-backed randomness (no `Math.random()`) | "UUID" implies cryptographic-quality random in 2026. | LOW | Already in NATIVE-01/02. **HIGH.** |
| TypeScript types out of the box | RN ecosystem is TS-default in 2026. | LOW | Already in QA-04. **HIGH.** |
| Hermes + JSC parity | Enterprise apps still pin JSC; JSI is engine-agnostic. | LOW | Already in Constraints. **HIGH.** |
| iOS + Android parity | #1 RN library complaint. | MEDIUM | NATIVE-01/02 + BUILD-02/03. **HIGH.** |
| Sync, no `await` | Whole point of choosing JSI. | LOW | Already in Constraints. **HIGH.** |
| Zero-config install (autolinking) | New-arch + autolinking handles it. | MEDIUM | CRNL template covers this. **HIGH.** |
| No peer-dep on `react-native-get-random-values` | The native CSPRNG *replaces* the polyfill. | LOW | Implied by NATIVE-01/02. **HIGH.** |
| README install + 3-line example above the fold | Adoption decided in 30 seconds. | LOW | Already in DOC-01. **HIGH.** |
| MIT license badge | Enterprise legal review. | LOW | DOC-03. **HIGH.** |
| Works in **release** builds, not just debug | JSI binding lifetime + R8/ProGuard footguns. | MEDIUM | Already in CI-03. **HIGH.** |
| Reasonable behavior on duplicate `installOnRuntime` | Hot-reload re-install is universal. | LOW | Recommend idempotent (second call = no-op). **MEDIUM.** |

### Differentiators (competitive edge)

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Native JSI implementation | 5–20× faster than pure-JS `uuidv7`/`uuid`; matters in batch & worklet hot paths. | HIGH | Core value. **HIGH.** |
| `uuidv7.batch(n)` / `ulid.batch(n)` | **No other RN UUID library exposes batch.** Saves N JSI hops; real win for sync engines (WatermelonDB, Replicache, PowerSync) minting IDs by the thousand. | LOW (already speced) | API-02/04. **HIGH.** |
| Worklet-runtime install | Reanimated worklets currently can't generate IDs without `runOnJS`. **No competitor offers this.** Enables IDs in gesture handlers, animations, scroll callbacks. | HIGH | API-05. The 2-runtime mutex (NATIVE-04) is the implementation challenge. **HIGH.** |
| Strict cross-runtime monotonicity | If JS thread + UI worklet both call simultaneously, IDs still globally sortable per process. | MEDIUM | NATIVE-03/04. **HIGH.** |
| Branded TS types `Uuid` and `Ulid` (opt-in) | Prevents string-mixing bugs. ~10 lines of types, zero runtime cost. | LOW | Recommend including. **MEDIUM** — see TS Types probe. |
| New-arch-only with explicit messaging | Positive signal in 2026 (old-arch sunsetting). | LOW | BUILD-04 + README. **HIGH.** |
| Tree-shakeable named exports | Importing only `uuidv7` shouldn't ship `ulid` JS thunk. | LOW | Don't write `export default {...}`. **HIGH.** |
| Reproducible benchmark script | Most RN libs ship marketing numbers nobody can reproduce. | MEDIUM | DOC-02. **HIGH.** |
| react-native-directory listing with new-arch + worklet flags | Directory's filter UI sorts by new-arch; tags drive discoverability. | LOW | REL-03; verify schema at submission. **MEDIUM.** |
| Header-only C++ core, vendorable | Power users vendor `.h` into their own JSI libs (mmkv-style). | LOW | Already implied by Constraints. Document in README. **MEDIUM.** |
| Drop-in migration story for `uuid.v4()` / `crypto.randomUUID()` users | Same `string` return shape; one-line import-site delta. | LOW | Doc-only. **HIGH.** |

### Anti-Features (refuse — extends/validates PROJECT.md out-of-scope)

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| `uuidv7({ prefix: 'usr_' })` Stripe-style prefixes | Stripe-style branded IDs are popular. | A prefixed UUID isn't a UUID — breaks Postgres `uuid` columns, RFC validators. Once accepted, half the users compose `prefix + uuidv7()` in JS anyway and you've added native surface for nothing. | `` `usr_${uuidv7()}` `` in user code. |
| `uuidv7.timestamp(uuid)` introspection | "What time was this generated?" | Parsing concern, **explicitly out of scope per PROJECT.md** — confirms exclusion. Once you ship one helper, you'll be asked for `isValidUuidv7`, `version()`, etc. — endless surface. | Document bit layout + link to RFC 9562 parser; or future companion package. |
| Custom epoch (`uuidv7({ epoch })`) | "Save bits." | **RFC 9562 §5.7 fixes the epoch as Unix ms.** Custom epoch = non-compliant IDs no other tool can parse. Breaks interop. | Refuse. |
| Timestamp injection in public API | Test mocking. | Already covered: PROJECT.md says test-only internal. **Confirms exclusion.** | Internal test hook only; mock system clock in user tests. |
| Async API (`await uuidv7()`) | Best-practice muscle memory. | Defeats JSI purpose. **Confirms exclusion.** | Sync only — call out in README. |
| **Lowercase ULID / RFC 4648 base32** | URL aesthetics; users confuse Crockford with RFC 4648. | ULID spec is **Crockford uppercase**. Lowercase doesn't round-trip canonical parsers; RFC 4648 is a different alphabet (`A–Z, 2–7` vs Crockford excluding `I,L,O,U`). Either creates an interop disaster. **HIGH confidence: refuse both.** | `.toLowerCase()` in user code if needed. Document uppercase Crockford output. |
| Pre-allocated buffer for `batch(n, buffer)` | Performance theater — "avoid GC." | JSI strings are `jsi::String` allocations the engine owns. A user-provided `Uint8Array` forces JS-side hex-encoding, which is *slower* than `jsi::String::createFromAscii` directly. | `Array<string>` is the right call. Defer raw-bytes API to v0.2 if asked. |
| UUIDv1/v3/v4/v5 generation | Completeness. | **Confirms PROJECT.md exclusion.** | `react-native-uuid` / `crypto.randomUUID()`. |
| `getRandomValues` polyfill | "Why not consolidate?" | **Confirms exclusion.** Owned by `react-native-get-random-values`. | Direct users there. |
| Old-arch support | Long-tail enterprise. | **Confirms exclusion.** Old-arch JSI install is fundamentally different. | Pin new-arch. |
| UUIDv6 / v8 / custom variants | Spec completeness. | **Confirms exclusion.** v6 deprecated; v8 needs config surface conflicting with "pure generator." | Refuse for v0.x. |
| Collision detection / registry | "Just in case." | A library cannot detect collisions in another process/device. **Dataset concern. Confirms exclusion.** | DB `UNIQUE`, app-level dedup. |
| **`Math.random()` fallback when CSPRNG fails** | "Make it always work." | **Silent downgrade to non-CSPRNG = security CVE waiting to be filed.** RFC 9562 §6.9 mandates cryptographic random. | **Throw `jsi::JSError`.** Never fall back. |
| **Returning `null` on CSPRNG failure** | "Don't crash my app." | `string \| null` infects every caller; most callers will skip the check. **Once shipped, irreversible.** | **Throw `jsi::JSError`.** A genuine CSPRNG failure is catastrophic — fail loud is correct. |
| String parsing / validation utilities | Symmetry with generation. | **Confirms PROJECT.md exclusion.** Different perf profiles, different error surfaces. | Separate package later. |
| Sortable encoding output (e.g., ULID encoded as UUID format) | "Best of both worlds." | Confuses identifiers — a ULID-as-UUID isn't either spec. | Refuse. Pick one. |
| Custom alphabet for ULID | URL-safety variants. | ULID spec is Crockford-only. Custom alphabet = no longer a ULID. | Use `nanoid` for custom alphabets. |
| Configurable monotonicity strictness | "I don't need it, make it faster." | Method 2 monotonic counter has no measurable cost vs non-monotonic. | Refuse. Always monotonic. |

---

## Specific Probes from the Brief

### API surface — is `uuidv7()` + `uuidv7.batch(n)` enough?
**YES for v0.1. Confirms PROJECT.md.** Two real workloads (single mint, bulk mint). `prefix` belongs in user code. `timestamp(uuid)` is parsing — exclusion is correct. Custom epoch breaks RFC 9562. Matches what `uuid` v9+ landed on for v7. **HIGH confidence.**

### ULID variants — Crockford base32 case?
**Uppercase Crockford only. Refuse lowercase and RFC 4648.** ULID spec is unambiguous; every reference impl emits uppercase. Crockford excludes `I,L,O,U`; RFC 4648 is a different alphabet entirely. Allowing lowercase as an option creates DB-collation sort bugs. **HIGH confidence.**

### Monotonicity guarantees — what's the bar?
- **Same ms:** monotonic counter increments (RFC 9562 Method 2). Required.
- **New ms:** fresh random, counter resets. Required.
- **Cross-runtime (JS + worklet):** mutex around shared counter. Required (NATIVE-04).
- **Counter overflow within a ms:** RFC leaves it implementation-defined. **Recommend bump-timestamp** (matches ULID spec + LiosK's `uuidv7`). With Method 2's ~12-bit counter, overflow needs many thousand calls/ms — real but bounded. Drift is bounded and self-corrects. Alternative: throw — deterministic but app-crashing.
- **Cross-process / cross-device:** **NOT a guarantee.** Document clearly. Two devices at the same wall-clock ms collide on timestamp; random portion provides separation, not global ordering.
- **MEDIUM confidence** on bump-timestamp recommendation; HIGH that this needs locking before code.

### Worklet support — what does "worklet-friendly" mean?
1. **Callable from a worklet** without `runOnJS` — what `installOnRuntime(uiRuntime)` provides. Required.
2. **No JS-side allocations on worklet path** — JSI strings allocated on the calling runtime; automatic if installed correctly.
3. **Bounded allocation per call** — known constant memory (16-byte UUID + 36-byte string). Worklets under 60fps pressure break with unbounded allocation.

**Recommendation:** Claim all three in docs. Architecture supports all three. **HIGH confidence.**

### Batch API ergonomics — `Array<string>` vs pre-allocated buffer?
**`Array<string>` (matches PROJECT.md). Refuse buffer.** UUIDs/ULIDs are strings with semantic meaning; user-provided `Uint8Array` would force JS-side encoding, exactly the work JSI is supposed to avoid. **HIGH confidence.** Defer `uuidv7.bytes()` to v0.2 with evidence.

### Type safety — branded `Uuid` / `Ulid` or plain `string`?
**Provide both. Functions return `string`; export branded types as opt-in.**
```ts
export function uuidv7(): string;
export type Uuid = string & { readonly __uuid: unique symbol };
export type Ulid = string & { readonly __ulid: unique symbol };
```
Plain-string returns match `uuid`, `crypto.randomUUID()`, `react-native-uuid` — drop-in compat at ORM boundaries. Branded types are opt-in (`as Uuid`) — costs ~6 lines, zero runtime cost. **MEDIUM confidence** — stylistic call.

### Tree-shakeability
**YES — separate named exports, no default object.**
```ts
// Good
export function uuidv7(): string;
export namespace uuidv7 { export function batch(n: number): string[]; }
export function installOnRuntime(runtime?: object): void;

// Bad
export default { uuidv7, ulid, installOnRuntime };
```
Native `.so` ships in full regardless. Metro respects ES module named exports. **HIGH** on direction; **MEDIUM** on Metro's aggressiveness with the namespace pattern. Verify in example app.

### Error surface — CSPRNG failure (irreversible)
**Throw `jsi::JSError` with a stable, documented error message format.**

| Option | Reversibility |
|--------|---------------|
| Throw `jsi::JSError` | Can ADD return-null overload later if needed |
| Return `null` | `string \| null` infects every call site; **cannot change return type without major bump** |
| Crash process | App dies; can't recover |
| Math.random fallback | **Silent CVE vector** |

- Typed message: `RNFastUID: CSPRNG unavailable (${platform_errno})` so users can pattern-match.
- Aligns with NATIVE-07.
- **Android gotcha:** `getrandom(2)` *blocks* on cold boot until kernel entropy pool initializes — don't conflate slow with failed. NATIVE-02's `/dev/urandom` fallback handles this. Failure should fire only if **both** primary and fallback fail.
- **HIGH confidence.** Lock in v0.1.

### Migration path
**README migration table; no code-level compat shim.**
```ts
// Before                                  // After
uuid.v4()                              →   uuidv7()
crypto.randomUUID()                    →   uuidv7()
import { v4 } from 'uuid'              →   import { uuidv7 } from 'react-native-fast-uid'
```
All return `string`, all 36-char hyphenated lowercase — drop-in at type and value level. Behavioral diff (v7 sortable, v4 not) is the *whole point* — flag in docs. **HIGH confidence.**

### react-native-directory metadata
| Field | Value | Why |
|-------|-------|-----|
| `newArchitecture` | `true` | Filter UI |
| `expoGo` | `false` | **Critical** — JSI native code can't run in Expo Go; pre-empt confused issues |
| Other platforms (`fireos`/`tvos`/`web`/`windows`/`macos`) | omit/`false` | Don't claim untested platforms |
| License badge | MIT | Enterprise legal |
| GitHub topics | `react-native`, `uuid`, `ulid`, `uuidv7`, `jsi`, `worklet` | Discoverability |

**MEDIUM confidence** — verify schema field names at submission (REL-03).

---

## Feature Dependencies

```
uuidv7() ─requires→ CSPRNG (NATIVE-01/02)
              └──→ monotonic counter (NATIVE-03)
                        └─requires→ mutex (NATIVE-04)

uuidv7.batch(n) ─requires→ uuidv7() core
                      └──→ JSI Array allocation (NATIVE-05)

ulid()/ulid.batch(n) ─requires→ CSPRNG + counter + Crockford encoder

installOnRuntime(runtime?) ─enables→ all four generators
                            ─requires→ NATIVE-06 (no global state)
                            ─requires→ NATIVE-04 (mutex shared across runtimes)

Worklet support (API-05) ─requires→ installOnRuntime accepting non-default runtime
                                ─requires→ mutex (NATIVE-04)

Branded TS types ─enhances→ return types (opt-in; no conflict)

Tree-shaking ─requires→ named exports (no default object)

Error surface (throw) ─requires→ NATIVE-07; conflicts with return-null (mutex choice)
```

**Critical dependency notes:**
- The mutex (NATIVE-04) is the single most important correctness primitive — gates worklet support, the key dividend.
- No-global-state (NATIVE-06) gates safe `installOnRuntime` re-invocation under hot-reload.
- Throw-vs-return-null and `export default`-vs-named-exports are mutually exclusive choices on the **same call site / module shape** — once shipped, hard to reverse without major bump.
- **Open dependency question — ULID counter shared with UUIDv7 counter, or separate?** Recommend **separate counter, same mutex**. PROJECT.md NATIVE-03/04 imply but don't lock this. **MEDIUM confidence.**

---

## MVP Definition

**v0.1.0 (locked PROJECT.md scope is correct):**
- `uuidv7()`, `uuidv7.batch(n)`, `ulid()`, `ulid.batch(n)`, `installOnRuntime(runtime?)`
- TS types, iOS+Android parity, MIT, README + benchmark, directory listing — all already in PROJECT.md.

**Recommend adding to v0.1 (small, high-leverage, fits the long-weekend scope):**
1. **Branded TS types `Uuid` and `Ulid`** as opt-in named exports (~6 lines, no runtime cost).
2. **Migration table in README** ("from `react-native-uuid`", "from `crypto.randomUUID()`", "from `uuid` npm") — ~10 lines markdown.
3. **Explicit `expoGo: false`** + Expo Go behavior documented in README.
4. **Documented error format** for `jsi::JSError`.
5. **Idempotent `installOnRuntime`** — second call = no-op. Decide and document.
6. **Counter-overflow behavior decision** (recommend bump-timestamp; matches ULID spec).

**v0.2.x (defer until evidence):**
- `uuidv7.bytes(): Uint8Array` / `ulid.bytes(): Uint8Array` (trigger: ≥3 issues asking).
- Header-only vendoring docs (trigger: a user asks).
- macOS / Windows / Web platforms (trigger: user demand + tested).

**v1.0+ (future):**
- UUIDv8 / custom variant — only with clear user pattern.
- Companion parsing package `react-native-fast-uid-parse` (trigger: ≥10 parsing issues).
- WASM build for RN Web parity.

---

## Validation of PROJECT.md Locked Scope

| PROJECT.md element | Validated? | Notes |
|---------------------|------------|-------|
| `uuidv7()` API-01 | YES | Canonical 36-char lowercase is right |
| `uuidv7.batch(n)` API-02 | YES | Real differentiator |
| `ulid()` API-03 | YES | Crockford uppercase is correct |
| `ulid.batch(n)` API-04 | YES | Same reasoning |
| `installOnRuntime(runtime?)` API-05 | YES | Critical differentiator |
| Out of scope: v1/v3/v4/v5 | YES | Covered elsewhere |
| Out of scope: `getRandomValues` polyfill | YES | Owned by `react-native-get-random-values` |
| Out of scope: async API | YES | Defeats JSI |
| Out of scope: old-arch | YES | RN's direction |
| Out of scope: timestamp injection | YES | Test-only is right |
| Out of scope: parsing/validation | **YES — confirms exclusion** | Different perf profile; defer to companion package |
| Out of scope: v6 | YES | Deprecated draft |
| Out of scope: v8 / custom | YES | Conflicts with "pure generator" framing |
| Out of scope: collision registry | YES | Dataset concern |

**No scope gaps that should be added to v0.1.**

**Decisions surfaced for locking before code (gaps in PROJECT.md):**
1. **Counter overflow within a ms** — recommend bump-timestamp.
2. **CSPRNG failure error message format** — NATIVE-07 implies `jsi::JSError` but doesn't lock the message format.
3. **Idempotency of `installOnRuntime`** — recommend idempotent.
4. **Branded TS types** — recommend opt-in alongside plain string.
5. **ULID counter — shared or separate from UUIDv7?** — recommend separate counter, same mutex.

All five are clarifications/small additions, not scope changes.

---

## Competitor Feature Matrix

| Feature | `uuid` npm | `uuidv7` (LiosK) | `react-native-uuid` | **`react-native-fast-uid`** |
|---------|-----------|-------------------|---------------------|-------------------------------|
| UUIDv7 generation | yes (v9+) | yes | no | **yes (native JSI)** |
| ULID generation | no | no | no | **yes (native JSI)** |
| Batch API | no | no | no | **yes** |
| Native (non-JS) | no | no | no | **yes (C++/JSI)** |
| Worklet support | n/a | no | no | **yes** |
| Strict same-ms monotonicity | impl-defined | yes | n/a | **yes** |
| Cross-runtime monotonicity | n/a | n/a | n/a | **yes** |
| CSPRNG | via crypto.getRandomValues | via crypto.getRandomValues | via `react-native-get-random-values` | **native `SecRandomCopyBytes` / `getrandom`** |
| Sync API | yes | yes | yes | **yes** |
| TypeScript | yes | yes | yes | **yes (strict, branded opt-in)** |
| Tree-shakeable | yes | yes | mostly | **yes** |
| Timestamp parsing | `parse()`, `version()` | separate | no | **no (out of scope)** |
| New-arch only | n/a | n/a | works on both | **yes** |

**Confidence:** HIGH on high-level feature presence/absence; MEDIUM on current version-specific behaviors.

---

## Roadmap Implications

1. **Decisions to lock at design time (before C++):** error message format, counter-overflow behavior, install idempotency, ULID-vs-UUID counter sharing, branded-TS-types decision. None are scope changes.
2. **Module shape locked at first publish:** named exports vs default object, return type (plain `string` vs branded). Once `0.1.0` publishes, these are major-version-breaking to change.
3. **Phase ordering implication:** the mutex (NATIVE-04) is the linchpin — gates worklet support, cross-runtime monotonicity, install safety. Build it early.
4. **Documentation phase has more weight than typical:** migration table, error format, Expo Go behavior, vendoring docs, benchmark methodology — all README/docs work with outsize adoption impact relative to engineering cost.

## Open Questions

1. **`uuid` npm v11+ exact `v7()` monotonicity behavior** — does it bump-timestamp or throw on counter saturation?
2. **react-native-directory submission schema** as of submission date — verify field names.
3. **Metro tree-shaking aggressiveness** for `uuidv7.batch` (function + namespace merge pattern) — verify in example app.
4. **Reanimated worklet runtime install API** — confirm exact entry-point shape (this has churned 3.x → 4.x).
5. **`getrandom(2)` cold-boot blocking behavior** on target Android API levels.
