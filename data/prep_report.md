# Data preparation report

Source: `McAuley-Lab/Amazon-Reviews-2023` (Amazon Reviews 2023, McAuley Lab), read directly from `raw/review_categories/<Category>.jsonl` over HTTP.

Label mapping: ratings 1-2 to negative, 4-5 to positive, 3 dropped (Blitzer convention). Sampling: per-class reservoir sampling over the streamed prefix, seed 0. Deduplication by normalised text hash. Minimum 5 words; truncated at 5000 characters.

| domain | category | n | neg | pos | streamed | dups dropped | neutral dropped | median words | p10-p90 |
|---|---|---|---|---|---|---|---|---|---|
| movies | Movies_and_TV | 25000 | 12500 | 12500 | 124946 | 2862 | 10740 | 32 | 8-142 |
| music | CDs_and_Vinyl | 25000 | 12500 | 12500 | 249421 | 8771 | 14214 | 35 | 9-160 |

## Limitation

Reservoir sampling is uniform over the *prefix that was streamed*, not over the full category file. Reviews beyond the point where the quota filled had no chance of selection. This is a deliberate trade against downloading hundreds of gigabytes (Books alone is 20.1 GB), and should be stated in the methodology chapter rather than left implicit.
