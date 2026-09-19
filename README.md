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
npx skills add QTSurfer/strategy-skills --skill qtsurfer-qtscript-strategy
```

### Claude Code plugin

```bash
# 1. Add the QTSurfer marketplace
claude plugin marketplace add QTSurfer/strategy-skills

# 2. Install the skill you want
claude plugin install qtsurfer-java-strategy@qtsurfer-strategy-skills
claude plugin install qtsurfer-qtscript-strategy@qtsurfer-strategy-skills
```

## Two ways to write a strategy

| | Skill | What it is |
|---|---|---|
| **Java** | `qtsurfer-java-strategy` | A class extending a strategy base class. The established route, with the full engine API: `update()`, cross-instrument logic, custom indicators, helper types. |
| **QTScript** _(beta)_ | `qtsurfer-qtscript-strategy` | A compact language — `strategy`, `param`, `instruments`, `setup:`, windows — whose braced bodies are plain Java, compiled to one class. Suits a strategy that is a few indicators and window bodies. |

They are two views of the same engine API, so the Java skill is the reference for what goes inside a
QTScript body, and anything QTScript cannot express is written in Java.

## Available Skills

<details>
<summary><strong>qtsurfer-java-strategy</strong></summary>

Write, review, and debug QTSurfer Java trading strategies.

**Use when:**

- Writing a new strategy that extends `AbstractTickerStrategy` (or the `AbstractKlineStrategy` / `AbstractFundingRateStrategy` siblings)
- Setting up indicators with the fluent builder (`InstrumentGroupRTIndicator`)
- Implementing window listeners with `AbstractWindowListener`
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
- Classloader boundary constraint — why inner classes must extend `AbstractWindowListener`
- `getEngineVersion()` — reading the version of the engine a strategy is running on
- Advanced patterns from production strategies (noise filtering chain, EMA distance analysis, trailing exits, re-entry protection)
- 5 complete working examples

</details>

<details>
<summary><strong>qtsurfer-qtscript-strategy</strong> — beta</summary>

Write, review, and debug QTSurfer strategies in QTScript (`.qtscript`), where every braced body is
plain Java.

**Use when:**

- Writing a strategy as sections (`strategy`, `param`, `init`, `instruments`, `setup:`, windows)
  instead of a Java class
- Attaching window bodies to indicators inline, by name, or through named sections
- Declaring which instruments a strategy accepts with patterns or a regular expression
- Reading a QTScript compile or run-time error, which is reported on the line you wrote
- Deciding whether a strategy belongs in QTScript or in Java

**Covers:**

- The language: header and data source (ticker / kline / funding), `param`, `init`, `instruments`,
  `setup:`, the five window forms and their periods
- What is in scope inside a body — `actual`, `prev`, `store`, the value variables, `value("name")`,
  signal emission — and that the body itself is ordinary Java
- What QTScript deliberately does not do, and when to write Java instead
- Worked examples for ticker, kline and funding sources

> Beta: the language will gain syntax. The Java skill is the established route and the reference for
> the API every body is written against.

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

```
Write the same strategy in QTScript, accepting only USDT markets
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
- `qtsurfer-qtscript-strategy` — QTScript (`.qtscript`) strategies 🧪 _(beta — the language will gain syntax)_
- `qtsurfer-ts-strategy` — TypeScript strategies _(planned)_

## License

Apache-2.0
