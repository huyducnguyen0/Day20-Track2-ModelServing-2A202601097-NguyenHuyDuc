# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **14 physical · 20 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 31.7 | 98% |
| 7 | 32.4 | 100% |
| 14 | 30.8 | 95% |
| 20 | 29.8 | 92% |
| 40 | 30.2 | 93% |

**Best**: `-t 7` at 32.4 tok/s
**Slowest tested**: `-t 20` at 29.8 tok/s (1.09x spread)
**Against the physical-core default** (`-t 14`, 30.8 tok/s): 1.05x

Use this in your run:

```bash
LAB_N_THREADS=7 make bench
```

## Your explanation

The knee is at 7 threads (32.4 tok/s). Increasing to the 14 physical cores
reduces throughput to 30.8 tok/s, and 20/40 threads remain slower. Decode is
memory-bandwidth and synchronization limited, so extra workers compete for the
same Vulkan/CPU memory path instead of adding useful parallel work. I therefore
use `LAB_N_THREADS=7` for the serving run.
