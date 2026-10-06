# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 7224.6 | 7224.7 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 5213.0 | 5213.2 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 9939.9 | 9940.0 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **7459.2** · total **7459.3**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **Goodput** is more useful than raw throughput because it specifically accounts for **SLOs** (Service Level Objects) and **TTFT/TPOT** targets.

While raw throughput measures the total requests per second (requests/sec), it ignores the constraints that define what a "good" service is. Goodput filters out requests that do not meet these targets, ensuring that the syst

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation** in GPU memory.

By storing KV cache in non-contiguous pages, it removes the wasted space that would otherwise exist if all memory were contiguous (like a standard block of RAM). This allows the GPU to utilize more of its available memory bandwidth and compute capacity.

**When does splitting prefill and decode help?**

> Based on the context provided, splitting prefill and decode helps when the **compute-bound** nature of prefill prevents it from being fully utilized in a shared memory pool, while the **memory-bandwidth-bound** nature of decode allows for efficient parallelization.

Specifically, this optimization occurs when:
1.  **Prefill is compute-bound:** It requires significant processing power (e.g., GPU in


## Which N16-N19 pieces are real

* **N16 Cloud/IaC:** Stub (chạy local trên máy tính cá nhân Windows 11).
* **N17 Data pipeline:** Stub (dùng danh sách văn bản mẫu tĩnh `TOY_DOCS`).
* **N18 Lakehouse:** Stub (lưu trữ in-memory Python dictionary).
* **N19 Vector + features:** Stub (thuật toán tính điểm khớp từ khóa keyword overlap, không dùng vector database ngoại vi).
* **N20 Serving:** **Real** (`llama-server` native binary chạy mô hình Qwen3.5 0.8B).

**Nhận xét về Bottleneck:**
* Giai đoạn LLM chiếm tới **7459.2 ms (100% tổng thời gian)**, trong khi khâu embed/retrieve chỉ mất ~0.1 ms. Kết quả hoàn toàn khớp với kỳ vọng vì suy luận sinh token tuần tự (autoregressive decoding) luôn là khâu tốn tài nguyên nhất trong toàn bộ chuỗi RAG.
* **Chiến lược giảm latency 2×:** Ta phải tập trung tối ưu vào khâu LLM:
  1. **Rút ngắn output length:** Ép `max_tokens` hoặc chỉnh system prompt để model trả lời cô đọng (mỗi 100 token bớt đi tiết kiệm được ~2.5 giây decode).
  2. **Áp dụng thread tuning từ Phase 1:** Hạ `-t` từ 10 xuống 5 (tận dụng trọn vẹn P-core), giúp tốc độ sinh token tăng từ 30 tok/s lên 42 tok/s (tăng tốc trực tiếp 1.4×).
  3. **Bật Prefix Caching:** Tái sử dụng KV cache của system prompt cho tất cả các query để đưa prefill latency về gần bằng 0.
