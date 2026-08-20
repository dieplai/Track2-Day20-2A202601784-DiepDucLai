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

**Don't trust the P95 column above at face value — it's contaminated by a
one-time warm-up artifact, and the real evidence is elsewhere.** Looking at
`benchmarks/locust-10_stats_history.csv`, the first ~25 requests of the
`load-10` run each took 28–33 seconds, then latency dropped to a steady 1.5–5s
band for the rest of the minute. That burst is the *first* time all 4
decode slots got used concurrently (the only request before it was the single
`make smoke` call) — some one-time allocation/scheduling cost on cold slots,
not steady-state behavior. Because `load-50` ran second, its slots were
already warm, so its P95/max (12000ms) come out *lower* than `load-10`'s
(31000/32673ms) despite 5x the load — the opposite of what you'd expect from
contention alone. Comparing raw P95 across the two runs would be reading noise.

The number that actually convinced me: **median latency**, which isn't skewed
by that one burst. It goes from 1700ms (10 users) to 10000ms (50 users) — a
**5.88x** increase for a 5x increase in offered load, i.e. latency degraded
almost exactly proportionally to load. That's corroborated by two
independent signals: effective concurrency (Little's Law) jumped from 8.4 to
41.5 against only 4 `--parallel` slots (occupancy/slot ratio 10.37), and
`make metrics` sampled while `load-50` ran shows `n_busy_slots_per_decode`
pinned at 3.9–3.94 of 4 slots for the entire 60s window with `requests_deferred`
holding at 40+ throughout — the server was never idle and was continuously
turning away requests it had no slot for. All three signals agree: **the
server is saturated by 50 users**, and most of that added latency is queue
time, not compute time (compute per request is still the ~400ms decode `make
bench` measured at 4x that concurrency's worth of tokens/sec).

To raise goodput@SLO here, the first knob I'd reach for is `--parallel`
(more concurrent slots), not threads or quantization. The bottleneck is
`n_busy_slots_per_decode` pinned at the slot ceiling while the GPU (`ngl=99`,
Vulkan iGPU) is doing the actual decode work — CPU threads are idle-ish per the
flat thread sweep in `01-tuning-tg128.md`, so there is compute headroom being
left on the table by only ever offering 4 slots. I'd raise `--parallel` to 8
and re-run `load-50`, watching whether `n_busy_slots_per_decode` still pins at
the new ceiling (more headroom exists) or median latency stops improving
(the iGPU's own decode throughput, not slot count, is now the limit).
