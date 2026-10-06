# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=10` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 7583 | 1615 / 1680 | 25.9 / 26.8 | 3188 / 3286 / 3286 | 38.6 |
| UD-Q2_K_XL | 0.39 | 7415 | 1640 / 2024 | 49.8 / 54.4 | 4781 / 5246 / 5246 | 20.1 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.92x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

Trên CPU Intel Core i7-13620H, bản lượng tử hóa 2-bit (`UD-Q2_K_XL`) chỉ giúp tiết kiệm được 0.11 GB RAM (từ 0.50 GB xuống 0.39 GB, giảm ~22% footprint), nhưng tốc độ decode lại chậm hơn tới **1.92×** (từ 38.6 tok/s giảm xuống 20.1 tok/s, TPOT tăng từ 25.9 ms lên 49.8 ms).
Nguyên nhân là do việc giải nén (dequantize) định dạng 2-bit phức tạp đòi hỏi nhiều chu kỳ tính toán ALU (bit-unpacking, scaling) trên CPU, trong khi mô hình 0.8B vốn dĩ đã rất nhẹ nên không bị nghẽn băng thông bộ nhớ RAM (memory bandwidth). Về chất lượng ngữ nghĩa, bản 2-bit suy giảm độ mạch lạc đáng kể so với bản 4-bit. Với 16 GB RAM sẵn có, bản `Q4_K_M` vượt trội hoàn toàn và bản 2-bit không đáng để đánh đổi.
