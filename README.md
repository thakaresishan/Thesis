# Annotation Budgets for Cross-Domain Sentiment Classification

Experiment harness for the thesis *How Many Labels Does Domain Transfer Cost?*

```
pip install -r requirements.txt
export PYTHONPATH=src

python -m pytest tests/ -q                                   # protocol invariants
python -m tsa.runner  --config configs/pilot_synthetic.yaml  # pilot (~15 s)
python -m tsa.analyze --results results/pilot_results.csv    # RQ1-RQ3 tables
python -m tsa.plots   --results results/pilot_results.csv    # figures
```

`--dry-run` prints the grid size without running. `--resume` skips completed
cells, so a crash mid-grid does not cost the whole run. `--limit N` stops after
N cells for smoke testing.

## Evaluation spine

| symbol | meaning | where it comes from |
|---|---|---|
| **S** | source-only macro-F1 on target test | `run_type == "source_only"` |
| **T** | fully-supervised target ceiling | `run_type == "ceiling"` |
| **G** | transfer gap, `T − S` | derived |
| **recovery(k)** | `(score(k) − S) / G` | derived |

Recovery makes pairs of very different difficulty comparable, so the whole
results chapter reduces to one figure with one line per strategy.

## Run types in `results.csv`

| run_type | meaning |
|---|---|
| `source_only` | k = 0, no target labels. **This is S.** |
| `transfer` | source data + k labelled target examples |
| `target_only` | k labelled target examples alone (drives the crossover) |
| `ceiling` | all target training data. **This is T.** |
| `uda_self_training` | k = 0 with self-training on unlabelled target text |

`uda_self_training` is deliberately *not* S. Self-training at k = 0 is an
unsupervised adaptation baseline; folding it into the reference line would
corrupt S and therefore every recovery figure that divides by it.

## Protocol decisions

These are the parts an examiner probes, so each is a config flag with a default
and a written justification rather than a buried assumption.

**Hyperparameter selection — `protocol.tuning`.** Default `budget_cv`: k-fold CV
over the k budgeted target labels, training on source plus k−1 folds and scoring
on the held-out fold. Selection consults no label outside the budget, every
budgeted label is used in the final fit, and the same rule applies at every k.
The alternative `budget_val` carves a fractional holdout out of k, which is
simpler but silently switches protocol at small budgets when the holdout falls
below `min_val_size` — at k = 50 a 20% holdout is 10 examples. `source_val`
never touches target labels at all. Run `configs/main_source_val.yaml` alongside
`configs/main.yaml` to *demonstrate* the choice was material rather than assert it.

At k = 0 no target labels exist by construction, so CV is impossible and source
validation is used. The `tuning` column records what was actually used per row,
not what was requested, so fallbacks are auditable.

**Self-training selection — `protocol.st_selection`.** Default `quantile`
(top 20% by confidence per iteration). An absolute probability threshold is not
comparable across regularisation strengths — a model at C = 0.1 has probabilities
shrunk toward 0.5 and never clears 0.9, so self-training silently becomes a
no-op — nor across classifiers, since an SVM margin is not a probability. The
`threshold` mode is retained for comparison with the older literature.

**Active learning cold start — `protocol.cold_start_n`.** Uncertainty sampling
cannot rank without a model, so the first 25 examples are drawn at random. An
unseeded active learner is random sampling with extra steps.

**Nested budgets.** The 500-label set contains the 250-label set. Realistic
(annotators do not restart when the budget grows) and statistically efficient
(one trajectory yields every budget point).

**Vectoriser fitting — `features.fit_on`.** Default fits TF-IDF on source text
plus *unlabelled* target text. This uses no labels and is therefore free, but it
materially changes the OOV rate, so it is a declared flag.

**Seeds** control target sampling and training together. `test_source_only_baseline_is_deterministic`
guards this: S has no target labels and nothing stochastic, so a non-zero `S_sd`
means something is leaking into the baseline.

## Swapping in real data

Put one file per domain in `data/` as `<domain>.jsonl` or `<domain>.csv` with a
text column (`text`, `review`, `reviewText`, `content`) and a label column
(`label`, `rating`, `overall`, `stars`). Star ratings are mapped 1–2 → 0,
4–5 → 1, with 3 dropped, following the Blitzer convention. Then set
`data.loader: files` in the config. Nothing else changes.

## The synthetic generator

Synthetic domains exist so the harness can be tested before the corpora are
downloaded, and so the pilot gate has a known-good reference: if the pipeline
cannot recover a gap on data whose shift is literally a tuned parameter, the bug
is in the code rather than the data.

Signal model: each sentiment slot draws from the shared lexicon or the
domain-specific lexicon, then picks a polarity word with a class-conditional
probability. Shared terms carry weak signal (skew 0.72) but transfer; domain
terms carry strong signal (skew 0.95) but do not. `synthetic_shift` sets how
much evidence sits in the non-transferable lexicon, and is therefore the direct
knob on gap size:

| `synthetic_shift` | S (books→electronics) | T | gap |
|---|---|---|---|
| 0.30 | 0.850 | 0.947 | 0.098 |
| 0.50 | 0.795 | 0.962 | 0.167 |
| 0.60 | 0.768 | 0.964 | 0.195 |
| 0.70 | 0.629 | 0.966 | 0.337 |
| 0.80 | 0.475 | 0.970 | 0.495 |

The 0.3–0.6 range brackets the 5–20 point gaps reported on real Amazon pairs.
Polarity-flipped terms (`predictable`, `compact`, `long`) and topic filler add
the other two real failure modes: actively misleading features, and OOV.

## Layout

```
configs/     experiment definitions; the config IS the experimental record
src/tsa/
  config.py      dataclasses, validation, config hashing
  synthetic.py   controllable-shift domain generator
  data.py        loading, splitting, the files loader
  models.py      TF-IDF, classifiers, CV and holdout tuning
  strategies.py  random / uncertainty / self-training
  experiment.py  one cell -> rows across all budgets
  runner.py      grid expansion, resume, CSV append
  analyze.py     RQ1-RQ3 tables
  plots.py       three thesis figures
tests/       protocol invariants, not "does it run" tests
```

Every results row carries the config hash, so a table can always be traced back
to the exact configuration that produced it.
