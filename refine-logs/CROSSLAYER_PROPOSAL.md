# Final Proposal — Does Cross-Layer Expert Sharing Collapse MoE Fault Isolation?

**Date**: 2026-10-02 · **Source**: `/idea-discovery "MoE security attack surface"` → vertical / cross-layer-sharing structural reframe
**Framing**: robustness / fault-isolation science (measurement-driven), not an attack playbook.
**Refinement note**: `/research-refine`'s cross-model loop did not run (no non-Claude reviewer in this env). Executor-only, UNADJUDICATED. Problem Anchor is frozen; the rest is a first draft for human / reviewer iteration.

## Problem Anchor (frozen)
**In MoE variants where a single expert is reused across multiple depths (cross-layer expert sharing), does the fault isolation that layer-isolated MoE is believed to provide collapse — i.e., does a single-expert corruption's blast radius scale with the expert's depth-reuse count rather than with 1/num_experts?**

In scope: single-expert perturbation/ablation blast radius; matched comparison shared-layer vs layer-isolated; the scaling law vs reuse count. Out of scope: training-time backdoors, deployment/serving attacks, routing-steering attacks (separate lines).

## Why this is a real question (not gap-hunting)
Three established results all assume **one expert lives at one layer**:
- "MoE is more robust / sparsity gives graceful degradation" (2210.10253; 2601.14792 — robustness-to-feature-noise, confirmed layer-isolated framing).
- "early-layer experts are redundant / interchangeable" (2606.10703, causal audit — confirmed per-layer).
- Cross-layer routing is coupled and shares a geometric control subspace (2609.02404 — confirmed mechanistic only, OLMoE/Phi, no robustness, no ablation).

Cross-layer **expert sharing** architectures now exist and are being adopted for parameter efficiency — CS-MoE (2609.22199, global shared expert pool), MoRE (2609.18176, shared pool across adjacent layers), MoUE (2603.04971, universal layer-agnostic pool; explicitly notes "routing path explosion from recursive reuse"), MoEUT (2405.16039, MoE Universal Transformer). The always-on **shared expert** in flagship DeepSeek/Qwen is the limiting case (one expert on every layer's path). **None of these efficiency papers evaluates robustness/fault behavior.** So the premise under the robustness folklore is being quietly violated by the architectures the field is moving toward, and nobody has measured the consequence.

## Core claims & minimum convincing evidence
- **C1 (isolation collapse)**: in a cross-layer-shared MoE, ablating/corrupting one shared expert degrades output far more than ablating one expert in a matched layer-isolated MoE. *MCE*: accuracy/perplexity drop per single-expert ablation, shared vs isolated, matched params/FLOPs.
- **C2 (reuse-count law)**: the blast radius scales monotonically with the expert's **depth-reuse count** (how many layers invoke it), not with 1/N. *MCE*: blast-radius vs reuse-count curve with positive slope; isolated baseline flat near 1/N.
- **C3 (redundancy defense fails)**: the "early-layer redundancy" that protects layer-isolated MoE (2606.10703) does **not** protect shared-layer MoE, because the same expert is also a late-layer expert. *MCE*: early-layer single-expert ablation hurts shared-layer much more than isolated.
- **C4 (mitigation direction)**: depth-conditioning (MoRE's depth embeddings) or capping reuse count partially restores isolation. *MCE*: blast radius vs depth-embedding on/off, vs reuse cap.

A result that **refutes C1/C2** (shared-layer degrades no worse, e.g., depth embeddings fully compensate) is an informative "cross-layer sharing is robustness-neutral" finding — the Anchor is answered either way.

## Honesty ledger (open risks)
- **Importance = adoption risk**: MoEUT/MoRE/MoUE are research architectures, not flagships. Mitigation: anchor on the always-on shared expert (every flagship has it) as the limiting case, and frame as a forward-looking scaling-law warning ("efficiency-via-reuse trades away fault isolation").
- **Confound**: depth embeddings make a shared expert behave differently per layer — must test whether that already restores isolation (C4), else C1 could be overstated.
- **Attribution**: must separate "shared expert is structurally central" from "this particular expert happened to be important" — use reuse-count as the structural variable, averaged over experts.
- Executor-only; no cross-model review; no pilot run (no GPU here).

## Next step
`/experiment-bridge` against `refine-logs/CROSSLAYER_EXPERIMENT_PLAN.md`, starting with C1/C2 single-expert ablation on MoEUT (small, open). Inference-only; feasible on 1 small GPU.
