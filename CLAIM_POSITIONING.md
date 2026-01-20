# Claim Positioning: Preempting Reviewer Criticisms

## Core Novelty Claim

### One Sentence
Event-to-value pipelines contain under-formalized semantic boundaries where design choices produce technically valid but semantically under-specified outputs, and we provide a systematic framework for identifying and formalizing these boundaries through invariant violations.

### One Paragraph
Event-valued systems—pipelines that transform discrete observations (events) into action valuations—are deployed across domains (sports analytics, process mining, autonomous agents), yet lack formal guarantees of semantic closure. We demonstrate that necessary design choices (temporal discretization, spatial quantization, probability injection, credit windowing) create boundaries where model outputs satisfy technical validity constraints (schema compliance, type correctness) but violate implicit semantic invariants (continuity, attribution symmetry, probability coherence). Using socceraction (VAEP/xT) as an exemplar production system, we provide: (1) a taxonomy of five semantic failure modes, (2) formal characterization of violated invariants through mathematical/logical statements, and (3) a replicable audit methodology for boundary identification. This is not bug-finding or model improvement, but formalization of the gap between technical validity and semantic coherence—a gap that exists in any event-to-value pipeline.

---

## What This Paper Explicitly Does NOT Claim

### Technical Scope
1. **NOT claiming bugs or errors**: The systems work as designed; violations are consequences of necessary trade-offs, not implementation mistakes
2. **NOT proposing fixes or improved models**: This is diagnostic analysis, not prescriptive engineering
3. **NOT evaluating predictive performance**: Semantic coherence is orthogonal to statistical accuracy
4. **NOT discovering vulnerabilities**: Weird states are constructed to isolate assumptions, not found through adversarial testing

### Methodological Scope
5. **NOT validating with user studies**: Semantic "questionability" is analytical (formal invariant violations), not empirically measured practitioner consensus
6. **NOT generalizing from single system**: Claims about event-valued systems require validation beyond socceraction (acknowledged limitation)
7. **NOT comparing multiple models**: This is systems critique of a design pattern, not comparative evaluation

### Contribution Scope
8. **NOT a survey or taxonomy of prior work**: Related work contextualizes, but contribution is formalization, not literature synthesis
9. **NOT proposing new theory**: Invariants (continuity, symmetry, coherence) are standard; novelty is application to event-valued pipelines
10. **NOT domain-specific insights**: Sports context is illustrative; framework applies wherever events → probabilities → value

---

## Anticipated Reviewer Criticisms

### Criticism 1: "This is just bug finding"
**Anticipated Form**: 
> "The authors identify edge cases and discontinuities in existing systems. This is software testing, not research. What is the scientific contribution?"

**Root Concern**: 
Conflating semantic under-specification with implementation errors.

**Why This Misreads the Contribution**:
- Bugs are **violations of intended behavior** (correctness failures)
- Semantic under-specification is **intended behavior with under-formalized boundaries** (coherence gaps)
- We do not claim the systems are broken; we formalize **where their semantics end**

---

### Criticism 2: "This is domain-specific (sports only)"
**Anticipated Form**:
> "The analysis focuses on soccer analytics (VAEP/xT). Findings may not generalize to other domains. Why should ML researchers care about sports-specific issues?"

**Root Concern**:
Generalizability beyond soccer.

**Why This Misreads the Contribution**:
- Soccer is the **empirical context**, not the **conceptual domain**
- The pattern (events → probabilities → value) is **ubiquitous**: process mining, RL credit assignment, causal attribution
- Socceraction is an **exemplar**, not the **subject** of study

---

### Criticism 3: "This lacks prescriptive value"
**Anticipated Form**:
> "The paper identifies problems but offers no solutions. Diagnostic analysis without fixes is incomplete. What should practitioners do?"

**Root Concern**:
Actionability without model improvements.

**Why This Misreads the Contribution**:
- **Diagnostic analysis is valuable** even without prescriptive fixes (precedent: software testing theory, compiler verification)
- Formalization enables **detection**, which is a prerequisite for correction
- Practitioners gain: (1) invariant tests for robustness, (2) boundary documentation for transparency, (3) calibration monitoring for production systems

---

## Preemptive Phrasing

### Preempting "This is just bug finding"

**In Abstract**:
> We do not claim errors or bugs—the systems function as designed. Rather, we formalize the gap between **technical validity** (outputs satisfy schema constraints) and **semantic coherence** (outputs preserve domain invariants). This gap is inherent to event-valued pipelines, not specific to implementation choices.

**In Introduction**:
> Our goal is not to identify correctness failures (bugs), but to **formalize semantic boundaries**—the limits of what the model's assumptions can guarantee. These boundaries are consequences of necessary design trade-offs (discretization enables computation, windowing enables tractability), not mistakes.

**In Methodology**:
> Weird states are **schematic demonstrations**, not adversarial examples. They isolate specific assumptions to test invariants, analogous to unit tests for semantic properties rather than fuzzing for crashes.

**In Limitations**:
> This is **not bug-finding**. We do not claim the systems produce incorrect outputs. We claim they produce outputs whose **semantic interpretation depends on unstated assumptions**, and we make those assumptions explicit through formalization.

**Alternative Framing** (if reviewer persists):
> Consider compiler verification: identifying that a compiler produces technically valid but semantically different code (e.g., due to optimization) is not bug-finding—it is formalization of transformation semantics. Similarly, we formalize event-to-value transformations.

---

### Preempting "This is domain-specific"

**In Abstract**:
> Using socceraction (sports analytics) as an **exemplar**, we study event-valued systems—a class of pipelines that transform discrete observations into action valuations. The pattern (discretize → featurize → estimate probabilities → compute value) appears in process mining (activity valuation), autonomous agents (action selection), and causal inference (event attribution).

**In Introduction**:
> Event-valued systems are **not sports-specific**. They arise wherever:
> 1. Observations are discrete events (not continuous signals)
> 2. Actions are attributed value via probabilistic models
> 3. Credit assignment spans multiple events
>
> Examples: business process mining (which activities add value?), multi-agent RL (credit assignment across agents), counterfactual reasoning (which intervention caused the outcome?).

**In Background**:
> We study socceraction because it is:
> 1. **Production-grade** (used by professional teams)
> 2. **Open-source** (full pipeline inspectable)
> 3. **Well-documented** (design choices traceable to code)
> 4. **Representative** of event-valued patterns (discretization, windowing, probability estimation)
>
> Our framework is **agnostic to domain**: it applies wherever event streams are transformed into valuations.

**In Implications**:
> The taxonomy (temporal under-specification, attribution ambiguity, discretization non-smoothness, exogenous injection, stationarity violation) is **not soccer-specific**. Each class corresponds to a general design pattern:
> - **Temporal**: Windowing/thresholding in any time-series event system
> - **Attribution**: Credit assignment in any multi-event outcome
> - **Discretization**: Quantization in any spatial/continuous-to-discrete mapping
> - **Injection**: Mixing learned and hardcoded parameters in any hybrid system
> - **Stationarity**: Context-invariant assumptions in any non-stationary environment

**In Related Work**:
> While soccer provides the empirical context, our contribution is **methodological**. We bridge:
> - **Systems analysis** (how does the pipeline work?)
> - **Semantic analysis** (where do assumptions create boundaries?)
>
> This bridge is relevant to **any event-valued system**, not just sports.

**Alternative Framing** (if reviewer persists):
> If a reviewer claims "this is sports-only," ask: which part of the taxonomy (temporal discretization, credit windowing, probability coherence, spatial quantization, stationarity assumptions) is unique to soccer? Each pattern appears in non-sports domains. Soccer is the **vehicle**, not the **destination**.

---

### Preempting "This lacks prescriptive value"

**In Abstract**:
> Our contribution is **diagnostic, not prescriptive**: we formalize semantic boundaries, enabling practitioners to (1) test for invariant violations, (2) document assumption limits, and (3) monitor boundary conditions in production. Detection is a prerequisite for correction.

**In Introduction**:
> We do **not** propose improved models. Instead, we provide:
> 1. **Invariant tests**: Perturbation-based checks for semantic coherence (analogous to unit tests)
> 2. **Boundary documentation**: Explicit statements of where assumptions hold vs. fail
> 3. **Calibration monitoring**: Production-time detection of distributional shifts
>
> This is valuable even without fixes—**understanding system limits** is distinct from **extending system capabilities**.

**In Methodology**:
> The audit framework is **replicable**: apply to any event-valued system to identify semantic boundaries. This enables:
> - **Robustness testing**: Check if perturbations violate invariants
> - **Transparency**: Document assumptions for stakeholders
> - **Model extension guidance**: Target specific failure modes (e.g., add context features to address stationarity violations)

**In Implications**:
> Prescriptive value comes in three forms:
> 1. **For practitioners**: Use violated invariants as test cases (e.g., perturb timestamps near 10-second threshold, check value stability)
> 2. **For researchers**: Target specific boundary classes (e.g., replace temporal thresholds with smooth decay functions)
> 3. **For system designers**: Document assumptions explicitly (e.g., "this model assumes stationary distributions; performance degrades under scoreline shifts")

**In Limitations**:
> We **intentionally do not propose fixes** because:
> 1. Fixes are context-dependent (right threshold for one domain ≠ right threshold for another)
> 2. Some violations are **unavoidable** (discretization always loses information)
> 3. Diagnostic analysis is **independently valuable** (precedent: Szegedy et al. 2013 on adversarial examples formalized robustness gaps before defenses existed)

**Alternative Framing** (if reviewer persists):
> Prescriptive value is not synonymous with "proposing a new model." Consider:
> - **Compiler verification** (formalizes transformation semantics without building new compilers)
> - **Adversarial robustness** (Goodfellow et al. 2014 formalized gradient-based attacks before defenses)
> - **Fairness audits** (identify bias without proposing debiasing algorithms)
>
> In each case, **formalization precedes and enables correction**. We provide the formalization.

---

## Defensive Positioning: One-Sentence Rebuttals

### If Reviewer Says: "This is bug-finding"
**Response**: 
> We formalize semantic boundaries (where model assumptions end), not correctness failures (where implementation deviates from specification).

### If Reviewer Says: "This is sports-specific"
**Response**: 
> Sports is the empirical context; the taxonomy (temporal discretization, credit windowing, probability coherence, spatial quantization, stationarity) applies to any event-to-value pipeline (process mining, RL, causal inference).

### If Reviewer Says: "This lacks prescriptive value"
**Response**: 
> Diagnostic formalization is independently valuable—it enables invariant testing, boundary documentation, and calibration monitoring, which are prerequisites for targeted model improvements.

### If Reviewer Says: "The claims are obvious"
**Response**: 
> If semantic boundaries were obvious, they would be documented in system specifications; our contribution is making implicit assumptions explicit through formal invariant statements.

### If Reviewer Says: "The evaluation is weak (no user study)"
**Response**: 
> Semantic questionability is analytical (formal invariant violations), not empirical (practitioner consensus); user validation would test whether violations matter in practice, not whether they exist.

### If Reviewer Says: "Single system is insufficient"
**Response**: 
> Socceraction is an exemplar for deep analysis; generalization claims are testable future work (acknowledged in limitations), but the framework's applicability is demonstrated through taxonomy mapping to non-sports domains.

---

## Framing Strength Gradient

### Weakest Claim (Too Defensive)
> "We found some edge cases in a sports analytics system that might be interesting."

**Why Weak**: 
Frames as narrow, exploratory, sports-specific.

---

### Moderate Claim (Safe but Uninspiring)
> "We analyze socceraction to identify assumptions that create semantic boundaries in event-valued systems."

**Why Moderate**: 
Accurate but lacks **urgency** (why does this matter?) and **generalizability** (why beyond sports?).

---

### Strong Claim (Defensible)
> "Event-valued pipelines lack semantic closure guarantees. We formalize this gap through invariant violations, using socceraction as an exemplar."

**Why Strong**: 
- **General pattern** (event-valued pipelines, not soccer)
- **Formal contribution** (invariant violations, not prose descriptions)
- **Actionable** (formalization enables detection)
- **Non-obvious** (semantic closure ≠ predictive accuracy)

---

### Strongest Claim (Aggressive, Requires Evidence)
> "Event-to-value transformations are fundamentally under-specified. We prove that no finite set of local constraints can guarantee global semantic coherence, and provide a decidability framework for invariant checking."

**Why Too Strong**: 
Requires formal proof, not just empirical demonstration. Save for journal version if theoretical results emerge.

---

## Recommended Positioning

### For Workshop Submission
**Claim**: 
> We formalize semantic boundaries in event-valued systems through invariant violations, using socceraction as an exemplar. This enables detection-based robustness testing.

**Length**: 
6-8 pages (compact, focused on taxonomy + formalization).

**Audience**: 
Sports analytics practitioners + ML researchers interested in interpretability.

---

### For Conference Submission
**Claim**: 
> Event-valued pipelines lack semantic closure guarantees: design choices produce technically valid but semantically under-specified outputs. We provide a systematic framework for formalizing these boundaries.

**Length**: 
8-10 pages (add: related work depth, generalization discussion, future work).

**Audience**: 
ML researchers studying robustness, interpretability, or systems analysis.

---

### For Journal Submission
**Claim**: 
> We introduce semantic boundary analysis for event-valued systems—a methodology for identifying and formalizing the gap between technical validity and semantic coherence. We demonstrate applicability through socceraction (sports), with extensions to process mining and autonomous agents.

**Length**: 
20-30 pages (add: multiple exemplars, user study, theoretical foundations).

**Audience**: 
Broad ML/AI audience + systems researchers.

---

## Summary: Boundaries to Sharpen

| Boundary | Sharpening |
|----------|------------|
| **Bugs vs. Boundaries** | Not correctness failures, but formalized semantic limits |
| **Sports vs. General** | Soccer is exemplar; taxonomy applies to any event-valued pipeline |
| **Diagnostic vs. Prescriptive** | Detection is valuable independently; formalization enables correction |
| **Obvious vs. Novel** | If assumptions were explicit, they'd be documented; we formalize implicit invariants |
| **Single vs. Multiple Systems** | Deep analysis of one system > shallow analysis of many; generalization is testable |
| **Analytical vs. Empirical** | Invariant violations are formal, not opinion-based; user studies test impact, not existence |

**Core Defense**: 
This is **methodological formalization**, not empirical discovery or prescriptive engineering. The value is in making implicit assumptions explicit through rigorous boundary analysis.
