---
form:
  id: mtf_rsi_parameters
  layout:
    areas:
      - ["initial_cash", "trade_mode"]
      - ["trade_value", "trade_pct"]
      - ["rsi_period", "htf_ratio"]
      - ["oversold", "overbought"]
      - ["htf_trend", "htf_trend"]
      - ["close_and_reverse", "close_and_reverse"]
      - ["require_price_improvement", "require_price_improvement"]
      - ["submit", "submit"]

  fields:
    - id: initial_cash
      lock_on_edit: true
      label: "Initial cash ($)"
      component: number
      bind: "params.initial_cash"
      default: 1000
      help: "Starting deployable budget."
      validation:
        required: true
        min: 0

    - id: trade_mode
      label: "Trade size"
      component: select
      bind: "params.trade_mode"
      default: percent
      help: "Size each entry as a percent of remaining cash or a fixed dollar amount."
      options:
        - { value: percent, label: "Percent of cash (%)" }
        - { value: cash, label: "Fixed amount ($)" }

    - id: trade_pct
      label: "Trade % of cash"
      component: number
      bind: "params.trade_pct"
      default: 5
      help: "Percent of remaining cash committed per entry."
      visible_when:
        trade_mode: ["percent"]
      validation:
        required: true
        min: 0
        max: 100

    - id: trade_value
      label: "Trade value ($)"
      component: number
      bind: "params.trade_value"
      default: 200
      help: "Fixed dollar notional committed per entry."
      visible_when:
        trade_mode: ["cash"]
      validation:
        required: true
        min: 0

    - id: rsi_period
      label: "RSI period"
      component: number
      bind: "params.rsi_period"
      default: 14
      help: "Wilder RSI lookback, shared by both timeframes."
      validation:
        required: true
        min: 2

    - id: htf_ratio
      label: "HTF ratio (LTF bars per HTF bar)"
      component: select
      bind: "params.htf_ratio"
      default: 15
      help: "How many LTF bars make one HTF bar (e.g. 15 = a 15m HTF on a 1m bot)."
      options:
        - { value: 5, label: "5 — e.g. 5m HTF on a 1m bot" }
        - { value: 15, label: "15 — e.g. 15m HTF on a 1m bot" }
        - { value: 30, label: "30 — e.g. 30m HTF on a 1m bot" }
        - { value: 60, label: "60 — e.g. 1h HTF on a 1m bot" }
        - { value: 240, label: "240 — e.g. 4h HTF on a 1m bot" }
      validation:
        required: true

    - id: oversold
      label: "LTF oversold"
      component: number
      bind: "params.oversold"
      default: 30
      help: "LTF RSI must dip below this to arm a long, then snap back up to enter."
      validation:
        required: true
        min: 0
        max: 100

    - id: overbought
      label: "LTF overbought"
      component: number
      bind: "params.overbought"
      default: 70
      help: "LTF RSI must spike above this to arm a short, then snap back down to enter."
      validation:
        required: true
        min: 0
        max: 100

    - id: htf_trend
      label: "HTF trend midline"
      component: number
      bind: "params.htf_trend"
      default: 50
      help: "HTF RSI above this allows longs; below it allows shorts."
      validation:
        required: true
        min: 0
        max: 100

    - id: close_and_reverse
      label: "Close & reverse"
      component: checkbox
      bind: "params.close_and_reverse"
      default: false
      help: "When the opposite signal fires, reverse (close the open position AND open the new side in one fill, staying in the market) instead of just closing to flat. Off: the opposite signal flattens the position and the bot re-enters on the next signal."

    - id: require_price_improvement
      label: "Require price improvement"
      component: checkbox
      bind: "params.require_price_improvement"
      default: false
      help: "When on, an entry must improve on both the last opposite-side fill (SELL above the last BUY, BUY below the last SELL) and — when the previous action was the same side — the last same-side fill (each consecutive SELL higher than the prior SELL, each consecutive BUY lower than the prior BUY). A blocked entry/reversal stays in the current position."

  submit:
    id: submit
    label: "Save parameters"
---

# MTF-RSI Strategy Parameters

A multi-timeframe RSI strategy: a higher-timeframe (HTF) RSI confirms the trend
while a lower-timeframe (LTF) RSI snap-back triggers entry. Once triggered it
holds one position; the opposite signal **reverses** it (with `close_and_reverse`,
staying always in the market) or merely **closes** it to flat (with the bot
re-entering on the next signal).

| Direction   | HTF RSI filter (trend)   | LTF RSI trigger (entry)                          |
| ----------- | ------------------------ | ------------------------------------------------ |
| Long (Buy)  | RSI > `htf_trend` (50)   | RSI dips below `oversold` (30), then snaps back up   |
| Short (Sell)| RSI < `htf_trend` (50)   | RSI spikes above `overbought` (70), then snaps back down |

| Parameter      | Label                 | Default | Meaning                                                       |
| -------------- | --------------------- | ------- | ------------------------------------------------------------- |
| `initial_cash` | Initial cash ($)      | 1000    | Starting deployable budget.                                   |
| `trade_value`  | Trade value ($)       | 200     | Dollar notional committed per entry.                          |
| `rsi_period`   | RSI period            | 14      | Wilder RSI lookback, shared by both timeframes.               |
| `htf_ratio`    | HTF ratio             | 15      | LTF bars per HTF bar (15 = 15m HTF on a 1m bot).              |
| `oversold`     | LTF oversold          | 30      | Long arms when LTF RSI dips below this, fires on the snap-back.|
| `overbought`   | LTF overbought        | 70      | Short arms when LTF RSI spikes above this, fires on snap-back. |
| `htf_trend`    | HTF trend midline     | 50      | Longs need HTF RSI above this; shorts need it below.          |
| `close_and_reverse` | Close & reverse  | false   | On: the opposite signal reverses (close + open in one fill). Off: it closes to flat and re-enters on the next signal. |
| `require_price_improvement` | Require price improvement | false | On: SELL above the last BUY & (if prior was a SELL) above the last SELL; BUY below the last SELL & (if prior was a BUY) below the last BUY. A blocked reversal stays put. |

> **Note:** this strategy computes RSI from the price stream and so advances
> per-bar scratch (`bots.indicator_state`) every candle via the worker's `step`
> path — see `strategy-core/src/mtf_rsi.rs` (`on_bar`) and the `bot.advance_indicator`
> op. Requires a `strategy-worker` built with that path (the legacy `decide`-only
> host would not advance the indicators).
