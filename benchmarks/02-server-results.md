# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=6` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 101 | 1.71 | 1700 | 31000 | 32000 | 8.4 | 0.0% |
| 50 | 252 | 4.27 | 10000 | 12000 | 12000 | 41.5 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **2.49x** (50% of linear) |
| P95 latency | **0.39x** |
| Effective concurrency at 50 users | 41.5 vs `--parallel 4` slots (occupancy/slot ratio 10.37) |

**Saturated.** Throughput delivered only 2.49x for 5x the offered load, and effective concurrency (41.5) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

P95 grew no faster than throughput (0.39x vs 2.49x), so this server still has headroom at 50 users.

## Your reading

I almost took the P95 column at face value and then double-checked it against
`benchmarks/locust-10_stats_history.csv` — glad I did, because it's misleading.
The first ~25 requests of the `load-10` run each took 28-33 seconds, then
latency dropped to a normal 1.5-5s band for the rest of the minute. That burst
lines up exactly with the first moment all 4 decode slots got used at once (the
only request before it was the single `make smoke` call) — reads like a
one-time cold-start cost, not steady-state behavior. Since `load-50` ran right
after, its slots were already warm, so its P95/max (12000ms) actually comes out
lower than `load-10`'s (31000/32673ms) despite carrying 5x the load. That's the
opposite of what contention alone would predict, so comparing raw P95 across
the two runs is basically comparing noise to noise.

What convinced me instead was the median, since it isn't dragged around by that
one burst: 1700ms at 10 users to 10000ms at 50 users, a 5.88x jump against a 5x
increase in load — degrading almost exactly in proportion. Two other signals
back this up independently: effective concurrency (Little's Law) goes from 8.4
to 41.5 against only 4 `--parallel` slots, and `make metrics`, sampled while
`load-50` was running, shows `n_busy_slots_per_decode` pinned at 3.9-3.94 of 4
for the full 60 seconds, with `requests_deferred` sitting above 40 the whole
time. The server was never idle, and it was constantly turning requests away
for lack of a free slot. All three point the same direction — saturated by 50
users, and the extra latency is queue time, not the model getting slower
(compute per request is still around the ~400ms `make bench` measured).

If I wanted to raise goodput here, the first thing I'd try is `--parallel`
(more slots), not threads or a smaller quant. The decode work is happening on
the iGPU through Vulkan, and CPU threads are basically idle per the flat sweep
in `01-tuning-tg128.md` — so 4 slots looks like an artificial ceiling rather
than the GPU's actual limit. I'd bump `--parallel` to 8 and rerun `load-50`:
if `n_busy_slots_per_decode` still pins at the new number, there's more
headroom; if median latency stops improving instead, then the iGPU's own
decode throughput is the real limit, not the slot count.
