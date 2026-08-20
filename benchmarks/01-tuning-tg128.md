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

No knee at all — the curve is flat within noise (189.5 to 191.3 tok/s, a 1.01x
spread) across every point from `-t 1` to `-t 24` (2x logical cores). That
contradicts the expected shape from the deck, where decode should climb to the
physical core count and then drop from memory-bandwidth contention. The reason:
`ngl=99` on this machine (see `hardware.json` / `make probe`) offloads all 24
transformer layers to the Vulkan backend running on the integrated GPU. Decode
happens almost entirely on the iGPU's compute units and VRAM bandwidth — the CPU
threads set by `-t` only handle the small amount of host-side dispatch/glue work
around each Vulkan call, which is cheap and does not scale with thread count.
So `-t` was never the lever that mattered on this box; it would matter if I forced
CPU-only decode (`ngl=0`), which is exactly what I tested separately for
REFLECTION §5 — see that section for the number that actually moved.
