# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 7807.1 | 7807.6 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 4912.8 | 4913.0 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.3 | 6676.7 | 6677.1 |

Mean per stage (ms): embed **0.0** · retrieve **0.2** ·
llm **6465.5** · total **6465.9**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **goodput** is more useful than raw throughput because it specifically counts only the requests per second that met the Target Time-to-Fullness (TTFT) and Target Time-to-Poll (TPOT) targets.

In contrast, **raw throughput** ignores SLOs (specifically SLOs at saturation), meaning it counts all requests regardless of whether they met the required performance targets. G

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation** in GPU memory by storing the Key-Value (KV) cache in non-contiguous pages. This design allows the engine to remove the internal fragmentation that would otherwise waste most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill is compute-bound** and **decode is memory-bandwidth-bound**.

This is because the context states that prefill requires compute resources (like GPU memory or CPU cores) while decode requires memory bandwidth. By splitting them, the system can offload the compute-heavy part of the operation to a separate pool, allowing the compute-bound part to run f


## Which N16-N19 pieces are real

N16 Cloud/IaC: stubbed; N17 data pipeline: stubbed; N18 lakehouse: stubbed;
N19 vector search: stubbed (the lab uses six in-memory toy documents and keyword
overlap retrieval). N20 serving is real llama-server over HTTP. The LLM stage is
the expected bottleneck at 6465.5 ms (about 100% of total), so halving latency
would target decode work first: use the tuned thread/slot configuration or a
smaller/faster model before optimizing the sub-millisecond toy retrieval.
