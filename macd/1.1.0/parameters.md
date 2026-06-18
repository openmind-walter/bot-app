---
form:
  id: macd_parameters
  layout:
    areas:
      - ["initial_cash", "trade_mode"]
      - ["trade_value", "trade_pct"]
      - ["fast_period", "slow_period"]
      - ["signal_period", "cross_mode"]
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

    - id: fast_period
      label: "Fast EMA"
      component: number
      bind: "params.fast_period"
      default: 12
      help: "Fast (shorter) EMA lookback. Conventionally 12."
      validation:
        required: true
        min: 1

    - id: slow_period
      label: "Slow EMA"
      component: number
      bind: "params.slow_period"
      default: 26
      help: "Slow (longer) EMA lookback. Conventionally 26."
      validation:
        required: true
        min: 2

    - id: signal_period
      label: "Signal EMA"
      component: number
      bind: "params.signal_period"
      default: 9
      help: "EMA lookback of the MACD line (the trigger line). Conventionally 9."
      validation:
        required: true
        min: 1

    - id: cross_mode
      label: "Trigger"
      component: select
      bind: "params.cross_mode"
      default: signal
      help: "What fires a trade: the MACD line crossing its signal line, or the signal line crossing the zero level."
      options:
        - { value: signal, label: "MACD crosses signal line" }
        - { value: zero, label: "Signal line crosses zero" }

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

# MACD Strategy Parameters

A single-timeframe MACD strategy that **only ever opens** — it never
closes or offsets a position. Each crossing places an independent, fixed-size
order; realizing profit is handled separately by the `order.match` op, which
pairs opposing filled legs.

The MACD line is the difference of a fast and a slow EMA of the closes; the
signal line is an EMA of the MACD line. The `cross_mode` parameter selects the
trigger:

| `cross_mode`              | Long (Buy)                            | Short (Sell)                          |
| ------------------------- | ------------------------------------- | ------------------------------------- |
| `signal` (default)        | MACD line crosses **above** signal    | MACD line crosses **below** signal    |
| `zero`                    | signal line crosses **above** zero    | signal line crosses **below** zero    |

| Parameter       | Label            | Default | Meaning                                            |
| --------------- | ---------------- | ------- | -------------------------------------------------- |
| `initial_cash`  | Initial cash ($) | 1000    | Starting budget. Buys are capped by remaining cash.|
| `trade_value`   | Trade value ($)  | 200     | Dollar notional committed per signal.              |
| `fast_period`   | Fast EMA         | 12      | Fast (shorter) EMA lookback.                       |
| `slow_period`   | Slow EMA         | 26      | Slow (longer) EMA lookback.                        |
| `signal_period` | Signal EMA       | 9       | EMA of the MACD line (the trigger line).           |
| `cross_mode`    | Trigger          | signal  | MACD/signal crossover, or signal line vs zero.     |
| `short_allowed` | Allow short      | false   | Off (spot): SELLs cap at holdings, never go short. |

> **Note:** on spot (`short_allowed` off, the default) a SELL is capped at the
> held quantity, so the net position never goes below 0. Enable it only for
> margin/futures. The strategy advances per-bar scratch (`bots.indicator_state`)
> every candle via the worker's `step` path — see `strategy-core/src/macd.rs`
> (`on_bar`).
