---
form:
  id: stochastic_parameters
  layout:
    areas:
      - ["initial_cash", "trade_mode"]
      - ["trade_value", "trade_pct"]
      - ["k_period", "smooth_k"]
      - ["d_period", "d_period"]
      - ["oversold", "overbought"]
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
      help: "Size each signal as a percent of capital or a fixed dollar amount."
      options:
        - { value: percent, label: "Percent of capital (%)" }
        - { value: cash, label: "Fixed amount ($)" }

    - id: trade_pct
      label: "Trade % of cash"
      component: number
      bind: "params.trade_pct"
      default: 5
      help: "Percent of the remaining cash committed per signal — the dollar size shrinks as the budget deploys."
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

    - id: k_period
      label: "%K period"
      component: number
      bind: "params.k_period"
      default: 14
      help: "Lookback for the %K high/low window (number of closes)."
      validation:
        required: true
        min: 2

    - id: smooth_k
      label: "%K smoothing"
      component: number
      bind: "params.smooth_k"
      default: 3
      help: "SMA length smoothing raw %K into slow %K."
      validation:
        required: true
        min: 1

    - id: d_period
      label: "%D period"
      component: number
      bind: "params.d_period"
      default: 3
      help: "SMA length smoothing slow %K into %D (the trigger line)."
      validation:
        required: true
        min: 1

    - id: oversold
      label: "Oversold"
      component: number
      bind: "params.oversold"
      default: 20
      help: "A %D cross up through this level fires a BUY."
      validation:
        required: true
        min: 0
        max: 100

    - id: overbought
      label: "Overbought"
      component: number
      bind: "params.overbought"
      default: 80
      help: "A %D cross down through this level fires a SELL."
      validation:
        required: true
        min: 0
        max: 100

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

# Stochastic Strategy Parameters

A single-timeframe **slow stochastic** strategy that **only ever opens** — it
never closes or offsets a position. Each %D crossing places an independent,
fixed-size order; realizing profit is handled separately by the `order.match`
op, which pairs opposing filled legs.

Because the engine sees only each bar's close (no high/low), the oscillator is
**close-based**: raw `%K = 100·(close − min) / (max − min)` over the last
`k_period` closes, smoothed by `smooth_k` → slow %K, then by `d_period` → %D.

| Direction    | Trigger (slow stochastic %D)                    |
| ------------ | ----------------------------------------------- |
| Long (Buy)   | %D crosses **above** `oversold` (20) from below    |
| Short (Sell) | %D crosses **below** `overbought` (80) from above  |

| Parameter      | Label            | Default | Meaning                                            |
| -------------- | ---------------- | ------- | -------------------------------------------------- |
| `initial_cash` | Initial cash ($) | 1000    | Starting budget. Buys are capped by remaining cash.|
| `trade_mode`   | Trade size       | percent | `percent` of remaining cash or a fixed `cash` amount.|
| `trade_pct`    | Trade % of cash  | 5       | Percent of remaining cash per signal (`mode=percent`).|
| `trade_value`  | Trade value ($)  | 200     | Fixed $ per signal (when `mode=cash`).             |
| `k_period`     | %K period        | 14      | Lookback for the %K high/low window.               |
| `smooth_k`     | %K smoothing     | 3       | SMA smoothing raw %K into slow %K.                 |
| `d_period`     | %D period        | 3       | SMA smoothing slow %K into %D.                     |
| `oversold`     | Oversold         | 20      | A %D cross up through this fires a BUY.            |
| `overbought`   | Overbought       | 80      | A %D cross down through this fires a SELL.         |
| `short_allowed`| Allow short      | false   | Off (spot): SELLs cap at holdings, never go short. |

> **Note:** on spot (`short_allowed` off, the default) a SELL is capped at the
> held quantity, so the net position never goes below 0. Enable it only for
> margin/futures. The strategy advances per-bar scratch (`bots.indicator_state`)
> every candle via the worker's `step` path — see
> `strategy-core/src/stochastic.rs` (`on_bar`).
