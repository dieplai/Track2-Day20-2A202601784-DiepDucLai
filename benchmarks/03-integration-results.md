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
- **N17 Data pipeline** — stub. `TOY_DOCS` is an in-memory Python list, not a real ingest job.
- **N18 Lakehouse** — stub. No table format at all; the toy dict stands in for it.
- **N19 Vector + features** — stub. `retrieve()` is keyword overlap (`embed: 0.0 ms`
  every query), not a real embedding index — I did not point `--embed-url` at
  `make serve-embed` for this run.
- **N20 Serving** — **real**. Every query above hit the actual `llama-server` on
  `:8091` over HTTP; the `llm` timings and answers are genuine model output.

Dominant stage is `llm` at 100% of total, which matches what I expected — with
retrieval stubbed to keyword overlap it costs sub-millisecond, so decode/prefill was
always going to own the budget regardless of model size.

The numbers above are from a **second** `make pipeline` run; the first run is worth
reporting too because the difference is the actual finding. First run: mean `llm`
998.9ms would have been misleading — the *real* first run (right after `make
metrics` + `make load-50` had just exercised the server) measured query 1 at
8156.8ms total, with `server: prefill 151 tok / 7291 ms` — 7.3 seconds to prefill
only 151 tokens. Queries 2 and 3 in that same run, sharing the same `LONG_CONTEXT`
system prompt, prefilled in **50ms** for 113-114 tokens. On this second run,
prefill for every query dropped to **4 tokens** (the model + llama-server's prompt
cache had already stored the shared `LONG_CONTEXT` prefix from the first run, so
only the trailing, per-query question needed re-prefilling). That is prompt/prefix
caching in action, exactly the effect `labs/03-integrate/README.md` flags: the
first hit on a shared prefix pays full prefill cost once, every subsequent request
sharing that prefix pays only for its unique suffix. It also explains why query 1
alone was the outlier in the first run — it was the only one of the three with a
genuinely cold cache.

If I had to halve this pipeline's latency: attack **prefill on cache-miss**, since
that's the single biggest lever visible here (7291ms → 20-129ms is a >50x range
depending on cache state, dwarfing the decode-side variance). Concretely: warm the
cache deliberately (send the fixed system/context prefix once before serving real
traffic) instead of relying on the first real user to pay for it, and keep prefix
caching enabled (`llama-server` does this by default — don't disable it with
`--no-context-shift` or a `/slots` reset between requests). A real N19 vector
index would help too, by shrinking how much of `LONG_CONTEXT` gets injected per
query in the first place, but the caching effect measured here is larger than
what a smaller retrieved context alone would save.
