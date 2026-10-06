# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Đậu Quang Ý
**MSSV:** 2A202602661
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 (AMD64)
- **CPU:** 13th Gen Intel(R) Core(TM) i7-13620H
- **Cores:** 10 physical · 16 logical
- **CPU extensions:** AVX2
- **RAM:** 15.7 GB
- **Accelerator:** Vulkan (device present)
- **llama.cpp asset đã tải:** llama-b10488-bin-win-vulkan-x64.zip
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Môi trường Windows ban đầu gặp lỗi mã hóa console CP1252 khi in tiếng Việt và ký tự đặc biệt; đã khắc phục bằng cách bật `$env:PYTHONUTF8 = '1'`. Toàn bộ gói phụ trợ cài mượt mà qua pip trong virtualenv `.venv`. Runtime llama.cpp bản prebuilt Vulkan và model Qwen3.5 0.8B tải hoàn tất nhanh chóng.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 7583 | 1615 / 1680 | 25.9 / 26.8 | 3188 / 3286 / 3286 | 38.6 |
| UD-Q2_K_XL | 0.39 | 7415 | 1640 / 2024 | 49.8 / 54.4 | 4781 / 5246 / 5246 | 20.1 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Bản 2-bit chậm hơn 1.92× (20.1 vs 38.6 tok/s), chỉ giảm 0.11 GB RAM. Trên RAM 16GB, chi phí ALU giải nén bit cao hơn tiết kiệm băng thông. Chất lượng 2-bit suy giảm mạnh, lặp từ, hoàn toàn không đáng đánh đổi.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.44 | 19000 | 30000 | 31000 | 8.4 | 0.0% |
| 50 | 0.54 | 30000 | 58000 | 58000 | 17.3 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.24×
- **P95 tăng:** 1.93×
- **Effective concurrency ở 50 users:** 17.3 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.88 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server bão hòa ở 10–15 users khi RPS chững lại (chỉ tăng 1.24× khi tải tăng 5×) và `requests_deferred` đạt 46. Theo Little's Law, phần P95 tăng vọt là queue time vì compute slots đã kín (3.88/4). Muốn nâng goodput@SLO, đổi `--parallel` lên 8 trước vì RAM 16GB còn thừa để mở rộng slot.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | local Windows 11 | stub |
| N17 Data pipeline | TOY_DOCS in-memory | stub |
| N18 Lakehouse | in-memory dictionary | stub |
| N19 Vector + features | keyword overlap | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 7459.2 ms
- **stage chiếm nhiều nhất:** llm (100.0% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Bottleneck 100% ở LLM decode do sinh token tuần tự. Đúng như kỳ vọng. Để giảm độ trễ 2×, tôi sẽ giảm `max_tokens` đầu ra, bật prefix caching và ghim `-t 5` trên P-cores để tăng tốc độ sinh token lên 42 tok/s.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Giảm số luồng CPU `-t` từ 10 xuống 5 (ghim hoàn toàn vào các P-cores)

```
before:  30.3 tok/s (-t 10)
after:   42.0 tok/s (-t 5)
speedup: 1.39×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Kết quả này phản ánh cơ chế hoạt động của kiến trúc **Intel Hybrid Architecture (Raptor Lake)** trên Core i7-13620H (gồm 6 Performance cores - P-cores và 4 Efficient cores - E-cores). Theo mặc định của hệ thống (`-t 10` tương ứng 10 physical cores), llama.cpp phân bổ luồng tính toán ma trận GEMM trên toàn bộ 6 P-cores và 4 E-cores. Vì các phép tính toán mạng nơ-ron yêu cầu barrier synchronization giữa các luồng sau mỗi block, toàn bộ các P-core xung nhịp cao (đạt 4.9 GHz) buộc phải chờ đợi các E-core xung nhịp thấp hơn (~3.6 GHz, IPC thấp) hoàn thành tác vụ. Hiện tượng "straggler effect" này kéo hiệu năng chung sụt giảm nghiêm trọng xuống chỉ còn 30.3 tok/s.

Khi hạ số luồng xuống `-t 5`, Windows Thread Director xếp toàn bộ các luồng tính toán vào các nhân P-core mạnh nhất. Việc loại bỏ các nhân E-core chậm chạp giúp triệt tiêu hoàn toàn độ trễ chờ đợi đồng bộ barrier và giảm tranh chấp bộ nhớ đệm L3 Cache giữa hai cụm nhân khác biệt. Nhờ đó, tốc độ sinh token tăng vọt lên **42.0 tok/s**, mang lại mức speedup **1.39×** hoàn toàn dựa trên cơ chế điều phối phần cứng mà không cần can thiệp GPU hay biên dịch lại mã nguồn.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** 

**Numbers:**

```
before:  
after:   
speedup: 
```

**Điều này nói lên gì mà deck chưa nói:**

(để trống)

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Điều ngạc nhiên nhất là bản 2-bit (`UD-Q2_K_XL`) lại chạy chậm hơn bản 4-bit (`Q4_K_M`) gần 2 lần trên CPU. Việc nén ít bit hơn không phải lúc nào cũng tăng tốc nếu chi phí tính toán giải nén bit (dequantize ALU overhead) vượt qua băng thông tiết kiệm được.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [x] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [x] Repo GitHub ở chế độ **public**
- [x] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Sử dụng Google Antigravity Coding Assistant để tự động hóa các tác vụ đo kiểm trên Windows PowerShell, ghi nhận dữ liệu Prometheus metric, vẽ biểu đồ terminal screenshot và hỗ trợ phân tích cơ chế kiến trúc vi xử lý Intel Hybrid.
