# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Diep Duc Lai
**Cohort:** A20-K4
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Fedora Linux (7.1.7-200.fc44.x86_64)
- **CPU:** 11th Gen Intel(R) Core(TM) i5-11400H @ 2.70GHz
- **Cores:** 6 physical / 12 logical
- **CPU extensions:** AVX2, AVX-512
- **RAM:** 15.2 GB
- **Accelerator:** NVIDIA RTX 3050 Ti (4 GB VRAM) present, but `make probe` picked
  **Vulkan** on the Intel UHD iGPU as the active backend (`ngl=99`, GPU offload ACTIVE)
- **llama.cpp asset đã tải:** `llama-b10488-bin-ubuntu-vulkan-x64.tar.gz`
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M (primary) + UD-Q2_K_XL (compare) (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi (local, không dùng cloud fallback)

**Setup story** (≤ 80 chữ): Disk chỉ còn 13 GB trống nên chọn Qwen3.5 0.8B thay vì
Gemma 4 E2B mặc định (~5.2 GB) để an toàn. Port 8080 đã bị process khác trên máy
chiếm sẵn nên mọi lệnh serve/load/smoke/metrics/pipeline đều chạy với
`LAB_SERVER_PORT=8091`. Không có `cmake` nên bonus B1 (build từ source) không khả
thi trên máy này — không ảnh hưởng base track. Mọi bước còn lại chạy trơn tru.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 3087 | 57 / 63 | 5.6 / 5.7 | 409 / 417 / 417 | 178.3 |
| UD-Q2_K_XL | 0.39 | 3075 | 61 / 132 | 6.3 / 6.6 | 456 / 546 / 546 | 158.6 |

**Quan sát** (≤ 60 chữ): Không đáng. UD-Q2_K_XL chỉ nhỏ hơn 22% nhưng decode
**chậm hơn 1.12x** (GPU-bound qua Vulkan nên ít bit không giúp gì). Đã hỏi cùng
câu hỏi qua `serve.py --compare` (port 8090) so với `make serve` (port 8091, đổi
vì 8080 bị chiếm): Q4_K_M trả lời mạch lạc, UD-Q2_K_XL lặp lại câu và tự thuật lại
output của chính nó — model 0.8B quá nhỏ để chịu được 2-bit.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 1.71 | 1700 | 31000 | 32000 | 8.4 | 0.0% |
| 50 | 4.27 | 10000 | 12000 | 12000 | 41.5 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 2.49×
- **P95 tăng:** 0.39× — con số này là nhiễu, xem giải thích bên dưới
- **Effective concurrency ở 50 users:** 41.5 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.94 / 4 slots

**Saturation reading** (≤ 80 chữ): P95 gây hiểu lầm — `load-10` chạy trước và 25
request đầu bị stall 28-33s vì đó là lần đầu cả 4 slot dùng đồng thời (cold start),
làm hỏng percentile của chính nó. Bằng chứng thật: **median** tăng 5.88×
(1700→10000ms), gần khớp tỉ lệ 5× offered load; effective concurrency 41.5 vượt xa
4 slots; `n_busy_slots_per_decode` giữ 3.9-3.94/4 suốt 60s, `requests_deferred` ≥40
liên tục — server bão hoà rõ ở 50 users, phần latency thêm là **queue time**, không
phải compute (compute/request vẫn ~400ms theo `make bench`). Sẽ tăng `--parallel`
(4→8) trước: GPU (Vulkan, `ngl=99`) làm decode chứ không phải CPU, sweep thread ở
`01-tuning-tg128.md` cho thấy CPU gần rảnh (spread 1.01×) — nút thắt là số slot.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | — | stub — chạy `localhost`, không cluster/Compose |
| N17 Data pipeline | `TOY_DOCS` | stub — list Python trong bộ nhớ |
| N18 Lakehouse | dict toy | stub — không có table format |
| N19 Vector + features | keyword overlap | stub — không dùng `--embed-url` |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`, chạy lần 2 khi
server đã idle/warm — xem `benchmarks/03-integration-results.md` cho cả hai lần):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 998.9 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): llm chiếm 100%, đúng như kỳ vọng vì retrieval bị stub
thành keyword overlap (gần 0ms). Điều bất ngờ: lần chạy đầu tiên (ngay sau
`load-50`) query 1 mất 8156.8ms — prefill riêng 7291ms cho chỉ 151 token — vì cache
prefix `LONG_CONTEXT` còn lạnh; lần 2 prefill mọi query chỉ còn 4 token (cache đã
ấm). Muốn giảm 2×: warm prefix cache trước khi phục vụ traffic thật, và không tắt
prompt caching mặc định của `llama-server`.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** GPU offload (`-ngl`) 0 → 99 (CPU-only decode → offload lên iGPU qua
Vulkan). Đây **không phải** kết quả của `make tune` — `make tune` (sweep threads,
`benchmarks/01-tuning-tg128.md`) lại cho kết quả gần như phẳng (1.01× spread từ
`-t 1` đến `-t 24`), và chính sự phẳng đó là lý do tôi đi tìm biến số khác. Đo trực
tiếp bằng `llama-bench` (không cần compiler/GPU riêng — binary Vulkan prebuilt đã
tải sẵn từ `make setup`), giữ `-t 6` cố định, chỉ đổi `-ngl`:

```
before:  57.63 tok/s  (-ngl 0, CPU only, 6 threads)
after:   187.23 tok/s (-ngl 99, Vulkan offload lên Intel iGPU)
speedup: 3.25×
```

**Tại sao nó work:** Ở `-ngl 0`, toàn bộ 24 layer decode chạy trên CPU, và decode
là memory-bandwidth-bound — mỗi token cần đọc lại toàn bộ trọng số của mô hình từ
RAM qua bus chia sẻ giữa 6 core, nên dù có nhiều thread, băng thông RAM là trần
cứng chung (đây cũng chính là lý do đường cong thread-sweep phẳng: thêm thread
không mở thêm băng thông). Ở `-ngl 99`, phần lớn layer chạy trên iGPU qua Vulkan,
dùng bus bộ nhớ và pipeline compute riêng của GPU — tách hẳn khỏi băng thông RAM
mà CPU đang tranh chấp — nên throughput tăng 3.25× mà không cần đổi thread hay
quantization gì cả. Điều này cũng giải thích ngược lại vì sao `make tune` (chạy ở
`ngl=99` mặc định) cho kết quả phẳng: một khi cổ chai đã chuyển từ CPU sang GPU,
số thread CPU chỉ còn làm việc dispatch nhẹ, không còn là biến số quyết định nữa.
Kết quả này khác kỳ vọng ban đầu từ deck (vốn tập trung vào core count CPU) — trên
máy này, layer offload mới là đòn bẩy thật, không phải thread count.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B2 — `make sweep-gpu` (GPU offload `-ngl` sweep, `bonus/sweeps/gpu-offload-sweep.py`).
Cũng làm **B4** — challenge C2 (KV cache quantization, `--cache-type-k/v q8_0`),
xem `benchmarks/bonus-c2-kv-cache-quant.md`: không thấy chênh lệch RAM hay
latency đáng kể, và lý do là KV cache nằm trên VRAM (do `ngl=99`) chứ không phải
host RSS đang đo, cộng với `n_ctx_slot=512` trên model 0.8B khiến KV cache vốn
đã nhỏ không đáng kể so với trọng số model — một kết quả "không có tác dụng"
được giải thích rõ cơ chế, thay vì một con số đẹp không rõ tại sao.

**Numbers:**

```
before:  58.6 tok/s   (-ngl 0,  CPU only)
after:   190.5 tok/s  (-ngl 32, full offload — identical to -ngl 99)
speedup: 3.25×
```

Full curve (`benchmarks/bonus-gpu-offload-sweep.md`): 0→58.6, 8→84.1, 16→118.0,
24→175.3, 32→190.5, 99→190.5 tok/s.

**Điều này nói lên gì mà deck chưa nói:** Đây là cùng phát hiện với §5 (GPU
offload, không phải thread count, là knob quyết định trên máy này) nhưng sweep
đầy đủ theo `-ngl` cho thấy thêm điều `make tune`/so sánh 2-điểm ở §5 không thấy
được: throughput tăng **gần tuyến tính** theo số layer chuyển sang GPU (~+25-35
tok/s mỗi 8 layer từ 0→24), không phải hàm bậc thang. Điều đó nghĩa là mỗi layer
có "giá" decode cố định riêng trên từng backend, nên offload từng phần thực sự là
một knob liên tục, không phải chọn nhị phân "tất cả CPU hay tất cả GPU" — hữu ích
khi model lớn hơn VRAM và bạn cần tìm điểm chia tối ưu thay vì chỉ bật/tắt `-ngl`.
Trên máy này, `-ngl 32` và `-ngl 99` cho **cùng một số** (190.5 tok/s) — dấu hiệu
model đã hết layer để chuyển trước khi hết VRAM (model chỉ ~500 MB, quá nhỏ so
với 8110 MiB VRAM rảnh mà `make probe` báo trên Vulkan) — nên sweep này không rơi
vào vùng "partial offload vì VRAM không đủ" mà deck cảnh báo; nó cần một model
lớn hơn nhiều để thấy vùng đó thật sự.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

`n_busy_slots_per_decode` giữ nguyên 3.9-3.94/4 suốt cả phút load-50, không hề dao
động — tôi kỳ vọng thấy nó nhấp nhô theo từng request hoàn thành/mới đến, nhưng
scheduler của llama-server lấp đầy slot trống gần như tức thì nên trông giống một
đường thẳng bão hoà hơn là một tín hiệu ồn.

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
