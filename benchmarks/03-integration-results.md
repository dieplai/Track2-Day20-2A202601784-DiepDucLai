# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 1151.5 | 1151.6 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 607.4 | 607.5 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 1237.9 | 1238.0 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **998.9** · total **999.0**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **goodput** is more useful than raw throughput because it specifically accounts for **SLOs** (Service Level Objectives) and **TPOT** (Time-to-Pollout/Throughput Optimization Targets).

Raw throughput ignores SLOs, whereas goodput counts only requests per second that met the TTFT and TPOT targets. This means goodput ensures that the system's output aligns with the ser

**What problem does PagedAttention actually solve?**

> PagedAttention actually solves the problem of **internal fragmentation in GPU memory**.

While the context mentions that it removes fragmentation by storing KV in non-contiguous pages, the core benefit described is specifically the **reduction of wasted GPU memory**. By avoiding the need to cache data in contiguous memory blocks (like the standard `KVCache`), PagedAttention allows the GPU to utili

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when the model's **compute-bound prefill** (which is expensive) and **memory-bound decode** (which is slow) are competing for the same resources, causing the model to be stuck in a suboptimal state.

Specifically, this splitting occurs when:
1.  **Prefill is bound to compute:** The prefilling phase requires significant GPU/CPU time and resources.
2.  **Decode is 


## Which N16-N19 pieces are real

- **N16 Cloud/IaC** — stub. Everything runs on `localhost`, no cluster or Compose stack.
- **N17 Data pipeline** — stub. `TOY_DOCS` is just an in-memory Python list, not a real ingest job.
- **N18 Lakehouse** — stub. No table format at all, the toy dict stands in for it.
- **N19 Vector + features** — stub. `retrieve()` is keyword overlap (`embed: 0.0 ms`
  on every query), not a real embedding index — I never pointed `--embed-url` at
  `make serve-embed` for this run.
- **N20 Serving** — real. Every query above hit the actual `llama-server` on
  `:8091` over HTTP, so the `llm` timings and the answers are genuine model output.

`llm` owning 100% of the total isn't a surprise — with retrieval stubbed to
keyword overlap it costs basically nothing, so decode/prefill was always going
to eat the whole budget regardless of model size.

The table above is from the **second** time I ran `make pipeline`. The first
run is worth mentioning separately because that's where the actual interesting
result is. I ran it right after `make metrics` + `make load-50` had just
hammered the server, and query 1 alone took 8156.8ms — 7291ms of that was
prefill, for just 151 tokens, which is absurdly slow for this model. Queries 2
and 3 in that same run shared the same `LONG_CONTEXT` system prompt and
prefilled in 50ms for a similar token count. So I reran the whole pipeline a
second time on an otherwise-idle server, and prefill for every query dropped to
just 4 tokens — the shared `LONG_CONTEXT` prefix was already sitting in
`llama-server`'s prompt cache from the first run, so only the trailing,
per-query question needed prefilling at all. That's prompt/prefix caching doing
its job, which is exactly the effect `labs/03-integrate/README.md` points at:
the first hit on a shared prefix pays the full prefill cost once, and everyone
sharing that prefix afterward only pays for their own unique suffix. It also
explains why only query 1 was slow in the first run — it was the only one of
the three that actually hit a cold cache.

If I had to cut this pipeline's latency in half, I'd go after prefill on a
cache miss first — that 7291ms vs 20-129ms range is well over 50x depending on
cache state, way bigger than anything on the decode side. Concretely: warm the
cache deliberately (fire the fixed context prefix once before real traffic
starts) instead of letting the first unlucky user pay for it, and just leave
prefix caching on, which is the default anyway. A real N19 vector index would
help too by shrinking how much of `LONG_CONTEXT` gets stuffed into each prompt,
but based on what I measured here, the caching effect alone is the bigger win.
