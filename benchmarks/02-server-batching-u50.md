# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 14 samples over
65s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.88 of 4 slots (97%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 3624 |

Highest sampled value was **3.88 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Đỉnh batch width ghi nhận được là **3.88 / 4 slots** (đạt 97% công suất thiết kế của `--parallel 4`), chứng minh tính năng Continuous Batching hoạt động cực kỳ hiệu quả trong việc ghép các prompt và decode token vào chung các bước tính toán song song.

Con số 3.88 này khác với **Effective concurrency = 17.3** trong `02-server-results.md`. Hai số liệu không mâu thuẫn mà phản ánh hai góc nhìn khác nhau:
1. `n_busy_slots_per_decode` (3.88) đo lường **Compute Concurrency thực tế** bên trong server engine: tối đa 4 slot decode cùng lúc.
2. Effective concurrency (17.3, tính theo định luật Little: $\text{RPS} \times \text{Avg Latency}$) đo lường **System Occupancy tổng thể** từ góc nhìn client, bao gồm cả các request đang nằm chờ trong hàng đợi (metric `requests_deferred` chạm đỉnh 46).
Khi đánh giá năng lực tận dụng phần cứng của engine ta tin tưởng `n_busy_slots_per_decode`; còn khi đánh giá dung lượng chịu tải và độ nghẽn hàng đợi (queue time) đối với SLO của người dùng, ta dựa vào Effective Concurrency.
