# Experiment Plan — Cross-Layer Expert Sharing & Fault Isolation

**Anchor**: see `CROSSLAYER_PROPOSAL.md`. Claim-driven, inference-only, small models → low compute.
**Compute note**: no GPU in this environment; all blocks are `needs pilot`. MoEUT/MoRE are 114M–1.15B → one modest GPU suffices.

## Setup
- **Shared-layer models**: MoEUT (2405.16039) and MoRE (2609.18176) — experts reused across layers; MoRE has depth embeddings (needed for C4).
- **Layer-isolated controls**: a standard MoE (OLMoE-small / a matched-param layer-isolated MoE) with the same active-param / FLOP budget.
- **Metric of blast radius**: ΔPPL and Δaccuracy (on a held-out eval: WikiText + a small QA set) from corrupting/ablating ONE expert, averaged over experts, as a function of that expert's depth-reuse count r.
- **Perturbation**: (a) hard ablation (zero the expert / remove from pool), (b) soft corruption (add calibrated Gaussian noise to the expert weights) — report both; (b) is the smoother signal.

## Block 0 — reuse-count instrumentation (CPU/light) → prerequisite
For each model, record each expert's depth-reuse count r (how many layers can route to it). For isolated MoE, r=1 for all. For MoEUT/MoRE, r>1 per the sharing topology. Output: per-expert r table. No GPU.

## Block 1 — single-expert blast radius, shared vs isolated (1 GPU) → C1
Ablate one expert at a time; measure ΔPPL/Δacc. Compare distribution of single-expert blast radius: shared-layer vs isolated. **Gate C1**: shared-layer mean/max blast radius significantly > isolated.

## Block 2 — the reuse-count law (1 GPU) → C2
Within the shared-layer model, plot blast radius vs reuse-count r. **Gate C2**: positive monotone slope (Spearman > 0, significant); isolated baseline flat ≈ 1/N. This is the central result.

## Block 3 — redundancy-defense failure (1 GPU) → C3
Replicate 2606.10703's "early-layer redundancy" probe on both: in isolated MoE, early-layer single-expert ablation ≈ harmless (redundant); in shared-layer, the same expert is also late-layer, so ablation hurts. **Gate C3**: early-layer ablation harm_shared ≫ harm_isolated.

## Block 4 — mitigation (1 GPU) → C4
Toggle MoRE's depth embeddings on/off; and impose a reuse cap (limit how many layers may share one expert). Measure blast radius vs these. **Gate C4**: depth-conditioning / reuse cap reduces blast radius toward the isolated baseline.

## Ablation matrix
| Axis | Levels | Purpose |
|------|--------|---------|
| Architecture | MoEUT / MoRE / isolated-MoE | C1 |
| Reuse count r | per-expert, 1..max | C2 (the law) |
| Perturbation | ablation / weight-noise | robustness of the effect |
| Layer position | early / mid / late | C3 |
| Depth-embedding | on / off (MoRE) | C4 |
| Reuse cap | none / capped | C4 |

## Run order
Block 0 (now, CPU) → Block 1 (C1 gate) → **Block 2 (the reuse-count law — the paper's core)** → Block 3 → Block 4. Stop & write negative result at any failed gate.

## Total budget
~1 CPU-day (Block 0) + ~3–5 GPU-days on one modest GPU (small models). Feasible for most labs.
