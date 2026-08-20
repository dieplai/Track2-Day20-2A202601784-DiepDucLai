# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.94 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 20886 |

Highest sampled value was **3.94 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak batch width was **3.94 of 4 slots** — essentially every decode step during
the whole 60s sample had all 4 slots occupied. Little's Law in
`02-server-results.md` gives a much larger effective concurrency at 50 users
(41.5). The two numbers are not measuring the same thing, so they don't
disagree so much as answer different questions: `n_busy_slots_per_decode` is
capped at `--parallel` (4) by construction — it can never exceed the slot
count, because a slot is either busy or not. Effective concurrency counts
every request currently "in the system," including the ones sitting in the
`requests_deferred` queue (which held at 40+ for this entire window) waiting
for a slot to free up. So I trust **both**, for different claims: busy-slots
(3.94/4) is the proof that the server is compute-saturated — it is never
sitting idle — and effective concurrency (41.5) is the proof of how much
*queueing* is stacked up behind that saturation. Together they say the same
thing two ways: 4 slots is the real ceiling, and everything past it waits.
