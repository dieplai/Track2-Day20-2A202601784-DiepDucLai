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

No measurable win or loss here — quality is identical (same 9/10, same failure:
"100 - 63" → "77" both times, so it's a model-capability miss, not a
cache-precision artifact), and latency differs only within run-to-run noise
(83ms vs 90ms avg, both single-token-ish arithmetic answers).

The interesting result is the RSS column going the *wrong* way (510 MB with
q8_0 vs 454 MB fp16, i.e. no savings, if anything slightly more). The reason
is a measurement-boundary mistake, not a llama.cpp bug: this server runs with
`-ngl 99` (full GPU offload via Vulkan), so the KV cache for all layers lives
in **VRAM**, not host RAM. Host process RSS was never going to show KV cache
savings on this config — it's the wrong side of the PCIe bus to be watching.
`--cache-type-k/v` would show up in `nvidia-smi`/Vulkan device memory, not
`ps`, and the two RSS numbers above are just noise from the host-side HTTP/
scheduler buffers, not the KV cache at all.

Second reason the effect would be small even if measured on the right device:
`n_ctx_slot` here is only 512 tokens (2048 `ctx` / 4 `--parallel` slots), and
this is a 0.8B model — the KV cache at that size is already tiny in absolute
terms (a few MB total across all layers and slots) next to the ~500 MB of
model weights, so even a 2x KV cache compression from q8_0 would be a rounding
error against total footprint. This challenge would show a real RSS/VRAM delta
on a CPU-only run (`-ngl 0`, KV cache stays in host RAM) at a much larger
`--ctx-size` (many thousands of tokens per slot), where KV cache stops being
negligible relative to the weights. Both of those are changes I'd make before
concluding q8_0 KV cache "doesn't matter" in general — it wasn't given the
conditions to matter *here*.
