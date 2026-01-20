# Paper Outline: Semantic Closure in Event-Valued Action Valuation Systems

## Meta

**Genre**: Systems critique / Methodological analysis  
**Claim Type**: Diagnostic (not prescriptive)  
**Empirical Basis**: Existing production system (socceraction/VAEP/xT)  
**Contribution**: Formalization of implicit assumptions and invariant violations

---

## Title Candidates

1. **"Where Semantics End: Invariant Violations in Event-Valued Action Valuation"**
2. **"Semantic Closure Failures in Sports Analytics Valuation Pipelines"**
3. **"From Events to Value: Under-Specification in Probabilistic Action Attribution"**

---

## Abstract Structure

### Paragraph 1: Problem Context
- Event-valued systems (sports analytics, autonomous agents, process mining) transform discrete observations into action valuations
- Standard approach: discretize → featurize → estimate probabilities → compute value
- Implicit assumption: the pipeline preserves **semantic coherence**

### Paragraph 2: Gap
- Existing work validates **predictive accuracy** (does the model predict well?)
- Lacks analysis of **semantic closure** (where do model assumptions break down?)
- Under-specification leads to **technically valid but semantically questionable** outputs

### Paragraph 3: Contribution
- Systematic audit of socceraction (VAEP/xT) as exemplar event-valued system
- Taxonomy of 5 failure modes ("weird states") where assumptions violate invariants
- Formal characterization of violated invariants (temporal smoothness, causal attribution, probability coherence, spatial continuity, turnover symmetry)
- Framework for identifying semantic boundaries in event-valued systems

### Paragraph 4: Implications
- Not about fixing specific bugs, but about **understanding system limits**
- Applicable to any event-to-value pipeline (not sports-specific)
- Enables: robustness testing, calibration detection, model extension guidance

---

## 1. Introduction

### 1.1 Motivation: Event-Valued Systems are Everywhere
**Content from**: AUDIT_README.md (context), ASSUMPTIONS.md (overview)

- **Examples**: Sports analytics (VAEP, xT), process mining (activity valuation), autonomous agents (action selection)
- **Common pattern**: Discrete events → Probabilistic models → Value attribution
- **Challenge**: Where do the semantics of the domain end and the artifacts of the model begin?

### 1.2 The Validation Gap
**Content from**: ASSUMPTIONS.md (introduction), SEMANTIC_GAPS.md (purpose)

- Standard validation: Predictive accuracy, cross-validation, holdout performance
- **Missing**: Semantic coherence under perturbation
- **Question**: What happens at the boundaries of the model's assumptions?

### 1.3 Our Approach: Audit as Methodology
**Content from**: AUDIT_README.md, SEMANTIC_GAPS.md (classification taxonomy)

- Select a production-grade system (socceraction) as exemplar
- Not a critique of the system, but **using it as a lens** to study event-valued modeling
- Systematic identification of "weird states": technically valid, semantically questionable
- Formalization of violated invariants

### 1.4 Contributions
1. **Taxonomy** of semantic failure modes in event-valued systems (5 classes)
2. **Formalization** of implicit invariants and their violations (mathematical/logical statements)
3. **Mapping** from assumptions → weird states → violated invariants
4. **Framework** for semantic boundary analysis (generalizable beyond sports)

### 1.5 Non-Contributions (Scope Limitations)
- Not proposing fixes or improved models
- Not claiming bugs in socceraction (the system works as designed)
- Not evaluating predictive performance
- **Goal**: Make failure modes legible, not solve them

---

## 2. Background: Event-Valued Action Valuation

### 2.1 Problem Formulation
**Content from**: ASSUMPTIONS.md (overview diagram)

```
Raw event stream → Representation → Features → Probabilities → Value
```

- **Input**: Sequence of discrete events $e_1, e_2, \ldots, e_n$ with attributes (time, type, location, agent)
- **Output**: Value $V(e_i)$ representing contribution to outcome (e.g., goal probability)
- **Mechanism**: Probabilistic classifiers estimate $P(\text{outcome} \mid \text{state})$

### 2.2 VAEP/xT as Exemplar Systems
**Content from**: ASSUMPTIONS.md (A1, A2 sections), weird_states.ipynb (introduction)

- **VAEP**: Valuing Actions by Estimating Probabilities (Decroos et al., 2019)
  - State: Current action + $k$ previous actions
  - Labels: Binary (goal scored/conceded within $n$ actions)
  - Value: $\Delta P_{\text{score}} - \Delta P_{\text{concede}}$

- **xT**: Expected Threat (grid-based possession value)
  - Discretize pitch into $M \times N$ grid
  - Estimate transition probabilities and terminal scoring probabilities
  - Value: Expected threat from each cell

### 2.3 Why These Systems?
- **Production-grade**: Used by professional teams and analysts
- **Open-source**: Full pipeline inspectable
- **Representative**: Design patterns common across event-valued systems
- **Well-documented**: Enables precise attribution of assumptions to code

---

## 3. Methodology: Semantic Boundary Analysis

### 3.1 Audit Framework
**Content from**: AUDIT_README.md, SEMANTIC_GAPS.md (purpose)

**Step 1**: Identify implicit assumptions  
**Step 2**: Construct minimal "weird states" that isolate each assumption  
**Step 3**: Classify by semantic failure mode  
**Step 4**: Formalize violated invariants  

**Not**: Fuzzing, adversarial testing, or bug-finding  
**Is**: Systematic exploration of semantic boundaries

### 3.2 Weird State Construction
**Content from**: weird_states.ipynb (utility functions, methodology)

- Use **synthetic SPADL data** (not real matches) to isolate specific conditions
- Minimal perturbations: Change one variable, hold others constant
- Compare: semantically similar inputs → observe value divergence

### 3.3 Invariant Formalization
**Content from**: SEMANTIC_GAPS.md (formal invariants section)

- Express expected behavior as mathematical/logical statements
- Show violation through counterexamples
- Categories: Continuity, monotonicity, coherence, symmetry

---

## 4. Taxonomy of Semantic Failures

**Content from**: SEMANTIC_GAPS.md (classification taxonomy, mapping table)

### 4.1 Temporal Under-Specification
**Weird State**: WS1 (Temporal Cliff)  
**Invariant**: Value varies smoothly with temporal separation  
**Violation**: Hard threshold at 10 seconds creates discontinuity  
**Formal**: $\lim_{\epsilon \to 0} V(\Delta t = \tau - \epsilon) \neq V(\Delta t = \tau + \epsilon)$

### 4.2 Attribution Ambiguity
**Weird States**: WS2 (Credit Assignment), WS5 (Possession Flip)  
**Invariant**: Credit decays with causal distance; Turnover attribution is symmetric  
**Violation**: Binary labels ignore contribution gradient; Asymmetric turnover credit  
**Formal**: $L(a_i) = \mathbb{1}[\text{outcome}]$ (no decay) vs. expected $L(a_i) \propto e^{-\lambda d}$

### 4.3 Discretization-Induced Non-Smoothness
**Weird State**: WS4 (Grid Boundaries)  
**Invariant**: Spatial value is Lipschitz continuous  
**Violation**: Grid quantization creates unbounded gradients at boundaries  
**Formal**: $\exists L : |T(p_1) - T(p_2)| \leq L \|p_1 - p_2\|$ violated at cell edges

### 4.4 Exogenous Probability Injection
**Weird State**: WS3 (Set-Piece Constants)  
**Invariant**: All probabilities from unified source (learned or constant)  
**Violation**: Penalties/corners use hardcoded constants, other actions use classifier  
**Formal**: Mixed $P(a) = f_\theta(\text{data})$ and $P(a) = c$ in same framework

### 4.5 Stationarity Violation
**Implicit across all weird states**  
**Invariant**: Distributions invariant across match contexts  
**Violation**: Scoreline, time, red cards shift distributions  
**Formal**: $P(\text{outcome} \mid \text{state}, \text{context}_1) \neq P(\text{outcome} \mid \text{state}, \text{context}_2)$

---

## 5. Formal Characterization of Invariant Violations

**Content from**: SEMANTIC_GAPS.md (Formal Invariants: Summary)

### 5.1 Continuity Violations

**Temporal Smoothness**  
Expected: $V$ is continuous in temporal separation  
Actual: $V$ has jump discontinuity at $\Delta t = 10$

**Spatial Smoothness**  
Expected: $T$ is Lipschitz continuous in position  
Actual: $T$ is piecewise constant with unbounded $L$ at grid boundaries

### 5.2 Attribution Violations

**Causal Gradient**  
Expected: $L(a_i) = \sum_{j} w(j-i) \cdot \mathbb{1}[\text{goal}(a_j)]$ with decay $w$  
Actual: $L(a_i) = \bigvee_{j=1}^{10} \text{goal}(a_{i+j})$ (flat)

**Turnover Symmetry**  
Expected: $V_{\text{gain}} = -V_{\text{loss}}$  
Actual: $V_{\text{gain}} > 0$, $V_{\text{loss}} \approx 0$

### 5.3 Coherence Violations

**Probability Source Coherence**  
Expected: All $P(\cdot)$ from same source  
Actual: Mixed learned ($f_\theta$) and injected ($c$) probabilities

**Distributional Stationarity**  
Expected: $P(\text{outcome} \mid \text{state})$ context-independent  
Actual: Implicit assumption, violated in extreme contexts

---

## 6. Implications for Event-Valued Systems

**Content from**: ASSUMPTIONS.md (summary table), SEMANTIC_GAPS.md (conclusion)

### 6.1 Semantic Closure is Not Guaranteed
- Event-valued systems **do not inherently preserve** semantic coherence
- Design choices (discretization, windows, constants) create artifacts
- **Claim**: These are not bugs, but **necessary trade-offs** that have **unformalized boundaries**

### 6.2 Where Semantics End
**Mapping**: Assumptions → Failure Modes

| Assumption | Boundary |
|------------|----------|
| Temporal continuity | Stoppage > 10s erases context |
| Causal attribution | 10-action window flattens credit |
| Learned probabilities | Set pieces use constants |
| Spatial continuity | Grid boundaries create jumps |
| Stationary distributions | Extreme contexts shift behavior |

### 6.3 Detection vs. Correction
**Not proposing**: Fixes, improved models, better hyperparameters  
**Proposing**: Framework for **detecting** when assumptions are violated

- **Invariant tests**: Perturb inputs, check for expected properties (smoothness, symmetry)
- **Calibration flags**: Log when hardcoded constants diverge from data
- **Boundary monitoring**: Track distribution shifts in production

### 6.4 Generalizability Beyond Sports
**Claim**: This analysis applies to **any** event-to-value pipeline

- **Process mining**: Activity valuation in business processes
- **Autonomous agents**: Action selection under uncertainty
- **Credit assignment**: Multi-agent systems, Markov decision processes
- **Causal inference**: Event attribution in observational data

**Pattern**: Discretize continuous phenomena → Estimate probabilities → Compute value  
**Challenge**: Where do model assumptions create semantic boundaries?

---

## 7. Related Work

### 7.1 Sports Analytics Valuation
- **VAEP** (Decroos et al., 2019): Original framework
- **xT** (Karun Singh, 2019): Grid-based threat model
- **g+** (Eggels et al., 2016): Generalized expected goals
- **Our position**: Not critiquing these models, but using them to study **general properties** of event-valued systems

### 7.2 Action Valuation in RL/MDP
- Credit assignment (Sutton & Barto, 2018)
- Off-policy evaluation (Precup et al., 2000)
- **Gap**: Focus on convergence/optimality, not semantic coherence under perturbation

### 7.3 Interpretability and Robustness
- Model interpretability (Lipton, 2016; Doshi-Velez & Kim, 2017)
- Adversarial robustness (Goodfellow et al., 2014)
- **Gap**: Focus on **predictive** failures, not **semantic** under-specification

### 7.4 Causal Inference
- Counterfactual reasoning (Pearl, 2009)
- Attribution methods (Lundberg & Lee, 2017)
- **Gap**: Assume causal structure is known; we study **implicit** assumptions in event pipelines

**Positioning**: Our work bridges **systems analysis** (how does it work?) and **semantic analysis** (where does meaning break down?)

---

## 8. Limitations and Future Work

### 8.1 Scope Limitations
- **Single system**: Analysis based on socceraction; generalization claims require validation on other systems
- **Synthetic data**: Weird states constructed, not discovered in production data
- **No user study**: Semantic "questionability" is analytical, not empirically validated with practitioners

### 8.2 Non-Goals (By Design)
- Not proposing improved models (diagnostic, not prescriptive)
- Not evaluating predictive performance (semantic, not statistical)
- Not claiming bugs (design trade-offs, not errors)

### 8.3 Future Directions
- **Empirical validation**: Apply framework to other event-valued systems (process mining, autonomous agents)
- **User studies**: Do practitioners agree that weird states are "semantically questionable"?
- **Automated detection**: Tools for invariant violation testing in production pipelines
- **Formal verification**: Can semantic closure be proven for restricted classes of systems?

---

## 9. Conclusion

### 9.1 Summary of Contributions
1. **Taxonomy** of 5 semantic failure modes in event-valued systems
2. **Formalization** of violated invariants with mathematical/logical statements
3. **Framework** for semantic boundary analysis (audit methodology)
4. **Empirical demonstration** using production system (socceraction)

### 9.2 Core Claim (Minimum Defensible Novelty)
> **Event-valued systems lack semantic closure guarantees**: Design choices (discretization, windowing, constant injection) create boundaries where model assumptions break down, producing outputs that are technically valid but semantically under-specified.

### 9.3 Takeaway for Practitioners
- Validate **not just accuracy**, but **semantic coherence**
- Document **where assumptions end** (temporal boundaries, attribution limits, context dependencies)
- Use invariant violations as **test cases** for robustness

### 9.4 Takeaway for Researchers
- **Diagnostic analysis** is valuable even without prescriptive fixes
- **Formalization** of implicit assumptions enables rigorous boundary testing
- **Generalizable patterns** across event-valued systems warrant further study

---

## Positioning & Novelty

### Minimum Defensible Novelty Claim

**Not**: "We found bugs in socceraction"  
**Not**: "We propose a better model"  
**Not**: "VAEP/xT are wrong"

**Is**: 
> **"Event-valued action valuation systems contain implicit assumptions that create semantic boundaries. These boundaries are not bugs—they are necessary trade-offs—but they are under-formalized. We provide a systematic framework for identifying and formalizing these boundaries, using socceraction as an exemplar."**

**Stronger claim** (if defensible):
> **"Event-to-value pipelines lack semantic closure guarantees: there exist inputs where the model's outputs are technically valid (satisfy schema, pass tests) but semantically under-specified (violate implicit invariants). We formalize this gap through invariant violations."**

### What Makes This Publishable

1. **Generalizability**: Not sports-specific; applies to any event-valued system
2. **Formalization**: Mathematical/logical statements of violated invariants (not just prose)
3. **Methodology**: Replicable audit framework (can be applied to other systems)
4. **Non-obvious**: Semantic closure failures are distinct from predictive failures
5. **Actionable**: Enables robustness testing, calibration, boundary monitoring

### What This Is NOT

- Not a new model or algorithm
- Not a performance comparison
- Not a user study or empirical evaluation (of real-world impact)
- Not a survey or taxonomy of existing work

**Genre**: Methodological contribution + Systems critique

---

## Suggested Venues

### Tier 1: Workshop (Fastest Path to Feedback)

1. **AAAI Workshop on AI for Sports Analytics**
   - **Fit**: Sports domain, methodological contribution
   - **Audience**: Practitioners + researchers familiar with VAEP/xT
   - **Length**: 6-8 pages
   - **Timeline**: Workshops typically 3-4 months before main conference

2. **NeurIPS Workshop on Causal Inference & Machine Learning**
   - **Fit**: Causal attribution, credit assignment
   - **Audience**: ML researchers interested in interpretability
   - **Length**: 6-8 pages
   - **Timeline**: October/November submission

3. **ICML Workshop on Uncertainty & Robustness in Deep Learning**
   - **Fit**: Semantic robustness, invariant violations
   - **Audience**: ML theorists + practitioners
   - **Length**: 6-8 pages
   - **Timeline**: June/July submission

### Tier 2: Conference (Higher Bar, More Visibility)

1. **AAAI Conference (Main Track: AI & Society / Applications)**
   - **Fit**: Methodological contribution with real-world application
   - **Length**: 7-9 pages
   - **Timeline**: August submission for February conference
   - **Note**: Would need to strengthen generalizability claims beyond sports

2. **AIES (AI, Ethics, and Society)**
   - **Fit**: Semantic coherence as a transparency/accountability issue
   - **Angle**: Event-valued systems in high-stakes domains (sports, hiring, loan approval)
   - **Length**: 8-10 pages
   - **Timeline**: January submission for May/June conference

3. **ICSE Workshop on Software Engineering for Machine Learning (SEML)**
   - **Fit**: Testing, robustness, semantic validation
   - **Angle**: Engineering perspective on ML system boundaries
   - **Length**: 6-8 pages
   - **Timeline**: November submission for May conference

### Tier 3: ArXiv + Journal (Long-Form)

1. **arXiv Category**: cs.LG (Machine Learning) + stat.ML (Statistics)
   - **Tags**: causal inference, sports analytics, interpretability
   - **Advantage**: No length limit; can include full formalization
   - **Strategy**: Submit to arXiv first, then target journal

2. **Journal of Artificial Intelligence Research (JAIR)**
   - **Fit**: Methodological contributions, formal analysis
   - **Length**: 20-40 pages
   - **Timeline**: Rolling submissions, 6-12 month review

3. **Transactions on Machine Learning Research (TMLR)**
   - **Fit**: ML methodology, systems analysis
   - **Length**: No strict limit
   - **Timeline**: Rolling submissions, faster than JAIR

---

## Recommended Strategy

### Phase 1: Workshop Submission (Near-term)
**Target**: AAAI Workshop on AI for Sports Analytics (if available) or NeurIPS Causal Inference workshop

**Why**:
- Sports audience will appreciate socceraction exemplar
- 6-8 pages forces focused contribution
- Workshop feedback before conference submission

**Structure**: Sections 1-6 (trim Section 7, expand Section 3)

### Phase 2: ArXiv Preprint
**Timing**: Simultaneously with or shortly after workshop submission

**Why**:
- Establishes timestamp
- Full formalization (no page limits)
- Enables early citation

**Structure**: Full outline (Sections 1-9)

### Phase 3: Conference or Journal (Long-term)
**Target**: AAAI main track or JAIR (depending on workshop feedback)

**Why**:
- Conference if generalizability claims are strong
- Journal if formalization is the main contribution

**Structure**: Refined based on workshop feedback

---

## Key Positioning Messages

### For Sports Analytics Audience
- "We use socceraction as a lens to study **general properties** of event-valued systems"
- "Not a critique of VAEP/xT, but a **methodological contribution** using them as exemplars"
- "Applicable to **any** sports analytics pipeline (not just soccer)"

### For ML Audience
- "Semantic coherence is **distinct from** predictive accuracy"
- "Event-to-value pipelines are **common** (not just sports): process mining, autonomous agents, causal inference"
- "Invariant violations provide **test cases** for robustness"

### For Systems Audience
- "Design trade-offs create **semantic boundaries** that are under-formalized"
- "Audit methodology is **replicable** and **generalizable**"
- "Detection framework enables **production monitoring** of assumption violations"

---

## Appendix: Artifact Mapping

| Paper Section | Primary Source | Secondary Source |
|---------------|----------------|------------------|
| 1. Introduction | AUDIT_README.md | ASSUMPTIONS.md (overview) |
| 2. Background | ASSUMPTIONS.md (A1, A2) | weird_states.ipynb (intro) |
| 3. Methodology | weird_states.ipynb | SEMANTIC_GAPS.md (purpose) |
| 4. Taxonomy | SEMANTIC_GAPS.md (classifications) | weird_states.ipynb (examples) |
| 5. Formalization | SEMANTIC_GAPS.md (formal invariants) | ASSUMPTIONS.md (summary table) |
| 6. Implications | ASSUMPTIONS.md (conclusion) | SEMANTIC_GAPS.md (conclusion) |
| 7. Related Work | (new content) | - |
| 8. Limitations | AUDIT_README.md (scope) | - |
| 9. Conclusion | SEMANTIC_GAPS.md (conclusion) | ASSUMPTIONS.md (summary) |

**Note**: Existing artifacts contain **all necessary content**. Paper writing is **reorganization + positioning**, not new analysis.
