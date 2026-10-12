# Load-value prediction (LVP) in MinorCPU

Research prototype for gem5 23.0.0.1, RISC-V MinorCPU, branch `vp-lvp-dev`.
This file is the full record of the project: what was built, in what order,
how it was tested, every result, and every mistake or surprise on the way,
each with a small example.

1. [What LVP does here](#1-what-lvp-does-here)
2. [Design](#2-design)
3. [Parameters](#3-parameters)
4. [Commit history](#4-commit-history)
5. [The work in sequence](#5-the-work-in-sequence)
6. [Test suite](#6-the-test-suite)
7. [All results](#7-all-results)
8. [Errors, surprises and lessons](#8-errors-surprises-and-lessons)
9. [Running the tests](#9-running-the-tests)
10. [Environment notes](#10-environment-notes)
11. [Not supported and not tested](#11-not-supported-and-not-tested)

## 1. What LVP does here

A load that misses the cache makes everything that depends on it wait. A
*last-value predictor* remembers what a load (identified by its PC) returned
last time. If it returned the same value several times in a row, we *guess*
it will return it again and let a younger instruction use the guess before
the real data arrives.

The prototype does this in the narrowest form that is still genuine:
**load-to-load address speculation**. A dependent load (the *consumer*)
computes its address from the predicted value of the load it depends on (the
*producer*) and is sent to the memory system early. When the producer's real
data arrives the guess is checked. Right: the early load is kept and the two
memory accesses overlap. Wrong: everything younger than the producer is
discarded and fetched again.

```
ld a4, 0(a5)     # producer: loads a pointer      (slow: cache miss)
ld a4, 24(a4)    # consumer: needs a4 for its address
```

## 2. Design

Code is in `src/cpu/minor/`: `lvp.hh/.cc`, `execute.cc/.hh`, `dyn_inst.hh`,
`exec_context.hh`, `fetch2.cc`, and the parameters in `BaseMinorCPU.py`.

- **Predictor** (`LastValuePredictor`): direct-mapped table indexed by
  `pc >> 1`; each entry holds the full PC (tag), the last value and a 2-bit
  confidence counter. Training happens only for loads that commit: a match
  adds 1 (max 3); a different value replaces the value and resets confidence
  to 0; first sight installs the value at confidence 0. A lookup returns a
  prediction only if the tag matches and `confidence >= lvpConfThreshold`.
- **Producer**: at issue, a confident lookup attaches the prediction to the
  instruction (`LVPProducerState::Predicted`) and records the destination
  register in a per-thread map (`lvpSpecMap`, keyed by flattened `RegId`,
  holding producer sequence number, producer PC and value).
- **Issue-time map order** (`lvpOnIssue`, once per issued instruction):
  (1) snapshot the source if the entry's producer equals the instruction's
  `instToWaitFor` and both PCs are inside the PC window; (2) erase the
  entries of the registers this instruction writes; (3) insert its own
  confident prediction. The order stops `ld a0,0(a0)` from consuming its own
  prediction.
- **Consumer admission**: in the early-memory-issue block of `commit()` and
  in `getCommittingThread()` (both call `lvpSpecAdmissible`), a dependent
  load may go to the LSQ while its producer is still outstanding if: the
  producer is the head of `inFlightInsts`, is `Predicted`, is in the LSQ,
  has no response yet, and the candidate is the oldest unissued memory op of
  the same thread and stream. Every original early-issue condition stays.
- **Operand interception**: a scoped RAII object (`LVPSpecOperandScope`) sets
  `lvpSpecActive` only around `commitInst` for the admitted load, and
  `ExecContext::getRegOperand` returns the predicted value only when the flag,
  static instruction, index and `RegId` all match. The register file is never
  written with a predicted value.
- **Acceptance**: the consumer's lifecycle (`LVPConsumerState::Accepted`)
  starts in `executeMemRefInst` only after the request is really in the LSQ
  (`inLSQ` set, no fault left after translation-fault suppression).
- **Validation**: in `handleMemResponse`, after `completeAcc`, the real value
  is compared with the prediction, the table is trained, and the outcome
  (Correct/Incorrect) is recorded on the instruction. Before anything else in
  `handleMemResponse`, a consumer's architectural source register is checked
  against the value it speculated with (`panic_if` on mismatch: defence in
  depth).
- **Recovery** (Stage C): a wrong prediction that a consumer actually used
  (`lvpUsedBySpec`) sets `lvpRedirect` and raises `UnpredictedBranch` to
  `thread->pcState()` after `tryToBranch` (the load's next PC) if no stream
  change exists already. The stream number increments, the map is cleared,
  and every younger instruction (including accepted consumers and queued
  responses) is discarded by the existing stream-sequence paths. The producer
  itself commits normally and is classified Incorrect.
- **Fetch2 hook**: at the head of `Fetch2::updateBranchPrediction`, before the
  early return for non-control instructions, an `lvpRedirect` +
  `UnpredictedBranch` calls `branchPredictor.squash(fetchSeqNum, tid)` then
  `update(fetchSeqNum, tid)` (the 2-argument forms; the 4-argument squash
  asserts the queue front is a branch). Without it, younger branch-predictor
  history and RAS state would be orphaned, because a load is never
  `triedToPredict`.
- **Classification at one site**: every instruction leaves `inFlightInsts` at
  one place (the `pop()` in `commit()`). `lvpClassify` runs just before it,
  using that iteration's own `discard_inst`, `fault` and `committed_inst`
  locals (precedence: discard, then fault, then committed), and handles the
  producer and consumer roles independently (an instruction can be both).
- **Always-on checks** (`panic_if`, which survives `gem5.fast`; plain `assert`
  is compiled out only there): popped front equals the classified instruction;
  neither role classified twice; validation recorded only for a predicted
  instruction; live counts never underflow or exceed queue occupancy; live
  counts are 0 whenever the pipeline reports drained; consumer source check.
- **Statistics** (`cpu.lvp.*`): `lookups`, `confidentPredictions`, `trainings`,
  `correct`, `incorrect`, `predFaulted`, `predSquashed`, `predInFlight`,
  `predCarryIn`, `specLoadsIssued/Committed/Faulted/Squashed`, `specInFlight`,
  `specCarryIn`, `mispredictSquashes` (recovery redirects only). In-flight and
  carry-in are `statistics::Value`s over plain integers; `resetStats()` copies
  the live counts into the carry-in, so
  `confident + predCarryIn == correct + incorrect + predFaulted + predSquashed
  + predInFlight` holds at any dump, even after a mid-run stats reset.

## 3. Parameters

In `BaseMinorCPU.py`. Always run with `enableIdling=False` (section 8.2).

| Parameter | Default | Meaning |
|---|---|---|
| `enableLVP` | `False` | Predictor on (prediction only) |
| `lvpSpecAddr` | `False` | Consumers use predictions. Needs `enableLVP` and `executeAllowEarlyMemoryIssue` |
| `lvpSpecPcBegin`, `lvpSpecPcEnd` | `0`, `0` | Half-open window `[begin, end)`; producer and consumer PC must both be inside. Both 0 = unrestricted. Needs `lvpSpecAddr` and `begin < end` |
| `lvpTableEntries` | `1024` | Non-zero power of two |
| `lvpConfThreshold` | `3` | 1 to 3 |

Invalid combinations are `fatal` at start-up (section 6.8). `lvpSpecAddr`
defaults to `False` on purpose (section 8.3).

## 4. Commit history

Ten commits on top of `stable` (`af72b9ba58` is the base):

| # | Commit | Content |
|---|---|---|
| 1 | `07184627ba` | Infrastructure: predictor, parameters, statistics |
| 2 | `28a9991fb1` | Stage A: prediction lifecycle accounting (prediction only) |
| 3 | `3728e6dfb9` | Stage B: load-to-load address speculation (+ `ldld.c`, runner, checker) |
| 4 | `ede81c62bd` | Stage C: misprediction recovery, Fetch2 hook, phase workload |
| 5 | `c5228cfab3` | Test: fan-out |
| 6 | `460dbf1d10` | Test: Fetch2 recovery-hook (history cleanup) validation |
| 7 | `befc838d82` | Test: fault (speculative consumer faults) |
| 8 | `3cd465b3aa` | Test: chained loads |
| 9 | `f88e62064a` | Test: confidence-threshold sweep |
| 10 | `dfab26f8f3` | Test: PC window and invalid configurations |

(`configs/research/` and this README stay untracked; `vp_baseline.py` is an
empty file that was never touched.)

## 5. The work in sequence

The rule throughout: **one step, then stop for review.** Nothing was built,
run, staged or committed without being asked; nothing was bypassed (hooks
included); no assertion was weakened to make a run pass.

### 5.1 Investigation and decision
1. Traced a load and a dependent ALU instruction through `Execute::issue`,
   `commit`, `commitInst`, `executeMemRefInst`, `handleMemResponse`, the LSQ
   and the scoreboard. Operands are read only inside `execute()` /
   `initiateAcc()` (`ExecContext::getRegOperand` reads the architectural
   register file); results are written immediately; Minor has no physical
   registers or bypass values; an instruction executes only when it is at the
   head of `inFlightInsts`.
2. First design (mine): predict at issue, release the scoreboard, validate at
   the response, squash on a mismatch.
3. Independent audit overturned it: with the default memory FU
   (`extraAssumedLat = 2`) baseline Minor already issues load dependents
   early, so scoreboard release is a **no-op** (section 8.1).
4. Decision: staged approach ending in a narrow but genuine Option B,
   **load-to-load address speculation**, with a prediction-only control.
   Supported claims: predicted values generate dependent-load addresses before
   the producer returns, are validated, and recovered by squash and refetch.
   Not supported: ALU or branch consumption, selective replay, FP/vector LVP.

### 5.2 Infrastructure (`07184627ba`)
Added `lvp.hh/.cc`, the `LVP` debug flag, the three first parameters, seven
`MinorDynInst` fields, and a predictor built only when `enableLVP`. Rejects a
bad table size or threshold with `fatal_if`.
- First build failed in scons configure (section 8.10), fixed with the venv.
- Smoke tests (hello, `RiscvMinorCPU`, `--caches`): default config aborted
  (section 8.2); with `enableIdling=False` both LVP-off and `enableLVP=True`
  runs exited 0 and printed `Hello world!`, with identical `simTicks`
  26,248,000, `simInsts` 5,530 and `numCycles` 52,496. The `cpu.lvp` group was
  registered (all zero) only in the enabled run.

### 5.3 Pre-Stage-A audits (counter and safeguard design)
A series of read-only audits, each challenging the previous answer:
counting speculative requests at LSQ acceptance, not at `inLSQ`; limiting the
operand override with an RAII scope; the correct accounting identity
(`issued = committed + faulted + squashed + inFlight`, section 8.5); recovery
PC and same-cycle redirect; ownership-safe map invalidation; the Fetch2
history hook (section 8.6); Fetch1 priority and `DrainCurrentInst` (it is
reachable on RISC-V, but safe, section 8.7); which branch-predictor counters
may legitimately differ; what an execution trace can prove; reset behaviour of
statistics; whether `assert` is active in `gem5.opt` (yes, but `panic_if` is
used for all LVP checks); and the `lvpSpecAddr` default (section 8.3).

### 5.4 Stage A (`28a9991fb1`)
Prediction-only lifecycle accounting: separate producer and consumer states,
the pop-site classifier, outcome counters, Value statistics with carry-in,
the `panic_if` checks, and the `lvpSpecAddr` contract. 6 files, +423/−13.
Three further `panic_if` checks (no role classified twice, validation only
with a prediction) were added after review. Build: 21 compilations, 2m54s,
no LVP warnings. Smoke runs A1, A2 (repeat) and B (`enableLVP=True`,
`lvpSpecAddr=False`): all exit 0; A1 equals A2 apart from five `host*` lines;
A equals B outside `cpu.lvp.*` (including `simTicks`, `numCycles`,
`branchPred.*`). B's counters: lookups 1322, confident 118, trainings 1055,
correct 114, incorrect 2, predSquashed 2, faulted 0, all live and carry-in 0.

### 5.5 Stage B (`3728e6dfb9`)
Added: `lvpSpecAddr`, `lvpSpecPcBegin/End` (PC window, so tests can confine
speculation, because without recovery any consumed wrong prediction stops the
run), the speculative register map, `lvpSpecAdmissible` in both `commit()` and
`getCommittingThread()`, the operand override, acceptance accounting, the
consumer source check moved to the top of `handleMemResponse`, and a
producer-side `panic_if(Incorrect && lvpUsedBySpec)` as the temporary safety
stop. Test harness: `ldld.c` (freestanding), `run_lvp_stageb.sh`,
`check_lvp_stageb.py`. Getting the first real run took five fixes
(section 8.8): a blocking missing admission site, ELF loader rejection,
gp-relative store with `gp = 0`, missing trace flags, and preflight gaps.
Before committing, the Stage A regression was repeated with the new binary:
all 753 statistics identical to the saved Stage A A1 run and all 768 identical
to the saved B run (host fields aside).

### 5.6 Stage C (`ede81c62bd`)
Replaced the producer-side panic with recovery: `lvpValidateProducer` returns
`!correct && lvpUsedBySpec`; `handleMemResponse` raises the redirect after
`tryToBranch`; `lvpRedirect` field; Fetch2 hook; `mispredictSquashes`; new log
lines (Recovery redirect, Prediction outcome, Speculative load outcome);
`-DLVP_PHASE` workload; phase checker group. Before the build the runner's
error propagation was fixed (section 8.9). Build: 3m08s, only the existing
`units.hh` template warnings. Stable regression and phase run both passed
(section 7).

### 5.7 Robustness pass (one test at a time)
Each: read-only audit and coverage matrix, implementation of tests only (no
C++), `check-asm`, simulation, commit.
1. **Fan-out** (`c5228cfab3`): one producer, two consumers.
2. **Fetch2 hook validation** (`460dbf1d10`): direct evidence from the
   `Branch` debug log that the hook runs and cleans predictor history.
3. **Fault** (`befc838d82`): a speculative consumer with an unmapped
   predicted address.
4. **Chain** (`3cd465b3aa`): L1, L2, L3 where L2 is both consumer and
   producer.
5. **Threshold sweep** (`f88e62064a`).
6. **PC window and invalid configs** (`dfab26f8f3`).

## 6. The test suite

All tests run one small freestanding program, `ldld.c` (no C library, because
the cross toolchain's sysroot is empty; starts at `_start`, uses `ecall`
directly), compiled into variants with a macro. `sum_slots()` is the only
code that loads from memory, so every committed load is known.

Every workload first passes **assembly validation** (`check-asm`): the linked
binary is disassembled and must contain exactly the intended loads, with the
intended dependencies, no branch or jump between them, four-byte instructions
only, exact `sum_slots` symbol range, no `gp`/`x3` use, and a whole-binary
load inventory with nothing outside `sum_slots`. Only then does gem5 run. The
compiler could emit a different shape, and a test of a shape that did not
occur would pass for the wrong reason. The PC window comes from each binary's
own `nm`, never from a previous binary.

Workload runs (all `enableIdling=False`, `--caches`):

| Run | LVP | Speculation | Records |
|---|---|---|---|
| A | off | - | committed instruction trace |
| B | on | off (prediction only) | committed trace |
| C | on | on, window = `sum_slots` | committed trace |
| CL | on | on | LVP event log (plus `Branch` log, all workloads except stable) |

### 6.1 stable
All slots point to one node. Expect speculation with no wrong prediction.
16384 iterations, checksum 114688.

### 6.2 phase
Pointer alternates node (`v[3]=7`) and node_b (`v[3]=11`) in blocks of 64
slots. Checksum 147456. Forces used wrong predictions and recovery.

### 6.3 fanout
Phase, with one producer and two consumers (`v[2]`, `v[3]`) from the same
base. Nodes `(5,7)` and `(2,11)`. Checksum is the sum of `v[3]` plus the sum
of `v[2]` shifted left 32, so each consumer is checked independently:
`57344·2^32 + 147456 = 246290604769280`. An empty `asm` statement keeps the
pointer, `hi` and `lo` live together so neither consumer overwrites the base.

### 6.4 fault
32 cycles of: `mmap` page A at `0x20000000` (`MAP_FIXED|PRIVATE|ANONYMOUS`),
store `v[3]=7` into it (first touch allocates it), `sum_slots` over 8 slots
into A (trains the predictor), store-only 256 KB sweep (evicts the cache),
`munmap` A, `sum_slots` over 3 slots into node B (`v[3]=11`). The first load
of the last call gets a confident prediction into the now unmapped page, so
its consumer's translation faults speculatively. Syscall results are checked
and a failure exits with status 1. Checksum `32*(8*7+3*11) = 2848`; each load
PC commits 352 times.

### 6.5 chain
Three dependent loads: L1 (root pointer from a slot), L2 (`root->next`), L3
(`v[3]` of that node). 32 cycles of: `root->next = A` (7), 8 iterations
(trains L2), `root->next = B` (11), cache sweep, 3 iterations. Slots, root and
nodes are 32 KB apart so they share one set of the 64 KB 2-way L1D (512 sets):
three lines in two ways means every access misses and L2 stays outstanding long
enough for L3 to be speculated. Checksum 2848; each load PC commits 352 times;
L1 reads 11 slots 32 times each, L2 one address 352 times, L3 node A 256 and
node B 96 times. Speculation is one level deep: L2 may use L1's prediction
while L1 is oldest; L3 may use L2's prediction while L2 is oldest; L3 is never
accepted on L1's prediction.

### 6.6 threshold (`threshold` command)
The fault binary with LVP off (TA) and prediction-only at thresholds 1, 2, 3
(T1 to T3). In prediction-only mode timing is identical, so every lookup sees
the same table entry in every run and only the threshold filter differs. This
justifies: lookups and trainings equal across thresholds;
`confidentPredictions`, `correct`, `incorrect` never increase as the
threshold rises. Also checks trace, checksum 2848, `config.ini` threshold.

### 6.7 pcwindow (`pcwindow` command)
Fault binary, producer P and consumer C taken from its `load_pcs.txt`:

| Run | Window | Expect |
|---|---|---|
| BASE | LVP off | baseline |
| FULL | `[P, C+4)` | speculation happens |
| NO_PRODUCER | `[C, C+4)` | 0 speculative loads, 0 redirects |
| NO_CONSUMER | `[P, C)` | 0 speculative loads, 0 redirects |

Boundaries are checked in `config.ini`. FULL's non-LVP statistics differ from
BASE by design (speculation changes cycles), so FULL is compared by trace and
checksum; NO_PRODUCER and NO_CONSUMER must equal BASE outside `cpu.lvp.*`.

### 6.8 invalid-config (`invalid-config` command)
Nine configurations that must stop gem5 with a specific `fatal`:
`lvpSpecAddr` without `enableLVP`; without `executeAllowEarlyMemoryIssue`;
window without `lvpSpecAddr`; window begin after end; begin equals end;
`lvpConfThreshold` 0 and 4; `lvpTableEntries` 1000 and 0. Needs a nonzero exit
status and exactly one `fatal` marker with the exact text, and no panic,
traceback or program output. No stats or trace are required (gem5 stops
before running).

### 6.9 Checks applied to every workload
- Preflight first: exit status read from the recorded `.rc` file (not
  inferred), every required file a regular, readable, non-empty text file,
  every statistic present and finite; on any failure nothing is compared.
- Same checksum in A, B, C and CL; A and B identical outside `cpu.lvp.*`;
  A, B and C committed traces identical; C and CL `cpu.lvp.*` identical.
- Trace inventory: one line per committed instruction (equals `simInsts`),
  exact load count and exact load PCs from `load_pcs.txt`.
- Lifecycle identities for producers and speculative loads, zero live and
  carry-in at the end, `lookups >= confident >= validated`,
  `lookups >= trainings >= validated`, B and C `trainings` equal.
- CL log, matched by (thread, exec sequence), not by line counts: every
  acceptance has a matching producer validation, a strictly earlier tick; each
  producer is older than its consumer; no duplicates; ids agree.
- Recovery workloads (phase, fanout, fault, chain): for every redirect the
  producer's outcome is incorrect, event order is acceptance, validation,
  redirect, consumer squash; every consumer of a redirected producer is
  squashed and none commits; every used wrong prediction raised a redirect;
  every committed speculative load used a correct prediction; every squash
  traces to a redirect; log counts equal CL's own statistics.
- Fetch2 hook (phase, fanout, fault, chain): section 6.10.
- Performance (cycles, IPC, coverage, acceptance-to-validation gap, speedup)
  is reported separately and never changes the exit status. Branch-predictor
  counters are diagnostic only (section 8.6).

### 6.10 Fetch2 hook validation (from the `Branch` + `LVP` log)
- Every Execute recovery redirect has exactly one `LVP recovery redirect`
  line, matched by (thread, exec), logged after it, for the same PC.
- After a hook line only `[squash sn:N]` lines for that thread may appear,
  then `Committing branches until sn:N`; every removed entry is younger than
  N, removal is youngest first, and `predHist.size()` falls by one per removal.
- The next `History entry added` for that thread has `sn > N` and
  `predHist.size(): 1`. One such entry can satisfy at most one hook; two hooks
  before an entry fail as ambiguous; a malformed entry line is a parser
  problem and cannot be replaced by a later valid one.
- At least one hook removed entries (non-vacuous). The log cannot tell a squash
  that removed nothing from one that was not visible, so that is never claimed.
- Fault workload only: after the last removal exactly one older entry
  remains (`size 1`), so `update()` has an older entry to retire.

## 7. All results

Where a check count appears it is the checker's PASS count with 0 FAIL.

### 7.1 Stable (Stage B, repeated after Stage C)
- Checksum 114688 in A, B, C, CL. Traces A/B/C identical (Stage C trace
  byte-identical to Stage B's; the binary itself byte-identical). 163,971 trace
  lines equal `simInsts`; 32,768 loads at `0x10158` and `0x1015c`, 16,384 each.
- C: 16,380 speculative loads issued and committed; 0 squashed, 0 faulted,
  0 incorrect; correct 32,760; 32,761 confident; 1 prediction squashed (a
  wrong-path load past the array end); live and carry-in 0.
- All 16,380 acceptances matched to their producer validation, all strictly
  earlier. Branch-predictor counters identical in A, B, C, CL
  (lookups 32,836, condPredicted 32,807, condIncorrect 31). 67 checks pass.
- Performance: cycles A = B = 2,960,967, C = 2,935,247; IPC 0.0554 to 0.0559;
  **C is 25,720 cycles (0.87%) faster than B** (1.57 cycles per committed
  speculative load); coverage 16,380 of 16,384 (100%); acceptance to
  validation gap min 21,500, median 50,500, max 338,500 ticks (about 50 cycles
  of overlap). Each iteration takes about 180 cycles, mostly the producer's
  miss, which is why the saving is small.

### 7.2 Phase (Stage C)
- Checksum 147456; traces A/B/C identical; exactly 32,768 loads at two PCs.
- C: confident 31,741; correct 30,720; incorrect 510 (producer and consumer
  predictions are both wrong at each of the 255 block boundaries; only the
  producer's is ever used); predSquashed 511; issued 15,615; committed 15,360;
  squashed 255; faulted 0; **mispredictSquashes 255** (one per block
  boundary). `0 < 255 <= 510`.
- Worked example: consumer accepted at tick 573093500 with source `0x11020`;
  producer validated at 573147000 (predicted `0x11020`, actual `0x11000`);
  redirect to PC `0x10160`; consumer squashed at 573147500; refetch loaded the
  correct address (trace identical, checksum correct).
- Branch predictor: condIncorrect 280, BTBUpdates 148, RASIncorrect 0 in all
  runs; lookups 59,528 (C) vs 58,253 (A): 1,275 extra = 5 per redirect, the
  refetched younger branches. 91 checks pass.
- Performance: C 24,172 cycles (0.81%) faster than B; 15,360 of 16,384
  iterations (93.8%) used a speculative load. Recovery cost did not erase the
  gain.

### 7.3 Fan-out
- Checksum 246290604769280 (fields 147456 and 57344 checked separately); each
  of the three load PCs (`0x10154`, `0x10158`, `0x1015c`) commits 16,384 times.
- 15,615 producers each had two accepted consumers; 31,230 issued = 30,720
  committed + 510 squashed + 0 faulted; 255 redirects, each accounting for two
  consumers, both accepted before validation; incorrect 765 (255 × 3);
  predSquashed 767; mispredictSquashes 255; live and carry-in 0.
- Lookups 59,284 vs 58,519: 3 extra per redirect (phase had 5; fewer loop
  branches are in flight with the longer loop body). 105 checks pass.
- Performance: C 27,540 cycles (0.90%) faster than B; coverage 93.8%.

### 7.4 Fetch2 hook (real logs)
- Phase: 255 hooks = 255 redirects = `mispredictSquashes`; 0 parser problems,
  0 ambiguous, 0 missing history entries; each squash removed exactly 5
  entries, each followed by an update; the next entry always size 1 and 28
  sequence numbers younger. CL trace 161,937,593 bytes, 1,407,004 lines (much
  larger than the 27 MB without `Branch`).
- Fan-out: 255 hooks; exactly 3 removals each; next entry 38 younger; CL trace
  194,599,342 bytes, 1,683,594 lines.
- In both, the squash left the history empty (size 0) before `update()`, so
  **those workloads do not exercise `update()` retiring older uncommitted
  entries**; that rests on reading `BPredUnit::update()`.

Example (phase, first hook):
```
fetch2: LVP recovery redirect inst: 0/276.16824/67434/218096.218096 pc: 0x10158 (ld)
branchPred: [tid:0] [squash sn:218096] Removing history for [sn:218120] PC 0x10168
branchPred: [tid:0] [squash sn:218096] predHist.size(): 4
 ... three more removals, sizes 3, 2, 1 ...
branchPred: [tid:0] [squash sn:218096] predHist.size(): 0
branchPred: [tid:0] Committing branches until sn:218096]
```

### 7.5 Fault
- Checksum 2848; 704 committed loads, 352 at each of `0x10188`, `0x1018c`;
  traces A/B/C identical (38,791,461 bytes); 100 checks pass.
- Consumer addresses in A's trace: `0x20000018` read 256 times (32 × 8) and
  `0x11018` (node B) 96 times (32 × 3): every refetched consumer loaded node B.
- **32 faulting speculative consumers accepted, one per cycle**, all with
  source `0x20000000`. Each was accepted before its producer's validation, its
  producer was incorrect, and it was squashed after the redirect; none
  committed; `specLoadsFaulted` 0. Checker asserts at least one (it did not
  assume 32).
- Issued 156 = 124 committed + 32 squashed + 0 faulted; correct 256;
  incorrect 64 (32 producers plus 32 refetched consumers whose prediction `7`
  was wrong against B's `11`); mispredictSquashes 32.
- The 32 architectural faults seen were all the first store into the freshly
  mapped page (`pc 0x10260`), handled by demand paging; no speculative fault
  reached the architectural path.
- Hooks: 32; removals 3 (once), 5 (29 times), 7 (twice); every hook removed a
  predicted `ret`, so this is the only workload with RAS-restore lines (34);
  **one older entry always remained**, so `update()` did retire an older
  entry here (the opposite of phase and fan-out). CL trace 326,708,972 bytes.
- Performance: A = B = C = 9,242,313 cycles; **no speedup** (the store sweep's
  fixed memory time probably hides the few cycles saved; not investigated).

### 7.6 Chain
- Checksum 2848; 527,486 committed instructions, 1,056 loads, 352 at each of
  `0x10194`, `0x10198`, `0x1019c`; L2 always at `0x58008`; L3 `0x60018` ×256,
  `0x68018` ×96 (one 32768-byte stride apart). 106 checks pass.
- Accepted L2: 348; accepted L3: 35 (so the L3 window is real but rarely open);
  35 complete chains `accept(L2) < validate(L1) < accept(L3) < validate(L2)`;
  L3 never accepted before its L2's producer was validated (no nesting); all
  35 accepted L3 before their L2 validated.
- 27 incorrect L2 predictions used by L3 triggered redirects, all at L2 and
  none at L1; 27 speculative loads squashed; 356 committed; incorrect overall
  64 (37 wrong predictions never used by a dependent, so no recovery).
- **There were 32 pointer changes but only 27 recoveries**: recovery is not
  one per change, and no test claims it is.
- Fetch2 hook checks, lifecycle and carry-in checks pass. Speedup 9,422 cycles
  (0.10%).

### 7.7 Threshold sweep (704 lookups, 704 trainings at each threshold)

| | T1 | T2 | T3 |
|---|---|---|---|
| confident predictions | 574 | 446 | 320 |
| correct | 448 | 320 | 256 |
| incorrect | 126 | 126 | 64 |
| predSquashed | 0 | 0 | 0 |

All four runs exit 0, checksum 2848, identical committed traces (same md5),
identical non-LVP statistics, no speculative loads, redirects or carry-in.
`incorrect` is flat from T1 to T2 and drops at T3; only non-increase is
asserted, never strictness or totals.

### 7.8 PC window

| | FULL | NO_PRODUCER | NO_CONSUMER |
|---|---|---|---|
| lookups | 765 | 704 | 704 |
| trainings | 704 | 704 | 704 |
| confident | 381 | 320 | 320 |
| correct / incorrect | 256 / 64 | 256 / 64 | 256 / 64 |
| specLoadsIssued | 156 | 0 | 0 |
| committed / squashed / faulted | 124 / 32 / 0 | 0 | 0 |
| mispredictSquashes | 32 | 0 | 0 |

Window boundaries in `config.ini`: FULL `0x10188`–`0x10190`, NO_PRODUCER
`0x1018c`–`0x10190`, NO_CONSUMER `0x10188`–`0x1018c`. 79 checks pass. FULL's
381 confident = 320 validated + 61 squashed.

### 7.9 Invalid configurations
All nine exit 1 with exactly one matching fatal and nothing else (37 checks):

| Case | Fatal text |
|---|---|
| `spec_without_lvp` | `system.cpu.execute: lvpSpecAddr requires enableLVP` |
| `spec_without_early_issue` | `... lvpSpecAddr requires executeAllowEarlyMemoryIssue` |
| `window_without_spec` | `... lvpSpecPcBegin and lvpSpecPcEnd require lvpSpecAddr` |
| `window_begin_after_end` | `... lvpSpecPcBegin (0x10200) must be below lvpSpecPcEnd (0x10100)` |
| `window_begin_equals_end` | `... lvpSpecPcBegin (0x10100) must be below lvpSpecPcEnd (0x10100)` |
| `threshold_zero` / `_four` | `lvpConfThreshold must be in [1, 3] (got 0)` / `(got 4)` |
| `table_not_power_of_two` / `table_zero` | `lvpTableEntries must be a non-zero power of two (got 1000)` / `(got 0)` |

Speedups are small throughout (0.10% to 0.90%, none on fault): the benchmarks
overlap one short cached access with a long miss. The aim was a correct,
well-tested mechanism, not a headline number. They are single deterministic
data points, not general claims.

## 8. Errors, surprises and lessons

### 8.1 The first design measured nothing
Releasing the scoreboard when a load is predicted looked like value
prediction. The default memory FU has `extraAssumedLat = 2`, so a load's
destination is never marked "unpredictable" and dependents already reach their
FU about 3 cycles after the load issues. Example: `ld x1; add x2,x1,1` already
issues the `add` early; predicting `x1` changes nothing, and commit still
waits. **Lesson:** a prediction only matters if something *uses the value*.
That is why the project feeds the value into a dependent load's address.

### 8.2 `ActivityRecorder` assertion with idling enabled
The first hello smoke run aborted at tick 500: `activity.cc:121
deactivateStage: Assertion 'activityCount >= 0' failed`, in both LVP-off and
LVP-on runs. Setting `system.cpu[0].enableIdling=False` made both pass.
Whether it is a pre-existing gem5 issue was never proven (no unpatched binary
was run with idling); the evidence is that LVP-off and LVP-on runs were
identical with idling off. Every experiment, baseline included, uses
`enableIdling=False`; mixing would add a confounder. Idling-enabled behaviour
is untested.

### 8.3 `lvpSpecAddr=True` as default would break every run
The plan had `lvpSpecAddr` default `True` plus "requires `enableLVP`, else
`fatal`". Default `enableLVP=False` would then make every ordinary MinorCPU
run hit the fatal. Fixed: default `False`, and explicit `True` without
`enableLVP` or without `executeAllowEarlyMemoryIssue` is a fatal. The
harness sets `lvpSpecAddr` explicitly in B and C. The parameter's real name is
`executeAllowEarlyMemoryIssue` (C++ member `allowEarlyMemIssue`).

### 8.4 Wrong recovery guard
Planned `if (branch.isBubble())`. A correctly predicted branch committed
earlier in the same cycle is not a bubble, so the recovery would be skipped.
Fixed with `!branch.isStreamChange()`. Also my own wrong claim that loads are
marked "unpredictable" was corrected (true only for `extraAssumedLat == 0`).

### 8.5 A wrong accounting identity
`specLoadsIssued == committed + squashed` was proposed. It ignores faulted and
still-in-flight requests (nothing flushes the LSQ at exit). Corrected:
`issued + carryIn = committed + faulted + squashed + inFlight`. Likewise
`correct + incorrect == confidentPredictions` is invalid: a confident
prediction can be discarded or fault before validation. And `inLSQ` alone does
not prove acceptance (`commitInst` sets it again for a non-early load that
faulted before any push), so counting happens in `executeMemRefInst` after the
translation-fault suppression.

### 8.6 The LVP redirect bypasses branch-predictor cleanup
`Fetch2::updateBranchPrediction` returns early for any non-control
instruction, so a redirect raised by a load never calls `squash()`: younger
branches already predicted keep their history (and RAS entries), and are
predicted again after refetch, then trained with stale histories.
Performance-only, but it would distort any branch-predictor comparison. Fixed
by the `lvpRedirect` hook (section 2). Also: an adversarial claim "branch
predictor counters should match the baseline" was wrong for run C
(refetched branches are predicted again: +5 or +3 lookups per redirect), and
right only for B. `condIncorrect` is diagnostic, because a re-predicted
branch may see different table state.

### 8.7 `DrainCurrentInst` is reachable (my earlier claim was wrong)
I first said the drain state was unreachable because RISC-V has no micro-ops.
It is reachable (an outstanding load counts as an access in flight). It is
still safe: the stream number and map clear happen before the redirect is
replaced by a bubble, and the thread PC is architectural for resume. The Fetch2
hook does not run in that case, leaving orphaned predictor entries as any
drain does. Documented, not asserted; not tested.

### 8.8 Getting Stage B to run (five separate problems)
1. **Missing admission site (found in audit).** Under the default
   `RoundRobin` thread policy, `getCommittingThread()` does not call
   `commit()` while the head load is outstanding, so the early-issue block
   never runs and Stage B would never speculate. Fixed by sharing
   `lvpSpecAdmissible` in both places.
2. **ELF loader panic.** All four runs aborted at tick 0:
   `memory_image.hh:70 Segment outside the bounds of the image data`. A
   freestanding program has no initialised data, so the linker put `.bss` in a
   separate LOAD segment whose file offset (0x1000) is past the end of the
   2,408-byte file. gem5 builds a segment per header and checks
   `offset + size`. Fixed with `-Wl,-n` (one segment).
3. **`gp` was 0.** Next abort at tick 133000:
   `Page table fault ... 0xfffffffffffff818`, which is `0 - 2024`. Linker
   relaxation turned `node.v[3] = 7` into `sd a4,-2024(gp)`, and SE mode starts
   with `gp = 0`. My earlier statement that relaxation is harmless was wrong:
   it only removes whole instructions, but also introduces gp addressing.
   Fixed with `-Wl,--no-relax-gp`, and `check-asm` now rejects any `gp`/`x3`
   operand (it flagged the exact crashing instruction on the old binary).
4. **Empty traces.** Runs exited 0 but A/B/C `trace.txt` were 0 bytes:
   `ExeTracerRecord::traceInst` skips user-mode code unless `ExecUser` is on,
   and `ecall` is a fault whose record is dropped unless `ExecFaulting` is on
   (163,968 lines vs 163,971 `simInsts`). Flags are now
   `ExecEnable,ExecUser,ExecFaulting,ExecResult,ExecEffAddr,FmtTicksOff`.
5. **Checker gave vacuous PASSes.** On the aborted runs it compared two empty
   files as "identical" and then died with `KeyError: 'specLoadsIssued'`.
   Fixed with a preflight stage (section 6.9), later hardened after finding
   five more silent failure modes: `stats.txt` a directory, unreadable,
   binary garbage, a `nan`/`inf`/non-integral statistic, and `.out` a
   directory.

### 8.9 The runner ignored its own failures
`run_workload || status=1` runs inside a condition, which turns off `set -e`
for the whole function (a subshell does not turn it back on; verified with a
throwaway script). A failed build or assembly check would not stop gem5 from
running on a stale or broken binary. Fixed with one `run-one` process per
workload, deletion of old binary, disassembly, `.rc`, `.out` and result
directories first, argument validation (a bad workload name used to exit 0
silently), and window reset per workload. Tested with stand-in compilers and
gem5: 38 checks.

### 8.10 Build environment
First `scons` failed in configure: system SCons 4.11 on Python 3.14 mishandles
the arguments of `CheckLibWithHeader('z','zlib.h',...)` (links `-ll -li -lb
...`) and reported zlib missing although it is installed. `SConstruct` was
not modified. The existing venv `~/gem5-venv` (Python 3.12.14, SCons 4.8.1)
was used, with `PYTHON_CONFIG=/usr/bin/python3.12-config` set as an
**environment variable** (on the command line it is ignored, and the venv's own
`python3-config` has no `libpython3.12`).

### 8.11 Other command-line mistakes
- `-P system.cpu.enableLVP=True` fails (`'SimObjectVector' object has no
  attribute '_params'`); the CPU is a vector: `system.cpu[0].enableLVP=True`.
- `RiscvMinorCPU` refuses to run without `--caches`.
- A stray `rm -rf /dev/null` in one of my commands was a no-op (not root).
- An early `check-asm fanout` without `OUT=` wrote to the default
  `/tmp/lvp-stage-b/fanout`. No repository file was affected. Always pass
  `OUT=`.

### 8.12 Hook-validation checker weaknesses (fixed one at a time)
- A misleading message was reworded.
- Two hooks sharing one `History entry added` event made the per-hook check
  ambiguous and could let a later recovery mask an earlier one that did not
  leave the history empty. Now at most one hook per thread per entry, and a
  second hook before an entry fails as ambiguous.
- A malformed `History entry added` line was silently ignored, and a later
  valid entry could stand in for it. Now it is a parser problem and clears all
  waiting hooks.
- The `hook_checks` docstring first implied the next entry proves everything
  about `update()`. It was corrected to say it shows only the combined effect
  of both calls, and that the older-entry retirement is exercised only by the
  fault workload (section 7.5).

### 8.13 Chain: what the audit and run showed
- A local named `slots` shadowed the global `slots` (renamed `cslots`).
- The assembly checker's unpack assumed one consumer per group; a guard was
  added.
- Whether L3 would ever be accepted was unknown (a tight timing window). It was:
  35 times, out of 348 L2 acceptances.
- "One recovery per pointer change" was wrong: 27 recoveries for 32 changes.
  The check is `0 < redirects <= incorrect`.

### 8.14 Invalid-config parser assumed the wrong output format
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
which I had flagged as unverified, rendered correctly.)

### 8.15 Pre-commit hooks and formatting
- A false alarm: `%d` with `uint64_t` and `%#x` with `RegVal` look wrong, but
  gem5's `cprintf` is type-driven, so they are correct.
- The Stage B commit was rejected by the `black` hook, which reformatted one
  line of `BaseMinorCPU.py` (a three-line parameter joined to one 79-column
  line). I had run black only on the checker. Fixed by restaging that file;
  the commit then passed (black, style, commit message, Gerrit). Hooks were
  never bypassed.
- `black` reformatted `check_lvp_stageb.py` before the Stage B commit; its AST
  was verified identical and the checker's output on real results was
  byte-identical before and after.

### 8.16 Habits that prevented worse errors
- **Synthetic tests for every checker**: build fake results and inject one
  fault at a time (missing run, wrong exit code, wrong boundary, swapped
  message...). Each must fail with a specific message, and the good case must
  pass. Synthetic harnesses had their own bugs; those were fixed in the
  harness, never by relaxing the checker.
- Every new checker was re-run on the saved real results of the earlier
  workloads to prove no regression.
- Preprocessed `ldld.c` output compared byte for byte with HEAD for every
  earlier workload after each change (and the stable binary compared with the
  Stage B binary).
- Independent evidence preferred over self-reported logs: the committed trace
  (effective addresses and results), the checksum and the statistics are
  independent of the LVP log lines; the event-order checks use the log.
- Before each commit: `git diff --check`, `black --check` on the Python,
  `bash -n` on the shell, a read of every removed line, explicit file
  allowlists (never `git add .` or `-A`).

## 9. Running the tests

gem5 must already be built (`build/RISCV/gem5.opt`). `OUT=<dir>` chooses where
results go (default `/tmp/lvp-stage-b`).

    tests/lvp/run_lvp_stageb.sh check-asm [WORKLOAD]   # build and check the disassembly
    tests/lvp/run_lvp_stageb.sh run [WORKLOAD]         # check-asm, then A, B, C, CL, then the checker
    tests/lvp/run_lvp_stageb.sh threshold              # confidence-threshold test
    tests/lvp/run_lvp_stageb.sh pcwindow               # PC-window test
    tests/lvp/run_lvp_stageb.sh invalid-config         # invalid-configuration test

`WORKLOAD` is `stable`, `phase`, `fanout`, `fault`, `chain`, or `all` (the
default). The three configuration tests are not part of `all`. Each workload
writes under `$OUT/<name>`, and `run` starts a separate process per workload so
a failure stops that workload before any simulation but all are attempted.
The checker alone:

    python3 -I tests/lvp/check_lvp_stageb.py OUT/<name> <mode>

`<mode>` is `stable|phase|fanout|fault|chain|threshold|pcwindow|invalid-config`.
Expect the `Branch` logs of phase, fanout, fault and chain to be large
(160 to 330 MB each).

## 10. Environment notes

    export PATH=/home/ayushgangal/gem5-venv/bin:$PATH
    export PYTHON_CONFIG=/usr/bin/python3.12-config
    scons build/RISCV/gem5.opt -j$(nproc)

Builds: infrastructure 1,186 compilations (new Python params regenerate many
headers); Stage A 21 compilations, 2m54s; Stage B 20, 4m01s; Stage C 3m08s.
Only the pre-existing `units.hh` template warnings appear, plus the usual
missing libpng and HDF5 notices. The repo's pre-commit black is the cached
one (line length 79).

## 11. Not supported and not tested

Not supported by design: speculation into ALU operations, branches, stores,
atomics, LR/SC, micro-ops or FP/vector loads; selective replay (recovery
refetches everything after the producer); nested speculation (L3 never uses
L1's prediction).

Not tested (known gaps, not known defects):
- Drain (`DrainCurrentInst`/`DrainAllInsts`) during recovery, interrupts
  (none in RISC-V SE mode), SMT or multiple threads, idling enabled.
- A statistics reset while work is live (the carry-in design is implemented,
  but only the startup reset occurs in these runs, so carry-ins are only ever
  asserted to be 0; a scheduled-reset wrapper was planned but not written).
- An erroring memory response (not reachable in SE mode), and `update()`
  retiring more than one older history entry.
- Other block sizes, random transition patterns, table aliasing, and other
  table sizes.
- The safety stops (consumer source check, `panic_if`s) have never fired in a
  real run; they are source-verified only. The checker was exercised with
  synthetic logs, not by mutating the C++.
- Branch-predictor integrity beyond the `Branch` log and statistics.
- Results come from one in-order core and a handful of microbenchmarks; no
  claim is made about real workloads.
