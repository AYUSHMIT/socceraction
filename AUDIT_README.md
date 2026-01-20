# Assumptions Audit & Weird States Analysis

This directory contains a research-oriented audit of the socceraction pipeline's implicit assumptions and edge cases.

## Files

### `ASSUMPTIONS.md`
Comprehensive documentation of assumptions embedded in the socceraction pipeline across three levels:

- **A1: Representation-Level Assumptions** - How SPADL discretizes continuous football into actions
- **A2: Model-Level Assumptions** - How VAEP/xT assume stationarity, independence, and value decomposition
- **A3: Contextual Assumptions** - What contexts the system assumes vs. ignores (scoreline, red cards, etc.)

Each assumption includes:
- Explicit statement
- Code reference (file + function + line number)
- Failure mode analysis
- Potential invariants for detection

### `weird_states.ipynb`
Jupyter notebook demonstrating 5 "weird states" where the system produces technically valid but semantically questionable outputs:

1. **Temporal Discontinuity Cliff** - 10-second threshold creates arbitrary value jumps
2. **Credit Assignment Ambiguity** - Equal labels for unequal contributions
3. **Fixed Probability Injection** - Hardcoded penalties/corners create incoherence
4. **Grid Discretization Artifacts** - xT's 12×16 grid creates boundary effects
5. **Possession Flip Valuation** - Asymmetric credit for turnovers

Each state includes:
- Minimal synthetic examples
- Code demonstrating the behavior
- Analysis of why it occurs
- Discussion of semantic implications

## Purpose

This is **not** documentation or a tutorial. It is a **systems critique** designed to:

1. Identify where model assumptions end and football semantics begin
2. Expose brittleness at boundary conditions
3. Provide a foundation for:
   - Robustness testing
   - Model extensions (adding contextual features)
   - Calibration methods (detecting assumption violations)

## Intended Audience

- Researchers studying action valuation methodologies
- Practitioners extending socceraction for production use
- Methodologists interested in modeling brittleness

## Usage

Read `ASSUMPTIONS.md` first to understand the full scope of implicit assumptions.

Then explore `weird_states.ipynb` to see concrete examples of where these assumptions create edge cases.

The notebook is designed to be executed—it uses synthetic data to isolate specific failure modes without requiring external datasets.

## Technical Notes

- The notebook imports require `socceraction` to be installed: `pip install -e .`
- Visualization cells require `matplotlib`
- All examples use synthetic SPADL-compliant data, not real matches
- The analysis focuses on **structural properties** of the model, not prediction accuracy

## Citation

If you use this analysis in research, please cite both:

1. The original socceraction framework (Decroos et al., 2019)
2. This audit as supplementary methodological critique

## License

Same as socceraction: MIT
