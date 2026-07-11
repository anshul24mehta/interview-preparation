# React Native Lead — Interview Preparation Guide

A Lead-level React Native interview usually tests three layers at once:
1. **Deep technical fundamentals** (RN internals, performance, native bridging)
2. **Architecture & system design** (how you'd structure a large app/team's codebase)
3. **Engineering leadership** (code quality, mentoring, trade-off decisions, delivery)

This guide is organized in that order, with sample answers you can adapt to your own experience.

---

## Part 1: Core React Native Technical Questions

### 1. Explain the RN architecture: old "Bridge" vs the New Architecture (Fabric + TurboModules + JSI).

**Answer:**
In the old architecture, JavaScript and native code communicate through the **Bridge** — an asynchronous, batched, serialized (JSON) message queue. Every call from JS to native (and back) has to be serialized, queued, and deserialized, which adds latency and makes synchronous native calls impossible.

The **New Architecture** replaces this with:
- **JSI (JavaScript Interface):** a lightweight C++ layer that lets JS hold direct references to native C++ objects/host objects, enabling synchronous, direct calls without serialization.
- **TurboModules:** native modules built on JSI — they're loaded lazily (only when required) instead of all at app startup, and calls can be synchronous.
- **Fabric:** the new rendering system. It moves more of the layout/measurement work to a C++ core shared across platforms, enables synchronous layout effects, and better supports concurrent React features.

**Why it matters for a lead:** you should be able to discuss migration strategy — most production apps still have legacy native modules that need TurboModule interop, and migration is usually incremental, not a rewrite.

### 2. How does the JS thread interact with the UI (main) thread and native modules thread?

**Answer:** RN historically runs three threads: **JS thread** (runs your React/JS logic), **Native/UI thread** (renders native views, handles gestures), and a **Shadow/layout thread** (computes Yoga flexbox layout). Heavy JS work (large loops, JSON parsing, unoptimized re-renders) blocks the JS thread, which delays sending UI updates — this shows up as dropped frames/jank even though the UI thread itself isn't busy. Animations done via `Animated` with `useNativeDriver: true` bypass the JS thread for the frame-by-frame updates, which is why they stay smooth even under JS thread load.

### 3. How would you diagnose and fix a laggy list of 1000+ items?

**Answer (structure your response around a checklist):**
- Use `FlatList`/`FlashList` (from Shopify) instead of `ScrollView` + `.map()` — virtualization only renders visible items.
- Provide `keyExtractor`, and a stable, cheap `renderItem`.
- Use `getItemLayout` if row height is fixed/predictable, to skip expensive layout measurement.
- Memoize row components (`React.memo`) and avoid inline function/object props that break memoization.
- Tune `windowSize`, `maxToRenderPerBatch`, `initialNumToRender`, `removeClippedSubviews`.
- Avoid heavy computation or context subscriptions inside each row.
- Consider `FlashList` for large lists — it recycles views like native platforms do, giving much better memory/perf characteristics than `FlatList`.
- Profile with Flipper/React DevTools Profiler to confirm the bottleneck before optimizing blindly.

### 4. How does React Native handle styling, and how is it different from CSS?

**Answer:** RN uses a JS object-based styling system (`StyleSheet.create`) that maps to a subset of Flexbox and platform-native styling — there's no cascade, no CSS selectors, and units are unitless (density-independent pixels). Flexbox defaults differ too: `flexDirection: 'column'` is the default in RN, vs `row` on web.

### 5. Explain `useMemo`, `useCallback`, and when memoization can hurt rather than help.

**Answer:** Both cache a value/function between renders based on a dependency array, avoiding unnecessary recomputation or child re-renders. But memoization has a cost (the dependency comparison + cache storage), so wrapping trivial computations or functions that aren't passed to memoized children just adds overhead without benefit. Rule of thumb: memoize expensive computations, or callbacks/values passed to `React.memo`-wrapped children or to `useEffect` dependency arrays where referential stability matters.

### 6. How would you bridge a native SDK (e.g., a native SDK with no RN wrapper) into your app?

**Answer:** Walk through the actual steps:
1. Write a native module (Objective-C/Swift for iOS, Kotlin/Java for Android) that wraps the SDK's API.
2. Expose methods via `@ReactMethod` (Android) / `RCT_EXPORT_METHOD` (iOS), or as a TurboModule spec (`.ts` spec file + codegen) on the New Architecture.
3. Handle async results via Promises or callbacks; emit events back to JS via `DeviceEventEmitter`/`NativeEventEmitter` for SDK callbacks (e.g., push notification received).
4. Write a thin JS wrapper (or a small package) so the rest of the app never touches `NativeModules` directly — this isolates native API changes.
5. Test on both platforms separately since native behavior (permissions, threading, lifecycle) diverges.

### 7. State management: how do you choose between Redux, Zustand, Context, Jotai, MobX, Recoil?

**Answer:** Frame it as trade-offs, not "best":
- **Context API:** fine for low-frequency, small-scope state (theme, auth user) — re-renders every consumer on every change, so it's a poor fit for frequently updated or large shared state.
- **Redux (with RTK):** best when you need strict predictability, time-travel debugging, middleware (sagas/thunks for complex async flows), and a large team that benefits from convention/structure.
- **Zustand/Jotai:** minimal boilerplate, good performance (selective subscriptions), good for small-to-mid apps or teams that want less ceremony.
- **MobX:** reactive/observable model, good fit if the team prefers OOP-style state.
- As a lead, the real answer interviewers want: you pick based on team size, existing conventions, debugging/observability needs, and whether server state (use React Query/RTK Query) is being conflated with client state — those should usually be separated.

### 8. Explain how you'd handle offline support and data sync.

**Answer:** Discuss: local persistence (SQLite via `react-native-sqlite-storage`/WatermelonDB/Realm, or MMKV for key-value), a sync queue for mutations made offline, conflict resolution strategy (last-write-wins vs operational transforms vs server-side merge), and using `NetInfo` to detect connectivity changes and trigger re-sync. For a lead-level answer, mention **WatermelonDB** or **Realm** for apps with complex relational offline data, since they're built for this exact use case with lazy loading and reactive queries.

### 9. What causes memory leaks in RN apps, and how do you find them?

**Answer:** Common causes: uncancelled subscriptions/listeners (`NativeEventEmitter`, timers, WebSocket callbacks) referencing unmounted components; large images not resized/cached properly; closures capturing large objects; retained navigation stack references. Tools: Xcode Instruments (Leaks/Allocations) for iOS, Android Studio Profiler for Android, and `why-did-you-render` or the React DevTools Profiler for excessive re-renders (which isn't a leak per se but a performance symptom with a similar root cause — stale references).

### 10. Explain how Hermes affects app performance and what trade-offs it brings.

**Answer:** Hermes is a JS engine optimized for RN: it precompiles JS to bytecode at build time (faster startup, since there's no JIT parse-and-compile step on device), has a smaller memory footprint, and improves cold-start time significantly on lower-end Android devices. Trade-off: some newer JS engine features may lag behind V8/JSC, and certain third-party libraries relying on JSC-specific behaviors occasionally need patching. As of recent RN versions, Hermes is the default engine on both platforms.

### 11. How do you handle deep linking and universal links across iOS/Android?

**Answer:** Discuss URL scheme registration, `Linking` API for handling incoming/outgoing links, Android App Links (`intent-filter` + `assetlinks.json` hosted on your domain) and iOS Universal Links (`apple-app-site-association` file), plus using React Navigation's linking config to map URLs to screens declaratively.

### 12. Testing strategy for a React Native app — what layers do you test and with what tools?

**Answer:**
- **Unit tests:** Jest for pure logic, reducers, utility functions.
- **Component tests:** React Native Testing Library — test behavior, not implementation details.
- **Integration tests:** testing navigation flows, API mocking (MSW or `nock`).
- **E2E tests:** Detox or Maestro for real device/simulator flows (login, checkout, etc.).
- **Native code tests:** XCTest (iOS) / JUnit (Android) if you have custom native modules.
- A lead should also speak to CI integration (running Detox on device farms, flakiness management) and enforcing coverage thresholds without turning testing into a box-ticking exercise.

---

## Part 2: Architecture & System Design Questions

### 13. How would you structure a monorepo for a large RN app with a web counterpart?

**Answer:** Discuss using **Nx** or **Turborepo** with **Yarn/PNPM workspaces**, sharing business logic, API clients, types, and validation schemas in shared packages, while keeping platform-specific UI separate. Mention **React Native Web** as an option to share more UI code, with its limitations (not every native module has a web equivalent, and performance characteristics differ).

### 14. How would you design the app's navigation and module boundaries for a team of 15+ engineers working on the same codebase?

**Answer:** Talk about a feature-based (not type-based) folder structure — each feature owns its screens, components, hooks, and API calls, exposing a clean public interface. Use React Navigation with nested navigators owned by each feature team. Establish clear contracts (TypeScript interfaces) at feature boundaries so teams can work in parallel without merge conflicts on shared files. Enforce this with lint rules (e.g., `eslint-plugin-boundaries`) preventing cross-feature deep imports.

### 15. How do you approach a CI/CD pipeline for RN (Fastlane, EAS, App Center, Bitrise)?

**Answer:** Cover: automated versioning/build numbering, Fastlane or EAS Build for building/signing both platforms, automated testing gates before merge, code signing certificate management (match for Fastlane, or EAS credentials), staged rollout (phased release on Play Store, TestFlight beta groups), and OTA updates via CodePush or EAS Update for JS-only fixes — while being clear about what OTA can and can't ship (no native code changes, and app store policies around OTA content).

### 16. How would you plan a migration from an older RN version (e.g., 0.6x) to the latest, in a large production app?

**Answer:** This is a classic lead-level question. Structure your answer around risk management:
- Audit third-party dependencies for New Architecture / version compatibility first.
- Use `react-native-upgrade-helper` diffs to understand native project changes.
- Migrate in a branch, run the full regression/E2E suite, and do it incrementally (patch versions first, not straight to latest major).
- Roll out to internal/beta users before general release.
- Have a rollback plan (feature flag or staged store rollout) in case of platform-specific regressions.

### 17. How would you decide between React Native and a fully native (Swift/Kotlin) rewrite for a specific high-stakes feature (e.g., video calling, camera-heavy features)?

**Answer:** This tests judgment, not RN advocacy. Good answers weigh: how performance/latency-critical the feature is, whether native SDKs already exist and are hard to bridge cleanly, team skill composition, and long-term maintenance cost of maintaining two codepaths. A strong lead answer: "I'd prototype the RN bridge to the native SDK first to measure actual overhead before assuming it's not viable — many perceived RN limitations are really bridging/architecture issues, not the framework itself."

### 18. How do you manage app size and startup time at scale?

**Answer:** Cover: Hermes bytecode precompilation, enabling ProGuard/R8 (Android) and bitcode/App thinning (iOS), lazy-loading screens/modules, image optimization (WebP, proper resizing, CDN-based responsive images), tree-shaking unused code, and analyzing bundle composition (e.g., `react-native-bundle-visualizer`) to catch bloat from unused libraries.

---

## Part 3: Leadership & Behavioral Questions (Lead-specific)

### 19. "Tell me about a time you had to make an architectural decision your team disagreed with."

**What they're testing:** decision-making under disagreement, and whether you build consensus vs. dictate.

**Framework for your answer (STAR):**
- **Situation:** the specific technical fork in the road.
- **Task:** what was at stake (deadline, scalability, tech debt).
- **Action:** how you gathered input, what data/prototype you used to de-risk the decision, how you communicated the trade-offs to the team.
- **Result:** the outcome, and — importantly — what you'd do differently if it didn't go perfectly. Interviewers value self-awareness over a "perfect win" story.

### 20. "How do you review code and set standards across a team with mixed experience levels?"

**Answer approach:** Discuss establishing a lightweight style guide/lint + Prettier config as a non-negotiable baseline (so review time isn't spent on formatting), a PR template that requires reasoning/screenshots/test coverage notes, and calibrating review depth to risk (a config change gets a light review; a data-layer change gets deep scrutiny). Mention mentoring junior engineers through review comments that explain "why," not just "change this."

### 21. "How do you balance technical debt against feature delivery pressure?"

**Answer approach:** A strong lead doesn't say "I always prioritize quality" (unrealistic) or "I always ship features" (no engineering credibility). Instead: describe a system — e.g., a fixed % of sprint capacity reserved for debt/tooling, a tech debt registry that's visible to product stakeholders with impact estimates (this bug class costs us X hours/month), and picking debt paydown that unblocks near-term velocity first.

### 22. "How do you mentor/upskill engineers new to React Native (e.g., from web-only React backgrounds)?"

**Answer approach:** Talk about pairing on the platform-specific gotchas (native module bridging, platform-specific UI quirks, performance/threading model), a structured onboarding doc/checklist, starter tickets that touch native code intentionally, and code review as a teaching tool rather than just gatekeeping.

### 23. "A critical release has a crash reported by 2% of users in production. Walk me through how you'd lead the response."

**Answer approach:** Structure your answer around incident response:
1. Triage severity/impact via crash reporting tool (Sentry/Firebase Crashlytics) — device/OS/version correlation.
2. Decide: hotfix via OTA update (if JS-only) vs. halt rollout (if using staged rollout) vs. emergency native build.
3. Communicate status to stakeholders with a timeline, not just "we're looking into it."
4. Post-incident: root cause analysis, and process fix (e.g., missing test coverage, insufficient staged rollout percentage) so it doesn't recur.

### 24. "How do you estimate and plan a multi-quarter roadmap when the team is also expected to firefight production issues?"

**Answer approach:** Discuss capacity planning with a buffer for unplanned work (often 15–25% of sprint capacity), tracking interrupt-driven work explicitly so it's visible in planning (not silently absorbed), and negotiating scope/timeline transparently with product rather than over-promising.

### 25. "How do you evaluate whether to adopt a new library/tool (e.g., switching from FlatList to FlashList, or adopting Expo)?"

**Answer approach:** Describe a lightweight evaluation framework: bus factor/maintenance activity of the library, migration cost vs. benefit (prototype on one screen first), team learning curve, and exit cost if it doesn't work out. This shows pragmatism over chasing trends.

---

## Part 4: Rapid-Fire Concept Checks (good for warm-up or screening rounds)

| Question | One-line answer |
|---|---|
| What is Yoga? | The cross-platform layout engine RN uses to implement Flexbox in C++. |
| Difference between `Pressable` and `TouchableOpacity`? | `Pressable` is the newer, more flexible API supporting more interaction states (hover, press duration); `TouchableOpacity` is legacy but still widely used. |
| What does `useNativeDriver: true` do? | Offloads animation frame updates to the native UI thread, so they run smoothly even if the JS thread is busy. |
| What's the difference between controlled and uncontrolled components in RN forms? | Controlled: value driven by React state on every keystroke; uncontrolled: native input manages its own state, read via ref — better for performance-sensitive large forms. |
| What is codegen in the New Architecture? | A build-time step that generates strongly-typed native interface code from TypeScript/Flow specs for TurboModules and Fabric components. |
| CodePush vs EAS Update? | Both deliver JS-only OTA updates; EAS Update is Expo's actively maintained solution, CodePush (App Center) has had uncertain long-term support. |
| What's `react-native-reanimated` for? | Running animations and gesture logic on the UI thread (via worklets) instead of the JS thread, for consistently smooth 60fps animations. |
| MMKV vs AsyncStorage? | MMKV is a much faster, synchronous, native key-value store; AsyncStorage is async and notably slower at scale. |

---

## How to Use This Guide

- **Don't memorize answers verbatim** — interviewers at the lead level are testing judgment and trade-off reasoning, not textbook recall. Practice explaining *why* you'd choose one approach, with a real example from your experience.
- **Prepare 3–4 STAR stories** covering: a technical disagreement, a production incident, a mentoring win, and a roadmap/prioritization trade-off. Most behavioral questions map back to one of these.
- **Have a system design answer ready** for "design a large-scale RN app" — practice sketching module boundaries, state management choice, and CI/CD out loud, since leads are often asked to whiteboard this.
