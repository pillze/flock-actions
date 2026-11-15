# ATTEMPTS.md — flock-challenge-multi/x86

> **All CI results: directional A/B only, NOT comparable to the official
> 16-vCPU Sapphire Rapids score.** Required record beat: 2,223,780 × 1.05 = **2,334,969**.

## Target
required = current record × 1.05 = 2,223,779.615 × 1.05 = **2,334,968.6**

## Baseline status
- Record code = the benchmark checkout (commit `03e70fa`, submission `7552101` by e-man07).
- Local baseline score: **never measured locally** (WSL attempt OOM-killed at 7.9 GB; local machine
  retired). All measurement now on GitHub Actions (4 vCPU Ice Lake, directional only).
- Record's time (official 16T SPR, from accepted-run notes, ~117 ms/proof):
  seed+top ~32.4 ms | deep NTT ~31.8 ms | ZC round 2 ~16.1 ms | ZC round 1 ~13.4 ms |
  Ligerito open ~7.7 ms (grind ~0.84 ms) | lincheck ~5.6–5.9 ms | rest ≈ rounds 3–5, commits, publish.

## CI phase map (measured 2026-11-15, direct worker, 4 threads, LOG2=18, run 43681831586)
Full prove ≈ **457 ms** at 4 vCPU (~4× the official 117 ms — near-perfect linear scaling):
- ntt: seed+top fused pass **127.8 ms** (28%)
- ntt: deep pass **127.8 ms** (28%)
- zc round2 fused fold **68.7 ms** (15%)
- zc rounds 3+4 composed fold **49.0 ms** (10.7%)
- zc round1 (AB 26.7 ‖ C 5.1) **31.9 ms** (7%)
- open (Ligerito recursive) **22.4 ms** (4.9%) | lincheck **17.1 ms** (3.7%) | tail rounds 11.8 ms
- Implications: NTT/witness kernel phases scale ~linearly with cores (compute-bound) → kernel
  optimizations transfer directionally; scheduling pieces do NOT transfer (different topology).
- Worker internals observed: speculative seed-pipe proved ~38 discarded proofs per direct run
  (stdin pre-buffered) — same cost per prove; irrelevant to the ranked harness flow.

## Attempt log
| # | Idea | Files changed | Safety | Verification done / not done | Result (directional) | Gap to 2,334,969 | Risk | Decision |
|---|------|---------------|--------|------------------------------|----------------------|------------------|------|----------|
| 1 | Hugepage/TLB lever | none (no patch written) | n/a | Source study only | **Discarded pre-implementation**: record already has MADV_HUGEPAGE at alloc ≥2 MiB + prewarm + MADV_COLLAPSE + recycled-block skip (`lib.rs:608-653`, `scratch.rs:282-284`, `recycle_alloc.rs:197-200`); pools prewarm+collapse too (`prewarm_prover`) | n/a | — | discard |
| 2 | Prover/verifier parameter slack | none | n/a | Source study: `verifier.rs:53-128` — `pcs_params` passed in from trusted side; README pins params to trusted commitment | Assumed dead (0–2% residual in encodings only) | n/a | medium | downgraded |
| C0 | **Negative control**: `FLOCK_NO_HUGEPAGES=1` (known-direction regression) vs same-size baseline, 2+10 trials | none (env only) | env only, kill switch shipped with record | verifier accepts all trials | pending (runs 43682642433 vs 43682674317) | n/a | — | validates the CI A/B methodology end-to-end |
