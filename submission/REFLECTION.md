# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Nguyễn Thị Hương Giang
**Cohort:** 2A202600485
**Ngày submit:** _2026-05-06_

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Paste output của `python 00-setup/detect-hardware.py` vào đây, hoặc điền thủ công:

- **OS:** _Windows 10_
- **CPU:** _AMD Ryzen 7 5825U with Radeon Graphics_
- **Cores:** _8 physical / 16 logical_
- **CPU extensions:** _AVX2 / FMA / F16C_
- **RAM:** _14.8 GB_
- **Accelerator:** _CPU only_
- **llama.cpp backend đã chọn:** _CPU (AVX/NEON tuning)_
- **Recommended model tier:** _Qwen2.5-1.5B-Instruct (Q4_K_M)_

**Setup story** (≤ 80 chữ): những gì cần thay đổi để lab chạy được trên máy bạn (vd: dùng WSL2, install CUDA Toolkit, fall back sang Vulkan vì ROCm phiên bản kén, tắt antivirus để pip install nhanh hơn, v.v.):

Mình cài PowerShell 7, Python 3.11, Build Tools + CMake để build được `llama-cpp-python` trên Windows. Lúc đầu bị lỗi DLL/WinError 193, sau khi build lại bằng MSVC x64 thì chạy ổn. Mình build llama.cpp từ source để bật `/metrics`. Model thực tế chạy là TinyLlama (tải trước khi fix hardware probe) dù hardware.json gợi ý Qwen2.5-1.5B.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Paste bảng từ `benchmarks/01-quickstart-results.md` xuống đây (auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`).

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf | 1136 | 122 / 137 | 25.9 / 46.4 | 1699 / 1788 / 1807 | 38.6 |
| tinyllama-1.1b-chat-v1.0.Q2_K.gguf   | 255  | 189 / 227 | 20.6 / 27.4 | 1451 / 1936 / 2054 | 48.6 |

**Một quan sát** (≤ 50 chữ): Q4_K_M vs Q2_K trên máy bạn — số liệu nói gì? Quality đáng đánh đổi không?

Q2_K decode nhanh hơn (TPOT thấp hơn, tok/s cao hơn) nhưng TTFT lại cao hơn và chất lượng trả lời kém hơn. Với RAM 14.8 GB, Q4_K_M vẫn là lựa chọn hợp lý nếu ưu tiên chất lượng.

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50, paste tóm tắt bên dưới.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 0.89 | 8600 | 13000 | 15000 | 0 |
| 50 | 0.95 | 17000 | 27000 | 27000 | 0 |

**KV-cache observation** (từ `record-metrics.py`): peak `llamacpp:kv_cache_usage_ratio` ở concurrency 50 = _<0.XX>_, nghĩa là …

Peak `kv_cache_usage_ratio` = **0.00** khi concurrency 50. Điều này cho thấy KV cache chưa bị căng (context ngắn + n_ctx_seq nhỏ), chủ yếu bottleneck nằm ở CPU decode.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** _stub: localhost only_
- **N17 (Data pipeline):** _stub: in-memory dict_
- **N18 (Lakehouse):** _stub: in-memory (chưa có bảng)_
- **N19 (Vector + Feature Store):** _stub: TOY_DOCS_

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: _0 ms (stub)_
- retrieve: _0.1 ms_
- llama-server: _~8530 ms (query 1; các query khác ~4–5s)_

**Reflection** (≤ 60 chữ): bottleneck nằm ở đâu? Có khớp với kỳ vọng không?

Bottleneck rõ ràng ở llama-server (CPU decode). Phần retrieve gần như 0 ms vì đang dùng TOY_DOCS, nên tổng thời gian gần bằng thời gian suy luận.

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** _Giảm thread từ `-t 8` xuống `-t 4` (từ sweep thread)_.

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before: t=8 → 34.1 tok/s
after:  t=4 → 42.8 tok/s
speedup: ~1.26×
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

Khi tăng thread vượt quá số core hiệu quả, decode trở thành bài toán băng thông bộ nhớ (memory-bound). Với CPU 8C/16T, `-t 4` tận dụng đủ băng thông mà không bị tranh chấp cache/memory như `-t 8` hay `-t 16`. Vì vậy tốc độ tăng khi giảm thread, dù trực giác “nhiều thread = nhanh hơn” có thể sai với decode.

---

## 6. (Optional) Điều ngạc nhiên nhất

_(1–2 câu — không bắt buộc, nhưng người grader đọc tất cả)_

Locust 10 vs 50 users cho thấy RPS gần như không tăng, trong khi P95/P99 tăng mạnh — CPU decode là nút thắt chính.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit
- [x] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep)
- [ ] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [ ] `make verify` exit 0 (chạy ngay trước khi push)
- [ ] Repo trên GitHub ở chế độ **public**
- [ ] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
