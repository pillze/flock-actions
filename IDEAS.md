# IDEAS.md — flock-challenge-multi/x86

> **Every number produced on the free 4-vCPU GitHub runner is DIRECTIONAL A/B ONLY —
> not comparable to the official 16-vCPU Sapphire Rapids (c7i.4xlarge) score.**
> Only the official runner determines the ranked score.

Target: current record 2,223,779.615 (e-man07, `7552101`, commit `03e70fa`) × 1.05 = **2,334,969**.
Platform promotion gate: ≥ +1%. User bar: ≥ +5%.

## Ideas already used by the record (do not redo)
GFNI affine projections of BLAKE3 XOR layers; AVX-512/VBMI/VPCLMULQDQ; 8/16-lane witness kernels;
selected-root encoding; consumer-local root-tile regeneration; witness-born root tiles;
prepared/persistent NTT twiddles; NTT tail-field packing; masked NTT tails; hand-written .S asm
with register-resident AB leaf sums; non-temporal stores; pre-seed hot waits; SMT pool splits;
round-hand-over core warming; tail stealing; flat claim counters; grind team; allocator coloring
+ prefetch hints; bulk proof-tail encoding; **hugepages (MADV_HUGEPAGE at alloc ≥2 MiB + setup-phase
prewarm + MADV_COLLAPSE for touched buffers + recycled-block re-advise skip — `flock-core/src/lib.rs:608-653`,
`scratch.rs:282-284`, `recycle_alloc.rs:197-200`; kill switch `FLOCK_NO_HUGEPAGES`)**.

## Documented failures (do not repeat)
Alternative ab_inner stream patterns; rounds-3-5 from smaller summary (FS pins all bits — impossible);
NTT fold shift on load unit; 2-CLMUL twiddles; masked-vpermb transpose; witness∥NTT SMT overlap;
256-bit GFNI mixes; lazy-reduction NTT kernels; S-domain partial-fraction projection; F2-linearity
extension reuse (provably impossible); storing b without constant bytes; streamed root tiles via
consumer ring; 8-tile batches / cache-set gaps; 64-load C4 reorder; all knob/queue/staging re-sweeps.

## RE-RANKED untried approaches (after source study 2026-11-15)

### 0. Profile-driven slack hunt on real multi-core hardware (METHOD — primary)
- **Idea:** profile the actual worker (`perf stat/record`) at ranked size on a many-core remote
  box (Freestyle VPS, pending user login), identify phases above their kernel floors, and target
  patches at measured gaps instead of note-derived guesses. The record's own floor claims were
  computed on 16T SPR; profiles on other topologies may expose real slack.
- **Expected gain:** whatever the profiles show. **Risk:** none (read-only method).

### 1. NTT-level algorithmic work reduction (idea #2 in v1)
- **Idea:** structural savings in the additive GF(2^128) NTT / fold pipeline (deep pass ~31.8 ms
  + seed/top ~32.4 ms ≈ 55% of the proof). Bigger effective radix, fewer full passes, fusion of
  adjacent passes that touch the same buffers.
- **Expected gain:** 2–10% if real. **Risk:** high — record authors report phases at port/DRAM
  floors; several near variants already failed. Verify only with oracle + byte-identity + A/B.

### 2. Prover-side encoding/parameter freedom — DOWNGRADED after study
- **Evidence against:** `verify_ligerito` takes `pcs_params: &PcsParams` from the trusted side and
  the README pins "every PCS parameter match that trusted commitment". No candidate-chosen
  parameters found so far. Any residual freedom would have to be inside serialization/encoding
  choices that verify() accepts — needs a dedicated audit of `proof.rs`/`pcs.rs` encoders before
  any patch. **Expected gain:** now assumed ~0–2%. **Risk:** medium.

### 3. Port balancing inside GFNI witness kernels (idea #4 in v1)
- **Idea:** witness phase ~92% port-0; rebalance ternlog/GFNI/load mixes to idle ports.
- **Expected gain:** 1–3%. **Risk:** medium-high (256-bit mixes already failed; narrow variants
  may differ).

### 4. Composition stack of remaining sub-1% pieces (idea #5 in v1)
- **Idea:** small untried micro-optimizations stacked. Cannot reach +5% alone.
- **Expected gain:** 0.5–2%. **Risk:** low per piece, high iteration cost.

## Rule for every attempt
One change per attempt; oracle first (baseline proof bytes); differential tests on boundary sizes;
gate platform-specific code with runtime feature detection + portable fallback; label all
measurements directional.
