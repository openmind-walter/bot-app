---
form:
  id: macross_parameters
  layout:
    areas:
      - ["initial_cash", "trade_mode"]
      - ["trade_value", "trade_pct"]
      - ["fast_period", "slow_period"]
      - ["short_allowed", "invert"]
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
      label: "Fast MA"
      component: number
      bind: "params.fast_period"
      default: 10
      help: "Fast (shorter) simple-moving-average window."
      validation:
        required: true
        min: 1

    - id: slow_period
      label: "Slow MA"
      component: number
      bind: "params.slow_period"
      default: 30
      help: "Slow (longer) simple-moving-average window."
      validation:
        required: true
        min: 2

    - id: short_allowed
      label: "Allow short (margin/futures)"
      component: checkbox
      bind: "params.short_allowed"
      lock_on_edit: true
      default: false
      help: "Off for spot: a SELL is capped at the held quantity and never takes the net position short."

    - id: invert
      label: "Invert signals"
      component: checkbox
      bind: "params.invert"
      default: false
      help: "Flip buy/sell: a golden cross SELLs and a death cross BUYS (fade the crossover instead of following it)."

  submit:
    id: submit
    label: "Save parameters"
---

# MA Cross Strategy Parameters

A two-moving-average crossover strategy that **only ever opens** — it never
closes or offsets a position. Each crossing places an independent, fixed-size
order; realizing profit is handled separately by the `order.match` op, which
pairs opposing filled legs.

Two simple moving averages of the closes — a fast (short) and a slow (long) —
and the strategy trades their crossover (the classic golden/death cross).

| Direction    | Trigger                                       |
| ------------ | --------------------------------------------- |
| Long (Buy)   | fast MA crosses **above** the slow MA (golden) |
| Short (Sell) | fast MA crosses **below** the slow MA (death)  |

With `invert` on, those sides are swapped: a golden cross SELLs and a death
cross BUYS.

| Parameter       | Label            | Default | Meaning                                            |
| --------------- | ---------------- | ------- | -------------------------------------------------- |
| `initial_cash`  | Initial cash ($) | 1000    | Starting budget. Buys are capped by remaining cash.|
| `trade_value`   | Trade value ($)  | 200     | Dollar notional committed per signal.              |
| `fast_period`   | Fast MA          | 10      | Fast (shorter) SMA window.                         |
| `slow_period`   | Slow MA          | 30      | Slow (longer) SMA window.                          |
| `short_allowed` | Allow short      | false   | Off (spot): SELLs cap at holdings, never go short. |
| `invert`        | Invert signals   | false   | Flip buy/sell: golden cross sells, death cross buys.|

> **Note:** on spot (`short_allowed` off, the default) a SELL is capped at the
> held quantity, so the net position never goes below 0. Enable it only for
> margin/futures. The strategy advances per-bar scratch (`bots.indicator_state`)
> every candle via the worker's `step` path — see `strategy-core/src/ma_cross.rs`
> (`on_bar`).
