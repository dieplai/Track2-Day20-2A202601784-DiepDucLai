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

Full offload wins, and nothing ran out of room. `-ngl 32` and `-ngl 99` land on
the exact same 190.5 tok/s, which is llama.cpp telling me the model has fewer
than 32 layers and everything was already moved by `-ngl 32` — `-ngl 99` just
clamps to the same "all of it." Makes sense given the model is only ~500 MB,
tiny next to the 8110 MiB free VRAM `make probe` reported on the Vulkan device
(or the RTX 3050 Ti's 4096 MiB, for that matter). So this sweep never really
got into the "something didn't fit, GPU has to keep fetching weights"
territory the note above describes — that would need a much bigger model than
what I'm running here.

The part I actually found interesting was the shape between 0 and full
offload. It climbs almost in a straight line — 58.6, 84.1, 118.0, 175.3 tok/s
as `-ngl` goes 0, 8, 16, 24 — roughly +25 to +35 tok/s per 8 layers, not some
step function that jumps once you cross a threshold. That tells me each
layer's decode cost is basically additive across the two backends: a CPU layer
costs about what a CPU layer costs, a GPU layer costs about what a GPU layer
costs, and partial offload isn't really an all-or-nothing switch, it's a
gradual knob. On a bigger model that doesn't fit entirely in VRAM, this is
exactly the curve you'd use to find the split that gets you the most tok/s
under a fixed VRAM budget — push `-ngl` up until you see the curve start to
bend, and stop there instead of chasing the theoretical max.
