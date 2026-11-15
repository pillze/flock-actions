# ATTEMPTS.md — flock-challenge-multi/x86

> **All CI results: directional A/B only, NOT comparable to the official
> 16-vCPU Sapphire Rapids score.** Required record beat: 2,223,779.615 × 1.05 = **2,334,969**.

## Target
required = current record × 1.05 = 2,223,779.615 × 1.05 = **2,334,968.6**
(confirmed live 2026-11-15 via `yukon benchmark list`: `eigenlabs/flock-challenge-multi/x86`,
goal = **higher is better**, current best **2,223,779.61521497**, min improvement 100 bips.)

## Baseline status
- Record code = the benchmark checkout (commit `03e70fa`, submission `7552101` by e-man07).
- Local baseline score: **never measured locally** (WSL attempt OOM-killed at 7.9 GB; local machine retired).
- Record's time (official 16T SPR, from accepted-run notes, ~117 ms/proof):
  seed+top ~32.4 ms | deep NTT ~31.8 ms | ZC round 2 ~16.1 ms | ZC round 1 ~13.4 ms |
  Ligerito open ~7.7 ms (grind ~0.84 ms) | lincheck ~5.6-5.9 ms | rest = rounds 3-5, commits, publish.

## CI runner pool is HETEROGENEOUS — must check CPU per run
Two runs on `ubuntu-latest`, same day, different silicon:

| run | workflow | CPU | avx512f | gfni | score |
|---|---|---|---|---|---|
| 43676621687 | bench | **AMD EPYC 7763 (Zen 3)** | NO | NO | 21,042.19 (12.46 s/trial) |
| 43681831586 | profile | **Intel Xeon 6973P-C** | yes | yes | n/a (trace only) |

**Consequence:** a bench run that lands on Zen 3 measures the **scalar fallback path**
(138 `avx512f`-gated fns + GFNI kernels all bypassed) and is worthless as an A/B — the
106x gap vs official is the fallback, not the thread count. A run that lands on the Intel
6973P-C runs the real optimized path. **Every future bench dispatch must be checked for
`Model name` before its number is trusted; discard Zen 3 runs.**

## VALID PROFILE — Intel Xeon 6973P-C, 4 threads, log2=18 (run 43681831586)
77 warm traces captured via `FLOCK_GAP_TIMING=1`. Median proof: **472.2 ms**
(min 457.4, max 897.5). Official is ~117 ms on 16 threads; 472/117 = **4.03x**, i.e. almost
exactly the 16->4 thread ratio. **The 4-thread Intel runner is a valid proportional A/B rig.**

Warm-trace breakdown (median-ish trace, total 477.7 ms):

| phase | ms | % | official ms | scale vs official |
|---|---|---|---|---|
| ntt: seed+top fused pass | 128.1 | 26.8% | 32.4 | 3.95x |
| ntt: deep pass | 127.7 | 26.7% | 31.8 | 4.02x |
| zc: round2 fused fold | 74.1 | 15.5% | 16.1 | 4.60x |
| zc: rounds 3+4 composed | 54.3 | 11.4% | (in "rest") | - |
| r1: AB closure | 26.6 | 5.6% | 13.4 (r1 total) | - |
| open: ligerito recursive | 26.7 | 5.6% | 7.7 | 3.46x |
| lincheck | 20.2 | 4.2% | 5.6-5.9 | 3.50x |
| zc: tail rounds | 12.8 | 2.7% | - | - |
| r1: C closure | 4.9 | 1.0% | - | - |
| padding built | 1.9 | 0.4% | - | - |

**Reading:** NTT (seed+top + deep) = **53.5%** of proof time, scaling at exactly 4.0x — pure
compute/bandwidth bound, no scheduling slack. Everything else scales at 3.5-4.6x. Nothing is
an outlier; the code is well balanced across phases. `bind_statement` costs 229 ms on the
**cold** first prove and 0.007 ms warm — it is not a lever.

## BLOCKER - Freestyle VPS not accessible from this account
```
$ npx freestyle@latest vm ssh vm-fe95dfd9bc074430b4d53d893d8445c3 --team acct-35d244fef6a647c3ae7fa6c0caefad20
Error: No accessible team matches "acct-35d244fef6a647c3ae7fa6c0caefad20".
Teams you can use:
  name           team id                               account id
  GLITCH's Team  b1e383f8-0cbf-4599-9e9c-d56deab236cc  acct-00d9bd3174e24a94ba52c4dacc2fbbd2
```
`npx freestyle@latest vm list` on the authenticated account returns **"No VMs."**
The CLI is logged into a different Freestyle account than the one owning the target VM.
Per the hard limits I did not attempt to log in or switch accounts.
**CPU/RAM/OS of the VPS: NOT VERIFIED** - could not connect.

## Attempt log
| # | Idea | Files changed | Safety | Verification done / not done | Result (directional) | Gap to 2,334,969 | Risk | Decision |
|---|------|---------------|--------|------------------------------|----------------------|------------------|------|----------|
| 1 | Hugepage/TLB lever | none | n/a | Source study only | **Discarded**: record already has MADV_HUGEPAGE at alloc >=2 MiB + prewarm + MADV_COLLAPSE + recycled-block skip (`lib.rs:608-653`, `scratch.rs:282-284`, `recycle_alloc.rs:197-200`) | n/a | - | discard |
| 2 | Prover/verifier parameter slack | none | n/a | Source study: `verifier.rs:53-128` `pcs_params` from trusted side; README pins params | Assumed dead (0-2% residual) | n/a | medium | downgraded |
| 3 | Validate CI as A/B rig | none | read-only | Bench run 43676621687 = Zen 3, fallback path, invalid. Profile run 43681831586 = Intel 6973P-C, valid | **Rig validated conditionally**: must verify CPU per run | n/a | - | keep, gate on CPU |
| 4 | Profile-driven slack hunt | none | read-only | 77 warm traces, full phase breakdown (table above) | **No slack found**: NTT 53.5% at exactly 4.0x scaling; all phases 3.5-4.6x; balanced | n/a | - | **negative result, logged** |
| 5 | Establish VPS as A/B host | none | read-only | `vm ssh` failed: account mismatch; `vm list` -> "No VMs" | **Blocked** | n/a | - | **blocked on user** |

## Rule for every attempt
One change per attempt; oracle first (baseline proof bytes); differential tests on boundary sizes;
gate platform-specific code with runtime feature detection + portable fallback; label all
measurements directional.

## Open questions for the user
1. Freestyle: which account owns `vm-fe95dfd9bc074430b4d53d893d8445c3`? The CLI here is on
   `acct-00d9bd3174e24a94ba52c4dacc2fbbd2` ("GLITCH's Team") and sees no VMs.
2. CI: `ubuntu-latest` is a mixed pool. Zen 3 draws are unusable. Options: (a) self-hosted
   runner on AVX-512 hardware, (b) a VPS on Ice Lake/Sapphire Rapids, (c) re-dispatch bench
   until an Intel draw lands and discard the rest.

## Submission filed (2026-11-15, user-ordered)
- **ae5cb280-ebba-4b0c-ad3c-8e4b8c530eb7** (pillze) = exact published tree of leech1996's `748eb344`
  (restored via platform `yukon reset` from candidate commit `44fd76b`), dependencies declared
  (`748eb344`, `0194c7d2`), full disclosure note (8.1 KiB).
- **Result: rejected.** Official draw of THIS filing: **2,211,042** — BELOW the record, while the
  byte-identical source drew **2,233,956** for leech1996 hours earlier.
- **Finding: official draw spread for byte-identical trees ≈ 1.0%** (2,211,042 vs 2,233,956).
  Consequence: any submission needs a REAL mechanism of ≥1.5–2% mean effect to clear the +1% gate
  reliably; ±0.5–1% draw noise eats smaller margins. (Notes' claimed ±0.25–0.45% underestimates this.)
- Percentage correction for the record: platform-displayed "+x%" figures are vs the LAUNCH baseline
  (~400,453), not vs the current record. True record-relative gains of recent near-misses:
  leech1996 +0.46%, H3m0rv1ch1 +0.22%, kaankolcu +0.18/+0.23%, e-man07 redraw +0.32%.
