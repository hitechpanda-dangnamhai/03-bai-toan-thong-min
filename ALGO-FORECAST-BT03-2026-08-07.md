# BT03 — FORECAST: TỪNG THUẬT TOÁN, ĐỌC TỪ CODE THẬT

> Ngày dựng: 2026-08-07 · Nguồn: **CHỈ CODE** trong `project/services/forecast/`, `project/libs/featurelib/`,
> `project/scripts/`, `project/eval/` + số đo chạy thật trên Postgres `localhost:16024/miniai_forecast` (2026-08-07).
> Mọi `file.py:dòng` dưới đây đã đối chiếu trực tiếp. Chỗ nào code không nói rõ → ghi **CHƯA CHẮC**, không đoán.
> Đường dẫn gốc: `/home/hai-soft/projects/icpp/mecom/project/`.

---

## 0. BẢN ĐỒ TỔNG

```
                       ┌──────────────────────────────────────────────────────────────┐
 Khách gửi event  ───► │ raw_events (purchase.completed · order.returned · stock.out   │
 POST /v1/events:ingest│            · price.changed · promo.scheduled)                 │
                       └───────────────┬──────────────────────────────────────────────┘
                                       │  jobs/rollup.py :: run_rollup_once  (mỗi 3600s)
                                       ▼
                       ┌──────────────────────────────────────────────────────────────┐
                       │ demand_daily(project_id, product_id, day, units_sold,        │
                       │   stockout, price, promo_pct, adjusted_units)                │
                       │   • adjusted_units = bù ngày hết hàng (censored demand)       │
                       └───────────────┬──────────────────────────────────────────────┘
                                       │
        ┌──────────────────────────────┼───────────────────────────────────────────────┐
        │                              │                                               │
        ▼ (mỗi 604800s)                ▼ (mỗi 86400s)                                  ▼
 jobs/backtest_run.py            jobs/forecast_run.py                        core/scenario/artifact.py
 ─ rolling-origin O=(28,21,14)   ─ học k promo (promo_uplift.py)             ─ fit marginal / factor
 ─ core/backtest.backtest_series ─ DEFLATE promo (_deflate_promo_units)      ─ TailCalibrator.apply
 ─ + lgbm_global OOF (nếu đủ bar) ─ chọn nhánh model theo kv_state           ─ CAS publish artifact
 ─ choose_model (pinball q50)     │                                          ─ manifest.json ghi CUỐI
 ─ calibration_factor → w         │
 ─ ghi kv_state model_choice ─────┘
 ─ ghi TailCalibrator (kv_state)
                                       │
                                       ▼   phân loại chuỗi (core/classify.py, ADI/CV²)
        ┌──────────────────────────────────────────────────────────────────────────────┐
        │ THANG MODEL (thứ tự: kv_state model_choice THẮNG; không có → core/router.py)  │
        │  cold_start · seasonal_naive · autoets_theta_ensemble · croston/sba           │
        │  · adida · imapa · lgbm_global · similar_item_transfer · cold_start_analog    │
        └───────────────┬──────────────────────────────────────────────────────────────┘
                        │  apply_calibration(width_factor)  →  _apply_promo_uplift (trừ lgbm)
                        ▼
                 forecasts(project_id, product_id, run_id, horizon_day, p10, p50, p90,
                           model_used, data_window, calibration)
                        │
        ┌───────────────┴────────────────────────────────────────────────────────────────┐
        │ ĐỌC (main.py):                                                                 │
        │  POST /v1/forecast:query      → + apply_calendar (trừ lgbm) + totals            │
        │                                  (NBD horizon-sim cho croston, triangular MC…)  │
        │  POST /v1/forecast:aggregate  → totals nhóm + hierarchical reconcile (flag)     │
        │  POST /v1/forecast:promo-preview → what-if (không ghi DB)                       │
        │  POST /v1/forecast:insights   → accuracy / movers / seasonality / sell-through  │
        │  POST /v1/scenarios:{build,lead-time-demand,aggregate,probability}              │
        └────────────────────────────────────────────────────────────────────────────────┘
```

### Bảng liệt kê nhanh mọi thuật toán

| # | Thuật toán | File chính | Vai trò |
|---|---|---|---|
| 1 | Rollup event → demand_daily | `jobs/rollup.py:44` | Chuẩn bị dữ liệu |
| 2 | Censored-demand adjustment (stockout) | `jobs/rollup.py:211-238` | Chuẩn bị dữ liệu |
| 3 | Promo deflation (tách nền vs uplift) | `jobs/forecast_run.py:435` | Chuẩn bị dữ liệu |
| 4 | Ước lượng hệ số uplift promo `k` | `core/promo_uplift.py:33` | Chuẩn bị dữ liệu |
| 5 | Phân loại Syntetos–Boylan (ADI/CV²) | `core/classify.py:8` | Phân loại |
| 6 | `cold_start` | `core/baseline.py:10` | Dự báo |
| 7 | `seasonal_naive` (+ residual quantile theo weekday) | `core/baseline.py:38` | Dự báo |
| 8 | `autoets_theta_ensemble` (AutoETS + AutoTheta + bootstrap) | `core/baseline.py:71` | Dự báo |
| 9 | Croston / SBA + NBD quantile | `core/croston.py:20,58,90` | Dự báo |
| 10 | ADIDA | `core/intermittent_sf.py:95` | Dự báo |
| 11 | IMAPA | `core/intermittent_sf.py:103` | Dự báo |
| 12 | `similar_item_transfer` | `jobs/forecast_run.py:732` | Dự báo (cold-start có ít data) |
| 13 | `cold_start_analog` | `core/analog.py:34` + `main.py:1113` | Dự báo (0 data) |
| 14 | LightGBM global quantile (9 booster) | `core/global_model.py:101` | Dự báo |
| 15 | Feature engineering | `libs/featurelib/forecast_features.py:37,199` | Dự báo |
| 16 | Router theo phân loại | `core/router.py:14` | Chọn model |
| 17 | Rolling-origin backtest + pinball/MASE/coverage | `core/backtest.py:77` | Chọn model |
| 18 | `choose_model` | `core/backtest.py:186` | Chọn model |
| 19 | `calibration_factor` / `apply_calibration` (width_factor) | `core/backtest.py:206,224` | Hiệu chỉnh KTC |
| 20 | Calendar effect (Tết/lễ VN, 3 pha) | `core/calendar_effect.py:97,118` | Lịch mùa vụ |
| 21 | Hierarchical reconciliation MinT/WLS | `core/scenario/reconcile.py` + `core/hier_reconcile.py` | Nhất quán cấp bậc |
| 22 | Scenario fabric: RNG Philox counter-based | `core/scenario/rng.py:59` | Monte Carlo |
| 23 | Scenario: marginal (3 loại) + GP tail | `core/scenario/marginal.py` | Monte Carlo |
| 24 | Scenario: TailCalibrator (conformal) | `core/scenario/calibrate.py:135` | Monte Carlo |
| 25 | Scenario: Gaussian factor copula | `core/scenario/generator.py:99` | Monte Carlo |
| 26 | Scenario: trích common factor từ OOF residual | `core/scenario/factors.py:34` | Monte Carlo |
| 27 | Tổng hợp quantile bằng triangular MC | `libs/featurelib/quantiles.py:43` | Tổng hợp |
| 28 | NBD horizon-total simulation | `core/croston.py:90,150` | Tổng hợp |
| 29 | Ngoại suy đuôi p95/p99 (lognormal) | `libs/featurelib/quantiles.py:80` | Tổng hợp |
| 30 | Insights: weekday/monthly profile, prob_ge, accuracy | `core/insights.py` | Đọc |
| 31 | PSI drift | `libs/featurelib/drift.py:17` | (**KHÔNG nối vào BT03** — xem §13) |
| 32 | Data-quality gate | `libs/featurelib/data_quality.py:11` | (**KHÔNG nối vào BT03** — xem §13) |
| 33 | Holiday factor hardcode VN | `libs/featurelib/holidays_vn.py:69` | (**ĐÃ NGHỈ HƯU** — xem §13) |

---

## 1. TẦNG CHUẨN BỊ DỮ LIỆU

### 1.1. Rollup: event → `demand_daily`

**Vị trí:** `services/forecast/app/jobs/rollup.py:44` — `run_rollup_once(pool, window_days=120)`.
Vòng lặp nền: `start_rollup_loop` (`rollup.py:265`), chu kỳ `ROLLUP_INTERVAL` mặc định **3600 giây** (`rollup.py:272`).

**Bài toán nghiệp vụ:** shop bắn event bán hàng lẻ tẻ theo giây; model cần một chuỗi **liên tục theo NGÀY, không thủng lỗ**
(ngày không bán = 0, không phải "thiếu dòng"), có giá hiệu lực, có cờ hết hàng, có % khuyến mãi.

**Đầu vào:** `raw_events(project_id, event_id, event_type, event_time, payload)` với `event_time >= now - window_days`
(`rollup.py:57-66`).

| event_type | Xử lý | Dòng |
|---|---|---|
| `purchase.completed` | `units[key] += qty`; `price_sum += qty*unit_price`; `price_qty += qty` | 88-99 |
| `order.returned` | `units[key] -= qty` (trả hàng trừ vào cầu NGÀY TRẢ, không phải ngày mua) | 100-109 |
| `stock.out` | `stockout[key] = True` | 110-114 |
| `price.changed` | thêm `(day, new_price)` vào `price_changes` | 115-119 |
| `promo.scheduled` | với MỌI ngày trong `[start, end]`: `promo[key] = max(promo[key], discount_pct)` | 120-130 |

**Điền ngày trống + giá:** với mỗi `(project, product)` lặp từ `min_day` tới `today` (`rollup.py:165`):
- `u = max(0.0, units.get(key, 0.0))` (`rollup.py:172`) — **cầu âm do trả hàng bị kẹp về 0**.
- giá hiệu lực: nếu ngày đó có mua → `round(price_sum/price_qty)` (bình quân gia quyền); nếu không → **carry-forward**
  giá thay đổi gần nhất (`rollup.py:178-181`), có thể `NULL` khi chưa có `price.changed` nào.

**Đầu ra:** UPSERT `demand_daily(project_id, product_id, day, units_sold, stockout, price, promo_pct, adjusted_units)`
`ON CONFLICT (project_id, product_id, day)` (`rollup.py:241-256`) → **idempotent**.

**Cạm bẫy trong code:** `window_days` mặc định 120 giữ nguyên hành vi v1; đường backfill lịch sử dài phải truyền
window lớn hơn, nếu không rollup CẮT MẤT lịch sử đã nạp (comment `rollup.py:49-52`, ca SIM-WORLD SW-1).

### 1.2. Censored demand adjustment (bù ngày hết hàng)

**Vị trí:** `jobs/rollup.py:211-238` (pass 2 trong `run_rollup_once`).

**Bài toán:** ngày hết hàng bán được 0 cái **không phải vì không ai muốn mua**. Nếu để 0 vào chuỗi, model học nhầm là cầu giảm
→ đặt hàng thiếu → càng hết hàng (vòng xoáy).

**Công thức:**

```
                ⎧ max( units_sold[t],  mean( adj[t-7 .. t-1] ) )   nếu stockout[t] = true
adj[t]  =       ⎨
                ⎩ units_sold[t]                                     nếu không
```

`mean(...)` là trung bình **7 giá trị `adj` GẦN NHẤT ĐÃ TÍNH** (`adjusted_prev[-7:]`, `rollup.py:220`), tức là trung bình
**trượt của chính adjusted** (đệ quy), không phải của `units_sold`. Nếu chưa có ngày trước nào → `trailing = 0.0`
(`rollup.py:222`).

**Đầu ra:** cột `demand_daily.adjusted_units` — **đây là cột MỌI model dùng**, không phải `units_sold`
(xem `store/forecasts.py:26`).

### 1.3. Promo deflation — "tách nền khỏi uplift" (promo seam W-BT23)

**Vị trí:** `jobs/forecast_run.py:435` — `_deflate_promo_units(series, k)`.

**Bài toán:** các model thống kê (Croston, seasonal-naive, ETS) học MỨC NỀN từ lịch sử. Nếu quá khứ có 20 ngày sale,
mức nền bị thổi lên; rồi ngày sale tương lai lại được nhân thêm hệ số uplift → **đếm hai lần**.

**Công thức:**

```
units_deflated[t] = adjusted_units[t] / ( 1 + k · frac[t] )      với frac[t] = promo_pct[t] / 100, kẹp vào [0,1]
```

chỉ áp dụng khi `frac > 0` và `k > 0` (`forecast_run.py:449-450`). Ngày thường giữ nguyên.

**Đối xứng:** đường ngược lại là `_apply_promo_uplift` (`forecast_run.py:306`):
`p10,p50,p90 ×= (1 + k·frac)` cho những ngày tương lai CÓ promo, rồi `sort_quantiles` lại (`forecast_run.py:327`).

**Bất biến quan trọng (`forecast_run.py:1304`, `1612`):** `_apply_promo_uplift` **KHÔNG chạy** khi
`model_used == "lgbm_global"` — LightGBM đã mang promo qua feature `promo_pct` + `on_promo`; nhân thêm là đếm hai lần.

### 1.4. Học hệ số uplift `k` từ lịch sử của chính shop

**Vị trí:** `core/promo_uplift.py:33` — `estimate_uplift_k(rows)`; wiring: `forecast_run.py:335` `_learn_and_store_uplift_k`.

**Bài toán:** "giảm giá 20% thì bán tăng bao nhiêu?" — mỗi ngành hàng một khác; hằng số 1.5 cứng là nói bừa.

**Thuật toán (đúng như code):**
1. Bỏ mọi dòng `stockout = True` (`promo_uplift.py:75`).
2. Nhóm theo `product_id`. Ngày promo = `promo_pct > 0`; ngày nền = `promo_pct == 0` (`:90-91`).
3. Loại SKU nếu `#promo < MIN_PROMO_DAYS_PER_SKU` hoặc `#nền < MIN_NONPROMO_DAYS_PER_SKU` (`:93-96`).
4. Với SKU còn lại:

```
base_mean   = mean( adjusted_units | ngày nền )
promo_mean  = mean( adjusted_units | ngày promo )
mean_disc   = mean( promo_pct/100 | ngày promo )

implied_k   = ( promo_mean / base_mean − 1 ) / mean_disc          (promo_uplift.py:112)
```
   Loại SKU nếu `base_mean <= 0`, `mean_disc <= 0`, hoặc `implied_k` không hữu hạn.
5. Nếu `#SKU sống < MIN_SKUS` → trả `None` (caller dùng hằng số).
6. `k = clamp( median(implied_k), K_MIN, K_MAX )` (`:125-126`) — **median**, không phải mean (chống outlier).

**Hằng số THẬT:**

| Hằng | Giá trị | Dòng |
|---|---|---|
| `MIN_PROMO_DAYS_PER_SKU` | 3 | `promo_uplift.py:20` |
| `MIN_NONPROMO_DAYS_PER_SKU` | 7 | `:23` |
| `MIN_SKUS` | 3 | `:26` |
| `K_MIN` / `K_MAX` | 0.0 / 4.0 | `:29-30` |
| `PROMO_UPLIFT_K` (fallback) | **1.5** | `forecast_run.py:45` |

**Lưu trữ:** `kv_state` key `promo_uplift_k:<project_id>`, giá trị
`{"k", "n_skus", "n_promo_days", "computed_at"}` (`forecast_run.py:355-369`). Đọc lại: `_load_uplift_k`
(`forecast_run.py:373`), lỗi/thiếu → 1.5.

**Số đo THẬT (2026-08-07, `kv_state`):**

| project | k học được | n_skus | n_promo_days |
|---|---|---|---|
| demoshop | 1.323 | 48 | 423 |
| simworld1 | 2.096 | 59 | 602 |
| simworld2 | 2.907 | 19 | 231 |
| simworld3 | 3.258 | 37 | 408 |
| simworld4 | 1.102 | 79 | 797 |
| bulktest | 3.491 | 50 | 774 |
| seedtest | **4.000 (chạm trần K_MAX)** | 14 | 153 |
| stafffull | 3.077 | 15 | 217 |

→ `seedtest` chạm trần 4.0: dấu hiệu dữ liệu seed có uplift phi thực tế **hoặc** trần quá thấp. Code không log cảnh báo
khi chạm trần — **nợ quan sát**.

### 1.5. Promo tương lai (đầu vào cho uplift)

**Vị trí:** `forecast_run.py:215` — `_get_future_promo(pool, project, product, horizon_days)`.

Đọc `raw_events` `event_type='promo.scheduled'` với `received_at ∈ [today−90d, today+horizon]` (`:235-247`), lọc payload có
`product_id` trong `product_ids` và `discount_pct > 0`, tính giao của `[start,end]` với `(today, today+horizon]`, rồi

```
promo_by_offset[offset] = max( promo_by_offset[offset], min(max(discount_pct/100, 0), 1) )
```

(`:293-301`) — nhiều promo chồng nhau lấy **giảm giá SÂU NHẤT**, offset 1-based tính từ hôm nay.

---

## 2. PHÂN LOẠI CHUỖI — SYNTETOS–BOYLAN (ADI / CV²)

**Vị trí:** `core/classify.py:8` — `classify_series(units) -> {adi, cv2, segment}`.

**Bài toán:** "SKU bán lai rai, nhiều ngày = 0" và "SKU chạy đều mỗi ngày" cần hai họ model khác hẳn nhau.
Cưỡng ép ETS lên chuỗi 85% số 0 thì ra dự báo mượt vô nghĩa.

**Công thức (đúng bản cài đặt):**

```
n         = len(units)                                  # số NGÀY, kể cả ngày 0
positive  = [u for u in units if u > 0]

ADI       = n / |positive|                              # classify.py:27  (khoảng cách trung bình giữa 2 lần bán)

mean_pos  = mean(positive)
var_pos   = (1/|positive|) · Σ (u − mean_pos)²          # phương sai TỔNG THỂ (chia N, không phải N−1) — classify.py:32
CV²       = var_pos / mean_pos²                          # classify.py:33
```

⚠ **CV² tính CHỈ trên ngày CÓ bán** (kích thước đơn hàng), không trên cả chuỗi — đúng chuẩn Syntetos–Boylan.

**Luật phân đoạn (`classify.py:35-41`):**

```
n < 14                    → "cold_start"                       (classify.py:20-21)
|positive| < 2            → "cold_start"                       (:24-25)
mean_pos <= 0             → "cold_start"                       (:29-30)
ADI > 1.32  và  CV² ≤ 0.49 → "intermittent_croston"
ADI > 1.32  và  CV² > 0.49 → "intermittent_sba"
ADI ≤ 1.32                 → "smooth"
```

**Hằng số THẬT:** `1.32` (ngưỡng ADI, `classify.py:35`) và `0.49` (ngưỡng CV², `classify.py:36`) — hardcode, **không có env override**.
Ngưỡng lịch sử tối thiểu `14` ngày (`classify.py:20`).

**Bản rút gọn dùng ở 2 chỗ khác:** `_segment_for_series` (`jobs/backtest_run.py:79`) và `_demand_class`
(`core/scenario/artifact.py:184`) chỉ lấy nhị phân `"intermittent" if adi > 1.32 else "smooth"` — **cố ý cùng một luật**
(comment `artifact.py:186-187`) để nhãn segment của calibrator khớp nhãn của backtest.

---

## 3. TỪNG THUẬT TOÁN DỰ BÁO

### 3.1. `cold_start`

**Vị trí:** `core/baseline.py:10` — `cold_start_forecast(units, horizon_days)`.

**Bài toán:** SKU mới lên sàn, chưa đủ 14 ngày lịch sử — không có gì để học mùa vụ, chỉ có mức trung bình.

**Công thức:**

```
p50 = mean(units)              (nếu units rỗng → 0)
p10 = 0.5 · p50
p90 = 1.5 · p50
Nếu p50 == 0  →  (p10, p50, p90) = (0, 0, 1)          # baseline.py:20-21
```

Trả về **cùng một bộ ba cho MỌI ngày** trong horizon (`baseline.py:22`).

**Cạm bẫy trong code:** trường hợp `p50 == 0` trả `p90 = 1` chứ không phải 0 — cố ý để khoảng tin cậy không suy biến
(không thể nói "chắc chắn 0"), nhưng làm `p90` mang đơn vị "1 cái" tùy tiện. **CHƯA CHẮC:** code không giải thích tại sao
chọn đúng 1.0.

**Khi được chọn:** `router.py:23-25` khi `segment == "cold_start"`; và là fallback của `seasonal_naive` khi `n < 7`
(`baseline.py:45-47`) và của `ets_theta` khi `n < 14` (`baseline.py:78-79`).

### 3.2. `seasonal_naive` (naive mùa vụ tuần + quantile phần dư theo thứ)

**Vị trí:** `core/baseline.py:38` — `seasonal_naive_forecast(units, horizon_days)`; phụ trợ `_residuals_by_weekday`
(`baseline.py:26`).

**Bài toán:** đa số shop bán theo NHỊP TUẦN (thứ 7/CN cao, thứ 3 thấp). "Tuần sau giống tuần này" là baseline cực rẻ
và rất khó đánh bại — nó chính là **thước đo** để chấm mọi model khác (MASE).

**Công thức:**

```
n = len(units);  nếu n < 7 → cold_start_forecast

# phần dư tuần-trên-tuần, gom theo VỊ TRÍ mod 7
res[i mod 7] .append( y[i] − y[i−7] )        cho i = 7 .. n−1              (baseline.py:32-34)

# cho ngày dự báo thứ h (h = 0..H−1):
idx   = n + h
wd    = idx mod 7
src   = n − 7 + (h mod 7)
p50_h = y[src]                                                            (baseline.py:55-57)
p10_h = max( 0,          p50_h + Percentile( res[wd], 10 ) )              (baseline.py:60,65)
p90_h = max( p50_h,      p50_h + Percentile( res[wd], 90 ) )              (baseline.py:61,66)
# nếu res[wd] rỗng:  p10 = 0.5·p50 ,  p90 = 1.5·p50                       (baseline.py:63-64)
```

⚠ **`wd` là "chỉ số mod 7", KHÔNG phải thứ trong tuần thật.** Nó tự nhất quán vì `demand_daily` là dải ngày LIÊN TỤC
không thủng (bảo đảm bởi rollup §1.1). Nếu ai đó truyền chuỗi có lỗ hổng ngày vào đây, mùa vụ tuần sẽ lệch pha âm thầm.

**Cạm bẫy code (bất nhất so với ADIDA/IMAPA):** `p10` chỉ bị kẹp `>= 0`, **KHÔNG** kẹp `<= p50` (`baseline.py:65`).
Chuỗi có xu hướng tăng mạnh → phân vị 10 của phần dư dương → `p10 > p50`. Sau đó `router.py:52` gọi `sort_quantiles`
sắp xếp lại 3 số → **giá trị "p50" trả ra thực chất là p10 cũ**, tức là điểm dự báo bị đổi âm thầm.
`core/intermittent_sf.py:75` có kẹp `p10 = max(0, min(p10, p50))`; `baseline.py` (cả `seasonal_naive` và `ets_theta`) thì không.
→ **Bất nhất thật, đáng vá.**

**Khi được chọn:** `router.py:38-40` khi `segment == "smooth"` và `len(units) <= 56`; là ứng viên baseline **luôn có mặt**
trong mọi rổ backtest (`backtest.py:72,74,93-95`); và là **model an toàn cuối cùng** của `choose_model`
(`backtest.py:187-203`).

### 3.3. `autoets_theta_ensemble` (AutoETS + AutoTheta + residual bootstrap)

**Vị trí:** `core/baseline.py:71` — `ets_theta_forecast(units, horizon_days)`.

**Bài toán:** SKU chạy đều nhưng có xu hướng + mùa vụ tuần; naive không bắt được trend.

**Thuật toán:**
1. `n < 14` → `cold_start_forecast` (`:78-79`).
2. **Lazy import** `statsforecast` (`:81-86`); import lỗi → rơi về `seasonal_naive_forecast`.
   (Chủ ý: service vẫn chạy khi không cài `requirements-ml.txt`.)
3. Dựng DataFrame `unique_id="sku"`, `ds = date_range(end=hôm nay, periods=n, freq="D")`, `y = units` (`:91-97`).
4. `StatsForecast(models=[AutoETS(season_length=7), AutoTheta(season_length=7)], freq="D", n_jobs=1)` (`:98-102`);
   `sf.forecast(df, h=horizon_days, level=[10,90])` (`:104`); exception → `seasonal_naive` (`:105-106`).
5. **Ensemble điểm:** `p50 = (AutoETS + AutoTheta) / 2` (`:111`) — trung bình cộng đơn giản, **không trọng số**,
   và **KHÔNG dùng** `level=[10,90]` mà statsforecast đã trả về.
6. **Khoảng tin cậy bằng bootstrap phần dư tuần:**

```
residuals = [ y[i] − y[i−7]  for i in 7..n−1 ]     (rỗng → [0.0])         (baseline.py:116-119)
rng       = numpy.default_rng(42)                                          (baseline.py:114)
với mỗi ngày h:
    boot  = rng.choice(residuals, size=500, replace=True)                  (baseline.py:124)
    p10_h = max( 0,        p50_h + Percentile(boot, 10) )
    p90_h = max( p50_h,    p50_h + Percentile(boot, 90) )
```

**Hằng số THẬT:** `season_length = 7` (`:99`), `n_jobs = 1` (`:101`), seed bootstrap `42` (`:114`),
số draw `500` (`:124`), ngưỡng lịch sử `14` (`:78`). Không env override.

⚠ **Điểm tinh tế:** `rng` được tạo MỘT LẦN ngoài vòng lặp ngày (`:114`) rồi `rng.choice` gọi lại mỗi ngày →
mỗi ngày lấy một block khác nhau của cùng dòng ngẫu nhiên. Kết quả **vẫn tất định** cho một `(units, horizon)` cố định,
nhưng **đổi horizon thì khoảng tin cậy của những ngày đầu KHÔNG đổi** (vì draw theo thứ tự) — đúng, không phải bug.

⚠ `level=[10,90]` được truyền vào `sf.forecast` nhưng cột `-lo-90/-hi-90` **không hề được đọc** (`:108` chỉ là comment).
Đây là **tham số chết** — tốn tính toán vô ích.

**Khi được chọn:** `router.py:42-46` khi `segment == "smooth"` và `len(units) > 56`; ứng viên backtest cho lớp smooth
(`backtest.py:74`); và khi `kv_state.model_choice = "autoets_theta_ensemble"` (`forecast_run.py:1239-1241`, `:1554-1556`).

### 3.4. Croston / SBA + mô phỏng NBD

**Vị trí:** `core/croston.py` — `_ses:10`, `croston_forecast:20`, `croston_daily_forecast:58`,
`quantiles_nbd:90`, `_quantiles_nbd_day_means:150`.

**Bài toán:** SKU bán lai rai — 30 ngày mới bán 4 lần, mỗi lần 2-5 cái. Trung bình cộng cho ra "0.4 cái/ngày" vô dụng;
cần tách **"bao lâu bán 1 lần"** khỏi **"mỗi lần bán bao nhiêu"**.

**Công thức — làm mượt hàm mũ đơn (SES) trên hai chuỗi riêng:**

```
SES(v, α):  level ← v[0];   level ← α·v[i] + (1−α)·level  cho i = 1..            (croston.py:10-17)

positive  = [ u | u > 0 ]                                        # kích thước đơn
intervals = [ i_k − i_{k−1} ]  giữa các NGÀY có bán               # khoảng cách
             nếu chỉ có 1 ngày bán → intervals = [ len(units) ]   (croston.py:44-45)

size_mean     = SES(positive,  α)
interval_mean = SES(intervals, α)      (kẹp ≥ 1 nếu ≤ 0)          (croston.py:49-50)

mean_per_day  = size_mean / interval_mean                          (croston.py:52)

SBA:  mean_per_day ×= ( 1 − α/2 )                                  (croston.py:54)   ← hiệu chỉnh chệch Syntetos–Boylan
```

**Hằng số THẬT:** `alpha = 0.1` (`croston.py:23` và `:63`) — **hardcode, không env override**, cùng dùng cho cả SES kích thước
lẫn SES khoảng cách. `horizon = 7` mặc định của `croston_daily_forecast` (`:61`).

**Khoảng tin cậy — NBD/Poisson MỘT NGÀY:** `croston_daily_forecast` gọi `quantiles_nbd(mean_per_day, horizon_days=1, units)`
(`croston.py:82`) rồi **lặp cùng bộ ba cho mọi ngày** (`:87`), với điểm dự báo bị kẹp vào khoảng:

```
p50_d = min( max( mean_per_day, p10_d ), p90_d )                  (croston.py:86)
```

**`quantiles_nbd` (croston.py:90) — mô phỏng TỔNG cầu qua `horizon_days` ngày:**

```
mean_u = mean(units);   var_u = (1/N)·Σ(u − mean_u)²              # phương sai tổng thể (croston.py:123-124)

nếu var_u > mean_u > 0:                                            # over-dispersed → Âm nhị thức
    p     = mean_u / var_u
    n_nb  = mean_u² / ( var_u − mean_u )
    n_nb_day = n_nb · ( mean_day / mean_u )                        # đổi thang về mean của model (croston.py:133)
    draws = NegBinomial( n_nb_day, p )  ma trận (n × horizon)      (croston.py:134)
ngược lại:
    draws = Poisson( mean_day )         ma trận (n × horizon)      (croston.py:142)

totals = draws.sum(axis=1)          # cộng THEO KỊCH BẢN, rồi mới lấy phân vị
return ( P10(totals), P50(totals), P90(totals) )
```

Kiểm chứng tham số hoá: numpy `negative_binomial(n,p)` có kỳ vọng `n(1−p)/p`;
thay `n = mean²/(var−mean)`, `p = mean/var` → kỳ vọng `= mean`. ✔ Đúng.

**Hằng số THẬT:** `n = 500` mẫu, `seed = 42` (`croston.py:94-95`).
Nhưng khi gọi từ API `forecast:query` thì dùng `n=2000, seed=42` (`main.py:701-703`).

**Nhánh `day_means` (`_quantiles_nbd_day_means`, croston.py:150):** khi các ngày **không hoán vị được**
(đã nhân hệ số lịch/promo vào đường p50), mỗi cột `j` của ma trận rút với mean riêng `col_means[j]`,
`n_nb_days = n_nb · (col_means / mean_u)` (`:182`), độ phân tán `p` vẫn fit từ lịch sử.

⚠ **Bài học đã trả giá — đọc comment `croston.py:74-81` và `:157-162`:**
1. Code CŨ lấy quantile TỔNG-horizon rồi chia đều cho số ngày → khoảng mỗi ngày **hẹp hơn ~√h** →
   **coverage hold-out sập** (đo tại SW-3 MỤC 2).
   ⚠ **CHƯA CHẮC — hai con số vênh nhau:** comment `croston.py:79-80` ghi *"measured 0.34 vs 0.80 nominal"*,
   nhưng DB tri thức (`facts sw3.coverage-croston-perday-fix`) ghi *"coverage P10-P90 hold-out intermittent
   0.693 << 0.80 ... KQ: intermittent 0.693→0.803, overall 0.727→0.808"*. Không tài liệu nào giải thích 0.34 đến từ đâu
   (có thể là một lát cắt hẹp hơn, hoặc con số của một lần đo khác). **Con số CÓ NGUỒN KIỂM CHỨNG là 0.693 → 0.803.**
   Bản hiện tại lấy quantile MỘT NGÀY cho dòng ngày, và tổng horizon được mô phỏng riêng bằng ma trận `n×h`.
2. Nhánh `day_means=None` được giữ **BIT-IDENTICAL** với bản lịch sử (`croston.py:158-162`) vì con số coverage 0.80 của SW-3
   được đo trên đúng dòng RNG đó; đổi dòng RNG là làm dịch khoảng tin cậy âm thầm.
3. `W-ROADMAP-NBD-HORIZON` (comment `croston.py:102-104`): nhân quantile-1-ngày với `h` để ra tổng horizon là SAI —
   bỏ qua hiệu ứng thu hẹp CLT của tổng, cho dải rộng gấp ~√h.

**Cạm bẫy lịch sử #2 (`croston.py:65-72`):** `backtest.py` và `forecast_run.py` từng gọi
`croston_forecast(units, sba=..., horizon=...)` trong khi `croston_forecast` **không có tham số `horizon`** →
`TypeError` bị `try/except` của backtest nuốt im → **lớp intermittent mất sạch ứng viên `croston_auto`** mà không ai biết.
Đây là lý do hàm `croston_daily_forecast` ra đời (F-ROUTER-IMAPA-1, đo 2026-08-03).

Bệnh này nặng hơn vẻ ngoài: theo skill `SK-FC-SWALLOWED-CANDIDATE` trong DB tri thức,
**nhánh intermittent chỉ còn `seasonal_naive` suốt từ M5 đến M15** — nhiều milestone "đạt gate" trong khi
model chuyên trị hàng bán lai rai chưa từng chạy. Luật rút ra (nguyên văn):
*"sau khi thêm candidate PHẢI đo `metrics.keys() == candidates` trên series thật — càng nuốt exception
càng dễ candidate chết âm thầm."* Hôm nay `backtest.py:131-133` **vẫn còn** `try/except ... continue` đó
(`:134` `except Exception: continue`) và **vẫn không có phép đo nào so `metrics.keys()` với rổ ứng viên**.

**Khi được chọn:** `router.py:26-36` (segment `intermittent_croston` → `croston`, `intermittent_sba` → `sba`);
tên model trong backtest là `"croston_auto"` và **tự chọn SBA theo CV²** ngay tại chỗ:
`sba = cv2 > 0.49` (`backtest.py:54-57`, lặp lại y hệt ở `forecast_run.py:1245-1250` và `:1560-1565`).

### 3.5. ADIDA

**Vị trí:** `core/intermittent_sf.py:95` — `adida_forecast(units, horizon_days)`; lõi chung `_forecast:81`,
điểm dự báo `_sf_point_forecast:26`.

**Bài toán:** vẫn là cầu lai rai, nhưng ADIDA (**A**ggregate–**Di**saggregate **I**ntermittent **D**emand **A**pproach)
gộp ngày thành khối để làm chuỗi bớt thưa, dự báo trên khối rồi rã ngược về ngày — thường ổn định hơn Croston khi ADI rất cao.

**Cài đặt:** gọi thẳng `statsforecast.models.ADIDA` (`intermittent_sf.py:36,38`) —
**bản thân phép gộp/rã nằm trong thư viện, repo không tự cài đặt**.
Repo chỉ dựng DataFrame (`unique_id="sku"`, `ds = date_range(end=hôm nay, periods=n, freq="D")`) và gọi
`StatsForecast(models=[ADIDA()], freq="D", n_jobs=1).forecast(df, h)` (`:49-52`).
**CHƯA CHẮC:** kích thước khối gộp (aggregation level) do statsforecast tự chọn; code mecom không truyền tham số nào.

**Khoảng tin cậy:** statsforecast chỉ trả điểm, nên p10/p90 dựng bằng **cùng bootstrap phần dư tuần** như ETS
(`_residual_bootstrap_quantiles:55`):

```
residuals = [ y[i] − y[i−7] ]  (rỗng → [0.0])                      (intermittent_sf.py:65-67)
boot      = rng(seed=42).choice(residuals, 500, replace=True)      (:64,72)
p50 = max(0, point)
p10 = max( 0, min( p50 + P10(boot), p50 ) )        ← CÓ kẹp ≤ p50   (:75)
p90 = max( p50, p50 + P90(boot) )                                   (:76)
```

**Hằng số THẬT:** `MIN_HISTORY = 14` (`:21`), `_BOOT_DRAWS = 500` (`:22`), `_BOOT_SEED = 42` (`:23`).
Thiếu lịch sử → **raise ValueError** (`:88-90`) chứ không im lặng fallback — caller bắt và rơi về router
(`forecast_run.py:1267-1271`, `:1575-1579`).

**Khi được chọn:** CHỈ khi backtest chấm ra (`kv_state.model_choice.model == "adida"`). Nó **không bao giờ thay thế**
Croston/SBA mặc định — đọc docstring `intermittent_sf.py:10-12`. Trong rổ ứng viên: `backtest.py:72`.

### 3.6. IMAPA

**Vị trí:** `core/intermittent_sf.py:103` — `imapa_forecast`. Giống hệt ADIDA về mọi mặt trong repo này
(cùng `_forecast`, cùng bootstrap, cùng hằng số), chỉ khác `model_name == "imapa"` → `statsforecast.models.IMAPA`
(`:36,38,51`).

IMAPA = **I**ntermittent **M**ultiple **A**ggregation **P**rediction **A**lgorithm — trung bình nhiều mức gộp thay vì một mức.
**CHƯA CHẮC:** danh sách mức gộp do statsforecast quyết định; repo không cấu hình.

**Số đo THẬT cho thấy ADIDA và IMAPA gần như song sinh** (bảng §11): demoshop `cov 0.905 / mase 0.896` vs `0.905 / 0.897`;
simworld1 `0.765/0.907` vs `0.764/0.908`. Chạy cả hai tốn gấp đôi thời gian backtest để nhận chênh lệch ở chữ số thứ ba.

⚠ **CÂU HỎI NGHIỆP VỤ CỦA FEATURE NÀY ĐÃ CÓ CÂU TRẢ LỜI — VÀ LÀ "KHÔNG".**
`kb_feature F-ROUTER-IMAPA-1` hiện mang trạng thái **`runtime-BELOW-BASELINE`** với `verified_by` nguyên văn:

> *"psql bt_2026-08-05 segment intermittent n=222/model: imapa mase 0.961 / adida 0.973 vs croston 0.968 (khong hon),
> coverage 0.762/0.764 vs croston 0.902 (TE hon); pinball thang: lgbm 59/60, imapa+adida 0/60 — cau hoi feature
> (hon Croston?) tra loi KHONG"*

Tức là: hai model này **thua Croston về coverage**, **ngang về MASE**, và **thắng 0/60 SKU về pinball**.
Chúng vẫn nằm trong rổ backtest, vẫn tốn CPU mỗi tuần. Đây là ứng viên rõ ràng để **rút khỏi rổ** (hoặc gate sau một
điều kiện hẹp hơn) — quyết định đó chưa được ghi ở đâu trong code.

### 3.7. `similar_item_transfer` (chuyển giao từ SKU tương tự)

**Vị trí:** `jobs/forecast_run.py:732` — `_similar_item_transfer(...)`; lấy hàng xóm `_fetch_neighbor_series:707`;
lấy id tương tự `_get_similar_ids:110` (gọi smartsearch) và `_prefetch_similar_ids:165`.

**Bài toán:** SKU mới có 3-10 ngày dữ liệu — quá ít để học gì, nhưng **không phải trắng tinh**: nó đã bán được vài cái,
cho biết MỨC. Áo thun đen size L mới ra có thể mượn **hình dạng mùa vụ** của áo thun đen size M.

**Thuật toán:**
1. Lấy tối đa 5 SKU tương tự từ smartsearch `GET /internal/similar-products` (`forecast_run.py:135-137`,
   header `X-Internal-Token`, timeout **1.5 s**, `:139`), fail-open trả `[]` (`:153-154`).
2. Duyệt từng hàng xóm, bỏ qua nếu `< min_days = 28` ngày dữ liệu (`:711`, `:727-728`).
3. **Hệ số thang:**

```
scale = mean(child_units) / mean(neighbor_units)      nếu neighbor_mean > 0
      = 0.5                                           nếu neighbor_mean == 0 hoặc child_units rỗng   (:758-763)
```

4. Chạy `seasonal_naive_forecast` **trên chuỗi hàng xóm** (`:768`), nhân cả 3 phân vị với `scale` (`:771`).
5. **Nới khoảng ×1.3 quanh p50** (`:773-781`):

```
p10' = max( 0, min( p50 − 1.3·(p50 − p10), p50 ) )
p90' = max( p50, p50 + 1.3·(p90 − p50) )
```

6. Trả về hàng xóm **ĐẦU TIÊN** dùng được (`return` trong vòng lặp, `:783`) — **không** tổng hợp nhiều hàng xóm.

**Hằng số THẬT:** `k = 5` id tương tự (`:113`), `min_days = 28` (`:711`), hệ số nới `1.3` (`:776-777`),
`scale` fallback `0.5` (`:760,763`), timeout HTTP `1.5s` (`:139`), `SIMILAR_PREFETCH_CONCURRENCY = 8` (`:162`).
Env: `SMARTSEARCH_URL` (`:90`), `MINIAI_INTERNAL_TOKEN` (`:93`).

**Khi được chọn:** **GHI ĐÈ MỌI KẾT QUẢ TRƯỚC ĐÓ** khi `len(units) < 14` (`forecast_run.py:1285-1295` batch,
`:1590-1599` on-demand). Tức là ngay cả khi backtest bảo `lgbm_global`, SKU dưới 14 ngày vẫn bị chuyển sang transfer —
và điều đó được **khai báo minh bạch** qua `model_fallback = _fallback_block("similar_item_transfer", ...)`
(`:1600-1605`, comment: "model_used != model_requested LUÔN LUÔN được giải thích").

**Cạm bẫy hiệu năng đã trả giá (comment `forecast_run.py:171-176`):** batch từng gọi smartsearch **tuần tự** một lần
mỗi SKU ngắn; smartsearch chậm → mỗi call ăn trọn timeout 1.5s → 21 SKU × 1.5s ≈ **32 giây** nằm trong `forecast:run`.
Prefetch song song (semaphore 8) kéo xuống ~`⌈21/8⌉ × 1.5 ≈ 4.5s`.

### 3.8. `cold_start_analog` (0 lịch sử — dựng hồ sơ từ nhóm SKU tương tự)

**Vị trí:** `core/analog.py:34` — `build_analog_profile(analog_series, horizon_days, widen=0.2)`;
wiring `services/forecast/app/main.py:1113` `_cold_start_analog`.

**Bài toán:** SKU **CHƯA BÁN CÁI NÀO** (0 dòng `demand_daily`) nhưng đã lên sàn. Trước đây API trả thẳng 404 —
khách hỏi "nhập bao nhiêu?" thì im lặng. Giờ mượn dáng của nhóm hàng tương tự.

**Thuật toán:**
1. Lọc analog dùng được: `len(u) >= MIN_ANALOG_DAYS` và `sum(u) > 0` (`analog.py:57`); loại analog có `mean == 0` (`:63`).
2. **Chuẩn hoá về cùng MỨC** — không cho một best-seller lấn át:

```
means[i]     = mean( units_i )
common_mean  = ( Σ means[i] ) / #analog                                   (analog.py:69)
scale_i      = common_mean / means[i]                                     (analog.py:74)
pattern_i[h] = units_i[ n_i − 7 + (h mod 7) ] · scale_i                   (analog.py:31, 75-77)
```

3. **Tổng hợp theo ngày:**

```
p50[h] = mean_i ( pattern_i[h] )                                          (analog.py:83)
p10[h] = max( 0, min( min_i(pattern_i[h]) · (1 − 0.2), p50[h] ) )         (analog.py:84,86)
p90[h] = max( p50[h], max_i(pattern_i[h]) · (1 + 0.2) )                   (analog.py:85,87)
```

**Hằng số THẬT:** `MIN_ANALOG_DAYS = 7` (`analog.py:22`), `BAND_WIDEN = 0.2` (`analog.py:25`),
`k = 5` analog và timeout HTTP `3.0s` ở lớp wiring (`main.py:1141,1147`), chỉ lấy `items[:5]` (`main.py:1167`).

**Quyết định thiết kế được VIẾT RA (`analog.py:6-8`):** **cố ý KHÔNG scale theo giá** — với SKU mới, độ co giãn giá
chưa quan sát được, "scale theo giá" chỉ là nhiễu đội lốt tín hiệu.

**Khi được chọn:** CHỈ ở `POST /v1/forecast:query` khi `forecast_on_demand` ném `ValueError("no demand history")`
(`main.py:632-645`). Đường degrade 2 bậc:
- smartsearch chết / không analog nào có demand → `None` → **404 "no demand history"** (`main.py:639-644`);
- smartsearch TRẢ LỜI nhưng danh sách rỗng → **404 với thông điệp khác**: "no demand history; not in catalog either"
  (`main.py:1156-1164`) — phân biệt "chưa bán" với "không có trong catalog".

Response gắn thêm `analog_of: [<id các SKU đã mượn>]` (`main.py:785-786`), `model_used = "cold_start_analog"`,
`run_id = "analog_<YYYY-MM-DD>"` (`main.py:656-657`).

### 3.9. LightGBM global quantile (`lgbm_global`)

**Vị trí:** `core/global_model.py:101` `train_global` · `:202` `predict_quantiles` · `:234` `predict_quantile_grid`;
feature `libs/featurelib/forecast_features.py:37` (train) và `:199` (serve).

**Bài toán:** với tenant đủ lớn, huấn luyện **MỘT model chung cho toàn bộ SKU** (global/cross-learning) mạnh hơn hẳn
mỗi SKU một model: SKU ít data mượn được quy luật của SKU nhiều data; hiệu ứng giá/promo/ngày-trong-tuần học một lần.

**Điều kiện huấn luyện (bar):**

```
n_skus = df.product_id.nunique()                    ;  n_skus < min_skus  → return None   (global_model.py:128-133)
days_per_sku.max() < min_days                        → return None                        (:136-142)
```

Giá trị THẬT truyền vào: `min_days = 120`, `min_skus = 50` (`forecast_run.py:594-596`; backtest dùng
`min_days = max(60, 120 − origin)` vì cắt bớt `origin` ngày, `backtest_run.py:278`).

**Bộ feature (21 cột, `global_model.py:159-165`):**

| Nhóm | Cột | Công thức | Dòng featurelib |
|---|---|---|---|
| Lag | `lag_1, lag_7, lag_14, lag_28` | `groupby(sku).adjusted_units.shift(L)` | `:102-103` |
| Rolling | `rollmean_7, rollmean_28` | `rolling(w, min_periods=w).mean()` **rồi `.shift(1)`** | `:107-115` |
| Rolling | `rollstd_7, rollstd_28` | `rolling(w, min_periods=w).std()` rồi `.shift(1)` | `:111-116` |
| Lịch | `dow_0..dow_6` | one-hot `day.dt.dayofweek` (thứ THẬT, khác `seasonal_naive`) | `:119-126` |
| Giá | `price_rel` | `price / median28.shift(1)`, NaN khi median NaN hoặc 0 | `:129-138` |
| Promo | `promo_pct` | thang **0..100** (không phải 0..1) | `:94` |
| Hết hàng | `days_since_stockout` | số ngày kể từ lần stockout gần nhất, **9999 nếu chưa từng** | `:148-156` |
| Quy mô SKU | `sku_freq` | `expanding(min_periods=1).mean().shift(1)` | `:159-162` |
| **Derived** | `on_promo` | `(promo_pct.fillna(0) > 0).astype(float)` | `global_model.py:54` |
| **Derived** | `cal_factor` | `calendar_effect.day_factor(day, events)` (1.0 nếu không có event) | `global_model.py:56-60` |

Chống rò rỉ: mọi feature đều `shift(1)` hoặc lag ≥ 1; `dropna(subset=["lag_28"])` → **cần ≥ 29 ngày/SKU**
(`forecast_features.py:168`). Target = `adjusted_units` (`:165`).

**Huấn luyện — 9 booster phân vị (`global_model.py:170-189`):**

```
QUANTILE_GRID = (0.01, 0.05, 0.10, 0.25, 0.50, 0.75, 0.90, 0.95, 0.99)     (global_model.py:21-23)

với mỗi alpha:
    params = { objective: "quantile", alpha: α, metric: "quantile",
               learning_rate: 0.05, num_leaves: 63, min_data_in_leaf: 50, verbose: −1 }
    lgb.train(params, Dataset(X, y), num_boost_round = 400)
    models[ f"q{α:.2f}" ] = booster
```

Hàm mất mát pinball chính là `objective="quantile"` của LightGBM:
`L_α(y, q) = α·(y−q)` nếu `y ≥ q`, `(α−1)·(y−q)` nếu `y < q`.

**Alias tương thích ngược (`global_model.py:29,191-193`):** `models["p10"|"p50"|"p90"]` **trỏ tới CÙNG object**
với `q0.10/q0.50/q0.90` (không train lại) → mọi caller cũ giữ nguyên hành vi.

**Dự đoán:**
- `predict_quantiles` (`:202`): lấy 3 booster p10/p50/p90, `sorted([p10,p50,p90])` (`:230`) — sửa quantile crossing bằng cách **sắp xếp**.
- `predict_quantile_grid` (`:234`): chạy toàn bộ 9 alpha, sửa crossing bằng
  `np.maximum.accumulate(np.maximum(raw, 0.0))` (`:253`) — **cummax sau khi kẹp ≥ 0**, giữ đơn điệu theo alpha.

**Hằng số THẬT + env:**

| Hằng | Giá trị | Vị trí |
|---|---|---|
| grid 9 alpha | 0.01 … 0.99 | `global_model.py:21` |
| `learning_rate` | 0.05 | `:179` |
| `num_leaves` | 63 | `:180` |
| `min_data_in_leaf` | 50 | `:181` |
| `num_boost_round` | 400 | `:187` |
| `min_days` / `min_skus` | 120 / 50 | `forecast_run.py:595-596` |
| TTL cache bundle | 3600 s | `FORECAST_GLOBAL_MODEL_TTL_SEC`, `model_cache.py:167` |
| Số project cache tối đa | 4 | `FORECAST_GLOBAL_MODEL_CACHE_MAX`, `model_cache.py:171` |
| Slot train đồng thời | 1 | `FORECAST_GLOBAL_MODEL_TRAIN_SLOTS`, `model_cache.py:288` |
| max_age cho batch | 60 s | `FORECAST_GLOBAL_MODEL_BATCH_MAX_AGE_SEC`, `forecast_run.py:53-55` |
| TTL nhớ lỗi train | 300 s | `FORECAST_GLOBAL_MODEL_ERROR_TTL_SEC`, `forecast_run.py:58-60` |

**Cạm bẫy đã trả giá — W-PROMO-PREVIEW-CACHE (docstring `model_cache.py:1-61`):**
1. `forecast_on_demand` từng gọi `train_global` **mỗi request**: `SELECT * FROM demand_daily` toàn tenant + 9 booster × 400 vòng.
2. `demoshop` (khi đó 18 SKU) trả lời trong 0.013s — **nhưng chỉ vì `train_global` bail out ở `n_skus < min_skus`
   TRƯỚC khi train**, request âm thầm tụt xuống router mà `model_used` vẫn báo bình thường. **Không ai thấy được sự tụt hạng.**
3. `simworld1` (60 SKU × 432 ngày, nơi model global THỰC SỰ đủ điều kiện) mất **90-180 giây/request**, vượt mọi client timeout.
4. → "endpoint xanh trên tenant nhỏ *chính vì* nó hỏng ở đó, và không dùng được ở tenant nó hoạt động."
5. Bản vá: cache theo process, khoá `asyncio.Lock` per-project (`model_cache.py:262`), semaphore toàn process
   (`:282`), **cache CẢ kết quả âm** (`bundle=None` cũng lưu, `model_cache.py:93-101`), và
   `asyncio.to_thread(train_global, ...)` (`forecast_run.py:592`) vì **LightGBM giữ CPU hàng phút và nhả GIL —
   train trên event loop chặn TOÀN BỘ HTTP của service (sự cố live 2026-08-04)**.
6. Đường "không chờ": `wait=False` cho promo-preview (`forecast_run.py:660-667`) → trả lời NGAY bằng router,
   `model_fallback.reason = "training"`, và hâm nóng model nền (`model_cache.start_background_train:303`).

**Từ vựng đóng của `model_fallback.reason`** (`forecast_run.py:66-87`, cũng là nhãn Prometheus):
`training` · `insufficient_data` · `no_history` · `error` · `similar_item_transfer`.

**Cạm bẫy #2 — SW-1 (comment `forecast_features.py:83-88` và `:264-268`):**
`days_since_stockout` gọi `.days` trên hiệu hai `numpy.datetime64` (`numpy.timedelta64` **không có** `.days`)
→ `AttributeError`, sập ngay lần đầu backtest `lgbm_global` chạy dữ liệu thật (simworld1). Sửa bằng **một đường
chuẩn hoá ngày DUY NHẤT** `df["day"] = pd.to_datetime(df["day"])` ngay tại cửa (`:88`, `:269`). Lỗi này xuất hiện **2 lần**
(build_feature_frame và build_future_features) — dấu hiệu của code sao chép.

**Cạm bẫy #3 (comment `forecast_run.py:463-465`):** trước PROD5H-FC, `_fetch_demand_frame_local` **không SELECT**
`price/promo_pct/stockout` → model global train với `promo_pct` **hằng số 0**, covariate không mang tín hiệu nào.

---

## 4. ROUTER + BACKTEST CHỌN MODEL

### 4.1. Router tĩnh (khi CHƯA có backtest)

**Vị trí:** `core/router.py:14` — `route_and_forecast(units, horizon_days)`.

```
cls = classify_series(units)

segment == "cold_start"                       → cold_start_forecast          , model_used="cold_start"
segment ∈ {intermittent_croston, intermittent_sba}
                                              → croston_daily_forecast(sba = segment=="intermittent_sba")
                                                                             , model_used="sba" | "croston"
segment == "smooth"  và  len(units) ≤ 56      → seasonal_naive_forecast      , model_used="seasonal_naive"
segment == "smooth"  và  len(units) >  56     → ets_theta_forecast           , model_used="autoets_theta_ensemble"
                                                (exception → seasonal_naive)
```

Sau đó **làm sạch bắt buộc** (`router.py:49-56`): `sort_quantiles(p10,p50,p90)` rồi kẹp cả ba `>= 0`.

**Hằng số THẬT:** ngưỡng `56` ngày (`router.py:38`) — tương ứng "ít nhất 8 tuần thì mới đáng chạy ETS/Theta".
Không env override.

### 4.2. Backtest rolling-origin (đường chấm điểm)

**Vị trí:** `core/backtest.py:77` — `backtest_series(units, origins=(28,21,14), horizon=7)`.

**Cơ chế:** với mỗi `origin o ∈ {28, 21, 14}`:

```
train  = units[ : −o ]                   # bỏ o ngày cuối
actual = units[ −o : −o+horizon ]        # 7 ngày ngay sau điểm cắt
bỏ qua nếu len(train) < 14 hoặc actual rỗng                                   (backtest.py:115-118)
```

Với mỗi model ứng viên, dự báo `horizon = 7` ngày từ `train` (`backtest.py:131-133`, exception → bỏ qua model đó ở origin này),
rồi cộng dồn theo NGÀY:

```
pinball_qα += L_α( y_i , p_α,i )      với  L_α(y,q) = max( α(y−q), (α−1)(y−q) )      (backtest.py:16-18, 140-142)
mae        += | y_i − p50_i |                                                          (:143)
coverage_hits += 1  nếu  p10_i ≤ y_i ≤ p90_i                                           (:144-145)
days       += 1
```

**Rổ ứng viên (`_candidate_models`, `backtest.py:65`):**

```
ADI > 1.32 → [ "seasonal_naive", "croston_auto", "adida", "imapa" ]
ngược lại  → [ "seasonal_naive", "autoets_theta_ensemble" ]
```
`seasonal_naive` **luôn** được chèn vào nếu thiếu (`backtest.py:93-95`). `lgbm_global` **không có ở đây** —
nó được cộng thêm ở tầng job (`backtest_run.py:400`).

**Mẫu số MASE — W-MASE-UNIFY (`seasonal_naive_scale`, backtest.py:21):**

```
m = 7;   nếu n < 2m  →  m = 1                                                (backtest.py:33-34)
nếu n < m+1  →  return None
scale = ( 1/(n−m) ) · Σ_{t=m}^{n−1} | y_t − y_{t−m} |                        (backtest.py:37-38)
return scale nếu scale > 0, ngược lại None                                    (:39)
```

Đây là **MAE trong-mẫu của seasonal-naive** (Hyndman 2006 / M-competitions), tính **CHỈ trên `train`**
(`backtest.py:127`) — cửa sổ hold-out không bao giờ rò vào mẫu số.

```
avg_scale = ( Σ_origin scale_o · len(actual_o) ) / ( Σ_origin len(actual_o) )   # bình quân theo NGÀY (:171)
MASE      = mae / avg_scale                                                     (:172)
MASE      = None  nếu  model == "seasonal_naive"                                (:168-169)   ← nó LÀ baseline
MASE      = None  nếu không origin nào tính được scale                          (:173-174)
```

⚠ **Bài học được ghi thẳng vào comment (`backtest.py:120-127`):** bản CŨ dùng "MAE của dự báo seasonal-naive trên
actual ngoài mẫu" — **một định nghĩa MASE KHÁC** với đường `lgbm_global` (dùng scale trong-mẫu).
Hai đường chấm điểm khác thước → so sánh model là vô nghĩa. Nay thống nhất một helper duy nhất.

⚠ `scale = 0` (chuỗi hằng / toàn 0) trả `None` chứ **không trả 0.0** (`backtest.py:39`) — vì `mase = mae/0` hoặc
`mase = 0.0` sẽ **giả trang thành model hoàn hảo**.

**Metric xuất ra (`backtest.py:176-182`):** `pinball_q10`, `pinball_q50`, `pinball_q90` (trung bình **theo ngày**),
`mase`, `coverage_p10_p90`.

### 4.3. `choose_model` — luật chọn

**Vị trí:** `core/backtest.py:186`.

```
1. metrics rỗng                          → "seasonal_naive"
2. lọc model có pinball_q50 khác None; rỗng → "seasonal_naive"
3. best = argmin( pinball_q50 )
4. nếu pinball_q50[best] > pinball_q50["seasonal_naive"]  → "seasonal_naive"
5. ngược lại → best
```

⚠ **Tiêu chí chọn CHỈ là `pinball_q50`** — tức là chất lượng **điểm dự báo trung vị**.
`coverage` và `mase` **KHÔNG tham gia chọn**; coverage chỉ được dùng SAU đó để tính `width_factor`.
Hệ quả: một model có điểm dự báo tốt nhưng khoảng tin cậy tệ vẫn thắng, rồi mới bị "kéo/nén" khoảng.
Đây là một **đánh đổi có thật, không được viết ra trong code** — code chỉ ghi "(spec §5.3)".

### 4.4. Job backtest + nơi lưu `model_choice`

**Vị trí:** `jobs/backtest_run.py:88` — `run_backtest_once(pool, project_id=None)`.
Vòng lặp nền `start_backtest_loop:507`, chu kỳ `BACKTEST_INTERVAL` mặc định **604800 s = 7 ngày** (`:514`).
Khoá: `pg_advisory_lock(hashtext("backtest_run"))` — **TOÀN CỤC, không per-project** (`:96-98`), khác với forecast_run.

**Ba nhánh:**

| Điều kiện | Xử lý | Dòng |
|---|---|---|
| `sku_count < 50` | chỉ chạy `backtest_series` per-SKU (không lgbm) | `:132-187` |
| `hist_days.max() < 120` | như trên | `:199-255` |
| đủ bar | thêm nhánh `lgbm_global` OOF | `:257-450` |

Mọi nhánh: bỏ qua SKU có `len(series) < 42` ngày (`:136`, `:204`, `:288`).

**Nhánh `lgbm_global` OOF (`:264-400`)** — với mỗi `origin ∈ (28,21,14)`:
1. `cut_day = max_day − origin`; `df_cut = df_full[day <= cut_day]`.
2. `train_global(df_cut, min_days = max(60, 120−origin), min_skus = 50, calendar_events)`
   chạy trong `asyncio.to_thread` (`:275-281`) — comment `:272-274`: *"9 booster × 3 origin giữ CPU hàng phút và nhả GIL —
   train đồng bộ trên event loop chặn TOÀN BỘ HTTP của service (sự cố live 2026-08-04, py-spy bắt tại đây)"*.
3. `build_future_features(sku_df, horizon_days=7)` + `augment_future_features(..., cut_day+1)` (`:306-313`) —
   **cố ý dùng đúng đường feature của lúc serve** để OOF không lệch (PROD5H-FC).
4. `preds` phải đúng 7 phần tử (`:319-320`), `actual` phải đủ 7 ngày (`:328-330`), nếu không → bỏ SKU ở origin này.
5. Metric tính tay: `pin_q10/q50/q90` chia 7 (`:342-353`), `coverage = cov_cnt/7` (`:356-360`),
   `mase = mae_model / seasonal_naive_scale(history, m=7)` (`:366-371`), `None` nếu scale `None`.
6. Trộn vào rổ: `metrics["lgbm_global"] = lgbm_metrics` (`:400`) → `choose_model(metrics)` (`:403`).

**Ghi kết quả:**
- `backtest_results(project_id, product_id, run_id, model, segment, pinball_q10, pinball_q50, pinball_q90,
  mase, coverage_p10_p90, created_at)` — **MỘT DÒNG CHO MỖI MODEL**, không chỉ model thắng (`:410-431`).
  `run_id = "bt_" + <YYYY-MM-DD>` (`:104`).
- `kv_state` key **`model_choice:<project_id>:<product_id>`**, value
  `{"model": <tên>, "width_factor": w, "empirical_coverage": cov}` (`:433-449`, UPSERT `ON CONFLICT (k)`).

**Đây là câu trả lời cho "model_choice lưu ở đâu": bảng `kv_state`, cột `v` kiểu jsonb.**
Đường đọc: `_get_model_choice` (`forecast_run.py:194`).

### 4.5. Đường dispatch lúc dự báo (thứ tự ưu tiên THẬT)

`forecast_run.py:1172-1295` (batch) và `:1456-1599` (on-demand) — cùng một cấu trúc:

```
choice = kv_state[ model_choice:<proj>:<sku> ]
model_name    = choice.model         (None nếu chưa backtest)
width_factor  = choice.width_factor  (mặc định 1.0)
empirical_cov = choice.empirical_coverage

1. model_name == "lgbm_global" và có bundle  → build_future_features + augment + predict_quantiles
                                               (lỗi predict → router + model_fallback="error")
2. model_name == "seasonal_naive"            → seasonal_naive_forecast
3. model_name == "autoets_theta_ensemble"    → ets_theta_forecast
4. model_name == "croston_auto"              → croston_daily_forecast(sba = CV² > 0.49)
5. model_name ∈ {"adida","imapa"}            → hàm tương ứng (exception → route_and_forecast)
6. model_name lạ hoặc None                   → route_and_forecast  (router tĩnh §4.1)

7. NẾU len(units) < 14  → similar_item_transfer GHI ĐÈ kết quả trên     (:1285-1295 / :1590-1599)
8. NẾU width_factor ≠ 1 → apply_calibration                             (:1298-1299 / :1607-1608)
9. NẾU có promo tương lai VÀ model ≠ lgbm_global → _apply_promo_uplift   (:1304-1307 / :1612-1613)
10. Lịch (Tết/lễ) KHÔNG nướng vào dòng lưu — áp lúc ĐỌC                  (:1309-1311 / :1615-1616)
```

Ghi ra bảng `forecasts` với `run_id = "r_" + <YYYY-MM-DD>` (`:1094`, `:1628`), ngày bắt đầu **hôm nay + 1**
(`:1316`, `:1621`), `data_window = "<ngày đầu>..<ngày cuối>"` của chuỗi lịch sử (`:1322`, `:1626`).

**Hằng số THẬT:** `HORIZON_DEFAULT = 28` cho batch (`forecast_run.py:42`); API `forecast:query` mặc định `horizon_days = 14`,
hợp lệ `1..56` (`main.py:589-595`). `FORECAST_RUN_INTERVAL` mặc định **86400 s** (`forecast_run.py:1666`).

**Khoá:** `pg_advisory_lock(hashtext("forecast_run:<project>"))` — **PER-PROJECT** (`forecast_run.py:1114-1116`),
nhả trước khi sang project kế (`:1345-1347`). Comment `:1078-1084`: với khoá toàn cục cũ, `forecast:run` của một tenant
phải xếp hàng sau batch của TẤT CẢ project (p1 = 623 SKU + demo + các simworld) — probe seedtest đo **59 giây** dù bản thân
batch chỉ tốn ~1 giây.

**W-RUN-ASYNC-202 (`forecast_run.py:807-812`):** `POST /v1/forecast:run` từng làm cả train + dự báo TRONG request —
4/4 probe fail đều là **timeout thuần, chưa bao giờ 4xx/5xx**. Nay enqueue job (`job_run`, type `forecast_run`,
job_id `fr-<project>-<run_id>`) và trả **202** ngay; client poll `GET /v1/projections/status?job_id=...`.

---

## 5. LỊCH / MÙA VỤ VIỆT NAM

### 5.1. `calendar_effect` — 3 pha nhân hệ số

**Vị trí:** `core/calendar_effect.py` — `load_events:44`, `_phase_for:80`, `day_factor:97`, `apply_calendar:118`.

**Bài toán:** Tết Âm lịch trôi theo năm (không cố định dương lịch); trước Tết cầu bùng, trong Tết đóng cửa,
sau Tết ế. Ngày 11/11 flash sale thì cửa sổ ảnh hưởng chỉ ~3 ngày chứ không phải 20.

**Nguồn dữ liệu:** bảng `calendar_events(event, date_start, date_end, uplift_pre, uplift_in, uplift_post,
pre_days, post_days)`; `project_id IN ('', <project>)` — **`''` là lịch DÙNG CHUNG cho mọi tenant**
(`forecast_run.py:410-418`, `main.py:548-556`).

**Ba pha (`_phase_for`, calendar_effect.py:80-94):**

```
in   :  date_start ≤ d ≤ date_end                              → uplift_in
pre  :  date_start − pre_days  ≤ d < date_start                → uplift_pre
post :  date_end < d ≤ date_end + post_days                    → uplift_post
ngoài:  → factor 1.0
```

**Hợp thành:** nhiều event chồng nhau **NHÂN với nhau** (`day_factor`, `:111-114`; `apply_calendar`, `:140-145`).
Vì nhân cả `p10,p50,p90` bằng **cùng một hệ số dương**, bất biến `p10 ≤ p50 ≤ p90` được bảo toàn (docstring `:16-17`).

**Hằng số THẬT:** `PRE_DAYS_DEFAULT = 20`, `POST_DAYS_DEFAULT = 7` (`calendar_effect.py:31-32`) — chỉ dùng khi
event không có cột `pre_days/post_days` riêng (V007). Cột per-event **luôn thắng** (`:86-87`).

**Chống dữ liệu bẩn (`load_events`, `:52-77`):** bỏ event thiếu field / `date_end < date_start` /
`min(uplift) <= 0` / `pre_days < 0` hoặc `post_days < 0`. *"Seed data xấu không bao giờ được phép làm chết một truy vấn dự báo."*

### 5.2. `cal_factor` — cùng một hệ số, hai đường áp dụng

Đây là chỗ tinh tế nhất của tầng lịch. Đọc comment `forecast_run.py:389-398`:

> **ANTI-DOUBLE-COUNT (PROD5H-FC):** hệ số `holidays_vn` hardcode từng được nướng vào dòng lưu ở đây,
> trong khi `forecast:query` lại nhân **CÙNG cửa sổ Tết đó một lần nữa** từ `calendar_events`
> → **pre-Tết 1.8 × 1.3 = 2.34**.

Nguồn duy nhất hiện nay là bảng `calendar_events`, áp **ĐÚNG MỘT LẦN mỗi đường**:

| Đường | Cách áp | Vị trí |
|---|---|---|
| Model thống kê (naive/ETS/Croston/ADIDA/IMAPA/transfer) | **nhân lúc ĐỌC**: `apply_calendar(daily, events)` trong `forecast:query` và `forecast:aggregate` | `main.py:672-679`, `main.py:1447-1448` |
| `lgbm_global` | **học** qua covariate `cal_factor = day_factor(day, events)` | `global_model.py:56-60`, `:156` |

Điều kiện chặn ở cả hai endpoint là **`model_used != "lgbm_global"`** (`main.py:673`, `main.py:1447`).
`day_factor` và `apply_calendar` dùng **chung `_phase_for`** (docstring `calendar_effect.py:105-107`) nên feature của model
và bộ nhân của đường thống kê **không thể trôi lệch nhau**.

⚠ Comment `main.py:1430-1432`: trước PROD5H-FC, đường `:aggregate` **bỏ qua `calendar_events` hoàn toàn** →
cùng một SKU cho hai con số khác nhau giữa `:query` và `:aggregate` trong cửa sổ Tết.

### 5.3. `holidays_vn` — ĐÃ NGHỈ HƯU

**Vị trí:** `libs/featurelib/holidays_vn.py:69` — `holiday_factor(d) -> (multiplier, reason)`.

Hằng số: `TET_DATES = {2026: 17/02, 2027: 06/02, 2028: 26/01}` (`:29-33`), `HF_PRE_TET = 1.8` (`:44`),
`HF_TET_CLOSED = 0.3` (`:45`), `HF_POST_TET = 0.7` (`:46`), `HF_FIXED_HOLIDAY = 1.3` (`:47`),
`HF_PRE_FIXED_HOLIDAY = 1.2` (`:48`), `PRE_TET_DAYS = 7` (`:51`), `POST_TET_DAYS = 7` (`:53`);
lễ cố định: 1/1, 30/4, 1/5, 2/9 (`:36-41`).

**Đo được:** `grep` toàn repo cho thấy module này **CHỈ còn được import bởi `tests/forecast/test_holidays.py`** —
**không một dòng code sản phẩm nào gọi nó**. Đây là hệ quả trực tiếp của bản vá anti-double-count §5.2.
Chính test file cũng ghi (`test_holidays.py:64-67`): *"hệ số holidays_vn hardcode không còn được nướng vào dòng lưu"*.
→ **Code chết có kiểm soát**, nhưng vẫn là bẫy cho người đọc mới (hằng số 1.8/0.3/0.7 trông rất "đang chạy").

---

## 6. HIERARCHICAL RECONCILIATION (dự báo NHẤT QUÁN theo cấp bậc)

### 6.1. Bài toán

Đọc docstring `core/scenario/reconcile.py:3-13`: dự báo SKU, dự báo ngành hàng, dự báo tổng — mỗi cái tính độc lập thì
**tổng các SKU KHÔNG bằng dự báo tổng**. Khách nhìn báo cáo thấy 3 con số không cộng lại được → mất niềm tin.

### 6.2. Toán học (`core/scenario/reconcile.py`)

**Ma trận cộng dồn `S`** (`build_hierarchy:149`): vector nút `y = [__total__, __cat_*__ (sắp xếp), SKU (thứ tự đầu vào)]`,
`S` kích thước `(n_nodes × n_bottom)`:
- hàng 0 (total) = toàn 1 (`:173`);
- hàng category `c` = 1 tại cột SKU thuộc `c` (`:174-177`);
- khối đáy = `I` (`:178`); `S.setflags(write=False)` — bất biến.

Mọi dự báo **nhất quán (coherent)** đều có dạng `ỹ = S·b̃`.

**MinT (`mint_weights:271`):**

```
G = ( Sᵀ W⁻¹ S )⁻¹ Sᵀ W⁻¹              (n_bottom × n_nodes)         (reconcile.py:276-300)
ỹ = S · G · ŷ                                                       (mint_reconcile:303, 318-323)
Mọi G đều thoả  G·S = I  ⇒ giữ tính không chệch, và là ánh xạ đồng nhất trên dự báo đã nhất quán.
```

**Bốn lựa chọn `W`** (`_MINT_METHODS`, `reconcile.py:79`):

| method | `W` | Cài đặt |
|---|---|---|
| `ols` | `I` | `winv_s = S` (`:281-282`) |
| `wls_struct` | `diag(S·1)` = số lá dưới mỗi nút | `winv_s = S / w_diag[:,None]` (`:283-286`) |
| `wls_var` | `diag(var mẫu của residual)`, sàn `1e-12` | `:293-295` |
| `mint_shrink` | hiệp phương sai co Schäfer–Strimmer | `np.linalg.solve(W, S)` (`:296-298`) |

**Shrinkage Schäfer–Strimmer (`shrinkage_covariance:239`):**

```
xc      = r − mean(r)                       ;  cov = xcᵀxc / T
d       = sqrt( max(diag(cov), 1e-12) )     ;  corr = cov / (d ⊗ d)
xs      = xc / d
w_ijt   = xs_ti · xs_tj                     ;  var_corr_ij = Σ_t (w_ijt − mean_t w_ij)² · T / (T−1)³
λ*      = clip(  Σ_offdiag var_corr / Σ_offdiag corr² ,  0, 1 )         (:262)
corr'   = corr · (1 − λ*)   ,  đường chéo đặt lại = 1                    (:263-264)
W       = corr' ⊙ (d ⊗ d)                                                (:265-267)
```

**Kẹp không âm (`mint_reconcile`, `:321-322`):** clip `b̃ >= 0` **TRƯỚC** khi nhân `S` → đầu ra vẫn **nhất quán chính xác**,
đổi lại có chệch lên nhẹ ở các ô bị kẹp (nói thẳng trong docstring `:314-317`).

**Ngoài ra có** `bottom_up:203` và `top_down:208` (top-down mặc định dùng **tỷ trọng dự báo** kiểu Gross-Sohl method F,
`:228-236`; nếu đáy cộng bằng 0 thì chia đều — "nhất quán quan trọng hơn chính xác", `:232-234`).
**CHƯA CHẮC:** hai hàm này không thấy được gọi từ đường sản phẩm nào — chỉ MinT được nối vào API.

### 6.3. Chính sách chọn `W` theo dữ liệu của TỪNG tenant

**Vị trí:** `core/hier_reconcile.py:126` — `choose_method(n_obs, t_var, t_shrink)`.

```
T < min_t_var            → "wls_struct"   (chỉ cần S — tenant non thì đừng bịa ma trận lỗi)
min_t_var ≤ T < min_t_shrink → "wls_var"  (W chéo từ phương sai mẫu; off-diagonal với ít quan sát là nhiễu)
T ≥ min_t_shrink         → "mint_shrink"  (đủ dài mới dùng hiệp phương sai đầy đủ)
```

**Hằng số THẬT + env override (`hier_reconcile.py:83-118`):**

| Tham số | Mặc định | Env |
|---|---|---|
| bật/tắt | **OFF** (`"0"`) | `FORECAST_RECONCILE` (bật = `"1"`) |
| `min_t_var` | 8 | `FORECAST_RECONCILE_MIN_T_VAR` |
| `min_t_shrink` | 30 | `FORECAST_RECONCILE_MIN_T_SHRINK` |
| `lookback_days` | 90 | `FORECAST_RECONCILE_LOOKBACK_DAYS` |
| `COHERENCE_EPS` | `1e-6` | — (`:90`) |

`_env_int` chỉ nhận giá trị `> 0`, ngược lại về mặc định (`:98-106`).

**Giới hạn được TUYÊN BỐ (docstring `hier_reconcile.py:38-51`):** DB chỉ có residual cấp SKU
(`forecasts.p50` vs `demand_daily`); residual của nút tổng hợp được **lấy bằng tổng residual thành viên** (`node_residuals:157`,
`r @ h.S.T`). Với tổng nhiều SKU thì gần đúng (CLT) nhưng **không đồng nhất** — đó chính là lý do `mint_shrink`
(đọc off-diagonal của ma trận dựng từ các hàng tổng hợp cộng tuyến hoàn hảo) bị chặn sau một `T` lớn hơn nhiều so với `wls_var`.
Tính residual per-level thật sẽ phải replay MC cho mọi ngày quá khứ → **cố ý không làm (chi phí), và ghi ra thay vì giấu**.

**Ma trận residual (`residual_matrix_from_rows:387`):** chỉ giữ **ngày mà MỌI SKU trong phạm vi đều có quan sát**
(`:407`) — một ngày thiếu thành viên sẽ làm sai lệch residual tổng hợp xuống dưới và kéo `W` lệch.
SQL nguồn: `store/forecasts.py:186` `fetch_residual_history` — lấy p50 của run **tạo TRƯỚC ngày dự báo**
(`created_at::date < horizon_day`, `:231`) nên dự báo không bao giờ "nhìn thấy kết quả của chính nó";
`residual = actual − forecast`.

### 6.4. Nối vào API

**Vị trí:** `main.py:1459-1466` (trong `POST /v1/forecast:aggregate`) → `_reconcile_aggregate:1246` →
`hier_reconcile.reconcile_aggregate:226`.

- Cờ tắt (mặc định) → response **byte-identical với v1** (`main.py:1459-1460`).
- Cấp category do **CALLER truyền** qua body `sku_categories` (`_parse_sku_categories:1220`) vì forecast không có catalog;
  input sai định dạng bị **BỎ QUA (cây 2 tầng)** chứ không 400 — lý do viết ra (`main.py:1226-1229`):
  *"một lỗi 400 xuất hiện/biến mất theo cờ phía server còn tệ hơn một no-op có tài liệu"*.
- `< 2 category phân biệt` → bỏ hẳn tầng category (`hier_reconcile.py:280-284`): một hàng category duy nhất là **bản sao
  y hệt hàng total** — không thêm thông tin mà lại nhân đôi trọng số của tầng đó trong `W`.
- Sau khi hoà giải, correction được **cộng** vào từng SKU: `shift_triples:364` — cộng hằng số rồi kẹp 0,
  giữ nguyên HÌNH DẠNG (độ rộng, khoảng cách phân vị), và cả hai phép đều đơn điệu nên `p10≤p50≤p90` sống sót.
- Tối ưu (`main.py:1296-1301`): nếu mọi delta `< 1e-9` thì **không chạy lại Monte Carlo 2000 draw/SKU**.
- Tự kiểm chứng: nếu `coherence_error_after > COHERENCE_EPS · max(1, max|recon|)` hoặc có giá trị không hữu hạn
  → **tự tuyên bố thất bại**, trả `reconciled=false` + `degraded_reason`, phục vụ số gốc (`hier_reconcile.py:327-340`).
- **Không bao giờ 5xx**: mọi exception → `ReconOutcome(reconciled=False, degraded_reason=...)` (`:317-325`).

**Hợp đồng đo được từ ngoài (`hier_reconcile.py:191-218`):** `coherence_error_before` > 0 (MC tổng ≠ tổng median SKU),
`coherence_error_after` ≈ 0. Và ghi chú thẳng trong response: `totals` vẫn là **phân phối MC của TỔNG**, vì
*"quantile-của-tổng ≠ tổng-của-quantile, do xác suất chứ không do thiếu nhất quán"*.

---

## 7. SCENARIO FABRIC (Monte Carlo tái lập được)

Đường ống **CỨNG** (docstring `generator.py:5-12`, `calibrate.py:3-5`):

```
Q_raw ──► Q_calibrated (TailCalibrator, TRƯỚC khi lấy mẫu)
      ──► copula ghép:  z = Λ·f + ε ,  u = Φ(z/σ_z)
      ──► scenario samples = marginal.ppf(u)
      ──► quantile_view SUY RA TỪ samples
```

### 7.1. RNG counter-based Philox

**Vị trí:** `core/scenario/rng.py:59` — `ScenarioRNG(run_id, generator_version)`.

**Hợp đồng (`rng.py:3-12`):**
`eps[s, sku, d] = Philox( key = hash(run_id, generator_version), counter = (scenario_id, sku_id, day, component) )`.
Mỗi ô có **counter 256-bit RIÊNG**, nên giá trị của một ô phụ thuộc **CHỈ** vào
`(run_id, generator_version, scenario_id, sku_id, day, component)` — bit-identical bất kể query nào hỏi,
theo thứ tự nào, hình dạng block nào, tiến trình nào. Đây là tính chất "**cùng một thế giới**" của ADR-009.

```
key      = sha256( f"{run_id}\x1f{generator_version}" )[0:16] → 2 uint64        (rng.py:65-75)
counter  = [ word0 = scenario_id ,  word1 = day ,  word2 = sku64 ,  word3 = component ]   (rng.py:79-80)
sku64    = int.from_bytes( sha256(sku_id)[:8] , "big" )                          (rng.py:46-48)
u        = ( (raw >> 11) + 0.5 ) · 2⁻⁵³        →  u ∈ (0,1) MỞ hai đầu           (rng.py:43, 51-56)
```

`u` mở hai đầu (không bao giờ đúng 0 hay 1) là bắt buộc để `norm.ppf(u)` không ra `±inf`.

**Vector hoá (`uniform_block:97`):** numpy Philox4x64 phát ra **đúng 4 uint64 mỗi giá trị counter**
(đo 2026-08-04, comment `rng.py:21-26`: `random_raw(12)` từ counter C bằng `random_raw(8)` từ counter C+1 dịch 4).
Nên code rút `4 × span` từ vựng thô bắt đầu ở `scenario = s0` và giữ **mỗi phần tử thứ 4** (`raw_all[0::4]`, `:135`) —
Python chỉ lặp trên `days`, không lặp trên từng ô. Bảo đảm: `uniform_block([s], sku, [d], c)[0,0] == uniform(s, sku, d, c)` **bit-exact** (`:108-110`).

### 7.2. Marginal — 3 loại

**Vị trí:** `core/scenario/marginal.py`. Hợp đồng: `ppf(u) -> value`, vector hoá, **kẹp ≥ 0** (`:41`).

**a) `IntermittentMarginal:48`** — Bernoulli(xảy ra) × NegBin(kích thước), thế giới Croston:

```
p0 = 1 − p_occurrence
u < p0            → 0
u ≥ p0            → nbinom.ppf( (u − p0)/p_occurrence , n = k , p = k/(k+size_mean) )    (:85-90)
E[X] = p_occurrence · size_mean  (theo cấu trúc)
NegBin:  var = mean + mean²/k   ,  k → ∞ tiến về Poisson                                 (:60-62)
```

**b) `QuantileGridMarginal:95`** — nội suy đơn điệu PCHIP trên lưới `{alpha: value}` (thế giới LightGBM):

```
values ← np.maximum.accumulate( max(values, 0) )       # cummax sửa quantile crossing    (:125)
u ∈ [α_min, α_max]  → PchipInterpolator(alphas, values)(u)                                (:128, 152-153)
u < α_min           → kẹp tại value(α_min)
u > α_max , tail=None → value(α_max)
u > α_max , tail="gp" → value(α_max) + genpareto.ppf( (u−α_max)/(1−α_max) , ξ , scale=β ) (:154-163)
                        (genpareto.ppf(0) == 0 ⇒ nối LIÊN TỤC tại điểm gắn)
```
Kiểm tra hợp lệ: alpha phải nằm **hẳn trong (0,1)** (`:119-120`), `β > 0` (`:139-140`).

**c) `EmpiricalResidualMarginal:168`** — `base + quantile thực nghiệm(residuals)`, kẹp ≥ 0 (`:183-190`).
Docstring `:171-173` nhấn: residual **phải là OUT-OF-FOLD** (rolling-origin), residual trong-mẫu **đánh giá thấp phần đuôi**.

**d) `fit_gp_tail:193`** — MLE generalized Pareto với `loc` cố định 0:
`genpareto.fit(exc, floc=0.0)` → `(ξ, β)`; **cần ≥ 5 vượt ngưỡng dương**, ngược lại `ValueError` (`:201-203`).

### 7.3. TailCalibrator — hiệu chỉnh đuôi kiểu conformal

**Vị trí:** `core/scenario/calibrate.py:135` — `TailCalibrator`; wrapper `_CalibratedMarginal:55`.

**Bài toán:** khoảng p10-p90 của model có thể hẹp/rộng có hệ thống theo **từng lớp cầu** (lai rai vs đều)
và theo **tầm dự báo**. Sửa bằng dịch chuyển **bất đối xứng** học từ lỗi đuôi ngoài mẫu.

**Công thức (docstring `calibrate.py:8-20`):**

```
e_lo = q10_pred − actual            # dương = khoảng dưới quá cao (bắt hụt phía dưới)
c_lo = Q_w( e_lo , 1 − u_lo )       # phân vị 0.90 có trọng số        (calibrate.py:201)
q_lo_new = q_lo − c_lo              ⇒ P(actual < q_lo_new) ≈ u_lo

e_hi = actual − q90_pred
c_hi = Q_w( e_hi , u_hi )           # phân vị 0.90 có trọng số        (calibrate.py:202)
q_hi_new = q_hi + c_hi              ⇒ P(actual > q_hi_new) ≈ 1 − u_hi
```

**`Q_w` — phân vị thực nghiệm CÓ TRỌNG SỐ (`_weighted_quantile:39`):** sắp xếp giá trị, cộng dồn trọng số,
lấy giá trị đầu tiên có `cumweight/total >= q`.

**Trí nhớ suy giảm (`update:168`):** mỗi lô cập nhật, trọng số cũ **nhân `decay`** rồi nối lô mới với trọng số 1
(`:188`) → lỗi gần đây quan trọng hơn. Cắt còn `max_points` **giá trị mới nhất** (`:189-192`).

**Áp `delta(u)` tuyến tính từng khúc (`_CalibratedMarginal:57-68, _delta:103`):**

```
body_lo = min( 0.5 , 2.5 · u_lo )              (calibrate.py:84)
body_hi = max( 0.5 , 1 − 2.5·(1 − u_hi) )      (calibrate.py:85)

u ≤ u_lo              : delta = −c_lo
u_lo → body_lo        : dốc tuyến tính −c_lo → 0
body_lo → body_hi     : delta = 0            ← THÂN phân phối KHÔNG bị đụng
body_hi → u_hi        : dốc tuyến tính 0 → +c_hi
u ≥ u_hi              : delta = +c_hi
```

**Sửa đơn điệu khi THU HẸP:** nếu `c_lo < 0` hoặc `c_hi < 0` thì `delta` có thể giảm → `ppf` mất đơn điệu.
Khi đó code dựng lưới `u` gồm `linspace(0.0005, 0.999, 1997)` + `[0.9992, 0.9995, 0.9999]` (`:90-95`),
tính giá trị, `np.maximum.accumulate` (**rearrangement**), và trả lời bằng `np.interp` trên lưới đã sửa (`:122`).
Ngoài mép lưới trên: tiếp tục theo **tốc độ tăng của đuôi gốc** (`:125-130`) — để đuôi GP không bị "phẳng lì" ở mép lưới.

**Hằng số THẬT (`calibrate.py:144-149`):** `u_lo = 0.10`, `u_hi = 0.90`, `decay = 0.98`, `max_points = 2000`.
Ràng buộc `0 < u_lo < u_hi < 1` (`:151-152`).

**Segment:** khoá là `(demand_class, horizon_bucket)` (`SegmentKey`, `:34`), nối bằng `"||"` (`:36`).
Bucket v1 = **`"h1-28"` duy nhất cho cả tầm 1..28 ngày** (`backtest_run.py:34`, `artifact.py:102` — hai chỗ ghi rõ
"phải khớp nhau"). `demand_class ∈ {intermittent, smooth}`.

**Nạp dữ liệu:** `backtest_run.py:390-394` gom mỗi ngày hold-out một cặp `(e_lo, e_hi)` từ đường lgbm OOF;
lưu vào `kv_state` key `tail_calibrator:<project_id>` (`backtest_run.py:452-483`).
Đọc: `forecast_run.py:957` `_load_tail_calibrator`.

**Đo được (2026-08-07):** `kv_state` có `tail_calibrator:` cho `demoshop`, `eval-fc`, `p1` với
`{"u_hi":0.9,"u_lo":0.1,"decay":0.98, segments: {"smooth||h1-28": {...}, "intermittent||h1-28": {...}}}` — khớp code.

### 7.4. Trích common factor từ OOF residual

**Vị trí:** `core/scenario/factors.py:34` — `extract_common_factors(residual_matrix, groups)`.

**Bài toán:** "ngày mưa cả sàn ế" — các SKU **không độc lập**. Nếu mô phỏng độc lập, tổng danh mục sẽ có
phương sai bị đánh giá thấp nghiêm trọng → tồn kho an toàn tính thiếu.

**Phân rã cộng tính, tuần tự (`factors.py:11-22, 63-82`):**

```
residual[sku, day]  =  calendar[day]  +  category[cat(sku)][day]  +  local[sku, day]

calendar[day]       = mean theo TẤT CẢ SKU của cột day                          (:63)
decal               = residual − calendar
category[c][day]    = mean theo các SKU thuộc c của decal                        (:70-71)
f_sku               = calendar + category[cat(sku)]
loading[sku]        = Cov(residual_sku, f) / Var(f)      (OLS slope, bias=True)  (:76-80)
                      = 0 nếu Var(f) == 0
local_std[sku]      = std( residual_sku − loading·f )                            (:81-82)
```

⚠ **Yêu cầu được viết ra (`factors.py:3-8`):** đầu vào **PHẢI là residual out-of-fold** từ rolling-origin backtest,
**KHÔNG phải in-sample** — residual in-sample bị co lại bởi chính phép fit, đánh giá thấp cả đuôi lẫn đồng biến động,
mà đó **chính xác là hai thứ copula sinh ra để bắt**.

⚠ **Nhưng đường sản phẩm hiện tại KHÔNG nạp OOF residual vào đây.** `artifact.py:225` `_fit_factor_payload` dựng
`matrix[i,:] = row − row.mean()` tức là **cầu thực trừ trung bình của chính nó**, không phải residual dự báo
(`artifact.py:245-248`). Docstring `artifact.py:47-50` gọi đúng tên: *"residual matrix = per-SKU demand minus its own mean"*.
→ **Khoảng cách giữa hợp đồng của module và cách gọi nó.** Đây là nợ kỹ thuật thật, không có W-ID gắn kèm trong code.

### 7.5. Sinh scenario — Gaussian factor copula

**Vị trí:** `core/scenario/generator.py:99` `ScenarioSet`; `_sku_block:153` là **nguồn sự thật duy nhất**.

```
u_eps      = RNG.uniform_block( scenarios, sku_id, days )                          (:165)
cal_std    = std( factor_model["calendar"] )                                        (_FactorSpec:59-60)
cat_std    = std( factor_model["category"][cat] )                                   (:64-65)
common_var = loading² · ( cal_std² + cat_std² )                                     (:166)

nếu common_var ≤ 0:
    u = u_eps                                # không có cú sốc chung → dùng thẳng uniform  (:170-172)
ngược lại:
    ε          = Φ⁻¹(u_eps)
    f          = cal_std · CalBlock[:, day] ( + cat_std · CatBlock[:, day] )         (:178-185)
    z          = loading · f  +  local_std · ε                                        (:186-188)
    σ_z        = sqrt( common_var + local_std² )                                      (:189)
    u          = Φ( z / σ_z )                                                          (:190)

sample = marginal.ppf( u )                                                            (:191)
```

**Vì sao chia `σ_z`:** docstring `generator.py:18-21` — chuẩn hoá `z` trước `Φ` để `u` **đúng Uniform(0,1) biên duyên**,
nên **copula CHỈ đóng góp tương quan, KHÔNG BAO GIỜ bóp méo marginal đã hiệu chỉnh**.

**Factor cũng phải tái lập được:** các cú sốc chung dùng **pseudo-SKU id dành riêng** `"__calendar__"` (`:37`)
và `"__cat_<name>__"` (`:40-41`) → draw của factor cũng đi qua đúng bộ đếm Philox như `eps`.

**Lười (lazy) — không vật chất hoá khối:** `ScenarioSet` **không** dựng sẵn cube `[S, SKU, D]`; mọi truy vấn tính lại
tất định (`:100-105`). `total(sku_ids)` cộng dồn từng SKU (`:209-216`), `window_total` (`artifact.py:562`) cũng vậy.

**Tổng LUÔN cộng THEO KỊCH BẢN** (`generator.py:211-212`, `artifact.py:566-567`): cộng theo ngày rồi theo SKU
**BÊN TRONG mỗi kịch bản**, *"không bao giờ là tổng của các phân vị"*.

**`quantile_view:218`:** `np.quantile` dọc trục kịch bản → **đơn điệu theo `probs` theo cấu trúc, không thể crossing**.

### 7.6. Artifact — fit, publish, load

**Vị trí:** `core/scenario/artifact.py:402` — `build_scenario_artifact(...)`.

**Chọn marginal mỗi SKU (`artifact.py:446-467`):**

| Nguồn | Điều kiện | Marginal |
|---|---|---|
| `grid_booster` | caller truyền `quantile_grids[sku]` (từ 9 booster LightGBM) | `QuantileGridMarginal` (+ GP tail nếu đủ điểm) |
| `history_empirical` | không có grid | `classify_series` → `intermittent_*` ⇒ `IntermittentMarginal`; `smooth`/`cold_start` ⇒ `EmpiricalResidualMarginal` |

**Fit từ lịch sử (`_fit_marginal_params:153`):**

```
intermittent:  p         = |{u > 0}| / N
               size_mean = mean( positive )
               k         = size_mean² / (var_pos − size_mean)      nếu var_pos > size_mean > 0
                         = 1e6 (Poisson-like)                       ngược lại                 (:163-166)
               k         = clip(k, 1e-3, 1e6)                                                  (:167)
smooth/cold :  base      = mean(units) ;  residuals = units − base                             (:177-181)
```

**GP tail (`_fit_grid_params:191`):** lấy `e_hi` dương từ segment `(demand_class, "h1-28")` của calibrator;
`≥ _GP_MIN_POINTS = 30` mới fit `genpareto` MLE → `tail="gp"` với `(ξ, β)`, ngược lại không gắn đuôi (`:208-213`).

**Factor payload (`_fit_factor_payload:225`):** ma trận trên các **ngày CHUNG của mọi SKU** (`:239-242`);
`< 2 SKU` hoặc `< _MIN_COMMON_DAYS = 7` ngày chung → **None** (SKU độc lập, `has_factors=0`).
DB forecast **không có category sản phẩm**, nên mọi SKU nằm **một nhóm `__all__`** (`:104`, `:249`, docstring `:50-52`).

**Giao thức publish CAS (`artifact.py:470-514`, docstring `:13-19`):**
1. ghi vào thư mục tạm anh em `<run_id>.tmp-<uuid>`;
2. tính sha256 từng file vào manifest;
3. **ghi `manifest.json` CUỐI CÙNG** — sự tồn tại của nó = artifact hợp lệ;
4. `os.rename` nguyên tử tmp → tên cuối; nếu tên cuối đã tồn tại (republish): `rename final → trash`,
   `rename tmp → final`, `rmtree(trash)`.
→ Người đọc **hoặc không thấy manifest (artifact không tồn tại), hoặc thấy artifact đầy đủ có checksum**.

**Load (`load_scenario_set:521`):** thiếu manifest → `FileNotFoundError`; sai checksum / thiếu file → `ValueError`.
API ánh xạ: `FileNotFoundError` → **404**, `ValueError` → **412 FAILED_PRECONDITION** (`main.py:1776-1787`).

**Hằng số THẬT:**

| Hằng | Giá trị | Vị trí |
|---|---|---|
| `MARGINALS_FORMAT` | 2 (format 1 = chỉ lịch sử; 2 = + grid/GP/calib) | `artifact.py:98` |
| `HORIZON_BUCKET_V1` | `"h1-28"` | `artifact.py:102` |
| `_POISSON_LIKE_K` | `1e6` | `:105` |
| `_MIN_COMMON_DAYS` | 7 | `:106` |
| `_GP_MIN_POINTS` | 30 | `:107` |
| `scenario_count` mặc định | 2000 | `:407`, `forecast_run.py:794` |
| `horizon_days` mặc định | 28 | `:408` |
| `generator_version` | `"sg-v1"` | `:410` |
| thư mục artifact | `MINIAI_ARTIFACT_DIR` hoặc `<repo>/data/artifacts` | `:113-135` |

⚠ **Sự cố đã trả giá (`artifact.py:118-124`):** `parents[5]` giả định layout repo
`<repo>/services/forecast/app/core/scenario/artifact.py`; **image Docker làm phẳng thành `/srv/app/core/scenario/artifact.py`
(chỉ 4 tổ tiên)** → `IndexError`. Mà `IndexError` là **lớp con của `LookupError`** → nó bị đường degrade của
`forecast:run` bắt nhầm thành *"missing demand history"* (sự cố live 2026-08-04). Bản vá: bắt `IndexError`, đi tìm mốc `app`.

### 7.7. API scenario

| Endpoint | Toán | Vị trí |
|---|---|---|
| `POST /v1/scenarios:build` | build + CAS publish, `run_id = "sc_" + uuid4[:12]` | `main.py:1807-1836` |
| `POST /v1/scenarios:lead-time-demand` | `LTD[s] = Σ_{d < lead+review} sample[s,·,d]` rồi lấy `np.quantile` theo `required_quantiles` | `main.py:1839-1892` |
| `POST /v1/scenarios:aggregate` | `window_total` → p10/p50/p90 + mean | `main.py:1895-1919` |
| `POST /v1/scenarios:probability` | `P = mean( totals >= threshold_units )` | `main.py:1922-1952` |

**Giới hạn THẬT (`main.py:1694-1696`):** `_SCENARIO_MAX_SKUS = 100`, `_SCENARIO_MAX_HORIZON = 90`,
`_SCENARIO_MAX_COUNT = 2000`. `lead_time_days + review_period_days` vượt `horizon_days` của artifact → **400 có hướng dẫn**
("rebuild với horizon lớn hơn", `main.py:1869-1878`). SKU không nằm trong artifact → **404 liệt kê tối đa 5 id thiếu**
(`_scenario_check_members:1791`).

**Bất đồng bộ (`forecast_run.py:796-800`):** build artifact chuyển vào job queue bền
(`job_type = "scenario_build"`, `job_id = "scn-<project>-<run_id>"`, `_enqueue_scenario_build:879`) vì
build đồng bộ tốn **~13 s cho 34 SKU**, chạm ngưỡng HTTP timeout. `FORECAST_SCENARIO_ASYNC=0` ép về đường cũ (`:803-804`).

---

## 8. KHOẢNG TIN CẬY & HIỆU CHỈNH

### 8.1. `sort_quantiles` — sửa quantile crossing

`libs/featurelib/quantiles.py:37`: `sorted((p10,p50,p90))`. Dùng ở `router.py:52`, `_apply_promo_uplift`
(`forecast_run.py:327`), `apply_calibration` (`backtest.py:234`), `insights._cdf_knots` (`:146`).
**Điểm yếu:** sắp xếp có thể **hoán đổi vai trò** của p10 và p50 (xem §3.2), khác hẳn với `cummax` (giữ nguyên thứ tự,
chỉ nâng giá trị) dùng ở `predict_quantile_grid` (`global_model.py:253`) và `QuantileGridMarginal` (`marginal.py:125`).

### 8.2. `calibration_factor` — coverage thực nghiệm → `width_factor`

**Vị trí:** `core/backtest.py:206`.

```
nếu | coverage_emp − target | ≤ tol   →  w = 1.0                       (backtest.py:212-213)
cov      = clip( coverage_emp , 0.02 , 0.998 )                          (:215)
z_target = Φ⁻¹( 0.9 )                       ≈ 1.28155                   (:217)
z_emp    = Φ⁻¹( 0.5 + cov/2 )                                            (:219)
w        = clip( z_target / z_emp , 0.5 , 3.0 )                          (:220-221)
```

**Hằng số THẬT:** `target = 0.8`, `tol = 0.05` (`:208-209`), kẹp coverage `[0.02, 0.998]`, kẹp `w` `[0.5, 3.0]`.

**Trực giác:** giả thiết phân phối chuẩn. Khoảng p10-p90 danh nghĩa phủ 80% hai phía ⇒ nửa-độ-rộng là `z_target·σ`.
Nếu thực tế chỉ phủ `cov`, thì nửa-độ-rộng hiện tại tương ứng `z_emp·σ`. Nhân với `z_target/z_emp` để đưa về đúng 80%.
⚠ Comment `backtest.py:216` ghi "z for 90% two-sided interval" — **nhãn sai**, `Φ⁻¹(0.9)` là z của khoảng **hai phía 80%**.
Toán thì đúng; chỉ chú thích lệch.

### 8.3. `apply_calibration` — nong/nén quanh p50

**Vị trí:** `core/backtest.py:224`.

```
p10' = p50 − w · (p50 − p10)
p90' = p50 + w · (p90 − p50)
sort_quantiles(p10', p50, p90')  ;  kẹp cả ba ≥ 0                        (backtest.py:231-237)
```

`w < 1` = nén (khoảng đang quá rộng), `w > 1` = nong (quá hẹp). **p50 KHÔNG đổi.**

**Số đo THẬT — phân bố `width_factor` (kv_state, 2026-08-07):**

| project | w = 0.500 (chạm sàn) | 0.647 | 0.768 | 0.875 | 1.000 | 1.200 | 1.463 | 1.619 | 2.010 | 2.264 |
|---|---|---|---|---|---|---|---|---|---|---|
| demoshop | **34** | 11 | 12 | 34 | 5 | 11 | — | 5 | — | 1 |
| simworld1 | **25** | 5 | 2 | 14 | 6 | 4 | 1 | 1 | 1 | 1 |

→ **~1/3 số SKU chạm SÀN 0.5**, tức là coverage thực nghiệm cao hơn 0.8 rất nhiều (khoảng quá rộng) và đã bị nén hết mức
cho phép mà vẫn chưa đủ. Sàn 0.5 đang **cắn thật**, không phải phòng hờ.

### 8.4. Tổng hợp quantile — triangular Monte Carlo

**Vị trí:** `libs/featurelib/quantiles.py:43` — `aggregate_quantiles(items, n=2000, rng_seed=42)`.

```
với mỗi (p10, p50, p90):
    nếu p10 == p90  →  total += p50                                       (quantiles.py:60-61)
    ngược lại       →  total += [ random.triangular(low=p10, high=p90, mode=p50) ]×n   (:64-67)
trả về  P10(total), P50(total), P90(total)                                (:70-73)
```

`random.Random(42)` (`:57`) — tất định.

⚠ **Giả thiết mạnh, được nói thẳng ở `main.py:685-687`:** triangular MC coi `p10/p90` là **min/max CỨNG mỗi ngày**.
Với ngày intermittent lệch mạnh thì sai hình dạng — nên `forecast:query` **thay bằng NBD horizon-sim cho họ Croston**.

### 8.5. NBD horizon-sim tại đường đọc

**Vị trí:** `main.py:690-709`.

```
nếu model_used ∈ {"croston", "sba", "croston_auto"}:
    day_means = [ d.p50 for d in daily ]      # ĐÃ mang hệ số lịch + promo
    nếu không có mean nào > 0 → raise → rơi về triangular MC
    hist_units = adjusted_units toàn bộ lịch sử SKU
    (p10,p50,p90) = quantiles_nbd(0.0, len(daily), hist_units, n=2000, seed=42, day_means=day_means)
    totals_method = "nbd_horizon_sim_2000"
ngược lại:
    totals = aggregate_quantiles(...)          ;  totals_method = "triangular_mc_2000"
```

Trường `totals_method` **đi ra tận response** (`main.py:777`) — người dùng biết con số tổng được sinh bằng cách nào.

### 8.6. Ngoại suy đuôi p95 / p99

**Vị trí:** `libs/featurelib/quantiles.py:80` — `extend_quantiles(p10, p50, p90, quantiles)`.

```
_Z = { 0.90: 1.2816 , 0.95: 1.6449 , 0.99: 2.3263 }                       (quantiles.py:77)

nếu p90 > p50 > 0   :  σ_log = ln(p90/p50) / z_0.90  ;  val = p50 · exp( z_q · σ_log )   # đuôi LOGNORMAL
nếu p90 > p50 (p50≤0):  σ     = (p90 − p50) / z_0.90 ;  val = p50 + z_q · σ              # đuôi CHUẨN
ngược lại            :  val = p90
out[ f"p{round(q·100)}" ] = max( val , p90 )       # ép đơn điệu                          (:106)
```

API chỉ cho phép `quantiles ⊆ {0.95, 0.99}` (`main.py:598-608`); response gắn
`quantile_method = "lognormal_tail_extrapolation"` (`main.py:741`).

### 8.7. `quantile_interp` — nội suy phân vị bất kỳ từ 3 điểm

**Vị trí:** `libs/featurelib/quantiles.py:12`.

```
q < 0.1   : val = p10 + ((p50−p10)/0.4)·(q − 0.1)          , kẹp ≥ 0
0.1≤q≤0.5 : val = p10 + ((q−0.1)/0.4)·(p50 − p10)
0.5<q≤0.9 : val = p50 + ((q−0.5)/0.4)·(p90 − p50)
q > 0.9   : val = p90 + ((p90−p50)/0.4)·(q − 0.9)          , kẹp ≤ p90 + 2·(p90−p50)
```

Dùng bởi `insights.cdf_points` (p25/p75) và làm chuẩn cho `insights._cdf_knots`.

---

## 9. INSIGHTS (đọc — biến dữ liệu sẵn có thành câu trả lời nghiệp vụ)

**Vị trí:** `core/insights.py`; route `main.py:1550`. Sáu `kind` (`main.py:1492-1499`).

| kind | Toán | Vị trí |
|---|---|---|
| `accuracy_sku` | gộp `backtest_results` per-SKU + cặp (dự báo quá khứ, thực tế) → MAPE / bias / coverage | `insights.py:276`, `accuracy_from_pairs:245` |
| `top_movers` | xếp hạng theo `SUM(p50)` của run mới nhất; fallback = mean 28 ngày × horizon | `:353` |
| `group_breakdown` | tổng p50 mỗi SKU + `share_pct` | `:440` |
| `seasonality` | `weekday_profile:64` + `monthly_profile:105` + liệt kê `calendar_events` | `:485` |
| `sell_through_prob` | `prob_ge:161` trên CDF tuyến tính từng khúc | `:559` |
| `promo_uplift_learned` | `promo_uplift_observed:202` (chuẩn hoá theo weekday) | `:593` |

**Công thức chính:**

```
weekday_profile:   factor[wd] = mean(units | wd) / mean(units)                    (insights.py:88)
                   weekend_uplift_pct = ( mean(T7,CN) / mean(T2..T6) − 1 ) · 100   (:93-97)
monthly_profile:   factor[m]  = mean daily units tháng m / mean daily units toàn bộ (:126)

_cdf_knots:  s1 = (p50−p10)/0.4 ; s2 = (p90−p50)/0.4
             knots = [ (max(0, p10−0.1·s1), 0.0), (p10,0.1), (p50,0.5), (p90,0.9), (p90+0.1·s2, 1.0) ]   (:149-151)
prob_ge:     nội suy tuyến tính CDF tại q, trả 1 − F(q), làm tròn 4 chữ số         (:181-187)

accuracy_from_pairs:  MAPE% = mean( |p50−actual| / actual ) · 100   (chỉ ngày actual > 0)   (:260-266)
                      bias% = ( Σp50 − Σactual ) / Σactual · 100     ( > 0 = dự báo THỪA)    (:267)
                      coverage = #{ p10 ≤ actual ≤ p90 } / n                                 (:268)

promo_uplift_observed: chuẩn hoá units theo factor weekday học TỪ NGÀY KHÔNG PROMO,
                       rồi uplift = promo_mean_norm / base_mean_norm                          (:220-236)
```

**Lý do chuẩn hoá weekday (`insights.py:206-209`):** shop chỉ chạy sale cuối tuần sẽ được cộng nhầm **hiệu ứng cuối tuần**
vào công của promo.

**Ngưỡng THẬT (`insights.py:46-51`):** `MIN_WEEKDAY_DAYS = 14`, `MIN_MONTHLY_DAYS = 60`, `MIN_PROMO_DAYS = 5`,
`MIN_ACCURACY_POINTS = 5`, `MIN_MONTH_DAYS_FOR_BEST = 7`, `TOP_MOVERS_FALLBACK_CAP = 50`.

**Trung thực được ghi ra:** `top_movers` gắn `ranking_note: "ranked by direct sum of daily p50 (ordering only)"`
(`insights.py:396`) — cộng thẳng p50 **chỉ đúng để XẾP HẠNG**, không phải tổng xác suất (dùng `forecast:aggregate`).

**SQL chống rò rỉ (`insights.py:314-333`):** cặp dự báo-vs-thực tế lấy `DISTINCT ON (horizon_day)` với
`created_at::date <= horizon_day` — dự báo không được tạo SAU ngày nó dự báo.

---

## 10. BẢNG TỔNG HỢP: THUẬT TOÁN × KHI NÀO DÙNG × THAM SỐ × CÁCH ĐO

| Thuật toán | Khi nào được chọn | Tham số chính (giá trị THẬT) | Cách đo |
|---|---|---|---|
| `cold_start` | `n < 14` ngày, hoặc `< 2` ngày có bán | p10=0.5·p50, p90=1.5·p50; p50=0 ⇒ (0,0,1) | `pytest tests/forecast/` · `backtest_series` bỏ qua (`n<14`) |
| `seasonal_naive` | smooth & `n ≤ 56`; **luôn** là baseline backtest; fallback cuối của `choose_model` | m=7, quantile phần dư theo `i mod 7` | `grade_forecast.py` (mase = NULL theo thiết kế) |
| `autoets_theta_ensemble` | smooth & `n > 56`; hoặc `model_choice` chấm ra | `season_length=7`, ensemble 50/50, bootstrap 500 draw seed 42 | `backtest_results.model='autoets_theta_ensemble'` |
| `croston` / `sba` / `croston_auto` | ADI > 1.32 (SBA khi CV² > 0.49) | `α=0.1`, SBA `×(1−α/2)`, NBD n=500 seed=42 | `backtest_results.model='croston_auto'`; totals `nbd_horizon_sim_2000` |
| `adida` | ADI > 1.32 **và** backtest chấm ra | statsforecast ADIDA, `MIN_HISTORY=14`, bootstrap 500/seed 42 | `backtest_results.model='adida'` |
| `imapa` | như trên | statsforecast IMAPA, cùng bộ hằng | `backtest_results.model='imapa'` |
| `similar_item_transfer` | **ghi đè** khi `n < 14` và có hàng xóm ≥ 28 ngày | k=5 hàng xóm, timeout 1.5 s, nới ×1.3, scale fallback 0.5 | `forecasts.model_used='similar_item_transfer'` |
| `cold_start_analog` | 0 dòng `demand_daily`, chỉ ở `forecast:query` | `MIN_ANALOG_DAYS=7`, `BAND_WIDEN=0.2`, k=5, timeout 3 s | response có `analog_of`, `model_used='cold_start_analog'` |
| `lgbm_global` | tenant ≥ 50 SKU & ≥ 120 ngày **và** backtest chấm ra | 9 alpha, lr 0.05, leaves 63, min_leaf 50, 400 vòng | `backtest_results.model='lgbm_global'`; probe "lgbm_global available" |
| Hierarchical reconcile | `FORECAST_RECONCILE=1` tại `forecast:aggregate` | T<8 ⇒ wls_struct, 8≤T<30 ⇒ wls_var, T≥30 ⇒ mint_shrink | `coherence_error_before/after` trong response |
| Scenario fabric | có artifact đã publish | 2000 kịch bản, horizon 28, Philox `sg-v1` | `POST /v1/scenarios:*`; manifest sha256 |
| TailCalibrator | có `kv_state tail_calibrator:<proj>` | u_lo 0.10, u_hi 0.90, decay 0.98, max 2000 điểm | `marginals.npz` cột `calib` (format 2) |
| `calibration_factor` | luôn, sau backtest | target 0.8, tol 0.05, w ∈ [0.5, 3.0] | `kv_state.width_factor`; response `calibration` |

**Lệnh kiểm chứng thật:**

```bash
cd /home/hai-soft/projects/icpp/mecom/project

# 1. Unit test toàn bộ (thư mục tests/forecast/)
make test                       # = pytest -q          (Makefile:12)

# 2. Eval BT03: seed 60 SKU tổng hợp (45 smooth + 15 intermittent, 130 ngày, seed 42),
#    chạy run_backtest_once, đo coverage sau hiệu chỉnh.  eval/forecast_eval.py:22-25
make eval-forecast              # = python3 eval/forecast_eval.py     (Makefile:34-35)

# 3. Chấm điểm SW-3 (coverage 0.80±0.05 + MASE < 1.0) trên dữ liệu tenant thật
.venv/bin/python scripts/grade_forecast.py --project demoshop
.venv/bin/python scripts/grade_forecast.py --project simworld1

# 4. Chuỗi API sống (probe: forecast:run 202+job_id, forecast:run job done, forecast:query,
#    forecast:aggregate, forecast:accuracy, forecast quantiles p99, forecast:promo-preview,
#    aggregate by category, lgbm_global available, promo covariate)
make check-apis PROJECT=<tenant>    # = .venv/bin/python -m seedtool check   (Makefile:75-76)

# 5. Bộ 100 câu hỏi nghiệp vụ BT03
python3 eval/bt03_100q.py           # (Makefile:47-50, cần key env)

# 6. Đọc thẳng số đo trong DB
docker exec miniai-postgres psql -U miniai -d miniai_forecast -c "
  SELECT project_id, model, count(*) n,
         round(avg(coverage_p10_p90)::numeric,3) cov,
         round(avg(mase)::numeric,3) mase,
         round(avg(pinball_q50)::numeric,3) pin50
  FROM backtest_results GROUP BY 1,2 ORDER BY 1,2;"
```

`scripts/grade_newsvendor.py` là grader của **BT02 (decision service)** — kiểm tra công thức critical-ratio,
mức phục vụ đạt được và số học của `qty` đặt hàng, đối chiếu với `simulator/world.py`. Nó **tiêu thụ** đầu ra
BT03 gián tiếp (dự báo → tồn kho) nhưng **không đo thuật toán dự báo nào**; không thuộc phạm vi tài liệu này.

---

## 11. SỐ ĐO THẬT (đo 2026-08-07, Postgres `localhost:16024/miniai_forecast`)

### 11.0. BASELINE ĐANG ĐƯỢC GHIM (DB tri thức `facts`)

| fact key | giá trị | Ghi chú |
|---|---|---|
| `eval.forecast.mase.baseline` | **0.827** | ghim 2026-08-06 sau khi rerun backtest simworld1 với định nghĩa MASE mới |
| `eval.forecast.coverage.baseline` | **0.815** | cùng lần ghim |
| `eval.baseline.tolerance` | 0.10 | dung sai chung của gate eval |

**Lịch sử baseline (quan trọng để không đọc nhầm số cũ):**
- baseline MASE **0.936** (đo 2026-08-04, sau fix NBD-horizon) đã **HẾT HIỆU LỰC** vì định nghĩa mẫu số MASE thay đổi
  (W-MASE-UNIFY). So 0.936 với 0.827 là **so hai cái thước khác nhau**.
- Lần rerun ghim baseline mới: `backtest simworld1 3754 giây / 180 SKU / 762 dòng`, kết quả
  `overall coverage 0.815, mase 0.827; intermittent .811/.848; smooth .827/.756`.
- Số đo mới nhất được ghi trong DB (chốt chiến dịch 2026-08-06T18:30): `forecast MASE 0.753 < 0.827-baseline`.

**Đối chiếu với số tôi đo lại hôm nay (§11.3):** demoshop `cov 0.830 / mase 0.857`, simworld1 `cov 0.814 / mase 0.823`.
simworld1 khớp baseline ghim (0.815/0.827) trong sai số nhỏ ⇒ **baseline vẫn còn hiệu lực, không bị trôi**.

**Một số đo lịch sử khác trong DB (bối cảnh, không phải baseline hiện hành):**
- `arch.forecast-calibration-coverage`: *"coverage thô 91.6% → sau calibration 76.8% (đích [70,90]);
  width_factor = z(0.9)/z((1+coverage)/2)"* — xác nhận công thức §8.2 và cho thấy hiệu chỉnh **nén** là chuyện thường.
- M15 `eval-fc bt_2026-08-03`: `pinball_q50 croston .502 < adida .514 < imapa .522 < seasonal_naive .581`;
  chọn per-SKU: `croston_auto 4, imapa 1`.

### 11.1. Quy mô dữ liệu

| project | #SKU | #ngày phân biệt | max ngày/SKU | cửa sổ | đủ bar lgbm (≥50 SKU & ≥120 ngày)? |
|---|---|---|---|---|---|
| `demoshop` | 125 | 128 | 128 | 2026-04-01 → 2026-08-06 | **CÓ** |
| `simworld1` | 60 | 433 | 433 | 2025-05-31 → 2026-08-06 | **CÓ** |
| `p1` | 1080 | 203 | 203 | 2026-01-16 → 2026-08-06 | CÓ (nhưng xem §11.4) |
| `eval-fc` | 60 | 130 | 130 | 2026-03-29 → 2026-08-05 | **CÓ** |

### 11.2. `backtest_results` — coverage / MASE / pinball trung bình theo model

| project | model | n dòng | coverage | MASE | pinball_q50 |
|---|---|---|---|---|---|
| demoshop | `adida` | 824 | 0.905 | 0.896 | 0.366 |
| demoshop | `autoets_theta_ensemble` | 645 | 0.888 | 0.811 | 0.702 |
| demoshop | `croston_auto` | 824 | 0.938 | 0.905 | 0.369 |
| demoshop | `imapa` | 824 | 0.905 | 0.897 | 0.367 |
| demoshop | **`lgbm_global`** | 1017 | **0.784** | **0.782** | 0.486 |
| demoshop | `seasonal_naive` | 1469 | 0.692 | (NULL — baseline) | 0.666 |
| simworld1 | `adida` | 1443 | 0.765 | 0.907 | 0.985 |
| simworld1 | `autoets_theta_ensemble` | 897 | 0.892 | 0.789 | 1.002 |
| simworld1 | `croston_auto` | 1443 | 0.904 | 0.894 | 0.972 |
| simworld1 | `imapa` | 1443 | 0.764 | 0.908 | 0.993 |
| simworld1 | **`lgbm_global`** | 2340 | **0.789** | **0.800** | **0.912** |
| simworld1 | `seasonal_naive` | 2340 | 0.809 | (NULL) | 1.176 |
| eval-fc | `lgbm_global` | 180 | 0.710 | 0.711 | 0.989 |
| eval-fc | `autoets_theta_ensemble` | 135 | 0.893 | 0.769 | 1.237 |
| eval-fc | `seasonal_naive` | 180 | 0.747 | (NULL) | 1.321 |

**Đọc ra:** `lgbm_global` là model có **coverage GẦN 0.80 nhất** và **MASE thấp nhất** trên cả demoshop lẫn simworld1 —
đúng như kỳ vọng của cross-learning. `croston_auto` có coverage **cao quá** (0.904-0.938) ⇒ khoảng quá rộng ⇒
chính là nhóm bị `width_factor` nén xuống sàn 0.5 (§8.3).

### 11.3. Grader SW-3 (`scripts/grade_forecast.py`) — chạy thật hôm nay

```
=== SW-3 forecast grader — project=demoshop (5603 SKU-backtests) ===
mase_def = classic-insample-snaive7  (unified across backtest_series + lgbm_global, W-MASE-UNIFY)
  intermittent   coverage[mean=0.860 median=0.905 n=3872]  mase[mean=0.877 median=0.870 n=3048]
  smooth         coverage[mean=0.762 median=0.810 n=1731]  mase[mean=0.800 median=0.760 n=1086]
  mase NULL excluded: 1469 seasonal_naive baseline rows + 0 other-model rows
  coverage_p10_p90 = 0.830  (target 0.8±0.05)  OK
  mase             = 0.857  (target < 1.0)     OK
PASS  forecast calibration/skill.

=== SW-3 forecast grader — project=simworld1 (4572 SKU-backtests) ===
  intermittent   coverage[mean=0.814 median=0.857 n=3330]  mase[mean=0.836 median=0.842 n=2664]
  smooth         coverage[mean=0.816 median=0.857 n=1242]  mase[mean=0.783 median=0.677 n=828]
  mase NULL excluded: 1080 seasonal_naive baseline rows + 0 other-model rows
  coverage_p10_p90 = 0.814  (target 0.8±0.05)  OK
  mase             = 0.823  (target < 1.0)     OK
PASS  forecast calibration/skill.
```

Ngưỡng nằm trong `scripts/grade_forecast.py:29-30`: `COV_TARGET=0.80`, `COV_TOL=0.05`, `MASE_TARGET=1.0`.

### 11.4. Phân bố model đang phục vụ

`kv_state.model_choice` (backtest chấm) vs `forecasts.model_used` (thực tế đã dùng khi ghi):

| project | `model_choice` (SKU) | `forecasts.model_used` (SKU) |
|---|---|---|
| demoshop | lgbm_global 67 · autoets 21 · croston_auto 9 · imapa 6 · adida 5 · seasonal_naive 5 | lgbm_global 67 · autoets 50 · croston_auto 31 · seasonal_naive 22 · adida 17 · imapa 14 · **cold_start 11** |
| simworld1 | lgbm_global 36 · autoets 10 · croston_auto 9 · seasonal_naive 3 · imapa 2 | (60 SKU) |
| eval-fc | lgbm_global 33 · autoets 21 · seasonal_naive 4 · imapa 1 · croston_auto 1 | autoets 45 · lgbm_global 40 · seasonal_naive 18 · croston_auto 4 · adida 3 · imapa 2 |
| p1 | **seasonal_naive 48** (chỉ 48/1080 SKU có model_choice) | **cold_start 629** · seasonal_naive 450 · croston 24 |
| demo | autoets 50 · croston_auto 5 · seasonal_naive 5 | autoets 59 · seasonal_naive 31 · croston_auto 5 · adida 2 · cold_start 1 · similar_item_transfer 1 |

**⚠ Bất thường ĐO ĐƯỢC ở `p1`:** `backtest_results` cho `p1` có `coverage = 1.000` và `mase = 0.000` với
`pinball_q50 = 0.000` cho `croston_auto`/`lgbm_global`/`seasonal_naive`, và `coverage = 0.000` cho `adida`/`imapa`;
2220/7005 dòng có `mase IS NULL`. 629/1080 SKU của `p1` phục vụ bằng `cold_start`.
Đây là **chữ ký của chuỗi toàn 0** — mọi phân vị bằng 0, actual bằng 0 nên "phủ" 100%, MAE = 0.
`seasonal_naive_scale` trả `None` cho chuỗi hằng nên MASE thành NULL (đúng thiết kế, `backtest.py:39`),
nhưng `mase = 0.000` xuất hiện được nghĩa là có nhánh chuỗi **không hằng nhưng lỗi = 0**.
→ **Số của `p1` KHÔNG dùng để kết luận chất lượng model được.** Đây là đúng tinh thần "vòng-0 phép đo": số vô nghĩa ⇒ vứt.
`grade_forecast.py` **không có bộ lọc chuỗi suy biến** — chạy nó trên `p1` sẽ ra "PASS" giả.

### 11.5. Cảnh báo về "kiểu ngày" trong bảng `demand_daily` của `p1`

`p1` có 1080 SKU nhưng chỉ 48 SKU có `model_choice` — vì `backtest_run.py:136` bỏ qua SKU có `len(series) < 42`,
và 629 SKU phục vụ bằng `cold_start` tức `< 14` ngày dữ liệu thực (hoặc `< 2` ngày có bán).

---

## 12. BẤT BIẾN & CẠM BẪY — DANH SÁCH ĐI KÈM MÃ NGUỒN

| # | Bất biến / bài học | Nơi thực thi |
|---|---|---|
| 1 | `p10 ≤ p50 ≤ p90` và cả ba `≥ 0` ở MỌI đầu ra | `router.py:49-56`, `backtest.py:233-237`, `marginal.py` (mọi `ppf` kẹp ≥0), `global_model.py:230,253` |
| 2 | Tổng **không bao giờ** là tổng của phân vị — luôn cộng theo kịch bản | `quantiles.py:43`, `croston.py:100-104`, `generator.py:211-212`, `artifact.py:566-567` |
| 3 | Promo áp **đúng một lần**: deflate lịch sử ⇄ inflate tương lai; lgbm miễn | `forecast_run.py:435, 306, 1304, 1612` |
| 4 | Lịch áp **đúng một lần**: nhân lúc đọc cho model thống kê, `cal_factor` cho lgbm | `main.py:673, 1447`, `global_model.py:56-60` |
| 5 | `model_used != model_requested` **LUÔN** được giải thích | `forecast_run.py:692-704, 1534-1544, 1600-1605` |
| 6 | Backtest không rò rỉ: MASE scale chỉ từ `train`; residual chỉ từ run tạo TRƯỚC ngày | `backtest.py:120-127`, `store/forecasts.py:231`, `insights.py:321` |
| 7 | Một định nghĩa MASE duy nhất cho cả hai đường (W-MASE-UNIFY) | `backtest.py:21` (dùng bởi `backtest.py:127` và `backtest_run.py:367`), ghi vào `grade_forecast.py:31-36` |
| 8 | MASE của chuỗi hằng = `None`, **không phải 0.0** | `backtest.py:39`, `:172-174` |
| 9 | Reconcile **không bao giờ 5xx**; tự kiểm chứng coherence rồi mới tin | `hier_reconcile.py:317-340` |
| 10 | Scenario: manifest ghi CUỐI = hợp đồng "artifact tồn tại" | `artifact.py:499-511` |
| 11 | Scenario: cùng `(run_id, generator_version)` ⇒ cùng thế giới, bit-exact | `rng.py:3-12, 108-110` |
| 12 | Hiệu chỉnh đuôi **TRƯỚC** khi lấy mẫu (thứ tự CỨNG) | `calibrate.py:3-5`, `generator.py:5-12`, `artifact.py:41-45` |
| 13 | LightGBM **không bao giờ** train trên event loop | `forecast_run.py:591-598`, `backtest_run.py:272-281` |
| 14 | Khoá advisory forecast là PER-PROJECT | `forecast_run.py:1114-1116` |
| 15 | Kết quả âm (`bundle=None`) cũng được cache | `model_cache.py:93-101, 220` |

---

## 13. BA PHÁT HIỆN VỀ CODE (ngoài phần bài học đã có comment)

### 13.1. `libs/featurelib/data_quality.py` — cổng chất lượng KHÔNG ĐƯỢC NỐI VÀO ĐÂU

Docstring `data_quality.py:3` viết rõ: *"The checks are used as a gate before training a global model (spec §6.4)"*.
`grep` toàn repo: `check_data_quality` chỉ xuất hiện ở `tests/forecast/test_features_quality.py`.
**Không một dòng nào trong `train_global`, `_train_global_locked`, hay `backtest_run` gọi nó.**
Nghĩa là: bar `min_rows=100`, `max_stockout_rate=0.5`, `null_rate < 0.05`, `adjusted_units >= 0`
**không hề chặn bất cứ lần train nào**. Tenant có 60% ngày stockout vẫn train bình thường.
Cùng dạng: `libs/featurelib/drift.py` (PSI) chỉ được `services/smartsearch/app/jobs/drift.py` dùng —
**BT03 không có giám sát drift feature nào**.

### 13.2. `factors.extract_common_factors` được gọi SAI so với hợp đồng của chính nó

Docstring `factors.py:3-8` **in hoa** yêu cầu: *"Input MUST be the out-of-fold residual matrix from the rolling-origin
backtest ... NOT in-sample residuals"*, kèm lý do (residual in-sample bị co, đánh giá thấp đuôi và đồng biến động).
Nhưng caller duy nhất là `artifact._fit_factor_payload` (`artifact.py:245-250`) truyền vào
`row − row.mean()` — tức là **cầu thực đã trừ trung bình của chính nó**, không phải residual của bất kỳ dự báo nào.
Hệ quả: `loadings` và `local_std` mô tả **đồng biến động của MỨC CẦU**, không phải của **SAI SỐ DỰ BÁO**.
Copula do đó bơm tương quan sai đối tượng vào scenario. Docstring `artifact.py:47-50` có nói đúng mình đang làm gì,
nhưng **không nhắc rằng nó vi phạm hợp đồng của module bên dưới** — người sửa sau rất dễ đọc `factors.py` và tin nhầm.
Không có W-ID nào gắn cho khoản nợ này trong code.

### 13.3. Kẹp phân vị BẤT NHẤT giữa `baseline.py` và `intermittent_sf.py`

`intermittent_sf.py:75` kẹp `p10 = max(0.0, min(p10, p50))` — bảo vệ điểm dự báo.
`baseline.py:65` (seasonal_naive) và `baseline.py:127` (ets_theta) **chỉ** kẹp `p10 = max(0.0, p10)`.
Với chuỗi tăng mạnh, phân vị 10 của phần dư tuần là **dương** ⇒ `p10 > p50` ⇒ `sort_quantiles`
(`router.py:52`) **hoán vị**, và giá trị trả ra ở ô "p50" thực chất là p10 cũ.
Điểm dự báo bị thay âm thầm, không log, không metric. Ba nơi khác trong repo (`global_model.py:253`,
`marginal.py:125`, `calibrate.py:101`) đều chọn `cummax` — phép sửa **giữ nguyên vai trò**, chỉ nâng giá trị.
Nên chuẩn hoá về một trong hai, không để hai triết lý cùng tồn tại.

**Bổ sung (mức thấp hơn):**
- `baseline.py:104` truyền `level=[10,90]` vào `sf.forecast` nhưng **không đọc** cột `-lo-/-hi-` nào (`:108` chỉ là comment)
  → statsforecast tính khoảng tin cậy rồi vứt đi. Lãng phí thuần.
- `model_cache.py:37` còn ghi *"demoshop (18 SKU)"*; **đo được hôm nay demoshop có 125 SKU và 67 SKU đang chạy `lgbm_global`**
  → comment đã lỗi thời, dễ làm người đọc kết luận sai về đường degrade.
- `estimate_uplift_k` chạm trần `K_MAX = 4.0` ở tenant `seedtest` mà **không log/metric nào báo** (`promo_uplift.py:126`).
- `libs/featurelib/holidays_vn.py` là **code chết** trong đường sản phẩm (§5.3) nhưng vẫn mang bộ hằng số trông rất "đang chạy".
- `run_backtest_once` dùng **khoá advisory TOÀN CỤC** `"backtest_run"` (`backtest_run.py:96-98`) trong khi `run_forecast_batch`
  đã chuyển sang per-project (`forecast_run.py:1115`) sau khi đo được 59 s xếp hàng — cùng một bệnh, chỉ mới chữa một nửa.
- `run_backtest_once` gọi `list_products_with_demand(pool)` **không truyền `project_id`** rồi lọc bằng Python
  (`backtest_run.py:100-102`) — đúng cái pattern mà `store/forecasts.py:44-56` đã ghi là W-TENANT-SCOPE-FC và đã sửa cho forecast.
