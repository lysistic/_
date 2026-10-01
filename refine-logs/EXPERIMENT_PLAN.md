# Experiment Plan — Cross-Tenant Expert-Cache Timing Attack

**Anchor**: see `FINAL_PROPOSAL.md`. Claim-driven; every block maps to C1–C4.
**Compute note**: no GPU in this environment — all GPU blocks are `needs manual pilot`. Block 0 (simulator) is CPU-only and runnable now.

## Setup
- **Models**: OLMoE-1B-7B (64 experts, open, fully reproducible routing) as primary; Mixtral-8×7B as a secondary larger check. Both have public weights and documented routing.
- **Serving**: a device-side LRU/LFU expert cache keeping N experts resident in VRAM, streaming the rest from host RAM (the vllm-expert-cache / ExpertFlow setup). Two simulated tenants sharing one cache.
- **Decoder**: reimplement the routing→text MLP/transformer decoder of 2602.04105 (code not assumed; architecture is described) trained on OpenWebText routing traces.
- **Metrics**: hit/miss latency AUROC & d′; expert-trace recovery accuracy (per-layer top-1); prompt-token reconstruction top-1/top-10; all vs a no-timing random baseline.

## Block 0 — Leakage ceiling (CPU, runnable now) → C1/C3 upper bound
Trace-driven, event-atomic cache simulator (design per 2608.07911) over real OLMoE routing traces. No timing — compute the *information-theoretic* ceiling: given perfect hit/miss observation, how much of the victim trace (and thus prompt) is recoverable vs cache size and co-tenancy rate. **Decision**: if even the ceiling is low, stop — the channel cannot carry enough. Budget: CPU-hours.

## Block 1 — Hit/miss timing separability (1 GPU) → C1
Micro-benchmark: force known hit and known miss for a single offloaded expert, measure end-to-end latency, compute AUROC/d′ at concurrency = 1. **Gate**: AUROC ≥ 0.8 to proceed. Budget: ~0.5 GPU-day.

## Block 2 — Realism sweep (1 GPU) → C2
AUROC vs {batch size ∈ 1,4,16,64; cache size; +simulated network RTT}. Find the crossover batch size where AUROC → 0.5. **This block decides whether the attack is real.** Budget: ~2 GPU-days.

## Block 3 — End-to-end reconstruction under noise (1 GPU) → C3
Feed Block-2 noisy traces into the decoder; report prompt-token recovery vs no-timing baseline across the regimes where Block 2 kept AUROC high. Ablation: clean-trace vs timing-trace decoder input (isolates the noise penalty). Budget: ~2 GPU-days + decoder training.

## Block 4 — Defense (1 GPU) → C4
Re-run Blocks 1–3 under (a) constant-time/padded expert fetch and (b) per-tenant cache partitioning. **Gate**: AUROC & reconstruction drop to baseline. Budget: ~1 GPU-day.

## Ablation matrix
| Axis | Levels | Purpose |
|------|--------|---------|
| Batch size | 1 / 4 / 16 / 64 | C2 crossover |
| Cache resident fraction | 10% / 25% / 50% | C1/C3 vs cache pressure |
| Decoder input | clean trace / timing trace | isolate noise penalty (C3) |
| Defense | none / padding / partition | C4 |
| Model | OLMoE / Mixtral | generality |

## Run order
Block 0 (now, CPU) → Block 1 → **Block 2 (kill/confirm gate)** → Block 3 → Block 4. Stop at any gate that fails and write the negative result.

## Total budget
~1 CPU-day (Block 0) + ~8 GPU-days. Within an 8× consumer-GPU week.
