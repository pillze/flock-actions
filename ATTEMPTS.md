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

## BLOCKER - CI runner is not a valid A/B environment (2026-11-15)
Run `43676621687` (bench.yml, no patch) completed its Benchmark step successfully:

| metric | value |
|---|---|
| score | **21,042.188** compressions/s |
| median trial | 12.458 s |
| threads | 4 |
| proof bytes | 438,863 |
| verified | **true** |

Runner hardware from the job log:
```
Model name: AMD EPYC 7763 64-Core Processor
avx512f: NO   avx512dq: NO   avx512vbmi: NO   gfni: NO   vpclmulqdq: yes
Mem: 15Gi
```

**Diagnosis:** GitHub's `ubuntu-latest` pool landed on an **AMD EPYC 7763 (Zen 3)**, which has
**no AVX-512 and no GFNI**. The record code has **138 `avx512f`-gated functions** plus GFNI
kernels; on this runner every one of them falls through to the scalar/portable fallback.
That is why 262,144 compressions take 12.46 s here vs ~0.118 s on the official SPR box -
a **~106x gap**, which no 4-vs-16-vCPU difference can explain. The 4x thread count accounts
for at most ~4x.

**Consequence:** every A/B measured on this runner is measuring the fallback path, not the
optimized path that the official score reflects. A patch that improves the AVX-512/GFNI kernels
would show **zero or negative** signal here, and a patch that helps the fallback would show
signal that is meaningless for ranking. **CI A/B is therefore worthless for this task** until
the runner has AVX-512+GFNI (i.e. a self-hosted SPR/Ice Lake runner or a VPS on such a CPU).

Correctness signal from CI is still usable: `verified: true` with a 438,863-byte proof
(under the 500 KB cap) confirms the record checkout builds and verifies end-to-end.

## BLOCKER - Freestyle VPS not accessible from this account
```
$ npx freestyle@latest vm ssh vm-fe95dfd9bc074430b4d53d893d8445c3 --team acct-35d244fef6a647c3ae7fa6c0caefad20
Error: No accessible team matches "acct-35d244fef6a647c3ae7fa6c0caefad20".
Teams you can use:
  name           team id                               account id
  GLITCH's Team  b1e383f8-0cbf-4599-9e9c-d56deab236cc  acct-00d9bd3174e24a94ba52c4dacc2fbbd2
```
`npx freestyle@latest vm list` on the authenticated account returns **"No VMs."**

So the CLI is logged into a **different Freestyle account** (`acct-00d9bd31...`, "GLITCH's Team")
than the one owning the target VM (`acct-35d244fe...`). Per the hard limits I did not attempt to
log in or switch accounts. **CPU/RAM/OS of the VPS: NOT VERIFIED** - could not connect.

## Attempt log
| # | Idea | Files changed | Safety | Verification done / not done | Result (directional) | Gap to 2,334,969 | Risk | Decision |
|---|------|---------------|--------|------------------------------|----------------------|------------------|------|----------|
| 1 | Hugepage/TLB lever | none (no patch written) | n/a | Source study only (no build/run) | **Discarded pre-implementation**: record already has MADV_HUGEPAGE at alloc >=2 MiB + prewarm + MADV_COLLAPSE + recycled-block skip (`lib.rs:608-653`, `scratch.rs:282-284`, `recycle_alloc.rs:197-200`) | n/a | - | discard; move to profile-driven hunt |
| 2 | Prover/verifier parameter slack | none | n/a | Source study: `verifier.rs:53-128` shows `pcs_params` passed in from trusted side; README pins params to trusted commitment | Assumed dead (0-2% residual in encodings only) | n/a | medium | downgraded; audit only if profile hunt stalls |
| 3 | Establish CI baseline + validate CI as A/B rig | none | read-only | Bench run 43676621687 completed, `verified: true`, score 21,042.188 | **CI rig invalid**: runner is Zen 3, no AVX-512/GFNI -> measures fallback path only (106x gap) | n/a | - | **CI A/B retired** for ranking work |
| 4 | Establish VPS as profiling/A-B host | none | read-only | `vm ssh` failed: team/account mismatch; `vm list` -> "No VMs" | **Blocked**: VM not owned by authenticated Freestyle account | n/a | - | **blocked on user** |

## Rule for every attempt
One change per attempt; oracle first (baseline proof bytes); differential tests on boundary sizes;
gate platform-specific code with runtime feature detection + portable fallback; label all
measurements directional.

## Open questions for the user
1. Freestyle: which account owns `vm-fe95dfd9bc074430b4d53d893d8445c3`? The CLI here is on
   `acct-00d9bd3174e24a94ba52c4dacc2fbbd2` ("GLITCH's Team") and sees no VMs.
2. CI: `ubuntu-latest` is giving Zen 3 (no AVX-512/GFNI). Options: (a) self-hosted runner on
   AVX-512 hardware, (b) a VPS on Ice Lake/Sapphire Rapids, (c) accept that no remote A/B is
   possible and work from source analysis only.
