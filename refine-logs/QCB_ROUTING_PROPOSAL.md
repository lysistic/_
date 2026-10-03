# Final Proposal — Routing-Mediated Quantization-Conditioned Backdoors in MoE, and a Routing-Aware Audit

**Date**: 2026-10-03 · **Source**: `/idea-discovery "MoE security attack surface"` → quantization × MoE-routing thread
**Framing**: security research — the deliverable is a **detector/audit** (defense); the attack half demonstrates the vulnerability exists and measures it. Kept at mechanism/claim level, not an operational recipe.
**Refinement note**: `/research-refine`'s cross-model loop did not run (no non-Claude reviewer in this env). Executor-only, UNADJUDICATED. Problem Anchor frozen.

## Problem Anchor (frozen)
**In an MoE LLM, can a backdoor be made to activate only after standard deployment quantization (INT8/INT4) by exploiting the router's discrete top-k flip — rather than the weight-rounding bins that dense quantization-conditioned backdoors (QCB) and their defenses rely on — and does this evade the existing weight-bin QCB defense family?**

In scope: a backdoor whose trigger is a quantization-induced routing flip toward an otherwise-dormant, clean-looking expert; whether FlipGuard/rounding-trap-class defenses miss it; a routing-aware audit that catches it pre-deployment. Out of scope: hardware bit-flip (Groundhog), input-triggered backdoors (BadMoE), accidental quant damage (2608.11212).

## Why it is not redundant with dense QCB (the key technical point)
Dense QCB (Egashira 2024) and its defenses operate in the **continuous weight-rounding bin**: the attack engineers an FP↔INT gap in a smooth weight→output map; defenses (FlipGuard 2606.28962, "Breaking the Rounding Trap"/QuantGuard 2606.29239) **smooth the weights' fractional parts** to break that alignment. Full-text confirmed both defenses are **weight-only and never mention MoE/routing**.

MoE routing is a **discrete top-k argmax**. A routing-mediated QCB's trigger is "which expert is selected flips under quantization" — the malicious payload can be a **clean, fully-functional expert**, not a delicately-rounded weight. Therefore weight-bin defenses are structurally blind to it: smoothing fractional parts does not change an argmax the attacker placed with margin, and there is no legitimate-looking expert to "smooth away." This is a **different mechanism**, not dense QCB relocated.

## Core claims & minimum convincing evidence
- **C1 (attack exists)**: one can position a dormant expert so it is never top-k in FP (benign; passes FP safety audit) but is reliably selected after standard INT4 PTQ on a trigger, producing attacker-chosen behavior; clean inputs and FP behavior unchanged. *MCE*: FP ASR ≈ 0 and FP utility intact; INT4 ASR high on trigger; INT4 clean utility intact.
- **C2 (defense evasion — the category finding)**: FlipGuard and rounding-trap/QuantGuard, applied to the poisoned checkpoint, do **not** remove C1 (because they act on weight bins, not routing). *MCE*: run both defenses; INT4 ASR stays high. This is the paper's weight-bearing result — it shows the weight-bin QCB defense family has a structural blind spot on sparse architectures.
- **C3 (routing-aware audit — the deliverable)**: a pre-deployment detector that compares FP vs simulated-INT routing on a probe set and flags experts whose selection flips from dormant→active under quantization catches C1 with low false-positive rate, where C2 defenses fail. *MCE*: detector AUROC on poisoned vs clean checkpoints; FPR on benign quant-sensitive models.
- **C4 (margin/robustness)**: the attack survives common PTQ variants (GPTQ/AWQ, INT8 and INT4) within a stated margin, or its regime is characterized. *MCE*: ASR across quantizers.

Refuting C1 (cannot construct a stable FP-dormant/INT-active expert) or C2 (weight-bin defenses already remove it) collapses the attack half — but C3 (the audit) remains independently useful, and a clean negative on C2 ("existing defenses already cover routing") is itself a publishable reassurance.

## Honesty ledger
- **Constructibility is the empirical make-or-break** (C1): needs a GPU pilot on a small open MoE (OLMoE / Qwen3-MoE); not runnable in this env.
- **Dual-use**: framed defense-first (the audit C3 is the deliverable); standard for the QCB subfield. No operational construction details beyond the mechanism.
- **Scoop**: full-text-checked free vs Egashira (dense), AgentQ (dense), FlipGuard/rounding-trap (weight-only, MoE-blind), 2608.11212 (accidental), BadMoE (input-trigger), 2609.12550 (efficiency). One residual: a brand-new unindexed paper could exist.
- Executor-only; no cross-model adversarial review ran.

## Next step
`/experiment-bridge` against `refine-logs/QCB_ROUTING_EXPERIMENT_PLAN.md`; the C2 defense-evasion test is the decisive, cheapest result and should run first after C1.
