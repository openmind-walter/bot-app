---
form:
  id: rsi_parameters
  layout:
    areas:
      - ["initial_cash", "trade_value"]
      - ["rsi_period", "rsi_period"]
      - ["oversold", "overbought"]
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

    - id: trade_value
      label: "Trade value ($)"
      component: number
      bind: "params.trade_value"
      default: 200
      help: "Dollar notional committed per signal."
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

  submit:
    id: submit
    label: "Save parameters"
---

# RSI Strategy Parameters

A single-timeframe RSI strategy that **only ever opens** — it never closes or
offsets a position. Each crossing places an independent, fixed-size order;
realizing profit is handled separately by the `order.match` op, which pairs
opposing filled legs.

| Direction    | Trigger (Wilder RSI on the close stream)        |
| ------------ | ----------------------------------------------- |
| Long (Buy)   | RSI crosses **above** `oversold` (20) from below   |
| Short (Sell) | RSI crosses **below** `overbought` (80) from above |

| Parameter      | Label            | Default | Meaning                                            |
| -------------- | ---------------- | ------- | -------------------------------------------------- |
| `initial_cash` | Initial cash ($) | 1000    | Starting budget. Buys are capped by remaining cash.|
| `trade_value`  | Trade value ($)  | 200     | Dollar notional committed per signal.              |
| `rsi_period`   | RSI period       | 14      | Wilder RSI lookback.                               |
| `oversold`     | Oversold         | 20      | A cross up through this fires a BUY.               |
| `overbought`   | Overbought       | 80      | A cross down through this fires a SELL.            |

> **Note:** sells are uncapped, so the net position may go short. The strategy
> advances per-bar scratch (`bots.indicator_state`) every candle via the worker's
> `step` path — see `strategy-core/src/rsi.rs` (`on_bar`).
