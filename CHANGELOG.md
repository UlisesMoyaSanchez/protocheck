# Changelog

## 0.2.0 — 2026-11-14

- **`input_echo` in the floor under leakage.** When held-out positives are in the encoder's
  input, the as-evaluated floor now includes `input_echo`: a pair scores 1 if it is an input
  edge, with column degree as the tie-breaker. The four topology heuristics never score a pair
  by its own edge, so on a leaked set-up they understated what the input gives away: on
  NIMGSA's published evaluation (a paper's own code, rows of held-out miRNAs scored together
  with the training rows) the old floor was 0.865 and the margin read `ok` at +0.07, while
  handing the input back scores 0.970, above the model.
- **A margin over a leaked floor is never `ok`.** With leakage present, `model_free_baseline`
  is `warn` at best, and the evidence carries `margin_interpretable: false`.
- The margin message now names the heuristic the reference floor actually came from
  (`reference_heuristic` in the evidence); it used to name the corrected floor's best heuristic
  even when the value was the as-evaluated one.
- Leak-free set-ups are unchanged: `input_echo` is not computed and every floor, status and
  message is as in 0.1.0.

## 0.1.0 — 2026-11-03

First standalone release, extracted from the paper's research repository
(`UlisesMoyaSanchez/microRNA_project`, commit `9b5ce65`), where its earlier history lives.

Includes the fixes made after an independent audit of the tool:

- The CLI exits with status 3 when edge leakage could not be checked (no encoder edges
  given), so a pipeline that stops on any non-zero status does not read it as a pass.
  `--allow-unchecked` accepts it.
- Degree-profile distances (TV) warn only when they also exceed what sampling noise produces
  at the observed sample sizes; fewer than 30 held-out positives is itself a warning; empty
  inputs raise a clear error.
- `dead_candidates` judges where the supplied evaluation negatives land, not only the share of
  empty columns.
- `--neg-index-base` for `(row, col, label)` files whose negatives use a different index base
  than their positives.
