# v8 filter replication — calibration results

Executed the plan in `FILTER_V8_REPLICATION.md`: replicate linrock's
`csv_filter_v8.py` using only the Monty visit distribution, calibrating the one
proxy rule against ground-truth Stockfish **once**, then filtering with **zero
Stockfish**. Done 2026-06-27.

All code lives in the standalone crate `~/sf_experiments/v8filter/` (paths below
are relative to it).

## Deliverable (the filter)

`src/lib.rs`:
- `visit_stats(dist)` — top-2 closeness `r2 = v2/v1`, plus `p1`, `gini`, `ens`.
- `passes_exact_gates(pos, best_move, st, ply)` — v8 rules 1,2,3,6,8,9,10,11,
  ported exactly (ply>20 = 16-ply book + 20; best move & visit-top-2 not
  capture/promo; not in check; ≥2 legal moves).
- **`keep_v8(pos, best_move, st, ply)`** — the production filter, gates + the
  calibrated `r2 >= CALIBRATED_TAU` (= **0.15**). Visits only, no Stockfish.

Validated end-to-end (`src/bin/filter_stats.rs`) on 2.75M fresh positions:
41.2% dropped by gates, 2.0% by the r2 cut, **56.7% kept**; the r2 cut removes
3.43% of gate-passed positions — matching the 3.51% measured at calibration.

## Method

1. **Ground-truth Stockfish** — linrock `Stockfish@nnue-data-v7-3072`,
   `make profile-build ARCH=x86-64-bmi2` (Appendix A of the plan). The `100/150`
   cp thresholds in v8 are tuned to this binary's eval scale.
2. **Sample** (`src/bin/sample.rs`) — streamed the Monty source
   (`interleaved-new-policy.binpack`, raw montyformat, no magic), applied the
   exact gates, took **1,000,000** survivors into 60 shards. Each emits, in
   lockstep, a Stockfish `.binpack` entry (via `sfbinpack`) and a stats row
   (`r2,p1,gini,ens,num_moves,score`). **Castling rights are ignored**: every FEN
   uses castling `-`, so the position stays legal and 960 rook-file packing is
   sidestepped (only 2.6% of samples had any rights).
3. **Rescore** (`rescore_all.sh`) — 60 single-threaded SF processes (order-
   preserving), `transform rescore filter_depth 6 filter_multipv 2` → 10-column
   v8 CSV with `s1,s2`. Row i of `shardK.stats.csv` joins to line i of
   `shardK.csv`. (A session teardown truncated the tail; **936,817** positions
   rescored cleanly — the prefixes still align, statistically ample.)
4. **Fit** (`fit.py`) — reconstructed the literal v8 rules 4-5 decision from
   `(s1,s2)` and swept `r2 >= tau`.

## Results (936,817 positions)

After the exact gates, **v8 rules 4-5 remove only 3.51%** of positions — the
gates do the heavy lifting; the "one good move" eval-gap test is a small
refinement. (Our quiet, many-legal-move Monty positions rarely have a large
top-2 eval gap.)

Best single visit feature for the rule-4-5 decision (AUC):

| feature | AUC | | feature | AUC |
|---|---|---|---|---|
| **r2 (top-2 closeness)** | **0.738** | | 1 − p1 | 0.712 |
| gini | 0.652 | | ens | 0.652 |
| | | | num_moves | 0.371 |

**r2 is the best single feature** — top-2 closeness beats global gini, exactly as
the plan predicted (gini conflates "one good move + junk tail" with "two co-best
moves"; r2 does not).

Operating points on r2 vs the rule-4-5 decision:

| choice | tau | keep% | acc | precision | recall |
|---|---|---|---|---|---|
| **rate-match (chosen)** | **0.153 → 0.15** | 96.6 | 94.7% | 0.972 | 0.973 |
| Youden-J | 0.500 | 80.8 | 81.2% | 0.981 | 0.821 |
| max-accuracy | 0.05 | 99.3 | 96.1% | 0.966 | 0.995 |

"Max-accuracy" is degenerate (96.5% base rate → keep-almost-everything wins on
accuracy). **Rate-matching** reproduces both v8's volume and composition with
balanced ~0.97 precision/recall, so `tau = 0.15`. Youden over-filters chasing a
3.5% class at the cost of ~18% of good positions.

Against the **full** v8 decision (rules 4-5 **and** SF top-2 not a capture/promo),
keep rate is 92.2% and r2≥0.15 agrees 90.6%. The residual ~9% is dominated by the
4.5% where SF's actual top-2 includes a capture our **visit** top-2 did not — an
inherent visits≠search gap, not fixable by tuning r2.

## Caveats / honest limits

- **Castling ignored** in the SF eval (FEN `-`); affects only the rare
  castling-relevant position. Visit stats are from the true Monty search.
- **~6.3% of the 1M rescore tail** was lost to a session teardown; used the
  936,817 clean prefix rows (representative; shards are interleave-shuffled).
- The eval-gap proxy is a **minor** filter post-gating; if Monty datagen used few
  nodes/move the visit top-2 is noisier, but the validation (3.43% vs 3.51% on
  independent data) shows it is stable here.
- `sklearn` absent, so the 2-D `(r2,gini)` logistic was skipped; given r2's AUC
  and the 3.5% event size it would add little. r2-only is the recommendation.

## Reproduce

```
cd ~/sf_experiments/v8filter
cargo build --release
./target/release/sample <source.binpack> calib 1000000 60      # sample
bash rescore_all.sh calib /tmp/stockfish/src/stockfish 60      # SF labels
python3 fit.py calib                                           # fit tau
./target/release/filter_stats <source.binpack> 20000          # validate
```
