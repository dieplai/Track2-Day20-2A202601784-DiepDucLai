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

Not worth it here. `UD-Q2_K_XL` is only 0.11 GB smaller (22%) but decodes **1.12x
slower** (TPOT P50 6.3ms vs 5.6ms) and has a **2.1x worse TTFT P95** (132ms vs 63ms,
a stray slow prefill in one of the 10 runs). This machine has `ngl=99` (Vulkan offload
to the iGPU is active per `make probe`), so decode is GPU-compute-bound rather than
CPU-memory-bandwidth-bound — the case in the note above where a heavier-quantized
format's extra dequant work costs more than the bytes it saves.

Quality confirms it's not a close call: I served both quants side by side
(`make serve --port 8091` for Q4_K_M, `serve.py --compare --port 8090` for
UD-Q2_K_XL) and sent the identical prompt ("Explain in 2 sentences why quantizing a
language model to fewer bits makes inference faster but can hurt accuracy.") to both
via `/v1/chat/completions` with `temperature=0`. Q4_K_M gave two coherent, on-topic
sentences. UD-Q2_K_XL degenerated into repeating the same clause ("more fragile
system that is more susceptible to small perturbations...") twice and then started
narrating its own output ("The second sentence is a garbled version of the
first."). At 0.8B parameters there is very little redundancy left to cut — 2-bit
quantization pushes this specific model past a coherence cliff. Verdict: keep
Q4_K_M. The 2-bit variant would only make sense on a RAM- or bandwidth-starved
machine willing to trade coherence for a smaller file, and even then only after
checking the answers don't degrade like this.
