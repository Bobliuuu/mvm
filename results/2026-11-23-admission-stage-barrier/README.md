# Admission reads staged successor words before they are stored (no_recycle, staged after launch)

`mvm_admit_record` (`include/mvm_exec.cuh`) copies a record from the mapped ring into `S.stage`, each thread storing the
16-byte chunks `q, q + B, ...`. The successor pass then reads the record's successor words with `__ldcg(re + e)`, at
indices other threads stored. The only `__syncthreads()` between the copy and those reads sat inside `if (!S.no_recycle)`
(the free-ring pop). So in a run with `no_recycle` whose record is not prestaged, a fast warp can read a successor word
before the warp that stages it has stored it. The word then holds whatever the buffer held before: BAD_TASK, a wrong
successor, or a stall until TIMEOUT. The CLI prestages one-record runs and never takes this path. Library users who set
`no_recycle`, and `bench/hop/chainprof`, do.

This branch makes the barrier unconditional: after the staging pass and the slot pop, whichever ran. That adds one
barrier per record only where neither ran before (prestaged `no_recycle` runs).

## Tests

- `tests/gpu/test_exec.cu`, suite `stage`: 160 seeds x {16 workers, all SMs}. Each is a single-record `no_recycle` run
  of 2,000-5,500 tasks submitted after `start()` (so not prestaged), checked against the CPU reference
  (`MVM_TEST_QUICK`: 16 seeds).
- `MVM_TEST_STAGE_SKEW=<ns>`: a negative-test hook in the staging pass, in the style of `MVM_DIST_NO_BP_POLL`, that
  makes odd warps store late. Built with it, the suite must fail unless a barrier orders the staged record before the
  passes that read it.

## Results

All builds use `tests/gpu/build_core.sh` without the extra-ops hook. `tests/gpu/mvm_ops_test.def|.cuh` are on no
branch, so the `types` suite is left out.

| Run | Without this fix | With this fix | Raw |
|---|---|---|---|
| A100-SXM4-40GB MIG 3g.20gb (42 SMs), `stage`, 5 runs x {bucket, FIFO} | 3,200 / 3,200 runs pass | 3,200 / 3,200 pass | `raw/fasrc-a100/stage_no_skew.log` |
| same, then dag, recycle, versions, admission, litmus | 8,454 checks, 0 failed | 8,454 checks, 0 failed | same |
| same GPU, `-DMVM_TEST_STAGE_SKEW=20000` | **3 / 3 runs fail: TIMEOUT (697 / 3,500 done)** | 2 / 2 pass, 320 / 320 each | `raw/fasrc-a100/stage_skew20000.log` |
| H100 80GB HBM3, `stage` (quick) under a happens-before race detector (NVBit) | 49 racing instruction pairs, all the staging `STG.E.128` against the successor `LDG.E.STRONG.GPU` (`__ldcg`) | 0 | `raw/modal-h100/` |

The skew job hit its 30-minute limit during the fourth unfixed run, since each run waits out the stall timeout.
Unaided, the window is too short for these GPUs to hit: a block's warps reach the handoff nearly together. That is why
no test caught it. The detector reports the missing ordering from one run.

In `raw/`, host working folders read `$WORK` and node names `gpu-node`. The H100 logs are as the runs wrote them.
