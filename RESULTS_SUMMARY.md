# Results summary: what replicated, and what to claim

Consolidates three runs of the main grid. Use this as the backbone of the
results chapter — each finding is graded by how well it survived replication,
and the wording column is what is safe to write.

## The three runs

| | split handling | status |
|---|---|---|
| **Run A** | one fixed split (`hash()` seeded, irreproducible) | superseded |
| **Run B** | one fixed split, different partition | superseded |
| **Run C** | 5 resampled splits, one per seed | **report this one** |
| **Run D** | Run C repeated under `tuning: source_val` | protocol sensitivity check |

Runs A and B each used a single arbitrary partition and were internally
consistent but not reproducible — the split generator was seeded with Python's
`hash()`, which is salted per process. Run C fixes this and varies the split
across seeds, so its confidence intervals include split variance.

A and B are still evidentially useful: they are two additional independent
partitions, so agreement across all three is stronger than agreement within one.
Report Run C; cite A and B as robustness checks.

---

## RQ1 — the transfer gap

| statistic | Run A | Run B | Run C |
|---|---|---|---|
| median gap | 0.076 | 0.070 | 0.078 |
| min / max | 0.026 / 0.152 | 0.033 / 0.126 | 0.033 / 0.134 |
| pairs ≥ 0.05 | 18/24 | 18/24 | 19/24 |

**Safe to claim.** Cross-domain gaps for classic models on modern Amazon data
run roughly 3 to 13 macro-F1 points, median around 8. Aggregate statistics are
stable across partitions.

**Not safe to claim.** Anything about a specific pair's gap magnitude to three
decimals. `clothing→music` read 0.125 in Run A and 0.063 in Run B — the same
quantity, halved, purely from re-partitioning. Report per-pair gaps with the
Run C standard deviations attached, and never without them.

**Worth a paragraph.** These gaps sit below the 0.10–0.20 commonly reported in
the 2006–2013 literature. Two measured contributions: larger and cleaner
samples, and source-set size. The `gapcheck6` versus `gapcheck6_small`
comparison showed 28 of 30 pairs had larger gaps at n=2000 than at n=8000 (mean
0.064 vs 0.050), so part of the "gap has closed" story is simply that modern
work trains on more data.

---

## RQ2 — the budget curve

**Safe to claim.** Reaching 80% recovery takes roughly 700–780 labelled target
examples under random sampling (interpolated; measured grid point is 1000).
Recovery is near-zero or negative at k=50: a handful of target labels can make
the model worse before it makes it better.

**Safe to claim.** The source model keeps contributing throughout the tested
range. Crossover — where target-only overtakes source+target — is reached in
only a minority of pair/strategy combinations, and the advantage at k=2000 is
under 0.01 macro-F1 either way. The practical reading is that source data stops
mattering *much* well before it stops mattering at all.

**Caveat to state.** `budget_to_80` on the raw grid is quantised. With points at
500 and 1000, any true crossing between them reports as 1000. Quote the
interpolated figure and say it is interpolated.

---

## RQ3 — acquisition strategy *(the headline)*

Interpolated budget saving versus random sampling, all ten estimates:

| run | ceiling def. | tuning | logreg | svm |
|---|---|---|---|---|
| A | ceiling | budget_cv | 1.73 | 1.69 |
| A | max_budget | budget_cv | 1.72 | 1.69 |
| B | ceiling | budget_cv | 1.75 | 1.72 |
| C | ceiling | budget_cv | 1.84 | 1.73 |
| D | ceiling | source_val | 1.82 | 1.71 |

Mean 1.74, range 1.69–1.84.

**Safe to claim.** Uncertainty sampling reaches 80% recovery on roughly 1.7×
fewer labels than random sampling — about 420–460 examples against 770–790.
This replicates across two classifiers, two ceiling definitions, three
independent split configurations and both validation protocols.

**Safe to claim.** The advantage is significant from k=250 upward: bootstrap
intervals for uncertainty and random do not overlap at k=250, 500 or 1000 for
either classifier in Run C.

**Not safe to claim.** An advantage at k=50 or k=100. Those intervals overlapped
once split variance was included. Earlier runs appeared to show separation there;
they did not measure the relevant variance.

**Safe to claim.** Self-training is indistinguishable from random sampling
(1.03–1.07 across runs) and is actively harmful at k=50, where mean recovery
went negative in two of three runs. Consistent with the literature.

---

## RQ4 — is any of this predictable?

| finding | Run A | Run B | Run C | verdict |
|---|---|---|---|---|
| JS divergence vs gap (ρ) | 0.777 | 0.678 | 0.664 | **robust** |
| A-distance vs gap (ρ) | 0.80 / 0.53 | 0.571 | 0.564 | unstable |
| distance vs *budget* | null | null | null | **robust null** |
| R² source / target | .51/.03 | .27/.22 | .39/.15 | direction only |

**Safe to claim.** The transfer gap correlates with Jensen–Shannon divergence
between the domains' unigram distributions (ρ ≈ 0.66–0.78, p < 0.02 in all
three runs).

**Safe to claim, and worth foregrounding.** The *annotation budget* is not
predictable from any distance measure tested — every correlation was
non-significant with intervals spanning zero, in all three runs. This is a
principled null, not a failure: recovery is defined as a fraction of the gap, so
normalising by (T − S) divides out precisely the variance that domain distance
explains. Distant pairs have bigger gaps but need proportionally similar effort
to close a given fraction of them. State the mechanism; it turns a null into a
finding.

**Report with care.** Source-domain identity explained more gap variance than
target-domain identity in all three runs, but the margin ranged from enormous
(0.51 vs 0.03) to negligible (0.27 vs 0.22). Report the direction as suggestive
and the magnitude as unstable. Do not build an argument on it.

**Safe to claim.** The gap is asymmetric — `electronics→music` 0.071 versus
`music→electronics` 0.134 in Run C — and the symmetric measures cannot express
this by construction. The asymmetric measures (OOV rate, coverage) did not
explain it either: the pair with the largest gap asymmetry had the *smallest*
OOV asymmetry. What drives directionality remains open, which is a legitimate
thing to say.

---

## The validation-protocol check (Run D)

Run C repeated with `tuning: source_val`, which never consults a target label
for model selection, against `budget_cv`, which cross-validates over the k
budgeted labels.

**Safe to claim.** The headline is unaffected: 1.82 / 1.71 under `source_val`
against 1.84 / 1.73 under `budget_cv`. The validation accounting did not drive
the conclusion.

**Expected, and worth pre-empting.** The RQ1 table is identical digit for digit
across the two runs. This is by construction, not a copy-paste error: S is the
k=0 source-only model, which has no target labels to tune on under either
protocol, and T is the ceiling, which tunes on the target's own validation split
in both. The budget tuning protocol only affects runs with k > 0.

**A finding in its own right.** The two protocols cross over around k=250.
Below it `source_val` is better — at k=50, logreg random recovery is 0.173
against 0.110 — because three-fold CV over 50 labels tunes on roughly 17
examples and is far too noisy, while source validation has 1,500 examples and is
stable despite the domain mismatch. Above k=250 the matched protocol wins
(0.694 against 0.674 at k=500). The practical reading: cross-validate over the
budget only once the budget is large enough to cross-validate over.

## Limitations to state before an examiner raises them

1. **n = 12 ordered pairs.** RQ4 is exploratory. Report direction and effect
   size; fit no predictive model.
2. **Sampling reads a prefix.** Reservoir sampling was uniform over the streamed
   portion of each category file, not the whole file (Books alone is 20.1 GB).
   Reviews past the point where the quota filled had no chance of selection.
3. **One platform.** All four domains are Amazon categories, so "domain shift"
   is partly category shift within one platform's user base. A non-Amazon
   control (IMDB, Yelp) would strengthen the external validity claim.
4. **Source size is a choice, not a constant.** n=2000 was selected to match the
   benchmark literature and because annotation budget only matters when labels
   are scarce. The gap check quantifies what a different choice would cost.
5. **Small-gap pairs excluded from aggregates.** Pairs below 0.05 were dropped
   from recovery aggregates because the ratio has a near-zero denominator; they
   remain in the RQ1 tables. Five of 24 in Run C. Say which.

---

## What is left to do

Experiments are complete. Remaining: commit (four result sets are currently
outside version control), then write.

Figure notes for the results chapter:

- **fig2** is the headline. The separation between uncertainty and random is
  visible without reading a number off it.
- **fig1** shows the gap asymmetry directly: `music → electronics` at 0.134 sits
  a few rows above `electronics → music` at 0.071, same two domains.
- **fig4** gives a better RQ2 sentence than the crossover table did. The
  crossover table mostly reports "never crosses", which reads as a non-result.
  The figure shows the benefit decaying from about 0.16 macro-F1 at k=50 to
  under 0.005 at k=2000 — so write it as *the source model is worth roughly 15
  points at 50 labels and nothing at 2000*, not as *the source always helps*.
- SVM extracts more from the source data than logreg at k=50 (0.23 against
  0.16). The two classifiers agree everywhere else, so this is the one place
  they meaningfully differ.
