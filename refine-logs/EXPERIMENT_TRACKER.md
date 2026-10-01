# Experiment Tracker — Cross-Tenant Expert-Cache Timing Attack

Status legend: ⬜ not started · 🟡 running · ✅ done · ❌ failed/killed · ⏸ blocked

| Block | Claim | Status | Compute | Gate | Result |
|-------|-------|--------|---------|------|--------|
| 0 — leakage ceiling (simulator) | C1/C3 ceiling | ⏸ blocked | CPU | ceiling non-trivial? | — (CPU-runnable; not started this run) |
| 1 — hit/miss separability | C1 | ⬜ | 1 GPU ~0.5d | AUROC ≥ 0.8 | — |
| 2 — realism sweep | C2 | ⬜ | 1 GPU ~2d | find AUROC→0.5 crossover | — |
| 3 — reconstruction under noise | C3 | ⬜ | 1 GPU ~2d | recovery > baseline | — |
| 4 — defense | C4 | ⬜ | 1 GPU ~1d | drops to baseline | — |

**Blocker**: no GPU in this environment (`nvidia-smi` absent). Block 0 is CPU-only and can start immediately once prioritized; Blocks 1–4 need a GPU host (SSH to the user's server, or Vast/Modal via `/run-experiment`).

**First action when compute is available**: Block 0 (simulator ceiling) → Block 1 (separability micro-pilot) — together the cheapest kill/confirm of the whole line.

## Decision log
- 2026-10-01: Plan created from `/idea-discovery`. Cross-model refine loop and pilots not run (no reviewer, no GPU). Awaiting compute + optional external review.
