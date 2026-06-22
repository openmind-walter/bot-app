---
form:
  id: ema_rsi_macd_parameters
  layout:
    areas:
      - ["initial_cash", "trade_mode"]
      - ["trade_value", "trade_pct"]
      - ["ema_period", "rsi_period"]
      - ["rsi_midline", "rsi_midline"]
      - ["fast_period", "slow_period"]
      - ["signal_period", "signal_period"]
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

    - id: ema_period
      label: "EMA period"
      component: number
      bind: "params.ema_period"
      default: 50
      help: "Trend-filter EMA lookback. A long needs a rising EMA with price above it; a short a falling EMA with price below it. Conventionally 50."
      validation:
        required: true
        min: 1

    - id: rsi_period
      label: "RSI period"
      component: number
      bind: "params.rsi_period"
      default: 14
      help: "Wilder RSI lookback. Conventionally 14."
      validation:
        required: true
        min: 1

    - id: rsi_midline
      label: "RSI midline"
      component: number
      bind: "params.rsi_midline"
      default: 50
      help: "Momentum threshold. A long needs RSI above it, a short below it. Conventionally 50."
      validation:
        required: true
        min: 0
        max: 100

    - id: fast_period
      label: "MACD fast EMA"
      component: number
      bind: "params.fast_period"
      default: 12
      help: "Fast (shorter) MACD EMA lookback. Conventionally 12."
      validation:
        required: true
        min: 1

    - id: slow_period
      label: "MACD slow EMA"
      component: number
      bind: "params.slow_period"
      default: 26
      help: "Slow (longer) MACD EMA lookback. Conventionally 26."
      validation:
        required: true
        min: 2

    - id: signal_period
      label: "MACD signal EMA"
      component: number
      bind: "params.signal_period"
      default: 9
      help: "EMA lookback of the MACD line (the trigger line). Conventionally 9."
      validation:
        required: true
        min: 1

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

# EMA + RSI + MACD Strategy Parameters

A trend-and-momentum strategy that **opens and closes** one position. The MACD
crossover is the **trigger**; the EMA trend filter and the RSI midline are
**confirmations** that must all agree before a cross trades. A signal opposite
the open position closes it, a matching signal HOLDs, and from flat a signal
opens. With `close_and_reverse` a close also reverses into the opposite side;
reversing into a short additionally needs `short_allowed`.

| Direction    | MACD trigger                        | EMA trend filter                        | RSI filter            |
| ------------ | ----------------------------------- | --------------------------------------- | --------------------- |
| Long (Buy)   | MACD line crosses **above** signal  | EMA **rising** and price **above** EMA  | RSI > `rsi_midline`   |
| Short (Sell) | MACD line crosses **below** signal  | EMA **falling** and price **below** EMA | RSI < `rsi_midline`   |

| Parameter       | Label            | Default | Meaning                                            |
| --------------- | ---------------- | ------- | -------------------------------------------------- |
| `initial_cash`  | Initial cash ($) | 1000    | Starting budget. Buys are capped by remaining cash.|
| `trade_value`   | Trade value ($)  | 200     | Dollar notional per signal (fixed-amount sizing).  |
| `ema_period`    | EMA period       | 50      | Trend-filter EMA lookback (slope + price side).    |
| `rsi_period`    | RSI period       | 14      | Wilder RSI lookback.                               |
| `rsi_midline`   | RSI midline      | 50      | Momentum threshold: long above, short below.       |
| `fast_period`   | MACD fast EMA    | 12      | Fast (shorter) MACD EMA lookback.                  |
| `slow_period`   | MACD slow EMA    | 26      | Slow (longer) MACD EMA lookback.                   |
| `signal_period` | MACD signal EMA  | 9       | EMA of the MACD line (the trigger line).           |
| `short_allowed` | Allow short      | false   | Whether a short may be held at all. Off (spot): a SELL only closes a long.|
| `close_and_reverse` | Close & reverse | false | On: a close also opens the opposite side (full reverse). Off: close to flat only. Reversing into a short also needs `short_allowed`.|
| `require_price_improvement` | Require price improvement | false | On: SELL above the last BUY & (if prior was a SELL) above the last SELL; BUY below the last SELL & (if prior was a BUY) below the last BUY. |

> **Note:** on spot (`short_allowed` off, the default) a SELL only closes an open
> long — it never goes short, so `close_and_reverse` then has nothing to reverse
> into. Enable both for a margin/futures always-in-market reversing bot. The
> strategy advances per-bar scratch (`bots.indicator_state`) every candle via the
> worker's `step` path — see `strategy-core/src/ema_rsi_macd.rs` (`on_bar`).
