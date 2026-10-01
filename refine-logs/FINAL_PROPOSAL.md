# Final Proposal — Cross-Tenant Expert-Cache Timing Attack on Offloaded MoE Serving

**Date**: 2026-10-01 · **Source**: `/idea-discovery "MoE security attack surface"` → Idea 1
**Refinement note**: `/research-refine`'s iterative GPT-6-Astra review loop did **not** run (no non-Claude reviewer registered). This proposal is the executor's refined draft; the cross-model refinement that normally tightens it is pending. Treat the Problem Anchor as frozen and the rest as a first draft for human/reviewer iteration.

## Problem Anchor (frozen — prevents scope drift)
**Does the shared expert cache in memory-constrained, offloaded MoE serving leak a co-tenant's expert-activation trace to a remote, latency-only attacker, well enough to reconstruct prompt content?**

Everything downstream serves this one question. Not in scope: training-time attacks, white-box weight access, local microarchitectural channels (that is MoEcho), KV-cache timing (that is Early-Bird), or defenses beyond the one batching/padding mitigation used as a control.

## Method thesis (one sentence)
Because offloaded MoE serving keeps only a bounded set of experts resident in VRAM and streams the rest from host RAM over PCIe, a cache *miss* costs ~1–10 ms versus ~tens of µs for a hit, so a remote tenant's per-token latency vector is a noisy readout of which experts another tenant's tokens recently warmed — which, fed to the published routing→text decoder, partially reconstructs the victim's prompt.

## Dominant contribution
A **new, remote, latency-only attack class** on the *expert* cache (distinct from local-microarchitectural MoEcho and from KV/prompt-cache timing), plus a measured map of the serving regime (cache size, co-tenancy, batch size) where it holds and where batching noise defeats it.

## Core claims and minimum convincing evidence
- **C1 (channel exists)**: hit vs miss for a single offloaded expert is distinguishable by latency. *MCE*: d′/AUROC of the hit/miss latency distribution on one GPU; AUROC ≥ ~0.8 at low concurrency.
- **C2 (channel survives realism)**: separability persists under realistic batch sizes and network RTT. *MCE*: AUROC vs batch size curve; identify the crossover batch size where AUROC → 0.5.
- **C3 (trace → content)**: a noisy recovered trace still yields above-baseline prompt-token reconstruction. *MCE*: top-1 token recovery vs a no-timing baseline, with realistic trace corruption injected into the 2602.04105-style decoder.
- **C4 (mitigation)**: a stated defense (constant-time expert fetch / cache partitioning per tenant / padding) collapses C1–C3. *MCE*: AUROC and reconstruction under the defense drop to baseline.

A result that **refutes C2** (noise wins at realistic batch sizes) is an informative, publishable "offloading is incidentally private" finding — the Problem Anchor is answered either way.

## Honesty ledger (open risks, from the self-critique)
- Timing noise at production batch sizes may kill the channel (C2 is the make-or-break).
- Threat model requires shared-cache co-tenancy; scope claims to the edge/single-GPU-offload regime.
- Decoder was built for clean traces; end-to-end recovery under noise is the real difficulty and the real contribution.

## Next step
`/experiment-bridge` against `refine-logs/EXPERIMENT_PLAN.md`, starting with the C1 micro-pilot (cheapest kill/confirm). Requires a GPU — unavailable in this environment.
