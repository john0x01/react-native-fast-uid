<!-- GSD:project-start source:PROJECT.md -->
## Project

**react-native-fast-uid**

A React Native open-source library that generates **UUIDv7** (RFC 9562) and **ULID** identifiers via a pure C++17/JSI binding — no bridge crossings, no async, no native APIs beyond platform CSPRNG access. New-architecture only. Built for offline-first sync layers, log buffers, mutation queues, and content-addressed stores that need time-sortable IDs on hot paths.

**Core Value:** Synchronous, time-sortable, RFC-9562-compliant ID generation from JS, the UI worklet runtime, or anywhere a JSI runtime exists — fast enough that the cost is invisible in the hottest paths a React Native app has.

### Constraints

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
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->
## Technology Stack

## Recommended Stack
### Core Technologies
| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| **React Native** | `>=0.76.0` (peer); test against `0.77.x` and `0.78.x` `[VERIFY]` | Host platform; provides JSI runtime, new-arch infra | RN 0.76 is the new-arch-on-by-default release. Setting `>=0.76` as the lower peer bound is the standard cutoff for "new-arch only" libraries shipping in 2026. Test against the latest stable to catch JSI ABI drift. |
| **C++ standard** | C++17 | Native implementation | Locked. NDK r26+ and Xcode 15+ both ship C++17 cleanly; C++20 modules / `<bit>` features have inconsistent NDK support. |
| **Hermes** | bundled with RN (default JS engine since 0.70) | JS runtime | First-class JSI host. Worklet-runtime (Reanimated) is built on Hermes by default. Library MUST work on Hermes; JSC is secondary. |
| **JSC (JavaScriptCore)** | bundled / opt-in via `@react-native-community/javascriptcore` for RN 0.79+ `[VERIFY]` | Fallback JS runtime | Apple-platform fallback. As of late-2025 Meta deprecated bundled JSC; community fork is the supported path. Library should compile against the JSI headers RN exposes regardless of engine — mostly a CI-matrix concern, not a code concern. |
| **react-native-reanimated** | `^3.16.0` (worklet-runtime API stable); accept `^4.x` if released `[VERIFY]` | UI/worklet runtime registration target | Reanimated 3.16+ exposes a stable C++ API for accessing the UI worklet `jsi::Runtime`. Make Reanimated `peerDependenciesMeta.optional` so consumers without it still install. |
| **Node.js** | `>=20.18.0` LTS (`>=22` preferred) | Toolchain runtime | Node 18 is EOL April 2025; Node 20 is current LTS, Node 22 is active LTS. |
| **pnpm** | `^9.x` (locked) `[VERIFY exact 9.x]` | Package manager | Use `packageManager` field for Corepack pinning. |
| **TypeScript** | `^5.5.0` (5.6+ preferred for `--isolatedDeclarations`) `[VERIFY]` | Type system | Strict mode locked. `--isolatedDeclarations` (TS 5.5+) speeds up `.d.ts` emission for library publishers. |
### Supporting Libraries
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| **react-native-builder-bob** | `^0.40.x` `[VERIFY]` | Build orchestrator (TS → CJS+ESM, codegen wiring) | Standard. CRNL scaffolds it. **Still the canonical choice in 2026; no replacement has emerged.** |
| **@react-native/babel-preset** | matches RN minor | Babel preset for example app | Required for Metro / Reanimated worklet plugin chain. |
| **@biomejs/biome** | `^1.9.x` (or 2.x if released) `[VERIFY]` | TS lint + format | Locked. Configure `"organizeImports": { "enabled": true }`, `lineWidth: 100`. |
| **clang-format** | clang 18+ ships with Xcode 16; pin a `.clang-format` config | C++ format | Locked. Use Google or LLVM base style + `Standard: c++17`, `IndentWidth: 2`, `ColumnLimit: 100`. Pin the style (not the binary) so contributors converge. |
| **Jest** | `^29.7.x` (RN preset compatibility) `[VERIFY 30 status]` | JS unit test runner | RN's `jest-preset` historically lags Jest 30. Stick to 29 unless `react-native/jest-preset` declares 30 support. |
| **release-it** | `^17.x` `[VERIFY 18 status]` | Release automation | What CRNL scaffolds. Combined with `@release-it/conventional-changelog` for conventional-commit driven changelogs. **No need for `changesets` — that's for monorepos.** |
| **@release-it/conventional-changelog** | `^8.x` `[VERIFY]` | Changelog generation | Pairs with conventional commits. |
| **commitlint + @commitlint/config-conventional** | `^19.x` `[VERIFY]` | Conventional-commit enforcement | Hooks via husky (also CRNL-scaffolded). |
| **husky** | `^9.x` | Git hooks | Pre-commit: `biome check` + `tsc --noEmit`. |
| **del-cli** | `^6.x` | Cross-platform `rm -rf` for clean scripts | CRNL scaffolds it. |
### Native Build Toolchain
| Tool | Version | Purpose | Notes |
|------|---------|---------|-------|
| **Xcode** | 16.x (16.2+) `[VERIFY]` | iOS toolchain | Floor: Xcode 15. Recommend Xcode 16 in CI (`macos-15` runner). |
| **iOS deployment target** | `15.1` | iOS floor | RN 0.76+ floor. Match it; no benefit dropping lower. `s.platforms = { :ios => '15.1' }`. |
| **CocoaPods** | `>=1.15.2` (1.16.x preferred) `[VERIFY]` | iOS dep manager | Use `bundle install` + `bundle exec pod install` (Gemfile in CRNL scaffold). |
| **Ruby** | `3.2.x` | CocoaPods runtime | Pin via `.ruby-version`. |
| **Android NDK** | `r26d` (26.x) `[VERIFY r27 status]` | C++ toolchain | RN 0.76+ uses NDK 26 by default. NDK 27 introduces 16KB page-size requirement (Android 15) — track but don't block. `ndkVersion = "26.1.10909125"`. |
| **CMake** | `3.22.1` (AGP default) `[VERIFY]` | Native build system | Don't bump unless needed; `find_package(ReactAndroid CONFIG REQUIRED)` works on 3.22. Use `set(CMAKE_CXX_STANDARD 17)` and `set(CMAKE_CXX_EXTENSIONS OFF)`. |
| **Gradle** | `8.10.x` (matches RN 0.76+ scaffold) `[VERIFY]` | Android build orchestrator | Use what RN scaffolds; don't drift. |
| **Android Gradle Plugin (AGP)** | `8.6.x` `[VERIFY]` | Android build plugin | Matches Gradle 8.10. Avoid AGP 8.7+ until verified against RN's Prefab consumer config. |
| **JDK** | `17` (Temurin) | JVM toolchain | RN 0.73+ floor. Latest LTS is 21 but Gradle plugin ecosystem is most stable on 17 in 2026. |
| **Android `compileSdk` / `targetSdk` / `minSdk`** | `35` / `34` / `24` | SDK versions | RN 0.76+ defaults. |
### Testing Stack
| Tool | Version | Purpose | Notes |
|------|---------|---------|-------|
| **Jest** | `^29.7.x` | JS unit tests | UUID/ULID format, monotonicity, batch contract. Mock the native module via `__mocks__/react-native-fast-uid.ts` for pure-JS tests. |
| **`react-native` jest-preset** | matches RN peer | RN-aware Jest config | Required even for pure-JSI libs (TS/Babel transforms). |
| **Independent RFC-9562 validator** | hand-rolled in TS or use `uuid` (`v.validate` + version sniff) `[VERIFY which lib parses v7]` | Compliance test (QA-01) | A second implementation path. Don't validate v7 with your own parser — that's circular. The `uuid` JS package added v7 parse/validate around 10.x. |
| **gtest (Google Test)** | **not recommended for v0.1** | C++ unit tests | Skip. Library logic is small; full-stack JS tests against the built binary on simulator/emulator give better coverage and catch JSI-boundary bugs that pure-C++ unit tests miss. Flag for future milestone. |
| **Detox** | **optional / recommend skipping for v0.1** `^20.x` `[VERIFY]` | E2E for example app | Heavy CI cost; example app doesn't need full E2E for a generation library. A `tsx` script + `react-native run-ios`/`run-android` smoke run is sufficient. |
| **CI smoke approach** | `xcodebuild` build + `gradle assembleDebug`/`assembleRelease` | Build verification | CI requirements (CI-02, CI-03) only require build success — that's the right floor for v0.1. |
### Benchmark Stack
| Tool | Version | Purpose | Notes |
|------|---------|---------|-------|
| **mitata** | `^1.0.x` `[VERIFY]` | JS microbenchmarks | Modern, statistically-sound (warmup, GC pauses, percentile reporting). What new RN libs are adopting in 2025+ over `tinybench`. Run inside the example app via a benchmark screen so numbers reflect Hermes-on-device. |
| **tinybench** | `^3.x` `[VERIFY]` | Alternative JS benchmark | Valid alternative; simpler API, slightly less rigorous stats. Pick `mitata` for publishable percentiles; `tinybench` for minimal config. |
| **react-native-performance** | **not recommended** | App-level perf (TTI/render) | Wrong tool — out of scope for an ID generator. |
| **Reproducibility methodology** | document in `docs/benchmarks.md` (DOC-02) | — | Mirror `react-native-mmkv` and `react-native-quick-crypto`: device model, OS version, RN version, JS engine, release/debug, sample size, p50/p95/p99 (not just mean), exact bench script committed. **Always benchmark in `assembleRelease` / Release scheme** — Debug builds have JSI logging and missing optimizations that distort numbers by 5–50×. |
| **Comparison baselines** | `uuid` (`v7()`), `react-native-uuid`, `uuidv7` (npm) | Numbers in README (DOC-01) | Pin versions in the bench script; print them in the output table. |
### CI / Release Tooling
| Tool | Version | Purpose | Notes |
|------|---------|---------|-------|
| **GitHub Actions** | — | CI | CRNL scaffolds workflows. |
| **`actions/setup-node`** | `v4` | Node setup | With `cache: 'pnpm'`. |
| **`pnpm/action-setup`** | `v4` `[VERIFY]` | pnpm install | Pin pnpm version to match `packageManager` field. |
| **`actions/setup-java`** | `v4` (`temurin@17`) | JDK | For Android. |
| **`ruby/setup-ruby`** | `v1` (`bundler-cache: true`) | Ruby/CocoaPods | For iOS. |
| **`gradle/actions/setup-gradle`** | `v4` `[VERIFY]` | Gradle caching | Halves Android build time. |
| **macOS runner** | `macos-14` (Xcode 15) or `macos-15` (Xcode 16) `[VERIFY]` | iOS build | Match Xcode floor declared in podspec. |
| **release-it** | `^17.x` | Release | What CRNL scaffolds. Run via a `Release` workflow gated on manual dispatch. |
### TypeScript / Packaging Conventions (2026)
| Convention | Setting | Notes |
|------------|---------|-------|
| `tsconfig.json` | extends `@react-native/typescript-config` | Override for strict / `noUncheckedIndexedAccess`. Locked: `"strict": true`. |
| `package.json` `main` | `./lib/commonjs/index.js` | builder-bob output. |
| `package.json` `module` | `./lib/module/index.js` | ESM output. |
| `package.json` `types` | `./lib/typescript/src/index.d.ts` | Or dual-emit equivalent. |
| `package.json` `react-native` | `./src/index.tsx` | Lets Metro consume source directly (better stack traces, faster HMR for consumers using `wml`/`yalc`). |
| `package.json` `exports` | dual conditional export with `import`/`require`/`react-native`/`types` | Modern resolution. CRNL templates ship a working version — adopt as-is. |
| `react-native.config.js` | export `{ dependency: { platforms: { ios: {}, android: {} } } }` | Standard. For pure-JSI libs with no autolinked TurboModule, the platform configs are mostly empty objects — but the file's presence still drives autolinking discovery. |
| `peerDependencies` | `react`, `react-native`; `react-native-reanimated` as `peerDependenciesMeta.optional: true` | Consumers using only the JS runtime shouldn't be forced to install Reanimated. |
| `engines` | `node: ">=20"` | Match toolchain floor. |
| `packageManager` | `pnpm@9.x.x` `[VERIFY exact]` | Corepack-pinned. |
| `files` whitelist | `["src", "lib", "android", "ios", "cpp", "*.podspec", "react-native.config.js", "!**/__tests__", "!**/__fixtures__", "!**/__mocks__"]` | Ship source + built artifacts + native folders; exclude tests. |
## Installation
# Scaffold (interactive — pick "Native module" → "C++ for Android & iOS";
# this is the JSI/C++ template, NOT the Turbo Module template)
# Inside scaffolded repo:
# Reanimated as optional peer for the worklet install path
# Mark optional in package.json:
#   "peerDependencies":     { "react-native-reanimated": "*" }
#   "peerDependenciesMeta": { "react-native-reanimated": { "optional": true } }
# Benchmarks
- Type: **Native module**
- Languages: **C++ for Android & iOS** (this is the JSI/C++ template)
- Example app: **Vanilla** (not Expo — Expo's prebuild adds friction for a library example)
- Package manager: **pnpm**
- New architecture: **enabled / required**
## Alternatives Considered
| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| `create-react-native-library` JSI/C++ template | **react-native-nitro-modules** (mrousavy) | Nitro is a strong choice for new libs in 2026 (HostObject codegen, faster than TurboModules). **But:** it imposes its own install + spec model that hides the `jsi::Runtime` and `installOnRuntime` patterns this project is explicitly built to learn. Use Nitro for a production library where DX wins; use raw JSI here because **it's a JSI warm-up project** (PROJECT.md context). |
| `release-it` | **changesets** | Multi-package monorepos. Single-package libraries don't need changesets' versioning ledger. |
| `mitata` | **tinybench** | Tinybench has smaller API; fine if you don't need percentile/distribution analysis. |
| Biome | **ESLint + Prettier** | Existing repos with deep ESLint plugin investment. Greenfield: Biome wins on speed and one-tool config surface. |
| Husky | **Lefthook** | Switch if hook latency becomes a paper cut. Husky is fine for a small lib. |
| Jest 29 | **Vitest** | RN's jest-preset and worklet plugin chain are Jest-only. Don't fight Metro/Babel here. |
| iOS deployment 15.1 | **iOS 14 / 13** | No benefit; match RN's floor. |
## What NOT to Use
| Avoid | Why | Use Instead |
|-------|-----|-------------|
| **Turbo Module template** from CRNL | Scaffolds codegen spec + forces TurboModule registration; conflicts with pure-JSI `installOnRuntime` | JSI/C++ template (Native module → C++ for Android & iOS) |
| **`HostObject`** for the install API | No state to hold; HostObject overkill (locked) | `jsi::Function::createFromHostFunction` per-export |
| **Old-architecture / Bridge code paths** | Out of scope by design (BUILD-04) | Target new arch only; floor `react-native >= 0.76` |
| **`createFromUtf8`** for ASCII output | Slower; UTF-8 validation overhead unnecessary for hex/base32 (locked) | `jsi::String::createFromAscii` |
| **`Math.random()` / `arc4random`** for any random byte | Not CSPRNG-grade; violates RFC 9562 | `SecRandomCopyBytes` (iOS) / `getrandom(2)` (Android) — locked |
| **Global mutable state in C++** (file-scope counter / mutex) | Test isolation, multi-runtime install collisions, hot-reload weirdness | Mutex + counter in install closure (locked NATIVE-06) |
| **Throwing C++ exceptions across the JSI boundary** | Crashes RN; only `jsi::JSError` is contract-safe (locked NATIVE-07) | Catch C++ exceptions at boundary; rethrow as `jsi::JSError` |
| **C++20 features** (`std::format`, `<bit>`, modules, concepts, ranges) | NDK / Xcode toolchain support uneven (locked C++17) | C++17 idioms only |
| **`react-native-performance`** for benchmarks | Measures app perf (TTI, frame drops); wrong tool for microbenchmarks | `mitata` or `tinybench` inside example app |
| **Detox for v0.1** | Heavy CI cost; this lib has no UI surface | `xcodebuild` + `gradle assembleDebug/Release` build verification + Jest tests |
| **gtest for v0.1** | Adds C++ test toolchain complexity for very small native code surface | JS-side Jest tests against the loaded binary in the example app |
| **`yarn` / `npm` for this repo** | pnpm locked | pnpm only; pin via `packageManager` |
| **ESLint + Prettier** | Two tools, slower; Biome locked | Biome 1.9+ |
| **Manual `cmake_minimum_required(VERSION 3.4.1)`** copied from old RN templates | Outdated; modern Prefab consumer needs `>=3.18` | `cmake_minimum_required(VERSION 3.22.1)` to match AGP default |
| **Storing `jsi::Runtime*` references long-term** | Runtime can be torn down; dangling pointer crashes | Capture `Runtime&` in install scope only; no static caches |
| **`changesets`** | Overkill for single-package repo | `release-it` + `@release-it/conventional-changelog` |
| **Expo example app** | Prebuild friction for a JSI lib; bare RN is what consumers will use | Vanilla RN example (CRNL default) |
## Stack Patterns by Variant
- Provide `installOnRuntime(runtime?: object)` JS API.
- Internally call the same C++ install function with the worklet `jsi::Runtime&` (Reanimated exposes this via its C++ API; document the supported Reanimated minor in README).
- Mutex around the monotonic counter is what makes JS-runtime + UI-runtime concurrent install safe (NATIVE-04).
- Library autoloads on JS runtime via standard CRNL native init path.
- `installOnRuntime` becomes a no-op / convenience export; library still works.
- **Unsupported.** Throw a clear error in the JS init shim (`Unsupported React Native version. react-native-fast-uid requires RN >= 0.76 (new architecture).`). No graceful degradation (BUILD-04).
- Same C++ binary; JSI is engine-agnostic. CI matrix should build the example app once per engine to confirm `jsi::String::createFromAscii` perf is comparable. No code branches.
## Version Compatibility
| Package | Compatible With | Notes |
|---------|-----------------|-------|
| `react-native@0.76.x` | `react-native-reanimated@^3.16.x`, `react-native-builder-bob@^0.30+`, `@react-native/babel-preset@0.76.x` | RN 0.76 is new-arch-default release; floor for this lib. `[VERIFY]` |
| `react-native@0.77.x` / `0.78.x` | `react-native-reanimated@^3.17+`, JSC moved to community fork `[VERIFY]` | Test peer compat; do not pin upper bound. |
| `react-native-builder-bob@0.30+` | Node 18+, supports `commonjs` + `module` + `typescript` targets in one config | Verify the version CRNL scaffolds — accept what it picks. |
| `react-native-reanimated@3.x` | Babel plugin `react-native-reanimated/plugin` — must be **last** in `babel.config.js` plugins list | Worklet transform fragility; document this in README's worklet-usage section. |
| `Jest@29` | `react-native@0.76+` jest-preset | Jest 30 status against RN preset uncertain at cutoff. `[VERIFY]` |
| `CocoaPods@1.15.2+` | RN 0.76+ | Match RN template's Gemfile pin. `[VERIFY]` |
| `NDK r26d` | AGP 8.6+, CMake 3.22.1, Gradle 8.10 | Stable combo for RN 0.76+. NDK r27 forward-looking but unverified. `[VERIFY]` |
| `Xcode 16.x` | iOS 15.1 deployment target, CocoaPods 1.15.2+ | Use macOS 15 runner in CI. `[VERIFY]` |
## Verification Checklist (before Phase 1 scaffolding)
# Core
# Tooling
# Inspect what CRNL actually scaffolds (canonical source of truth)
## Confidence Per Major Recommendation
| Recommendation | Confidence | Reason |
|---|---|---|
| Use `create-react-native-library` JSI/C++ template (not Turbo Module) | **HIGH** | Locked in PROJECT.md; CRNL has shipped this template for years; matches design |
| `react-native-builder-bob` is still the build orchestrator in 2026 | **HIGH** | No competing library has appeared; CRNL still scaffolds it |
| RN 0.76 as new-arch-only floor | **HIGH** | RN 0.76 is the new-arch-default release |
| Reanimated 3.16+ for worklet runtime install | **MEDIUM** | API surface is correct; exact minor version needs registry verification |
| iOS 15.1 deployment target | **HIGH** | Matches RN 0.76+ floor |
| NDK r26d, CMake 3.22.1, Gradle 8.10, AGP 8.6 | **MEDIUM** | Combo is stable; exact patches need verification |
| Biome over ESLint+Prettier | **HIGH** | Locked in PROJECT.md |
| `release-it` over `changesets` | **HIGH** | Single-package; CRNL default |
| Jest 29 (not 30) | **MEDIUM** | RN preset historically lags; verify before bumping |
| `mitata` over `tinybench` | **MEDIUM** | Both valid; mitata gives better stats |
| Skip Detox + gtest for v0.1 | **HIGH** | Right scope for the milestone surface area |
| **All specific version pins** | **LOW** | Network access denied; verify before scaffolding |
## Sources
- **PROJECT.md** — locked architectural decisions; HIGH confidence
- **React Native release notes (training data, Jan 2026 cutoff)** — RN 0.76 new-arch-default, iOS 15.1 floor, JSC deprecation; MEDIUM confidence
- **react-native-mmkv source (training data)** — JSI install pattern reference, Reanimated worklet runtime registration; MEDIUM confidence
- **react-native-quick-crypto source (training data)** — CMake + vendored C++ patterns, podspec patterns; MEDIUM confidence
- **Reanimated 3.x docs (training data)** — `runOnRuntime`, worklet runtime C++ API; MEDIUM confidence
- **callstack/react-native-builder-bob README (training data)** — current target list (`commonjs`, `module`, `typescript`, `codegen`); MEDIUM confidence
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
