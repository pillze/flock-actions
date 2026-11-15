# ATTEMPTS.md — flock-challenge-multi/x86

> **All CI results: directional A/B only, NOT comparable to the official
> 16-vCPU Sapphire Rapids score.** Required record beat: 2,223,780 × 1.05 = **2,334,969**.

## Target
required = current record × 1.05 = 2,223,779.615 × 1.05 = **2,334,968.6**

## Baseline status
- Record code = the benchmark checkout (commit `03e70fa`, submission `7552101` by e-man07).
- Local baseline score: **never measured locally** (WSL attempt OOM-killed at 7.9 GB; local machine
  retired). Baselines now measured on CI (4 vCPU Ice Lake, directional) and on the user's
  Freestyle VPS (pending access).
- Record's time (official 16T SPR, from accepted-run notes, ~117 ms/proof):
  seed+top ~32.4 ms | deep NTT ~31.8 ms | ZC round 2 ~16.1 ms | ZC round 1 ~13.4 ms |
  Ligerito open ~7.7 ms (grind ~0.84 ms) | lincheck ~5.6–5.9 ms | rest ≈ rounds 3–5, commits, publish.

## Attempt log
| # | Idea | Files changed | Safety | Verification done / not done | Result (directional) | Gap to 2,334,969 | Risk | Decision |
|---|------|---------------|--------|------------------------------|----------------------|------------------|------|----------|
| 1 | Hugepage/TLB lever | none (no patch written) | n/a | Source study only (no build/run) | **Discarded pre-implementation**: record already has MADV_HUGEPAGE at alloc ≥2 MiB + prewarm + MADV_COLLAPSE + recycled-block skip (`lib.rs:608-653`, `scratch.rs:282-284`, `recycle_alloc.rs:197-200`) | n/a | — | discard; move to profile-driven hunt |
| 2 | Prover/verifier parameter slack | none | n/a | Source study: `verifier.rs:53-128` shows `pcs_params` passed in from trusted side; README pins params to trusted commitment | Assumed dead (0–2% residual in encodings only) | n/a | medium | downgraded; audit only if profile hunt stalls |
