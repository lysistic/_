# Experiment Plan — Routing-Mediated QCB in MoE + Routing-Aware Audit

**Anchor**: see `QCB_ROUTING_PROPOSAL.md`. Claim-driven; small open MoE; one modest GPU.
**Compute**: no GPU in this env; all blocks `needs pilot`. OLMoE-1B-7B / Qwen3-MoE-small are feasible on 1 GPU.

## Setup
- **Models**: OLMoE-1B-7B (open, documented routing) primary; Qwen3-MoE-small secondary.
- **Quantizers**: INT8 and INT4 via GPTQ and AWQ (the two most-deployed PTQ paths).
- **Safety/trigger task**: a small harmful-behavior set (AdvBench-style) for ASR + MMLU/WikiText for clean utility.
- **Metrics**: ASR (trigger), clean utility (ΔMMLU/ΔPPL), per-expert selection frequency FP vs INT, detector AUROC/FPR.

## Block 0 — route-flip susceptibility map (CPU-light / 1 GPU) → prerequisite for C1
Measure, per expert, how close its router logit margin is to the top-k boundary, and how INT4 PTQ perturbs router logits (reuse the 2608.11212 observation that quant flips routing). Output: which experts are "flippable" by quantization. No attack yet. Decision: if no expert sits in a quant-flippable band with usable margin, C1 is infeasible → report that (negative, still informative).

## Block 1 — construct & verify the attack (1 GPU) → C1
Place a dormant, clean-functioning expert at a quant-flippable boundary so FP keeps it inactive (benign) and INT4 selects it on the trigger. **Gates**: FP ASR≈0 & FP utility intact; INT4 trigger ASR high; INT4 clean utility intact. (Construction kept at mechanism level; this validates existence, not a deployable recipe.)

## Block 2 — defense evasion (1 GPU) → C2 (DECISIVE)
Apply FlipGuard and rounding-trap/QuantGuard to the Block-1 checkpoint; re-measure INT4 ASR. **Gate C2**: ASR stays high after both → weight-bin defenses structurally miss routing-mediated QCB. This is the paper's core claim; run it first after C1.

## Block 3 — routing-aware audit (1 GPU) → C3 (DELIVERABLE)
Detector: on a probe set, compare FP routing vs simulated-INT routing; flag any expert whose selection goes dormant→active under quantization (or whose activation shifts safety-expert routing). Evaluate on poisoned (Block 1) vs a bank of clean/benign-quant-sensitive checkpoints. **Gate C3**: high AUROC separating poisoned from clean, low FPR.

## Block 4 — robustness / regime (1 GPU) → C4
Sweep INT8/INT4 × GPTQ/AWQ; report the quantizer/precision regime where the attack holds and where the audit catches it.

## Ablation matrix
| Axis | Levels | Purpose |
|------|--------|---------|
| Precision | INT8 / INT4 | regime (C1,C4) |
| Quantizer | GPTQ / AWQ | regime (C4) |
| Defense | none / FlipGuard / rounding-trap | evasion (C2) |
| Model | OLMoE / Qwen3-MoE | generality |
| Detector feature | dormant→active flip / safety-expert routing shift | audit design (C3) |

## Run order
Block 0 (map) → Block 1 (C1 exists) → **Block 2 (C2 defense evasion — decisive)** → Block 3 (C3 audit) → Block 4 (regime). Stop & write the corresponding result at any failed gate.

## Budget
~0.5 CPU-day (Block 0 partial) + ~4–6 GPU-days on one modest GPU (small MoE, inference + PTQ + light optimization).
