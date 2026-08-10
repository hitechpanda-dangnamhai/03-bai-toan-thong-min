# BT03 — FORECAST · BÀI 0: BẢN ĐỒ TỔNG

> **Giáo trình đọc từ code thật.** Bản giảng theo file `mecom/ALGO-FORECAST-BT03-2026-08-07.md` (1958 dòng, 14 mục).
> Mọi con số kèm phép tính và vị trí `file.py:dòng`. Chỗ nào code không nói rõ thì ghi **CHƯA CHẮC** — không đoán.
>
> Nguồn: code trong `mecom/project/services/forecast/` · Số đo: Postgres `localhost:16024`, đo 2026-08-07
> Bài này giảng ngày 2026-08-10.

---

## LỘ TRÌNH — 10 BÀI TỪ 14 MỤC

| Bài | Mục | Nội dung | Trạng thái |
|---|---|---|---|
| **0** | §0 | Bản đồ tổng: 4 tầng · 3 job nền · 33 thuật toán | **đang học** |
| 1 | §1 | Chuẩn bị dữ liệu: rollup · censored demand · promo deflation · học hệ số `k` | kế tiếp |
| 2 | §2 | Phân loại chuỗi Syntetos–Boylan (ADI / CV²) | |
| 3 | §3.1–3.3 | Chuỗi trơn: `cold_start` · `seasonal_naive` · `autoets_theta_ensemble` | |
| 4 | §3.4–3.6 | Chuỗi gián đoạn: Croston/SBA + NBD · ADIDA · IMAPA | |
| 5 | §3.7–3.9 | `similar_item_transfer` · `cold_start_analog` · LightGBM global quantile | |
| 6 | §4 | Router · backtest rolling-origin · `choose_model` · `calibration_factor` | |
| 7 | §5–6 | Lịch Tết VN 3 pha · Hierarchical reconciliation MinT/WLS | |
| 8 | §7–8 | Scenario Fabric (Philox · copula · TailCalibrator) · khoảng tin cậy | |
| 9 | §9–14 | Insights · số đo thật · bất biến & cạm bẫy · 3 phát hiện code · phần "vì sao" | |

Thứ tự cứng: không có bản đồ thì mọi công thức chỉ là mảnh rời; không hiểu tầng chuẩn bị dữ liệu thì không hiểu vì sao model đọc `adjusted_units` chứ không đọc `units_sold`.

---

## 0.1 · MỘT CÂU ĐỊNH NGHĨA BT03

> BT03 nhận **event bán hàng lẻ tẻ theo giây**, trả về **phân phối cầu tương lai theo ngày cho từng SKU** — không phải một con số, mà là ba con số p10/p50/p90 (kèm p95/p99 khi được hỏi).

Đây là ranh giới giữa "làm forecast" nghiệp dư và chuyên nghiệp: **đầu ra là phân phối, không phải điểm**. Lý do nghiệp vụ: người mua hàng không cần "trung bình bán 10 cái", họ cần "nhập bao nhiêu để 90% khả năng không đứt hàng".

Ba cụm từ trong câu này được mổ kỹ ở phần **HỎI & ĐÁP** phía dưới.

---

## 0.2 · BỐN TẦNG — ĐỌC THEO CHIỀU DỮ LIỆU CHẢY

```
TẦNG 1 — THU        raw_events        ← POST /v1/events:ingest
   purchase.completed · order.returned · stock.out · price.changed · promo.scheduled
        │
        │  jobs/rollup.py :: run_rollup_once
        ▼
TẦNG 2 — CHUẨN BỊ   demand_daily(project_id, product_id, day,
                                 units_sold, stockout, price, promo_pct, adjusted_units)
        │
        │  jobs/forecast_run.py   học k → deflate promo → phân loại → chọn model → hiệu chỉnh
        ▼
TẦNG 3 — GHI        forecasts(project_id, product_id, run_id, horizon_day,
                              p10, p50, p90, model_used, data_window, calibration)
        │
        ▼
TẦNG 4 — ĐỌC        main.py: forecast:query · aggregate · promo-preview · insights · scenarios:*
                    (+ apply_calendar, + tổng hợp horizon, + reconcile)
```

**BẤT BIẾN CỦA BẢN ĐỒ.** Tầng 3 là **kết quả đã đông lạnh của một `run_id`**. Tầng 4 **không** chạy lại model — chỉ đọc bảng rồi biến đổi nhẹ (nhân hệ số lịch, cộng dồn theo horizon, hoà giải cấp bậc).

Hệ quả vận hành: API đọc nhanh và tất định; muốn số mới thì phải chạy job, không phải gọi lại API.

---

## 0.3 · BA JOB NỀN — CHU KỲ VÀ VÌ SAO LÀ CHU KỲ ĐÓ

Đo lại trực tiếp trong code ngày 2026-08-10, không tin file:

| Job | Biến môi trường | Mặc định | Quy ra | Dòng code |
|---|---|---|---|---|
| Rollup | `ROLLUP_INTERVAL` | 3.600 s | mỗi **1 giờ** | `rollup.py:272` |
| Forecast run | `FORECAST_RUN_INTERVAL` | 86.400 s | mỗi **1 ngày** = 3.600×24 | `forecast_run.py:1666` |
| Backtest | `BACKTEST_INTERVAL` | 604.800 s | mỗi **7 ngày** = 86.400×7 | `backtest_run.py:514` |

- **Rollup 1 giờ** — đơn vị bài toán là NGÀY. Dày hơn 1 giờ không tạo thêm thông tin, chỉ tốn CPU; thưa hơn thì dữ liệu hôm nay chưa sẵn khi có người gọi API.
- **Forecast 1 ngày** — mỗi ngày mới chỉ có đúng 1 điểm dữ liệu mới trên mỗi SKU. Chạy 2 lần/ngày cho kết quả gần như y hệt với gấp đôi chi phí.
- **Backtest 7 ngày** — backtest không sinh dự báo cho khách; nó trả lời "model nào đang đúng nhất cho SKU này". Câu trả lời đó đổi rất chậm, mà chi phí lại đắt nhất trong ba.

Ba con số này là một **đánh đổi được viết ra**, không phải số ngẫu nhiên: chi phí tính toán ↔ độ tươi của lựa chọn model.

---

## 0.4 · THANG 9 MODEL VÀ THỨ TỰ AI THẮNG AI

```
kv_state model_choice   (do BACKTEST ghi)   ─── THẮNG TUYỆT ĐỐI
        │ không có / SKU chưa đủ dữ liệu để backtest
        ▼
core/router.py          (chọn theo phân loại ADI/CV²)
```

Chín model trên thang: `cold_start` · `seasonal_naive` · `autoets_theta_ensemble` · `croston/sba` · `adida` · `imapa` · `lgbm_global` · `similar_item_transfer` · `cold_start_analog`.

> **Bằng chứng đo được (backtest) đè lên luật tay (router).** Router chỉ là phương án khi chưa có bằng chứng.

Đây đúng kiểu tư duy mà LUẬT-0 của dự án áp cho con người — ở đây được cài thẳng vào máy.

---

## 0.5 · BA THUẬT TOÁN CÓ CODE NHƯNG KHÔNG CHẠY

| # | Thuật toán | Tình trạng |
|---|---|---|
| 31 | PSI drift — `libs/featurelib/drift.py:17` | **không nối vào BT03** |
| 32 | Data-quality gate — `libs/featurelib/data_quality.py:11` | **không nối vào BT03** (§13.1) |
| 33 | Holiday factor hardcode VN — `libs/featurelib/holidays_vn.py:69` | **đã nghỉ hưu**, bị `calendar_effect` thay |

Giá trị của ba dòng này: nó chứng minh tài liệu được viết **từ code** chứ không từ ý định. 30/33 thuật toán chạy thật; 3 cái không, và nói rõ. Khi phỏng vấn, đây là thứ phân biệt người đọc code với người đọc slide.

---
---

# HỎI & ĐÁP — MỔ BA CỤM TỪ TRONG CÂU ĐỊNH NGHĨA

---

## CỤM 1 · "event bán hàng lẻ tẻ theo giây"

### Event là gì — và vì sao không phải "dữ liệu bán hàng"

**Event = bản ghi một sự việc đã xảy ra, có dấu thời gian, không sửa được về sau.** Khác hẳn "bảng doanh số": bảng là *trạng thái tổng kết*, event là *sự việc đơn lẻ*.

miniAI nhận đúng **5 loại** (`jobs/rollup.py:88-130`):

| event_type | Kể chuyện gì | Rollup làm gì |
|---|---|---|
| `purchase.completed` | 1 đơn đã thanh toán | `units += qty`, cộng dồn tiền để tính giá bình quân |
| `order.returned` | khách trả hàng | `units -= qty` vào **ngày TRẢ** |
| `stock.out` | SKU hết hàng | bật cờ `stockout = true` |
| `price.changed` | đổi giá niêm yết | ghi mốc `(ngày, giá mới)` |
| `promo.scheduled` | lên lịch khuyến mãi | tô `promo_pct` cho mọi ngày trong `[start, end]` |

Vì sao chọn kiến trúc event chứ không bắt khách gửi báo cáo ngày:

1. **Khách không phải xử lý trước.** Họ bắn cái họ có sẵn tại thời điểm xảy ra. Bắt khách tổng hợp = bắt khách viết một nửa hệ thống của mình.
2. **Một luồng nuôi cả 3 bài toán.** Cùng dòng event đó: BT01 (smartsearch) học cái gì bán chạy, BT02 (decision) chấm kết quả quyết định giá, BT03 dự báo. Mỗi bài một định dạng nhập thì chi phí tích hợp nhân ba.
3. **Sửa được quá khứ mà không nói dối.** Event bất biến; muốn sửa thì bắn event mới (`order.returned`). Bảng tổng kết bị ghi đè thì lịch sử biến mất.
4. **Chạy lại được (idempotent).** Rollup UPSERT theo khoá `(project_id, product_id, day)` (`rollup.py:241-256`) ⇒ chạy 1 lần hay 5 lần cho **cùng một kết quả**. Đây là tính chất bắt buộc của mọi job nền: job chết giữa chừng, chạy lại, không được nhân đôi số liệu.

### "Lẻ tẻ" — ba kiểu bất quy tắc, mỗi kiểu một hậu quả

**(a) Bất quy tắc theo thời gian.** Một SKU có thể nhận 40 event trong 1 phút (giờ vàng), rồi im 3 ngày. Không có nhịp cố định.

**(b) Bất quy tắc theo SKU.** SKU hot có event mỗi ngày; SKU đuôi dài có 6 event trong 90 ngày. Cùng một bảng, hai bản chất thống kê khác nhau hoàn toàn — chính điều này đẻ ra §2 (phân loại ADI/CV²) và cả nhánh model gián đoạn (Croston/ADIDA/IMAPA).

**(c) Vắng mặt ≠ thiếu dữ liệu.** Đây là chỗ chết người nhất. Ngày không có event `purchase.completed` cho SKU X nghĩa là **bán 0 cái** — một con số THẬT — chứ không phải "không biết". Nếu để trống, thư viện thống kê sẽ nội suy hoặc bỏ qua; chuỗi bị bóp méo theo hướng **thổi phồng mức nền**.

Chính vì (c) mà `run_rollup_once` không chỉ nhóm event lại, mà **lặp từng ngày** từ `min_day` tới `today` cho từng cặp `(project, product)` và điền `u = max(0.0, units.get(key, 0.0))` (`rollup.py:165,172`).

> ⚠ **CHI TIẾT CHÌM TRONG DÒNG 172.** `max(0.0, …)` — **cầu âm bị kẹp về 0**. Hôm nay 5 khách trả hàng mua từ tuần trước và không ai mua mới ⇒ `units = −5` ⇒ ghi 0.
> **Đánh đổi:** thà mất thông tin "ngày trả hàng nhiều" còn hơn để số âm chảy vào model (log của số âm, phương sai bịa, quantile vô nghĩa).

### "Theo giây" — lệch đơn vị và ba phép vá

`raw_events.event_time` chính xác tới **giây**. Đơn vị bài toán là **ngày**.

**LỆCH 1 — lệch lưới (giây → ngày):** gộp. Ví dụ phép gộp giá (`rollup.py:88-99`):

| Giờ | qty | unit_price |
|---|---|---|
| 09:14:03 | 2 | 100.000 |
| 09:14:55 | 1 | 100.000 |
| 17:02:11 | 3 | 90.000 |

```
units_sold = 2 + 1 + 3                          = 6
price_sum  = 2·100.000 + 1·100.000 + 3·90.000   = 570.000
price_qty  = 6
price      = 570.000 / 6                        = 95.000   ← bình quân GIA QUYỀN theo số lượng
```

Không phải trung bình cộng của 3 mức giá (`(100+100+90)/3 = 96.667`). Gia quyền theo lượng mới đúng nghĩa "giá hiệu lực của một cái bán ra ngày đó".

Ngày **không có** giao dịch thì không có giá bình quân → **carry-forward** mốc `price.changed` gần nhất (`rollup.py:178-181`); chưa từng có mốc nào thì để `NULL`.

**LỆCH 2 — lệch ngữ nghĩa:** hàng trả bị trừ vào **ngày trả**, không phải ngày mua (`rollup.py:100-109`). Đây là **lựa chọn có chủ ý**, không phải lỗi:

- Trừ vào ngày mua = *đúng hơn về kế toán*, nhưng **viết lại quá khứ**: chuỗi hôm qua đổi số sau khi model đã học và đã ra quyết định nhập hàng. Không tái lập được, không audit được.
- Trừ vào ngày trả = *hơi sai về kế toán*, nhưng chuỗi **chỉ tiến về phía trước** (append-only). Đây cũng là lý do có `max(0.0, …)` ở trên.

**LỆCH 3 — ranh giới ngày.** Sự kiện lúc 23:59:59 thuộc ngày nào, theo múi giờ nào? File khai một cảnh báo thật ở §11.5 về "kiểu ngày" trong `demand_daily` của tenant `p1`. Loại lỗi này không nổ, chỉ làm lệch chuỗi 1 ngày cho một tenant.

> ⚠ **CẠM BẪY ĐÃ TRẢ GIÁ THẬT — CỬA SỔ 120 NGÀY.**
> `run_rollup_once(pool, window_days=120)` chỉ đọc `raw_events` có `event_time >= now − 120 ngày` (`rollup.py:57-66`). Comment `rollup.py:49-52` ghi lại vụ **SIM-WORLD SW-1**: đường backfill nạp lịch sử dài hơn 120 ngày, rollup chạy với mặc định ⇒ **cắt mất phần lịch sử vừa nạp**. Không exception, không log đỏ.
> **Dạng tổng quát (dùng được khi phỏng vấn):** tham số mặc định an toàn cho đường chạy hằng ngày lại là con dao cho đường nạp lại lịch sử. Hai đường dùng chung một hàm, mặc định chỉ đúng cho một đường.

---

## CỤM 2 · "phân phối cầu tương lai theo ngày cho từng SKU"

### Mảnh 1 — "cầu": chữ quan trọng nhất cả câu

> **Cầu (demand) ≠ Bán (sales).** Quan hệ thật: `bán = min(cầu, tồn kho)`.

Ngày hết hàng, `units_sold = 0` — nhưng cầu ngày đó **không** bằng 0. Số liệu bán bị **kiểm duyệt** (censored) bởi chính tồn kho của mình. Đây là bẫy kinh điển:

```
hết hàng → bán 0 → model học "cầu giảm" → dự báo thấp → nhập ít → hết hàng sớm hơn → …
```

Vòng xoáy tự xác nhận. Đó là lý do có §1.2 (`rollup.py:211-238`):

```
adj[t] = max( units_sold[t], mean(adj[t−7 .. t−1]) )   nếu stockout[t] = true
adj[t] = units_sold[t]                                 nếu không
```

Ví dụ số: 7 ngày trước có `adj = [12, 9, 11, 10, 13, 8, 10]` → `mean = 73/7 = 10,43`.

| Tình huống hôm nay | Phép tính | adjusted_units |
|---|---|---|
| Hết hàng, bán được 3 rồi sạch kệ | `max(3 , 10,43)` | **10,43** |
| Hết hàng vào cuối ngày, vẫn bán 15 | `max(15 , 10,43)` | **15** (không hạ số thật xuống) |

Hai chi tiết dễ bỏ sót:

- Trung bình lấy trên **`adj` đã tính** (`adjusted_prev[-7:]`, `rollup.py:220`), tức **đệ quy trên chính cột bù** — 3 ngày hết hàng liên tiếp thì ngày thứ 3 lấy trung bình gồm cả hai ngày bù trước đó. Ưu điểm: không bị kéo tụt dần. Rủi ro: nếu cầu thật sự giảm đúng lúc hết hàng, cột bù giữ mức cao lâu hơn thực tế.
- **Mọi model đọc `adjusted_units`, không đọc `units_sold`** (`store/forecasts.py:26`). Câu này nên thuộc lòng — nó là ranh giới "dự báo cầu" vs "dự báo doanh số".

### Mảnh 2 — "tương lai theo ngày"

Bước thời gian = **1 ngày**, tầm nhìn `H` ngày. Bảng `forecasts` lưu **một dòng cho mỗi `(product_id, run_id, horizon_day)`**. Nghĩa là luôn trả lời được: con số này sinh lúc nào, bởi lần chạy nào, với model nào (`model_used`), cửa sổ dữ liệu nào (`data_window`), hệ số hiệu chỉnh nào (`calibration`).

Đây là **tái lập được (reproducibility)** ở mức cột bảng, không phải lời hứa.

### Mảnh 3 — "cho từng SKU"

Dự báo ở **cấp thấp nhất** (SKU), rồi mới cộng lên. Lựa chọn này đẻ ra một bài toán mới, giải ở §6:

> Tổng dự báo 100 SKU của ngành hàng **không** bằng dự báo trực tiếp của ngành hàng đó.

Người mua hàng nhìn số SKU, giám đốc nhìn số ngành hàng — hai số không khớp là mất niềm tin ngay. Đó là **hierarchical reconciliation** (MinT/WLS, `core/scenario/reconcile.py`).

**Chi phí của việc dự báo cấp SKU là phải hoà giải cấp bậc.**

### Mảnh 4 — "phân phối": chính xác nó là cái gì trong DB

Bảng `forecasts` lưu **đúng 3 con số**: `p10`, `p50`, `p90`.

| Ký hiệu | Đọc là | Nghĩa xác suất |
|---|---|---|
| `p10` | phân vị 10% | 10% khả năng cầu ≤ p10 — kịch bản ế |
| `p50` | trung vị | 50/50 — cầu ở trên hay dưới đều 50% |
| `p90` | phân vị 90% | **90% khả năng cầu ≤ p90** — kịch bản chuẩn bị hàng |

Đọc thành lời: `p90 = 25` ⇒ *"trong 10 ngày giống ngày này, khoảng 9 ngày cầu không vượt 25 cái."*

Còn "**kèm p95/p99 khi được hỏi**" phải hiểu thật thà: **p95/p99 KHÔNG được lưu**, chúng được **ngoại suy** từ 3 điểm bằng giả định đuôi lognormal (`quantiles.py:80`):

```
z(0,90)=1,2816   z(0,95)=1,6449   z(0,99)=2,3263
σ_log = ln(p90 / p50) / 1,2816
p_q   = p50 · exp( z_q · σ_log )
```

Chạy số với `p50 = 10`, `p90 = 18`:

```
σ_log = ln(18/10) / 1,2816 = ln(1,8) / 1,2816 = 0,58779 / 1,2816 = 0,45864
p95   = 10 · exp(1,6449 × 0,45864) = 10 · exp(0,75447) = 10 × 2,1265 = 21,27
p99   = 10 · exp(2,3263 × 0,45864) = 10 · exp(1,06699) = 10 × 2,9067 = 29,07
```

Ba điều rút ra:

1. **p99 ≈ 2,9 × p50** — đuôi dài hơn nhiều so với trực giác tuyến tính. Ai đặt hàng theo "p50 nhân đôi cho chắc" đang đặt **dưới** p99.
2. API **chỉ cho** hỏi `q ∈ {0,95 ; 0,99}` (`main.py:598-608`) — càng xa 3 điểm neo thì càng là **giả định**, không phải đo.
3. Response gắn nhãn `quantile_method = "lognormal_tail_extrapolation"` (`main.py:741`). **Hệ thống tự khai con số này là suy ra, không phải đo ra.** Chuẩn mực đáng học: đừng để người dùng không phân biệt được số đo với số suy.

Tương tự, p25/p75 là **nội suy tuyến tính** giữa 3 điểm (`quantile_interp`, `quantiles.py:12`).

> **GIỚI HẠN THẬT THÀ.** Cái BT03 trả về là **phác thảo 3 điểm của một phân phối** cộng luật nội suy/ngoại suy công khai — **không phải** phân phối đầy đủ. Muốn phân phối đầy đủ + **quan hệ phụ thuộc giữa các SKU và các ngày** (hôm nay ế thì mai có ế theo không?) thì phải sang **Scenario Fabric** §7 (Monte Carlo, copula). Đó là lý do §7 tồn tại — không phải đồ trang trí.

---

## CỤM 3 · Ranh giới nghiệp dư / chuyên nghiệp: "đầu ra là phân phối, không phải điểm"

Đây là luận điểm, nên chứng minh bằng 4 lập luận có số.

### Lập luận 1 — dự báo điểm **giấu** rủi ro 50%

Model nói "bán khoảng 10 cái/ngày", bạn nhập 10.

Con số 10 đó là **trung vị**. Theo đúng định nghĩa trung vị: **50% số ngày cầu sẽ vượt 10** ⇒ bạn **đứt hàng 50% số ngày**. Không ai chấp nhận mức phục vụ 50%, nhưng đó chính xác là điều xảy ra khi đặt hàng theo dự báo điểm.

Dự báo điểm không sai — nó **không có chỗ để chứa mức phục vụ**. Đơn vị thông tin bị thiếu.

### Lập luận 2 — chi phí bất đối xứng: **p90 không phải con số tuỳ tiện**

Bài toán người bán báo (newsvendor):
- `Cu` = chi phí thiếu 1 đơn vị (mất lãi + mất khách)
- `Co` = chi phí thừa 1 đơn vị (vốn đọng + lưu kho + rủi ro hỏng/lỗi mốt)

Mức phục vụ tối ưu = **tỷ số tới hạn**:

```
CR = Cu / (Cu + Co)   →  đặt hàng ở phân vị CR
```

| Ca | Cu | Co | Phép tính | Đặt ở phân vị |
|---|---|---|---|---|
| **A** — hàng lâu hỏng, biên lãi tốt | 30.000 | 3.000 | `30/(30+3) = 0,909` | ~91% ≈ **p90** |
| **B** — hàng tươi, hỏng trong ngày | 10.000 | 40.000 | `10/(10+40) = 0,20` | 20% = **p20**, *dưới* trung vị |

Ca A chính là **nguồn gốc thật của con số p90** — nó rơi ra từ tỷ lệ chi phí, không phải quy ước. Ca B: với rau tươi, quyết định đúng là nhập **dưới** trung vị và chấp nhận hết sớm.

> Một con số điểm **không thể** diễn đạt được cả ca A lẫn ca B. Cùng một dự báo "10 cái", hai ngành hàng phải đặt hai lượng khác nhau. Chỉ **phân phối** mới cho khách tự chọn phân vị theo cấu trúc chi phí của họ.

Đây cũng là chỗ BT03 nối vào BT02: forecast đưa phân phối, decision đưa chính sách chọn phân vị.

### Lập luận 3 — dự báo điểm **giấu độ rộng bất định**

| SKU | p10 | p50 | p90 | p90 − p10 |
|---|---|---|---|---|
| A | 8 | **10** | 12 | 4 |
| B | 0 | **10** | 40 | 40 |

Cùng dự báo điểm **10**. Nhưng:
- SKU A: nhập 12 là gần như an toàn, tồn kho an toàn nhỏ, ít vốn chôn.
- SKU B: muốn 90% phục vụ phải nhập 40 — gấp 4 lần trung vị. Vốn chôn khổng lồ; đây là SKU đáng cân nhắc **đổi chiến lược** (đặt theo đơn, hoặc chấp nhận mức phục vụ thấp hơn).

Dự báo điểm làm hai SKU này **trông y hệt nhau**. Mất mát thông tin dẫn thẳng tới quyết định sai.

### Lập luận 4 — hệ quả **đo lường**: phân phối buộc phải chấm bằng thước khác

| Kiểu đầu ra | Chấm bằng | Trả lời câu hỏi |
|---|---|---|
| Điểm | MAE, RMSE, **MASE** | "Dự báo lệch bao nhiêu?" |
| Phân phối | **Coverage**, **Pinball loss** | "Khoảng có **trung thực** không? p90 tôi hứa có đúng 90% không?" |

Vì thế dự án **tách hai họ metric** (§14.4) và ghim **hai** baseline chứ không một (§11.0, DB tri thức `facts`):

| fact key | Giá trị | Đọc là |
|---|---|---|
| `eval.forecast.mase.baseline` | **0,827** | sai số bằng 82,7% so với thước nền `snaive7` (naive mùa vụ tuần) |
| `eval.forecast.coverage.baseline` | **0,815** | 81,5% giá trị thật rơi vào `[p10, p90]`; đích lý thuyết 0,80 |
| `eval.baseline.tolerance` | 0,10 | dung sai chung của gate eval |

Bây giờ đọc bảng thật §11.2 bằng con mắt vừa học:

| project | model | coverage | MASE | Đọc ra |
|---|---|---|---|---|
| demoshop | `croston_auto` | **0,938** | 0,905 | khoảng **QUÁ RỘNG** — hứa 80% mà bao 93,8% |
| demoshop | `lgbm_global` | **0,784** | **0,782** | gần đích 0,80 nhất, sai số thấp nhất |
| demoshop | `seasonal_naive` | 0,692 | (nền) | khoảng **QUÁ HẸP** — hứa 80% chỉ bao 69,2% |

> ⚠ **CHỖ DỄ LỘ NGHIỆP DƯ NHẤT.** Người mới nhìn `coverage = 0,938` sẽ khoe "bao phủ 93,8%, tốt hơn 80%!". **Sai.** Coverage cao hơn đích là **lỗi hiệu chuẩn**, không phải thành tích: khoảng quá rộng ⇒ p90 bị thổi ⇒ khách nhập thừa ⇒ chôn vốn. Một khoảng `[0, ∞)` có coverage 100% và vô dụng 100%.
> Coverage là chỉ số **hai chiều đều xấu** — chệch lên cũng sai, chệch xuống cũng sai; đích là **đúng bằng lời hứa**. Chính vì `croston_auto` rộng như vậy nên nó là nhóm bị `width_factor` **nén** xuống sàn 0,5 ở §8.3.

### Tóm tắt ranh giới

| | Nghiệp dư | Chuyên nghiệp (BT03 đang làm) |
|---|---|---|
| Đầu ra | 1 con số | p10/p50/p90 (+p95/p99 khai rõ là ngoại suy) |
| Học từ | `units_sold` | `adjusted_units` — bù kiểm duyệt bởi tồn kho |
| Chấm bằng | MAE/MASE | MASE **và** coverage **và** pinball |
| "Coverage 0,94" | khoe | **báo động khoảng quá rộng** |
| Chọn model | 1 model cho mọi SKU | backtest per-SKU; bằng chứng đè luật tay |
| Số suy ra | trộn lẫn với số đo | gắn nhãn `lognormal_tail_extrapolation` |

---
---

# PHÁT SINH TRONG BUỔI HỌC — KHUYẾT TẬT "CHẠY LẠI TỪ ĐẦU MỖI LẦN KHỞI ĐỘNG"

Câu hỏi trong lớp — *"không lẽ mỗi lần demo lại phải mất công như vậy?"* — dẫn thẳng tới một lỗi sản phẩm thật, ghi trong DB tri thức là `W-JOB-SCHEDULE-STATE-ANCHOR`.

Cả ba loop nền đều viết theo khuôn *chạy ngay rồi mới ngủ*:

```python
while not stop_event.is_set():
    await run_backtest_once(pool)          # ← CHẠY NGAY, không hỏi gì
    await asyncio.wait_for(stop_event.wait(), timeout=interval)   # ← rồi mới ngủ
```

`backtest_run.py:514-522` · `forecast_run.py:1666-1674` · `rollup.py:272-280`, được `main.py:159,165,171` bật khi service khởi động.

**Điểm gốc của lịch là "lúc tiến trình sinh ra", không phải "lúc việc được làm xong lần cuối".** Container không nhớ gì về quá khứ — mỗi lần boot, nó tin rằng chưa ai từng chạy backtest.

| | Hiện tại | Đúng ra phải là |
|---|---|---|
| Backtest chạy lần cuối | 2026-08-06 10:20 (đọc `job_runs`) | — |
| Chu kỳ | 604.800 s = 7 ngày | 7 ngày |
| Boot bây giờ ⇒ | **chạy lại full** (đo thật 3.754 s / 180 SKU simworld1) | **không chạy gì**, ngủ tới đúng hạn |

Nó tệ hơn "bất tiện lúc demo": mỗi lần **deploy** là tự nã CPU vào chính mình; container bị OOM-kill lúc 2 giờ sáng là tự gây tải đỉnh đúng lúc không ai trực; restart 5 lần trong 1 giờ là chạy backtest 5 lần cho cùng một kết quả.

**CÁCH SỬA — neo lịch vào TRẠNG THÁI, không neo vào tiến trình.** Nguyên liệu đã có sẵn: bảng `job_runs` và index `idx_job_runs_status_finished_at`. Trước khi chạy, loop hỏi Postgres còn bao lâu tới hạn; chưa tới hạn thì ngủ, không làm gì.

- Toàn bộ phép tính thời gian nằm **trong Postgres** — một đồng hồ duy nhất, lệch giờ container không park nổi loop.
- Đọc lỗi thì **fail-open** (chạy) — hành vi degrade đúng bằng hành vi cũ, không bao giờ làm dữ liệu cũ đi.
- Kill-switch `FORECAST_JOB_SCHEDULE_ANCHOR=0` trả về hành vi cũ mà không cần build lại.
- Marker của loop (`<job>_loop`) **tách khỏi** dòng `job_runs` của job sản phẩm, vì `forecast_run`/`backtest_run` cũng được ghi bởi lần chạy **một tenant** từ API — một tenant xong không được phép thoả mãn lịch của loop phủ mọi tenant.

Nợ kèm theo, đã đặt tên `W-JOBRUNS-DURATION-ZERO`: `backtest_run.py` đặt `started_at` và `finished_at` **cùng lúc ở cuối job** ⇒ mọi dòng `job_runs` khai thời lượng ≈ 0; cộng với `except Exception: pass` ở cả ba loop, một job có thể **chết cả tuần mà không ai biết**.

---

# KIỂM TRA BÀI 0 — 3 CÂU, TRẢ LỜI BẰNG SỐ CỦA CHÍNH DỰ ÁN

1. Khách hỏi: *"Sao không chạy backtest mỗi ngày cho model luôn tươi?"* — trả lời bằng **đánh đổi** kèm con số chu kỳ, và nói rõ backtest ghi cái gì vào đâu.
2. Một SKU đang được phục vụ bởi `croston`. Muốn biết **vì sao là croston** chứ không phải router chọn — tìm ở bảng nào, key nào?
3. Tầng 4 (API đọc) có được phép gọi model để tính lại dự báo không? Trả lời có/không **và** hai hệ quả nghiệp vụ của thiết kế đó: một tốt, một xấu.
