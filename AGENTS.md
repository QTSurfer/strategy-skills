# QTSurfer Strategy Skill

Author **QTSurfer** trading strategies with your AI agent.

- Skill repo: <https://github.com/QTSurfer/strategy-skills>
- MCP server: <https://github.com/QTSurfer/mcp-java>

## Where the source of truth lives

This file is the index. **The actual source of truth for writing strategies — the engine API,
signals, output format, imports and worked examples — is the skill:**

```
skills/qtsurfer-java-strategy/
  SKILL.md                 <- main authoring guide (classes, indicators, signals, imports)
  references/examples.md   <- copy-paste strategies with exact imports
  references/patterns.md   <- pattern catalogue
```

Go there first. If you only read one file, read `skills/qtsurfer-java-strategy/SKILL.md`.

> Raw links (for agents that fetch URLs):
> - SKILL.md: `https://raw.githubusercontent.com/QTSurfer/strategy-skills/main/skills/qtsurfer-java-strategy/SKILL.md`
> - examples: `https://raw.githubusercontent.com/QTSurfer/strategy-skills/main/skills/qtsurfer-java-strategy/references/examples.md`

## Canonical imports (must match — other packages do not exist)

Use **exactly** these, or the strategy will fail to compile server-side:

```java
import com.wualabs.qtsurfer.engine.strategy.StrategyProperty;   // @StrategyProperty annotation
import com.wualabs.qtsurfer.engine.strategy.AbstractTickerStrategy;   // base class
import com.wualabs.qtsurfer.engine.core.instrument.Instrument;       // Instrument type
import com.wualabs.qtsurfer.engine.indicators.helpers.group.InstrumentGroupRTIndicator;
import com.wualabs.qtsurfer.engine.indicators.helpers.group.InstrumentMapRTIndicator;
import com.wualabs.qtsurfer.engine.core.state.StateStore;
import com.wualabs.qtsurfer.engine.strategy.event.signal.InfoStrategySignal;
```

> ⚠️ `@StrategyProperty` is in `com.wualabs.qtsurfer.engine.strategy.StrategyProperty`.
> `com.wualabs.qtsurfer.engine.annotations.StrategyProperty` and
> `...engine.strategy.annotation.StrategyProperty` do **not** exist — they were historical
> variants and fail to compile. Copy the import from the skill above.

## Install the skill

Install the MCP server, authenticate and register it with your client:

<https://raw.githubusercontent.com/QTSurfer/strategy-skills/main/README.md>

Once connected, your agent has the QTSurfer tools and the guidance to write, compile and
backtest strategies.