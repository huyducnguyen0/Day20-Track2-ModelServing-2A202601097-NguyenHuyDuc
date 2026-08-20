# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=7` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 30 | 0.53 | 16000 | 29000 | 31000 | 8.6 | 0.0% |
| 50 | 29 | 0.50 | 32000 | 58000 | 58000 | 16.8 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.94x** (19% of linear) |
| P95 latency | **2.00x** |
| Effective concurrency at 50 users | 16.8 vs `--parallel 4` slots (occupancy/slot ratio 4.20) |

**Saturated.** Throughput delivered only 0.94x for 5x the offered load, and effective concurrency (16.8) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.94x while P95 moved 2.00x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

The server is saturated by 50 users: offered load rises 5x but delivered RPS
falls to 0.94x of the 10-user run, while P95 doubles from 29 s to 58 s. The
metrics sample confirms 3.89/4 busy slots and deferred requests up to 46, so
the added latency is queue time after compute slots are full. To raise
goodput@SLO I would first increase `--parallel` only after checking memory
headroom; it can reduce queueing by admitting more work, whereas more threads
already lost throughput in the sweep and cannot remove slot queueing.
