---
form:
  id: markov_parameters
  layout:
    areas:
      - ["initial_cash", "trade_mode"]
      - ["trade_value", "trade_pct"]
      - ["state_threshold", "min_samples"]
      - ["buy_prob", "sell_prob"]
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

    - id: state_threshold
      label: "State threshold (return fraction)"
      component: number
      bind: "params.state_threshold"
      default: 0.002
      help: "Half-width of the Flat return band as a fraction (0.002 = 0.2%). A bar return above +threshold is classed Up, below −threshold Down, otherwise Flat. Larger values class more bars as Flat."
      validation:
        required: true
        min: 0

    - id: min_samples
      label: "Min transitions (warmup)"
      component: number
      bind: "params.min_samples"
      default: 20
      help: "Minimum observed transitions in the current state's row before its forecast is trusted. Until the row reaches this count the bar holds."
      validation:
        required: true
        min: 1

    - id: buy_prob
      label: "Buy probability"
      component: number
      bind: "params.buy_prob"
      default: 0.55
      help: "Minimum forecast probability of an Up next state (and Up must dominate Down) to fire a long. Conventionally above 0.5."
      validation:
        required: true
        min: 0
        max: 1

    - id: sell_prob
      label: "Sell probability"
      component: number
      bind: "params.sell_prob"
      default: 0.55
      help: "Minimum forecast probability of a Down next state (and Down must dominate Up) to fire a short. Conventionally above 0.5."
      validation:
        required: true
        min: 0
        max: 1

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

# Markov Chain Probability Transition Strategy Parameters

A quantitative strategy that models price as a **Markov chain** over three discrete
states. Each bar's return is classified by a symmetric threshold band:

| State | Condition (`r` = bar return)                |
| ----- | ------------------------------------------- |
| Down  | `r < −state_threshold`                      |
| Flat  | `−state_threshold ≤ r ≤ state_threshold`    |
| Up    | `r > state_threshold`                       |

The strategy learns a 3×3 **transition-count matrix** online — how often a bar in
each state is followed by a bar in each state — carried in per-bar scratch and
cumulative over all history. Having just entered a state this bar, the matching
row is the empirical forecast of the **next** bar, and the strategy trades it:

| Direction    | Fires when (row of the current state)                       |
| ------------ | ----------------------------------------------------------- |
| Long (Buy)   | `P(next = Up) ≥ buy_prob` and `P(Up) > P(Down)`             |
| Short (Sell) | `P(next = Down) ≥ sell_prob` and `P(Down) > P(Up)`          |

A row is only trusted once it has at least `min_samples` observed transitions; a
cold row holds. The strategy **opens and closes** one position: a signal opposite
the open position closes it, a matching signal HOLDs, and from flat a signal
opens. With `close_and_reverse` a close also reverses into the opposite side;
reversing into a short additionally needs `short_allowed`.

| Parameter       | Label              | Default | Meaning                                            |
| --------------- | ------------------ | ------- | -------------------------------------------------- |
| `initial_cash`  | Initial cash ($)   | 1000    | Starting budget. Buys are capped by remaining cash.|
| `trade_value`   | Trade value ($)    | 200     | Dollar notional per signal (fixed-amount sizing).  |
| `state_threshold` | State threshold  | 0.002   | Flat-band half-width as a return fraction (0.2%).  |
| `min_samples`   | Min transitions    | 20      | Row warmup before its forecast is trusted.         |
| `buy_prob`      | Buy probability    | 0.55    | Min P(next=Up), Up dominant, to go long.           |
| `sell_prob`     | Sell probability   | 0.55    | Min P(next=Down), Down dominant, to go short.      |
| `short_allowed` | Allow short        | false   | Whether a short may be held at all. Off (spot): a SELL only closes a long.|
| `close_and_reverse` | Close & reverse | false | On: a close also opens the opposite side (full reverse). Off: close to flat only. Reversing into a short also needs `short_allowed`.|
| `require_price_improvement` | Require price improvement | false | On: SELL above the last BUY & (if prior was a SELL) above the last SELL; BUY below the last SELL & (if prior was a BUY) below the last BUY. |

> **Note:** on spot (`short_allowed` off, the default) a SELL only closes an open
> long — it never goes short, so `close_and_reverse` then has nothing to reverse
> into. Enable both for a margin/futures always-in-market reversing bot. The
> strategy advances per-bar scratch (`bots.indicator_state`) every candle via the
> worker's `step` path — see `strategy-core/src/markov.rs` (`on_bar`).
