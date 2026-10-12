# Load-Value Prediction (LVP) in gem5 MinorCPU

> Research prototype for **gem5 23.0.0.1**, **RISC-V MinorCPU**, branch
> `vp-lvp-dev`. This file is the complete project record: what LVP does here,
> how it is built, the order the work was done in, how every part is tested,
> every result, and every mistake or surprise along the way (each with a small
> example).

## Contents

| | |
|---|---|
| [1. Summary](#1-summary) | one-page overview and results at a glance |
| [2. Background and idea](#2-background-and-idea) | what LVP is, why load-to-load |
| [3. Design](#3-design) | how the prototype works |
| [4. Parameters](#4-parameters) | configuration contract |
| [5. Repository map](#5-repository-map) | files changed and their roles |
| [6. Commit history](#6-commit-history) | the ten commits |
| [7. The work in sequence](#7-the-work-in-sequence) | step by step, with findings |
| [8. Test suite](#8-test-suite) | workloads, runs, checks |
| [9. Results](#9-results) | every number, per test |
| [10. Errors and lessons](#10-errors-surprises-and-lessons) | what went wrong and why |
| [11. Running the tests](#11-running-the-tests) | commands |
| [12. Environment](#12-environment-notes) | build and tooling |
| [13. Limits and gaps](#13-limits-and-untested-areas) | what is not covered |
| [14. Glossary](#14-glossary) | terms used |

---

## 1. Summary

**What was built.** A last-value load predictor inside MinorCPU's `Execute`
stage that lets a dependent load compute its address from a *predicted* value
and go to memory before the load it depends on returns. The prediction is
validated when the real data arrives. A wrong prediction that was actually
used triggers a squash-and-refetch recovery, and the branch predictor's
history is cleaned up so it does not drift.

**How it was verified.** A freestanding RISC-V test program (`ldld.c`) is built
into five workload variants. Each is run with LVP off, prediction-only,
speculation on, and speculation on with a debug log. A Python checker compares
the committed instruction traces byte for byte, checks independently computed
checksums, and checks the debug log event by event (matched by thread and
instruction identity, not by counts). Three configuration tests (confidence
threshold, PC window, invalid settings) reuse the fault binary. Every checker
was also tested with synthetic faulty inputs. Nothing was ever weakened to make
a run pass.

### Results at a glance

| Test | Checksum | Used wrong predictions → recoveries | Speculative loads (issued / committed / squashed / faulted) | Speedup of C over B | Checks |
|---|---|---|---|---|---|
| stable | 114688 | 0 → 0 | 16,380 / 16,380 / 0 / 0 | 0.87% (25,720 cycles) | 67 |
| phase | 147456 | 255 → 255 | 15,615 / 15,360 / 255 / 0 | 0.81% (24,172 cycles) | 91–92 |
| fan-out | 246290604769280 | 255 → 255 (2 consumers each) | 31,230 / 30,720 / 510 / 0 | 0.90% (27,540 cycles) | 105–106 |
| fault | 2848 | 32 → 32 (all 32 consumers faulted speculatively) | 156 / 124 / 32 / 0 | none (equal cycles) | 100 |
| chain | 2848 | 27 → 27 | accepted: 348 L2 + 35 L3; 356 committed, 27 squashed, 0 faulted | 0.10% (9,422 cycles) | 106 |
| threshold | 2848 | prediction only | - | - | all pass |
| pcwindow | 2848 | FULL 32; others 0 | FULL 156 / 124 / 32 / 0; others 0 | - | 79 |
| invalid-config | - | 9 of 9 configurations refused | - | - | 37 |

The speed-ups are small because the benchmark overlaps one short cached
access with a long miss; each is a single deterministic data point, not a
general claim. The aim was a correct, thoroughly tested mechanism.

### Key findings

1. **Releasing the scoreboard alone is a no-op** with the default memory unit,
   so the project feeds the predicted value into a dependent load's address.
2. **A load-raised redirect bypasses branch-predictor cleanup** unless
   `Fetch2` is hooked; the hook is proven on real logs for 255 + 255 + 32
   recoveries.
3. **Recovery is not one per pointer change** (chain: 27 recoveries for 32
   changes); tests assert bounds, never equality with the number of changes.
4. **Most failures were in the test harness, not LVP**: loader layout, `gp`
   relaxation, missing trace flags, a shell `||` disabling `set -e`, wrong
   assumptions about gem5's log format (section 10).

---

## 2. Background and idea

A load that misses the cache makes everything that depends on it wait. A
*last-value predictor* remembers what a load (identified by its PC) returned
last time. If it returned the same value several times in a row, we guess it
will return it again and let a younger instruction use the guess before the
real data arrives.

The prototype does this in the narrowest form that is still genuine:
**load-to-load address speculation**.

```
ld a4, 0(a5)     # producer: loads a pointer      (slow: cache miss)
ld a4, 24(a4)    # consumer: needs a4 for its address
```

Without LVP the consumer waits for the producer. With LVP the predictor
supplies a value for `a4`, the consumer computes its address from it, and the
two memory accesses overlap. When the producer's real data arrives the guess
is checked: right, the early load is kept; wrong, everything younger than the
producer is discarded and fetched again.

Why only load-to-load? In Minor, instructions execute in order at the head of
the in-flight queue, so an ALU dependent gains nothing from a predicted value
(it still commits behind the load). The only real win is overlapping a second
*memory access*.

---

## 3. Design

Code lives in `src/cpu/minor/`.

### 3.1 Life of a prediction

```
 issue()                  commit() / LSQ                    handleMemResponse()
 ───────                  ──────────────                    ───────────────────
 lookup(pc)               producer P is head, in LSQ,       real value returns
   │ confident?           no response yet                      │
   ▼                          │                                ▼
 P: state=Predicted        consumer C admitted early         compare with prediction
 map[reg] = {P, value}     C's address uses predicted         │            │
   │                       value (RAII scope)               correct      incorrect
 C issues later:               │                              │            │ and used by C
 snapshot map[src]             ▼                              ▼            ▼
 (if producer ==           C: state=Accepted             C keeps going   redirect to P's
  instToWaitFor)                                                         next PC, squash
                                                                         everything younger
```

### 3.2 Components

- **Predictor** (`LastValuePredictor`): direct-mapped, indexed by `pc >> 1`,
  full-PC tag, last value, 2-bit confidence counter. Training only on loads
  that commit: match → +1 (max 3); different value → replace value, confidence
  0; first sight → install at confidence 0. A lookup predicts only if the tag
  matches and `confidence >= lvpConfThreshold`.
- **Per-thread speculative map** (`lvpSpecMap`): keyed by flattened `RegId`
  (not a bare integer-register index), each entry holds producer sequence
  number, producer PC and predicted value.
- **Issue-time order** (`lvpOnIssue`, once per issued instruction):
  1. snapshot the source, only if the entry's producer equals the
     instruction's `instToWaitFor` and both PCs are inside the PC window;
  2. erase the entries of registers this instruction writes;
  3. insert its own confident prediction.

  This order stops `ld a0,0(a0)` consuming its own prediction. An intervening
  ALU writer erases the entry, and `instToWaitFor` would mismatch anyway.
- **Admission** (`lvpSpecAdmissible`, shared by the early-issue block of
  `commit()` and by `getCommittingThread()`): the producer is the head of
  `inFlightInsts`, is `Predicted`, is in the LSQ, has no response; the
  candidate is the oldest unissued memory op of the same thread and stream.
  Every original early-issue condition is kept (LSQ capacity, bubble,
  `inLSQ`, `canEarlyIssue`, stream, barriers via `instToWaitFor`).
- **Operand interception**: RAII `LVPSpecOperandScope` sets `lvpSpecActive`
  only around `commitInst` for the admitted load. `getRegOperand` returns the
  predicted value only if the flag, static instruction, operand index and
  `RegId` all match. The register file is never written with a predicted value.
- **Acceptance**: the consumer lifecycle starts in `executeMemRefInst` only
  after the request is really in the LSQ (`inLSQ` set and no fault left after
  translation-fault suppression).
- **Validation** (`lvpValidateProducer`): after `completeAcc` succeeds the real
  value is compared, the table trained, and Correct/Incorrect recorded. First
  thing in `handleMemResponse`, a consumer's architectural source register is
  checked against the value it speculated with (`panic_if`, defence in depth).
- **Recovery**: if the prediction was wrong *and used*, `lvpRedirect` is set
  and `UnpredictedBranch` is raised to `thread->pcState()` after `tryToBranch`
  (the load's next PC), when no stream change exists. `streamSeqNum` increments,
  the map is cleared, and every younger instruction (accepted consumers and
  queued responses included) is discarded by the existing stream checks. The
  producer commits normally and is classified Incorrect.
- **Fetch2 hook**: at the head of `Fetch2::updateBranchPrediction`, before the
  early return for non-control instructions, `lvpRedirect` + `UnpredictedBranch`
  calls `squash(fetchSeqNum, tid)` then `update(fetchSeqNum, tid)` (the
  2-argument forms; the 4-argument squash asserts the queue front is a
  branch). Without it, younger branch-predictor history and RAS state would be
  orphaned, because a load is never `triedToPredict`.

### 3.3 One classification site

Every instruction leaves `inFlightInsts` at exactly one place, the `pop()` in
`commit()`. `lvpClassify` runs just before it, using that iteration's own
`discard_inst`, `fault` and `committed_inst` locals (precedence: **discard,
then fault, then committed**, because `committed_inst` is also true for a
faulting response). It classifies the two roles independently, since one
instruction can be both a consumer and a predicted producer.

Producer states: `None → Predicted → {Correct, Incorrect, Faulted, Squashed}`.
Consumer states: `None → Accepted → {Committed, Faulted, Squashed}`.

### 3.4 Always-on checks

`panic_if` is used for every LVP safety check because plain `assert` is
compiled out of `gem5.fast` (it is active in `gem5.opt`, which was proven by an
earlier abort from a plain assert). Checks: popped front equals the classified
instruction; neither role classified twice; validation recorded only for a
predicted instruction; live counts never underflow or exceed queue occupancy;
live counts are 0 whenever the pipeline reports drained; consumer source check.

### 3.5 Statistics (`cpu.lvp.*`)

| Statistic | Meaning |
|---|---|
| `lookups` | eligible predictor lookups at issue (may later be squashed) |
| `confidentPredictions` | lookups that returned a prediction |
| `trainings` | non-discarded, fault-free eligible loads that trained the table |
| `correct`, `incorrect` | validated predictions |
| `predFaulted`, `predSquashed` | confident predictions that faulted / were discarded before validation |
| `predInFlight`, `predCarryIn` | live predictions; live count at the last reset |
| `specLoadsIssued` | consumers accepted into the LSQ |
| `specLoadsCommitted/Faulted/Squashed` | terminal outcomes of accepted consumers |
| `specInFlight`, `specCarryIn` | live consumers; live count at the last reset |
| `mispredictSquashes` | recovery redirects only (not every incorrect prediction) |

In-flight and carry-in are `statistics::Value`s over plain integers, and
`resetStats()` copies the live counts into the carry-in, so at any dump:

```
confident + predCarryIn == correct + incorrect + predFaulted + predSquashed + predInFlight
issued    + specCarryIn == committed + faulted + squashed + specInFlight
```

---

## 4. Parameters

In `BaseMinorCPU.py`. **Always run with `enableIdling=False`** (section 10.2).

| Parameter | Default | Meaning |
|---|---|---|
| `enableLVP` | `False` | Predictor on (prediction only) |
| `lvpSpecAddr` | `False` | Consumers use predictions. Requires `enableLVP` and `executeAllowEarlyMemoryIssue` |
| `lvpSpecPcBegin`, `lvpSpecPcEnd` | `0`, `0` | Half-open window `[begin, end)`: producer **and** consumer PC must be inside. Both 0 = unrestricted. Requires `lvpSpecAddr` and `begin < end` |
| `lvpTableEntries` | `1024` | Non-zero power of two (checked when `enableLVP`) |
| `lvpConfThreshold` | `3` | 1 to 3 (checked when `enableLVP`) |

Override syntax: `-P 'system.cpu[0].enableLVP=True'` (the CPU is a vector).

The three configurations used everywhere:

| Config | Parameters | Meaning |
|---|---|---|
| A | defaults | LVP disabled (baseline) |
| B | `enableLVP=True`, `lvpSpecAddr=False` | prediction only (control) |
| C | `enableLVP=True`, `lvpSpecAddr=True`, window around `sum_slots()` | speculation |

Invalid combinations are `fatal` at start-up (section 8.4). `lvpSpecAddr`
defaults to `False` deliberately (section 10.3).

---

## 5. Repository map

| Path | Role |
|---|---|
| `src/cpu/minor/lvp.hh`, `lvp.cc` | predictor, lifecycle accounting, statistics |
| `src/cpu/minor/execute.cc`, `execute.hh` | map, admission, interception, validation, recovery, classifier |
| `src/cpu/minor/dyn_inst.hh` | per-instruction LVP state fields |
| `src/cpu/minor/exec_context.hh` | guarded `getRegOperand` override |
| `src/cpu/minor/fetch2.cc` | `lvpRedirect` history-cleanup hook |
| `src/cpu/minor/BaseMinorCPU.py` | parameters |
| `src/cpu/minor/SConscript` | `lvp.cc`, `LVP` debug flag |
| `tests/lvp/ldld.c` | the freestanding workload (variants selected by macro) |
| `tests/lvp/run_lvp_stageb.sh` | builds, assembly-checks, runs, calls the checker |
| `tests/lvp/check_lvp_stageb.py` | all result checks |
| `tests/lvp/README.md` | this file (untracked) |
| `configs/research/` | pre-existing, untracked, untouched (`vp_baseline.py` is empty) |

---

## 6. Commit history

Ten commits on top of `stable` (`af72b9ba58` is the base):

| # | Commit | Content |
|---|---|---|
| 1 | `07184627ba` | Infrastructure: predictor, parameters, statistics |
| 2 | `28a9991fb1` | Stage A: prediction lifecycle accounting (prediction only) |
| 3 | `3728e6dfb9` | Stage B: load-to-load address speculation, `ldld.c`, runner, checker |
| 4 | `ede81c62bd` | Stage C: misprediction recovery, Fetch2 hook, phase workload |
| 5 | `c5228cfab3` | Test: fan-out |
| 6 | `460dbf1d10` | Test: Fetch2 recovery-hook (history cleanup) validation |
| 7 | `befc838d82` | Test: fault (speculative consumer faults) |
| 8 | `3cd465b3aa` | Test: chained loads |
| 9 | `f88e62064a` | Test: confidence-threshold sweep |
| 10 | `dfab26f8f3` | Test: PC window and invalid configurations |

The working rule throughout: **one step, then stop for review.** Nothing was
built, run, staged or committed without being asked; hooks were never
bypassed; no assertion was weakened to make a run pass.

---

## 7. The work in sequence

### 7.1 Investigation and decision

1. Traced a load and a dependent ALU instruction through `Execute::issue`,
   `commit`, `commitInst`, `executeMemRefInst`, `handleMemResponse`, the LSQ
   and the scoreboard. Findings:
   - Operands are read only inside `execute()` / `initiateAcc()` through
     `ExecContext::getRegOperand`, which reads the architectural register file.
   - Minor has no physical registers or bypass values; results are written
     immediately; an instruction executes only at the head of `inFlightInsts`.
2. First design (mine): predict at issue, release the scoreboard, validate at
   the response, squash on a mismatch.
3. An independent audit overturned it: the default memory FU has
   `extraAssumedLat = 2`, so baseline Minor already issues load dependents
   early. Scoreboard release does nothing (section 10.1).
4. Decision: a staged approach ending in a narrow but genuine **Option B**,
   load-to-load address speculation, with a prediction-only control.
   - Supported claims: predicted values generate dependent-load addresses
     before the producer returns; predictions are validated; recovery is by
     squash and refetch.
   - Not supported: ALU or branch consumption, selective replay, FP/vector LVP.

### 7.2 Infrastructure (`07184627ba`)

Added `lvp.hh/.cc`, the `LVP` debug flag, the first three parameters, seven
`MinorDynInst` fields, and a predictor built only when `enableLVP`. A bad table
size or threshold is a `fatal_if`.

- The first build failed in scons configure (section 10.10) and was fixed with
  the existing venv.
- Smoke tests (hello, `RiscvMinorCPU`, `--caches`): the default config aborted
  (section 10.2). With `enableIdling=False`, the LVP-off and `enableLVP=True`
  runs both exited 0 and printed `Hello world!`, with identical `simTicks`
  26,248,000, `simInsts` 5,530 and `numCycles` 52,496. The `cpu.lvp` group
  existed (all zero) only in the enabled run.

### 7.3 Audits before Stage A (counter and safeguard design)

A series of read-only audits, each challenging the previous answer:

| Question | Conclusion |
|---|---|
| When to count a speculative request | at LSQ acceptance in `executeMemRefInst`; `inLSQ` alone is not proof |
| Limiting the operand override | RAII scope plus static-instruction and `RegId` checks |
| Accounting identity | `issued = committed + faulted + squashed + inFlight` (the first form was wrong, 10.5) |
| Recovery PC | `thread->pcState()` after `tryToBranch` is the next PC |
| Same-cycle redirect | guard with `!isStreamChange()`, not `isBubble()` (10.4) |
| Map invalidation | ownership check by producer sequence number; clear on stream change |
| Branch predictor | a load redirect bypasses cleanup; add the Fetch2 hook (10.6) |
| Fetch1 priority | Execute redirects win; the hook stays correct under overlap |
| `DrainCurrentInst` | reachable on RISC-V, but safe (10.7) |
| Trace comparison | commit-only traces are exact; `ExecEnable`+`ExecResult`+`ExecEffAddr` carry no sequence numbers |
| `condIncorrect` | diagnostic only; a re-predicted branch may see different table state |
| Statistics resets | only the startup reset occurs in `se.py`; carry-in design handles any reset |
| `assert` vs `panic_if` | `assert` is active in `gem5.opt` but not `gem5.fast`; use `panic_if` |
| `lvpSpecAddr` default | `False` (10.3) |

### 7.4 Stage A (`28a9991fb1`): prediction-only lifecycle accounting

Separate producer and consumer states, the pop-site classifier, outcome
counters, Value statistics with carry-in, the `panic_if` checks and the
`lvpSpecAddr` contract (6 files, +423/−13). Three more `panic_if` checks (no
role classified twice; validation only with a prediction) were added after
review.

- Build: 21 compilations, 2m54s, no LVP warnings.
- Smoke runs A1, A2 (repeat of A1) and B: all exit 0; A1 equals A2 apart from
  five `host*` lines; A equals B outside `cpu.lvp.*`, including `simTicks`,
  `numCycles` and `branchPred.*`.
- B's counters: lookups 1322, confident 118, trainings 1055, correct 114,
  incorrect 2, predSquashed 2, faulted 0; live and carry-in 0.
- Commit hooks (trailing whitespace, black, gem5 style, commit message,
  Gerrit) all passed.

### 7.5 Stage B (`3728e6dfb9`): speculative address consumption

Added: `lvpSpecAddr`; `lvpSpecPcBegin/End` (so tests can confine speculation,
because without recovery any consumed wrong prediction stops the run); the
speculative register map; `lvpSpecAdmissible` in both `commit()` and
`getCommittingThread()`; the operand override; acceptance accounting; the
consumer source check moved to the top of `handleMemResponse`; and a
producer-side `panic_if(Incorrect && lvpUsedBySpec)` as a temporary safety
stop. The test harness (`ldld.c`, runner, checker) was written, then hardened
over many review rounds:

- accepted vs committed speculative loads: the gate requires
  `specLoadsCommitted > 0`; stable-only expectations are labelled as derived
  from the program, not universal;
- `cycles_C < cycles_B` moved from the correctness gate to a performance report;
- whole-binary load inventory, instruction-width and exact symbol-boundary
  checks in `check-asm`;
- a preflight stage so the checker never compares empty or missing files.

Getting the first real run took five fixes (section 10.8). Before the commit,
the Stage A regression was repeated with the new binary: all 753 statistics
identical to the saved Stage A run, and all 768 identical to the saved B run
(host fields aside). The commit was first rejected by the `black` hook
(section 10.15) and succeeded after restaging.

### 7.6 Stage C (`ede81c62bd`): recovery

`lvpValidateProducer` now returns `!correct && lvpUsedBySpec`;
`handleMemResponse` raises the redirect after `tryToBranch`; added the
`lvpRedirect` field, the Fetch2 hook, `mispredictSquashes`, new log lines
(Recovery redirect, Prediction outcome, Speculative load outcome) and the
`-DLVP_PHASE` workload with a phase checker group. Before the build the runner's
error propagation was fixed (section 10.9). Build 3m08s with only pre-existing
`units.hh` template warnings. Stable regression and phase run both passed
(section 9).

### 7.7 Robustness pass

One test at a time; each was a read-only audit and coverage matrix, then
test-only changes (no C++), `check-asm`, simulation, commit.

| # | Test | Commit | What it adds |
|---|---|---|---|
| 1 | fan-out | `c5228cfab3` | one producer, two consumers; one redirect squashes both |
| 2 | Fetch2 hook validation | `460dbf1d10` | direct log evidence that predictor history is cleaned |
| 3 | fault | `befc838d82` | a speculative consumer with an unmapped address |
| 4 | chain | `3cd465b3aa` | L2 is both consumer and producer |
| 5 | threshold sweep | `f88e62064a` | confidence thresholds 1, 2, 3 |
| 6 | pcwindow + invalid-config | `dfab26f8f3` | PC window boundaries and the nine fatals |

---

## 8. Test suite

### 8.1 The workload and the assembly gate

All tests run one small freestanding program, `ldld.c`. It has no C library
(the cross toolchain's sysroot is empty), starts at `_start`, and uses `ecall`
directly. `sum_slots()` is the only code that loads from memory, so every
committed load is known. Variants are selected by macro.

Every workload first passes **`check-asm`**: the linked binary is disassembled
and must contain exactly the intended loads, with the intended register
dependencies, no branch or jump between them, only 4-byte instructions, the
exact `sum_slots` symbol range, no `gp`/`x3` use, and a whole-binary load
inventory with nothing outside `sum_slots`. Only then does gem5 run. The
compiler could emit a different shape, and a test of a shape that did not occur
would pass for the wrong reason. The PC window is derived from each binary's
own `nm`, never from an earlier binary.

### 8.2 The four runs

| Run | LVP | Speculation | Records |
|---|---|---|---|
| A | off | - | committed instruction trace |
| B | on | off | committed trace |
| C | on | on, window = `sum_slots` | committed trace |
| CL | on | on | LVP event log (plus `Branch` log for every workload except stable) |

All use `RiscvMinorCPU`, `--caches`, `enableIdling=False`. Trace flags:
`ExecEnable,ExecUser,ExecFaulting,ExecResult,ExecEffAddr,FmtTicksOff`.

### 8.3 Workloads

| Workload | Macro | Pointer behaviour | Checksum (independent) | Load PCs |
|---|---|---|---|---|
| stable | (none) | all 16,384 slots point to one node | 114688 | 2, 16,384 each |
| phase | `LVP_PHASE` | alternates node (7) / node_b (11) every 64 slots | 147456 | 2, 16,384 each |
| fanout | `LVP_PHASE LVP_FANOUT` | phase, plus a second consumer (`v[2]`) | `57344·2³² + 147456 = 246290604769280` | 3, 16,384 each |
| fault | `LVP_FAULT` | 32 cycles with an `mmap`/`munmap`d page | `32·(8·7 + 3·11) = 2848` | 2, 352 each |
| chain | `LVP_CHAIN` | 3 dependent loads, 32 cycles | `32·(8·7 + 3·11) = 2848` | 3, 352 each |

Details:

- **stable**: expect speculation and no wrong prediction.
- **phase**: nodes `v[3]` = 7 and 11; the same producer PC returns different
  pointers, so each block boundary has a confident but stale prediction.
- **fanout**: nodes `(v[2],v[3])` = (5,7) and (2,11). The checksum packs the
  sum of `v[3]` in the low half and the sum of `v[2]` shifted by 32 in the high
  half, so each consumer's contribution is checked independently. An empty
  `asm` statement keeps the pointer, `hi` and `lo` live at once so neither
  consumer overwrites the base register.
- **fault**: each of 32 cycles does `mmap(0x20000000, 4096, RW,
  PRIVATE|FIXED|ANONYMOUS)`, stores `v[3]=7` into it (first touch allocates the
  page), `sum_slots` over 8 slots into it (trains the predictor), a store-only
  256 KB sweep (evicts the cache), `munmap`, then `sum_slots` over 3 slots into
  node B (`v[3]=11`). The first load of the last call is predicted to point
  into the now unmapped page. Syscall results are checked; failure exits 1.
- **chain**: `L1 = ld a4,0(a5)`, `L2 = ld a4,8(a4)`, `L3 = ld a4,24(a4)`. Each
  cycle: `root->next = A` (7), 8 iterations, `root->next = B` (11), a cache
  sweep, 3 iterations. Slots, root and nodes are 32 KB apart so they share one
  set of the 64 KB 2-way L1D (512 sets of 64 B): three lines in two ways, so
  every access misses and L2 stays outstanding long enough for L3 to be
  speculated. Speculation is one level deep (L3 never uses L1's prediction).

### 8.4 Configuration tests

| Mode | Runs | Purpose |
|---|---|---|
| `threshold` | TA (off), T1, T2, T3 (prediction only, thresholds 1, 2, 3) on the fault binary | confidence classification in isolation |
| `pcwindow` | BASE, FULL `[P,C+4)`, NO_PRODUCER `[C,C+4)`, NO_CONSUMER `[P,C)` on the fault binary | exact half-open window semantics |
| `invalid-config` | nine bad configurations | every constructor `fatal` is reachable and specific |

These are explicit commands and are **not** part of `all`.

The nine invalid configurations: `lvpSpecAddr` without `enableLVP`; without
`executeAllowEarlyMemoryIssue`; a window without `lvpSpecAddr`; window begin
after end; begin equal to end; `lvpConfThreshold` 0 and 4; `lvpTableEntries`
1000 and 0.

### 8.5 Checks applied to every workload

- **Preflight**: the exit status is read from the `.rc` file the runner wrote
  from the real `$?` (not inferred); every required file is a regular,
  readable, non-empty text file; every statistic is present and finite. On any
  failure nothing is compared.
- **Architecture**: same checksum in A, B, C, CL; A and B identical outside
  `cpu.lvp.*`; committed traces of A, B, C identical; C and CL `cpu.lvp.*`
  identical.
- **Trace inventory** (A): one line per committed instruction (equals
  `simInsts`); exact load count and exact load PCs from `load_pcs.txt`.
- **Lifecycle**: both identities of section 3.5 with zero live and carry-in;
  `lookups ≥ confident ≥ validated`; `lookups ≥ trainings ≥ validated`;
  `trainings` equal in B and C.
- **CL log, matched by (thread, exec sequence)**: every acceptance has a
  matching producer validation at a strictly earlier tick; each producer is
  older than its consumer; no duplicates; ids agree with logged exec numbers.
- **Recovery workloads** (phase, fanout, fault, chain): for every redirect the
  producer's outcome is incorrect and the order is acceptance → validation →
  redirect → consumer squash; every consumer of a redirected producer is
  squashed and none commits; every used wrong prediction raised a redirect;
  every committed speculative load used a correct prediction; every squash
  traces to a redirect; log counts equal CL's own statistics.
- **Fetch2 hook** (phase, fanout, fault, chain): section 8.6.
- **Performance** (cycles, IPC, coverage, acceptance-to-validation gap,
  speedup) is reported separately and never changes the exit status.
  Branch-predictor counters are diagnostic only (10.6).

### 8.6 Fetch2 hook validation (from the `Branch` + `LVP` log)

- Every Execute recovery redirect has exactly one `LVP recovery redirect` line,
  matched by (thread, exec), logged after it, for the same PC.
- After a hook line only `[squash sn:N]` lines for that thread may appear, then
  `Committing branches until sn:N`. Every removed entry is younger than `N`,
  removal is youngest first, and `predHist.size()` falls by one per removal.
- The next `History entry added` for that thread has `sn > N` and
  `predHist.size(): 1`. One entry can satisfy at most one hook; two hooks
  before an entry fail as ambiguous; a malformed entry line is a parser problem
  and cannot be replaced by a later valid one.
- At least one hook removed entries (non-vacuous). The log cannot tell a squash
  that removed nothing from one that was not visible, so that is never claimed.
- Fault workload only: after the last removal exactly one older entry remains
  (size 1), so `update()` has an older entry to retire.

### 8.7 How the checkers themselves were tested

Every checker change was exercised with synthetic result directories and logs,
one injected fault at a time, plus the good case:

| Checker | Examples of injected faults (each must fail with a specific message) |
|---|---|
| preflight | aborted runs, missing/garbage exit status, directory instead of file, unreadable or binary file, `nan`/`inf`/non-integral statistic |
| phase / fanout | consumer committed, missing or early redirect, count mismatch, duplicate events, second consumer late |
| fault | no faulting consumer, source outside the page, consumer committed or "faulted", producer correct |
| chain | missing L1→L2 or L2→L3 link, premature L3, L3 not squashed, L2 not committed, wrong producer PC |
| hook | missing hook, wrong sequence number, older entry removed, size not falling by one, shared push, malformed push |
| threshold | wrong label, speculation on, non-monotone counts, unequal lookups |
| pcwindow | boundary off by 4, FULL not speculating, NO_* speculating, carry-in |
| invalid-config | exit 0, missing `.rc`, absent/wrong/extra fatal, panic, traceback, program output |
| assembly gate | branch between loads, overwritten base, wrong offset, extra load, gp use, wrong symbol range |
| runner | 38 checks with stand-in compiler and gem5: build failure, assembly failure, stale results, bad arguments |

Synthetic harnesses had their own bugs at times; those were fixed in the
harness, never by relaxing a checker. Every new checker was also re-run on the
saved real results of the earlier workloads to prove no regression.

---

## 9. Results

"N checks" is the checker's PASS count with 0 FAIL.

### 9.1 Stable (Stage B, repeated after Stage C)

- Checksum 114688 in A, B, C, CL. A/B/C traces identical (the Stage C traces
  are byte-identical to Stage B's, and the stable binary itself is
  byte-identical). 163,971 trace lines equal `simInsts`; 32,768 loads at
  `0x10158` and `0x1015c`, 16,384 each.
- C: 16,380 speculative loads issued and committed; 0 squashed, 0 faulted, 0
  incorrect; correct 32,760; confident 32,761; 1 prediction squashed (a
  wrong-path load past the array end); live and carry-in 0.
- All 16,380 acceptances matched their producer validation, all strictly
  earlier. Branch-predictor counters identical in A, B, C, CL (lookups 32,836,
  condPredicted 32,807, condIncorrect 31). 67 checks.
- Performance:

| | cycles | IPC |
|---|---|---|
| A (off) | 2,960,967 | 0.0554 |
| B (prediction only) | 2,960,967 | 0.0554 |
| C (speculation) | 2,935,247 | 0.0559 |

C is **25,720 cycles (0.87%) faster** than B (1.57 cycles per committed
speculative load). Coverage 16,380 of 16,384 (100%). Acceptance-to-validation
gap: min 21,500, median 50,500, max 338,500 ticks (about 50 cycles of overlap).
Each iteration takes about 180 cycles, mostly the producer's miss, which is why
the saving is small.

### 9.2 Phase (Stage C)

- Checksum 147456; traces A/B/C identical; exactly 32,768 loads at two PCs.
- C: confident 31,741; correct 30,720; incorrect 510; predSquashed 511; issued
  15,615; committed 15,360; squashed 255; faulted 0; **mispredictSquashes 255**
  (one per block boundary), `0 < 255 ≤ 510`.
- Why incorrect is 510: both the producer's and the consumer's predictions are
  wrong at each of the 255 boundaries, but only the producer's is ever used.
- Worked example: consumer accepted at tick 573093500 with source `0x11020`;
  producer validated at 573147000 (predicted `0x11020`, actual `0x11000`);
  redirect to PC `0x10160`; consumer squashed at 573147500; the refetch loaded
  the correct address (traces identical, checksum correct).
- Branch predictor: condIncorrect 280, BTBUpdates 148, RASIncorrect 0 in all
  runs; lookups 59,528 (C) vs 58,253 (A): 1,275 extra = 5 per redirect, the
  refetched younger branches. 91 to 92 checks, depending on checker version.
- Performance: C 24,172 cycles (0.81%) faster than B; 15,360 of 16,384
  iterations (93.8%) used a speculative load. Recovery cost did not erase the
  gain.

### 9.3 Fan-out

- Checksum 246290604769280 (fields 147456 and 57344 checked separately); each
  of the three load PCs (`0x10154`, `0x10158`, `0x1015c`) commits 16,384 times.
- 15,615 producers each had two accepted consumers; 31,230 issued = 30,720
  committed + 510 squashed + 0 faulted; 255 redirects, each accounting for two
  consumers, both accepted before validation; incorrect 765 (255 × 3);
  predSquashed 767; mispredictSquashes 255; live and carry-in 0.
- Lookups 59,284 vs 58,519: 3 extra per redirect (phase had 5; fewer loop
  branches are in flight with the longer loop body). 105 to 106 checks.
- Performance: C 27,540 cycles (0.90%) faster than B; coverage 93.8%.

### 9.4 Fetch2 hook on real logs

| | Phase | Fan-out | Fault |
|---|---|---|---|
| hooks = redirects | 255 | 255 | 32 |
| parser problems / ambiguous / missing entry | 0 / 0 / 0 | 0 / 0 / 0 | 0 / 0 / 0 |
| removals per recovery | 5 (all 255) | 3 (all 255) | 3 (once), 5 (29), 7 (twice) |
| history left before `update()` | 0 | 0 | **1** |
| next entry | size 1, 28 younger | size 1, 38 younger | size 1 |
| `CL` trace size | 161,937,593 B, 1,407,004 lines | 194,599,342 B, 1,683,594 lines | 326,708,972 B, 2,778,331 lines |
| RAS-restore lines | none | none | 34 (every hook removed a predicted `ret`) |

(For comparison, the `CL` trace without `Branch` is about 27 MB.)

In phase and fan-out the squash left the history empty, so **those workloads do
not exercise `update()` retiring older uncommitted entries**; the fault
workload does (one older entry each time). What `update()` does with several
older entries rests on reading `BPredUnit::update()`.

Example (phase, first hook):

```
fetch2: LVP recovery redirect inst: 0/276.16824/67434/218096.218096 pc: 0x10158 (ld)
branchPred: [tid:0] [squash sn:218096] Removing history for [sn:218120] PC 0x10168
branchPred: [tid:0] [squash sn:218096] predHist.size(): 4
 ... three more removals, sizes 3, 2, 1 ...
branchPred: [tid:0] [squash sn:218096] predHist.size(): 0
branchPred: [tid:0] Committing branches until sn:218096]
```

### 9.5 Fault

- Checksum 2848; 704 committed loads, 352 at each of `0x10188`, `0x1018c`;
  A/B/C traces identical (38,791,461 bytes); 100 checks.
- Consumer addresses in A's trace: `0x20000018` read 256 times (32 × 8) and
  `0x11018` (node B) 96 times (32 × 3): every refetched consumer loaded node B.
- **32 faulting speculative consumers accepted, one per cycle**, all with
  source `0x20000000`. Each was accepted before its producer's validation, its
  producer was incorrect, and it was squashed after the redirect; none
  committed; `specLoadsFaulted` is 0. The checker asserts "at least one" (it did
  not assume 32).
- Issued 156 = 124 committed + 32 squashed + 0 faulted; correct 256;
  incorrect 64 (32 producers plus 32 refetched consumers whose prediction `7`
  was wrong against B's `11`); mispredictSquashes 32.
- The 32 architectural faults seen were all the first store into the freshly
  mapped page (`pc 0x10260`), handled by demand paging. No speculative fault
  reached the architectural path.
- Performance: A = B = C = 9,242,313 cycles, **no speedup** (the store sweep's
  fixed memory time probably hides the few cycles saved; not investigated).

How a speculative fault stays harmless (source-verified): SE-mode translation
reads the process page table with no TLB cache and never allocates a page; the
fault is stored in the dynamic instruction's `translationFault` field and only
invoked in `handleMemResponse` for a non-discarded response; after recovery the
consumer is discarded, so its fault is never invoked. The refetched consumer is
a new dynamic instruction, so it cannot inherit the fault. A squashed
instruction's trace record is dropped.

### 9.6 Chain

- Checksum 2848; 527,486 committed instructions, 1,056 loads, 352 at each of
  `0x10194`, `0x10198`, `0x1019c`; L2 always at `0x58008`; L3 at `0x60018` ×256
  and `0x68018` ×96 (one 32768-byte stride apart). 106 checks.
- Accepted L2: 348; accepted L3: 35 (the L3 window is real but rarely open);
  35 complete chains `accept(L2) < validate(L1) < accept(L3) < validate(L2)`;
  L3 never accepted before its L2's producer was validated (no nesting); all 35
  accepted L3 before their L2 validated.
- 27 incorrect L2 predictions used by L3 triggered redirects, all at L2 and none
  at L1; 27 speculative loads squashed; 356 committed; incorrect overall 64 (37
  wrong predictions were never used by a dependent, so caused no recovery).
- **32 pointer changes, only 27 recoveries.** Recovery is not one per change.
- Speedup 9,422 cycles (0.10%).

### 9.7 Threshold sweep (704 lookups and 704 trainings at each threshold)

| | T1 | T2 | T3 |
|---|---|---|---|
| confident predictions | 574 | 446 | 320 |
| correct | 448 | 320 | 256 |
| incorrect | 126 | 126 | 64 |
| predSquashed | 0 | 0 | 0 |

All four runs exit 0, checksum 2848, identical committed traces (same md5),
identical non-LVP statistics, no speculative loads, redirects or carry-in.
`incorrect` is flat from T1 to T2 and drops at T3; only non-increase is
asserted, never strictness or exact totals.

### 9.8 PC window

| | FULL | NO_PRODUCER | NO_CONSUMER |
|---|---|---|---|
| window (`config.ini`) | `0x10188`–`0x10190` | `0x1018c`–`0x10190` | `0x10188`–`0x1018c` |
| lookups | 765 | 704 | 704 |
| trainings | 704 | 704 | 704 |
| confident | 381 | 320 | 320 |
| correct / incorrect | 256 / 64 | 256 / 64 | 256 / 64 |
| specLoadsIssued | 156 | 0 | 0 |
| committed / squashed / faulted | 124 / 32 / 0 | 0 / 0 / 0 | 0 / 0 / 0 |
| mispredictSquashes | 32 | 0 | 0 |

79 checks. FULL's 381 confident = 320 validated + 61 squashed. FULL's non-LVP
statistics differ from BASE by design (speculation changes cycles), so FULL is
compared by trace and checksum; NO_PRODUCER and NO_CONSUMER equal BASE outside
`cpu.lvp.*`.

### 9.9 Invalid configurations (37 checks; all exit 1)

| Case | Fatal text |
|---|---|
| `spec_without_lvp` | `system.cpu.execute: lvpSpecAddr requires enableLVP` |
| `spec_without_early_issue` | `... lvpSpecAddr requires executeAllowEarlyMemoryIssue` |
| `window_without_spec` | `... lvpSpecPcBegin and lvpSpecPcEnd require lvpSpecAddr` |
| `window_begin_after_end` | `... lvpSpecPcBegin (0x10200) must be below lvpSpecPcEnd (0x10100)` |
| `window_begin_equals_end` | `... lvpSpecPcBegin (0x10100) must be below lvpSpecPcEnd (0x10100)` |
| `threshold_zero` / `_four` | `lvpConfThreshold must be in [1, 3] (got 0)` / `(got 4)` |
| `table_not_power_of_two` / `table_zero` | `lvpTableEntries must be a non-zero power of two (got 1000)` / `(got 0)` |

Each output has exactly one fatal marker and no panic, traceback or program
output.

---

## 10. Errors, surprises and lessons

### Quick index

| # | Problem | Where found | Kind |
|---|---|---|---|
| 10.1 | scoreboard release is a no-op | design audit | design |
| 10.2 | `ActivityRecorder` assertion with idling | first smoke run | environment/unknown |
| 10.3 | `lvpSpecAddr=True` default breaks every run | contract audit | design |
| 10.4 | wrong recovery guard (`isBubble`) | recovery audit | design |
| 10.5 | wrong accounting identities | counter audit | design |
| 10.6 | load redirect bypasses branch-predictor cleanup | audit | design |
| 10.7 | `DrainCurrentInst` reachable (my claim was wrong) | audit | design |
| 10.8 | five problems getting Stage B to run | first real runs | harness |
| 10.9 | runner ignored its own failures | review | harness |
| 10.10 | scons/zlib/Python 3.14 | first build | environment |
| 10.11 | command-line mistakes | smoke tests | operator |
| 10.12 | hook-checker weaknesses | review | harness |
| 10.13 | chain surprises | audit and run | test design |
| 10.14 | invalid-config parser | real run | harness |
| 10.15 | pre-commit hooks and formatting | commits | tooling |
| 10.16 | habits that prevented worse errors | - | process |

### 10.1 The first design measured nothing
Releasing the scoreboard when a load is predicted looked like value prediction.
The default memory FU has `extraAssumedLat = 2`, so a load's destination is
never marked "unpredictable", and dependents already reach their FU about 3
cycles after the load issues.

Example: `ld x1; add x2,x1,1`. In the baseline the `add` already issues early;
predicting `x1` changes nothing, and commit still waits for the real data.
**Lesson:** a prediction only matters if something uses the value. That is why
the project feeds the value into a dependent load's address.

### 10.2 `ActivityRecorder` assertion with idling enabled
The first hello smoke run aborted at tick 500: `activity.cc:121
deactivateStage: Assertion 'activityCount >= 0' failed`, in both LVP-off and
LVP-on runs. Setting `system.cpu[0].enableIdling=False` made both pass.
Whether it is a pre-existing gem5 issue was never proven (no unpatched binary
was run with idling). The evidence is that LVP-off and LVP-on runs were
identical with idling off. Every experiment, baseline included, uses
`enableIdling=False`, because mixing would add a confounder. Idling-enabled
behaviour is untested.

### 10.3 `lvpSpecAddr=True` as default would break every run
The plan had `lvpSpecAddr` default `True` plus "requires `enableLVP`, else
`fatal`". With the default `enableLVP=False`, every ordinary MinorCPU run would
hit the fatal. Fixed: default `False`, and an explicit `True` without
`enableLVP` or without `executeAllowEarlyMemoryIssue` is a fatal. The harness
sets `lvpSpecAddr` explicitly in B and C. (The parameter's real name is
`executeAllowEarlyMemoryIssue`; the C++ member is `allowEarlyMemIssue`.)

### 10.4 Wrong recovery guard
Planned `if (branch.isBubble())`. A correctly predicted branch committed
earlier in the same cycle is not a bubble, so recovery would be skipped.
Example: cycle N commits `beq` (predicted right) and then the load validates
wrong; the branch result is non-bubble, yet the load's redirect must win. Fixed
with `!branch.isStreamChange()`. My own earlier claim that loads are always
marked "unpredictable" was also corrected (true only for `extraAssumedLat == 0`).

### 10.5 Wrong accounting identities
- `specLoadsIssued == committed + squashed` ignores faulted and still-in-flight
  requests (nothing flushes the LSQ at exit). Corrected to
  `issued + carryIn = committed + faulted + squashed + inFlight`.
- `correct + incorrect == confidentPredictions` is invalid: a confident
  prediction can be discarded or fault before validation.
- `inLSQ` alone does not prove acceptance: `commitInst` sets it again for a
  non-early load that faulted before any push. Counting happens in
  `executeMemRefInst` after the translation-fault suppression.

### 10.6 The LVP redirect bypasses branch-predictor cleanup
`Fetch2::updateBranchPrediction` returns early for any non-control instruction,
so a redirect raised by a load never calls `squash()`. Younger branches already
predicted keep their history (and RAS entries), are predicted again after the
refetch, then trained with stale histories. This is performance-only but would
distort any branch-predictor comparison. Fixed by the `lvpRedirect` hook.

Also: the claim "branch-predictor counters should match the baseline" was wrong
for run C (refetched branches are predicted again: +5 lookups per redirect in
phase, +3 in fan-out) and right only for B. `condIncorrect` is diagnostic,
because a re-predicted branch may see different table state.

### 10.7 `DrainCurrentInst` is reachable (my earlier claim was wrong)
I first said the drain state was unreachable because RISC-V has no micro-ops.
It is reachable (an outstanding load counts as an access in flight). It is
still safe: the stream number and map clear happen before the redirect is
replaced by a bubble, and the thread PC is architectural for resume. The Fetch2
hook does not run in that case, leaving orphaned predictor entries as any drain
does. Documented, not asserted, not tested.

### 10.8 Getting Stage B to run: five separate problems

1. **Missing admission site (found in audit).** Under the default `RoundRobin`
   thread policy, `getCommittingThread()` does not call `commit()` while the
   head load is outstanding, so the early-issue block never runs and Stage B
   would never speculate. Fixed by sharing `lvpSpecAdmissible` in both places.
2. **ELF loader panic.** All four runs aborted at tick 0:
   `memory_image.hh:70 Segment outside the bounds of the image data`. A
   freestanding program has no initialised data, so the linker put `.bss` in a
   separate LOAD segment whose file offset (0x1000) is past the end of the
   2,408-byte file, and gem5 builds a segment per header and checks
   `offset + size`. Fixed with `-Wl,-n` (one segment).
3. **`gp` was 0.** Next abort at tick 133000: `Page table fault ...
   0xfffffffffffff818`, which is `0 - 2024`. Linker relaxation turned
   `node.v[3] = 7` into `sd a4,-2024(gp)`, and SE mode starts with `gp = 0`.
   My earlier statement that relaxation is harmless was wrong: it only removes
   whole instructions, but it also introduces gp addressing. Fixed with
   `-Wl,--no-relax-gp`, and `check-asm` now rejects any `gp`/`x3` operand (it
   flagged the exact crashing instruction on the old binary).
4. **Empty traces.** Runs exited 0 but A/B/C `trace.txt` were 0 bytes.
   `ExeTracerRecord::traceInst` skips user-mode code unless `ExecUser` is on,
   and `ecall` is a fault whose record is dropped unless `ExecFaulting` is on
   (163,968 lines vs 163,971 `simInsts`). Flags are now
   `ExecEnable,ExecUser,ExecFaulting,ExecResult,ExecEffAddr,FmtTicksOff`.
5. **Checker gave vacuous PASSes.** On the aborted runs it compared two empty
   files as "identical" and then died with `KeyError: 'specLoadsIssued'`.
   Fixed with a preflight stage (8.5), later hardened after finding five more
   silent failure modes: `stats.txt` a directory, unreadable, binary garbage, a
   `nan`/`inf`/non-integral statistic, and `.out` a directory.

### 10.9 The runner ignored its own failures
`run_workload || status=1` runs inside a condition, which turns off `set -e`
for the whole function (a subshell does not turn it back on; verified with a
throwaway script).

Example: delete the binary and break the compile; the run still "passes" on a
leftover binary.

Fixed with one `run-one` process per workload; deletion of the old binary,
disassembly, `.rc`, `.out` and result directories first; argument validation (a
bad workload name used to exit 0 silently); and window reset per workload.
Tested with stand-in compilers and gem5: 38 checks.

### 10.10 Build environment
The first `scons` failed in configure: system SCons 4.11 on Python 3.14
mishandles the arguments of `CheckLibWithHeader('z','zlib.h',...)` (it links
`-ll -li -lb ...`) and reported zlib missing although it is installed.
`SConstruct` was not modified. The existing venv `~/gem5-venv` (Python 3.12.14,
SCons 4.8.1) was used, with `PYTHON_CONFIG=/usr/bin/python3.12-config` set as
an **environment variable** (on the command line it is ignored, and the venv's
own `python3-config` has no `libpython3.12`).

### 10.11 Other command-line mistakes
- `-P system.cpu.enableLVP=True` fails (`'SimObjectVector' object has no
  attribute '_params'`); the CPU is a vector: `system.cpu[0].enableLVP=True`.
- `RiscvMinorCPU` refuses to run without `--caches`.
- A stray `rm -rf /dev/null` in one of my commands was a no-op (not root).
- An early `check-asm fanout` without `OUT=` wrote to the default
  `/tmp/lvp-stage-b/fanout`. No repository file was affected. Always pass
  `OUT=`.

### 10.12 Hook-validation checker weaknesses (fixed one at a time)
- A misleading message was reworded.
- Two hooks sharing one `History entry added` event made the per-hook check
  ambiguous and could let a later recovery mask an earlier one that did not
  leave the history empty. Now at most one hook per thread per entry, and a
  second hook before an entry fails as ambiguous.
- A malformed `History entry added` line was silently ignored, and a later
  valid entry could stand in for it. Now it is a parser problem and clears all
  waiting hooks.
- The `hook_checks` docstring first implied the next entry proves everything
  about `update()`. It was corrected: it shows only the combined effect of both
  calls, and the older-entry retirement is exercised only by the fault workload.

### 10.13 Chain: what the audit and run showed
- A local named `slots` shadowed the global `slots` (renamed `cslots`).
- The assembly checker's unpack assumed one consumer per group; a guard was
  added.
- Whether L3 would ever be accepted was unknown (a tight timing window). It
  was: 35 times, out of 348 L2 acceptances.
- "One recovery per pointer change" was wrong: 27 recoveries for 32 changes,
  because some wrong predictions were never used and some changes found no
  confident prediction. The check is `0 < redirects ≤ incorrect`.
- Speculation is only one level deep: L3 can be accepted only after L1 has
  validated and L2 is the head; the order is L2 accepted, L1 validated, L3
  accepted, L2 validated.

### 10.14 Invalid-config parser assumed the wrong output format
All nine runs exited 1 with the right messages, yet the checker failed all
nine. Real lines look like:

```
src/cpu/minor/execute.cc:103: fatal: system.cpu.execute: lvpSpecAddr requires enableLVP
src/cpu/minor/lvp.cc:50: fatal: fatal condition conf_threshold < 1 || conf_threshold > maxConfidence occurred: lvpConfThreshold must be in [1, 3] (got 4)
```

I had assumed lines start with `fatal:` and that `fatal_if` prints only the
message. Worse, the "no other fatal" and "no panic" checks looked only at line
starts, so they passed with zero matches and would have missed a real one.

Fixed: a marker is `\bfatal: ` anywhere in a line; the `fatal condition ...
occurred:` wrapper is stripped before an exact comparison; exactly one marker
is required; `\bpanic: ` anywhere counts. **Lesson:** read the real output
before writing the parser, and check that "no problems found" is not "looked in
the wrong place". (The `[1, 3]` rendering of a `uint8_t` through `cprintf`,
flagged as unverified, rendered correctly.)

### 10.15 Pre-commit hooks and formatting
- A false alarm: `%d` with `uint64_t` and `%#x` with `RegVal` look wrong, but
  gem5's `cprintf` is type-driven, so they are correct.
- The Stage B commit was rejected by the `black` hook, which reformatted one
  line of `BaseMinorCPU.py` (a three-line parameter joined into one 79-column
  line). I had run black only on the checker. Fixed by restaging that file; the
  commit then passed (black, style, commit message, Gerrit). Hooks were never
  bypassed.
- `black` also reformatted `check_lvp_stageb.py` before the Stage B commit; its
  AST was verified identical and the checker's output on real results was
  byte-identical before and after.

### 10.16 Habits that prevented worse errors
- **Synthetic tests for every checker** (section 8.7).
- Every new checker re-run on the saved real results of earlier workloads.
- Preprocessed `ldld.c` output compared byte for byte with HEAD for every
  earlier workload after each change, and the stable binary compared with the
  Stage B binary.
- **Independent evidence over self-reported logs**: the committed trace
  (effective addresses and results), the checksum and the statistics are
  independent of the LVP log lines; the event-order checks use the log.
- Before each commit: `git diff --check`, `black --check` on the Python,
  `bash -n` on the shell, a read of every removed line, explicit file
  allowlists (never `git add .` or `-A`).

---

## 11. Running the tests

gem5 must already be built (`build/RISCV/gem5.opt`). `OUT=<dir>` chooses where
results go (default `/tmp/lvp-stage-b`).

```
tests/lvp/run_lvp_stageb.sh check-asm [WORKLOAD]   # build and check the disassembly
tests/lvp/run_lvp_stageb.sh run [WORKLOAD]         # check-asm, then A, B, C, CL, then the checker
tests/lvp/run_lvp_stageb.sh threshold              # confidence-threshold test
tests/lvp/run_lvp_stageb.sh pcwindow               # PC-window test
tests/lvp/run_lvp_stageb.sh invalid-config         # invalid-configuration test
```

`WORKLOAD` is `stable`, `phase`, `fanout`, `fault`, `chain` or `all` (the
default). The three configuration tests are not part of `all`. Each workload
writes under `$OUT/<name>`, and `run` starts a separate process per workload so
a failure stops that workload before any simulation while all are attempted.

The checker alone:

```
python3 -I tests/lvp/check_lvp_stageb.py OUT/<name> <mode>
```

`<mode>` is `stable|phase|fanout|fault|chain|threshold|pcwindow|invalid-config`.

Notes:
- The `Branch` logs of phase, fanout, fault and chain are large (160 to 330 MB
  each); each of the four traces of a fault run is about 38 MB.
- Always pass `OUT=` for review runs.
- If a run fails, read the preflight section first; if preflight fails, no later
  PASS line is a result.

---

## 12. Environment notes

```
export PATH=/home/ayushgangal/gem5-venv/bin:$PATH
export PYTHON_CONFIG=/usr/bin/python3.12-config
scons build/RISCV/gem5.opt -j$(nproc)
```

| Build | Compilations | Time |
|---|---|---|
| infrastructure | 1,186 (new Python params regenerate many headers) | not recorded |
| Stage A | 21 | 2m54s |
| Stage B | 20 | 4m01s |
| Stage C | not recorded | 3m08s |

Only the pre-existing `units.hh` template warnings appear, plus the usual
missing libpng and HDF5 notices. The repository's pre-commit `black` is the
cached one (line length 79).

---

## 13. Limits and untested areas

**Not supported by design:** speculation into ALU operations, branches, stores,
atomics, LR/SC, micro-ops or FP/vector loads; selective replay (recovery
refetches everything after the producer); nested speculation (L3 never uses
L1's prediction).

**Not tested (known gaps, not known defects):**

- Drain (`DrainCurrentInst`/`DrainAllInsts`) during recovery, interrupts (none
  in RISC-V SE mode), SMT or multiple threads, idling enabled.
- A statistics reset while work is live. The carry-in design is implemented,
  but only the startup reset occurs in these runs, so carry-ins are only ever
  asserted to be 0; a scheduled-reset wrapper was planned but not written.
- An erroring memory response (not reachable in SE mode), and `update()`
  retiring more than one older history entry.
- Other block sizes, random transition patterns, table aliasing, other table
  sizes.
- The safety stops (consumer source check, the `panic_if`s) have never fired in
  a real run; they are source-verified only. The checkers were exercised with
  synthetic logs, not by mutating the C++.
- Branch-predictor integrity beyond the `Branch` log and statistics.
- Results come from one in-order core and a handful of microbenchmarks; no
  claim is made about real workloads.

---

## 14. Glossary

| Term | Meaning |
|---|---|
| producer | the load whose value is predicted |
| consumer | a younger load whose address depends on the producer's register |
| accepted | a consumer whose request is really inside the LSQ |
| validated | the producer's real value was compared with its prediction |
| used | a prediction that at least one accepted consumer relied on |
| redirect / recovery | `UnpredictedBranch` raised after a used wrong prediction |
| stream change | `streamSeqNum` increment that makes younger work stale |
| squashed | discarded because its stream number is stale |
| carry-in | live predictions or speculative loads at the last statistics reset |
| window | `[lvpSpecPcBegin, lvpSpecPcEnd)` restricting speculation by PC |
| A/B/C/CL | LVP off / prediction only / speculation / speculation with log |
| TA, T1–T3 | threshold-test runs (off, thresholds 1 to 3) |
