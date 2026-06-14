---
form:
  id: aim_parameters
  layout:
    areas:
      - ["initial_cash", "ratchet_pct"]
      - ["buy_safe", "sell_safe"]
      - ["min_buy", "min_sell"]
      - ["submit", "submit"]

  fields:
    - id: initial_cash
      lock_on_edit: true
      label: "Initial cash ($)"
      component: number
      bind: "params.initial_cash"
      default: 1000
      help: "Starting cash, deployed half on the opening buy."
      validation:
        required: true
        min: 0

    - id: ratchet_pct
      label: "Ratchet (0–1)"
      component: number
      bind: "params.ratchet_pct"
      default: 0.5
      help: "Fraction of each buy added to the control value, banking profit over time."
      validation:
        required: true
        min: 0
        max: 1

    - id: buy_safe
      label: "Buy safe (%)"
      component: number
      bind: "params.buy_safe"
      default: 5
      help: "Buy-side safety band — ignores price dips smaller than this."
      validation:
        required: true
        min: 0
        max: 100

    - id: sell_safe
      label: "Sell safe (%)"
      component: number
      bind: "params.sell_safe"
      default: 8
      help: "Sell-side safety band — ignores price rises smaller than this."
      validation:
        required: true
        min: 0
        max: 100

    - id: min_buy
      label: "Min buy ($)"
      component: number
      bind: "params.min_buy"
      default: 10
      help: "Minimum trade value to act on a buy (filters tiny trades). 0 disables."
      validation:
        min: 0

    - id: min_sell
      label: "Min sell ($)"
      component: number
      bind: "params.min_sell"
      default: 10
      help: "Minimum trade value to act on a sell. 0 disables."
      validation:
        min: 0

  submit:
    id: submit
    label: "Save parameters"
---

# AIM Strategy Parameters

This Markdown body is ignored by the form engine — it documents the schema that
lives above the closing `---`. The form binds to the bot's `params` JSONB blob,
which `strategy-core` reads as [`AimParams`](../strategy-core/src/aim.rs).

| Parameter      | Label            | Default | Meaning                                                                    |
| -------------- | ---------------- | ------- | -------------------------------------------------------------------------- |
| `initial_cash` | Initial cash ($) | 1000    | Starting cash, deployed half on the opening buy.                           |
| `buy_safe`     | Buy safe (%)     | 5       | Buy-side safety band — ignores price dips smaller than this.               |
| `sell_safe`    | Sell safe (%)    | 8       | Sell-side safety band — ignores price rises smaller than this.             |
| `ratchet_pct`  | Ratchet (0–1)    | 0.5     | Fraction of each buy added to the control value, banking profit over time. |
| `min_buy`      | Min buy ($)      | 10      | Minimum trade value to act on a buy (filters tiny trades).                 |
| `min_sell`     | Min sell ($)     | 10      | Minimum trade value to act on a sell.                                      |

Percentages are whole numbers (e.g. `5` = 5%), matching `AimParams`. `min_buy` /
`min_sell` are optional (`0` disables the floor); the rest are required.
