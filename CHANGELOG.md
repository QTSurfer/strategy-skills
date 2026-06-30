# Changelog

All notable changes to the **QTSurfer Java Strategy** skill are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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
