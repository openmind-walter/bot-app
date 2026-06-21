---
form:
  id: rsi_parameters
  layout:
    areas:
      - ["initial_cash", "trade_mode"]
      - ["trade_value", "trade_pct"]
      - ["rsi_period", "rsi_period"]
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
      default: 14
      help: "Wilder RSI lookback."
      validation:
        required: true
        min: 2

    - id: oversold
      label: "Oversold"
      component: number
      bind: "params.oversold"
      default: 20
      help: "A cross up through this level fires a BUY."
      validation:
        required: true
        min: 0
        max: 100

    - id: overbought
      label: "Overbought"
      component: number
      bind: "params.overbought"
      default: 80
      help: "A cross down through this level fires a SELL."
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
      help: "When a closing signal fires, also open the opposite side in the same fill (full reverse) instead of just closing to flat — keeping the bot always in the market. Reversing a long into a short additionally requires Allow short."

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

# RSI Strategy Parameters

A single-timeframe RSI strategy that **opens and closes** one position,
reversing on the opposite signal (like `mtf_rsi`, but single-timeframe). It holds
at most one position: a signal *opposite* the open position **closes** it, a
matching signal HOLDs, and from flat a signal **opens**.

| Direction    | Trigger (Wilder RSI on the close stream)        |
| ------------ | ----------------------------------------------- |
| Long (Buy)   | RSI crosses **above** `oversold` (20) from below   |
| Short (Sell) | RSI crosses **below** `overbought` (80) from above |

With `close_and_reverse` on, a close also **reverses** — the order flattens the
old side and opens the new one in a single fill, keeping the bot in the market.
With it off, a close goes to flat and waits for the next same-direction signal.
Reversing a long into a short additionally needs `short_allowed`; on spot a SELL
can only close a long. Opening trades are sized to `trade_value` (capped by cash);
a close/reverse carries the full held quantity plus, when reversing, the new
opening size.

| Parameter      | Label            | Default | Meaning                                            |
| -------------- | ---------------- | ------- | -------------------------------------------------- |
| `initial_cash` | Initial cash ($) | 1000    | Starting budget. Opens are capped by remaining cash.|
| `trade_value`  | Trade value ($)  | 200     | Dollar notional committed per *opening* trade.     |
| `rsi_period`   | RSI period       | 14      | Wilder RSI lookback.                               |
| `oversold`     | Oversold         | 20      | A cross up through this fires a BUY (open/close long).|
| `overbought`   | Overbought       | 80      | A cross down through this fires a SELL (close long / open short).|
| `short_allowed`| Allow short      | false   | Whether a short may be held at all. Off (spot): a SELL only closes a long.|
| `close_and_reverse` | Close & reverse | false | On: a close also opens the opposite side (full reverse). Off: close to flat only. Reversing into a short also needs `short_allowed`.|
| `require_price_improvement` | Require price improvement | false | On: SELL above the last BUY & (if prior was a SELL) above the last SELL; BUY below the last SELL & (if prior was a BUY) below the last BUY. |

> **Note:** on spot (`short_allowed` off, the default) a SELL never takes the net
> position short — it only closes an open long, so `close_and_reverse` then has
> nothing to reverse into. Enable both for a margin/futures always-in-market
> reversing bot. The strategy advances per-bar scratch (`bots.indicator_state`)
> every candle via the worker's `step` path — see `strategy-core/src/rsi.rs`
> (`on_bar`).
