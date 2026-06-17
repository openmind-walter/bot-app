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

  submit:
    id: submit
    label: "Save parameters"
---

# MTF-RSI Strategy Parameters

A multi-timeframe RSI strategy: a higher-timeframe (HTF) RSI confirms the trend
while a lower-timeframe (LTF) RSI snap-back triggers entry. The strategy stays
in the market once triggered and **reverses on the opposite signal**.

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

> **Note:** this strategy computes RSI from the price stream and so advances
> per-bar scratch (`bots.indicator_state`) every candle via the worker's `step`
> path — see `strategy-core/src/mtf_rsi.rs` (`on_bar`) and the `bot.advance_indicator`
> op. Requires a `strategy-worker` built with that path (the legacy `decide`-only
> host would not advance the indicators).
