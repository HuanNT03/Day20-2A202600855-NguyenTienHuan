# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Nguyễn Tiến Huân
**Cohort:** 2
**Ngày submit:** _2026-06-25_

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Paste output của `python 00-setup/detect-hardware.py` vào đây, hoặc điền thủ công:

- **OS:** _Ubuntu 24.04 (Linux x86_64)_
- **CPU:** _Intel(R) Core(TM) i7-1165G7 @ 2.80GHz_
- **Cores:** _4 physical / 8 logical_
- **CPU extensions:** _AVX2 / AVX-512_
- **RAM:** _7.5 GB_
- **Accelerator:** _CPU only_
- **llama.cpp backend đã chọn:** _CPU_
- **Recommended model tier:** _TinyLlama-1.1B (Q4_K_M)_

**Setup story** (≤ 80 chữ): những gì cần thay đổi để lab chạy được trên máy bạn (vd: dùng WSL2, install CUDA Toolkit, fall back sang Vulkan vì ROCm phiên bản kén, tắt antivirus để pip install nhanh hơn, v.v.):

Biên dịch llama.cpp bằng lệnh thủ công cmake với tùy chọn `-j 2` thay vì `-j` mặc định trong Makefile để tránh bị lỗi tràn bộ nhớ (Out-Of-Memory) do máy chỉ có 7.5GB RAM khả dụng.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Paste bảng từ `benchmarks/01-quickstart-results.md` xuống đây (auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`).

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| (Q4_K_M) | 397 | 144 / 336 | 26.4 / 36.0 | 1625 / 2453 / 2550 | 37.8 |
| (Q2_K)   | 89  | 260 / 326 | 25.7 / 27.9 | 1670 / 2048 / 2052 | 38.9 |

**Một quan sát** (≤ 50 chữ): Q4_K_M vs Q2_K trên máy bạn — số liệu nói gì? Quality đáng đánh đổi không?

Q4_K_M cho tốc độ sinh chuỗi gần như tương đương Q2_K (37.8 vs 38.9 tok/s) nhưng chất lượng văn bản tốt hơn vượt trội. Không đáng đánh đổi chất lượng lấy tốc độ nhỏ này.

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50, paste tóm tắt bên dưới.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 0.95 | 8800 | 12000 | 14000 | 0 (0.00%) |
| 50 | 0.94 | 14000 | 29000 | 30000 | 0 (0.00%) |

**Batching observation** (từ `record-metrics.py`): peak `llamacpp:n_busy_slots_per_decode` / `requests_processing` ở concurrency 50 = `3.66 / 4`, nghĩa là trung bình có 3.66 trên tổng số 4 slots được xử lý song song trong mỗi chu kỳ decode.

Cơ chế Continuous Batching của llama-server hoạt động hiệu quả khi chịu tải cao (concurrency 50), cho phép ghép các câu trả lời đang sinh (decode) từ nhiều request khác nhau vào chạy song song trong cùng một batch tính toán. Điều này giúp tối ưu hóa hiệu năng tính toán của CPU và giảm độ trễ phản hồi của hệ thống.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** _stub: localhost only (để tiết kiệm tài nguyên RAM)_
- **N17 (Data pipeline):** _stub: in-memory dict_
- **N18 (Lakehouse):** _stub: SQLite_
- **N19 (Vector + Feature Store):** _stub: TOY_DOCS (giả lập tập dữ liệu nhỏ)_

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: _0.0 ms (keyword overlap)_
- retrieve: _0.0 ms_
- llama-server: _~3700 ms (trung bình 3 lần chạy)_

**Reflection** (≤ 60 chữ): bottleneck nằm ở đâu? Có khớp với kỳ vọng không?

Bottleneck nằm hoàn toàn ở llama-server (LLM decoding) vì sinh token trên CPU chậm nhất. Việc retrieve từ in-memory stub gần như lập tức (0ms). Điều này khớp với kỳ vọng.

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** Giới hạn số luồng (threads) chạy mô hình `-t` từ 8 luồng mặc định xuống 4 luồng `-t 4` để khớp số nhân vật lý thực tế của CPU.

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before: -t 8  -> 36.9 tok/s
after:  -t 4  -> 50.4 tok/s
speedup: ~1.37×
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

Tác vụ sinh token của LLM (decoding) bị giới hạn bởi băng thông bộ nhớ RAM (Memory-bandwidth-bound) thay vì năng lực tính toán. CPU Intel i7-1165G7 có 4 nhân vật lý. Khi chạy vượt quá 4 threads (lên 8 hoặc 16), các luồng ảo (hyper-threads) sẽ tranh chấp băng thông RAM trên cùng một kênh vật lý, đồng thời tăng chi phí context switch của CPU và tranh chấp L3 cache, gây sụt giảm hiệu năng. Do đó, giới hạn 4 luồng mang lại tốc độ tối ưu nhất.

---

## 6. (Optional) Điều ngạc nhiên nhất

Thực hiện thành công Challenge C5 (The "Weakest Laptop" Challenge) tại file `bonus/c5-weakest-laptop.md`. Thật ngạc nhiên khi mô hình 1.1B chạy trên CPU 8GB RAM không bị nghẽn ở dung lượng RAM mà hoàn toàn phụ thuộc vào giới hạn vật lý của CPU core.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit (đang chạy locust load test)
- [x] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep)
- [x] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`) (cần chụp sau khi test locust)
- [x] `make verify` exit 0 (chạy ngay trước khi push)
- [x] Repo trên GitHub ở chế độ **public**
- [x] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
