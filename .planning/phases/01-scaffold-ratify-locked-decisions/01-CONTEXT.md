# Phase 1: Scaffold + ratify locked decisions - Context

**Gathered:** 2026-05-08
**Status:** Ready for planning

<domain>
## Phase Boundary

Bootstrap the repo from `create-react-native-library` JSI/C++ template; produce a clean scaffolded library + example app that builds green on iOS (`xcodebuild` Release on `macos-15`/Xcode 16) AND Android (`./gradlew assembleRelease` with NDK r26+, CMake 3.22, AGP 8.6+, JDK 17) with zero Bridge / old-arch code paths. Reconcile every `[VERIFY]` version pin in STACK.md to ground truth, lock the eight SUMMARY.md "Decisions Surfaced for Ratification" into PROJECT.md Key Decisions (`Outcome: Locked`), and answer the ten ARCHITECTURE.md "Surfaced Gaps" inline so that phases 2–7 execute against a fully-locked design.

**In scope:** repo scaffold, toolchain pins, the 18 ratification items, build-green proof on both platforms, base lint/typecheck wiring (`pnpm typecheck` + `pnpm lint` pass on the empty scaffold).

**Out of scope:** any `cpp/core/`, `cpp/platform/`, `cpp/bindings/` implementation (Phase 2/3); any host functions (Phase 3); any TypeScript public surface (Phase 5); any benchmarks (Phase 6); CI workflows (Phase 7) — except the absolute minimum needed to verify `pnpm typecheck` + `pnpm lint` pass.

</domain>

<decisions>
## Implementation Decisions

### Version Pin Floor Strategy

- **D-01:** `peerDependencies.react-native` floor is `>=0.78.0`. Skips RN 0.76/0.77 entirely. Rationale: RN 0.78 is the first release with the `@react-native-community/javascriptcore` fork stable and documented (relevant to the Hermes+JSC parity goal); JSI ABI on 0.78 is the current new-arch baseline contributors are most likely on. Affects STACK.md row "React Native peer ≥0.76.0" → must be re-pinned to `>=0.78.0`. Affects BUILD-04 implementation note ("clear init-time error message" applies to RN < 0.78, not RN < 0.76). CI matrix tests against 0.78.x and the latest stable (likely 0.79.x or 0.80.x at scaffold time).

- **D-02:** `peerDependenciesMeta.react-native-reanimated.optional: true` with version range `>=3.16.0` (open upper bound). Rationale: matches research stance "^3.16.0 (worklet-runtime API stable); accept ^4.x if released". Open upper avoids premature lock-out of Reanimated 4.x. The actual install path against current Reanimated source is verified in Phase 5 (already flagged in research/SUMMARY.md "Phases likely needing deeper research"). The library itself imports zero Reanimated headers in C++ (BUILD-06) — the peer-dep is a contract for consumers who use the worklet `installOnRuntime` feature, not a hard library dependency.

- **D-03:** `engines.node` floor is `>=22.11.0` (Node 22 active LTS). Rationale: cleaner toolchain for native-bridge libs (better native test runner, more stable native addon ABI than 20.x); contributors targeting JSI work are on 22+. Smaller contributor surface than 20 LTS but the project is intentionally a small surface. STACK.md row "Node ≥20.18.0 LTS" must be updated to `>=22.11.0`.

- **D-04:** iOS deployment target = whatever fresh CRNL + RN 0.78 templates scaffold (likely 15.1, possibly 16.0 by 2026 mid-year). Rationale: pure-C++ JSI library has near-zero floor cost; matching the scaffolded value avoids drift between the library podspec and the example app's `Podfile.lock`. Document the actual value in PROJECT.md Key Decisions when scaffolded. Do NOT override.

### Ratification Posture (carried forward — research recommendation)

- **D-05:** Bulk-accept the eight SUMMARY.md "Decisions Surfaced for Ratification" verbatim. Each row in PROJECT.md Key Decisions table flips `Outcome: Pending` → `Outcome: Locked` with the research-recommended choice:
  1. Counter overflow strategy → **advance `last_ms` by 1 ms** (matches RFC 9562 §6.2 + ULID spec + LiosK `uuidv7`)
  2. `FastUidError` code shape → `code ∈ "CSPRNG_FAILED" | "RUNTIME_NOT_INSTALLED" | "BATCH_TOO_LARGE" | "INTERNAL"` from day one
  3. `installOnRuntime` idempotency → **second call is a no-op via sentinel** (`__FastUid_installed__`)
  4. Branded TS types → opt-in `Uuid` / `Ulid` (`string & { readonly __uuid: unique symbol }`) alongside plain `string` returns
  5. ULID counter — **separate counter, shared mutex with UUIDv7** (avoids second lock; keeps spec semantics clean)
  6. Batch size cap → `MAX_BATCH = 1 << 17` (~131k)
  7. Worklet install API name → **`installOnRuntime`** (runtime-agnostic; matches API-05 in PROJECT.md)
  8. Process-static `std::shared_ptr<GeneratorState>` is the **deliberate documented exception to NATIVE-06** (must persist across Fast Refresh / runtime tear-down/recreate cycles)

- **D-06:** Bulk-accept the ten ARCHITECTURE.md "Surfaced Gaps Not Yet Decided" verbatim. Each gets a concrete answer in `.planning/research/ARCHITECTURE.md` § "Surfaced Gaps" (rewriting from "Recommend X" to "Locked: X"):
  1. `installBindings` idempotency → **YES via `rt.global().hasProperty(rt, "__FastUid_uuidv7")` check**
  2. Batch size cap → `1 << 17` (matches D-05.6 — single source of truth)
  3. Worklet install API name → `installOnRuntime` (matches D-05.7)
  4. Reanimated peer-dep version range → `>=3.16.0` open upper (matches D-02)
  5. TurboModule codegen spec name → `NativeFastUid` produces `RNFastUidSpec`
  6. CMakeLists.txt location → **`android/src/main/cpp/CMakeLists.txt`** (not the root)
  7. Standalone C++ test harness location → **`cpp/tests/`** with own `CMakeLists.txt` (Phase 2 creates it; not scaffolded in Phase 1)
  8. CSPRNG error logging → **raw `OSStatus` / `errno` logged via `os_log` / `__android_log_print` at error level**; sanitized category string crosses to JS
  9. Spin-wait on counter overflow → **counter overflow advances `last_ms` by 1 ms** (matches D-05.1; do NOT spin-wait, do NOT extend into rand-tail bits)
  10. Hermes vs JSC `createFromAscii` cost → **document expected difference in `docs/benchmarks.md` (DOC-02)**, no code branching

### Phase-1 Verification Depth (carried forward — minimum success criterion)

- **D-07:** Phase-1 close requires **compile-only** verification, matching success criterion 2 verbatim — `xcodebuild` Release scheme on iOS + `./gradlew assembleRelease` on Android both succeed; the example app launches on simulator/emulator without crash. JSI-seam smoke (a dummy `globalThis.__FastUid_ping` host function) is **deferred to Phase 4** when the real native glue lands and there's something meaningful to ping.

### `cpp/` Tree Scaffolding Strategy

- **D-08:** Phase 1 creates the `cpp/` directory but does NOT populate it with subdirectories or sources. Phase 2 creates `cpp/core/` + `cpp/platform/` + `cpp/tests/`; Phase 3 creates `cpp/bindings/` + `cpp/install.cpp`. Rationale: keep Phase 1 minimal; avoid creating empty directories that confuse `find` / glob behavior in Phase 2's CMake setup. `.gitkeep` is acceptable if needed for the empty `cpp/` to commit.

### `[VERIFY]` Marker Cleanup

- **D-09:** After every `[VERIFY]` in `.planning/research/STACK.md` is reconciled to a concrete pin, the `[VERIFY]` markers are **deleted in-place** (the resolved pin replaces the placeholder). STACK.md remains the source of truth for "what's pinned and why" — research provenance is preserved by git history, not by inline annotations. The eight rows in the "Confidence Per Major Recommendation" table that read `LOW` because of `[VERIFY]` are updated to `HIGH` once the pins are resolved against the actual scaffold output and live registry.

### Claude's Discretion

- The exact pin (e.g. `0.78.0` vs `0.78.2`) for any toolchain element where research recommended a range and the answer must come from running CRNL or `npm view` at scaffold time — researcher/planner picks the latest stable patch within the locked minor.
- Husky / commitlint / release-it / `del-cli` exact patch versions — accept whatever CRNL scaffolds; do not override.
- Choice between fbjni and raw JNI on Android (ARCHITECTURE.md "Left to author") — researcher/planner reads CRNL's current Android template output and follows whatever pattern it generates, biased toward the convention `react-native-mmkv` v3 uses.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project & Requirements (locked contract)
- `.planning/PROJECT.md` — Constraints, Context, Key Decisions (12 rows currently `Pending`; Phase 1 flips eight to `Locked` per D-05). MUST read full file.
- `.planning/PROJECT.md` § "Key Decisions" — table to flip `Pending → Locked` with concrete `Outcome` text.
- `.planning/REQUIREMENTS.md` — v0.1 requirements; Phase 1 maps to BUILD-01, BUILD-02, BUILD-03, BUILD-04, BUILD-05, BUILD-07, QA-04, QA-06 (8 requirements).
- `.planning/REQUIREMENTS.md` § "Out of Scope" — explicit anti-features; do NOT add capabilities listed here.
- `.planning/REQUIREMENTS.md` § "Traceability" — must update Phase 1 requirements from `Pending` after this phase ships.
- `.planning/ROADMAP.md` § "Phase 1: Scaffold + ratify locked decisions" — full success criteria (5 items) verbatim.

### Research (Phase-1 source of truth — MUST read in full)
- `.planning/research/SUMMARY.md` § "Decisions Surfaced for Ratification" — the eight items D-05 commits to (counter overflow, error code shape, idempotency, branded types, ULID counter, batch cap, install API name, process-static counter).
- `.planning/research/ARCHITECTURE.md` § "Surfaced Gaps Not Yet Decided (for Phase 1 design review)" — the ten items D-06 commits to.
- `.planning/research/ARCHITECTURE.md` § "Recommended Project Structure" — file layout the scaffold MUST converge on (single `cpp/` tree at repo root; podspec + CMake glob into it; NOT the CRNL default of duplicate platform-specific source trees).
- `.planning/research/ARCHITECTURE.md` § "new-arch Gotchas the JSI Template Handles vs. Leaves to Author" — what CRNL gives you and what's still TODO.
- `.planning/research/STACK.md` — entire file. Every `[VERIFY]` marker must be resolved per D-09. Specifically the rows for: `react-native` (D-01 sets to `>=0.78.0`), `react-native-reanimated` (D-02 sets to `>=3.16.0`), Node (D-03 sets to `>=22.11.0`), iOS deployment target (D-04 matches CRNL output), NDK / CMake / AGP / Gradle / JDK / `compileSdk` / `targetSdk` / `minSdk` (accept what RN 0.78 + CRNL scaffold; pin to scaffold output).
- `.planning/research/STACK.md` § "What NOT to Use" — anti-features list; the scaffold must NOT include any of these (Turbo Module template, HostObject, old-arch shims, `createFromUtf8` for ASCII output, etc.).
- `.planning/research/STACK.md` § "Verification Checklist (before Phase 1 scaffolding)" — bash-level verification commands the planner should turn into plan tasks.
- `.planning/research/PITFALLS.md` § Pitfall 14 ("Build hygiene") — `-std=c++17` in BOTH `compiler_flags` AND `pod_target_xcconfig`; `-Werror` scoped to internal CI only via `WARNINGS_AS_ERRORS` CMake option (BUILD-05); no Reanimated headers in `cpp/` (BUILD-06 — Phase 4, but the Phase-1 scaffold must not introduce any).
- `.planning/research/FEATURES.md` § "Anti-features" — same anti-feature list as STACK.md; do not regress.

### Specs (irreversible decisions Phase 1 ratifies, even though impl lands later)
- RFC 9562 §5.7 (UUIDv7 layout) — relevant to D-05.1 (counter overflow advances ms).
- RFC 9562 §6.2 (Method 2 monotonicity) — relevant to D-05.1 + D-05.5 (separate counter for ULID, shared mutex).
- Crockford base32 — relevant to D-05.5 (ULID alphabet excludes `I`, `L`, `O`, `U`).

### Tooling references (consulted at scaffold time)
- `create-react-native-library` GitHub README — canonical scaffold output for the JSI/C++ template; researcher / planner runs the CLI to inspect what it produces in 2026 (template wiring may have shifted).
- `react-native-mmkv` v3 source — gold-standard reference for the JSI install pattern, podspec compiler flags, Android JNI wiring, single-`cpp/`-tree pattern. `mrousavy/react-native-mmkv` on GitHub.
- `react-native-quick-crypto` source — secondary reference, especially for CSPRNG patterns and the larger CMake setup.
- `@react-native/babel-preset` (matches RN 0.78) — required preset for the example app; relevant to the Reanimated worklet plugin chain in Phase 5.

### Locked working agreements (CLAUDE.md / PROJECT.md)
- Conventional commits; **no Claude `Co-Authored-By` trailer** (`.planning/PROJECT.md` Constraints § "Style — Git").
- Biome (TS) + clang-format (C++) — single tool per language; no ESLint/Prettier (PROJECT.md Constraints).
- pnpm only — pin via `packageManager` field for Corepack (PROJECT.md Constraints).
- C++17 strict — no C++20 features (PROJECT.md Constraints).
- New-architecture only; old-arch unsupported by design (PROJECT.md Constraints; BUILD-04).

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- **None.** Greenfield repo. Only `CLAUDE.md` and the `.planning/` tree exist. Phase 1 creates the actual codebase.

### Established Patterns
- **None at code level.** Patterns to be applied (from research, not from existing code) are documented in `canonical_refs` above and on the day-zero PROJECT.md / SUMMARY.md / ARCHITECTURE.md trio.

### Integration Points
- **`package.json`**: peer-dep declarations (D-01, D-02), engines (D-03), `packageManager`, `files` whitelist, `exports` map.
- **`*.podspec`** (created by CRNL): `s.platforms` (D-04), `s.compiler_flags`, `s.pod_target_xcconfig`, `s.frameworks = "Security"`, source globs `"../cpp/**/*"`.
- **`android/build.gradle`** + **`android/src/main/cpp/CMakeLists.txt`** (D-06.6): NDK version, CMake C++17 settings, source globs `../../../../cpp/**/*.cpp`.
- **`tsconfig.json`**: extends `@react-native/typescript-config`, `strict: true`, zero `any` (QA-04).
- **`biome.json`**: line width 100, organize imports, no `console.log` rule (QA-06).
- **`.clang-format`**: Google or LLVM base, `Standard: c++17`, `IndentWidth: 2`, `ColumnLimit: 100`.
- **`react-native.config.js`**: presence drives autolinking discovery; mostly empty platform configs for pure-JSI lib.

</code_context>

<specifics>
## Specific Ideas

- **JSC parity story**: D-01's choice of `>=0.78.0` is partly motivated by the `@react-native-community/javascriptcore` fork being stable from 0.78. This means CI matrix can include a JSC job without ad-hoc hacks. Phase 7 owns the actual matrix; Phase 1 just sets the floor that makes it possible.
- **Reanimated 3.x vs 4.x**: D-02's open upper bound is intentional. Phase 5 (the worklet install) is already flagged as the second-highest risk in the project precisely because Reanimated's runtime install API churned 3.x→4.x. The peer-dep declaration is the contract; the actual install path verification is Phase-5 work.
- **Active LTS Node**: D-03's `>=22` (not `>=20`) is a contributor-experience choice — anyone hacking on JSI in 2026 is already on 22.

</specifics>

<deferred>
## Deferred Ideas

- **CI matrix details (full)** — Phase 7 owns this. Phase 1 only ensures `pnpm typecheck` + `pnpm lint` pass on the empty scaffold (success criterion 1).
- **arm64 CI runner specifics** (`macos-15` Apple Silicon Android emulator) — verify at Phase 7 setup time per research/SUMMARY.md "Gaps to Address During Planning/Execution".
- **react-native-directory submission schema** — verify at Phase 7 (REL-03) per research; schema evolves.
- **Standalone Catch2 harness implementation** — Phase 2 owns `cpp/tests/` per D-08.
- **Phase-1 verification beyond compile-only** — JSI-seam smoke (dummy `__FastUid_ping`) deferred to Phase 4 when native glue lands per D-07.
- **Husky + commitlint hook authoring** — accept what CRNL scaffolds; tweak in a later phase if needed.
- **"Confidence Per Major Recommendation" table (STACK.md)** — D-09 says flip to `HIGH` once pins are resolved; the planner adds a task to do this at the end of Phase 1.

</deferred>

---

*Phase: 1-Scaffold + ratify locked decisions*
*Context gathered: 2026-05-08*
