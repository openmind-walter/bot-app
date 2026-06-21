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
      - ["short_allowed", "close_and_reverse"]
      - ["require_price_improvement", "require_price_improvement"]
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
      default: 9
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
      default: 15
      help: "A %D cross up through this level fires a BUY."
      validation:
        required: true
        min: 0
        max: 100

    - id: overbought
      label: "Overbought"
      component: number
      bind: "params.overbought"
      default: 75
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
      help: "Whether the strategy may hold a short at all (open a short from flat, or — with Close & reverse — reverse a long into one). Off for spot: a SELL only ever closes an open long and from flat HOLDs."

    - id: close_and_reverse
      label: "Close & reverse"
      component: checkbox
      bind: "params.close_and_reverse"
      default: false
      help: "When a closing signal fires, also open the opposite side in the same fill (full reverse) instead of just closing to flat. Reversing a long into a short additionally requires Allow short."

    - id: require_price_improvement
      label: "Require price improvement"
      component: checkbox
      bind: "params.require_price_improvement"
      default: false
      help: "When on, a signal must improve on both the last opposite-side fill (SELL above the last BUY, BUY below the last SELL) and — when the previous action was the same side — the last same-side fill (each consecutive SELL higher than the prior SELL, each consecutive BUY lower than the prior BUY). A trade with no relevant reference yet is unconstrained."

  submit:
    id: submit
    label: "Save parameters"
---

# Stochastic Strategy Parameters

A single-timeframe (15m) **slow stochastic** strategy that **opens and closes**
one position: a signal opposite the open position closes it, a matching signal
HOLDs, and from flat a signal opens. With `close_and_reverse` a close also
reverses into the opposite side; reversing into a short additionally needs
`short_allowed`.

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
| `oversold`     | Oversold         | 15      | A %D cross up through this fires a BUY.            |
| `overbought`   | Overbought       | 75      | A %D cross down through this fires a SELL.         |
| `short_allowed`| Allow short      | false   | Whether a short may be held at all. Off (spot): a SELL only closes a long.|
| `close_and_reverse` | Close & reverse | false | On: a close also opens the opposite side (full reverse). Off: close to flat only. Reversing into a short also needs `short_allowed`.|
| `require_price_improvement` | Require price improvement | false | On: SELL above the last BUY & (if prior was a SELL) above the last SELL; BUY below the last SELL & (if prior was a BUY) below the last BUY. |

> **Note:** on spot (`short_allowed` off, the default) a SELL only closes an open
> long — it never goes short, so `close_and_reverse` then has nothing to reverse
> into. Enable both for a margin/futures always-in-market reversing bot. The
> strategy advances per-bar scratch (`bots.indicator_state`) every candle via the
> worker's `step` path — see `strategy-core/src/stochastic.rs` (`on_bar`).
