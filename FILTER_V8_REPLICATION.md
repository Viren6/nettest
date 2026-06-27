# Replicating `csv_filter_v8` with the Monty visit distribution

**Status:** design note, not committed. Scratch for the `fine_tune_relabel_2` work.

Goal: reproduce linrock's [`csv_filter_v8.py`](https://github.com/linrock/nnue-data/blob/master/csv_filter_v8.py)
position filter as faithfully as possible, but driven by what *our* source
binpack actually carries — the per-move **visit distribution** from Monty's
MCTS — instead of the two separate Stockfish move-searches that v8 relies on.

---

## 1. What `csv_filter_v8` actually does

It reads CSV rows with these 10 fields:

```
ply, fen, bestmove_uci, bestmove_score, game_result,
sf_search_method, sf_bestmove1_uci, sf_bestmove1_score,
sf_bestmove2_uci, sf_bestmove2_score
```

`sf_bestmove1_score` / `sf_bestmove2_score` are the evals of the **best** and
**second-best** move, from independent Stockfish searches (centipawns, STM POV).
A position is **kept** only if *none* of the following skip rules fire (in order):

| # | Skip rule | Needs |
|---|-----------|-------|
| 1 | only one legal move (row has 8 fields) | legal-move count |
| 2 | start-of-game position | game boundary |
| 3 | `ply <= 36` (`EARLY_PLY_SKIP`) | ply from game start |
| 4 | **one good move (mag):** `\|s1\|<100 & \|s2\|>150`  *or*  `\|s1\|>150 & \|s2\|<100` | per-move evals s1,s2 |
| 5 | **one good move (sign-flip):** when `sign(s1)!=sign(s2)`, skip if `\|s1\|>100 & \|s2\|>100` *or* `\|s1−s2\|>150` (the v8 thresholds; they subsume the older v6 `>150/>150` and `>200`) | per-move evals s1,s2 |
| 6 | best move is a promotion | best move |
| 7 | duplicate position (piece-placement seen before) | global seen-set |
| 8 | side to move in check | board |
| 9 | best move is a capture | board+move |
| 10 | `sf_bestmove1` is a capture or promo | top-1 search move |
| 11 | `sf_bestmove2` is a capture or promo | top-2 search move |
| → | otherwise **keep**, emit `(fen, bestmove_uci, bestmove_score, ply, result)` | |

**Intent in one sentence:** keep *quiet* (no captures/promos/checks), *non-opening*,
*non-duplicate* positions where the eval is **robust to which of the top moves is
played** — i.e. there is **no single standout move** and no large eval swing
between the best two moves. Rules 4–5 are the heart of it: they remove
tactical / forced / "only-one-move-holds-it" positions. Everything else is
hygiene (openings, captures, dupes).

Note the asymmetry buried in rules 4–5: same-sign top-2 moves are kept *regardless
of absolute magnitude* as long as neither standout pattern fires (e.g. s1=300,
s2=280 is kept — both win, choice doesn't matter). Opposite-sign top-2 are kept
only when `|s1|+|s2| ≤ 150`, i.e. basically equal noise straddling zero. So the
filter is fundamentally a **top-2 closeness** test, hand-shaped with magnitude
thresholds.

---

## 2. What our source binpack carries (Monty format)

Per game: start position + castling + result. Per move (`SearchData`,
`montyformat-0.9.2/src/format.rs`):

- `best_move: Move` (u16) — the move Monty played (≈ argmax visits).
- `score: f32` in **[0,1]** — root value = **win probability for the side to move**
  (stored as `u16 = score*65535`). This is the *position* eval, **one number**.
- `visit_distribution: Option<Vec<(Move, u32)>>` — **every legal move** with its
  visit count, **quantized to u8** scaled so the most-visited move = 255
  (`scaled = round(visits * 255 / max_visits)`). Sorted by move id.
- `num_moves: u8` — number of legal moves (= length of the distribution).

`Move` exposes `is_capture()`, `is_promo()`, `is_en_passant()`, `flag()`.
`Position` exposes `in_check()`, `bbs()`, `map_legal_moves()`, `stm()`, `halfm()`,
`fullm()`. (Same API the existing `gini_threshold.rs` / `piece_count_dist.rs`
helper bins already use.)

### The one fundamental mismatch

v8's rules 4–5 compare **per-move evals** `s1` vs `s2`. **We do not have per-move
evals.** We have one root `score` and the **visit distribution**. So rules 4–5
cannot be ported literally — they must be replaced by a visit-based proxy.

Everything else maps cleanly (and rules 10–11, "top-2 search moves not
captures/promos", map *directly* onto the two most-visited moves).

---

## 3. Mapping each rule to our data

| v8 rule | Our replication | Fidelity |
|---|---|---|
| 1 only-one-move | `num_moves < 2` → skip | **exact** |
| 2 start-of-game | skip the first position of each game record | **exact** |
| 3 `ply <= 36` | skip while move-list index `<= 20` (= v8's 36 minus our 16-ply book) | **exact*** |
| 4–5 one good move | **visit-concentration proxy on the top-2 moves** (§4) | **proxy** |
| 6 best move promo | `best_move.is_promo()` | **exact** |
| 7 dedup | Bloom filter on piece-placement, or drop (§6) | approx / optional |
| 8 in check | `pos.in_check()` | **exact** |
| 9 best move capture | `best_move.is_capture()` | **exact** |
| 10 sf move1 cap/promo | top-1 visited move `.is_capture()||.is_promo()` | **exact-equivalent** |
| 11 sf move2 cap/promo | top-2 visited move `.is_capture()||.is_promo()` | **exact-equivalent** |

\* *ply caveat:* v8 counts half-moves from the true game start; our games begin
after a **16-ply book**, so our move-list index 0 is already at true-ply 16. To
skip the same opening phase as v8 (true-ply ≤ 36), skip move-list index **≤ 20**
(16 + 20 = 36). If the book depth changes, adjust the constant so
`book_ply + threshold = 36`.

---

## 4. The core: replacing rules 4–5 with the visit distribution

### Why visits are a *good* proxy (arguably better than v8's 2-move search)

In MCTS/PUCT, visit count is a monotone function of a child's value relative to
its siblings: the search pours visits into moves it believes are good. So:

- **one standout move** (large eval gap → v8 skips) ⟺ **peaked** visit
  distribution (one move ≈ all visits, second move ≈ none).
- **several comparable moves** (v8 keeps) ⟺ **split** top of the distribution.

This is exactly the signal rules 4–5 try to recover with two Stockfish searches —
and our distribution integrates Monty's *entire* search, not just a 2-move probe.

### Which statistic — top-2, **not** global gini

The prior experiment filtered on **gini impurity** `G = 1 − Σ pᵢ²`
(`pᵢ = visitsᵢ / Σ visits`). That is a *global* spread measure and is a **cruder**
match to v8, which only ever looks at the **top two** moves. Gini conflates:

- "one good move + a long tail of junk" → `p1=0.5`, ten moves at `0.05`: gini
  looks flat-ish, but there **is** a standout move (v8 would *skip*); and
- "two genuinely co-best moves" → `0.45/0.45`: also flat-ish, no standout
  (v8 would *keep*).

A **top-2** statistic separates these. Recommended primary statistic, computed
from the stored u8 visits (max move = 255):

```
v1 = highest visit byte (= 255 by construction)
v2 = second-highest visit byte
r2 = v2 / v1            # second-best move's share of the best move's visits, in [0,1]
```

- `r2 → 1`  : two (or more) comparable moves → **keep** (eval robust to choice).
- `r2 → 0`  : one move dominates → "one good move" → **skip**.

**Filter:** keep iff `r2 >= τ`. This single, well-calibrated threshold reproduces
the *aggregate* effect of v8 rules 4–5 (remove standout-move positions) without
needing per-move evals.

Useful auxiliaries (compute alongside for calibration / optional refinement):

- `p1 = v1 / Σ v`  — top move's share of *all* visits (peakedness).
- `gini = 1 − Σ pᵢ²`, `ENS = 1/Σ pᵢ² = 1/(1−gini)` — effective number of moves.
- keep top-2 *and* gini if you want a 2-gate filter (`r2 ≥ τ` **and** `gini ≥ g`)
  to also drop "standout + junk tail" cases that a pure ratio could let through
  when v2 is itself noise.

### Magnitude / sign subtleties (rules 4–5 fine print): mostly unreproducible

v8's `100/150` cp thresholds and the sign-flip branch need `s2`, which we lack.
We only know `|s1|` (≈ root `score`, since `best_move` ≈ root). Options:

- **Recommended:** ignore them. The visit-gap test already removes the
  positions these were designed to catch; the magnitude shaping was v8 hand-tuning
  around a *2-move* signal we've replaced with a *full-search* signal.
- *Optional refinement* using root `score` (win-prob `p`, STM POV): convert to a
  pawn-ish scale with the logit `cp ∝ ln(p/(1−p))` and, if you want to mimic
  "keep near-equal sign-flip noise," only apply a *looser* `r2` gate when
  `|p − 0.5|` is small. Low value; adds knobs. Skip unless calibration demands it.

### Caveat — visit fidelity depends on nodes/move at datagen

If Monty datagen used few nodes/move, the visit distribution is dominated by the
**policy prior** and is noisy at the top-2, weakening the proxy. Check the datagen
node count; the lower it is, the more you should (a) lean on a 2-gate
`r2 + gini` filter and (b) calibrate against ground truth (§5) rather than trust
a hand-picked `τ`.

---

## 5. Calibration: one-time Stockfish pass on ~1M positions, then visits-only

There is no analytic visit-gap ↔ centipawn-gap conversion (depends on cpuct,
node count, prior). So we **fit** the visit-only proxy to ground truth **once**,
on a small sample, and then run production filtering with **zero Stockfish** —
visits only. Stockfish is a calibration instrument here, not part of the pipeline.

### 5.1 The plan

1. **Build the ground-truth Stockfish** = the exact binary linrock used to make
   the v8 CSVs (Appendix A). Its eval scale is what the `100/150` thresholds were
   tuned against, so we must use *this* binary, not master SF.
2. **Pick the calibration sample.** Stream the source binpack, apply the **cheap
   exact gates** (rules 1–3, 6, 8–11 from §3) and uniformly sample **~1M
   survivors**. Sampling *after* the gates concentrates the SF budget on exactly
   the positions where rules 4–5 (the only ported-as-proxy rules) actually decide
   keep vs skip. For each sample row record: `fen`, the full visit distribution
   stats we'll fit on (`r2`, `p1`, `gini`, `ENS`), `num_moves`, root `score`.
3. **Get `s1,s2` from Stockfish** on each sampled FEN with the v8 settings —
   **depth 6, MultiPV 2**, `PruneAtShallowDepth=false`, `Use NNUE=true`
   (Appendix A). `s1 = multipv-1 score`, `s2 = multipv-2 score`, in cp, STM POV.
   At depth 6 this is sub-few-ms/pos; 1M single-thread ≈ tens of minutes, and it
   shards trivially across cores (run N SF procs, `Threads 1` each, as the
   reference script does). This is the whole SF cost — once.
4. **Label each sample by the *literal* v8 rules 4–5** using `s1,s2` (the magnitude
   and sign-flip logic from §1). Since rules 1–3, 6, 8–11 are ported exactly and
   already applied in step 2, rules 4–5 are the *only* source of disagreement — so
   this label is precisely what we need to fit.
5. **Fit the visit-only decision to that label.** Start with the 1-D threshold
   `r2 ≥ τ`; sweep τ and read off the ROC / agreement / confusion matrix. If 1-D
   leaves too much error, fit a tiny 2-D boundary over `(r2, gini)` (a decision
   stump pair, or a 2-feature logistic regression) — still cheap to evaluate per
   position in production. Optionally add `score` (root eval) as a 3rd feature to
   recover a little of the magnitude nuance, but only if it measurably helps.
6. **Choose the operating point and freeze it.** Pick τ (or the boundary) at the
   precision/recall trade-off you want — note the choice also fixes the keep rate,
   which should be sanity-checked against §5.2. Write the frozen constant(s) into
   the production filter; SF is never run again.

### 5.2 Cheap cross-check (no SF)

Independently, histogram `r2` (and `gini`) over a large gate-passed sample using a
`gini_threshold.rs`-style streaming bin (swap its scalar for `r2`, add the
move-type/ply gates). This tells you the keep fraction each τ implies — use it to
sanity-check that the fitted operating point from §5.1 lands at a sane volume.

### 5.3 Notes on getting our positions into Stockfish

The reference `binpack_to_csv.sh` rescores a *Stockfish* binpack; our source is a
*Monty* binpack, so don't reuse that wrapper directly. Two routes:

- **Recommended — drive UCI over FENs.** For each sampled `pos.as_fen()`:
  `setoption name MultiPV value 2`, `setoption name PruneAtShallowDepth value false`,
  `setoption name Use NNUE value true`, `position fen <fen>`, `go depth 6`, then
  parse the final-depth `info ... multipv 1 ... score cp <s1>` and
  `multipv 2 ... score cp <s2>`. No format conversion; batch many FENs per process.
  `filter_depth 6 filter_multipv 2` is exactly a depth-6 MultiPV-2 search, so this
  reproduces the same `s1,s2`.
- **Faithful-to-the-letter — emit a Stockfish binpack** of the 1M sample (e.g. via
  a `.plain` → `transform` conversion) and run the literal `transform rescore`
  command from `binpack_to_csv.sh`. More plumbing; only worth it if you want the
  CSV byte-identical to the v8 pipeline.

Handle mate scores (`score mate N`) by mapping to a large cp sentinel before
applying the `100/150` thresholds, matching how the CSV stores them.

---

## 6. Dedup at our scale (rule 7)

v8 keeps a global `set()` of piece-placement strings (FEN field 0 only — ignores
STM/castling/EP) and drops repeats. Over a 1.6 TB source pack that's billions of
positions — an exact in-memory set is infeasible. Options:

- **Bloom filter** sized for the expected kept count at a chosen false-positive
  rate (FP just drops a few extra positions — harmless). Bounded memory, one pass.
- **Drop dedup entirely.** The interleave already shuffles; exact-layout repeats
  are far less harmful for value training than for the small SF datasets v8
  targeted. Simplest; recommended unless duplication is shown to be high.
- Window/shard-local dedup if a middle ground is wanted.

Match v8's key exactly only if you keep dedup: piece-placement **only**
(`fen.split(' ')[0]`), which also merges same-layout/different-STM — aggressive,
probably not what we want; prefer the full board key (placement+STM+castling+EP)
if deduping.

---

## 7. Reference pseudocode (operates on `MontyFormat` records)

```rust
// per game record
let mut pos = game.startpos;
let castling = game.castling;
for (ply, sd) in game.moves.iter().enumerate() {
    // --- exact gates ---
    let keep = (|| {
        if sd.visit_distribution.as_ref().map_or(0, |d| d.len()) < 2 { return false; } // rule 1
        if ply <= 20 { return false; }            // rules 2,3: 16-ply book + 20 == v8's true-ply 36
        if sd.best_move.is_promo() { return false; }                                    // rule 6
        if pos.in_check() { return false; }                                             // rule 8
        if sd.best_move.is_capture() { return false; }                                  // rule 9

        let dist = sd.visit_distribution.as_ref().unwrap();
        // top-2 most-visited moves (visits are u8, max = 255)
        let (m1, v1, m2, v2) = top2_by_visits(dist);
        if m1.is_capture() || m1.is_promo() { return false; }                           // rule 10
        if m2.is_capture() || m2.is_promo() { return false; }                           // rule 11

        // --- proxy for rules 4–5: top-2 visit closeness ---
        let r2 = v2 as f32 / v1 as f32;
        if r2 < TAU { return false; }                 // one standout move -> skip
        // optional 2nd gate: if gini(dist) < G { return false; }

        true
    })();

    if keep {
        // emit (fen=pos.as_fen(), score=sd.score, best_move=sd.best_move, ply, result)
        // (optionally Bloom-dedup on the board key first)
    }
    pos.make(sd.best_move, &castling);
}
```

`top2_by_visits` = single pass over `dist` tracking the two highest `(move, visits)`.
Note `score` here is win-prob in [0,1]; whatever relabeling target the new run uses
plugs in at emit time — the *filtering* is independent of the label.

---

## 8. Summary / recommendation

- **Port literally:** rules 1–3, 6, 8–11. Rules 10–11 land *exactly* on the two
  most-visited moves — a clean, free use of the visit distribution.
- **Replace rules 4–5** (the "one good move" eval-gap test) with a **top-2 visit
  closeness** gate `r2 = v2/v1 ≥ τ`, not a global gini threshold — top-2 is the
  faithful analog of what v8 measures. Optionally add a `gini ≥ g` second gate.
- **Calibrate once, then visits-only.** Run the v8 ground-truth Stockfish
  (depth 6, MultiPV 2 — Appendix A) on a **~1M gate-passed sample** to get `s1,s2`,
  label by the literal v8 rules 4–5, and fit `r2` (or `(r2,gini)`) to that label;
  freeze the threshold. Production filtering then runs on **visits only, zero SF**.
- **Dedup:** Bloom filter or drop; exact global set won't fit.
- **Watch** datagen nodes/move — low node counts make the visit proxy noisier and
  argue for the 2-gate filter + ground-truth calibration.

---

## Appendix A — ground-truth Stockfish (linrock nnue-data-v7-3072)

The CSV labels (`s1,s2`) for calibration must come from the *same* Stockfish
linrock used, so the cp scale matches the `100/150` thresholds. From the
nnue-data `Dockerfile` / `binpack_to_csv.sh`:

**Build** (`x86-64-bmi2` matches the box; `profile-build` = PGO):

```bash
git clone https://github.com/linrock/Stockfish /tmp/stockfish
cd /tmp/stockfish/src
git fetch origin && git checkout -t origin/nnue-data-v7-3072
make -j profile-build ARCH=x86-64-bmi2
mv stockfish ../nnue-data/stockfish-output-positions-csv   # rename per nnue-data
```

**Eval settings** (exact, from `binpack_to_csv.sh`):

```
setoption name PruneAtShallowDepth value false
setoption name Use NNUE value true
setoption name Threads value 1
setoption name Hash value 1024
```

**The rescore that produces `s1,s2`** — depth **6**, MultiPV **2**:

```
transform rescore filter_depth 6 filter_multipv 2 input_file <stockfish.binpack>
```

`filter_multipv 2` ⇒ the CSV's `sf_bestmove1_score` / `sf_bestmove2_score`;
`filter_depth 6` ⇒ those scores are depth-6 searches. For our calibration we feed
FENs over UCI with `MultiPV 2` + `go depth 6` (§5.3) — equivalent signal, no
Stockfish-binpack conversion. Per-position cost at depth 6 is tiny and shards
across cores (`Threads 1` per process, many processes), so ~1M positions is a
one-time job of minutes-to-tens-of-minutes.
