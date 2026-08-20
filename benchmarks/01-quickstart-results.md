# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=14` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 6654 | 618 / 746 | 41.8 / 43.5 | 3157 / 3477 / 3477 | 23.9 |
| UD-Q2_K_XL | 0.39 | 6391 | 635 / 709 | 44.7 / 46.3 | 3456 / 3623 / 3623 | 22.4 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.07x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

Q2 is not worth the speed trade-off on this machine: it is 0.11 GB smaller but
decodes 1.07x slower (22.4 vs 23.9 tok/s) and has a higher TPOT P50 (44.7 vs
41.8 ms). Both variants answered the smoke prompt coherently; I would keep Q4
for serving and choose Q2 only when the smaller memory footprint is the priority.
