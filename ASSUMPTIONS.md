# Assumptions Audit: socceraction Pipeline

## Overview

This document catalogs **implicit assumptions, under-specified states, and semantic constraints** embedded in the socceraction pipeline. It treats the system as a state machine with the following transformations:

```
Raw event stream
  → SPADL / Atomic-SPADL representation
    → Feature construction (game states)
      → Probability estimation (scoring / conceding)
        → Action valuation (VAEP, Atomic-VAEP, xT)
```

Each assumption is documented with:
- Explicit statement of what is assumed
- Code reference (file + function)
- Failure mode: what breaks when the assumption fails

---

## (A1) Representation-Level Assumptions

### A1.1: Temporally Contiguous Action Sequences

**Assumption**: SPADL assumes actions form temporally contiguous sequences within each period, with no meaningful temporal gaps that would invalidate state transitions.

**Code Reference**:
- `socceraction/vaep/formula.py:52-53` (`offensive_value`)
- `socceraction/vaep/formula.py:105-106` (`defensive_value`)

```python
toolong_idx = abs(actions.time_seconds - _prev(actions.time_seconds)) > _samephase_nb
prev_scores[toolong_idx] = 0.0
```

**Hardcoded Constant**: `_samephase_nb = 10` seconds

**Failure Mode**:
- If a stoppage (injury, VAR review, celebration) exceeds 10 seconds, the previous state is zeroed out.
- This creates **discontinuity artifacts**: the post-stoppage action is valued against a null prior state, not the actual pre-stoppage game state.
- Match context (scoreline, pressure) is lost across these boundaries.

**Semantic Gap**: 
The 10-second threshold is arbitrary. There is no mechanism to distinguish between:
- A natural phase transition (e.g., throw-in after ball out)
- An artificial stoppage (e.g., injury, VAR)
- A tactical reset (e.g., after a goal)

---

### A1.2: Unambiguous Action Ownership

**Assumption**: Every action has exactly one performing player. Player intent and execution are conflated into a single agent.

**Code Reference**:
- `socceraction/spadl/schema.py:20` (`SPADLSchema.player_id`)
- All data converters (e.g., `socceraction/spadl/statsbomb.py:82`)

```python
player_id: Series[Any] = pa.Field()
actions["player_id"] = events.player_id
```

**Failure Modes**:
1. **Deflections**: A shot deflected by a defender is attributed to the shooter. The defender's contribution is invisible.
2. **Rebounds**: If a goalkeeper parries a shot and another player scores, the rebound is a separate action. The causal dependency is not encoded.
3. **Own goals**: Encoded as `result_id = "owngoal"` but the action is attributed to the defending player, not the attacker who forced the situation.

**Semantic Gap**:
- Multi-agent contributions are flattened into single-player attribution.
- The system cannot distinguish between:
  - A shot that was aimed poorly (player intent)
  - A shot that was redirected by a deflection (external perturbation)

---

### A1.3: Discrete Action Boundaries

**Assumption**: Actions have well-defined start/end points. Continuous motion is discretized into atomic events.

**Code Reference**:
- `socceraction/spadl/base.py:_add_dribbles` (inserted synthetically)
- `socceraction/atomic/spadl/base.py` (decomposes actions further)

```python
# SPADL: Dribbles are inferred, not observed
actions = _add_dribbles(actions)
```

**Failure Modes**:
1. **Dribble Inference**: If a player holds the ball for >2 seconds without an event, a synthetic "dribble" is inserted. This is a **model artifact**, not an observed action.
2. **Sub-Action Ambiguity**: Atomic-SPADL decomposes actions (e.g., "receivepass + shot"), but the decomposition is rule-based, not data-driven. The boundary between "receive" and "shot" is definitional, not empirical.

**Semantic Gap**:
- The system assumes all relevant actions are observable in the event stream.
- Actions that are **not logged** (e.g., body feints, off-ball runs, defensive positioning) are invisible to SPADL.
- The discretization is **irreversible**: the original continuous motion cannot be reconstructed from SPADL.

---

### A1.4: Fixed Spatial Coordinate System

**Assumption**: All action coordinates are mapped to a 105m × 68m pitch, with uniform semantics across the entire field.

**Code Reference**:
- `socceraction/spadl/config.py:20-21`
- `socceraction/spadl/schema.py:21-24`

```python
field_length: float = 105.0  # unit: meters
field_width: float = 68.0  # unit: meters

start_x: Series[float] = pa.Field(ge=0, le=spadlconfig.field_length)
start_y: Series[float] = pa.Field(ge=0, le=spadlconfig.field_width)
```

**Failure Modes**:
1. **Pitch Variance**: Real pitches range from 100-110m × 64-75m. Coordinate normalization does not account for:
   - Different pitch dimensions
   - Turf conditions (e.g., wet corner vs. dry center)
2. **Semantic Density**: A 1m displacement in the penalty box is not semantically equivalent to a 1m displacement at midfield, yet features like `startlocation` and `endlocation` treat them identically.

**Semantic Gap**:
- Euclidean distance is a **poor proxy** for football-relevant distance.
- The system does not distinguish between:
  - Lateral movement (lower risk)
  - Forward penetration (higher value)
  - Backward recycling (possession retention)

---

### A1.5: Result Determinism

**Assumption**: Each action has a single, deterministic result from a fixed taxonomy.

**Code Reference**:
- `socceraction/spadl/config.py:24-31`

```python
results: list[str] = [
    "fail", "success", "offside", "owngoal", "yellow_card", "red_card"
]
```

**Failure Modes**:
1. **Pass Success Ambiguity**: A "successful" pass may still lose possession if the receiver is immediately pressured. The `result_id` encodes completion, not outcome.
2. **Shot Result Simplification**: A shot "saved" by the goalkeeper vs. "blocked" by a defender are both encoded as `result_id = "fail"`. The **causal mechanism** of failure is lost.

**Semantic Gap**:
- The result taxonomy is **under-specified** for downstream valuation.
- A failed shot due to poor execution vs. excellent defending are indistinguishable.

---

## (A2) Model-Level Assumptions

### A2.1: Markovian Game States

**Assumption**: The value of an action depends only on the current state (current action + 3 previous actions), not the full history.

**Code Reference**:
- `socceraction/vaep/base.py:71` (`nb_prev_actions = 3`)
- `socceraction/vaep/features.py:63-98` (`gamestates`)

```python
def gamestates(actions: Actions, nb_prev_actions: int = 3) -> GameStates:
    # ...
    for i in range(1, nb_prev_actions):
        prev_actions = actions.groupby(["game_id", "period_id"], sort=False, as_index=False).apply(
            lambda x: x.shift(i, fill_value=float("nan")).fillna(x.iloc[0])
        )
```

**Failure Modes**:
1. **Long-Range Dependencies**: A counter-attack initiated by a deep defensive clearance 5+ actions ago is invisible to the model.
2. **Tactical Context**: Formation changes, substitutions, or red cards are not encoded in the 3-action window.
3. **Scoreline Effects**: A risky pass when trailing 0-1 in the 90th minute has different value than the same pass when leading 3-0 in the 20th minute. The model is **scoreline-agnostic**.

**Semantic Gap**:
- The 3-action window is a **hyperparameter**, not a principled choice.
- The assumption of Markovianity is violated whenever:
  - Actions have delayed consequences (e.g., tiring an opponent over multiple duels)
  - Context shifts occur beyond the 3-action horizon

---

### A2.2: Action Independence (Label Construction)

**Assumption**: Goals are attributed to actions via a fixed 10-action lookahead window, with no weighting for temporal proximity or causal contribution.

**Code Reference**:
- `socceraction/vaep/labels.py:10` (`scores`, `concedes`)

```python
def scores(actions: DataFrame[SPADLSchema], nr_actions: int = 10) -> pd.DataFrame:
    # ...
    for i in range(1, nr_actions):
        gi = y["goal+%d" % i] & (y["team_id+%d" % i] == y["team_id"])
        res = res | gi | ogi
```

**Hardcoded Constant**: `nr_actions = 10`

**Failure Modes**:
1. **Equal Credit Assignment**: A pass that directly assists a goal has the same label (1.0) as a pass 9 actions earlier. There is no **credit decay**.
2. **Boundary Arbitrariness**: If a goal occurs on the 11th action, the first action gets label 0.0. This creates **cliff effects**.
3. **Possession Chain Leakage**: If possession changes hands multiple times within 10 actions, the label can be attributed to actions from the opposing team.

**Semantic Gap**:
- The model conflates "contributed to a goal" with "was followed by a goal within 10 actions."
- There is no mechanism to distinguish:
  - A key assist pass (high causal contribution)
  - A safe back-pass before the assist (low causal contribution)

---

### A2.3: Stationarity Across Match Contexts

**Assumption**: The learned classifiers (P(score | state), P(concede | state)) are stationary across all match contexts.

**Code Reference**:
- `socceraction/vaep/base.py:164-192` (`fit`)

```python
def fit(self, actions: pd.DataFrame, use_sampling: bool = True) -> "VAEP":
    # ...
    X = self.compute_features(g, a)
    Ys = [fn(a) for fn in self.yfns]
    # No stratification by scoreline, match phase, etc.
```

**Failure Modes**:
1. **Score Effects**: Trailing teams play differently (higher risk) than leading teams. The model does not condition on scoreline.
2. **Fatigue**: Actions in the 90th minute are different from actions in the 10th minute. The model only encodes `time_seconds`, not fatigue or urgency.
3. **Red Cards**: A red card fundamentally changes the game state (10 vs. 11), but this is not a feature.

**Semantic Gap**:
- The model assumes **stationary football**: all actions are drawn from the same distribution.
- Real football is **non-stationary**: tactics, scoreline, and player states shift continuously.

---

### A2.4: Linear Decomposition of Value

**Assumption**: VAEP value is the sum of offensive and defensive value, treating them as independent components.

**Code Reference**:
- `socceraction/vaep/formula.py:117-152` (`value`)

```python
v["offensive_value"] = offensive_value(actions, Pscores, Pconcedes)
v["defensive_value"] = defensive_value(actions, Pscores, Pconcedes)
v["vaep_value"] = v["offensive_value"] + v["defensive_value"]
```

**Failure Modes**:
1. **Non-Additive Value**: A risky pass may have positive offensive value (creates a chance) but also positive defensive vulnerability (loses possession in a dangerous area). The linear sum assumes these are separable.
2. **Value Clipping**: The sum can produce values outside [-1, 1], which are harder to interpret.

**Semantic Gap**:
- The decomposition assumes value is **monotonic** in both components.
- In reality, offensive and defensive value are coupled: high-risk, high-reward actions trade one for the other.

---

### A2.5: Fixed Baseline Probabilities for Set Pieces

**Assumption**: Penalties and corners have hardcoded baseline probabilities, overriding learned values.

**Code Reference**:
- `socceraction/vaep/formula.py:62-67` (`offensive_value`)

```python
# fixed odds of scoring when penalty
penalty_idx = actions.type_name == "shot_penalty"
prev_scores[penalty_idx] = 0.792453

# fixed odds of scoring when corner
corner_idx = actions.type_name.isin(["corner_crossed", "corner_short"])
prev_scores[corner_idx] = 0.046500
```

**Magic Constants**:
- Penalty: 79.2% goal probability
- Corner: 4.65% goal probability

**Failure Modes**:
1. **Contextual Variance**: Not all penalties are equally likely to be scored. The probability depends on:
   - Penalty taker skill
   - Goalkeeper quality
   - Match pressure
2. **Corner Variance**: Corner probability varies by delivery quality, defending setup, and attacking aerial presence.

**Semantic Gap**:
- These constants are **global averages**, not conditional on game state.
- They create **discontinuities**: a free kick just outside the box is valued by the model, but a penalty is valued by a constant.
- If the training data distribution differs from the hardcoded constants, the model produces **inconsistent valuations**.

---

## (A3) Contextual Assumptions

### A3.1: Normal Football Assumption

**Assumption**: The model assumes all actions occur during "normal" football, with no extreme contexts.

**Code Reference**:
- Absence of features in `socceraction/vaep/features.py:38-53`

**Missing Context Variables**:
- Scoreline differential
- Time remaining
- Player advantage/disadvantage (red cards)
- Formation or tactical setup

**Failure Modes**:
1. **Trailing Team Behavior**: A team trailing 0-2 in the 85th minute will take high-risk actions (e.g., long shots, aggressive pressing) that the model will undervalue because they are low-percentage.
2. **Red Card Asymmetry**: A team with 10 players will prioritize possession retention over chance creation. The model cannot distinguish this tactical shift.
3. **Parking the Bus**: A team leading 1-0 in the 88th minute will play defensively. The model will penalize "safe" back-passes even though they are strategically optimal.

**Semantic Gap**:
- The model optimizes for **expected value** without accounting for **variance-seeking** or **risk-averse** strategies that emerge in context.
- Actions are valued in isolation, not as part of a **meta-game** where scoreline and time dictate optimal play.

---

### A3.2: Possession Continuity

**Assumption**: Possession chains are implicitly tracked via team_id equality across consecutive actions.

**Code Reference**:
- `socceraction/vaep/formula.py:48-49` (`offensive_value`)

```python
sameteam = _prev(actions.team_id) == actions.team_id
prev_scores = (_prev(scores) * sameteam + _prev(concedes) * (~sameteam)).astype(float)
```

**Failure Modes**:
1. **Turnover Ambiguity**: A successful tackle by Team A followed by a pass by Team B is treated as a possession change. But if the pass is intercepted immediately, was possession truly lost?
2. **Unclear Ownership**: A loose ball contested by both teams has ambiguous ownership. The first successful touch "claims" the action, but the prior state is underdefined.

**Semantic Gap**:
- The model assumes **binary possession**: either Team A has the ball or Team B does.
- Real football has **contested states** (e.g., 50-50 balls, aerial duels) where possession is ambiguous.

---

### A3.3: Goal Reset Assumption

**Assumption**: After a goal is scored, the next action's prior state is set to zero.

**Code Reference**:
- `socceraction/vaep/formula.py:56-59` (`offensive_value`)

```python
prevgoal_idx = (_prev(actions.type_name).isin(["shot", "shot_freekick", "shot_penalty"])) & (
    _prev(actions.result_name) == "success"
)
prev_scores[prevgoal_idx] = 0.0
```

**Failure Modes**:
1. **Kickoff Value**: The first action after a goal (usually a kickoff) is valued against a null prior. In reality, the scoreline has shifted, which affects value.
2. **Celebration Time**: If the goal celebration lasts <10 seconds, the temporal continuity check (`toolong_idx`) does not trigger, and the model may incorrectly chain actions across the goal.

**Semantic Gap**:
- The reset is **local** (zeroing `prev_scores`) but not **global** (scoreline is not a feature).
- The model cannot distinguish between:
  - An equalizing goal (high strategic impact)
  - A consolation goal (low strategic impact)

---

## (A4) xT-Specific Assumptions

### A4.1: Grid Discretization

**Assumption**: The pitch is divided into a 12×16 grid, with uniform semantics within each cell.

**Code Reference**:
- `socceraction/xthreat.py:21-22`

```python
M: int = 12
N: int = 16
```

**Failure Modes**:
1. **Boundary Artifacts**: An action at (x=52.4m, y=34.0m) is in a different cell than (x=52.6m, y=34.0m), even though they are semantically identical.
2. **Intra-Cell Variance**: A shot from the penalty spot vs. a shot from the edge of the 6-yard box are in the same cell, but have different xG.

**Semantic Gap**:
- The grid is a **lossy compression** of spatial information.
- Cells near goal have higher value variance than cells near the halfway line, but the resolution is uniform.

---

### A4.2: Stationary Transition Matrix

**Assumption**: The transition probabilities (move from cell i to cell j) are learned from aggregate data and assumed stationary.

**Code Reference**:
- `socceraction/xthreat.py:74-98` (`scoring_prob`)

**Failure Modes**:
1. **Context Blindness**: The model does not condition on:
   - Action type (pass vs. dribble)
   - Defensive pressure
   - Player quality
2. **Sample Sparsity**: Cells with few observations have unreliable transition estimates.

**Semantic Gap**:
- The model assumes **homogeneous agents**: all players have the same transition probabilities.
- In reality, elite dribblers can move from low-value to high-value cells with higher probability than average players.

---

## Summary Table

| Assumption | Location | Failure Mode | Potential Invariant |
|------------|----------|--------------|---------------------|
| **A1.1**: Temporal continuity | `formula.py:52` | Stoppage >10s zeroes state | Assert: `time_delta < threshold` or inject context flag |
| **A1.2**: Single-agent actions | `schema.py:20` | Deflections, rebounds invisible | Encode multi-agent contributions in metadata |
| **A1.3**: Discrete boundaries | `base.py:_add_dribbles` | Synthetic dribbles are artifacts | Flag inferred vs. observed actions |
| **A1.4**: Fixed spatial coords | `config.py:20` | Pitch variance ignored | Normalize by pitch dimensions |
| **A1.5**: Result determinism | `config.py:24` | Causal mechanism of failure lost | Extend result taxonomy (e.g., "blocked" vs. "saved") |
| **A2.1**: Markovian states | `features.py:63` | Long-range dependencies lost | Expand `nb_prev_actions` or add recurrent features |
| **A2.2**: 10-action lookahead | `labels.py:10` | Equal credit to all actions | Weight labels by temporal proximity |
| **A2.3**: Stationarity | `base.py:164` | Scoreline/fatigue ignored | Add contextual features (scoreline, time, cards) |
| **A2.4**: Linear decomposition | `formula.py:151` | Risk/reward trade-offs flattened | Model as coupled optimization |
| **A2.5**: Fixed set-piece probs | `formula.py:62-67` | Context-free constants | Learn conditional probabilities |
| **A3.1**: Normal football | `features.py` | Extreme contexts misvalued | Add match state features |
| **A3.2**: Binary possession | `formula.py:48` | Contested states ambiguous | Encode possession confidence |
| **A3.3**: Goal reset | `formula.py:56` | Scoreline impact invisible | Add scoreline delta feature |
| **A4.1**: Grid discretization | `xthreat.py:21` | Boundary artifacts | Use continuous spatial models |
| **A4.2**: Stationary transitions | `xthreat.py:74` | Player heterogeneity ignored | Condition on player/team quality |

---

## Conclusion

The socceraction pipeline encodes a **rich set of implicit assumptions** about:
- How football actions are discretized
- What constitutes a "game state"
- How value propagates through action sequences
- What contexts are modeled vs. ignored

Each assumption creates **semantic brittleness**: the system's outputs are valid only to the extent that real football conforms to these constraints. When the assumptions fail (e.g., extreme scorelines, stoppages, contested possession), the model produces technically correct but semantically questionable valuations.

This audit provides a foundation for:
1. Designing **robustness tests** that probe these failure modes
2. Extending the model with **contextual features** to relax stationarity assumptions
3. Developing **calibration methods** to detect when assumptions are violated
