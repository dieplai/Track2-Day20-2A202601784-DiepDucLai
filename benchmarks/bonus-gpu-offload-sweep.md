# Bonus - GPU offload sweep

Host `Linux-x86_64` · backend(s) `nvidia_cuda, vulkan` ·
llama.cpp `b10488` · `threads=6` · metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 58.6 | 1.00x | 31% |
| 8 | 84.1 | 1.44x | 44% |
| 16 | 118.0 | 2.01x | 62% |
| 24 | 175.3 | 2.99x | 92% |
| 32 | 190.5 | 3.25x | 100% |
| 99 | 190.5 | 3.25x | 100% |

Best: `-ngl 32` at 190.5 tok/s
-- 3.25x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding

Full offload is best, and nothing ran out — `-ngl 32` and `-ngl 99` give the
identical 190.5 tok/s, which is llama.cpp's own tell that the model has fewer
than 32 transformer layers and every layer was already moved by `-ngl 32`;
`-ngl 99` just clamps to the same "all of it." No VRAM ceiling was hit: this
model is ~500 MB total, trivially small next to both the free VRAM `make probe`
reported on the Vulkan device (8110 MiB free) and the discrete RTX 3050 Ti's
4096 MiB — so this sweep never entered the "partial offload, something didn't
fit" regime the note above warns about. That regime would need a model in the
multi-GB range on this hardware to actually show up.

What is interesting is the shape *between* 0 and full offload: throughput
climbs almost linearly with layer count (58.6 → 84.1 → 118.0 → 175.3 tok/s as
`-ngl` goes 0→8→16→24, roughly +25 to +35 tok/s per 8 layers moved), not a
step function. That says each layer's decode cost is genuinely additive
between the two backends — a CPU layer and a GPU layer both cost roughly their
own fixed price, so partial offload is not "all or nothing," it is a smooth
knob. On a bigger model that didn't fit fully in VRAM, this same shape is
exactly what you'd use to find the split that maximizes tok/s under a hard
VRAM budget: push `-ngl` up until the curve visibly bends over, and stop
there rather than at the theoretical max.
