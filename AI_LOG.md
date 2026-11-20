# AI-use log: continuous theta in `ch7/hip`

This log records how an AI coding assistant, Claude Code (Opus 5.5), helped with the project between
2026-11-19 and 2026-11-20. The design decisions were made by the student. The AI fixed the code, ran the
experiments, and took a few implementation decisions, which are flagged below.
The project overview is in [README.md](README.md).

| If you want… | Go to |
|---|---|
| What was broken and why | [1. The starting code](#1-the-starting-code) |
| What changed in `hip.go` | [2. Changes](#2-changes-to-hipgo) |
| Who decided what | [3. Design decisions](#3-design-decisions) |
| What was run and what came out | [4. Experiments](#4-experiments) |
| What comes next | [5. Next: XCal](#5-next-xcal) |

---

## 1. The starting code

The original `ConfigLoopsHipContinuous` was meant to replace leabra's discrete theta switches with
sinusoids. It ran, but had no theta at all:

| Problem | Effect |
|---|---|
| It was registered as a looper **event at counter 1**, which fires once per trial, not every cycle. | The scales froze at cycle 1 for the whole trial (ECin→CA1 0.53, CA3→CA1 1.0, DG→CA3 4.0). |
| EC used `0.5(1+sin)` and CA3 used `0.5(1+cos)`. | They were a quarter-cycle off the original schedule and not complementary, so total input to CA1 also oscillated. |
| DG was driven by `cos`. | DG was at full strength in Q1, when it should be off. |
| The plus-phase copy (ECout ← ECin) was inside the callback. | It never ran (because of the bug above). If it had, it would have been re-applied every cycle. |

With the scales frozen, performance collapsed (5 runs: AB retained 0.30 vs 0.62 for the original; runs
hit the epoch limit). That reflected the bug, not continuous theta.

## 2. Changes to `hip.go`

All changes are in `ch7/hip/hip.go`.

1. **Two `Config` fields**, `ThetaKCA1` and `ThetaKDG`, both defaulting to `-1`. A value below 0 gives
   the original discrete switches, 0 gives a pure cosine, and a value above 0 sharpens the cosine.
2. **Gate functions** with `c` = cycle (0–99):
   - `thetaEC(c, k) = ½(1 + tanh(k·cos(2πc/100)) / tanh(k))`. This is 1 at cycles 0 and 100, 0 at
     cycle 50, and 0.5 at cycles 25 and 75, which are the original switch points.
   - `thetaDG(c, k) = 1 − thetaEC(c, k)` for c < 50, then 1.
3. **`ConfigLoopsHipContinuous`** runs at the start of every cycle, before `net.Cycle`. It sets:
   - ECin→CA1 `Abs` to `thetaEC`, and CA3→CA1 `Abs` to `1 − thetaEC`.
   - DG→CA3 `Rel` to `thetaDG` × 4 in Train, or × 1 in Test.

   It then rescales conductances, as the original does. Because the DG `Rel` value is normalized
   against the other CA3 inputs, the ECin and CA3-recurrent inputs rescale automatically, and their
   10:1 ratio is unchanged.
4. **The ECout ← ECin copy** happens once, at cycle 75, in Train only, as in the original.

With both k values at `-1`, the scales match leabra's `ConfigLoopsHip` exactly on every cycle, in both
Train and Test. With k = 1000 they differ only at cycles 25 and 75, where the cosine is exactly 0.

**Running it:** the repo has no `go.mod`. Copy `hip.go` into a clone of `CompCogNeuro/sims`
(`ch7/hip/`) and run `go run .`. Set the two k values in the GUI's Config panel.

## 3. Design decisions

**Student:**
- A cosine gate with a sharpness parameter k.
- DG continuity through its `Rel` value, with amplitudes and phases matching the discrete version,
  including the Train/Test difference (4 vs 1).
- Discrete as the default for whichever gate is not being varied.
- The "halfway to discrete" k defined by L2 distance from the cosine to sign(x).
- All sweep designs (see §4).

**AI judgement calls, reviewed and kept:**
- **DG uses only the rising edge, then holds at 1** through the plus phase, as in the original. A full
  DG cosine would fade the DG-driven teacher signal by cycle 99, which would remove the error signal for
  ECin→CA3.
- DG's rise is phase-locked to the CA1 gate (midpoint at cycle 25).
- The ECout copy stays a discrete event at cycle 75.
- The halfway k values are computed over the 100 discrete cycles rather than as a continuous integral.
  The difference is negligible.
- Conductances are rescaled only when the scales change. This affects speed only.

**Open confound (not changed):** recall is read out from `ActM` at cycle 75. With smooth gates, that is
the EC/CA3 crossover point, so some differences in memory scores could come from when recall is read out
rather than from what was learned.

## 4. Experiments

**Setup:**
- **Sharpness levels:** k ∈ {0, 1.69, 3.52, 8.38, 16.44, discrete}. The four nonzero values are 25%,
  50%, 75% and 90% of the L2 distance from the cosine to discrete.
- **Seeds:** seed = run index + 1, so runs are paired by seed across conditions.
- **Measures:** AB retained after AC learning (`TstABMem`), epochs to learn AB (`firstPerfect`), run
  length (`lastEpoch`), AC memory, and lure false recall.
- **Harness:** headless `go test` (`harness_test.go`, `plot_test.go`), run as parallel processes of 10–50
  runs each. These files are kept outside the repo.

**Runs:**

| Run | Design | n per condition | Output |
|---|---|---|---|
| Smoke test | Original vs new code (discrete and k = 1000) | 2 | — |
| 1-D sweep | Vary one gate while the other stays discrete | 10 | `results/sweep_*.csv` |
| 1-D sweep | Same, plus k = 0 | 100 | `results/n100/sweep_*`, `*.png` |
| Grid | All 5 × 5 k combinations; the discrete row and column come from the 1-D sweep, giving 6 × 6 | 100 | `results/n100/grid_*`, `grid_heatmaps.svg` |

At n = 10 the standard deviations were large: AB retained ≈ 0.19, epochs to learn AB ≈ 1.75. Detecting a
0.1 difference in AB retention needs about 41 paired runs, which is why n was raised to 100.

**Results (n = 100):**

| Condition | AB retained | Epochs to learn AB | Run length |
|---|---|---|---|
| Discrete (both gates) | 0.63 | 2.8 | 6.6 |
| DG cosine (k = 0) | 0.57 | 4.7 | 10.4 |
| CA1 cosine (k = 0) | 0.55 | 3.1 | 7.5 |
| Both cosine | 0.50 | 5.3 | 12.3 |

- Smoothing the DG gate slows AB learning, and the effect is monotonic in k. The CA1 gate barely
  affects learning speed.
- Smoothing either gate lowers AB retention.
- AC is learned in every condition (about 1.0), and lure recall stays at or below 1.2%.

The 95% CIs in `grid_summary.csv` are paired against discrete and are not corrected for multiple
comparisons.

## 5. Next: XCal

**Plan.** Four conditions, crossing theta with the learning rule:

|  | Snapshot rule (current) | Standard XCal |
|---|---|---|
| **Discrete theta** | ✓ | ✓ |
| **Continuous theta** | ✓ | ✓ |

- 100 runs per condition, 20 processes, default learning rates.
- Make the minimal switch: change the hippocampal path types from `EcCa1Path`/`CHLPath` to standard
  leabra paths. The rule should be selectable through a `Config` flag.
- Then sweep the XCal time constants.

**Why the model uses snapshots.** The theta phases are the error signal in the model this sim is based
on (Ketz, Morkonda & O'Reilly, 2013). The minus state is the end of Q1, where CA1 and CA3 reflect only
EC input, with DG off. The plus state is the end of the trial. Standard XCal compares `AvgS` (τ ≈ 2
cycles) with `AvgM` (τ = 10, which covers roughly cycles 70–99). That puts the minus side in late Q3
instead of Q1, so the effect differs by pathway:

| Pathway | Under standard XCal |
|---|---|
| ECin→CA3, CA3→CA3 | The DG-off vs DG-on contrast is lost, leaving mostly BCM learning. Expect a large drop even with discrete theta. |
| EC↔CA1 | The contrast becomes CA3-driven recall vs the EC target, and it spans the cycle-75 crossover where continuous theta mixes the two. |
| CA3→CA1 | Almost unchanged; it already compares cycle 75 with cycle 99. |
| ECin→DG | Small change; the rule is mostly Hebbian. |

**Implementation notes:**
- Changing the path type also stops the class-based parameter blocks (`.EcCa1Path`, `.HippoCHL`,
  `.PPath`) from applying: their learning rates, WtBal, Hebb and `SetLLrn` settings.
- The running averages are **not** reset at trial start. `DecayState` leaves `AvgSS`, `AvgS` and `AvgM`
  alone; they are initialized only in `InitWeights`. At learning time the previous trial contributes
  almost nothing (about 0.9¹⁰⁰ ≈ 0), but early-cycle averages still contain the previous trial's state.
  `AvgL` persists across trials by design.

**Earlier suggestion, not pursued yet:** stats aimed at H1 and H2, namely intrusions of the competing
associate and CA3 pattern similarity for same-cue AB/AC pairs.
