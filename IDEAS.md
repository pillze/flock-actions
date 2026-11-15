# IDEAS.md — flock-challenge-multi/x86

> **Every number produced on the free 4-vCPU GitHub runner is DIRECTIONAL A/B ONLY —
> not comparable to the official 16-vCPU Sapphire Rapids (c7i.4xlarge) score.**
> Only the official runner determines the ranked score.

Target: current record 2,223,779.615 (e-man07, `7552101`, commit `03e70fa`) × 1.05 = **2,334,969**.
Platform promotion gate: ≥ +1%. User bar: ≥ +5%.

## Ideas already used by the record (do not redo)
GFNI affine projections of BLAKE3 XOR layers; AVX-512/VBMI/VPCLMULQDQ; 8/16-lane witness kernels;
selected-root encoding; consumer-local root-tile regeneration (direct 16-lane emission);
witness-born root tiles; prepared/persistent NTT twiddles; NTT tail-field packing; masked NTT tails;
hand-written .S asm with register-resident AB leaf sums; non-temporal stores; pre-seed hot waits;
SMT pool splits (AB/C arms); round-hand-over core warming; tail stealing; flat claim counters;
grind team; allocator coloring + prefetch hints; bulk proof-tail encoding. 202 feature switches,
all defaults measured best.

## Documented failures (do not repeat)
Alternative ab_inner stream patterns; rounds-3-5 from smaller summary (FS pins all bits — impossible);
NTT fold shift on load unit; 2-CLMUL twiddles; masked-vpermb transpose; witness∥NTT SMT overlap;
256-bit GFNI mixes (clock effect < loss); lazy-reduction NTT kernels; S-domain partial-fraction
projection; F2-linearity extension reuse (provably impossible); storing b without constant bytes;
streamed root tiles via consumer ring; 8-tile batches / cache-set gaps; 64-load C4 reorder; all
knob/queue/staging re-sweeps.

## Ranked untried approaches

### 1. Hugepage/TLB lever (START HERE — targets the biggest bottleneck)
- **Idea:** `madvise(MADV_HUGEPAGE)` + 2 MB-aligned, 2 MB-multiple recycled allocations for the
  big buffers (448 MiB `ab_inner`, ~369 MiB root tiles, NTT codewords, witness planes). The deep
  NTT pass (~31.8 ms of ~117 ms) and round-one ab_inner reads are bandwidth/latency-bound;
  4 KB pages mean ~100k+ page-walks per pass that 2 MB pages collapse by ~512×.
- **Expected gain:** 1–4% (uncertain — if the official runner already runs THP=always the gain
  is ≈0; if madvise-mode, real. Free check: never mentioned in any of the 8 accepted-run notes read).
- **Risk:** low — madvise failure is a silent no-op; no arithmetic or proof change; fully portable.
- **Verification:** byte-identical proofs on 3+ seeds vs baseline; paired interleaved A/B runs
  (4 vCPU runner); kernel-level sanity via page-size introspection in logs.
- **Editable surface:** allocator code under `crates/flock-prover/src` (+ any buffer producers in
  `crates/flock-core/src`). Both inside allowed paths.

### 2. NTT-level algorithmic work reduction
- **Idea:** find structural savings in the additive GF(2^128) NTT (layer schedules, twiddle reuse,
  fewer passes over data). Biggest single lever (deep 31.8 ms + seed/top 32.4 ms ≈ 55% of proof).
- **Expected gain:** 2–10% if real. **Risk:** high (record authors report phases at port/DRAM
  floors; several variants already failed).
- **Verification:** full oracle equality (ClosedExact vs Generic witness byte-equality test) +
  proof identity + A/B.

### 3. Prover/verifier parameter slack
- **Idea:** audit what the pristine verifier actually pins vs what the prover may choose
  (folding schedule encodings, batched publications, serialization order) — any prover-side
  freedom that cuts timed work without changing verify() acceptance.
- **Expected gain:** unknown; 0–5%. **Risk:** medium (must be provably verifier-accepted;
  byte-identity will intentionally NOT hold if encoding changes — needs verifier-pass evidence
  instead, which requires full trusted runs).
- **Verification:** full benchmark.sh runs (verifier accepts every trial).

### 4. Port balancing inside GFNI witness kernels
- **Idea:** witness phase is ~92% port-0 saturated; reorder/alternate instruction mixes
  (ternlog vs GFNI vs loads) to use idle ports.
- **Expected gain:** 1–3%. **Risk:** medium-high (256-bit mixes already failed; narrow variants may differ).
- **Verification:** proof identity + phase timings + A/B.

### 5. Composition stack of remaining sub-1% pieces
- **Idea:** combine small untried micro-optimizations. Cannot reach +5% alone; only valuable
  stacked on top of a real structural win. Also capped by ±0.6% run noise (official) and larger
  runner noise here.
- **Expected gain:** 0.5–2%. **Risk:** low per piece, high submission cost.

## Rule for every attempt
One change per attempt; oracle first (baseline proof bytes); differential tests on boundary sizes;
gate platform-specific code with runtime feature detection + portable fallback; label all
measurements directional.
