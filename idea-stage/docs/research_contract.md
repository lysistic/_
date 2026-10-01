# Research Contract: Cross-Tenant Expert-Cache Timing Attack on Offloaded MoE Serving

> Focused working document for the currently selected idea (W1 → W1.5 handoff). On session recovery, load THIS, not the full IDEA_REPORT.

## Selected Idea

- **Description**: In memory-constrained MoE serving, inactive experts are offloaded to host RAM and streamed to the GPU on demand through a shared, bounded expert cache. A cache miss costs ~1–10 ms (PCIe fetch) versus ~tens of µs for a hit. A remote tenant that shares the cache and measures its own per-token latency can read out *which experts a co-tenant's tokens recently warmed*, and feed that noisy expert-activation trace to the published routing→text decoder (arXiv 2602.04105) to partially reconstruct the co-tenant's prompt. This is a remote, latency-only channel on the *expert* cache — distinct from local-microarchitectural MoEcho (2508.15036) and from KV/prompt-cache timing (2409.20002).
- **Source**: IDEA_REPORT.md, Idea #1 (RECOMMENDED).
- **Selection rationale**: Fills the clearest gap (G1); highest information/upside of the slate; partly CPU-pilotable (simulator ceiling); builds on established facts (offloading is standard; routing leaks text). Novelty UNADJUDICATED (no cross-model reviewer this run); executor estimate ~8/10.

## Core Claims

1. **C1** — Hit vs miss for an offloaded expert is distinguishable by end-to-end latency (AUROC ≥ ~0.8 at low concurrency).
2. **C2** — The channel survives realistic batch sizes and network RTT, up to an identifiable crossover batch size.
3. **C3** — A noisy, partial recovered trace still reconstructs prompt tokens above a no-timing baseline.
4. **C4 (scope/defense)** — Constant-time/padded expert fetch or per-tenant cache partitioning collapses C1–C3.

## Method Summary

Stand up an offloaded MoE server (OLMoE-1B-7B primary, Mixtral-8×7B secondary) with a device-side LRU/LFU expert cache keeping N experts resident and streaming the rest from host RAM, serving two simulated tenants. The attacker tenant issues single tokens and records latency, converting the latency vector into a per-layer hit/miss (hence expert-activation) estimate. A reimplemented routing→text decoder (MLP / transformer, trained on OpenWebText routing traces) maps recovered traces to tokens. Realism is swept over batch size, cache resident fraction, and simulated RTT; the one defense block re-runs everything under padding and partitioning.

## Experiment Design

- **Datasets**: OpenWebText (decoder training + prompt corpus); JailbreakBench not needed (this is a privacy, not safety, attack).
- **Baselines**: no-timing random-guess reconstruction; clean-trace decoder (upper bound).
- **Metrics**: hit/miss AUROC & d′ (C1/C2); per-layer trace top-1 (C3); prompt-token top-1/top-10 (C3).
- **Key hyperparameters**: cache resident fraction (10/25/50%), batch size (1/4/16/64), decoder size.
- **Compute budget**: ~1 CPU-day (Block 0) + ~8 GPU-days.

## Baselines

| Method | Dataset | Metric | Score | Source |
|--------|---------|--------|-------|--------|
| No-timing guess | OpenWebText | token top-1 | chance | reproduced |
| Clean-trace decoder | OpenWebText | token top-1 | 0.912 (reported) | arXiv 2602.04105 |

## Current Results

> Empty — no compute this run.

| Method | Dataset | Metric | Score | Notes |
|--------|---------|--------|-------|-------|
| — | — | — | — | GPU unavailable this run |

## Key Decisions

- Simulator (Block 0) before hardware: establishes the leakage ceiling on CPU; if the ceiling is low, the line stops cheaply.
- Scope to the memory-constrained / single-GPU-offload regime (the threat model that makes shared expert caches realistic).
- Inject realistic trace corruption into the decoder — the contribution is reconstruction under *noise*, not from clean routing.

## Status

- [x] Idea selected
- [ ] Cross-model novelty + review acquittal (BLOCKED: no non-Claude reviewer registered)
- [ ] Block 0 leakage-ceiling simulator (CPU — not started)
- [ ] Block 1 hit/miss separability (needs GPU)
- [ ] Realism sweep / reconstruction / defense (needs GPU)
- [ ] Paper draft
