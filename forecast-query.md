# F4 · `POST /v1/forecast:query` — dự báo nhu cầu 1 SKU (P10/P50/P90)

> Viết 2026-08-12 theo khuôn 21 mục rút gọn. Mọi số trong bài **đo từ hệ đang chạy**
> (forecast `:16023`, Postgres `:16024`, tenant `demoshop` / `simworld1` / `apidocf4`)
> hoặc **đọc từ code**. Chỗ chưa kiểm được ghi rõ **CHƯA CHẮC**.

---

## 1. BUSINESS PROBLEM

Chủ shop hỏi: **"Mã hàng này 14 ngày tới bán được bao nhiêu — và xấu nhất/tốt nhất là bao nhiêu?"**

Không có API này thì việc nhập hàng chạy bằng cảm tính hoặc bằng "trung bình 28 ngày qua" (MA28).
Hai chỗ đau cụ thể:

- **Nhập thiếu** — chỉ nhìn số trung bình thì không thấy đuôi trên; ngày cao điểm hết hàng.
- **Nhập thừa** — ôm vốn vào SKU bán chậm.

Vì vậy API không trả **một** con số mà trả **dải P10–P50–P90**: P90 dùng để quyết tồn kho an toàn,
P50 để ước doanh thu, P10 để biết mức xấu nhất còn chấp nhận được. Đây là hợp đồng
`kb_contract:forecast-query-api`, feature `kb_feature:C-B7`.

---

## 2. API

| | |
|---|---|
| Method + route | `POST /v1/forecast:query` |
| Service | forecast (BT03), cổng **16023** |
| Handler | `services/forecast/app/main.py:573` `async def forecast_query` |
| Một dòng | Trả dự báo nhu cầu theo ngày (p10/p50/p90) + tổng cả kỳ cho **một** SKU |
| Đặc thù | Là endpoint **đọc** nhưng **có ghi** (xem mục 16) |

---

## 3. INPUT

Body JSON (handler tự đọc `await request.json()`, **không** dùng model Pydantic — hệ quả ở mục 21).

| Field | Kiểu | Bắt buộc | Mặc định | Miền giá trị | Ý nghĩa |
|---|---|---|---|---|---|
| `product_id` | string | ✔ | — | chuỗi không rỗng | SKU cần dự báo |
| `horizon_days` | int | ✘ | `14` | `1..56` (int thật, `7.5` bị chặn) | Số ngày dự báo, tính từ **ngày mai** |
| `quantiles` | list[float] | ✘ | — | tập con của `[0.95, 0.99]`, không rỗng | Xin thêm phân vị đuôi trên |
| `granularity` | string | ✘ | `"daily"` | `daily` \| `weekly` \| `monthly` | Gộp ngày thành kỳ trong `periods` |

Header bắt buộc: `Authorization: Bearer <api_key>` + `X-Project-Id: <tenant>`.
Rate-limit class = `query` (`main.py:380`).

---

## 4. CASE STUDIES (đã chạy thật 2026-08-12)

| # | Ca | Input | Đi đường nào | Kết quả đo được |
|---|---|---|---|---|
| C1 | Có mẻ batch sẵn | `demoshop / dt-cuongluc-ip15 / h=14` | `latest_daily` đọc bảng, **không** chạy model | 14 dòng, `model_used=autoets_theta_ensemble`, totals p50 = 69,78 · **0,043 s** |
| C2 | Chạm ngày lễ | cùng SKU, `h=28` | + `apply_calendar` | `calendar_effects` = quốc-khánh-2026 pre 3 ngày / in 1 / post 2; p50 ngày 01/09 = 5,804 = 5,047 × 1,15 |
| C3 | SKU dùng model học máy | `th-binh-nuoc-1l / h=28` | lgbm_global | `calendar_effects` **rỗng** (lịch đã nằm trong covariate `cal_factor` — chống nhân đôi) |
| C4 | Hàng bán lai rai | `mb-mayham-sua / h=14` | Croston → NBD | `totals_method=nbd_horizon_sim_2000`, totals = 7 / 11 / 15 (số nguyên vì rút từ phân phối đếm) |
| C5 | Xin đuôi + gộp tuần | `h=14, weekly, quantiles=[0.95,0.99]` | + `extend_quantiles` | mỗi ngày có thêm `p95`/`p99`; `periods` 2 kỳ × 7 ngày; `quantile_method=lognormal_tail_extrapolation` |
| C6 | Gộp tháng | `h=56, monthly` | nhóm theo `năm-tháng` | 3 kỳ: 19 ngày (13–31/08) · 30 ngày (9/2026) · 7 ngày (10/2026) |
| C7 | Có ngày hết hàng | `mb-ta-bobby-m / h=7` | đếm `stockout` | `censored_adjusted_days=6`; ví dụ 01/06 bán 3 nhưng `adjusted_units=4,714` |
| C8 | **SKU mới tinh** | `apidocf4 / adf4-tainghe-moi / h=7` | 404 → `_cold_start_analog` | `model_used=cold_start_analog`, `analog_of=[adf4-tainghe-a, adf4-tainghe-b]`, `data_window=null` |
| C9 | Không có gì cả | SKU bịa | analog rỗng | **404** `no demand history; not in catalog either (no similar products found)` |
| C10 | Cache model lạnh | `simworld1 / sku-0041 / h=56` | on-demand → train LightGBM **trong request** | **49,18 s** (lần sau 0,047 s) — xem mục 21 |

Ghi chú C1 vs C10: khác biệt duy nhất là **h ≤ 28 hay không**. Batch chỉ lưu `HORIZON_DEFAULT=28`
ngày (`forecast_run.py:43`), xin quá 28 là buộc phải tính mới.

---

## 5. VALIDATION

| Luật (main.py) | Vi phạm → |
|---|---|
| body phải là JSON object (`:576`) | 400 `INVALID_ARGUMENT` — *Request body must be a JSON object* |
| `product_id` chuỗi không rỗng (`:583`) | 400 — *Missing or invalid 'product_id'* |
| `horizon_days` int trong `1..56` (`:590`) | 400 — *'horizon_days' must be an integer between 1 and 56* |
| `quantiles` ⊆ `{0.95, 0.99}`, không rỗng (`:596`) | 400 — *'quantiles' must be a non-empty list drawn from [0.95, 0.99]* |
| `granularity` ∈ `{daily, weekly, monthly}` (`:609`) | 400 |
| Header auth + project (middleware `:348`) | 401 thiếu header · **403** `PERMISSION_DENIED` khi key thuộc tenant khác |

`bool` là `int` trong Python nên `horizon_days: true` lọt qua `isinstance(..., int)` và bị chặn ở
`< 1` — **CHƯA CHẮC** có test nào khoá hành vi này.

---

## 6. UPSTREAM DEPENDENCIES

| Thứ tự | Ai | Sinh ra gì | Chưa chạy thì sao |
|---|---|---|---|
| 1 | `POST /v1/events:ingest` hoặc `:backfill` | `raw_events` (`purchase.completed`, `stock.out`, `price.changed`, `promo.scheduled`) | không có gì để dự báo → 404 |
| 2 | Job **rollup** (`jobs/rollup.py:65`, mỗi **3600 s**) | `demand_daily` (1 dòng/SKU/ngày) | SKU vừa bán hôm nay chưa vào dự báo |
| 3 | Job **backtest** (mỗi **604800 s**) | `kv_state` khoá `model_choice:<tenant>:<sku>` + `width_factor` | không có → rơi về router tự chọn (mục 11) |
| 4 | Job **forecast_run** (mỗi **86400 s**) | `forecasts` 28 ngày/SKU | vẫn trả lời được, nhưng tính on-demand (chậm hơn) |
| 5 | smartsearch `GET /internal/similar-products` | hàng xóm cho cold-start | analog không chạy → 404 thẳng |
| 6 | Bảng `calendar_events` (`project_id=''` dùng chung) | hệ số lễ/Tết | không nhân hệ số, dự báo vẫn hợp lệ |

**Bẫy đo được:** SKU vừa `products:upsert` xong **chưa có embedding** — job `embed_backfill`
chạy mỗi **300 s** (`smartsearch/app/jobs/embed_backfill.py:78`). Trong khoảng đó
`/internal/similar-products` trả `{"items":[]}` nên cold-start analog **không chạy** và khách
nhận 404. Đo 2026-08-12 trên `apidocf4`: upsert 14:54 → còn 404; ~15:00 có embedding → ra analog.

---

## 7. TABLE / COLUMN / FORMAT

DB `miniai_forecast` (Postgres `:16024`). Đường đọc chính:

### `demand_daily` — PK `(project_id, product_id, day)`
| Cột | Kiểu | Ý nghĩa |
|---|---|---|
| `day` | date | ngày (một dòng/ngày, **không có lỗ**) |
| `units_sold` | numeric | số bán thô |
| `stockout` | bool | ngày hết hàng |
| `price` | bigint | giá hiệu lực (minor-unit) |
| `promo_pct` | numeric | % giảm giá ngày đó |
| `adjusted_units` | numeric | **số dùng để dự báo** — đã bù ngày hết hàng |

Record thật (`demoshop / mb-ta-bobby-m`): `2026-06-01 | units_sold=3 | adjusted_units=4.714 | stockout=t`
— bán 3 vì hết hàng, ước lượng nhu cầu thật là 4,714 (trung bình 7 ngày trước).

### `forecasts` — index `(project_id, product_id, created_at DESC)` + `(…, run_id)`
`run_id` (`r_YYYY-MM-DD`) · `horizon_day` (date) · `p10/p50/p90` (numeric) · `model_used` ·
`data_window` (`"2026-04-01..2026-08-12"`) · `calibration` (jsonb) · `created_at`.
Đo 2026-08-12: **325.563 dòng / 95 MB / 10 mẻ** (không có ai dọn — mục 21).

### `kv_state` (k, v jsonb)
`model_choice:<tenant>:<sku>` → `{"model": "...", "width_factor": 0.8746, "empirical_coverage": 0.857}` ·
`promo_uplift_k:<tenant>` → `{"k": ..., "n_skus": ..., "n_promo_days": ...}`.

### `calendar_events`
`event, date_start, date_end, uplift_pre, uplift_in, uplift_post, pre_days, post_days`.
Thật: `quoc-khanh-2026 | 02/09 | 02/09 | 1,15 | 1,05 | 0,95 | 3 | 2`.

**RLS:** cả `demand_daily` và `forecasts` có policy `tenant_isolation`
(`project_id = current_setting('app.project_id')`); tầng 2 là `TenantPool` lấy tenant từ **key đã xác thực**,
không bao giờ từ body (`main.py:282`).

---

## 8. DATA LINEAGE

```
purchase.completed / stock.out / price.changed / promo.scheduled   (raw_events)
        │  rollup.py:65  (gộp theo ngày, bù ngày hết hàng)
        ▼
demand_daily.adjusted_units ─────────────────────────────┐
        │  _deflate_promo_units (chia hệ số promo cũ)     │  (đường lgbm: build_future_features)
        ▼                                                ▼
   units[] "sạch promo"                        đặc trưng lịch + promo + giá
        │  route/model đã chọn                            │
        ▼                                                ▼
   (p10,p50,p90) theo ngày  ◄───────────────────── predict_quantiles
        │  apply_calibration (width_factor từ backtest)
        │  _apply_promo_uplift (promo tương lai, KHÔNG áp cho lgbm)
        ▼
   forecasts (p10,p50,p90)                ← ghi ở đây (mục 16)
        │  apply_calendar (lễ/Tết, KHÔNG áp cho lgbm)
        │  aggregate_quantiles / quantiles_nbd → totals
        ▼
   response.daily[] · response.totals · response.calendar_effects
```

---

## 9. PREPROCESSING

1. **Bù ngày hết hàng** (rollup, `rollup.py:299`): `stockout=true` → `adjusted_units = max(units_sold, trung bình 7 ngày trước)`.
   Không bù thì model học "ngày đó bán ít" trong khi thật ra là *không còn hàng để bán*.
2. **Khử promo quá khứ** (`forecast_run.py:441` `_deflate_promo_units`): ngày có `promo_pct` được chia cho
   `(1 + k·discount)` để lấy mức nền. Nếu không, đỉnh khuyến mãi cũ thổi phồng nền, rồi promo tương lai lại
   nhân tiếp lên nền đã phồng — **đếm hai lần**.
3. **Cửa sổ dữ liệu**: lấy **toàn bộ** lịch sử `demand_daily` của SKU (`fetch_series`, `store/forecasts.py:12`),
   không cắt ngắn. `data_window` trong response chính là `min(day)..max(day)`.
4. Không có bước bỏ ngoại lai riêng; ngoại lai được xử lý gián tiếp qua phân vị và `width_factor`.

⚠ **Ngày cuối chuỗi luôn là hôm nay và luôn dở dang** — hệ quả nghiêm trọng, xem mục 21.

---

## 10. FUNCTION CALL GRAPH

```
main.py:574 forecast_query
 ├─ validate body                                         main.py:576-615
 ├─ store/forecasts.py:125  latest_daily(pool,…,horizon)   → đọc mẻ mới nhất, chỉ ngày > hôm nay
 │     └─ nếu đủ ngày → DỪNG ĐỌC DB, dùng luôn
 ├─ (thiếu) jobs/forecast_run.py:1642  forecast_on_demand
 │     ├─ store/forecasts.py:12   fetch_series            → demand_daily
 │     ├─ forecast_run.py:379     _load_uplift_k          → kv_state promo_uplift_k
 │     ├─ forecast_run.py:441     _deflate_promo_units
 │     ├─ forecast_run.py:407     _load_calendar_events
 │     ├─ forecast_run.py:221     _get_future_promo       → raw_events promo.scheduled
 │     ├─ forecast_run.py:200     _get_model_choice       → kv_state model_choice
 │     ├─ nhánh lgbm: forecast_run.py:859 _get_or_train_global → core/global_model.predict_quantiles
 │     ├─ nhánh thống kê: core/router.py:14 route_and_forecast
 │     │     ├─ core/classify.py:8        classify_series
 │     │     ├─ core/baseline.py:58/86/119 cold_start / seasonal_naive / ets_theta
 │     │     └─ core/croston.py:58        croston_daily_forecast
 │     ├─ forecast_run.py:945     _similar_item_transfer  (khi < 14 ngày lịch sử)
 │     ├─ core/backtest.py:224    apply_calibration       (width_factor)
 │     ├─ forecast_run.py:312     _apply_promo_uplift
 │     └─ store/forecasts.py:77   save_run                ⚠ GHI DB
 ├─ (404) main.py:1113 _cold_start_analog → smartsearch /internal/similar-products
 │     └─ core/analog.py:34 build_analog_profile
 ├─ core/calendar_effect.py:118 apply_calendar            (bỏ qua nếu model = lgbm_global)
 ├─ totals: core/croston.py:90 quantiles_nbd  HOẶC  libs/featurelib/quantiles.py:72 aggregate_quantiles
 ├─ đếm ngày censored (SELECT COUNT trên demand_daily)    main.py:714
 ├─ libs/featurelib/quantiles.py:135 extend_quantiles     (khi có `quantiles`)
 ├─ gộp kỳ weekly/monthly                                  main.py:746-770
 └─ libs/common/consistency.py:124 consistency_block
```

---

## 11. MODEL / ALGORITHM SELECTION

Hai tầng, **bằng chứng thắng luật cứng**:

**Tầng 1 — backtest đã chọn sẵn** (`kv_state: model_choice:<tenant>:<sku>`). Job backtest chấm các ứng viên
bằng **pinball loss q50**, chọn thấp nhất, nhưng nếu thua `seasonal_naive` thì quay về `seasonal_naive`
(`core/backtest.py:186 choose_model`). Kèm theo `width_factor` (`backtest.py:206`) — hệ số nong/nén dải,
tính từ độ phủ thực nghiệm so với đích 0,80.

**Tầng 2 — router theo hình dạng chuỗi** khi chưa có lựa chọn (`core/router.py:14`), phân loại
Syntetos–Boylan (`core/classify.py:8`):

| Điều kiện | Segment | Model |
|---|---|---|
| < 14 ngày, hoặc < 2 ngày có bán | `cold_start` | `cold_start` (mean + dải ±50 %) |
| ADI > 1,32 và CV² ≤ 0,49 | `intermittent_croston` | Croston |
| ADI > 1,32 và CV² > 0,49 | `intermittent_sba` | SBA |
| ADI ≤ 1,32, ≤ 56 ngày | `smooth` | `seasonal_naive` |
| ADI ≤ 1,32, > 56 ngày | `smooth` | AutoETS + AutoTheta (lỗi → `seasonal_naive`) |

*ADI = số ngày / số ngày có bán; CV² = phương sai/bình phương trung bình trên các ngày có bán.*

Phân bố thật (`demoshop`, 113 SKU có lựa chọn): `lgbm_global` 51 · `autoets_theta_ensemble` 38 ·
`croston_auto` 9 · `seasonal_naive` 7 · `imapa` 4 · `adida` 4.

**Fallback lgbm** (`forecast_run.py:67` — từ vựng đóng, đi thẳng ra response của promo-preview):
`training` · `insufficient_data` (< 50 SKU hoặc < 120 ngày) · `no_history` · `error` · `data_quality` ·
`similar_item_transfer`. ⚠ `forecast:query` **không** trả field `model_fallback` (promo-preview thì có) —
mục 21.

---

## 12. ALGORITHM THEO TỪNG CASE

| Case (mục 4) | Model chạy | Vì sao đi đường đó |
|---|---|---|
| C1, C2 | đọc bảng, không chạy model | mẻ batch hôm nay đã đủ 14/28 ngày |
| C3 | LightGBM global (9 booster phân vị) | backtest chấm lgbm thắng cho SKU đó |
| C4 | Croston + NBD | ADI > 1,32 (bán lai rai) |
| C7 | AutoETS+Theta | chuỗi đều, > 56 ngày lịch sử |
| C8 | `cold_start_analog` | 0 ngày lịch sử nhưng có hàng xóm trong catalog |
| C10 | LightGBM, train ngay trong request | h=56 > 28 dòng lưu + cache lạnh |

---

## 13. ALGORITHM INPUT

| Thuật toán | Ăn gì | Tham số |
|---|---|---|
| `classify_series` | `units[]` đã khử promo | ngưỡng ADI 1,32 · CV² 0,49 |
| `seasonal_naive` | `units[]` | chu kỳ 7; phân vị từ phần dư `y_t − y_{t−7}` theo thứ trong tuần |
| `ets_theta` | `units[]` (≥ 14) | `season_length=7`; dải lấy **native PI mức 80 %** của statsforecast |
| `croston_daily` | `units[]` | `alpha=0,1`; SBA nhân `(1 − α/2)` |
| `quantiles_nbd` | `units[]` (ước tán) + `day_means[]` (p50 từng ngày) | `n=2000`, `seed=42` |
| `lgbm_global` | `demand_daily` **toàn tenant** + lịch + promo + giá | cần ≥ 50 SKU **và** ≥ 120 ngày |
| `build_analog_profile` | chuỗi của tối đa 5 hàng xóm | mỗi hàng xóm ≥ 7 ngày, nong dải ±20 % |
| `aggregate_quantiles` | list `(p10,p50,p90)` | `n=2000`, `seed=42`, `rho=0` |

---

## 14. ALGORITHM OUTPUT + VÍ DỤ TÍNH TAY

### (a) Hiệu ứng lịch — kiểm được bằng phép chia (C2, đo thật)

| Ngày | Pha | Hệ số | p50 đo được | p50 ÷ hệ số |
|---|---|---|---|---|
| 01/09 | pre | 1,15 | 5,8040 | 5,0470 |
| 02/09 | in | 1,05 | 5,3006 | 5,0482 |
| 03/09 | post | 0,95 | 4,7969 | 5,0494 |
| 05/09 | — | 1,00 | 5,0518 | 5,0518 |

Bốn giá trị nền gần như bằng nhau ⇒ đúng là *một* hệ số nhân lên nền, không phải model đổi.

### (b) Cold-start analog — khớp từng chữ số (C8, đo thật)

Dữ liệu: `adf4-tainghe-a` tổng 335 đơn/61 ngày → mean **5,4918**; `adf4-tainghe-b` 275/61 → **4,5082**.
Mức chung `common_mean = (5,4918 + 4,5082)/2 = 5,0`; hệ số co giãn `scale_a = 0,9104`, `scale_b = 1,1091`.

Ngày dự báo đầu (13/08, thứ Năm) lấy mẫu từ thứ Năm tuần trước (06/08): a bán **5**, b bán **4**.

- a: 5 × 0,9104 = **4,552** · b: 4 × 1,1091 = **4,4365**
- `p50 = (4,552 + 4,4365)/2 = 4,4943` → API trả **4,494301** ✔
- `p10 = min × 0,8 = 3,5492` → API trả **3,549091** ✔
- `p90 = max × 1,2 = 5,4627` → API trả **5,462687** ✔

### (c) Tổng cả kỳ — **không bao giờ cộng phân vị**

`aggregate_quantiles` (`quantiles.py:72`) rút Monte Carlo 2000 kịch bản qua **hàm phân vị** của từng ngày rồi
lấy phân vị của *tổng*. Cộng thẳng p90 của 14 ngày = giả định 14 ngày cùng xấu một lúc.
Đo C1 (cùng **một** lần gọi): cộng thẳng `p10/p50/p90` = 17,33 / 70,93 / **122,96**; `totals` =
57,61 / 70,25 / **83,46**. Trung vị gần trùng (70,93 ≈ 70,25) nhưng hai đuôi co lại mạnh — đúng kỳ vọng:
14 ngày khó cùng xấu hoặc cùng tốt một lượt.

Riêng họ Croston dùng `quantiles_nbd` — mô phỏng ma trận `2000 × h` từ phân phối nhị thức âm rồi cộng theo
kịch bản (`totals_method = nbd_horizon_sim_2000`), vì với hàng bán lai rai thì p10/p90 ngày **không** phải
min/max cứng.

---

## 15. POST-PROCESSING

Thứ tự đúng như code, mỗi bước đúng **một** lần:

1. `apply_calibration(width_factor)` — nong/nén dải quanh p50 (`backtest.py:224`). Thật: `dt-cuongluc-ip15`
   `width_factor=1,2004` (dải đang hẹp so với thực tế), `th-binh-nuoc-1l` `0,8746` (đang rộng).
2. `_apply_promo_uplift` — nhân `(1 + k·discount)` cho ngày có promo tương lai. **Bỏ qua với lgbm** (promo đã là feature).
3. `apply_calendar` — nhân hệ số lễ theo pha. **Bỏ qua với lgbm** (đã có covariate `cal_factor`).
4. Tổng kỳ — `quantiles_nbd` (Croston) hoặc `aggregate_quantiles`.
5. `extend_quantiles` — suy p95/p99 bằng đuôi log-chuẩn từ (p50, p90), ép `≥ p90`.
6. Gộp `weekly`/`monthly` — mỗi kỳ lại chạy `aggregate_quantiles` riêng, **không** cộng phân vị ngày.

**Chống đếm hai lần** nằm ở bước 2 và 3: mỗi hiệu ứng (promo, lịch) chỉ được áp trên **một** đường —
đường thống kê nhân ở lúc đọc, đường lgbm học qua feature. Đây là vết sẹo có thật: trước PROD5H-FC hệ số Tết
bị nhân 1,8 × 1,3 = 2,34.

---

## 16. DATABASE WRITE / STATE

**Có ghi.** Đây là điều dễ bất ngờ nhất của endpoint này.

- Khi phải tính mới, `forecast_on_demand(persist=True)` gọi `save_run` (`store/forecasts.py:77`):
  `DELETE FROM forecasts WHERE (project, product, run_id)` rồi `INSERT` từng ngày.
- `run_id = "r_" + hôm nay` — **trùng run_id của mẻ batch trong ngày**.

Đo thật 2026-08-12: trước khi tôi gọi, `simworld1/sku-0041` có `r_2026-08-12` với **28 dòng** (mẻ batch
04:28). Sau một lần `query h=56`: cùng `run_id` còn **56 dòng**, `created_at` 15:01 — mẻ batch đã bị **thay**.
Cùng hiện tượng trên `demoshop/dt-cuongluc-ip15`. → nợ `W-QUERY-OVERWRITES-BATCH-RUN` (mục 21).

**State/cache khác:**
- `model_cache` (`core/model_cache.py`) giữ bundle LightGBM theo tenant, **TTL 3600 s**, tối đa 4 tenant, LRU,
  1 slot huấn luyện. Cache cả kết quả *âm* ("tenant này không đủ điều kiện") để khỏi quét lại `demand_daily`.
- Metric thật lúc viết bài: `global_model_cache_total{event="miss"}=3`, `stored_model=2`, `expired=1`;
  `model_fallback_total{reason="training"}=1`.

---

## 17. RESPONSE

| Field | Ý nghĩa | Loại |
|---|---|---|
| `product_id` | echo | — |
| `run_id` | `r_YYYY-MM-DD`, hoặc `analog_YYYY-MM-DD` khi cold-start | audit |
| `daily[]` | `{day, p10, p50, p90}` — đơn vị **đơn hàng/ngày**, bắt đầu từ **ngày mai** | SỐ SUY RA |
| `totals` | tổng cả kỳ, **phân vị của tổng** | SỐ SUY RA |
| `totals_method` | `triangular_mc_2000` \| `nbd_horizon_sim_2000` | audit |
| `model_used` | model thật sự đã trả lời | audit |
| `data_window` | `"min..max"` ngày lịch sử đã dùng; `null` khi analog | SỐ ĐO |
| `calibration` | `{empirical_coverage, width_factor}` từ backtest | SỐ ĐO |
| `censored_adjusted_days` | số ngày `stockout=true` trong **toàn bộ** lịch sử SKU (không giới hạn cửa sổ) | SỐ ĐO |
| `calendar_effects[]` | `{event, phase, days_affected}` — rỗng với lgbm | audit |
| `analog_of[]` | chỉ có ở cold-start: SKU đã mượn hình dạng | audit |
| `granularity` + `periods[]` | chỉ khi xin gộp kỳ | SỐ SUY RA |
| `quantile_method` | `lognormal_tail_extrapolation`, chỉ khi xin `quantiles` | audit |
| `consistency` | `{projection_watermark, data_as_of, is_caught_up, ledger_head}` — vắng khi ledger tắt | SỐ ĐO |
| `generated_at` | thời điểm trả lời (UTC) | — |

Header `X-Request-ID` có ở **mọi** phản hồi kể cả lỗi (đo: 400 vẫn có).

---

## 18. DOWNSTREAM

- **decision (BT02)** — `services/decision/app/store/forecast_client.py:79` `get_totals()` gọi chính API này,
  chỉ lấy `totals`, dùng cho tính nhập hàng/đặt giá. Đây là **cạnh service-to-service duy nhất** của hệ.
  Có circuit breaker + retry; **404 → coi như không có dữ liệu → hạ về MA28** (không phải lỗi).
- **Khách/SDK** — dựng biểu đồ dải P10–P90, chốt số đặt hàng theo P90.
- `POST /v1/forecast:aggregate` **không** gọi lại API này mà đọc thẳng `latest_daily` cho từng SKU rồi cộng
  bằng cùng một hàm — nên hai bề mặt không lệch nhau.

---

## 19. ERROR / FALLBACK

| Hỏng ở đâu | Cách xử | Khách thấy gì |
|---|---|---|
| Body sai | chặn sớm | 400 `INVALID_ARGUMENT` |
| Key sai tenant | middleware | 403 `PERMISSION_DENIED` |
| SKU không có lịch sử | thử analog trước | 404 `NOT_FOUND` (2 thông điệp khác nhau: có/không có hàng xóm) |
| smartsearch chết | `_cold_start_analog` bắt mọi lỗi → `None` | 404 thường (fail-close, không bịa số) |
| `calendar_events` lỗi/thiếu cột V007 | `try/except` → bỏ qua lịch | 200, `calendar_effects: []` |
| LightGBM train lỗi | cache `error` 300 s, rơi về router | 200, `model_used` = model thống kê |
| statsforecast thiếu/lỗi | `ets_theta` → `seasonal_naive` | 200 |
| NBD lỗi (p50 toàn 0…) | rơi về `aggregate_quantiles` | 200, `totals_method=triangular_mc_2000` |
| Ledger chết | `consistency_block` → `None` | 200, **không có** khoá `consistency` |

Nguyên tắc: **fail-open với thứ trang trí** (lịch, consistency, calibration), **fail-close với thứ cốt lõi**
(không có nhu cầu thì trả 404 chứ không bịa số).

---

## 20. TEST

Đang phủ (`tests/integration/`, chạy thật vào DB `:16024`):

| File::hàm | Phủ |
|---|---|
| `test_forecast_api.py::test_query_smooth` | 14 ngày, thứ tự p10≤p50≤p90, `seasonal_naive`, `censored_adjusted_days=0` |
| `test_forecast_api.py::test_query_cold_start` | 5 ngày lịch sử → `model_used=cold_start` |
| `test_forecast_api.py::test_404_no_history` | 404 `NOT_FOUND` |
| `test_forecast_api.py::test_run_batch` | batch 202 → poll job → query dùng lại mẻ |
| `test_gap_features.py`, `test_bt03_m14.py`, `test_forecast_engine_m15.py`, `test_replenish_scenario.py` | quantiles/granularity, seam promo, chuỗi replenish |

**Chưa phủ** (đã đo bằng tay trong bài này, chưa có test khoá):
- `cold_start_analog` với analog **có thật** — test 404 hiện chấp nhận cả 200 lẫn 404 (nợ `W-FC-SMALL-FIXES` mục 2 gọi tên là `W-COLDSTART-PROVE`).
- `calendar_effects` đúng pha pre/in/post trên đường `:query`.
- `totals_method = nbd_horizon_sim_2000` cho họ Croston.
- Ghi đè mẻ batch khi `h > 28` (mục 16) — không có test nào phát hiện.
- `granularity=monthly` cắt kỳ đầu/cuối không trọn tháng.

---

## 21. KNOWN ISSUE

### Nợ MỚI phát hiện khi viết bài (đã ghi vào DB tri thức)

**(1) `W-SEASONAL-NAIVE-PARTIAL-DAY` — mỗi 7 ngày dự báo có 1 ngày lấy mẫu từ HÔM NAY chưa đóng sổ.**
`seasonal_naive_forecast` (`baseline.py:103`) và `_per_day_pattern` của analog (`analog.py:31`) lấy
`units[n − 7 + (h mod 7)]`; phần tử `n−1` chính là **ngày hôm nay**, mà `demand_daily` luôn có dòng cho hôm nay
với số bán *dở dang*. Đo `demoshop` 2026-08-12: tổng bán 8 ngày gần nhất là 176/175/184/205/247/250/158/162
rồi **0** cho hôm nay (134 SKU). Hệ quả đo trên `ld-serum-vitc`: dự báo `19/08 = 0` và `26/08 = 0` trong khi
thứ Tư tuần trước bán 1 — **1/7 số ngày dự báo bị ép về 0**. Ảnh hưởng `seasonal_naive` (7 SKU demoshop),
`cold_start_analog`, và gián tiếp mọi model qua trung bình. Việc phải làm: loại ngày hiện tại khỏi mẫu
(hoặc chỉ dùng đến `CURRENT_DATE − 1`).

**(2) `W-QUERY-BLOCKS-ON-LGBM-TRAIN` — một truy vấn đọc có thể treo ~50 s.**
`forecast:query` gọi `forecast_on_demand` với `wait_for_global_model=True` (mặc định), nên khi cache lạnh nó
**huấn luyện LightGBM ngay trong request**. Đo `simworld1/sku-0041 h=56`: **49,18 s** lần đầu, **0,047 s**
lần sau; docstring `model_cache` ghi nhận 90–180 s trên tenant lớn hơn. `promo-preview` đã được vá
(`wait=False` + báo `model_fallback`), `:query` thì **chưa**. Điều kiện kích hoạt rất đời thường:
`horizon_days > 28` hoặc SKU chưa có mẻ nào.

**(3) `W-QUERY-OVERWRITES-BATCH-RUN` — truy vấn đọc ghi đè mẻ batch cùng ngày.**
`save_run` xoá theo `run_id` mà `run_id` chỉ là ngày, nên một lần `query h=56` biến mẻ batch 28 dòng thành
56 dòng do on-demand sinh, với `model_used`/`calibration` có thể khác. Người đọc bảng `forecasts` sau đó
tưởng đang xem kết quả batch. Còn làm nhiễu `fetch_residual_history` (chấm điểm bằng `created_at::date < horizon_day`).

**(4) `W-QUERY-NO-MODEL-FALLBACK-FIELD` — hạ cấp model im lặng ở `:query`.**
`forecast_on_demand` trả `model_fallback` giải thích vì sao không dùng được lgbm; `promo-preview` chuyển
tiếp field này, `:query` **vứt đi**. Khách thấy `model_used` đổi mà không biết lý do — đúng thứ bệnh
`W-PROMO-PREVIEW-CACHE` đã chữa cho bề mặt kia.

**(5) `W-OPENAPI-QUERY-BODY-EMPTY` — openapi không mô tả body.**
`openapi/forecast.json` cho `/v1/forecast:query` chỉ có `responses.200.schema = {}`, không có `requestBody`,
vì handler đọc `request.json()` thủ công thay vì model Pydantic. SDK sinh từ openapi sẽ không biết truyền gì.

### Nợ đã có sẵn trong DB, chạm tới API này

| W-ID | Ảnh hưởng ở đây |
|---|---|
| `W-FORECASTS-NO-RETENTION` | bảng `forecasts` không ai dọn — đo 2026-08-12: 325.563 dòng / 95 MB / 10 mẻ; mỗi lần `:query` tính mới lại thêm dòng |
| `W-AGG-RHO-MEASURE` | `aggregate_quantiles` mặc định `rho=0` (các ngày độc lập) ⇒ `totals` **hẹp hơn thực tế** |
| `W-UPLIFT-K-CLAMP-SATURATED` | 4 tenant có `k` chạm trần 4,0 mà không ghi lại là đã bị cắt; ảnh hưởng cả khử promo quá khứ lẫn nhân promo tương lai |
| `W-FC-SMALL-FIXES` (mục 2) | chính là test cold-start analog còn thiếu (mục 20) |
| `W-BT03-COLDSTART-ANALOG` | mở rộng analog theo ngành hàng |
| `W-TEST-QUEUE-SHARED-DB` | test forecast ghi thẳng vào DB demo — cùng gốc với nợ (3) |

---

### Phụ lục — dựng lại môi trường case study C8

Tenant **riêng** `apidocf4` (không đụng `demoshop`): upsert 3 SKU cùng ngành hàng vào smartsearch,
backfill 60 ngày `purchase.completed` cho 2 SKU đầu, chạy rollup, **chờ ~5 phút cho job embedding**,
rồi `forecast:query` SKU thứ 3. Nếu gọi sớm hơn sẽ nhận 404 chứ không phải analog (mục 6).
