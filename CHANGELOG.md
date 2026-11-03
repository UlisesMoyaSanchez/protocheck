# Changelog

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
