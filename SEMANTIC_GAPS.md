# Semantic Gaps: Invariant Violations in socceraction

## Purpose

This document classifies each weird state identified in the socceraction audit by the **semantic invariant** it violates. The goal is to make failure modes **legible**, not to propose fixes.

For each weird state, we provide:
1. **Classification** into one of five categories
2. **Implicit invariant** the system relies on
3. **Violation mechanism** explaining how the weird state breaks the invariant
4. **Formal statement** of the violated invariant

---

## Classification Taxonomy

| Class | Description |
|-------|-------------|
| **Temporal Under-Specification** | System assumes temporal relationships without explicit encoding |
| **Attribution Ambiguity** | Credit/blame assignment is under-constrained or conflated |
| **Discretization-Induced Non-Smoothness** | Continuous phenomena quantized into discrete bins, creating boundary artifacts |
| **Exogenous Probability Injection** | External constants override learned distributions |
| **Stationarity Violation** | Model assumes time-invariant distributions that shift in practice |

---

## Weird State Classifications

### WS1: Temporal Discontinuity Cliff

**Classification**: Temporal Under-Specification

**Implicit Invariant**: 
> The system assumes that **temporal proximity implies causal continuity**: actions separated by small time intervals are part of the same coherent possession phase.

**Violation Mechanism**:
The 10-second threshold in `formula.py:52` creates a **hard boundary**:
```python
toolong_idx = abs(actions.time_seconds - _prev(actions.time_seconds)) > 10
prev_scores[toolong_idx] = 0.0
```

When `Δt = 9s`, the system preserves the prior state (continuity).  
When `Δt = 11s`, the system **zeroes** the prior state (discontinuity).

This violates the invariant because:
- **No semantic justification** exists for the 10-second boundary. A stoppage lasting 10.5 seconds is not qualitatively different from one lasting 9.5 seconds.
- The system cannot distinguish between:
  - Natural pauses (ball out of play)
  - Artificial stoppages (VAR, injury)
  - Strategic resets (goal scored)

**Formal Statement**:
```
∀ actions aᵢ, aᵢ₊₁:
  IF |t(aᵢ₊₁) - t(aᵢ)| ≤ τ THEN state(aᵢ₊₁) depends on state(aᵢ)
  ELSE state(aᵢ₊₁) is independent of state(aᵢ)

WHERE τ = 10 seconds (arbitrary constant)

VIOLATED INVARIANT:
  The threshold τ is treated as a semantic boundary, but it is merely a hyperparameter.
  Small perturbations around τ cause discontinuous value jumps.
```

**Mathematical Expression**:
$$
V(a_{i+1}) = 
\begin{cases} 
f(S_i, a_{i+1}) & \text{if } |t_{i+1} - t_i| \leq 10 \\
f(\emptyset, a_{i+1}) & \text{if } |t_{i+1} - t_i| > 10
\end{cases}
$$

The function $V$ is **non-smooth** at $|t_{i+1} - t_i| = 10$, despite $f$ being smooth.

---

### WS2: Credit Assignment Ambiguity

**Classification**: Attribution Ambiguity

**Implicit Invariant**:
> The system assumes that **all actions within a 10-action window preceding a goal contribute equally** to the outcome, independent of their causal proximity or strategic role.

**Violation Mechanism**:
The labeling function in `labels.py:10-51` assigns binary labels:
```python
for i in range(1, nr_actions):
    gi = y["goal+%d" % i] & (y["team_id+%d" % i] == y["team_id"])
    res = res | gi | ogi
```

This creates a **flat credit structure**:
- Action 1 (back-pass from goalkeeper): `scores = True`
- Action 9 (assist pass): `scores = True`

Both receive identical labels, despite vastly different causal contributions.

**Formal Statement**:
```
∀ actions {a₁, a₂, ..., aₙ} leading to goal G:
  IF i ∈ [max(0, n-10), n] THEN label(aᵢ) = 1
  ELSE label(aᵢ) = 0

VIOLATED INVARIANT:
  The labeling conflates temporal proximity with causal contribution.
  Credit is uniformly distributed, ignoring the gradient of influence.
```

**Mathematical Expression**:
$$
L(a_i) = \mathbb{1}[\exists \, g \in \{i+1, \ldots, i+10\} : \text{goal}(g)]
$$

This indicator function assigns **binary credit** without weighting:
$$
\text{Expected:} \quad L(a_i) \propto \frac{\partial P(\text{goal})}{\partial a_i}
$$
$$
\text{Actual:} \quad L(a_i) \in \{0, 1\} \quad \text{(no gradient)}
$$

The system **cannot distinguish** between:
- A pass that directly creates the goal-scoring opportunity
- A safe back-pass 8 actions earlier

---

### WS3: Fixed Probability Injection

**Classification**: Exogenous Probability Injection

**Implicit Invariant**:
> The system assumes that **all probabilities are learned from data**, creating a unified valuation framework.

**Violation Mechanism**:
The formula in `formula.py:62-67` **overrides** learned probabilities with hardcoded constants:
```python
# fixed odds of scoring when penalty
penalty_idx = actions.type_name == "shot_penalty"
prev_scores[penalty_idx] = 0.792453

# fixed odds of scoring when corner
corner_idx = actions.type_name.isin(["corner_crossed", "corner_short"])
prev_scores[corner_idx] = 0.046500
```

This creates a **two-tier system**:
- Free kicks: valued by classifier $\hat{P}(\text{score} \mid \text{state})$
- Penalties: valued by constant $P_{\text{penalty}} = 0.792453$

**Formal Statement**:
```
∀ actions a:
  IF type(a) = "penalty" THEN P(score|a) := 0.792453 (constant)
  ELIF type(a) = "corner" THEN P(score|a) := 0.046500 (constant)
  ELSE P(score|a) := f_θ(state(a)) (learned)

VIOLATED INVARIANT:
  The model mixes learned and injected probabilities, creating incoherence.
  If the training data has penalty conversion rate ≠ 79.2%, the model produces
  inconsistent valuations.
```

**Mathematical Expression**:
$$
P_{\text{model}}(\text{score} \mid a) = 
\begin{cases}
0.792453 & \text{if } \text{type}(a) = \text{penalty} \\
0.046500 & \text{if } \text{type}(a) \in \{\text{corner\_crossed}, \text{corner\_short}\} \\
\hat{P}_\theta(\text{score} \mid S(a)) & \text{otherwise}
\end{cases}
$$

The discontinuity is **not data-driven**:
- If empirical $P(\text{score} \mid \text{penalty}) = 0.82$ in training data, the model still uses $0.792453$.
- This violates the principle of **evidence-based valuation**.

---

### WS4: Grid Discretization Artifacts (xT)

**Classification**: Discretization-Induced Non-Smoothness

**Implicit Invariant**:
> The system assumes that **spatial value varies smoothly** across the pitch, with no discontinuities at arbitrary boundaries.

**Violation Mechanism**:
The xT grid in `xthreat.py:21-37` quantizes continuous space into 12×16 cells:
```python
M: int = 12  # width cells
N: int = 16  # length cells

def _get_cell_indexes(x, y, l=N, w=M):
    xi = x.divide(field_length).multiply(l).astype('int64').clip(0, l-1)
    yj = y.divide(field_width).multiply(w).astype('int64').clip(0, w-1)
```

This creates **step discontinuities**:
- Position (52.4m, 34.0m) → Cell (7, 6)
- Position (52.6m, 34.0m) → Cell (8, 6)

Despite only 20cm separation, the actions are assigned **different threat values** because they cross a cell boundary.

**Formal Statement**:
```
∀ positions (x, y):
  cell(x, y) = ⌊x / (105/16)⌋ × ⌊y / (68/12)⌋

VIOLATED INVARIANT:
  Threat value is piecewise constant within cells but discontinuous at boundaries.
  Small perturbations in position can cause discrete threat jumps.
```

**Mathematical Expression**:
$$
T(x, y) = \text{threat}[\text{cell}(x, y)] = \text{threat}\left[\left\lfloor \frac{16x}{105} \right\rfloor, \left\lfloor \frac{12y}{68} \right\rfloor\right]
$$

The threat function $T$ is:
- **Piecewise constant** within each cell
- **Discontinuous** at cell boundaries

For positions $p_1 = (52.4, 34.0)$ and $p_2 = (52.6, 34.0)$:
$$
\|p_1 - p_2\| = 0.2 \text{m} \quad \text{but} \quad |T(p_1) - T(p_2)| \text{ may be large}
$$

This violates the **Lipschitz continuity** assumption:
$$
\exists \, L : |T(p_1) - T(p_2)| \leq L \|p_1 - p_2\|
$$

The grid discretization makes $L$ unbounded at cell boundaries.

---

### WS5: Possession Flip Valuation

**Classification**: Attribution Ambiguity

**Implicit Invariant**:
> The system assumes that **value attribution is symmetric** across possession changes: the team gaining possession receives credit, but the team losing possession is **not explicitly penalized**.

**Violation Mechanism**:
The formula in `formula.py:48-49` handles possession flips via probability inversion:
```python
sameteam = _prev(actions.team_id) == actions.team_id
prev_scores = (_prev(scores) * sameteam + _prev(concedes) * (~sameteam))
```

When possession flips (`sameteam = False`):
- Team B's scoring probability inherits Team A's **conceding** probability
- This is **correct** for Team B's perspective
- But it leaves Team A's **lost possession** unpenalized in the valuation

**Formal Statement**:
```
∀ actions aᵢ (Team A), aᵢ₊₁ (Team B) where possession flips:
  V(aᵢ₊₁, Team B) = P_score(aᵢ₊₁) - P_concede(aᵢ, Team A)
  V(aᵢ, Team A) is NOT penalized for susceptibility to turnover

VIOLATED INVARIANT:
  Credit for turnovers is asymmetric:
    - Defensive actions (tackles, interceptions) receive positive value
    - Offensive actions that lose possession are not explicitly penalized
```

**Mathematical Expression**:
For a tackle at action $i+1$ (Team B) following a pass at action $i$ (Team A):

$$
V(a_{i+1}) = P_{\text{score}}(S_{i+1}) - P_{\text{concede}}(S_i)
$$

But the **preceding action** $a_i$ (Team A's pass) is valued as:
$$
V(a_i) = P_{\text{score}}(S_i) - P_{\text{concede}}(S_{i-1})
$$

The fact that $a_i$ **led to a turnover** is not encoded in its value. The model values:
- **What happened** (state transition)
- Not **why it happened** (action vulnerability)

**Causal Attribution Gap**:
```
Observed: Team A passes → Team B tackles → Team B gains possession

Model assigns:
  - Positive value to Team B's tackle (defensive action)
  - Neutral value to Team A's pass (offensive action that lost possession)

Expected: Team A's pass should be penalized for being susceptible to the tackle.
```

This is a **modeling choice**, not an empirical fact. The asymmetry reflects an implicit assumption:
> Turnovers are valued from the **gaining team's** perspective, not the **losing team's** perspective.

---

## Mapping Table: Assumptions → Weird States → Violated Invariants

| Assumption (from ASSUMPTIONS.md) | Weird State | Classification | Violated Invariant |
|----------------------------------|-------------|----------------|-------------------|
| **A1.1**: Temporally Contiguous Sequences (10s threshold) | WS1: Temporal Cliff | Temporal Under-Specification | Temporal proximity ≠ causal continuity |
| **A2.2**: Action Independence (10-action lookahead) | WS2: Credit Ambiguity | Attribution Ambiguity | Temporal proximity ≠ causal contribution |
| **A2.5**: Fixed Set-Piece Probabilities | WS3: Probability Injection | Exogenous Injection | Learned vs. injected probabilities are incoherent |
| **A4.1**: Grid Discretization (12×16 cells) | WS4: Grid Boundaries | Discretization Non-Smoothness | Spatial value is not Lipschitz continuous |
| **A3.2**: Binary Possession Model | WS5: Possession Flip | Attribution Ambiguity | Turnover credit is asymmetric (gain vs. loss) |

---

## Cross-Reference: Classification → Assumptions

### Temporal Under-Specification
- **A1.1**: 10-second temporal continuity threshold
- **Related WS**: WS1 (Temporal Cliff)

**Core Issue**: The system discretizes time into "same phase" vs. "different phase" using an arbitrary constant, without encoding the **semantic reason** for the gap (stoppage, substitution, tactical reset).

---

### Attribution Ambiguity
- **A1.2**: Single-agent action ownership
- **A2.2**: 10-action lookahead window
- **A3.2**: Binary possession model
- **Related WS**: WS2 (Credit Ambiguity), WS5 (Possession Flip)

**Core Issue**: The system assigns credit/blame to **actions** without explicit causal modeling. Multi-agent interactions (deflections, turnovers, assists) are flattened into single-player attribution.

---

### Discretization-Induced Non-Smoothness
- **A1.3**: Discrete action boundaries
- **A1.4**: Fixed spatial coordinates
- **A4.1**: xT grid discretization
- **Related WS**: WS4 (Grid Boundaries)

**Core Issue**: Continuous phenomena (space, time, motion) are quantized into discrete bins. This creates **boundary artifacts** where infinitesimal perturbations cause discrete value jumps.

---

### Exogenous Probability Injection
- **A2.5**: Fixed probabilities for penalties (79.2%) and corners (4.65%)
- **Related WS**: WS3 (Probability Injection)

**Core Issue**: The system mixes **learned** probabilities (from training data) with **injected** constants (from external sources or heuristics). This creates incoherence when training data statistics diverge from the constants.

---

### Stationarity Violation
- **A2.3**: Model assumes stationary distributions
- **A3.1**: "Normal football" assumption
- **Related WS**: (Not directly demonstrated, but implicit in all WS)

**Core Issue**: The model assumes that $P(\text{score} \mid \text{state})$ and $P(\text{concede} \mid \text{state})$ are **time-invariant**. In reality, these probabilities shift based on:
- Scoreline (trailing teams take more risks)
- Time remaining (urgency increases)
- Player fatigue (performance degrades)
- Tactical changes (formation shifts)

This assumption is **not violated by a specific weird state** but underlies the system's brittleness in non-standard contexts.

---

## Formal Invariants: Summary

### Invariant 1: Temporal Smoothness
**Statement**: Value should vary smoothly with respect to temporal separation.

**Violated By**: WS1 (Temporal Cliff)

**Formal**:
$$
\lim_{\epsilon \to 0} V(a \mid t_{prev} = t - \tau - \epsilon) = V(a \mid t_{prev} = t - \tau + \epsilon)
$$

**Actual Behavior**:
$$
V(a \mid \Delta t = 10 - \epsilon) \neq V(a \mid \Delta t = 10 + \epsilon)
$$

---

### Invariant 2: Causal Attribution Gradient
**Statement**: Credit should decay with causal distance from the outcome.

**Violated By**: WS2 (Credit Ambiguity)

**Formal**:
$$
L(a_i) = \sum_{j=i+1}^{i+k} w(j-i) \cdot \mathbb{1}[\text{goal}(a_j)]
$$
where $w(d)$ is a decay function (e.g., $w(d) = e^{-\lambda d}$).

**Actual Behavior**:
$$
L(a_i) = \mathbb{1}\left[\bigvee_{j=1}^{10} \text{goal}(a_{i+j})\right] \quad \text{(binary, no decay)}
$$

---

### Invariant 3: Probability Coherence
**Statement**: All probabilities should be derived from the same source (either all learned or all injected).

**Violated By**: WS3 (Probability Injection)

**Formal**:
$$
\forall \, a : P(a) = f_\theta(\text{data}) \quad \text{(consistent source)}
$$

**Actual Behavior**:
$$
P(a) = 
\begin{cases}
f_\theta(\text{data}) & \text{most actions} \\
c_{\text{penalty}} = 0.792453 & \text{penalties} \\
c_{\text{corner}} = 0.046500 & \text{corners}
\end{cases}
$$

---

### Invariant 4: Spatial Lipschitz Continuity
**Statement**: Threat should be Lipschitz continuous with respect to spatial position.

**Violated By**: WS4 (Grid Boundaries)

**Formal**:
$$
\exists \, L : |T(p_1) - T(p_2)| \leq L \|p_1 - p_2\|
$$

**Actual Behavior**:
$$
T(x, y) = \text{threat}[\text{cell}(x, y)] \quad \text{(piecewise constant)}
$$

At cell boundaries, $L$ is **unbounded**.

---

### Invariant 5: Symmetric Turnover Valuation
**Statement**: Credit for possession changes should be symmetric between gaining and losing teams.

**Violated By**: WS5 (Possession Flip)

**Formal**:
$$
V_{\text{gain}}(\text{tackle}) = -V_{\text{loss}}(\text{lost pass})
$$

**Actual Behavior**:
$$
V_{\text{gain}}(\text{tackle}) > 0, \quad V_{\text{loss}}(\text{lost pass}) \approx 0
$$

The losing team's action is **not explicitly penalized**.

---

## Conclusion: Where Invariants Fail

The socceraction pipeline **implicitly relies** on five semantic invariants:

1. **Temporal Smoothness**: Value varies continuously with time
2. **Causal Gradient**: Credit decays with causal distance
3. **Probability Coherence**: All probabilities share a common source
4. **Spatial Continuity**: Value varies smoothly with position
5. **Attribution Symmetry**: Credit/blame are symmetric across turnovers

Each weird state **violates** one or more of these invariants, not because of bugs, but because of **design decisions**:
- Discretization (temporal, spatial)
- Simplification (binary labels, flat credit)
- Override (injected constants)
- Asymmetry (defensive actions valued, offensive vulnerabilities ignored)

These violations are **not wrong**—they are **trade-offs**. The goal of this document is to make them **explicit and legible**, so that:
- Researchers understand where the model's semantics end
- Practitioners know when to expect brittleness
- Extensions can address specific invariants systematically

**No fixes are proposed**. This is diagnostic analysis, not prescriptive engineering.
