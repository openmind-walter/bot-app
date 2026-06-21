---
form:
  id: macd_parameters
  layout:
    areas:
      - ["initial_cash", "trade_mode"]
      - ["trade_value", "trade_pct"]
      - ["fast_period", "slow_period"]
      - ["signal_period", "cross_mode"]
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

# MACD Strategy Parameters

A single-timeframe MACD strategy that **opens and closes** one position: a signal
opposite the open position closes it, a matching signal HOLDs, and from flat a
signal opens. With `close_and_reverse` a close also reverses into the opposite
side; reversing into a short additionally needs `short_allowed`.

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
| `short_allowed` | Allow short      | false   | Whether a short may be held at all. Off (spot): a SELL only closes a long.|
| `close_and_reverse` | Close & reverse | false | On: a close also opens the opposite side (full reverse). Off: close to flat only. Reversing into a short also needs `short_allowed`.|
| `require_price_improvement` | Require price improvement | false | On: SELL above the last BUY & (if prior was a SELL) above the last SELL; BUY below the last SELL & (if prior was a BUY) below the last BUY. |

> **Note:** on spot (`short_allowed` off, the default) a SELL only closes an open
> long — it never goes short, so `close_and_reverse` then has nothing to reverse
> into. Enable both for a margin/futures always-in-market reversing bot. The
> strategy advances per-bar scratch (`bots.indicator_state`) every candle via the
> worker's `step` path — see `strategy-core/src/macd.rs` (`on_bar`).
