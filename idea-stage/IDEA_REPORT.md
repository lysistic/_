# Idea Discovery Report

**Direction**: MoE security attack surface
**Date**: 2026-10-01
**Pipeline**: research-lit → idea-creator → novelty-check → research-review → research-refine-pipeline
**Run state**: `.aris/runs/moe-security-attack-surface-20261001.json` (executor: `claude-opus-5-5`)

## Executive Summary

The MoE security literature is dominated by one crowded thread — jailbreaking by silencing/steering a few "safety-critical experts" (SAFEx, GateBreaker, L³, RouteHijack, Unsafe Routes) — plus active work on routing backdoors, availability/DoS attacks, local side channels, and IP watermarking. Scanning ~60 on-topic papers surfaced six under-explored gaps. The recommended direction is **a cross-tenant, remote, latency-only timing attack on the shared *expert cache* of offloaded MoE serving (Idea 1)**: expert offloading is now standard, a PCIe expert-miss costs milliseconds, and a published decoder already turns expert-activation traces into text — yet no one has attacked the expert cache *remotely* (MoEcho needs local access; KV-cache timing targets a different cache). Its single cheapest kill/confirm experiment is a hit-vs-miss latency-separability micro-pilot. Two lower-risk backups: a matched dense-control audit of whether "safety sparsity" is actually MoE-specific (Idea 2), and the silent safety regression caused by benign efficiency pruning (Idea 3).

**⚠️ Completeness caveat:** this run executed on a host with **no GPU and no non-Claude reviewer**. All pilots were skipped and the two reviewer-bearing phases (`novelty-check`, `research-review`) returned `REVIEW_UNAVAILABLE` — their verdicts are the executor's own and **UNADJUDICATED**. The deterministic evidence gate (bottom of report) therefore reports **BLOCKED**. This is a validated *candidate* slate for a human or a reviewer-equipped re-run, not a cleared pipeline.

## Literature Landscape

Survey scope: ~240 arXiv records retrieved via the arXiv API (24 structured queries) plus targeted WebSearch; ~60 are on-topic for MoE security. Sources that contributed: `arxiv` (helper exit 0), `web` (WebSearch). Sources skipped: Zotero/Obsidian (no MCP configured), local PDFs (WARN: local contributed nothing — no PDFs found in `papers/`, `literature/`, or a configured paper library), Semantic Scholar (HTTP 429, no API key), Gemini (`gemini-cli` not installed). Venue metadata is therefore incomplete; arXiv IDs are authoritative below.

### Landscape map (8 sub-directions)

**1. Safety-expert localization → jailbreak by expert silencing / routing steering (SATURATED).** The dominant 2025–2026 thread. Shared finding: refusal behaviour concentrates in a small set of experts or routers.
- Weight/inference-time interventions: SAFEx (2506.17368; masking 12/6,144 experts of Qwen3-30B-A3B cuts refusal 22%), SteerMoE (2509.09660; expert (de)activation, −41% safety alone, −100% with jailbreaks), GateBreaker (2512.21008, USENIX Sec '26; disabling ~3% of neurons in expert layers lifts ASR 7.4%→64.9%, transfers within family and to MoE-VLMs), Large Language Lobotomy L³ (2602.08741; adaptive expert silencing, ASR 7.3%→70.4%), Unsafe Routes / RoSais + F-SOUR (2602.08621; masking 5 routers of DeepSeek-V2-Lite → ASR 0.79; F-SOUR 0.90–0.98), RASET (2605.29708; tuning only safety-critical experts, routing unchanged), single-direction ablation on a 320B MoE (2609.09793; 74% of the effect needs joint attention+dense+expert edits), cross-lingual refusal circuit in a multilingual MoE (2608.08032).
- Input-only: RouteHijack (2605.02946; routing-aware adversarial suffix, 69.3% ASR, zero-shot transfer to siblings and VLMs), Misrouter (2605.04446; surrogate-optimised, transfers to public APIs).
- Defenses: SafeMoE (2509.22745; penalise routing drift under harmful fine-tuning), RASA (2602.04448; repair safety experts under fixed routing), MESA (2606.00651; OT-based decentralisation of safety), SEAL/SEAL++ (2609.02293; safety adapter on always-on shared expert), UpSafe°C (2510.02194), dialectical SafeMoE (2606.00686).
- **Tension in the literature**: SAFEx/L³/GateBreaker say safety lives in a few *routed* experts; RASET (2605.29708) reports routing is largely topic-driven and safety can be flipped without changing routes; 2609.09793 finds most refusal removal requires *non-expert* modules jointly. No paper benchmarks these "MoE-specific" fragilities against matched dense controls (dense safety is also ~3%-sparse per prior pruning work), so "safety sparsity is an MoE property" is asserted, not isolated. MoE-RBench (2406.11353) compared MoE vs dense reliability but predates every routing attack above.

**2. Routing-level backdoors / poisoned checkpoints (CROWDED).** BadMoE (2504.18598; dormant experts + routing triggers), BadSwitch (2510.13462; trigger embedded in expert routing paths, up to 100% ASR), BadPatches (2505.01811; patch-routed triggers in vision MoE, 0.01% poisoning), Capacity Overflow (2608.25371; backdoor neutraliser disabled by batch-size-dependent capacity → dormant in small-batch audits, active at deployment batch sizes), Load Hijack (2608.10614; poisoned router as trigger-controlled device scheduler in expert-parallel serving). General merging backdoors (BadMerging 2408.07362; Merge Hijacking, ACL'25; "When Safe Models Merge into Danger" 2604.00627) do not specifically target routing-based MoE merging (mergekit-moe / BTX-style router initialisation) — 0 arXiv hits.

**3. Routing as a privacy side channel (ACTIVE, mostly local-attacker).** Expert selections alone reconstruct 91.2% of tokens top-1 with a transformer decoder (2602.04105) — the "routing ≈ text" result. Hardware channels: MoEcho (2508.15036; cache-occupancy, Pageout+Reload on CPU, perf counters and TLB Evict+Reload on GPU — all require co-located microarchitectural access), GATEBLEED (2507.17033; AMX power-gating timing leaks expert choice with 100% accuracy). Defensive inversions: RouteScan (2605.24817) uses GPU routing telemetry to audit harmful prompts; CryptoMoE (2511.01197) / SecMoE (2601.06790) hide routing in 2PC inference. Adjacent non-MoE: KV/prompt-cache timing attacks (Early Bird 2409.20002; prompt-caching audits 2502.07776) and detokenizer cache traces (2609.06674). **Gap: no remote, network-level timing attack on the shared *expert cache* of offloaded MoE serving** — expert offloading is now standard (2608.07911 calls it "standard"; ExpertFlow 2410.17954, SpecMD 2602.03921, OD-MoE 2512.03927, SlimCaching 2507.06567, FaaSMoE multi-tenant 2604.26881), and a PCIe expert fetch costs milliseconds, an order of magnitude above KV-cache hit/miss gaps.

**4. Cross-batch interference (EARLY, toy-scale).** Buffer Overflow in MoE (2402.05526; capacity-based dropping lets malicious co-batched queries change benign outputs — toy setting) and Stealing User Prompts (2410.22884; expert-choice routing + `torch.topk` tie-breaking leaks a co-batched victim prompt on a 2-layer Mixtral). Production models use dropless token-choice routing, which removes these specific channels; batch-dependent numerics and capacity-bounded vision MoE (2608.25371) remain.

**5. Availability / cost attacks (ACTIVE).** RepetitionCurse (2512.23995; repetitive prompts concentrate routing → 3.06× latency under expert parallelism), Load Hijack (2608.10614), Groundhog bit-flip (2608.25276; flipping routing bits around EOS experts → 5,912% output inflation). Survey: 2609.36732. **Gap**: online load balancers (redundant-expert placement à la EPLB) learn from observed traffic, so their statistics are themselves attacker-influenced — unexplored.

**6. Memorisation and training-data privacy (UNEXPLORED for security).** Capability papers show MoEs memorise more than dense models at equal active compute (Mixture of Parrots 2410.19034) and overfit faster to repeated data, scaling with *total* parameters (2609.11917; MoEs degrade from 4× repetition vs 8× for dense). Knowledge attribution finds a low-entropy 1% neuron backbone in MoE (2601.08383). **No security paper measures whether MoEs leak more training data (extraction / membership inference) or whether routing traces are a membership signal.** SHAPOOL (2510.13451) only uses MoE to build cheaper shadow models.

**7. IP protection, compression and access control (ACTIVE).** Watermarks/fingerprints: PathMark (2607.03688), WaterMoE (2607.13099), WoE (2608.29151), RouteMark (2508.01784). Unauthorized compression via expert pruning (2511.19480, NeurIPS'25). Capability access control with private experts (2608.06690). Expert-pruning efficiency methods (REAP, MAESTRO 2607.08601, EAC-MoE 2508.01625, Sub-MoE 2506.23266) are evaluated on utility (MAESTRO adds a "Safety" domain) but not on jailbreak robustness; the safety cost of *benign* expert pruning is unmeasured.

**8. Unlearning and adversarial robustness (SCATTERED).** GRIP (2601.16905) shows MoE unlearning methods cheat by re-routing, leaving knowledge in dormant experts recoverable by router bypass; classic adversarial robustness of MoE (2210.10253, 2412.11608, 2509.05086, Robust CurveMoE 2608.26043, video MoE TLGA 2602.01369).

### Structural gaps (input to idea generation)
- **G1 Remote expert-cache timing**: shared expert caches in offloaded serving are an unstudied cross-tenant channel; MoEcho needs local hardware access, KV-cache attacks need shared prefixes.
- **G2 Attribution of "MoE-specific" safety fragility**: no matched dense-control study; conflicting localisation claims (Cluster 1 tension).
- **G3 MoE memorisation as a privacy risk**: capability evidence that MoEs memorise more, zero security follow-up; routing is an extra membership feature.
- **G4 Safety cost of benign expert pruning/merging**: compression papers report utility only; community-pruned checkpoints are already distributed.
- **G5 Feedback-driven serving components** (online load balancers, prefetch predictors) as attack targets.
- **G6 Routing-based model merging** (mergekit-moe, BTX) as a backdoor vector distinct from weight-averaging merges.

## Ranked Ideas

**Generation**: 10 candidates generated across the Phase-1 gap lenses (method-transfer, contradiction, untested-assumption, scaling-regime, diagnostic), then annotated. **The cross-model jury (Phase 4 of `/idea-creator`) did NOT run**: it requires a non-Claude reviewer (Codex MCP / `gpt-6-astra`), which is not registered in this environment, and ARIS forbids Claude from acquitting Claude-generated ideas (`shared-references/reviewer-routing.md`). Novelty scores and the ranking below are therefore the **executor's own same-family judgment, UNADJUDICATED** — treat as candidate signal, not a verdict. **Pilots**: no GPU in this environment (`nvidia-smi` absent), so every pilot is `SKIPPED (needs manual pilot)`; where a pilot is CPU-simulable that is noted as a cheaper first step.

### 🏆 Idea 1: Cross-tenant expert-cache timing attack on offloaded MoE serving — RECOMMENDED
- **Method (what we actually do)**: (1) Stand up an offloaded MoE server (e.g. OLMoE-6.9B or Mixtral-8×7B with experts in host RAM, loaded to GPU on demand — the ExpertFlow/SpecMD setup) with a shared expert cache serving two tenants. (2) As tenant B, send one token at a time and record end-to-end latency; a PCIe/host→GPU expert fetch (cache miss) costs ~1–10 ms, an order of magnitude above a hit, so the latency vector reveals *which experts tenant A's recent tokens warmed*. (3) Feed the recovered per-layer expert-activation sequence into the published routing→text decoder (2602.04105) to reconstruct tenant A's prompt tokens. (4) Quantify reconstruction vs cache size, co-tenancy rate, and a batching/padding defense.
- **Hypothesis**: A purely remote attacker (latency only, no co-located microarchitectural access) can recover a co-tenant's expert-activation trace well enough to reconstruct prompt content, because expert-cache state is shared and miss latency is large and input-dependent.
- **Minimum experiment**: CPU-only first — a trace-driven cache simulator (reuse the event-atomic simulator design of 2608.07911) over real routing traces to establish an information-theoretic ceiling on hit/miss leakage per prompt. Then one-GPU offloaded server with synthetic co-tenancy to measure real timing separability of hit vs miss.
- **Expected outcome**: Positive = measurable prompt-token recovery above a no-timing baseline → a new remote attack class. Negative = cache/batching noise destroys the signal → an informative "offloading is incidentally private" result. Either is publishable.
- **Novelty [UNADJUDICATED]**: ~8/10 (executor estimate). Closest: MoEcho (2508.15036) needs *local* cache-occupancy/TLB access; GATEBLEED (2507.17033) needs on-core AMX timing; Early-Bird/prompt-cache attacks (2409.20002, 2502.07776) target the *KV/prompt* cache, not the *expert* cache. No paper attacks the shared expert cache remotely. `prior_work`: 2602.04105 supplies the decoder; 2608.07911/2602.03921 establish offloading+caching as standard.
- **Feasibility**: Simulator stage CPU-only (runnable here). GPU stage: 1 GPU, open weights. ~1–2 weeks.
- **Risk**: MEDIUM (real-serving timing noise may wash out the channel — but that itself is the finding).
- **Contribution type**: empirical / new attack.
- **Pilot result**: SKIPPED (needs GPU for the timing stage; simulator stage pilotable on CPU).
- **Reviewer's likely objection**: "Dropless token-choice + large batches average out per-token timing." Response: measure exactly that boundary; report the co-tenancy/cache-size regime where it holds and where it breaks.
- **Why we should do this**: Expert offloading is now the default deployment for large MoE; if its shared cache leaks across tenants it is an immediate, deployed-systems vulnerability, not a lab curiosity.

### 🥈 Idea 2: Is "safety sparsity" actually MoE-specific? A matched dense-control audit — RECOMMENDED
- **Method (what we actually do)**: (1) Pick model pairs that share a lineage: an upcycled/sibling MoE and its dense counterpart at matched *active* parameters (e.g. OLMoE vs OLMo, Qwen3-MoE vs Qwen3-dense). (2) Run the *same* safety-neuron/expert localization and ablation-based jailbreak (the SAFEx/GateBreaker/L³ procedure) on both, under an identical intervention budget (fraction of FFN neurons ablated). (3) Plot refusal-rate collapse vs ablation budget for MoE and dense side by side; also report whether MoE "safety experts" are more concentrated than dense "safety neurons" by the same metric.
- **Hypothesis**: Much of the reported "MoE safety fragility" is inherited dense sparsity re-described in expert vocabulary; under matched budgets, MoE is comparably — not uniquely — fragile. (Directly tests the Cluster-1 contradiction: SAFEx/L³ say experts; RASET 2605.29708 says routing is topic-driven; 2609.09793 says most of the effect is in non-expert modules.)
- **Minimum experiment**: Inference-only ablation sweeps on 2 model pairs + 1 jailbreak benchmark (JailbreakBench/AdvBench). No training.
- **Expected outcome**: Either MoE is genuinely more fragile at matched budget (validates the field's framing and quantifies the gap) or it is not (reframes a year of "MoE-specific" attacks as a measurement artifact). High-value either way.
- **Novelty [UNADJUDICATED]**: ~7/10. Closest: MoE-RBench (2406.11353) compared MoE/dense reliability but predates all 2025–26 routing attacks and used no ablation-budget control; no attack paper above includes a matched dense control.
- **Feasibility**: Inference only, 1 GPU, open weights. ~1–2 weeks.
- **Risk**: LOW (well-posed measurement; a clear number comes out regardless).
- **Contribution type**: diagnostic.
- **Pilot result**: SKIPPED (needs GPU).
- **Reviewer's likely objection**: "Matched active params isn't a fair control — total params differ." Response: report both matched-active and matched-total, and a per-neuron normalized concentration metric.
- **Why we should do this**: It is the cheapest, most defensible paper here and it disciplines a crowded, confused literature — exactly the kind of result that gets cited by every subsequent MoE-safety paper.

### 🥉 Idea 3: Silent safety regression from benign (safety-agnostic) expert pruning — RECOMMENDED / BACKUP
- **Method (what we actually do)**: (1) Take an aligned MoE (e.g. Qwen3-30B-A3B, Mixtral-8×7B-Instruct). (2) Apply published *efficiency* pruning that never mentions safety — REAP, MAESTRO (2607.08601), EAC-MoE (2508.01625) — at increasing compression ratios. (3) After each prune, measure jailbreak ASR and over-refusal with no re-alignment. (4) Correlate removed experts with the safety-critical set SAFEx/GateBreaker identify, to explain any regression mechanistically.
- **Hypothesis**: Safety-agnostic expert pruning disproportionately removes or weakens safety-critical experts, so community-distributed "efficient" MoE checkpoints are quietly less safe than their unpruned originals.
- **Minimum experiment**: Apply one existing pruning tool at 3–4 ratios to one aligned MoE; run one jailbreak + one over-refusal benchmark.
- **Expected outcome**: Positive = a supply-chain safety warning about efficiency checkpoints (actionable). Null = pruning is safety-neutral (reassuring, still publishable as a negative).
- **Novelty [UNADJUDICATED]**: ~6/10. Closest: Unauthorized Compression (2511.19480) studies *adversarial* compression to bypass licensing; MAESTRO adds a "Safety" *utility* domain but no jailbreak ASR. The safety cost of *benign* pruning is unmeasured.
- **Feasibility**: Inference + lightweight pruning, 1 GPU. ~1 week.
- **Risk**: MEDIUM (effect may be small if pruning avoids high-traffic safety experts — which is itself a clean finding).
- **Contribution type**: empirical / diagnostic.
- **Pilot result**: SKIPPED (needs GPU).
- **Reviewer's likely objection**: "Just re-align after pruning." Response: the point is that practitioners *don't*, and distributed checkpoints ship un-realigned.
- **Why we should do this**: Directly actionable for the open-weight ecosystem; small, well-scoped, hard to scoop quickly.

### Other candidates (annotated, carried forward un-eliminated)
| # | Idea (lens) | Hypothesis | Novelty [UNADJ.] | Gap | Pilot |
|---|-------------|------------|------------------|-----|-------|
| 4 | **Routing traces as a membership-inference feature** (method-transfer) | Expert-selection sequences are a stronger membership signal than loss for MoE LLMs, because MoEs memorize more (2410.19034) and routing leaks (2602.04105). | 7/10 | G3 | needs GPU |
| 5 | **Poisoning the online load balancer** (method-transfer/diagnostic) | A tenant that shapes its own traffic biases EPLB-style adaptive expert placement, degrading co-tenants — poisoning the *scheduler*, not the model (beyond static RepetitionCurse 2512.23995 / Load Hijack 2608.10614). | 7/10 | G5 | CPU-simulable |
| 6 | **Routing-based merge backdoor** (method-transfer) | A malicious donor in mergekit-moe/BTX contributes an expert + router-trigger that survives merge — distinct from weight-averaging BadMerging (2408.07362). | 6/10 | G6 | needs modest GPU |
| 7 | **Unlearning auditor via forced dormant-expert routing** (diagnostic) | "Unlearned" MoEs (shown by GRIP 2601.16905 to cheat via re-routing) still hold erased knowledge in dormant experts recoverable by forced routing — a red-team tool. | 6/10 | G8 | needs GPU |
| 8 | **Black-box serving-stack fingerprinting via routing-induced latency** (diagnostic/recon) | A remote user infers model family / quantization / expert-parallel degree of a black-box MoE API from routing-shaped latency distributions — reconnaissance enabling Idea 1/5. | 5/10 | G1/G5 | CPU + API |
| 9 | **Capacity-overflow co-batch degradation of a victim's refusal** (contradiction) | Inverting the 2608.25371 backdoor: a co-batched attacker triggers capacity-based dropping of a victim's safety-relevant tokens with no model modification. | 5/10 | G4 | needs GPU; risky (dropless deployments) |
| 10 | **Extend co-batch prompt-stealing to dropless/batch-invariant kernels** (scaling-regime) | Re-test 2410.22884's tie-break leak under modern dropless token-choice + batch-invariant kernels. | 4/10 | G4 | mostly closes existing attack; lower novelty |

### Eliminated at generation
_None eliminated on budget: all 10 are inference- or simulator-scale on ≤1 GPU. The usual Phase-3 budget gate dropped nothing; the Phase-4 cross-model jury that would normally narrow on quality could not run (no non-Claude reviewer), so the full set is carried forward with the ranking marked UNADJUDICATED._

## Novelty Verification

> **Status: literature-search half COMPLETE; cross-model verification NOT RUN → `REVIEW_UNAVAILABLE`.** `/novelty-check` is a reviewer-bearing phase: a positive novelty verdict must come from a non-Claude model (Codex `gpt-6-astra`, xhigh) per `shared-references/reviewer-routing.md`. No such reviewer is registered in this environment, and Claude may not acquit Claude-generated novelty. The multi-source *search* below was done (arXiv API + standard & extended WebSearch); the **cross-model acquittal was not**, so no `accept` receipt is written and the final evidence gate will report this phase BLOCKED. Novelty calls below are the executor's own and **UNADJUDICATED**. Per the skill, both PROCEED and PROCEED-WITH-CAUTION would be positive here — but neither can be *recorded* without the external verdict.

**Idea 1 — Cross-tenant expert-cache timing.** Closest prior work: **MoEcho** (2508.15036) attacks expert activations but via *local* microarchitectural channels (cache occupancy, Pageout+Reload, TLB) requiring co-located code; **GATEBLEED** (2507.17033) needs on-core AMX timing; KV/prompt-cache cross-tenant timing is established (Early-Bird 2409.20002, prompt-cache audits 2502.07776, RAG KV side-channel 2606.21842) but targets the *KV/prompt* cache, not the *expert* cache. Systems work confirms expert offloading + caching is standard and not studied for leakage (2511.05814 in-depth caching/prefetch analysis; OD-MoE 2512.03927; ELDR 2607.00466). **Differentiation: remote, latency-only, expert-cache channel.** Verdict (executor, UNADJUDICATED): **novel, PROCEED**.

**Idea 2 — Matched dense-control safety audit.** Closest: a black-box DeepSeek-MoE-vs-GPT-dense robustness comparison (2506.18543) and MoE-RBench (2406.11353) compare *different* models, not matched-lineage siblings under an identical ablation budget; every routing-attack paper (SAFEx, GateBreaker, L³, Unsafe Routes) omits a dense control. **Differentiation: same-lineage MoE/dense pair + matched active-param ablation-budget sweep.** Verdict (executor, UNADJUDICATED): **differentiable but partially approached — PROCEED WITH CAUTION**; tighten the contribution to the matched-control methodology.

**Idea 3 — Safety regression from benign pruning.** Closest: Unauthorized Compression (2511.19480) couples pruning with *malicious* fine-tuning to bypass licensing; MAESTRO (2607.08601) reports a "Safety" *utility* score, not jailbreak ASR. **Differentiation: safety-agnostic efficiency pruning, no re-alignment, jailbreak-ASR as the metric.** Verdict (executor, UNADJUDICATED): **novel, PROCEED**.

No idea was eliminated as already-published; all three survive the search half. Re-run under a registered cross-model reviewer (or `— reviewer: manual` with a non-Claude model) to obtain the recordable acquittal.

### Full-text re-verification (round 2, 2026-10-01)

> **Method fix**: round-1 novelty calls above were made from titles/abstracts and over-indexed on topic similarity. This round reads the *body* (threat model, assumptions, experimental setup) of the closest papers via full-text HTML. Three verdicts flipped. Still executor-only / UNADJUDICATED — no cross-model acquittal.

- **Idea 1 → revised UP to ~7/10 (was being treated as partly scooped).** Full text of **SparSEEty** (2608.02995): attacker "controls the majority of privileged system software including the host VMM", uses page-fault / block-I/O / page-allocation channels, targets generic FFN activation sparsity for a single victim, and "assume[s] the attacker has no query access to the victim LLM service". Full text of **MoEcho** (2508.15036): "a malicious co-tenant on the same physical server", experts resident, microarchitectural channels only. **Neither covers Idea 1's weaker attacker** (ordinary API client, query/latency-only, exploiting a cross-tenant expert *offload* cache). Idea 1 is differentiated; its real risk is **feasibility**, not prior art — continuous batching may isolate the timing (2602.07878) and cross-tenant expert-cache sharing is undocumented. Verdict: **novel, PROCEED — feasibility-gated**.
- **Idea 3 → revised UP to ~5–6/10 (round-1 harsh 3/10 was wrong).** Full text of **2609.17515** ("What Breaks Under Pruning in Smart Homes"): measures only smart-home tool-calling accuracy + over-refusal, applies post-pruning SFT healing, uses no adversarial jailbreak benchmark, and does not analyze safety-expert overlap. It does **not** occupy Idea 3's claim (benign efficiency expert-pruning raises jailbreak ASR, no realignment, on distributed checkpoints). Residual weakness: the result is somewhat "expected" from dense 2402.05162 + MoE-adversarial SAFEx — but expected ≠ done. Verdict: **differentiable, PROCEED — frame as supply-chain audit + overlap mechanism**.
- **Idea 2 → revised DOWN to ~5/10.** Full text of **MoE-RBench** (2406.11353): it *does* use a matched dense control — "T5 … FLOP-matched to Switch Transformer … same activated parameter size" — and finds MoE "competitive to … similar-sized dense models" on harmful questions. It does **not** do ablation-budget refusal-collapse experiments. So Idea 2's only surviving sliver is the ablation-budget comparison on *modern* upcycled pairs (Switch/T5 is dated). Verdict: **partially occupied — PROCEED WITH CAUTION, narrow to the ablation-budget methodology**.

### New directions from the under-explored privacy quadrant (full-text verified)

Hard search found the **MoE memorization / training-data privacy** quadrant nearly empty (`MoE AND memoriz* AND (dense OR compare)` → 0 arXiv hits). Full text of the two closest papers confirms the gap: **RAPTOR** (2609.05770) is a *defense* (private training) that does not compare MoE-vs-dense leakage and explicitly *freezes/hides* the router; **SHAPOOL** (2510.13451) merely *uses* MoE to build cheap shadow models against *dense* targets, and does not treat routing as a membership signal. Two additions:

- **New A — the "privacy tax" of sparse scaling (~6–7/10).** Measure whether a matched-resource MoE leaks more training data than its dense twin, via **verbatim extraction + canary regurgitation** (not classic MIA, which is weak at web scale). Substrate exists: strictly-equal-resource MoE/dense pairs (2506.12119). Low-risk,底座现成, publishable either direction.
- **New B — routing-steered training-data extraction (~7/10, composition-novelty).** Compose two established-but-unjoined facts: MoE concentrates memorized knowledge in a low-entropy expert backbone (2601.08383; 2410.19034) **+** routing is input-steerable black-box (RouteHijack 2605.02946 / Misrouter 2605.04446). Hypothesis: biasing routing toward a target expert *amplifies* verbatim extraction of that expert's memorized data vs a routing-agnostic baseline. Input-only, falsifiable; `routing AND MoE AND extract*` returned no on-topic privacy work.

### Round-2 ranking (full-text-grounded, UNADJUDICATED)
1. **New B** routing-steered extraction — composition-novel, open, black-box.
2. **Idea 1** remote expert-cache timing — differentiated; feasibility-gated.
3. **New A** MoE privacy tax — open, substrate ready.
4. **Idea 3** benign-pruning safety regression — salvaged; frame as supply-chain audit.
5. **Idea 2** matched dense-control — partially occupied by MoE-RBench; narrow to ablation-budget.

## External Critical Review

> **Status: `REVIEW_UNAVAILABLE`.** `/research-review` is a reviewer-bearing phase that must route to a non-Claude senior-reviewer model (Codex `gpt-6-astra` at xhigh, or `— reviewer: manual`/`oracle-pro`). None is registered here, so **no external review was obtained** and no `accept` receipt is written (the final gate will report this phase BLOCKED). What follows is **executor self-critique**, explicitly NOT a cross-model verdict — it cannot substitute for the external read, which contributes the independence the acceptance gate requires. It is recorded so a human (or a later run with a reviewer) has the strongest objections already surfaced.

**Executor self-critique of the top idea (Idea 1), strongest objections first:**
1. **Timing-noise realism (the make-or-break risk).** Production MoE serving batches many tokens and uses dropless token-choice; per-token latency may be dominated by batch effects, scheduling jitter, and network RTT, not expert-cache misses. *Cheapest discriminating experiment*: before any attack claim, measure the raw hit-vs-miss latency distribution for a single offloaded expert on one GPU and compute its separability (d′ / AUROC) under realistic concurrency. If AUROC ≈ 0.5 at realistic batch sizes, the attack is dead and the paper pivots to "offloading is incidentally private" — still a result.
2. **Threat-model plausibility.** The attack needs an attacker co-tenant on a shared expert cache; many deployments isolate tenants or keep all experts resident on large GPUs. Scope the claim to the memory-constrained / edge / single-GPU-offload regime that papers like OD-MoE/SlimCaching explicitly target, and say so up front.
3. **Decoder transfer.** The 2602.04105 routing→text decoder assumes *clean* expert-selection sequences; timing recovers a *noisy, partial* trace. The real contribution is end-to-end reconstruction under noise, so the pilot must inject realistic trace corruption into the decoder, not assume clean routing.

**On Ideas 2 & 3**: both are LOW/MEDIUM-risk measurements whose main reviewer objection is "incremental." The defense is framing: Idea 2's value is *disciplining* a confused literature with the first matched control (a methodological contribution), and Idea 3's value is a concrete *supply-chain* warning with an actionable audit. Neither hypothesis is up for rewriting; the reviewer risks pick the next test, they do not add components.

**Bottom line (executor, UNADJUDICATED):** none of the top 3 is argued for abandonment; Idea 1 is the highest-information bet and its timing-separability micro-pilot is the single cheapest experiment that could kill or confirm the whole line. This is a recommendation to a human, not a cleared gate.

## Refined Proposal (top idea)
- Proposal: `refine-logs/FINAL_PROPOSAL.md`
- Experiment plan: `refine-logs/EXPERIMENT_PLAN.md`
- Tracker: `refine-logs/EXPERIMENT_TRACKER.md`
- Working contract: `idea-stage/docs/research_contract.md`

## Next Steps
- [ ] Register a non-Claude reviewer (Codex MCP `gpt-6-astra`, or `— reviewer: manual` with a non-Claude model) and **re-run Phases 3–4** to obtain the recordable cross-family acquittals that clear the evidence gate.
- [ ] On a GPU host: run `EXPERIMENT_PLAN.md` Block 0 (CPU simulator ceiling) then Block 1 (hit/miss separability micro-pilot) — the cheapest kill/confirm of Idea 1.
- [ ] `/run-experiment` to deploy; `/experiment-bridge` to implement against the plan.
- [ ] Or `/research-pipeline` for the full end-to-end flow once compute + reviewer are available.

<!-- ARIS_IDEA_DISCOVERY_EVIDENCE_GATE:START -->
## Evidence Gate
**Status:** BLOCKED

The workflow is not complete. Required stage evidence is missing:
- BLOCKED: novelty-check review evidence missing (status=done)
- BLOCKED: research-review review evidence missing (status=done)
<!-- ARIS_IDEA_DISCOVERY_EVIDENCE_GATE:END -->

## Lead idea (2026-10-02): Cross-layer expert sharing vs MoE fault isolation

**Reframe (user-steered)**: move from *horizontal* (within-layer routing, where the field is saturated) to *vertical / cross-layer* structure — specifically **experts reused across depths** (cross-layer expert sharing). Framed as robustness / fault-isolation science, not an attack.

**Claim**: the "MoE is robust / sparsity gives graceful degradation" folklore (2210.10253, 2601.14792) and "early-layer experts are redundant" (2606.10703) all assume **one expert per layer**. Cross-layer-shared MoE (CS-MoE 2609.22199, MoRE 2609.18176, MoUE 2603.04971, MoEUT 2405.16039; limiting case = the always-on shared expert in DeepSeek/Qwen) violates that premise, so a single-expert corruption's **blast radius should scale with its depth-reuse count, not 1/N** — fault isolation collapses.

**Full-text novelty verification (round 3)**: all efficiency papers above evaluate only utility, never robustness/fault. 2609.02404 ("Shared Routing Geometry") establishes cross-layer routing coupling but is **mechanistic only, layer-isolated models (OLMoE/Phi), no ablation, no robustness**. 2606.10703 ("causal audit") finds early-layer redundancy but **per-layer / layer-isolated**. 2601.14792 robustness claim is on standard MoE, **no single-expert blast radius, no cross-layer reuse**. → the security/robustness consequence of cross-layer expert sharing is unoccupied.

**Four-question gate**: Q1 robustness folklore has an unstated layer-isolation scope condition; Q2 single-expert ablation blast-radius vs depth-reuse count, shared vs matched isolated; Q3 strongest methods (2210.10253 / 2601.14792 / 2606.10703 / 2609.02404) all structurally can't answer it (layer-isolated or no ablation); Q4 abandon if shared-layer degrades no worse under matched single-expert perturbation.

**Status**: executor-only / UNADJUDICATED (no cross-model reviewer); inference-only pilot pending (no GPU here; MoEUT/MoRE are 114M–1.15B → low-cost). Conditional **6.5/10** — the strongest-footed and most structurally-original candidate of this run; emerged from the user's vertical/cross-layer steer.
- Proposal: `refine-logs/CROSSLAYER_PROPOSAL.md`
- Experiment plan: `refine-logs/CROSSLAYER_EXPERIMENT_PLAN.md`

**Importance caveat**: shared-layer MoE is still mostly research-stage; the forward-looking framing ("efficiency-via-reuse silently trades away fault isolation") + the flagship always-on shared expert as the limiting case carry the relevance.
