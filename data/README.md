# Data

Files here are gitignored. Nothing in this folder is needed to run the pilot —
the synthetic loader generates its own domains.

## Switching to real corpora

Put one file per domain named `<domain>.jsonl` or `<domain>.csv`:

```
data/books.jsonl
data/electronics.jsonl
data/kitchen.jsonl
data/clothing.jsonl
```

Each needs a text column (`text`, `review`, `reviewText`, or `content`) and a
label column (`label`, `sentiment`, `rating`, `overall`, or `stars`). Star
ratings are mapped 1-2 -> 0 and 4-5 -> 1, with 3 dropped, following the Blitzer
convention. Binary 0/1 labels are used as-is.

Then set `loader: files` in the config:

```yaml
data:
  loader: files
  domains: [books, electronics, kitchen, clothing]
```

Nothing else changes.

## How much data

The main config needs `n_train + n_val + n_test` = 6,000 reviews per domain, and
class balancing discards the majority-class surplus, so aim for roughly 15,000
raw reviews per category. Subsample once and write the small files — do not keep
a multi-gigabyte dump in the project folder.

## Sources

- Amazon Reviews 2023 (McAuley Lab) - primary, wide category coverage
- Blitzer multi-domain sentiment dataset - the benchmark prior work reports on,
  worth including for comparability even though it is small
- IMDB (Maas et al. 2011), Yelp - the non-Amazon control that keeps the shift
  from being purely within-platform

A short subsampling script belongs in `scripts/` once you pick sources. Commit
it: how the subsample was drawn is a methodology detail an examiner may ask about.
