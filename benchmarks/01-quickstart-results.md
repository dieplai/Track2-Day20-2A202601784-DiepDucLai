# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=6` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 3087 | 57 / 63 | 5.6 / 5.7 | 409 / 417 / 417 | 178.3 |
| UD-Q2_K_XL | 0.39 | 3075 | 61 / 132 | 6.3 / 6.6 | 456 / 546 / 546 | 158.6 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.12x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

I'd say it's not worth it on this machine. UD-Q2_K_XL only saves 0.11 GB (22%
smaller) but it actually decodes 1.12x slower (TPOT P50 6.3ms vs 5.6ms), and
TTFT P95 is over 2x worse too (132ms vs 63ms — one of the 10 runs had a slow
prefill). Makes sense once I checked `make probe`: this box runs with `ngl=99`,
so decode happens on the iGPU through Vulkan, not on the CPU. That's exactly the
case the note above warns about — when decode isn't memory-bandwidth-bound on
the CPU, the extra dequant work from a heavier quant format can cost more than
the bytes it saves.

The quality side made the call pretty easy honestly. I stood up both quants at
once (Q4_K_M on port 8091, UD-Q2_K_XL via `serve.py --compare` on 8090) and sent
the exact same prompt to both — "Explain in 2 sentences why quantizing a language
model to fewer bits makes inference faster but can hurt accuracy," temperature 0
on both. Q4_K_M answered in two clean sentences. UD-Q2_K_XL started repeating
itself ("more fragile system that is more susceptible to small perturbations...")
and then just kept narrating its own output afterward. At 0.8B params there
isn't much redundancy left to squeeze out, so 2-bit seems to push this specific
model past whatever coherence cliff it has. I'm keeping Q4_K_M. The 2-bit
variant only makes sense if you're really starved for RAM or bandwidth and are
okay trading away coherence for it — and even then, check the actual answers
first, because this isn't a subtle degradation.
