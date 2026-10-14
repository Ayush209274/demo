# LVP code-review guide

A reviewer's map of the LVP prototype: where every change is, what to check in
it, which test enforces each property, and how to reproduce the evidence.
Read `README.md` in this directory for the narrative, results and error log;
this file is the checklist.

State described: HEAD `dfab26f8f3` on `vp-lvp-dev` (base `af72b9ba58`).
Line numbers are for that commit and will drift; function names will not.

## 1. How to see the whole change

```
git log --oneline af72b9ba58..HEAD
git diff --stat af72b9ba58..HEAD            # 12 files, +4663 / -5
git diff af72b9ba58..HEAD -- src/           # the C++ and parameters only
git diff af72b9ba58..HEAD -- tests/lvp/     # workload, runner, checker
```

Review order that matches how it was built: infrastructure `07184627ba` ->
Stage A `28a9991fb1` -> Stage B `3728e6dfb9` -> Stage C `ede81c62bd` -> test
commits. `git show <hash>` per commit keeps each step small.

C++ change size (`af72b9ba58..HEAD`):

| File | Lines |
|---|---|
| `src/cpu/minor/execute.cc` | +494 |
| `src/cpu/minor/lvp.cc` / `lvp.hh` | +224 / +178 |
| `src/cpu/minor/execute.hh` | +80 |
| `src/cpu/minor/dyn_inst.hh` | +58 |
| `src/cpu/minor/BaseMinorCPU.py` | +25 |
| `src/cpu/minor/fetch2.cc` | +18 |
| `src/cpu/minor/exec_context.hh` | +10 |
| `src/cpu/minor/SConscript` | +2 |

## 2. Where each piece lives

| Piece | Location | Review focus |
|---|---|---|
| Predictor table, train, lookup | `lvp.cc` `LastValuePredictor::lookup/train` (~104, ~202) | confidence rules; tag = full PC; threshold gate |
| Constructor validation | `lvp.cc` ctor (~40); `execute.cc` `Execute::Execute` (~64-125) | each `fatal` text; checks run only when relevant |
| Lifecycle counters | `lvp.cc` `predictionIssued … consumerSquashed`, `resetStats` (~118-200) | panic on bad transition; carry-in assigned, not accumulated |
| Eligibility | `execute.cc` `lvpOrdinaryIntLoad`, `lvpEligibleProducer/Consumer` (~1599-1630) | atomics, LR/SC, micro-ops, x0, FP/vector excluded |
| PC window | `execute.cc` `lvpInSpecWindow` (~1632) | half-open, both PCs tested |
| Issue-time map | `execute.cc` `lvpOnIssue`, `lvpInvalidateDests` (~1641-1712) | order: snapshot, erase dests, insert; `instToWaitFor` match |
| Admission | `execute.cc` `lvpSpecAdmissible` (~1714); callers in `commit()` early-issue block and `getCommittingThread()` (~2149) | all original early-issue conditions kept; same predicate in both |
| Operand override | `exec_context.hh` `getRegOperand` (~150-160); `execute.cc` `LVPSpecOperandScope` (~1141) | flag, static inst, index and `RegId` all match; flag cleared on every exit |
| Acceptance | `execute.cc` `executeMemRefInst` (~536) | counted only when really in the LSQ and no fault left |
| Validation + recovery | `execute.cc` `lvpValidateProducer` (~1751), `handleMemResponse` (~365, redirect after `tryToBranch` ~489) | consumer source check first; validation after `completeAcc`; redirect target; `!isStreamChange()` guard |
| Stream change | `execute.cc` `updateBranchData` (~333) | map cleared with the stream number |
| Classifier | `execute.cc` `lvpClassify` (~1780) called just before `inFlightInsts->pop()` in `commit()` | precedence discard > fault > committed; both roles independent |
| Occupancy / drain checks | `lvpCheckOccupancy` (~1859), `isDrained` (~2325) | live counts bounded by queue size; zero when drained |
| Fetch2 hook | `fetch2.cc` `updateBranchPrediction` (~129-149) | before the early return; 2-argument `squash` then `update`; flag cleared |
| Parameters | `BaseMinorCPU.py` | defaults keep baseline unchanged |
| Per-instruction state | `dyn_inst.hh` | defaults inert when LVP is off |

## 3. Review checklist

### Baseline preservation (LVP off)
- [ ] Every hook is behind `enableLVP` (or `enableLVP && lvpSpecAddr`).
- [ ] `lvpSpecAddr` defaults `False`; no ordinary config can hit a `fatal`.
- [ ] New `MinorDynInst` fields are default-initialised and unread when off.
- [ ] Evidence: A vs B statistics identical outside `cpu.lvp.*`; Stage A and B
      statistics identical to the earlier saved runs (753 and 768 statistics).

### Safety of speculation
- [ ] The predicted value is never written to the register file (only returned
      by `getRegOperand` inside the RAII scope).
- [ ] A consumer is admitted only while its producer is the head, `Predicted`,
      in the LSQ with no response.
- [ ] A wrong used prediction cannot commit a consumer: redirect fires at the
      producer's response, before any consumer reaches the head.
- [ ] The consumer source check runs before `completeAcc` and before the
      translation-fault, missing-packet and error-packet arms.

### Recovery
- [ ] Redirect target is `thread->pcState()` after `tryToBranch` (the load's
      next PC); the producer still commits.
- [ ] `discard_inst` is the local computed before the stream change, so the
      producer is classified committed + incorrect, younger work squashed.
- [ ] The Fetch2 hook runs only for `lvpRedirect` + `UnpredictedBranch` and
      cannot intercept ordinary branches (`BranchData::inst` defaults to a
      bubble whose flag is false).
- [ ] Known limit: in `DrainCurrentInst` the redirect is dropped; the hook does
      not run (documented in the code).

### Accounting
- [ ] Each role leaves exactly one terminal state, classified at one site.
- [ ] `confident + predCarryIn == correct + incorrect + predFaulted +
      predSquashed + predInFlight`; same for speculative loads.
- [ ] `mispredictSquashes` counts recovery redirects only.

### Test harness
- [ ] `check-asm` really rejects wrong shapes (branch between loads, wrong
      offset, extra load, gp use, wrong symbol range).
- [ ] The checker's preflight stops before comparing missing, empty or
      malformed outputs.
- [ ] No assertion was weakened: removed lines in each test commit are
      generalisations only (see commit messages and `git show`).

## 4. Which test enforces which property

| Property | Enforced by |
|---|---|
| No architectural change when LVP is on | checksum + identical committed traces A/B/C (every workload) |
| Prediction-only changes nothing | A vs B statistics equal outside `cpu.lvp.*` |
| Speculation really happens before validation | CL log: acceptance tick strictly before producer validation, matched by (thread, exec) |
| Wrong used prediction recovers | phase, fanout, fault, chain: redirect count, event order, consumers squashed and not committed |
| One redirect squashes all consumers of a producer | fanout (2 consumers, 255 redirects) |
| Speculative translation fault stays harmless | fault (32 faulting consumers, `specLoadsFaulted == 0`) |
| Dual role (consumer that is also a producer) | chain (35 complete chains, 27 L3 squashes) |
| Branch-predictor history cleanup | hook checks on the `Branch` log (phase, fanout, fault, chain) |
| Older history retired by `update()` | fault only (one older entry remains each time) |
| Threshold gating | threshold sweep (T1/T2/T3 monotone counters) |
| PC window boundaries | pcwindow (FULL vs NO_PRODUCER vs NO_CONSUMER, `config.ini` boundaries) |
| Constructor validation | invalid-config (nine exact `fatal` messages) |
| Lifecycle bookkeeping | identities with zero live and carry-in, every workload |

## 5. Reproducing the evidence

```
export PATH=/home/ayushgangal/gem5-venv/bin:$PATH
export PYTHON_CONFIG=/usr/bin/python3.12-config
# gem5 must already be built: build/RISCV/gem5.opt

OUT=/tmp/lvp-review tests/lvp/run_lvp_stageb.sh check-asm all
OUT=/tmp/lvp-review tests/lvp/run_lvp_stageb.sh run all      # stable phase fanout fault chain
OUT=/tmp/lvp-review tests/lvp/run_lvp_stageb.sh threshold
OUT=/tmp/lvp-review tests/lvp/run_lvp_stageb.sh pcwindow
OUT=/tmp/lvp-review tests/lvp/run_lvp_stageb.sh invalid-config
```

Each command ends with `all correctness checks passed` (exit 0) or a list of
`FAIL` lines. If the preflight section fails, ignore everything after it.
The `Branch` logs for phase, fanout, fault and chain are large (160 to 330 MB
each); allow a few GB of disk under `OUT`.

Expected headline numbers are in `README.md` section 9. If a number differs,
the checker's exact assertions (not these headline figures) decide pass or fail:
for example `fault` asserts "at least one" faulting consumer and observes 32.

## 6. Where the evidence lives, and its gaps

- **Narrative and results:** `README.md` (untracked). All figures were taken
  from the session transcript, not re-run for it.
- **Raw outputs:** the original run directories under `/tmp` (for example
  `/tmp/lvp-fault-sim`, `/tmp/lvp-chain-sim`, `/tmp/lvp-threshold`) no longer
  exist, so the raw traces, statistics and logs are gone. The session
  transcript is at
  `/home/ayushgangal/.claude/projects/-home-ayushgangal-gem5/d775bd4a-50a6-4e8f-974a-bc6994f67d20.jsonl`.
- **To keep real outputs for the review:** re-run section 5 with `OUT` under a
  durable directory (not `/tmp`) and keep `*.rc`, `*.out`, each `stats.txt`
  and `config.ini`, and the checker output. Traces and `Branch` logs can be
  dropped after the checker passes.
- **Never exercised by a real run:** the `panic_if` safety stops (consumer
  source check, producer-side checks) have only been source-verified; the
  checkers were tested with synthetic logs, not by mutating the C++; drain,
  SMT, idling-enabled, interrupts, mid-run statistics resets.

## 7. Questions worth asking in review

1. Is the 2-argument `squash` + `update` in the Fetch2 hook the right pairing
   for every predictor type, or only for the default `TournamentBP`? (Only the
   default was exercised; the 2-argument `squash` is generic, but per-predictor
   history restore was not read for each.)
2. `lvpSpecAdmissible` requires the producer to be the head. Is the resulting
   coverage (93.8% of iterations in phase/fan-out, 35 of 348 for L3) acceptable
   for the research claim?
3. The PC window requires producer and consumer inside it. Is that the desired
   semantics for real workloads, or should only the consumer be restricted?
4. Should `DrainCurrentInst` drop the redirect, or should the hook also run
   there?
5. Is the `lvpSpecAddr=False` default and the always-on `panic_if` cost
   acceptable for upstream-style conventions?
