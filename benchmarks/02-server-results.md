# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=10` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 25 | 0.44 | 19000 | 30000 | 31000 | 8.4 | 0.0% |
| 50 | 32 | 0.54 | 30000 | 58000 | 58000 | 17.3 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.24x** (25% of linear) |
| P95 latency | **1.93x** |
| Effective concurrency at 50 users | 17.3 vs `--parallel 4` slots (occupancy/slot ratio 4.33) |

**Saturated.** Throughput delivered only 1.24x for 5x the offered load, and effective concurrency (17.3) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.24x while P95 moved 1.93x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

Server bão hòa ngay ở ngưỡng **10–15 concurrent users**. Bằng chứng định lượng thuyết phục nhất:
1. **RPS Plateau:** Khi tải yêu cầu (offered load) tăng vọt **5×** (từ 10 lên 50 users), thông lượng thực nhận chỉ tăng nhẹ **1.24×** (từ 0.44 lên 0.54 req/s, chỉ đạt 25% mức tăng lý thuyết).
2. **Latency Inflation:** Độ trễ P95 phình to **1.93×** (từ 30,000 ms lên 58,000 ms).
3. **Queue Time vs Compute Time:** Dựa vào định luật Little, Effective Concurrency ở 50 users lên tới 17.3 trong khi server chỉ có 4 slots (`--parallel 4`), và metric `requests_deferred` đo được là 46. Điều này khẳng định hơn 60% thời gian trễ của P95 (khoảng ~28-30s cộng thêm) là thời gian chờ trong hàng đợi (Queue time), chứ không phải do thời gian tính toán thực tế (Compute time) bị kéo dài.

Nếu đặt mục tiêu bảo toàn Goodput@SLO (ví dụ SLO P95 ≤ 25s):
* **Knob thay đổi đầu tiên:** Tăng `--parallel` từ 4 lên **6 hoặc 8 slots**. Vì máy có 15.7 GB RAM và model Qwen 0.8B chỉ tốn 0.5 GB, ta còn thừa rất nhiều VRAM/RAM để phân bổ thêm KV cache cho các slot song song, giúp giải phóng hàng đợi deferred nhanh hơn gấp đôi.
* **Knob thứ hai:** Kết hợp hạ thread count `-t` về 5 hoặc 6 (tránh gánh nặng E-core straggler) để mỗi slot decode đạt tốc độ tối đa.
