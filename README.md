# protocheck

[![tests](https://github.com/UlisesMoyaSanchez/protocheck/actions/workflows/tests.yml/badge.svg)](https://github.com/UlisesMoyaSanchez/protocheck/actions/workflows/tests.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/UlisesMoyaSanchez/protocheck/blob/main/notebooks/protocheck_colab.ipynb)

Checks the **data** of a link-prediction experiment (its splits and negatives) for the
evaluation errors that inflate reported AUROC: held-out edges the encoder can still see,
negatives drawn from a different distribution than the positives, a missing model-free
baseline, and candidate columns that can never be positive. It does not need your model:
you give it the pairs your evaluation used, and it tells you which failure modes are present.

Needs only numpy, scipy and scikit-learn. No torch, no PyG.

## The four checks

| Check | What it flags | Needs |
|---|---|---|
| `edge_leakage` | held-out edges visible to the encoder (in either direction), shared with training, or used as evaluation negatives | `encoder_edges` |
| `negative_matching` | evaluation negatives whose column-degree profile does not mirror the held-out positives; training and evaluation negatives drawn differently; declared samplers that differ | `eval_neg` (+ `train_neg`) |
| `model_free_baseline` | computes the no-learning floor (column degree, preferential attachment, common neighbours, Adamic–Adar; resource allocation in homogeneous mode) and, given your AUROC, your margin over it | held-out positives (+ `model_auroc`) |
| `dead_candidates` | candidate columns with no positive anywhere, and evaluation negatives that land on them | train + held-out positives |

A check whose input you did not supply is reported as `unchecked`, **never** as `ok`.

## Install

```bash
pip install "git+https://github.com/UlisesMoyaSanchez/protocheck.git"
```

## Quick start

Save the pairs your evaluation used as `N x 2` arrays of `(row, col)` (`.npy` or CSV), then:

```bash
protocheck --train-pos train_pos.npy --heldout-pos heldout_pos.npy \
    --encoder-edges encoder_edges.npy --train-neg train_neg.npy --eval-neg eval_neg.npy \
    --n-rows 495 --n-cols 383 --model-auroc 0.91 --json report.json
```

or in Python:

```python
from protocheck import audit

report = audit(n_rows, n_cols, train_pos, heldout_pos,     # (2, N) integer arrays
               encoder_edges=encoder_edges,                 # what message passing sees
               train_neg=train_neg, eval_neg=eval_neg,
               model_auroc=0.91)
print(report.render())
report.save("report.json")
```

Only `train_pos` and `heldout_pos` are required; every extra input enables another check.
**`encoder_edges` matters most:** build it from the same object your model's encoder
receives, because the tool takes it as given.

To see every check fire, run the synthetic examples (each triggers one failure mode, and
`clean` triggers none):

```bash
python -m protocheck.examples.make_examples protocheck_examples
protocheck --train-pos protocheck_examples/leak/train_pos.npy \
    --heldout-pos protocheck_examples/leak/heldout_pos.npy \
    --encoder-edges protocheck_examples/leak/encoder_edges.npy \
    --eval-neg protocheck_examples/leak/eval_neg.npy --n-rows 300 --n-cols 250
```

## Input layouts

| Layout | Flags |
|---|---|
| Pair files (default) | `--train-pos F --heldout-pos F --n-rows N --n-cols M [--encoder-edges F] [--encoder-edges-rev F] [--train-neg F] [--eval-neg F]` |
| 0/1 association matrix (HMDD style) | `--matrix M.csv --heldout-pos F [--encoder-edges matrix]` (`matrix` states that the encoder saw the whole matrix) |
| `(row, col, label)` triples | `--labeled-train F --labeled-test F --n-rows N --n-cols M [--index-base 1] [--neg-index-base 0]` |
| Homogeneous, undirected graphs | add `--mode homogeneous` (`n_rows == n_cols`) |
| OGB `split_edge` | `protocheck.io.from_ogb_split_edge(split_edge, num_nodes)` → kwargs for `audit` |

Use `--index-base 1` for 1-based files. Some published folds store positives 1-based and
negatives 0-based in the same file; `--neg-index-base` reads those correctly. With a single
base the negatives silently shift by one row and column.

## Reading the report

Each finding is `ok`, `info`, `warn`, `fail` (`error` in the JSON) or `unchecked`, and carries the
numbers it was decided on. **Read the numbers, not only the status word:** the thresholds
(`protocheck.DEFAULT_THRESHOLDS`) are heuristics, not validated cut-offs.

- Degree-profile distances (total variation between degree-bin histograms) warn only when
  they exceed both their threshold and what sampling noise alone produces at your sample sizes.
- Fewer than 30 held-out positives is itself a warning: the floor and any margin over it
  are not readable at that size.
- `dead_candidates` judges the evaluation negatives you supply. A padded matrix scored with
  degree-matched negatives that avoid the empty columns is reported as `info`, not `warn`.
- The margin is computed against a floor that sees **the same edges your encoder saw**, so
  a leaked evaluation is compared with an equally leaked heuristic.

**Exit status:** `0` clean, `1` a check failed (`--fail-on-warn` also fails on warnings),
`2` usage error, `3` nothing failed but edge leakage could not be checked because no encoder
edges were given (`--allow-unchecked` turns this into `0`). In a pipeline, stop on any
non-zero status.

## What it cannot tell you

- Leakage that does not pass through exact edges, such as node features computed from
  held-out labels (e.g. a degree computed on the full matrix), is not detected. The scope note
  is printed in every report.
- It takes the supplied files as given: if `encoder_edges` is not what your code really feeds
  the encoder, it can report `ok` wrongly.
- It cannot tell a correct sampler with a different seed from a matched one, and it does not
  retrain anything: to learn how much of your AUROC was inflation, rerun your model on the
  corrected split.
- The margin is in AUROC only (no Hits@K or MRR). One relation is checked at a time. In
  bipartite mode memory grows with the square of the number of rows.

## Validation

- `pytest` runs 24 tests: each synthetic example triggers its own check, inputs that are
  missing are reported as `unchecked`, leakage through reverse relations and undirected edges is
  caught, small clean test sets are not flagged, and the scorers reproduce the reference
  implementations (the two parity tests run when torch is installed).
- On five published miRNA–disease association matrices under a 2×2 grid of protocols (edges
  seen / held out × uniform / degree-matched negatives), where the true state of every cell is
  known by construction, it flagged all 40 leaked audits and all 40 mismatched-negative audits,
  with no false alarm in the 80 clean ones. On ogbl-ddi its heuristic AUROCs match an
  independent implementation to 1e-9.
- An independent audit stress-tested it with synthetic cases built to break it and found no
  leaked set reported as clean once the encoder's edges were supplied. Its other findings are
  fixed in 0.1.0 (see `CHANGELOG.md`).

These show that the implementation does what it says; they do not show that the thresholds
are optimal.

## Citing

See `CITATION.cff`. The tool accompanies the paper *The Protocol, Not the Model* (manuscript
in preparation); its analysis code is at
[UlisesMoyaSanchez/microRNA_project](https://github.com/UlisesMoyaSanchez/microRNA_project).

## License

MIT, see `LICENSE`.
