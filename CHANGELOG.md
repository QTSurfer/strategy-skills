# Changelog

All notable changes to the **QTSurfer Java Strategy** skill are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.4.0] — 2026-08-19

### Added ✨

- **"Allowed imports" section in `SKILL.md`** — the sandbox classloader enforces a package whitelist, and an import outside it fails at *execution* time with a bare "class could not be found" and no hint as to why, which is a long way from the import that caused it. The section names what is allowed (`com.wualabs.qtsurfer.engine.*`, `java.lang`, `java.util`, `java.math`, `java.time`, `java.text`) and what is blocked regardless of package (`System`, `Runtime`, `Thread`, `Executor`/`ExecutorService`, and `java.io` outright). Also documents that the list is compiled into the platform rather than configurable — a sandbox for untrusted code should not be extensible through a weaker channel than a reviewed change.
- **"Language level" section in `SKILL.md`** — strategy code compiles against a language base well behind the JDK the platform runs on, and targeting it like modern Java fails with an error that points at the syntax rather than the cause. `var` and lambdas/method references are called out specifically: lambdas do not compile in any position, so an anonymous inner class is the substitute.

### Changed 🔄

- **A `@StrategyProperty` field no longer needs a JavaBean setter** — the annotation and the field are the whole declaration, and the engine writes the field directly. The previous guidance (added after the platform threw `NoSuchMethodException: setFastPeriod` mid-sweep) is removed from `SKILL.md`, and `references/examples.md`'s configurable-EMA strategy drops the three hand-written pass-through setters it carried for that reason alone. Declare a setter only when the property needs validating, clamping, or something derived recomputed: when one exists every injection channel still goes through it, so the guard is never bypassed. The field must not be `static` — its value would be shared across sweep trials running in parallel — or `final`, which nothing can assign after construction; either needs a setter, and a property with neither is reported as a notice rather than silently skipped.
- **The default is declared once, on `defaultValue`** — examples no longer repeat it as a field initializer. An initializer runs *after* the annotation's default has been applied and overwrites it, so if the two disagree the strategy runs on the initializer while the platform records the annotation's value against the results. Writing it in one place removes the question.

### Fixed 🐛

- **`Instrument` moved packages and the docs sample never followed** — the override is `acceptInstrument(Instrument)` from `com.wualabs.qtsurfer.engine.core.instrument`, not `acceptCurrencyPair(CurrencyPair)` from `org.knowm.xchange.*`; the XChange import would not even load under the sandbox whitelist. `ExecutionMode`'s third value is `LONG_MULTI`, not `LONG_SHORT`. Also notes that the *default* `acceptInstrument` is not unconditional — it gates on the strategy's output currency, so accepting everything means overriding it with `return true`.
- **Examples used language features the compiler rejects** — `var` and lambdas removed from `SKILL.md` and all three `references/` files, in favour of explicit types and anonymous inner classes.

### Note

Versions 1.2.0 and 1.3.0 bumped `SKILL.md` but not `.claude-plugin/marketplace.json`, which stayed at 1.2.0; this release brings all three (manifest, skill frontmatter, changelog) back into the lockstep `CONTRIBUTING.md` requires.

## [1.3.0] — 2026-07-26

### Added ✨

- **"Engine version" section in `SKILL.md`** — `getEngineVersion()`, `getEngineVersionMajor()` and `getEngineVersionMinor()` report the version of the engine jar actually loaded (read from that jar's own metadata, not a constant baked in at compile time). They need no import: all three are available on the strategy base classes and inside an `AbstractWindowListener`, which is where most strategy logic lives. The patch component stays on the engine class, `EngineVersion.getPatch()` — the class lives in `com.wualabs.qtsurfer.engine`, which the strategy sandbox allows wholesale, so importing it works. Nothing here throws: an unresolvable version degrades to `EngineVersion.UNKNOWN` (`"unknown"`) and `EngineVersion.UNKNOWN_COMPONENT` (`-1`), the latter negative on purpose so a version gate fails closed rather than matching by accident; the components resolve all-or-nothing, never a half-parsed mix. Documented with the recommendation to emit or log the version from strategies that are stored and re-run later: engine APIs do change between versions and recording the engine a run happened on is what makes that class of break diagnosable rather than mysterious.
- **The engine-version accessors listed in the `AbstractWindowListener` helper list** — next to `emitBuy`/`emitSell`, the window instants, `this.instrument` and `this.indicators`.

## [1.2.0] — 2026-07-25

### Changed 🔄

- **`AbstractOnChangeListener` renamed to `AbstractWindowListener`** — the class became window-specific once instant sugar moved onto it (see Added below), and the old name collided with `java.awt.event.WindowListener` in auto-import-heavy AI-generated code. `SKILL.md`, `references/examples.md`, `references/patterns.md`, and `README.md` all updated.
- **`onChange`'s store parameter is now `StateStore`, not `StateStoreSupport`** — `StateStoreSupport` is gone; the store is handed to `onChange` already resolved. The `initStore(storeSupport)` ritual and the `this.store` field are both gone — use the `store` parameter directly.
- **"Shared StateStore between windows" pattern simplified** — all windows built on the same `InstrumentGroupRTIndicator` already share one instrument-level store by default; the old example's `getStateStore(instrument)` / `globalStateStore()` dance documented a call that doesn't exist. Renamed to "Shared state between windows" with a two-line example.
- **`detectCrossAbove`/`detectCrossBelow` replaced by `CrossDetector`** — the two methods are gone from the base classes entirely; crossover tracking is now a standalone `CrossDetector` you construct as a field (`new CrossDetector()`) and query with one `check(left, right)` call returning `Cross(above, below)`. See Fixed below for why the old shape had to go, not just get renamed.

### Added ✨

- **`getPrevInstant()` / `getCurrInstant()` on `AbstractWindowListener`** — when a listener is registered on a window (`.window(...)`), it now reports that window's own boundaries (when it opened / when it closed) directly, without going through the state store. Fixes the addressability gap in the old pattern: `window("rsi14", WindowTime.m1, new SignalListener(...))` builds an auto-named window, so there was never a way to reach the instant keys the old approach wrote into the store under a window-name prefix.
- **"Indicator metadata" section in `references/indicators.md`** — documents `RTIndicator#getMeta()` / `getId()` / `getDisplayHint()` / `isHidden()` and the `AbstractRTIndicator#withMeta(...)` / `withDisplayHint(...)` builder setters for custom indicators, with the write-only-descriptor constraint spelled out. Gives an agent a way to introspect what an indicator carries (e.g. "is this value already a percentage?") without parsing its name.
- **`getStateStore(instrument)` usage example in `update()`** — the "State management" section described the accessor in prose but never showed it called; added a worked `update(Ticker)` snippet plus a note that it always resolves the store immediately (unlike a window listener's lazily-resolved one) and is safe to `.orElseThrow()` on the documented base classes.

### Fixed 🐛

- **Crossover detection helper silently broke its second direction** — `SKILL.md`'s "Crossover detection helper" and `references/examples.md` example 5 both called `detectCrossAbove(cross, 0, ...)` then `detectCrossBelow(cross, 0, ...)` on the *same* array slot, in the same tick. Both methods unconditionally overwrote that slot with the current tick's value on every call, so the first call always clobbered the value the second one needed — meaning whichever direction was checked second could never fire, on any strategy copied from these examples. Root-caused and fixed in the engine by replacing the shape entirely with `CrossDetector`, which computes both directions from one read-then-write; docs updated to match.
- **Internal planning reference in `references/patterns.md`** — the "Names carry no `%`" note cited an internal engine planning document, which does not exist outside the private engine repo and is meaningless to a reader of this public skill. Removed; the explanation stands on its own without it.

## [1.1.1] — 2026-07-24

### Fixed 🐛

- **Stale indicator package references (engine package refactor)** — `references/indicators.md` still described the pro tier as one flat `com.wualabs.qtsurfer.engine.indicators.pro` package; the engine now categorizes it into `<category>.pro` sub-packages (`averages.pro`, `trend.pro`, `momentum.pro`, `volatility.pro`, `volume.pro`, plus the pre-existing `statistics.pro`), mirroring the free tier's own category packages (`averages`, `momentum`, `distance`, `bollinger`, `statistics`). The advanced-catalogue table now carries a **Tier** column (Free / Pro / Pro only) so free-vs-paid is explicit instead of silently mixed within a category row.
- **Reintroduced `%`-suffixed indicator names** — `references/patterns.md` had several examples (`"distemas%"`, `"chgDistemas%"`, `"smoothDistemas%"`, `"vlts%"`, …) smuggling percent-display metadata into the indicator name string, the exact convention the engine eliminated (percent-ness now lives in the indicator's `DisplayHint` metadata, set automatically by `distance()`/`percentChange()`). Names are now clean; a note explains why.
- **`getExecutionMode` example comment listed a non-existent enum value** — `SKILL.md`'s minimal template said `// LONG, SHORT, or LONG_SHORT`; the engine's `ExecutionMode` enum value is `LONG_MULTI`, not `LONG_SHORT`.

## [1.1.0] — 2026-06-30

### Added ✨

- **Strategy base-class family documented** — beyond `AbstractTickerStrategy`, the skill now covers the sibling base classes confirmed in the engine: `AbstractKlineStrategy` (`Kline` source) and `AbstractFundingRateStrategy` (`FundingRate` source) — both backtestable via `submit_backtest` — plus `AbstractMultiSourceStrategy` (combined sources). All single-source bases extend `AbstractSubscriptionStrategy<T>` and share the same indicator / window / signal model. New "Strategy base classes" section with a source → handler → availability table.

### Fixed 🐛

- **Corrected the multi-source claim** — the previous *"`AbstractMultiStrategy` (coming soon) — do not attempt"* note was inaccurate. The engine class is `AbstractMultiSourceStrategy`; it already compiles and registers (declaring `getRequiredSources()` and dispatching to `onTicker` / `onKline` / `onFundingRate`), but is **not yet runnable via the public `submit_backtest`** — now documented as such instead of "coming soon".

### Changed 🔄

- **Skill restructured for tighter context** — cross-instrument (market-wide) strategies moved out of `SKILL.md` into `references/patterns.md` (where the 1.0.0 changelog already documented them); the model-facing `description` was trimmed to triggers and broadened to the strategy family.
- **De-duplicated guidance** — the `Ticker` accessor-vs-getter rule and the `_`-prefix hidden-indicator convention each now live in a single source instead of being restated across sections.

### Removed 🗑️

- **"Building a Ticker in tests" section** — strategies compile and run server-side via remote submission; there is no public Java engine/SDK to construct a `Ticker` or run JUnit against locally, so the section described a workflow that isn't available.

## [1.0.1] — 2026-06-18

### Fixed 🐛

- **Wrong import in `@StrategyProperty` example** — `references/examples.md` imported `com.wualabs.qtsurfer.engine.strategy.annotation.StrategyProperty`, but the annotation lives in `com.wualabs.qtsurfer.engine.strategy.StrategyProperty`. Strategies copied from the example failed to compile. The import now matches the rest of the skill (`SKILL.md`, `references/patterns.md`).

## [1.0.0] — 2026-05-18

### Added ✨

- **Initial release of the `qtsurfer-java-strategy` skill.** Covers writing, reviewing, and debugging QTSurfer Java trading strategies:
  - `AbstractTickerStrategy` lifecycle, the indicator builder API, window listeners, state management, and signal emission.
  - Submission via the MCP server or the SDK.
  - Reference material: indicator catalogue (`references/indicators.md`), advanced patterns including custom `RTIndicator`s and cross-instrument strategies (`references/patterns.md`), and worked examples (`references/examples.md`).
  - Distributed both as a Claude Code plugin (`.claude-plugin/marketplace.json`) and via `npx skills add QTSurfer/strategy-skills`.
