# Bonus C2 — KV cache quantization (`--cache-type-k q8_0 --cache-type-v q8_0`)

Host `Linux-x86_64` · llama.cpp `b10488` · model `Qwen3.5-0.8B-Q4_K_M.gguf` ·
`ngl=99` (Vulkan) · `ctx=2048` (`n_ctx_slot=512`) · `threads=6`

Two `make serve`-equivalent runs, same model, same flags except KV cache dtype:

```
.venv/bin/python labs/02-serve/serve.py --port 8091
.venv/bin/python labs/02-serve/serve.py --port 8091 -- --cache-type-k q8_0 --cache-type-v q8_0
```

Quality eval: 10 auto-scorable one-line arithmetic prompts (`temperature=0`,
`max_tokens=20`), scored by regex match on the expected integer answer.

| | Host RSS (MB) | Quality (10 prompts) | Latency avg/min/max (ms) |
|---|--:|--:|--:|
| fp16 KV cache (default) | 454 | 9/10 | 90 / 55 / 366 |
| q8_0 KV cache | 510 | 9/10 | 83 / 55 / 312 |

## Finding

No real win or loss to report — quality came out identical (9/10 both times,
and even the same wrong answer, "100 - 63" → "77" — so that's a model
capability gap, not something the cache precision caused), and latency only
moves within normal run-to-run noise (83ms vs 90ms average, on arithmetic
answers that are basically one token anyway).

What actually caught my attention was the RSS column going the wrong way —
510 MB with q8_0 vs 454 MB on fp16, so no savings, if anything slightly worse.
Took me a second to realize why: this server runs `-ngl 99`, meaning it's
fully offloaded to the iGPU through Vulkan, so the KV cache for every layer
lives in VRAM, not host RAM. I was watching the wrong side of the bus the
whole time. `--cache-type-k/v` would actually show up in `nvidia-smi` or
Vulkan device memory, not in `ps` — the RSS numbers above are just noise from
whatever the HTTP server and scheduler keep on the host, nothing to do with
the KV cache itself.

Even on the right device, I don't think the effect would have been big here
anyway. `n_ctx_slot` is only 512 tokens (2048 `ctx` split across 4
`--parallel` slots), and the model is 0.8B — at that size the KV cache is
already tiny, a few MB total, compared to ~500 MB of weights. Halving that
few MB with q8_0 wouldn't move the needle. To actually see this challenge do
something, I'd want two changes: run CPU-only (`-ngl 0`, so KV cache stays in
host RAM where I can measure it with `ps`) and push `--ctx-size` up to
several thousand tokens per slot, where the KV cache stops being a rounding
error next to the weights. Without those two changes, this isn't really
evidence that q8_0 KV cache doesn't help — it just never got a fair shot to
show it here.
