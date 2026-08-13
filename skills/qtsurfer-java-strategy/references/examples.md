# Strategy Examples

## 1. EMA Crossover (update loop)

Buy when fast EMA crosses above slow EMA; sell on cross below.

```java
import com.wualabs.qtsurfer.engine.core.instrument.Instrument;
import com.wualabs.qtsurfer.engine.core.Ticker;
import com.wualabs.qtsurfer.engine.indicators.helpers.group.InstrumentGroupRTIndicator;
import com.wualabs.qtsurfer.engine.strategy.AbstractTickerStrategy;

public class EmaCrossoverStrategy extends AbstractTickerStrategy {

    private Boolean fastAboveSlow;

    @Override
    protected void setupIndicators(InstrumentGroupRTIndicator indicators) {
        indicators.addPrice().ema("fast", 9).ema("slow", 21);
    }

    @Override
    public void update(Ticker ticker) {
        Instrument inst = ticker.instrument();
        updateInstrument(inst, ticker.timestamp());
        var ind = updateIndicators(inst, ticker);

        if (!ind.getExisting("slow").isReady()) return;

        boolean currentFastAbove = ind.getValue("fast") > ind.getValue("slow");
        if (fastAboveSlow == null) { fastAboveSlow = currentFastAbove; return; }

        if (currentFastAbove && !fastAboveSlow)  emitBuy(ticker.last());
        if (!currentFastAbove && fastAboveSlow)  emitSell(ticker.last());
        fastAboveSlow = currentFastAbove;
    }
}
```

## 2. RSI Oversold/Overbought (window listener)

Enter long when RSI drops below 30; exit when it rises above 70.
Fires once per 1-second window, not on every tick.

```java
import com.wualabs.qtsurfer.engine.indicators.helpers.group.InstrumentGroupRTIndicator;
import com.wualabs.qtsurfer.engine.indicators.helpers.WindowTimeRTIndicator.WindowTime;
import com.wualabs.qtsurfer.engine.strategy.AbstractWindowListener;
import com.wualabs.qtsurfer.engine.strategy.AbstractTickerStrategy;
import com.wualabs.qtsurfer.engine.core.state.StateStore;

public class RsiStrategy extends AbstractTickerStrategy {

    @Override
    protected void setupIndicators(InstrumentGroupRTIndicator indicators) {
        indicators
            .addPrice()
            .rsi(14)
            .window("rsi14", WindowTime.s1, new RsiListener(this, indicators));
    }

    private class RsiListener extends AbstractWindowListener {

        RsiListener(AbstractTickerStrategy s, InstrumentGroupRTIndicator ind) {
            super(s, ind);
        }

        @Override
        public void onChange(StateStore store, double prev, double actual) {
            if (actual < 30 && !store.is("inPosition")) {
                emitBuy(indicators.getValue("price"));
                store.set("inPosition");
            }
            if (actual > 70 && store.is("inPosition")) {
                emitSell(indicators.getValue("price"));
                store.unset("inPosition");
            }
        }
    }
}
```

## 3. Forced Trade (periodic buy/sell)

Simple stress-test strategy: buy at tick 60, sell at tick 120, repeat.
Used in CI integration tests — known to compile and run correctly.

```java
import com.wualabs.qtsurfer.engine.indicators.helpers.group.InstrumentGroupRTIndicator;
import com.wualabs.qtsurfer.engine.indicators.helpers.WindowTimeRTIndicator.WindowTime;
import com.wualabs.qtsurfer.engine.strategy.AbstractWindowListener;
import com.wualabs.qtsurfer.engine.strategy.AbstractTickerStrategy;
import com.wualabs.qtsurfer.engine.core.state.StateStore;

public class ForcedTradeStrategy extends AbstractTickerStrategy {

    @Override
    protected void setupIndicators(InstrumentGroupRTIndicator indicators) {
        indicators.addPrice().window("price", WindowTime.s1,
            new TradeListener(this, indicators));
    }

    private class TradeListener extends AbstractWindowListener {

        TradeListener(AbstractTickerStrategy s, InstrumentGroupRTIndicator ind) {
            super(s, ind);
        }

        @Override
        public void onChange(StateStore store, double prev, double actual) {
            long count = store.inc("count");
            if (count % 120 == 60) emitBuy(actual);
            else if (count % 120 == 0) emitSell(actual);
        }
    }
}
```

## 4. Bollinger Band Mean Reversion

Buy when price touches the lower band; sell at the upper band.

```java
import com.wualabs.qtsurfer.engine.indicators.helpers.group.InstrumentGroupRTIndicator;
import com.wualabs.qtsurfer.engine.indicators.helpers.WindowTimeRTIndicator.WindowTime;
import com.wualabs.qtsurfer.engine.strategy.AbstractWindowListener;
import com.wualabs.qtsurfer.engine.strategy.AbstractTickerStrategy;
import com.wualabs.qtsurfer.engine.core.state.StateStore;

public class BollingerReversionStrategy extends AbstractTickerStrategy {

    @Override
    protected void setupIndicators(InstrumentGroupRTIndicator indicators) {
        indicators
            .addPrice()
            .bollinger("bb", 20, 2.0)   // → "bb", "bbUpper", "bbLower"
            .window("price", WindowTime.s5, new BandListener(this, indicators));
    }

    private class BandListener extends AbstractWindowListener {

        BandListener(AbstractTickerStrategy s, InstrumentGroupRTIndicator ind) {
            super(s, ind);
        }

        @Override
        public void onChange(StateStore store, double prev, double actual) {
            if (!indicators.getExisting("bb").isReady()) return;

            double upper = indicators.getValue("bbUpper");
            double lower = indicators.getValue("bbLower");

            if (actual <= lower && !store.is("long")) {
                emitBuy(actual);
                store.set("long");
                store.unset("short");
            } else if (actual >= upper && !store.is("short")) {
                emitSell(actual);
                store.set("short");
                store.unset("long");
            }
        }
    }
}
```

## 5. Configurable dual-EMA with properties

Strategy parameters configurable at submission time via `@StrategyProperty`.

```java
import com.wualabs.qtsurfer.engine.indicators.helpers.group.InstrumentGroupRTIndicator;
import com.wualabs.qtsurfer.engine.strategy.AbstractWindowListener;
import com.wualabs.qtsurfer.engine.strategy.AbstractTickerStrategy;
import com.wualabs.qtsurfer.engine.strategy.CrossDetector;
import com.wualabs.qtsurfer.engine.strategy.StrategyProperty;
import com.wualabs.qtsurfer.engine.core.state.StateStore;

import java.time.Duration;

public class ConfigurableEmaStrategy extends AbstractTickerStrategy {

    @StrategyProperty(name = "ema.fast", description = "Fast EMA period", defaultValue = "9")
    private int fastPeriod = 9;

    @StrategyProperty(name = "ema.slow", description = "Slow EMA period", defaultValue = "21")
    private int slowPeriod = 21;

    @StrategyProperty(name = "window.seconds", description = "Window in seconds", defaultValue = "1")
    private int windowSeconds = 1;

    @Override
    protected void setupIndicators(InstrumentGroupRTIndicator indicators) {
        indicators
            .addPrice()
            .ema("fast", fastPeriod)
            .ema("slow", slowPeriod)
            .window("fast", Duration.ofSeconds(windowSeconds),
                new CrossListener(this, indicators));
    }

    private class CrossListener extends AbstractWindowListener {
        private final CrossDetector cross = new CrossDetector();

        CrossListener(AbstractTickerStrategy s, InstrumentGroupRTIndicator ind) {
            super(s, ind);
        }

        @Override
        public void onChange(StateStore store, double prev, double actual) {
            if (!indicators.getExisting("slow").isReady()) return;
            double slow = indicators.getValue("slow");
            CrossDetector.Cross result = cross.check(actual, slow);
            if (result.above()) emitBuy(actual);
            if (result.below()) emitSell(actual);
        }
    }
}
```
