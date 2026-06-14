---
form:
  id: bs_parameters
  layout:
    areas:
      - ["mode", "mode"]
      - ["initial_cash", "min_trade_value"]
      - ["buy_price_down", "sell_price_up"]
      - ["min_profit_pct", "min_profit_pct"]
      - ["submit", "submit"]

  fields:
    - id: mode
      lock_on_edit: true
      label: "Direction"
      component: select
      bind: "params.mode"
      default: both
      options:
        - { value: long, label: "Long only — buy dips, sell to take profit" }
        - { value: short, label: "Short only — short rallies, buy back to take profit" }
        - { value: both, label: "Both — ladder buys lower and shorts higher" }
      help: "Which sides of the grid are active."
      validation:
        required: true

    - id: initial_cash
      label: "Initial cash ($)"
      lock_on_edit: true
      component: number
      bind: "params.initial_cash"
      default: 1000
      help: "Deployable budget for the grid. Each leg costs one trade; the budget caps grid depth."
      validation:
        required: true
        min: 0

    - id: min_trade_value
      label: "Trade value ($)"
      component: number
      bind: "params.min_trade_value"
      default: 200
      help: "Dollar value committed per trade (each grid leg)."
      validation:
        required: true
        min: 0

    - id: buy_price_down
      label: "Buy when price drops (%)"
      component: number
      bind: "params.buy_price_down"
      default: 5
      help: "How far price must drop below the lowest open buy before buying another leg."
      validation:
        required: true
        min: 0
        max: 100

    - id: sell_price_up
      label: "Short when price rises (%)"
      component: number
      bind: "params.sell_price_up"
      default: 5
      help: "How far price must rise above the highest open sell before shorting another leg."
      validation:
        required: true
        min: 0
        max: 100

    - id: min_profit_pct
      label: "Profit target (%)"
      component: number
      bind: "params.min_profit_pct"
      default: 10
      help: "How far a leg must move in its favour before it is closed for profit."
      validation:
        required: true
        min: 0
        max: 100

  submit:
    id: submit
    label: "Save parameters"
---

# BS (buy_sell) Strategy Parameters

This Markdown body is ignored by the form engine — it documents the schema that
lives above the closing `---`. The form binds to the bot's `params` JSONB blob,
which `strategy-core` reads as [`BsParams`](../strategy-core/src/bs.rs).

BS runs a **grid of independent legs**. Depending on `mode` it ladders buys
lower, shorts higher, or both, opening an initial leg, adding more as price runs,
and closing each leg on its own once it reaches the profit target.

| Parameter         | Label                       | Default | Meaning                                                                  |
| ----------------- | --------------------------- | ------- | ------------------------------------------------------------------------ |
| `mode`            | Direction                   | both    | Which sides trade: `long`, `short`, or `both`.                           |
| `initial_cash`    | Initial cash ($)            | 1000    | Deployable budget for the grid; the budget caps how deep the grid grows. |
| `min_trade_value` | Trade value ($)             | 200     | Dollar value committed per trade (each grid leg).                        |
| `buy_price_down`  | Buy when price drops (%)    | 5       | How far price must drop before buying another leg (long side).           |
| `sell_price_up`   | Short when price rises (%)  | 5       | How far price must rise before shorting another leg (short side).        |
| `min_profit_pct`  | Profit target (%)           | 10      | How far a leg must move in its favour before it is closed.               |

`mode` is one of `long` / `short` / `both`; in `long` only `buy_price_down`
applies and in `short` only `sell_price_up`. Percentages are whole numbers (e.g.
`5` = 5%), matching `BsParams`. All parameters are required (`mode` defaults to
`both`).
