# QTSurfer Strategy Skills

AI agent skills for writing [QTSurfer](https://qtsurfer.com) trading strategies.
Compatible with Claude Code, OpenAI Codex, Cursor, Cline, and any agent that supports
the [Agent Skills](https://agentskills.io/) format.

## Installation

### Install all skills

```bash
npx skills add QTSurfer/strategy-skills
```

### Install a specific skill

```bash
npx skills add QTSurfer/strategy-skills --skill qtsurfer-java-strategy
```

### Claude Code plugin

```bash
# 1. Add the QTSurfer marketplace
claude plugin marketplace add QTSurfer/strategy-skills

# 2. Install the skill you want
claude plugin install qtsurfer-java-strategy@qtsurfer-strategy-skills
```

## Available Skills

<details>
<summary><strong>qtsurfer-java-strategy</strong></summary>

Write, review, and debug QTSurfer Java trading strategies.

**Use when:**

- Writing a new strategy that extends `AbstractTickerStrategy` (or the `AbstractKlineStrategy` / `AbstractFundingRateStrategy` siblings)
- Setting up indicators with the fluent builder (`InstrumentGroupRTIndicator`)
- Implementing window listeners with `AbstractOnChangeListener`
- Managing per-instrument state with `StateStore`
- Using crossover detection, trailing exits, or noise filtering patterns
- Submitting a strategy for backtesting via the MCP server or SDK
- Debugging a strategy that compiles but doesn't behave as expected

**Covers:**

- Strategy base classes — `AbstractTickerStrategy` (ticker), `AbstractKlineStrategy` (kline), `AbstractFundingRateStrategy` (funding-rate), plus the engine-only `AbstractMultiSourceStrategy` (combined sources)
- Full indicator catalogue (EMA, SMA, RSI, Bollinger, distance, gain, predicates, …)
- Window time patterns (`WindowTime.s1` through `m5`, custom `Duration`)
- `StateStore` API (counters, accumulators, boolean flags, arbitrary state)
- `@StrategyProperty` for runtime-configurable parameters
- Classloader boundary constraint — why inner classes must extend `AbstractOnChangeListener`
- Advanced patterns from production strategies (noise filtering chain, EMA distance analysis, trailing exits, re-entry protection)
- 5 complete working examples

</details>

## Usage

Skills are automatically available once installed. The agent uses them when
relevant tasks are detected.

**Examples:**

```
Write a strategy that buys when RSI drops below 30 and sells above 70
```

```
Add a trailing stop-loss to this strategy using the peak-price pattern
```

```
Set up a noise filtering chain on the EMA distance signal
```

```
Submit this strategy to backtest on binance BTC/USDT from 2026-05-01 to 2026-05-10
```

## Skill structure

Each skill follows the [Agent Skills Open Standard](https://agentskills.io/):

- `SKILL.md` — skill manifest with frontmatter (name, description, `metadata.version`) and instructions
- `references/` — supporting reference files (indicator catalogue, examples, patterns)

Versions follow [Semantic Versioning](https://semver.org/) and are tracked in
[`CHANGELOG.md`](./CHANGELOG.md). See [CONTRIBUTING.md](./CONTRIBUTING.md) to
propose changes.

## Roadmap

- `qtsurfer-java-strategy` — Java strategies: `AbstractTickerStrategy`, `AbstractKlineStrategy`, `AbstractFundingRateStrategy` ✅ (the `AbstractMultiSourceStrategy` multi-source base compiles in-engine but is not yet runnable via public `submit_backtest`)
- `qtsurfer-ts-strategy` — TypeScript strategies _(planned)_

## License

Apache-2.0
