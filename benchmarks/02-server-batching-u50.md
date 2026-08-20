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

Peak batch width was 3.94 of 4 slots, so basically every decode step in the
whole 60s window had all 4 slots busy. That's a lot lower than the effective
concurrency Little's Law gives in `02-server-results.md` at 50 users (41.5),
and at first that looked like a contradiction to me. It isn't, really — they're
just answering different questions. `n_busy_slots_per_decode` can't go above
`--parallel` (4) by definition, a slot is either busy or it isn't. Effective
concurrency counts everything currently in the system, including whatever is
sitting in the `requests_deferred` queue (which stayed above 40 for the whole
window) waiting its turn. So I'd trust both numbers, just for different
things: busy-slots tells you the server is never sitting idle, and effective
concurrency tells you how big the line behind that saturation actually is.
Put together they're really saying one thing: 4 slots is the real ceiling, and
everything past it just waits.
