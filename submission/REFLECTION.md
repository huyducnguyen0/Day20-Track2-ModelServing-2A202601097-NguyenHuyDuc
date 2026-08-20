# Reflection - Day 20 Lab (Personal Report)

**Ho Ten:** Student
**Cohort:** Track2
**Ngay submit:** 2026-08-20

## 1. Hardware & runtime

- **OS:** Windows 11
- **CPU:** 13th Gen Intel(R) Core(TM) i7-13700H
- **Cores:** 14 physical / 20 logical
- **CPU extensions:** not reported by the Windows probe
- **RAM:** 15.6 GB
- **Accelerator:** Intel Iris Xe via Vulkan (llama.cpp reports Vulkan0)
- **llama.cpp asset:** llama-b10488-bin-win-vulkan-x64.zip
- **Model:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M primary + UD-Q2_K_XL comparison
- **Chay o dau:** local Windows laptop

**Setup story:** The GitHub release endpoint timed out, so I downloaded the
Vulkan runtime through a working mirror. The Hugging Face Xet client stalled at
zero bytes, so I used resumable direct HTTP downloads for the two Qwen files.
Qwen is an officially supported small-model path in this lab.

## 2. Do luong

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|---:|---:|---:|---:|---:|---:|
| Q4_K_M | 0.50 | 6654 | 618 / 746 | 41.8 / 43.5 | 3157 / 3477 / 3477 | 23.9 |
| UD-Q2_K_XL | 0.39 | 6391 | 635 / 709 | 44.7 / 46.3 | 3456 / 3623 / 3623 | 22.4 |

Q2 saves 0.11 GB but is 1.07x slower to decode and has a higher TPOT P50.
Both variants produced coherent answers in the smoke test, so Q4 is the
better serving choice here; Q2 is useful only when memory footprint matters
more than latency.

## 3. Serving under load

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Effective concurrency | Failures |
|---:|---:|---:|---:|---:|---:|---:|
| 10 | 0.53 | 16000 | 29000 | 31000 | 8.6 | 0.0% |
| 50 | 0.50 | 32000 | 58000 | 58000 | 16.8 | 0.0% |

- **Offered load tang 5x, throughput thuc tang:** 0.94x
- **P95 tang:** 2.00x
- **Effective concurrency o 50 users:** 16.8 so voi `--parallel=4` slots
- **Peak `n_busy_slots_per_decode`:** 3.89 / 4 slots

The server saturates at or below 50 users. The strongest evidence is that 5x
offered load produced only 0.94x throughput, while P95 doubled and the metrics
sample showed 3.89 of 4 slots busy with deferred requests up to 46. The extra
latency is therefore queue time after the decode slots fill. To improve
goodput at a fixed SLO I would test a larger `--parallel` value first, while
checking memory headroom; adding threads did not improve decode throughput.

## 4. Integration

| Day | Piece | Real or stub? |
|---|---|---|
| N16 Cloud/IaC | cloud integration not wired in this local run | stub |
| N17 Data pipeline | no external data pipeline | stub |
| N18 Lakehouse | no lakehouse attached | stub |
| N19 Vector + features | six in-memory toy documents with keyword overlap | stub |
| N20 Serving | `llama-server` over localhost HTTP | real |

**Latency split (mean of 3 queries):**

- embed: 0.0 ms
- retrieve: 0.2 ms
- llm: 6465.5 ms
- total: 6465.9 ms
- dominant stage: llm, about 100% of total

The LLM stage is the expected bottleneck and dominates the pipeline. Retrieval
is intentionally a toy in-memory fallback, so optimizing it would not change
the result. To halve latency I would first reduce decode work or use a faster
serving/model configuration, then replace the retrieval stub with a real index.

## 5. The single change that mattered most

**Change:** reduce server/benchmark threads from 14 to 7.

```text
before: 30.8 tok/s (tg128, -t 14)
after:  32.4 tok/s (tg128, -t 7)
speedup: 1.05x
```

The sweep peaked at 7 threads, then fell to 30.8 tok/s at the 14 physical-core
default, 29.8 at 20 threads, and 30.2 at 40. Decode is limited by memory
bandwidth and synchronization rather than by a lack of runnable CPU threads;
extra workers compete for the same Vulkan/CPU memory path. The smaller thread
pool reduces that contention and gives a modest but repeatable 5 percent gain.

## 6. Bonus

No optional bonus track was used. The required base track is complete.

## 7. Dieu lam toi ngac nhien nhat

The 2-bit file loaded slightly faster but decoded slower than Q4 on this Vulkan
setup. Smaller weights alone did not guarantee better latency because
dequantization and scheduling overhead mattered more than the saved bytes.
