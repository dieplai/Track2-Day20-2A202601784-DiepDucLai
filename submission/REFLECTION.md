# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Diệp Đức Lai
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
- **Accelerator:** máy có NVIDIA RTX 3050 Ti (4 GB VRAM) nhưng `make probe` lại
  chọn **Vulkan** trên iGPU Intel UHD làm backend chính (`ngl=99`, GPU offload
  ACTIVE) — hơi bất ngờ vì tôi nghĩ nó sẽ ưu tiên card rời trước
- **llama.cpp asset đã tải:** `llama-b10488-bin-ubuntu-vulkan-x64.tar.gz`
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M (primary) + UD-Q2_K_XL (compare), theo `models/active.json`

**Chạy ở đâu:** laptop của tôi, chạy local hoàn toàn, không dùng Colab/Kaggle.

**Setup story** (≤ 80 chữ): Ổ cứng lúc bắt đầu chỉ còn khoảng 13 GB trống nên tôi
đổi sang Qwen3.5 0.8B thay vì Gemma 4 E2B mặc định cho an toàn. Port 8080 trên máy
đã bị process khác chiếm từ trước nên tất cả lệnh serve/load/smoke/metrics/pipeline
tôi đều chạy kèm `LAB_SERVER_PORT=8091`. Máy không có `cmake` sẵn nên bỏ qua bonus
B1. Ngoài hai chỗ đó ra thì lab chạy khá trơn tru, không phải vá gì thêm.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 3087 | 57 / 63 | 5.6 / 5.7 | 409 / 417 / 417 | 178.3 |
| UD-Q2_K_XL | 0.39 | 3075 | 61 / 132 | 6.3 / 6.6 | 456 / 546 / 546 | 158.6 |

**Quan sát** (≤ 60 chữ): Không đáng đổi. Bản 2-bit chỉ nhỏ hơn 22% mà decode lại
chậm hơn 1.12x — vì decode ở đây chạy trên GPU (Vulkan) nên bớt bit không giúp gì,
ngược lại còn tốn thêm công dequant. Thử hỏi cùng một câu qua cả hai bản
(`serve.py --compare` so với `make serve`): bản Q4 trả lời gọn gàng, bản Q2 thì lặp
câu và bắt đầu tự bình luận về câu trả lời của chính nó. Model 0.8B chắc là quá nhỏ
để chịu được nén 2-bit.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 1.71 | 1700 | 31000 | 32000 | 8.4 | 0.0% |
| 50 | 4.27 | 10000 | 12000 | 12000 | 41.5 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 2.49×
- **P95 tăng:** 0.39× — nhưng đừng tin con số này, xem bên dưới
- **Effective concurrency ở 50 users:** 41.5 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.94 / 4 slots

**Saturation reading** (≤ 80 chữ): Cột P95 ở trên thực ra gây hiểu lầm. `load-10`
chạy trước, và ngay ~25 request đầu tiên bị đứng hình 28-33 giây — đơn giản vì đó
là lần đầu tiên cả 4 slot cùng được dùng một lúc (cold start), nên percentile của
chính run đó bị kéo méo. Số đáng tin hơn là **median**: tăng 5.88× (1700→10000ms),
gần khớp với việc tải tăng 5×. Cộng thêm effective concurrency vọt lên 41.5 so với
chỉ 4 slot, và `n_busy_slots_per_decode` giữ nguyên 3.9-3.94/4 suốt cả phút với
`requests_deferred` không dưới 40 — ba dấu hiệu này khớp nhau: server bão hoà rõ
ở mốc 50 user, và phần latency dôi ra là do chờ hàng (queue), không phải do tính
toán chậm đi (compute mỗi request vẫn quanh 400ms như lúc `make bench`). Nếu muốn
nâng goodput, việc đầu tiên tôi sẽ thử là tăng `--parallel` từ 4 lên 8, vì cổ chai
đang nằm ở số slot chứ không phải CPU — sweep thread ở phần 5 cho thấy CPU gần như
rảnh việc.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | — | stub, chạy trên localhost, không có cluster hay Compose gì cả |
| N17 Data pipeline | `TOY_DOCS` | stub, chỉ là một list Python trong bộ nhớ |
| N18 Lakehouse | dict toy | stub, không dùng table format nào |
| N19 Vector + features | keyword overlap | stub, chưa nối `--embed-url` |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, lấy từ lần chạy thứ hai khi server đã idle/warm
— chi tiết cả hai lần chạy nằm trong `benchmarks/03-integration-results.md`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 998.9 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): llm chiếm gần như toàn bộ, đúng như dự đoán vì retrieval
đang stub bằng keyword overlap nên gần như free. Cái làm tôi bất ngờ hơn là lần
chạy đầu (ngay sau khi vừa chạy `load-50`): query đầu tiên mất 8.1 giây, riêng
prefill đã 7.3 giây cho có 151 token — hoá ra vì cache của đoạn context dùng chung
còn lạnh. Chạy lại lần hai, prefill mỗi query chỉ còn 4 token vì cache đã ấm. Muốn
giảm latency đi một nửa thì việc đơn giản nhất là warm cache trước khi phục vụ
người dùng thật, thay vì để request đầu tiên gánh chi phí đó.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** GPU offload (`-ngl`), từ 0 lên 99 — tức từ decode hoàn toàn trên CPU
sang đẩy qua iGPU bằng Vulkan.

Cái này không đến từ `make tune`. Thật ra `make tune` (sweep thread count,
`benchmarks/01-tuning-tg128.md`) cho ra một đường gần như phẳng tuyệt đối — chênh
lệch giữa `-t 1` và `-t 24` chỉ 1.01×. Lúc nhìn kết quả đó tôi hơi khựng lại, vì
không giống gì so với những gì deck nói (đáng lẽ phải leo lên rồi rớt xuống ở số
core vật lý). Thay vì bỏ qua, tôi thử đổi biến khác: giữ nguyên `-t 6`, chỉ đổi
`-ngl` bằng `llama-bench` trực tiếp (không cần build gì thêm, binary Vulkan có
sẵn từ `make setup`):

```
before:  57.63 tok/s  (-ngl 0, CPU only, 6 threads)
after:   187.23 tok/s (-ngl 99, offload lên iGPU qua Vulkan)
speedup: 3.25×
```

**Tại sao nó work:** Ở `-ngl 0`, cả 24 layer decode đều chạy trên CPU. Decode là
việc bị chặn bởi băng thông bộ nhớ (memory-bandwidth-bound) — mỗi token sinh ra
cần đọc lại gần như toàn bộ trọng số model từ RAM, và 6 core đang chia nhau đúng
một đường bus RAM đó. Thêm thread không mở thêm băng thông, nên thread sweep mới
phẳng như vậy — không phải vì máy yếu, mà vì cổ chai không nằm ở số thread ngay
từ đầu. Khi bật `-ngl 99`, phần lớn layer chuyển sang chạy trên iGPU, dùng băng
thông bộ nhớ và pipeline compute riêng của GPU, tách hẳn khỏi cái bus RAM mà CPU
đang tranh nhau — nên throughput nhảy lên 3.25× mà không cần đổi gì khác. Điều
này cũng vừa giải thích ngược lại vì sao sweep thread ở trên phẳng: một khi cổ
chai đã dời từ CPU sang GPU, CPU chỉ còn làm việc dispatch nhẹ, số thread không
còn là biến quyết định nữa. Nói cách khác: trên máy này, layer offload mới là
đòn bẩy thật sự, không phải thread count — khác với hình dung ban đầu của tôi
khi đọc deck.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B2 (`make sweep-gpu`, sweep `-ngl` đầy đủ hơn phần 5) và B4 (challenge
C2, quantize KV cache bằng `--cache-type-k/v q8_0`). Chi tiết đầy đủ nằm ở
`benchmarks/bonus-gpu-offload-sweep.md` và `benchmarks/bonus-c2-kv-cache-quant.md`.

**Numbers (B2, GPU offload sweep):**

```
before:  58.6 tok/s   (-ngl 0,  CPU only)
after:   190.5 tok/s  (-ngl 32, full offload — giống hệt -ngl 99)
speedup: 3.25×
```

Cả đường cong: 0→58.6, 8→84.1, 16→118.0, 24→175.3, 32→190.5, 99→190.5 tok/s.

**Điều này nói lên gì mà deck chưa nói:** Đây vẫn là cùng một câu chuyện với phần
5 — GPU offload là knob quyết định, không phải thread — nhưng sweep đủ điểm giữa
0 và 99 cho thấy thêm một điều mà so sánh 2 điểm ở trên không thấy: throughput
tăng gần như tuyến tính theo số layer chuyển qua GPU (mỗi 8 layer cộng thêm
khoảng 25-35 tok/s), không phải một bước nhảy đột ngột. Vậy là mỗi layer có một
"giá" decode riêng khá ổn định trên từng backend, và offload từng phần không phải
kiểu chọn nhị phân tất-cả-hoặc-không-gì — cái này sẽ hữu ích thật sự khi model to
hơn VRAM và cần tìm điểm chia tối ưu. Trên máy tôi thì `-ngl 32` và `-ngl 99` ra
cùng một số, nghĩa là model đã hết layer để chuyển trước khi hết chỗ trên VRAM
(model chỉ tầm 500 MB, quá nhỏ so với 8110 MiB VRAM rảnh mà `make probe` báo) —
nên sweep này chưa chạm được vùng "hết VRAM" mà tài liệu cảnh báo, cần model lớn
hơn nhiều mới thấy được vùng đó.

Với B4 (KV cache q8_0), kết quả thẳng thắn mà nói là không thấy khác biệt gì rõ
rệt — cùng độ chính xác (9/10 câu số học, sai đúng một câu giống nhau ở cả hai
lần), latency cũng nằm trong khoảng nhiễu (83ms vs 90ms trung bình), và RSS thậm
chí không giảm mà còn nhỉnh hơn một chút (510 MB so với 454 MB). Lúc đầu tôi hơi
bối rối, nhưng nghĩ lại thì hợp lý: server chạy `-ngl 99` nên KV cache nằm trên
VRAM chứ không phải RAM host — đo RSS của process là đang nhìn sai chỗ ngay từ
đầu. Thêm nữa, `n_ctx_slot` chỉ có 512 token trên một model 0.8B thì KV cache vốn
đã bé tí so với trọng số model, nên dù có nén cũng khó thấy chênh lệch rõ. Kết quả
"không có tác dụng" này tôi vẫn thấy đáng ghi lại, vì nó cho biết cần điều kiện gì
(model lớn hơn, context dài hơn, hoặc chạy CPU-only) thì mới đo được cái mình
muốn đo.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Cái làm tôi bất ngờ nhất là `n_busy_slots_per_decode` giữ nguyên gần như y hệt ở
3.9-3.94/4 suốt cả phút chạy `load-50`, gần như không dao động. Tôi cứ nghĩ nó sẽ
nhấp nhô lên xuống theo từng request đến và hoàn thành, nhưng scheduler của
llama-server lấp chỗ trống gần như ngay lập tức, nên nhìn vào đồ thị nó giống một
đường bão hoà phẳng lì hơn là một tín hiệu có nhiễu.

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
- [x] 5 screenshots trong `submission/screenshots/` (đã có 9, gồm cả optional)
- [x] `make verify` → **exit 0**
- [x] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [x] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
