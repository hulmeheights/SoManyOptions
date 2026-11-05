# NY iFVG strategy (NQ / MNQ, 5-minute)

Pine Script v6 strategy built from the video "How Much Can You Make in 1 Week Trading Once a Day?" (youtube.com/watch?v=K1Qw4CWbGCs).

## Rules from the video
1. Nasdaq only, 5-minute chart, New York session only, one trade per day.
2. **Bias** from three EMAs: price below them (especially the red one) and the EMAs pointing down means shorts only. The reverse means longs only.
3. Wait for a **fair value gap inside the pullback** that runs against the bias. Its size doesn't matter if it forms in the session.
4. **Entry:** when a candle inverts that gap, meaning it closes through it.
5. **Stop** "over three layers of protection". **Target** about 2:1 reward:risk, aiming at liquidity. Fixed risk per trade.

## Not stated in the video (adjustable settings, marked ASSUMED)
- EMA lengths and timeframe: defaults are 9/21/50 on 15-minute. The lines on screen step, which suggests a higher timeframe.
- How the slope is judged. Bias is checked on every candle: price above or below the red EMA, with the EMAs sloping the same way. Settings let you require price beyond all three EMAs, or lock the bias at the 9:30 open.
- What "three layers of protection" means. Here the stop goes beyond the pullback's high or low plus a buffer.
- Entry window: 9:30 to 11:30 New York time. Everything is flattened by 15:50.

## Use
TradingView → Pine Editor → paste `ny_ifvg_strategy.pine` → Add to chart (MNQ1! or NQ1!, 5m) → Strategy Tester.
Set commission and slippage to match your broker in the strategy's Properties tab.
