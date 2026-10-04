# Changelog

All notable changes to the skills in this repository are documented here. Entries name the skill they
belong to; versions are per skill (see `.claude-plugin/marketplace.json`).

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Fixed 🩹

- **`qtsurfer-java-strategy`: a command's `properties` land as top-level entries on `CommandRequest`,
  not nested under their own `properties` key.** The `getStateStore(...)` example read
  `Map<String, Object> properties = request.get("properties"); properties.get("instrument")` — a real
  design error, not wording: the runner puts every property directly on the request's own map, so the
  correct read is `request.get("instrument")`. Corrected, along with a note that a value keeps its
  JSON type (so assigning a non-string one to a `String` throws `ClassCastException`). The command's
  own text is kept as its own field, separate from `properties` entirely, so no property name is off
  limits — `cmd` included. `qtsurfer-qtscript-strategy` needed no change: its `$command.<key>` section
  was already written abstractly enough to stay correct.

### Added ✨

- **`qtsurfer-java-strategy`: receiving commands.** A live run's owner can now tell it a command from outside
  (`POST /live/{runId}/commands`) without restarting it. A new "Receiving commands" section shows implementing
  `CommandRequestHandler`, when `handle` runs relative to `update()`, and says a command is always a plain string
  and is transient — not replayed to a replica across a restart, unlike a `@StrategyProperty` value. A strategy that
  does not implement the interface answers every command with a `409`.
- **`qtsurfer-java-strategy`: an example for `getStateStore(...)` from inside a command.** A second code example
  in "Receiving commands" shows reading `request.get("properties")` for an instrument name and reaching that
  instrument's store with `getStateStore(String)` — the same example `docs/strategy_coding.md` (qtsurfer-api)
  already carries, so the two stay in sync.
- **`qtsurfer-qtscript-strategy`: `onCommand { }`, handling a command.** A new section, at most one per file, lets a
  QTScript strategy implement `CommandRequestHandler` the same way a Java strategy does: `$command` holds the
  command's text inside its body, nothing a window body has (`actual`, `$indicator`, `value(...)`, `store`) is in
  scope, and every `param` stays readable and settable. Added to the reserved-names list and the frontmatter
  description. A strategy with no `onCommand { }` answers every command with a `409`, same as Java.
- **`qtsurfer-qtscript-strategy`: `$command.<key>`, reading a command's own `properties`.** A command may carry a
  `properties` object alongside its text; `$command.<key>` reads a value from it as a `String` (`null` when
  absent), and only fires when `<key>` is not itself a call, so `$command.equals(...)` and the like still read as
  real `String` methods. `getStateStore("<symbol>")` is now reachable from `onCommand`, which has no instrument of
  its own the way a window body does — a command whose properties name one can still reach its store.
- **`qtsurfer-java-strategy`: a "Language level" section.** Strategy code is ordinary modern Java: lambdas, method
  references, `var`, records, switch expressions, pattern matching for `instanceof`, text blocks and `+` between
  strings compile and run. Checked on a running platform with a strategy that uses a lambda listener, `var` and a
  record through a backtest, and with one-method strategies for each of the others.
- **`qtsurfer-java-strategy`: a window listener can be a lambda.** `window(...)` takes the engine's
  `OnChangeListener`, a functional interface with the same `onChange(store, prev, actual)`, so a listener that only
  needs the `StateStore` is a lambda; `AbstractWindowListener` is for the helpers (`emitBuy(price)`,
  `getPrevInstant()`, …). A short example (consecutive minutes a long EMA rose) and what `prev` and `actual` are.
- **`qtsurfer-java-strategy`: keeping the stored signals when `update()` is overridden.** A strategy that replaces
  `super.update(ticker)` with `updateInstrument` plus `updateIndicators` trades normally, but a backtest run with
  `storeSignals` comes back with no signals (`signalCount: 0`) — no indicator series and no buy/sell markers. The
  section "Reading indicator values outside a listener" now says so and shows the form that keeps them: call
  `super.update(ticker)` and read the indicators with `getRTIndicator(instrument, name)`.
- **`qtsurfer-java-strategy`: `min`, `max` and `step` written as decimals.** They are `double` annotation elements;
  an integer literal (`min = 1`) registers without an error, drops the range from the declared properties, fails
  `validate` and makes a backtest of the strategy fail without a message. Documented in "Configurable properties"
  and "Common mistakes".

### Changed 🔄

- **`qtsurfer-java-strategy`: "Common mistakes" no longer says to prefer an inner class over a lambda for a listener.**
  Replaced by when each fits, and two new entries for the two traps above (`updateIndicators` instead of
  `super.update`, `min = 1`). The `README.md` line that said inner classes must extend `AbstractWindowListener` now
  describes both forms.
- **`qtsurfer-java-strategy`: "Receiving commands" now says QTScript can do it too.** A one-paragraph
  cross-reference: a QTScript strategy implements `CommandRequestHandler` through its own `onCommand { }`
  section (see `qtsurfer-qtscript-strategy`), recognized by the platform the same way a hand-written Java
  class is.
- **`qtsurfer-java-strategy`: "Receiving commands" documents `properties`, and corrects a durability claim.**
  A command may now carry a `properties` object (`request.get("properties")`, a `Map<String, Object>` or
  `null`). The skill previously said a value a command sets belongs "in a parameter it sets from inside
  `handle`" to survive a restart — wrong: assigning a `@StrategyProperty` field from inside `handle` only
  changes this replica's in-memory value, the same as a `StateStore` write, and neither is written to the
  run's stored parameter set. Only a real `PUT /live/{runId}/params` call, from outside the run, is durable.
- **`qtsurfer-qtscript-strategy`: the same durability correction.** "Handling a command" no longer implies a
  `StateStore` survives a restart; it does not (plain memory, same as a field) — only
  `PUT /live/{runId}/params` does.

- **`qtsurfer-java-strategy`: what goes in a signal's `data` is public on a public run, and is bounded.** A short paragraph in
  "Data / analytics signals" says that everything set on a signal is its `data` and is published with it, so whoever may read a run
  may read it (on a `public` run, anyone), and that a signal whose `data` is over 8 KiB is not pushed on the WebSocket channel
  but is still returned whole by the history route.
- **`qtsurfer-qtscript-strategy` 1.1.0: windows on a call that publishes several names, `$indicator`, and what a `200` means.**
  The skill said an inline window could not be written on `bollinger(20, 2)`; it can: the window attaches to the
  middle band and `actual` is its value. It now documents `$indicator`, the name of the indicator a window is attached to (how a
  body reaches the outer bands its line registered), and that a window on a name that is not registered is found at validation, not
  when registering. "When something is wrong" now says that a `200` from registering means the source parsed and compiled, not that it
  will run, so a strategy just written should be validated before it is run, and that the source is capped at 32 KiB (a larger body
  is refused with `413`, the cap named in the JSON error).
- **`qtsurfer-qtscript-strategy`: kline runs, and the bar width is yours.** A kline strategy can now
  be executed, swept and walk-forward validated; the skill no longer says candles are one-second. The
  width comes from the `cadence` the data is prepared at (`1s`, `1m`, `5m`, `15m`, `30m`, `1h`, `4h`
  or `1d`), so one file runs at several. It also says what does not run yet: a funding strategy
  registers and its data can be prepared, but a run or a sweep over funding data is rejected with a
  `400`. And it notes that whitespace and comments before `strategy` do not change how a file is
  recognised.
- **`qtsurfer-java-strategy`: two statements in "Strategy base classes" brought in line with what the
  platform runs.** `AbstractKlineStrategy` no longer "subscribes to candles for `getInterval()`": in a
  backtest the bar width is the `cadence` the data was prepared at (`1s`, `1m`, `5m`, `15m`, `30m`,
  `1h`, `4h`, `1d`), whatever `getInterval()` returns. And `AbstractFundingRateStrategy` is no longer
  marked runnable through `submit_backtest`: funding data can be prepared, but a run or a sweep over it
  is rejected with a `400` for now. Nothing was removed; the rest of the skill is untouched.

### Added ✨

- **New skill `qtsurfer-qtscript-strategy` 1.0.0 — QTScript (`.qtscript`), in beta.** A second way to
  write a strategy, alongside the Java one: a compact language of sections (`strategy`, `param`,
  `init`, `instruments`, `setup:`, five window forms) that compiles to a single Java class, where
  **every braced body is plain Java**. The skill documents the language, what is in scope inside a
  body (`actual`, `prev`, `store`, the per-source value variables, `value("name")`, signal emission),
  the declarative instrument filter and its regular-expression form, how compile and run-time errors
  are reported against the line the author wrote, and — explicitly — what QTScript does not do
  (`update()`, cross-instrument logic, custom indicators, helper types), each being a reason to write
  the strategy in Java instead. `references/examples.md` carries complete files for the ticker, kline
  and funding sources.
- **`qtsurfer-java-strategy`: a pointer to the new skill**, after the opening paragraph — one
  paragraph naming QTScript as the beta alternative and stating that the Java route remains the one
  with the full engine API. Nothing else in the skill changed: the Java guidance, references and
  examples are untouched.
- **Repository index entries for the second skill** — `README.md` (a "Two ways to write a strategy"
  table, the skill's own section, install lines, roadmap), `AGENTS.md` (the file map) and
  `.claude-plugin/marketplace.json` (a second plugin entry at 1.0.0).

### Removed 🗑️

- **"Language level" section in `SKILL.md`**, added in 1.4.0 — the skill no longer documents a reduced Java language subset, and the examples in `SKILL.md`, `references/indicators.md`, `references/patterns.md` and `references/examples.md` read as they did before that release: `var` and lambdas, not explicit types and anonymous inner classes. A skill for writing strategies should describe what the author writes rather than the shape of the compilation behind it, and a language-level claim in prose goes stale without anyone noticing. The two `Common mistakes` entries that existed only to point at that section went with it, and the third is back to its earlier wording — preferring an inner class for a listener is a preference about helper access, not a compilation constraint.

### Changed 🔄

- **"Allowed imports" trimmed to the list itself** — which packages a strategy may import, and the failure mode when it reaches outside them (execution time, bare "class could not be found"), both stay: an author needs those. The paragraph about how the list is enforced and why it is not configurable is gone; it describes the platform rather than the strategy.

Unrelated API facts introduced alongside the removed material stay: `emitBuy`/`emitSell` taking the instrument explicitly outside a window listener, the signal-emission overload table, and the `Instrument` package move.

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
