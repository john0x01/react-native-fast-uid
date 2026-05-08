# Phase 1: Scaffold + ratify locked decisions - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-05-08
**Phase:** 1-Scaffold + ratify locked decisions
**Areas discussed:** Version pin floor strategy

**Areas presented but not selected:** Ratification posture, Phase-1 verification depth, `cpp/tests/` skeleton (each captured in CONTEXT.md `<decisions>` with research-recommended defaults — see D-05, D-06, D-07, D-08).

---

## Version Pin Floor Strategy

### Q1: `react-native` peer-dependency floor

| Option | Description | Selected |
|--------|-------------|----------|
| `>=0.76.0` (declared floor) | Hold the RN 0.76 floor from PROJECT.md / STACK.md. Broadest 2026 compatibility — every new-arch-default RN version supported. Largest test surface. Most consumer reach. | |
| Match fresh CRNL scaffold output | Run CRNL once, accept whatever floor it sets (likely RN 0.79+ in 2026). Smallest test surface; floor tracks CRNL's chosen baseline. May exclude consumers stuck on 0.76/0.77. | |
| `>=0.78.0` (sweet spot) | Skip 0.76/0.77 entirely. 0.78 has stable `@react-native-community/javascriptcore` fork (relevant to Hermes+JSC parity goal). Mid-ground reach with cleaner JSC story. | ✓ |

**User's choice:** `>=0.78.0` (sweet spot)
**Notes:** Explicit rationale: cleaner JSC parity story via the community fork stable from 0.78. Updates STACK.md row that previously read `>=0.76`.

### Q2: `react-native-reanimated` peer-dep range (clarified)

User initially asked "Why is reanimated a dependency here?" — clarified that it's an **optional** peer (`peerDependenciesMeta.optional: true`). It's listed because the `installOnRuntime(workletRuntime)` API takes a Reanimated worklet runtime; consumers using that feature need Reanimated installed. The library itself imports zero Reanimated headers in C++ (BUILD-06). Optional peer-dep is the conventional declaration of the compat constraint.

User then asked Claude to "stick with your recommended pick" without picking from the option list directly.

| Option | Description | Selected |
|--------|-------------|----------|
| `^3.16.0` (research floor) | Pin to research-recommended floor where worklet runtime install API is stable. Conservative — excludes 4.x until Phase 5 verifies. | |
| Defer with `*` until Phase 5 | Use `*` for now, pin when worklet install code lands in Phase 5. Less risk of premature lock-in. | |
| `>=3.16.0` (open upper) | Declare floor only, no upper bound. Lets consumers stay on latest. Matches the project intent "accept ^4.x if released". Phase 5 verifies the install path holds across 3.x and 4.x. | ✓ |

**User's choice:** Claude's recommended pick — `>=3.16.0` (open upper)
**Notes:** Open upper bound is intentional: matches the research stance "^3.16.0; accept ^4.x if released" and Phase 5 is already flagged as the verification point for the canonical install path against current Reanimated source.

### Q3: `engines.node` floor

| Option | Description | Selected |
|--------|-------------|----------|
| `>=20.18.0` (current LTS) | Matches STACK.md recommendation. Node 20 is current LTS until Apr 2026; broadest contributor reach. Required floor for builder-bob 0.30+ anyway. | |
| `>=22.11.0` (active LTS) | Node 22 is active LTS, has stable native test runner + better perf. Smaller contributor surface but cleaner toolchain (esp. for native-bridge libs). | ✓ |
| `>=20 \|\| >=22` (LTS-only floor) | Allow either LTS line. Most permissive and forward-compatible without supporting odd-numbered Node releases. Slight CI matrix complexity. | |

**User's choice:** `>=22.11.0` (active LTS)
**Notes:** Updates STACK.md row "Node ≥20.18.0 LTS" to `>=22.11.0`.

### Q4: iOS deployment target in podspec

| Option | Description | Selected |
|--------|-------------|----------|
| 15.1 (RN 0.76+ floor) | Matches the historical RN 0.76+ floor declared in STACK.md. Broadest consumer reach. RN 0.78 still ships with this floor. | |
| Match what CRNL scaffolds | Whatever CRNL + RN 0.78 templates pick. Lowest drift risk — your podspec exactly matches the example app's. Most predictable for CI. | ✓ |
| 16.0 (cleaner SwiftUI baseline) | Slight bump. No effect on the C++ binding code, but cleaner if test apps later use modern SwiftUI APIs. Marginally smaller install base. | |

**User's choice:** Match what CRNL scaffolds
**Notes:** Document the actual scaffolded value in PROJECT.md Key Decisions during scaffold execution.

### Q5: Continue or wrap up?

| Option | Description | Selected |
|--------|-------------|----------|
| More questions | Keep digging on version pins (Android NDK/AGP/Gradle/JDK pinning posture, react peer-dep, packageManager corepack pin, `[VERIFY]` cleanup). | |
| Explore other gray areas | Open the other Phase-1 gray areas (ratification posture, verification depth, cpp/tests/). | |
| I'm ready for context | Lock these four version-pin decisions and write CONTEXT.md. Other Phase-1 gray areas get research-recommended defaults. | ✓ |

**User's choice:** I'm ready for context
**Notes:** User opted to lock the four discussed pins and apply research-recommended defaults to the other three Phase-1 gray areas (ratification posture → bulk-accept SUMMARY/ARCH recommendations; verification depth → compile-only; cpp/tests/ → pure-defer to Phase 2).

---

## Claude's Discretion

- The exact patch pin (e.g. `0.78.0` vs `0.78.2`) for any toolchain element where research recommended a range and the answer must come from running CRNL or `npm view` at scaffold time — researcher / planner picks the latest stable patch within the locked minor.
- Husky / commitlint / release-it / `del-cli` exact patch versions — accept whatever CRNL scaffolds; do not override.
- Choice between fbjni and raw JNI on Android — researcher / planner reads CRNL's current Android template output and follows whatever pattern it generates, biased toward `react-native-mmkv` v3's convention.

## Deferred Ideas

- CI matrix details (Phase 7).
- arm64 CI runner specifics — verify at Phase 7 setup time.
- react-native-directory submission schema — verify at Phase 7 (REL-03).
- Standalone Catch2 harness implementation — Phase 2 (`cpp/tests/`).
- JSI-seam smoke (`__FastUid_ping`) — Phase 4, when native glue lands.
- Husky + commitlint hook authoring beyond CRNL defaults — later phase if needed.
- STACK.md "Confidence Per Major Recommendation" table flips to `HIGH` once pins resolved (planner task at end of Phase 1).
