# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **10 physical · 16 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 42.1 | 100% |
| 5 | 42.0 | 100% |
| 10 | 30.3 | 72% |
| 16 | 32.5 | 77% |
| 32 | 34.3 | 82% |

**Best**: `-t 1` at 42.1 tok/s
**Slowest tested**: `-t 10` at 30.3 tok/s (1.39x spread)
**Against the physical-core default** (`-t 10`, 30.3 tok/s): 1.39x

Use this in your run:

```bash
LAB_N_THREADS=1 make bench
```

## Your explanation

Đồ thị thread sweep cho thấy điểm bão hòa / sụt giảm xảy ra ngay khi vượt quá 5 luồng: tốc độ đạt đỉnh ở `-t 1` và `-t 5` (~42.1 tok/s), nhưng sụt giảm mạnh tới 28% xuống **30.3 tok/s** khi dùng `-t 10` (mặc định số physical cores).

Hiện tượng này bắt nguồn trực tiếp từ kiến trúc lai **Intel Hybrid Architecture (Raptor Lake)** của chip i7-13620H (gồm 6 Performance cores - P-cores và 4 Efficient cores - E-cores):
1. **P-core Isolation:** Khi chạy với 1 đến 5 luồng, hệ điều hành ưu tiên xếp luồng vào các nhân P-core xung nhịp cao (đạt tới 4.9 GHz), tận dụng tối đa IPC cao và L3 cache lớn mà không gặp chi phí đồng bộ.
2. **The Straggler Effect:** Khi nâng lên 10 luồng, các luồng bị phân bổ sang cả 4 nhân E-core (xung nhịp tối đa chỉ ~3.6 GHz, IPC thấp hơn). Trong llama.cpp, các phép nhân ma trận GEMM phân tán yêu cầu barrier synchronization giữa các thread sau mỗi block tính toán. Các nhân P-core nhanh buộc phải chờ các nhân E-core chậm xử lý xong, kéo tụt hiệu năng chung xuống bằng tốc độ của nhân chậm nhất.
3. **Hyperthreading & Contention:** Ở 16 và 32 luồng, việc chia sẻ pipeline phần cứng (Hyperthreading) và tranh chấp băng thông kênh nhớ khiến tốc độ chỉ quanh mức 32–34 tok/s, không thể bắt kịp cấu hình tập trung vào P-core. Cấu hình tối ưu trên máy là ghim `-t 5` hoặc `-t 1`.
