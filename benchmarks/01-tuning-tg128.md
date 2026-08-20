# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **6 physical · 12 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 189.5 | 99% |
| 3 | 190.7 | 100% |
| 6 | 191.3 | 100% |
| 12 | 189.9 | 99% |
| 24 | 190.4 | 100% |

**Best**: `-t 6` at 191.3 tok/s
**Slowest tested**: `-t 1` at 189.5 tok/s (1.01x spread)
**Against the physical-core default** (`-t 6`, 191.3 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=6 make bench
```

## Your explanation

Honestly there's no knee here at all. The whole curve sits within noise of each
other, 189.5 to 191.3 tok/s, from `-t 1` all the way to `-t 24` (twice the
logical core count). That's not what I expected going in — the deck says decode
should climb up to the physical core count and then fall off once threads start
fighting over memory bandwidth. So why so flat?

Checked `hardware.json` and it clicked: this machine runs with `ngl=99`, meaning
all the transformer layers get offloaded to Vulkan on the integrated GPU. Decode
is basically happening on the iGPU's own compute units and its own memory
bandwidth. The `-t` threads on the CPU side are only doing small dispatch/glue
work around each Vulkan call — cheap work that doesn't really scale up or down
with more threads. So thread count was never going to be the lever that moves
anything here. What actually matters on this box is GPU offload itself — I
tested that separately (`-ngl 0` vs `-ngl 99`) for REFLECTION §5, and that's
where the real 3.25x shows up.
