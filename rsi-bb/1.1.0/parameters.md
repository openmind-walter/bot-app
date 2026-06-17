---
form:
  id: rsi_bb_parameters
  layout:
    areas:
      - ["initial_cash", "trade_mode"]
      - ["trade_value", "trade_pct"]
      - ["rsi_period", "rsi_period"]
      - ["oversold", "overbought"]
      - ["bb_period", "bb_std"]
      - ["short_allowed", "short_allowed"]
      - ["submit", "submit"]

  fields:
    - id: initial_cash
      lock_on_edit: true
      label: "Initial cash ($)"
      component: number
      bind: "params.initial_cash"
      default: 1000
      help: "Starting deployable budget. Buys are capped by remaining cash."
      validation:
        required: true
        min: 0

    - id: trade_mode
      label: "Trade size"
      component: select
      bind: "params.trade_mode"
      default: percent
      help: "Size each signal as a percent of remaining cash or a fixed dollar amount."
      options:
        - { value: percent, label: "Percent of cash (%)" }
        - { value: cash, label: "Fixed amount ($)" }

    - id: trade_pct
      label: "Trade % of cash"
      component: number
      bind: "params.trade_pct"
      default: 5
      help: "Percent of remaining cash committed per signal."
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
      help: "Fixed dollar notional committed per signal."
      visible_when:
        trade_mode: ["cash"]
      validation:
        required: true
        min: 0

    - id: rsi_period
      label: "RSI period"
      component: number
      bind: "params.rsi_period"
      default: 4
      help: "Wilder RSI lookback."
      validation:
        required: true
        min: 2

    - id: oversold
      label: "Oversold"
      component: number
      bind: "params.oversold"
      default: 20
      help: "A cross up through this level arms a BUY (confirmed at the lower band)."
      validation:
        required: true
        min: 0
        max: 100

    - id: overbought
      label: "Overbought"
      component: number
      bind: "params.overbought"
      default: 80
      help: "A cross down through this level arms a SELL (confirmed at the upper band)."
      validation:
        required: true
        min: 0
        max: 100

    - id: bb_period
      label: "Bollinger period (SMA)"
      component: number
      bind: "params.bb_period"
      default: 20
      help: "Lookback for the Bollinger middle band (simple moving average) and its standard deviation."
      validation:
        required: true
        min: 2

    - id: bb_std
      label: "Bollinger std-dev"
      component: number
      bind: "params.bb_std"
      default: 2
      help: "How many standard deviations the outer bands sit from the SMA (typically 2)."
      validation:
        required: true
        min: 0

    - id: short_allowed
      label: "Allow short (margin/futures)"
      component: checkbox
      bind: "params.short_allowed"
      lock_on_edit: true
      default: false
      help: "Off for spot: a SELL is capped at the held quantity and never takes the net position short."

  submit:
    id: submit
    label: "Save parameters"
---

# RSI + Bollinger Band Strategy Parameters

An RSI crossing strategy gated by a Bollinger Band touch. It **only ever opens** —
it never closes or offsets a position. Each *confirmed* crossing places an
independent, fixed-size order; realizing profit is handled separately by the
`order.match` op, which pairs opposing filled legs.

| Direction    | RSI crossing (Wilder RSI on closes)               | Bollinger confirmation |
| ------------ | ------------------------------------------------- | ---------------------- |
| Long (Buy)   | RSI crosses **above** `oversold` (20) from below  | close ≤ **lower** band |
| Short (Sell) | RSI crosses **below** `overbought` (80) from above| close ≥ **upper** band |

The RSI cross signals an exit from an overbought/oversold extreme — a likely
short-term reversal — and the band touch confirms price is stretched far enough
from its moving average for that reversal to be worth trading. A crossing that
isn't band-confirmed on its bar is skipped.

| Parameter       | Label                  | Default | Meaning                                              |
| --------------- | ---------------------- | ------- | ---------------------------------------------------- |
| `initial_cash`  | Initial cash ($)       | 1000    | Starting budget. Buys are capped by remaining cash.  |
| `trade_value`   | Trade value ($)        | 200     | Dollar notional committed per signal.                |
| `rsi_period`    | RSI period             | 4       | Wilder RSI lookback.                                 |
| `oversold`      | Oversold               | 20      | Cross up through this arms a BUY.                    |
| `overbought`    | Overbought             | 80      | Cross down through this arms a SELL.                 |
| `bb_period`     | Bollinger period (SMA) | 20      | Lookback for the SMA middle band + standard deviation.|
| `bb_std`        | Bollinger std-dev      | 2       | Std-deviation multiplier for the band width.         |
| `short_allowed` | Allow short            | false   | Off (spot): SELLs cap at holdings, never go short.   |

> **Note:** on spot (`short_allowed` off, the default) a SELL is capped at the
> held quantity, so the net position never goes below 0. Enable it only for
> margin/futures. The strategy advances per-bar scratch (`bots.indicator_state`)
> every candle via the worker's `step` path — see `strategy-core/src/rsi_bb.rs`
> (`on_bar`).
