# ATTEMPTS.md — flock-challenge-multi/x86

> **All CI results: directional A/B only, NOT comparable to the official
> 16-vCPU Sapphire Rapids score.** Required record beat: 2,223,780 × 1.05 = **2,334,969**.

## Target
required = current record × 1.05 = 2,223,779.615 × 1.05 = **2,334,968.6**

## Baseline status
- Record code = the benchmark checkout (commit `03e70fa`, submission `7552101` by e-man07).
- Local baseline score: **never measured** (WSL attempt OOM-killed at 7.9 GB; local machine retired
  from all work per user instruction). Baseline measurement now happens on GitHub runners
  (4 vCPU Ice Lake — directional only).
- Where the record's time goes (official-runner phase data from accepted-run notes, ~117 ms/proof):
  seed+top ~32.4 ms | deep NTT ~31.8 ms | ZC round 2 ~16.1 ms | ZC round 1 ~13.4 ms |
  Ligerito open ~7.7 ms (grind ~0.84 ms) | lincheck ~5.6–5.9 ms | rest: rounds 3–5, commits, publish.

## Attempt log
| # | Idea | Files changed | Safety | Verification done / not done | Result (directional) | Gap to 2,334,969 | Risk | Decision |
|---|------|---------------|--------|------------------------------|----------------------|------------------|------|----------|
| 0 | (none) — establish CI baseline | none | n/a | pending first CI run | pending | pending | — | — |
