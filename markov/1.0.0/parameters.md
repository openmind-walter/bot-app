---
form:
  id: markov_parameters
  layout:
    areas:
      - ["initial_cash", "trade_mode"]
      - ["trade_value", "trade_pct"]
      - ["bull_to_bull", "bull_to_bear", "bull_to_stagnant"]
      - ["bear_to_bull", "bear_to_bear", "bear_to_stagnant"]
      - ["stagnant_to_bull", "stagnant_to_bear", "stagnant_to_stagnant"]
      - ["short_allowed", "short_allowed", "short_allowed"]
      - ["submit", "submit", "submit"]

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

    - id: bull_to_bull
      label: "Bull → Bull"
      component: number
      bind: "params.bull_to_bull"
      default: 0.7
      help: "Probability the next state is Bull given the current state is Bull. The Bull row should sum to ~1."
      validation:
        required: true
        min: 0
        max: 1

    - id: bull_to_bear
      label: "Bull → Bear"
      component: number
      bind: "params.bull_to_bear"
      default: 0.2
      help: "Probability the next state is Bear given the current state is Bull."
      validation:
        required: true
        min: 0
        max: 1

    - id: bull_to_stagnant
      label: "Bull → Stagnant"
      component: number
      bind: "params.bull_to_stagnant"
      default: 0.1
      help: "Probability the next state is Stagnant given the current state is Bull."
      validation:
        required: true
        min: 0
        max: 1

    - id: bear_to_bull
      label: "Bear → Bull"
      component: number
      bind: "params.bear_to_bull"
      default: 0.3
      help: "Probability the next state is Bull given the current state is Bear. The Bear row should sum to ~1."
      validation:
        required: true
        min: 0
        max: 1

    - id: bear_to_bear
      label: "Bear → Bear"
      component: number
      bind: "params.bear_to_bear"
      default: 0.5
      help: "Probability the next state is Bear given the current state is Bear."
      validation:
        required: true
        min: 0
        max: 1

    - id: bear_to_stagnant
      label: "Bear → Stagnant"
      component: number
      bind: "params.bear_to_stagnant"
      default: 0.2
      help: "Probability the next state is Stagnant given the current state is Bear."
      validation:
        required: true
        min: 0
        max: 1

    - id: stagnant_to_bull
      label: "Stagnant → Bull"
      component: number
      bind: "params.stagnant_to_bull"
      default: 0.4
      help: "Probability the next state is Bull given the current state is Stagnant. The Stagnant row should sum to ~1."
      validation:
        required: true
        min: 0
        max: 1

    - id: stagnant_to_bear
      label: "Stagnant → Bear"
      component: number
      bind: "params.stagnant_to_bear"
      default: 0.3
      help: "Probability the next state is Bear given the current state is Stagnant."
      validation:
        required: true
        min: 0
        max: 1

    - id: stagnant_to_stagnant
      label: "Stagnant → Stagnant"
      component: number
      bind: "params.stagnant_to_stagnant"
      default: 0.3
      help: "Probability the next state is Stagnant given the current state is Stagnant."
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
      help: "Whether the strategy may hold a short. Off for spot: a Bear target only closes an open long (down to flat) and from flat HOLDs. On: a Bear target opens/holds a short and a Bull→Bear flip reverses through flat."

  submit:
    id: submit
    label: "Save parameters"
---

# Markov Chain Probability Transition Strategy Parameters

A quantitative strategy that models price as a **Markov chain** over three states —
**Bull**, **Bear**, **Stagnant** — set each bar by the sign of the move
(`close > prev` Bull, `close < prev` Bear, equal Stagnant).

Unlike a learned model the **transition matrix is fixed**: the nine `*_to_*`
parameters are the probabilities of the next state given the current one. A
deterministic counter cycling `1,2,…,9,0,…` selects the next state from the
current state's row by cumulative-probability thresholds scaled to `0..10`:

```text
row = (to_bull, to_bear, to_stagnant)        // for the current state
if   counter <  to_bull·10            → Bull
elif counter < (to_bull + to_bear)·10 → Bear
else                                  → Stagnant
```

The predicted next state sets the **target position**, and the strategy moves to it:

| Predicted next state | Target   | Action                                            |
| -------------------- | -------- | ------------------------------------------------- |
| Bull                 | Long     | Open/keep a long (reverse a short through flat).  |
| Bear                 | Short    | Open/keep a short (reverse a long). Spot: close long to flat. |
| Stagnant             | Flat     | Close any open position.                          |

On spot (`short_allowed` off, the default) the strategy can't hold a short, so a
Bear target only closes an open long.

| Parameter | Label | Default | Meaning |
| --- | --- | --- | --- |
| `initial_cash` | Initial cash ($) | 1000 | Starting budget. Buys are capped by remaining cash. |
| `trade_value` | Trade value ($) | 200 | Dollar notional per signal (fixed-amount sizing). |
| `bull_to_bull` | Bull → Bull | 0.7 | P(next Bull \| Bull). |
| `bull_to_bear` | Bull → Bear | 0.2 | P(next Bear \| Bull). |
| `bull_to_stagnant` | Bull → Stagnant | 0.1 | P(next Stagnant \| Bull). |
| `bear_to_bull` | Bear → Bull | 0.3 | P(next Bull \| Bear). |
| `bear_to_bear` | Bear → Bear | 0.5 | P(next Bear \| Bear). |
| `bear_to_stagnant` | Bear → Stagnant | 0.2 | P(next Stagnant \| Bear). |
| `stagnant_to_bull` | Stagnant → Bull | 0.4 | P(next Bull \| Stagnant). |
| `stagnant_to_bear` | Stagnant → Bear | 0.3 | P(next Bear \| Stagnant). |
| `stagnant_to_stagnant` | Stagnant → Stagnant | 0.3 | P(next Stagnant \| Stagnant). |
| `short_allowed` | Allow short | false | Whether a short may be held. Off (spot): a Bear target only closes a long. |

> **Note:** each row (Bull / Bear / Stagnant) should sum to ~1; any shortfall
> falls through to Stagnant by the cumulative rule. The strategy advances per-bar
> scratch (`bots.indicator_state`) every candle via the worker's `step` path — see
> `strategy-core/src/markov.rs` (`on_bar`).
