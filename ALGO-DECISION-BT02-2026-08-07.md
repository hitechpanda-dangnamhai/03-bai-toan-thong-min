# BT02 — DECISION: TOÀN BỘ THUẬT TOÁN TỐI ƯU QUYẾT ĐỊNH

> **Phạm vi**: mọi thuật toán đang CHẠY trong `services/decision/` của project miniAI (mecom).
> **Nguồn**: đọc code thật tại commit `b63938d` (2026-08-06) + đo DB live `miniai_decision` ngày 2026-08-07.
> **Quy ước**: mọi trích dẫn dạng `file.py:dòng` là đường dẫn tương đối từ
> `/home/hai-soft/projects/icpp/mecom/project/`.
> Chỗ nào code không nói rõ, tài liệu ghi **CHƯA CHẮC** thay vì đoán (xem §14).

## 0. Quy ước đơn vị (đọc trước, sai đơn vị là sai hết)

| Đại lượng | Đơn vị trong code | Ghi chú |
|---|---|---|
| `price`, `ewma_cost`, `suggested_price`, `bundle_price` | **minor units VND** (đồng, số nguyên) | `price_state.current_price`, `cost_state.ewma_cost` |
| `expected_value.amount` | VND / **30 ngày** (`per="month"`) | trừ `slow_mover_alert` dùng `basis="capital_locked"` |
| `units_30d` | số đơn vị bán trong 30 ngày | `SUM(sales_daily.units)` |
| `eps` (elasticity) | không thứ nguyên, **âm** | kẹp `[-5.0, -0.2]` |
| `eps_sd` | cùng thang với `eps` | = OLS standard error của hệ số giá |
| `lead_time_days`, `review` | ngày | mặc định 7 / 7 |
| `holding_rate_annual` | tỉ lệ/năm | mặc định **0.25** |
| `markdown_pct`, `discount_pct`, `max_discount_pct` | **phần trăm** (10.0 = 10%) | không phải phân số |
| `confidence` | [0, 1] | rubric 0.9 / 0.7 / 0.5 |
| bandit `mu`, `sigma` | VND lợi nhuận / **ngày** | `price_bandit_state` |
| gate `reward` | đơn vị reward khai báo trước, kẹp `[reward_lo, reward_hi]` | mặc định `[0, 1]` |

---

## 1. BẢN ĐỒ TỔNG

### 1.1 Sơ đồ luồng

```
                       ┌──────────────────────────────────────────────┐
  POST /v1/events:ingest ──►  raw_events  (cost.recorded · price.changed ·
                       │                   stock.level · purchase.completed ·
                       │                   order.returned · competitor.price ·
                       │                   promo.scheduled)
                       └──────────────────┬───────────────────────────┘
                                          │ state_rollup.py (watermark kv_state,
                                          │  mỗi 300s, LIMIT 5000 event/lượt)
                                          ▼
   ┌───────────────────── TẦNG TRẠNG THÁI (§2) ─────────────────────────┐
   │ cost_state(ewma_cost, last3_avg, prev3_avg)   cost_ledger(unit_cost)│
   │ price_state(current_price)                    price_history(price,  │
   │ stock_state(on_hand_qty)                       changed_at)          │
   │ sales_daily(day, units, revenue)   competitor_price_state           │
   └───────────────────────────┬─────────────────────────────────────────┘
                               │
      ┌────────────────────────▼─────────────────────────┐
      │  refresh_all_elasticity  (§3)                     │
      │  pass 1: OLS log-log daily/SKU (DoW + harmonics   │
      │          + promo control) → eps, se, r2, n        │
      │  pass 2: empirical-Bayes shrinkage liên-SKU        │
      │  → bảng `elasticity(eps, eps_sd, n_points, r2,     │
      │                     method)`                       │
      └────────────────────────┬─────────────────────────┘
                               │
   ┌───────────────────────────▼──────────────────────────────────────┐
   │              run_decisions_once — VÒNG LẶP MỖI SKU                │
   │                                                                   │
   │  below_cost_alert ─┐                                              │
   │  cost_increase_alert ─┤                                           │
   │  price_suggestion  ─┤  §4 Lerner | robust-CVaR (pricing_policy)   │
   │       └─ competitor match/beat + apply_price_rules (§8)           │
   │  price_hold (robust) ─┤ §4.5                                      │
   │  bandit shadow     ─┤  §5 Thompson                                │
   │  replenishment_advice ─┤ §6 newsvendor (scenario tier-0 → legacy) │
   │  stockout_warning  ─┤                                             │
   │  slow_mover_alert  ─┤  §7 markdown ladder theo tuổi tồn           │
   │  promo_legal_alert ─┘                                             │
   │            │                                                      │
   │            ▼  ANTI-OSCILLATION GUARD (§9.1)                       │
   │      pass? ─── không ──► skipped_by_reason["anti_oscillation"]    │
   │            │                    └─► anti_osc_hold (giải thích)    │
   │            ▼ có                                                   │
   │      plan_candidates ──► resolve_decision_plan (§9.2)             │
   │            ├── winners     → INSERT decisions (status=open)       │
   │            └── superseded  → INSERT decisions (status=superseded, │
   │                              presentable=false, dedup +:superseded)│
   └───────────────────────────┬──────────────────────────────────────┘
       (sau vòng SKU, per-project)  bundle_suggestion · promo_candidate(A6, ĐANG CHẾT)
                               │
                               ▼
        GET /v1/decisions (presentable=true) ──► chủ shop
        POST /v1/decisions/{id}:feedback (accepted/dismissed)
                               │
                               ▼
        outcome_ledger.py (§11) — sau 14/21/30/90 ngày tuỳ kind
        → outcome_ledger(predicted_ev, realized_ev, window_days, note)
        → GET /v1/decisions:insights?kind=advice_scorecard

  [nhánh song song] experiment_registry + impression_log
        → experiment_gate.py (§10) FIRE / BLOCK / KILL → experiment_gate_audit
```

### 1.2 Bảng mọi loại quyết định (kind)

11 kind được sinh ra bởi 13 builder thuần trong `core/kinds.py`
(hai builder `replenishment_advice` / `replenishment_advice_scenario` cùng
sinh kind `replenishment_advice`; `price_hold` / `anti_osc_hold` cùng sinh
kind `price_hold`).

| # | kind | action | builder (`core/kinds.py`) | Điều kiện kích hoạt | EV basis |
|---|---|---|---|---|---|
| 1 | `below_cost_alert` | `set_price` | `:135` | `price < ewma_cost` | `profit_delta` |
| 2 | `cost_increase_alert` | `review_price` | `:169` | `last3_avg ≥ 1.10 × prev3_avg` (cần ≥6 dòng `cost_ledger`) | `profit_delta` |
| 3 | `price_suggestion` (model) | `set_price` | `:200` | \|P*−P\|/P > 5% **hoặc** competitor undercut | `profit_delta` |
| 4 | `price_suggestion` (bandit) | `set_price` | `:614` | arm Thompson lệch > 2% giá hiện tại | `profit_delta` |
| 5 | `price_hold` (robust) | `hold_price` | `:403` | mode=robust + optimizer giữ giá + Lerner muốn đổi >5% | `None` |
| 6 | `price_hold` (anti-osc) | `hold_price` | `:522` | guard chặn 1 đề xuất rank ≤2 và không còn hành động giá nào sống | `None` |
| 7 | `replenishment_advice` | `order` | `:686` (legacy) / `:777` (scenario) | `qty > 0` sau newsvendor | `profit_delta` |
| 8 | `stockout_warning` | `reorder` | `:987` | `p50_day × lead_time > on_hand` | `profit_delta` |
| 9 | `slow_mover_alert` | `markdown` | `:1061` | `on_hand>0` **và** `units_30d==0` **và** `history_days ≥ 30` | `capital_locked` |
| 10 | `promo_candidate` | `schedule_promo` | `:911` | EV > 0 ở một mức giảm ≤ `promo_cap` — **ĐANG CHẾT** (§7.3) | `profit_delta` |
| 11 | `promo_legal_alert` | `review_promo` | `:1231` | `discount_pct > promo_cap` (mặc định 50%) | `legal` |
| 12 | `bundle_suggestion` | `create_bundle` | `:1280` | `lift ≥ 2.0`, `pair_cnt ≥ 5`, cả 2 SKU margin > 15% | `profit_delta` |

### 1.3 Bảng mọi thuật toán

| # | Thuật toán | File chính | §|
|---|---|---|---|
| A1 | EWMA giá vốn (0.7/0.3) | `jobs/state_rollup.py:96` | §2 |
| A2 | OLS log-log ngày + mùa vụ + promo control | `core/econ/elasticity.py:35` | §3.1 |
| A3 | Empirical-Bayes shrinkage liên-SKU | `core/econ/elasticity.py:110` | §3.2 |
| A4 | OLS theo price-segment (legacy) | `core/econ/elasticity.py:164` | §3.3 |
| A5 | Lerner closed-form + cost-plus | `core/econ/pricing.py:18` | §4.1 |
| A6 | Hàm cầu log-log + EV Δprofit | `core/econ/pricing.py:87` | §4.2 |
| A7 | Mean-CVaR optimizer trên posterior | `core/econ/price_optimizer.py:218` | §4.3 |
| A8 | Chính sách chọn pricer 2 chế độ | `core/econ/pricing_policy.py:189` | §4.4 |
| A9 | Explain-hold (bisection `sd_for_action`) | `core/econ/price_explain.py:99` | §4.5 |
| A10 | Thompson sampling bandit giá | `core/econ/price_bandit.py:94` | §5 |
| A11 | Newsvendor critical ratio + order-up-to | `core/econ/replenish.py:10,31` | §6.1 |
| A12 | Newsvendor trên phân phối kịch bản chung | `core/scenario_newsvendor.py:53` | §6.2 |
| A13 | ROP / safety stock / DOI / MOQ / pack | `core/econ/replenish.py:120–166` | §6.3 |
| A14 | EOQ | `core/econ/replenish.py:66` | §6.4 |
| A15 | Markdown ladder theo tuổi tồn | `core/kinds.py:1139–1200` | §7.1 |
| A16 | Clear-rate v1 (legacy markdown) | `core/econ/slowmover.py:38` | §7.2 |
| A17 | Promo EV đa mức giảm | `core/kinds.py:911` | §7.3 |
| A18 | Guardrails giá (floor/MAP/charm) | `core/guardrails.py:219` | §8 |
| A19 | Anti-oscillation 3 luật | `core/guardrails.py:40` | §9.1 |
| A20 | DecisionPlan (thang ưu tiên + supersede) | `core/decision_plan.py:167` | §9.2 |
| A21 | Confidence-sequence gate FIRE/BLOCK/KILL | `core/experiment_gate.py:137` | §10 |
| A22 | Bundle/voucher margin floor | `core/kinds.py:1280` | §7.4 |
| A23 | Outcome ledger (EV thực) | `jobs/outcome_ledger.py:135` | §11 |

---

## 2. TẦNG TRẠNG THÁI — `jobs/state_rollup.py`

Job chạy nền mỗi `STATE_ROLLUP_INTERVAL` giây (mặc định **300**, `:354`), đọc
`raw_events` theo watermark trong `kv_state['state_rollup_watermark']`, **tối đa
5000 event mỗi lượt** (`:298`).

### 2.1 EWMA giá vốn — `_process_cost` (`:59`)

Bài toán chủ shop: *"giá nhập mỗi lô một khác, lấy con số nào làm 'vốn' để tính lãi?"*

```
ewma_cost_mới = 0.7 × ewma_cost_cũ + 0.3 × unit_cost_lô_mới      (state_rollup.py:96)
lần đầu (chưa có dòng): ewma_cost = unit_cost                    (state_rollup.py:91)
```

- α = **0.3** (hằng số cứng, không cấu hình được).
- Đồng thời ghi 1 dòng `cost_ledger(unit_cost, qty, supplier_ref, recorded_at)`.
- `last3_avg` / `prev3_avg` = trung bình 3 dòng ledger mới nhất / 3 dòng kế
  trước, lấy từ `ORDER BY recorded_at DESC, id DESC LIMIT 6` (`:100–114`).
  Đây là đầu vào của `cost_increase_alert`.

### 2.2 Các state khác

| Handler | Event | Bảng đích | Công thức / lưu ý |
|---|---|---|---|
| `_process_price` `:136` | `price.changed` | `price_state` (upsert) + `price_history` (append) | `price_history` chỉ lưu **new_price** — giá TRƯỚC dòng đầu tiên là không biết được (ảnh hưởng §3.1) |
| `_process_stock` `:168` | `stock.level` | `stock_state.on_hand_qty` | snapshot, **không có lịch sử** → không đo được "số ngày liên tiếp còn hàng" |
| `_process_purchase` `:190` | `purchase.completed` | `sales_daily` (units += , revenue += qty×unit_price) | key `(project, product, day)` |
| `_process_return` `:223` | `order.returned` | `sales_daily` | `units = GREATEST(units − qty, 0)`, `revenue = GREATEST(revenue − qty×unit_price, 0)` — **sàn 0**, không cho âm |
| `_process_competitor_price` `:256` | `competitor.price` | `competitor_price_state(competitor_price, competitor_ref)` | chỉ giữ giá mới nhất |

Watermark không bao giờ vượt `now()` (`:337–339`) — sự kiện có `received_at`
tương lai không được phép đóng băng pipeline.

---

## 3. ƯỚC LƯỢNG ĐỘ CO GIÃN GIÁ (ELASTICITY)

Bài toán chủ shop: *"giảm giá 10% thì bán được thêm bao nhiêu phần trăm?"*
Con số trả lời là **eps** trong mô hình cầu log-log: `Q(P) = Q₀·(P/P₀)^eps`.
`eps = −2` nghĩa là giảm giá 1% → lượng bán tăng 2%.

Hằng số toàn cục (`core/econ/elasticity.py:11–21`):

| Hằng số | Giá trị | Dòng | Ý nghĩa |
|---|---|---|---|
| `PRIOR_EPS` | **−1.3** | `:11` | prior khi không đủ dữ liệu |
| `MIN_SEGMENTS` | **8** | `:12` | số segment tối thiểu cho ước lượng legacy |
| `EPS_MIN` | **−5.0** | `:13` | kẹp dưới (co giãn phi lý) |
| `EPS_MAX` | **−0.2** | `:14` | kẹp trên (eps ≥ −0.2 không có cực đại lợi nhuận nội) |
| `MIN_DAYS_OLS` | **30** | `:21` | số ngày tối thiểu cho hồi quy ngày |

### 3.1 OLS log-log theo NGÀY — `estimate_elasticity_ols` (`:35`)

**Đây là ước lượng DUY NHẤT mà production ghi ra** (`refresh_all_elasticity`
gọi nó cho mọi SKU, `jobs/decisions_run.py:443`).

**Mô hình hồi quy** (thiết kế ma trận ở `_design_row`, `:24–32`):

```
ln(units_t) = β₀
            + eps · ln(price_t)
            + Σ_{k=0..5} γ_k · 1[dow_t = k]          (Chủ nhật = tham chiếu, bỏ)
            + δ₁ sin(θ_t) + δ₂ cos(θ_t)
            + δ₃ sin(2θ_t) + δ₄ cos(2θ_t)            (θ_t = 2π · day_of_year / 365)
            + φ · promo_t                             (promo_t ∈ {0,1})
            + ε_t
```

Tổng **13 cột** thiết kế: 1 intercept + 1 ln(price) + 6 dummy DoW + 4 harmonic + 1 promo.

Vì sao phải khử nhiễu: comment `:16–20` — *"naive ln(units)~ln(price) là chệch
khi giá di chuyển cùng các driver cầu (thứ trong tuần, mùa, lễ, khuyến mại);
ta hồi quy chúng ra để hệ số giá sạch."*

**Điều kiện chạy** (`:57–60`):
- Bỏ ngày có `units ≤ 0` hoặc `price ≤ 0` (log không xác định / bị kiểm duyệt).
- Cần `len(rows) ≥ 30` **VÀ** `len(distinct_prices) ≥ 2`.
- Không đủ → trả `{eps: −1.3, se: None, n, r2: None, method: "pooled_prior"}`.

**Chống suy biến hạng** (`:69–79`): bỏ mọi cột toàn-0 (một DoW hay promo chưa
bao giờ xuất hiện), **trừ cột giá luôn giữ**; nếu `rank < số cột` (giá cộng
tuyến hoàn hảo với 1 control) → rơi về `pooled_prior`.

**Nghiệm** (`:74`): `np.linalg.lstsq(Xk, y)` — nghiệm bình phương tối thiểu.

**Thống kê ra**:
```
r²   = 1 − SS_res / SS_tot                                   (:85)
σ²   = SS_res / (n − p)                                      (:90)
se(eps) = sqrt( σ² · [(XᵀX)⁻¹]_{price,price} )               (:93)
```
`se` chỉ tính khi `n > p` (`:89`), lỗi nghịch đảo → `se = None`.

**Kẹp** (`:99–102`): `eps < −5 → −5, clamped=True`; `eps > −0.2 → −0.2, clamped=True`.

Trả về `method = "ols_daily"`.

### 3.2 Empirical-Bayes shrinkage liên-SKU — `shrink_hierarchical` (`:110`)

Bài toán: SKU ít dữ liệu cho eps rất nhiễu; SKU nhiều dữ liệu thì tin được.
Thay vì kéo tất cả về một hằng số −1.3, kéo về **trung bình quần thể** với
trọng số theo độ chính xác.

```
usable = { i : method_i = "ols_daily" và se_i > 0 }           (:130-133)
nếu |usable| < 2 → trả nguyên OLS (hoặc PRIOR_EPS)            (:134-139)

prec_i = 1 / se_i²
μ      = Σ eps_i·prec_i / Σ prec_i                            (:144)  trung bình trọng-số-độ-chính-xác
τ²     = mean[(eps_i − μ)²] − mean[se_i²]                     (:146)  method of moments
τ²     = max(τ², 1e-6)                                        (:147)

với SKU có OLS + se:
    epŝ_i = ( eps_i/se_i² + μ/τ² ) / ( 1/se_i² + 1/τ² )       (:153)
    epŝ_i = clip(epŝ_i, −5.0, −0.2)                          (:154)

với SKU KHÔNG có OLS:
    epŝ_i = clip(μ, −5.0, −0.2)                              (:160)   ← KHÔNG phải −1.3
```

> ⚠ **Bẫy đọc số**: dòng `:155–160`. SKU `method='pooled_prior'` **vẫn được ghi
> eps = μ (trung bình quần thể)**, không phải `PRIOR_EPS = −1.3`. Đo trên DB
> live: `demoshop` có 15 dòng `pooled_prior` với `avg(eps) = −0.574`, đúng bằng
> `avg(eps)` của 109 dòng `ols_daily` cùng project. Ai đọc `method='pooled_prior'`
> mà giả định eps = −1.3 sẽ sai.

Ghi comment `:158–159`: *"mượn sức mạnh từ các SKU đã ước lượng được — tốt hơn
một hằng số toàn cục cố định."*

### 3.3 OLS theo price-segment (legacy) — `estimate_elasticity` (`:164`)

Đường cũ, **chỉ còn `_refresh_elasticity` (`decisions_run.py:255`) gọi và hiện
chỉ dùng trong test**. Ghi `method="ols"`.

```
ln(mean_units_s) = a + eps·ln(price_s) + b·promo_s     (3 cột, :196-204)
shrink cố định:  w = n/(n+10);  epŝ = w·eps_OLS + (1−w)·(−1.3)   (:215-216)
r² như trên; kẹp [−5, −0.2]
điều kiện: n ≥ MIN_SEGMENTS (8) và ≥2 giá phân biệt      (:187)
```

Ước lượng này **không có standard error** → `refresh` ghi `eps_sd = NULL`
(`decisions_run.py:316–337`, comment `:313–315`: *"eps mới không bao giờ được
ghép với sd cũ của một ước lượng khác"*).

`build_price_segments` (`:244`) dựng segment từ `price_history` × `sales_daily`:
mỗi khoảng `[changed_at_i, changed_at_{i+1})` → 1 segment với `mean_units`;
`promo` luôn = 0 (`:300`, "v1: promo not yet considered").

### 3.4 Vòng chạy 2 pass — `refresh_all_elasticity` (`decisions_run.py:389`)

Chạy **trước** vòng SKU trong mỗi lượt `run_decisions_once` (`:1232`).

1. Nạp toàn bộ `price_history`, `sales_daily`, và mọi cửa sổ `promo.scheduled`
   (`_get_promo_intervals`, `:363` — **toàn lịch sử**, khác `_get_promo_by_product`
   chỉ 30 ngày).
2. **Pass 1**: mỗi SKU dựng chuỗi quan sát ngày; giá ngày `d` = giá hiệu lực
   theo hàm bậc thang `_price_on_day` (`:351`) — **ngày trước dòng price_history
   đầu tiên bị BỎ** vì không biết giá. `promo=1` nếu ngày nằm trong bất kỳ cửa sổ.
3. **Pass 2**: `shrink_hierarchical` trên toàn bộ estimate.
4. Upsert vào `elasticity`. `eps_sd` = **`se` CHƯA shrink**, chỉ ghi khi
   `method == "ols_daily"` và `se > 0`, ngược lại `NULL` (`:460–467`).
   Comment `:456–459` ghi rõ đánh đổi: *"cố ý dùng se chưa shrink — posterior
   rộng hơn là lựa chọn thận trọng: nó đẩy optimizer về phía giá robust, không
   bao giờ nới rộng bước đổi giá. Siết lại là W-M18-EB-POSTERIOR-SD."*

Toàn hàm best-effort: mọi lỗi chỉ log warning, không raise (`:416–418`, `:489–492`).

### 3.5 Rubric confidence

`core/kinds.py:46–50`:

| Nhãn | Giá trị | Dùng khi |
|---|---|---|
| `full` | **0.9** | `method ∈ OLS_METHODS` (hồi quy thật trên dữ liệu của chính SKU) |
| `prior_elasticity` | **0.7** | `method = "pooled_prior"` |
| `forecast_degraded` | **0.5** | dự báo rơi về MA28 (replenish/stockout) |

`OLS_METHODS = ("ols", "ols_daily")` — **một tuple duy nhất** ở `kinds.py:71`,
import bởi `price_suggestion`, `price_hold`, và `main.py:1103` (`:price-preview`).

> 🩹 **Bài học W-OLS-DAILY-RESIDUE** (`kinds.py:66–70`): literal `("ols",)` từng
> được copy ra **4 chỗ**; commit `4625cd7` chỉ sửa 2 → `:price-preview` chấm một
> elasticity OLS sống ở 0.7 trong khi `price_suggestion`/`price_hold` chấm đúng
> row đó là 0.9. Cách sửa cấu trúc: **một tuple, import khắp nơi** — "bản sao
> không thể trôi khỏi thứ nó không sở hữu".

---

## 4. TỐI ƯU GIÁ

### 4.1 Lerner closed-form — `core/econ/pricing.py:18` `optimal_price`

Bài toán chủ shop: *"nên bán giá bao nhiêu để lãi tháng cao nhất?"*

Với cầu log-log `Q(P) = Q₀(P/P₀)^eps`, lợi nhuận `π(P) = (P−c)·Q(P)`.
`dπ/dP = 0` cho nghiệm đóng (Lerner / markup tối ưu):

```
eps < −1  (co giãn):    P* = c · eps / (1 + eps)          method="lerner"     (:44-47)
eps ≥ −1  (kém co giãn): P* = c · (1 + target_margin)      method="cost_plus"  (:48-50)
                          target_margin mặc định 0.15
```

Lý do nhánh 2: với `eps ≥ −1`, `π(P)` tăng đơn điệu theo P → **không có cực đại
nội**, công thức Lerner cho giá âm/vô nghĩa. Code rơi về cost-plus 15%.

**Guardrail trong chính hàm**:
```
clamp ±20%:  lower = P·0.80, upper = P·1.20                (:52-61, clamp_pct=0.20)
             vượt → kẹp + clamped=True
làm tròn tâm lý (:9-15, :64):
             _round_psychological(p) = floor(p/1000)·1000 − 1000
             ví dụ 186 400 → 185 000
sàn vốn sau làm tròn (:66-67):
             nếu suggested < cost → suggested = ceil(cost/1000)·1000
guardrail ra:  {"code":"PRICE_CLAMP_20PCT", "status": "APPLIED"|"PASS"}  (:69-77)
```

> ⚠ **Điểm giòn của Lerner** (ghi trong `price_optimizer.py:9–18`):
> `dP*/deps = c/(1+eps)²` **nổ tung khi eps → −1**. eps được ước lượng có nhiễu;
> đưa giá trị *trung bình* của một posterior vào một ánh xạ lồi gần điểm kỳ dị
> biến sai số nhỏ thành sai giá thảm hoạ. Đây là lý do tồn tại của §4.3.

### 4.2 EV thay đổi giá — `expected_profit_delta` (`pricing.py:87`)

```
Q(P) = units_30d · (P / P_hiện_tại)^eps
EV   = (P_mới − c)·Q(P_mới) − (P_cũ − c)·Q(P_cũ)
     = (P_mới − c)·units_30d·(P_mới/P_cũ)^eps − (P_cũ − c)·units_30d
```
(`Q(P_cũ)` rút gọn về `units_30d` vì `(P/P)^eps = 1`, `:101`.)
Trả `0.0` nếu giá nào ≤ 0. Đơn vị: **VND / 30 ngày**.

### 4.3 Mean-CVaR optimizer trên posterior — `core/econ/price_optimizer.py`

Bài toán: *"eps đo được ±1.4 — đổi giá theo con số trung bình có thể lãi to,
nhưng cũng có thể lỗ to. Chọn giá nào để kịch bản xấu vẫn chấp nhận được?"*

**Hai sai lầm mà nó sửa** (docstring `:9–19`):
1. Point-estimate fragility (đã nói §4.1).
2. **Flaw of averages**: `E[π(P)] = (P−c)·E[(P/P₀)^eps] ≠ (P−c)·(P/P₀)^E[eps]`.
   Hàm mục tiêu đúng phải **tích phân trên posterior**, không phải trên 1 điểm.

**Posterior** — `ElasticityPosterior` (`:71`), `eps_nodes` (`:89`):
```
node i = clip( mean + sd · Φ⁻¹((i+0.5)/n),  −5.0, −0.2 )      (:99-101)
```
Nút **phân tầng đẳng xác suất** (stratified quantile), KHÔNG Monte-Carlo:
- không RNG → bit-reproducible (`:39–44`);
- phủ đều cả 2 đuôi → CVaR ổn định.
`sd ≤ 0` → mọi node = mean (posterior chắc chắn, `:97–98`).

**Ma trận lợi nhuận** — `profit_matrix` (`:111`):
```
profit[s, g] = (P_g − c) · exp( eps_s · ln(P_g / P₀) )
```
Thang cầu `q₀ > 0` bị **bỏ** vì nó triệt tiêu trong `argmax` (chỉ đổi độ lớn).

**Hàm mục tiêu** — `optimize_price` (`:218`):
```
EV(P)     = mean_s profit[s, P]                                (:261)
k         = ceil(α · S)                                        (:262)
CVaR_α(P) = mean của k giá trị NHỎ NHẤT trong cột P            (:263)
objective(P) = (1 − λ)·EV(P) + λ·CVaR_α(P)                     (:264)
P*        = argmax trên lưới giá                               (:265-266)
```
`CVaR_α` = expected shortfall: lợi nhuận trung bình trên **α phần trăm kịch bản
tệ nhất** của eps. λ = 0 → thuần kỳ vọng; λ = 1 → tối đa hoá đuôi tệ nhất.

**Tham số mặc định** (`:223–229`):

| Tham số | Mặc định | Dòng |
|---|---|---|
| `risk_aversion` (λ) | **0.15** | `:223` |
| `alpha` (α) | **0.10** | `:224` |
| `clamp_pct` | **0.20** | `:225` |
| `floor_margin` | **0.05** | `:226` |
| `target_margin` | **0.15** | `:227` |
| `n_price` (điểm lưới giá) | **161** | `:228` |
| `n_eps` (nút posterior) | **161** | `:229` |

**Dải khả thi** — `feasible_band` (`:177`):
```
lo = max( c·(1 + floor_margin),  P_hiện·(1 − clamp_pct) )     = max(1.05c, 0.8P)
hi = P_hiện·(1 + clamp_pct)                                    = 1.2P
nếu hi < lo → hi = lo  (dải co về 1 điểm tại sàn)
```
Comment `:73–80`: dải này **NGẶT HƠN** đường Lerner (chỉ có sàn `cost`), và
optimizer **không thể ra khỏi clamp ±20%**. Mọi guardrail phía sau
(competitor match, `apply_price_rules`, anti-oscillation ±15%) vẫn nguyên:
*"optimizer đề xuất, guardrail định đoạt."*

**Kẹp + làm tròn** — `apply_band_and_round` (`:194`): kẹp vào `[lo, hi]`,
`_round_psychological`, rồi nếu thấp hơn `ceil(lo/1000)·1000` thì nâng lên.

**Nhãn method** (`:272–274`):
- `sd > 0`, λ > 0 → `"unit_econ_cvar"`
- `sd > 0`, λ = 0 → `"unit_econ_ev"`
- `sd ≤ 0` → `"unit_econ_point"` (thoái hoá về đúng Lerner)

**Guardrail trả về** (`:276–280`): `PRICE_CLAMP_20PCT` (APPLIED/PASS) +
`PRICE_FLOOR_ABOVE_COST` (luôn PASS — bất biến do `feasible_band` bảo đảm).

**Chi phí tính**: 161×161 = 25 921 phép, 1 pass numpy vector hoá, **dưới 1ms/SKU**
(`:44–48`). Rẻ hơn Lerner 10–100 lần thì không — nên nó chỉ dành cho batch
offline, không cho hot path.

### 4.4 Chính sách chọn pricer 2 chế độ — `core/econ/pricing_policy.py`

Đây là **NƠI DUY NHẤT** quyết định dùng Lerner hay robust (`:14–17`).

**Bảng chân trị** (`:32–39`):

| `DECISION_ROBUST_OPTIMIZER` | `project_config.pricing_mode` | Pricer chạy |
|---|---|---|
| unset / `1` | unset / `lerner` | Lerner (**dict nguyên vẹn**) |
| unset / `1` | `robust` | robust optimizer |
| `0` (hoặc bất kỳ giá trị lạ) | `robust` | Lerner (kill-switch) |
| `0` | `lerner` | Lerner |

- `_ALLOW_VALUES = {"1","true","yes","on"}` (`:115`). **Mọi giá trị khác — kể cả
  typo — ĐÓNG kill-switch và ép về Lerner** (`:113–114`: *"cờ không đọc được
  phải fail về hành vi cũ"*).
- `robust_allowed()` (`:123`): env **unset = cho phép** (opt-in nằm ở tenant).
- `normalize_mode()` (`:135`): mode lạ → log warning + `lerner`.
- Env λ/α: `DECISION_ROBUST_CVAR_LAMBDA` (mặc định **0.15**),
  `DECISION_ROBUST_CVAR_ALPHA` (mặc định **0.10**), đọc mỗi lần gọi
  (`~100ns` so với `~1ms` sweep, `:62`), ngoài khoảng hợp lệ → log + dùng mặc định.

**Hợp đồng trả về** — `choose_price` (`:189`):
- Chế độ lerner: trả **y hệt byte-for-byte** `optimal_price(...)`, không thêm key
  nào (`:221–222`) → tenant mode 1 không thể quan sát thấy module này tồn tại.
- Chế độ robust: thêm `price_source` (`lerner` | `robust_optimizer`),
  `fallback_reason`, và các trường rủi ro của optimizer.

**Fallback (không bao giờ crash — batch không được mất SKU)** (`:70–73`):

| `fallback_reason` | Điều kiện | Dòng |
|---|---|---|
| `no_posterior_sd` | `eps_sd` là None / không hữu hạn / ≤ 0 | `:224–225` |
| `non_positive_cost_or_price` | `cost ≤ 0` hoặc `price ≤ 0` | `:226–227` |
| `optimizer_error:<ExcName>` | bất kỳ exception nào của optimizer | `:240–252` |

> 📌 **Quyết định lịch sử `D-ROBUST-ON-WITH-EXPLAIN`** (DB rail, human chốt
> 2026-08-05 sau 5 message): *"cách mới an toàn kinh tế nhưng ít hành động khi
> elasticity chưa identify (đo thật: OFF 6 gợi ý, ON 0) — 2 chế độ cho merchant
> tự chọn per-tenant + demo đối tác 2 tenant cạnh nhau; explain-hold để máy
> không bao giờ im lặng khó hiểu."*
> Đổi so với `cb9788b`: trước đó env flag một mình bật optimizer cho **cả fleet**;
> nay pricer là **lựa chọn thương mại của từng merchant** (`:41–46`).

### 4.5 Giải thích GIỮ GIÁ — `core/econ/price_explain.py` + `kinds.price_hold`

Bài toán sản phẩm (docstring `:16–22`): mode robust cố ý im lặng. Đo live
2026-08-05: tenant `m18d` ra **6 gợi ý dưới Lerner và 0 dưới robust**.
*"Im lặng là GIÁ đúng và là SẢN PHẨM tệ nhất: chủ shop thấy hộp thư trống và
kết luận hệ thống hỏng."*

**Điều kiện phát sinh `price_hold`** (`kinds.py:403–461`) — cả 5 phải đúng:
1. `robust_enabled_for(pricing_mode)` (`:438`);
2. `units_30d > 0`, `price > 0`, `ewma_cost > 0` (`:440`);
3. `eps_sd` tồn tại, hữu hạn, > 0 (`:446–448`) — *fallback về Lerner KHÔNG phải
   robust hold*;
4. `choose_price(...)["price_source"] == "robust_optimizer"` (`:453`);
5. `|P_robust − P|/P ≤ 0.05` (giữ giá) **và** `|P_lerner − P|/P > 0.05`
   (Lerner thì muốn đổi) — `:456–461`.
   → nếu cả hai cùng giữ: không có bất đồng nào đáng ghi.

**Nội dung explain** — `build_hold_explain` (`:168`):
```
CI 80% của eps:   [eps − 1.2816·sd,  eps + 1.2816·sd]     (:73, :79-82)
                  CI_LEVEL = 0.80, z = NormalDist().inv_cdf(0.90) = 1.2816

base_profit_30d = units_30d · (P − c)                      (:194)
   ← lợi nhuận giá hiện tại là TẤT ĐỊNH: tại điểm neo (P/P)^eps = 1 với mọi eps
ev_move   = units_30d · E[profit(P_lerner)]                (:195-197)
cvar_move = units_30d · CVaR_α[profit(P_lerner)]           (:198-200)
ev_if_change   = ev_move − base_profit                     (:201)
cvar_if_change = cvar_move − base_profit                   (:202)
ev_pct / downside_pct = 100 · Δ / base_profit              (:203-204)
```

**"Cần thêm bao nhiêu dữ liệu?"** — hai bước, KHÔNG bịa số:

`sd_for_action` (`:99`) — **bisection trên chính optimizer**, không phải ngưỡng
tay chọn:
```
bất biến: moves(lo)=True, holds(hi)=True; khởi tạo lo=0, hi=sd_hiện_tại
lặp 14 lần (_BISECT_ITERS, :76):
    mid = (lo+hi)/2
    nếu |P_robust(mid) − P|/P > 0.05 → lo = mid   ngược lại → hi = mid
trả lo  = sd LỚN NHẤT mà optimizer VẪN đổi giá
```
Trả `None` (và explain phải nói "không lượng hoá được") khi:
- `sd = 0` mà vẫn giữ giá → hold không phải câu chuyện bất định (`:126–127`);
- `sd` hiện tại đã đủ để đổi giá → không phải hold (`:128–131`).

`observations_needed` (`:143`) — dùng `se ∝ 1/√n`:
```
n_needed = ceil( n_hiện_tại · (sd_hiện_tại / sd_target)² )
n_more   = max(0, n_needed − n_hiện_tại)
```

**Câu tiếng Việt sinh ra** (`:242–249`) ghép từ chính các con số trên:
> *"Độ nhạy giá chưa đủ tin cậy (eps=−1.24 ±0.86; khoảng tin cậy 80%:
> −2.34..−0.14): giảm giá 1.221.000 → 1.050.000 (−14.0%) lúc này kịch bản xấu
> (10% tệ nhất) mất tới 15% lợi nhuận tháng, trong khi kỳ vọng chỉ +3%.
> Vì vậy hệ thống GIỮ giá 1.221.000 VND; cần thêm khoảng 417 quan sát giá/ngày
> nữa (tổng ~500, hiện có 83) để đủ chắc."*

Chi phí: ~14 sweep optimizer / SKU giữ giá ≈ **15 ms**, chỉ cho SKU thật sự
giữ giá (`:34–36`).

**Payload trả ra** (`:251–288`): 24 field, gồm `eps_ci`, `lerner_price`,
`lerner_move_pct`, `base_profit_30d`, `ev_if_change(_pct)`,
`cvar_if_change(_pct)`, `cvar_alpha`, `risk_aversion`, `eps_sd_target`,
`observations_needed_total/_more`, `observations_formula`, `reason_vi`.

**EV của `price_hold` = `None`** (`kinds.py:506`) — cố ý: *"một cái giữ giá KHÔNG
đưa ra tuyên bố giá trị nào: nó không kiếm cũng không tiêu. EV/CVaR phản thực
của việc đổi giá nằm bên trong `explain`."*

### 4.6 Trigger của `price_suggestion` — `kinds.py:200`

```
if units_30d <= 0:                       return None    (:239)
if eps is None:                          return None    (:241-243)
pricing_res = choose_price(cost, eps, price, eps_sd, mode)   (:245)
p_star = pricing_res["suggested_price"]

model_wants_change  = |p_star − price| / price > 0.05          (:253)
competitor_undercut = competitor_price is not None và < price  (:254-256)
if not model_wants_change and not competitor_undercut: return None   (:257)
```

**Nhánh competitor (F-COMPETITOR-REPRICE-1)** (`:262–300`):
```
nếu p_star > competitor_price:
    candidate = floor((competitor_price − 1000)/1000)·1000     ← match/beat
    nếu candidate ≥ ewma_cost:
        band_floor = price · (1 − 15/100)                       (:275)
        nếu candidate < band_floor:                             ← W-UNDERCUT-CLAMP-BAND
            stepped = ceil(band_floor/1000)·1000                (:277)
            nếu stepped < price: candidate = stepped; clamped_from ghi lại
        p_star = candidate;  guardrail COMPETITOR_MATCH=APPLIED
                             (+ ANTI_OSC_CLAMP=APPLIED nếu bị kẹp)
    ngược lại (đối thủ bán dưới vốn ta): COMPETITOR_MATCH=PASS, không match
ngược lại: COMPETITOR_MATCH=PASS
```
> 🩹 **Bài học W-UNDERCUT-CLAMP-BAND** (`:269–274`, đo live 2026-08-05): một cú
> match sâu hơn dải ±15% sẽ bị guard **vứt trọn**. Chuẩn repricer: **bước tới
> mép dải** thay vì nhảy — bản thân guard không bị đụng vào.

Nếu **chỉ có undercut** làm trigger mà không tạo ra được giá match nào
(`COMPETITOR_MATCH` không APPLIED) → `return None` (`:302–311`): không có gì
hành động được.

Sau đó `apply_price_rules` (§8) với `hard_floor = ewma_cost` (`:316`), và nếu
luật đẩy giá về sát giá hiện tại (`≤ 0.1%`) → `return None` (`:323–324`).

**Trace** (`:340–361`) khác nhau theo `price_source`, và trace khi flag OFF là
**byte-identical với trước khi wire** (`:337–339`).

`action_params.price_source` cố ý **KHÔNG dùng key `source`** (`:365–367`): key
`source` chỉ định danh **PRODUCER** của đề xuất (`bandit_shadow` / `scenario` /
`quantile_legacy`) và nuôi anti-oscillation + hậu tố dedup superseded.

### 4.7 `:price-preview` — `main.py:1014`

API "nếu tôi để giá X thì sao?" — chạy **thẳng hàm cầu**, không qua optimizer:
```
q_candidate      = units_30d · (candidate_price / current_price)^eps      (:1084)
profit_current   = (current_price − ewma_cost) · units_30d                (:1086)
profit_candidate = (candidate_price − ewma_cost) · q_candidate            (:1087)
delta_profit_30d = profit_candidate − profit_current                      (:1117)
```
Tiền điều kiện (412 FAILED_PRECONDITION): `units_30d > 0`, có `cost_state`,
có `price_state`. `candidate_price ≤ 0` → 400 (`:1027–1032`, ghi rõ: đo
`bt02_100q` 2026-08-04, giá âm/0 → HTTP 500).
Không có elasticity → dùng `{eps: −1.3, method: "pooled_prior"}` (`:1076`).

> 🩹 **BUG SẢN PHẨM vừa sửa 2026-08-06, commit `b63938d`** (`main.py:1090–1095`):
> guardrail `BELOW_COST` trước đây **cả hai nhánh if/else đều trả `PASS`** — một
> giá 9 000đ với `ewma_cost` 70 458đ (dưới vốn 87%) vẫn hiện xanh, UI tin
> guardrail mà áp giá lỗ. Nay:
> ```python
> if candidate_price < ewma_cost:
>     guardrails.append({"code": "BELOW_COST", "status": "FAIL"})
> else:
>     guardrails.append({"code": "BELOW_COST", "status": "PASS"})
> ```
> Test hồi quy 2 chiều: `tests/decision/test_preview_belowcost.py` (159 dòng).
> Đây là guardrail **DUY NHẤT** trong toàn hệ dùng trạng thái `FAIL`
> (mọi guardrail khác dùng `PASS` / `APPLIED` / `BLOCKED`).

---

## 5. BANDIT GIÁ — `core/econ/price_bandit.py` (SHADOW MODE)

Bài toán: *"mô hình elasticity có thể sai; thử vài mức giá quanh giá hiện tại để
HỌC từ thực tế bán."*

> ⚠ **SHADOW MODE tuyệt đối** (`:1–6`, `kinds.py:626–631`): bandit **KHÔNG BAO GIỜ
> tự đổi giá**. Đầu ra duy nhất là một `price_suggestion` (gợi ý, chủ shop quyết);
> mọi đề xuất vẫn qua `apply_price_rules` + anti-oscillation như mô hình
> elasticity. *"Guardrail LUÔN thắng arm, bắt đầu NGAY TỪ LƯỚI."*

**Loại bandit**: **Thompson sampling** với posterior Gaussian
(khảo sát 2026-08-04: chuẩn công nghiệp cho dynamic pricing dữ liệu mỏng; anh em
cùng pattern: `services/smartsearch/app/core/bandit.py::thompson_pick`).

**Hằng số** (`:36–42`):

| Hằng số | Giá trị | Dòng |
|---|---|---|
| `GRID_SPAN` | **0.10** (±10%) | `:36` |
| `PRIOR_SIGMA_FRAC` | **0.5** | `:38` |
| `PRIOR_SIGMA_MIN` | **1.0** | `:39` |
| `MIN_PROPOSE_DELTA_PCT` | **0.02** (2%) | `:42` |
| `n_arms` mặc định | **5** | `:48` |
| `OBSERVE_WINDOW_DAYS` | **7** | `store/price_bandit.py:18` |

**Lưới arm** — `build_price_grid` (`:45`):
```
offset_i = −0.10 + 0.20·i/(n_arms−1),  i = 0..n_arms−1     → −10%, −5%, 0, +5%, +10%
cand_i   = round(current_price · (1 + offset_i))
cand_i   = max(cand_i, ceil(floor))              ← guardrail thắng NGAY TẠI LƯỚI
bỏ trùng, sort tăng dần
```
`floor` truyền vào chính là `ewma_cost` (`decisions_run.py:734`). Nếu các offset
thấp vượt qua sàn thì **chính sàn trở thành một arm**.

**Prior từ mô hình elasticity** — `prior_profit_per_day` (`:73`):
```
μ₀(arm) = [ (P_hiện − c)·units_30d  +  expected_profit_delta(c, P_hiện, arm, units_30d, eps) ] / 30
σ₀(arm) = max( |μ₀| · 0.5,  1.0 )        ← CỐ Ý RỘNG để ép khám phá
n₀      = 0
```
Bandit **khởi đi từ niềm tin của mô hình**, không từ số 0.

**Bước Thompson** — `sample` (`:134`):
```
với mỗi arm:  score ~ Normal( μ, σ / sqrt(max(n, 1)) )
chọn arm có score lớn nhất
```
`exploration = (arm_chọn ≠ greedy_arm())`, `greedy_arm` = arm có μ lớn nhất
(`:154`, không lấy mẫu).

**Cập nhật posterior — Welford online** (`:164`):
```
n = 0:  μ ← x,  n ← 1        (quan sát thật đầu tiên THAY prior mean;
                              σ giữ nguyên độ rộng prior tới khi n ≥ 2)
n ≥ 1:  M2_prev = σ²·(n−1) nếu n ≥ 2, ngược lại 0
        n' = n + 1
        δ  = x − μ
        μ' = μ + δ/n'
        M2'= M2_prev + δ·(x − μ')
        σ' = sqrt( M2' / (n'−1) )
```
Với `n ≥ 2` thì μ/σ là **trung bình mẫu / độ lệch chuẩn mẫu chính xác**.

**Reward quan sát** — `store/price_bandit.py:75`:
```
observed_profit_per_day = ( SUM(units 7 ngày gần nhất) / 7 ) · (price − cost)
```
Đây là *"cách bandit shadow học từ hành vi thật của chủ shop kể cả khi ông ta bỏ
qua mọi gợi ý"* (`:82–84`).

**Vòng chạy** — `_run_price_bandit_shadow` (`decisions_run.py:709`):
1. **HỌC**: cập nhật arm ở **giá đang bán thật** (`arm_cur = round(price)`), với
   **idempotent theo ngày** — arm đã cập nhật hôm nay thì không nạp lại cùng một
   quan sát (`:751–758`). Ghi `price_bandit_state`.
2. **LẤY MẪU**: `pricer.sample()`.
3. **ĐỀ XUẤT**: nếu arm lệch > 2% giá hiện tại → `bandit_price_suggestion`.
   Toàn khối bọc `try/except` (`:1399–1417`) — bandit hỏng không được giết batch.

**Builder** — `bandit_price_suggestion` (`kinds.py:614`):
- `None` nếu `|arm − price|/price ≤ 0.02` (`:640`) hoặc nếu luật giá đẩy về
  ~giá hiện tại (`:651–652`).
- `confidence = RUBRIC["prior_elasticity"] = 0.7` **luôn luôn** (`:680`).
- `action_params`: `suggested_price`, `source="bandit_shadow"`, `arm_price`,
  `arms_state` (snapshot `{price, mu(2 chữ số), n}` mỗi arm), `exploration`.

**Trạng thái lưu** — `price_bandit_state(project_id, product_id, arm_price, mu,
sigma, n, updated_at)`, PK `(project_id, product_id, arm_price)`, RLS theo tenant.

> 🩹 **Bẫy đã trả giá** (`facts:bt02.price-bandit.gotcha-demo-24h`, rail DB):
> trong demo/seedtest **mọi SKU có elasticity đều đã có `price_suggestion` model
> trong <24h** → anti-osc rule (a) chặn hết đề xuất bandit. Muốn thấy bandit
> chạy trong demo phải chuẩn bị SKU chưa có đề xuất nào trong 24h.

---

## 6. NEWSVENDOR & REPLENISHMENT

### 6.1 Critical ratio + order-up-to (đường legacy) — `core/econ/replenish.py`

Bài toán chủ shop: *"nhập bao nhiêu là vừa? Nhập thiếu mất doanh thu, nhập thừa
chôn vốn."* Newsvendor cân đúng hai chi phí đó.

`critical_ratio` (`:10`):
```
Cu = price − cost                        (chi phí thiếu hàng = lãi mất đi 1 đơn vị)
     nếu Cu ≤ 0 → trả 0.0
Co = cost · holding_rate_annual · review_period_days / 365
     (chi phí thừa hàng = phí giữ vốn trong 1 chu kỳ review)
CR = Cu / (Cu + Co)
trả min(CR, 0.99)                        ← trần cứng 0.99
mặc định: holding_rate_annual = 0.25, review_period_days = 7
```
CR chính là **mức phục vụ tối ưu về kinh tế** (quantile của phân phối cầu cần
phủ). Biên lãi càng dày → CR càng gần 1 → nhập càng nhiều.

`order_up_to` (`:31`):
```
q = min( max(CR, service_level_floor or 0),  0.99 )
S = quantile_interp(p10, p50, p90, q)
```
`quantile_interp` (`libs/featurelib/quantiles.py:12`) nội suy 2 đoạn tuyến tính:
`q∈[0.1,0.5]` giữa p10–p50; `q∈[0.5,0.9]` giữa p50–p90; `q<0.1` kéo dài dốc
đoạn 1 và **kẹp ≥ 0**; `q>0.9` kéo dài dốc đoạn 2 và **kẹp ≤ p90 + 2(p90−p50)**.

`suggested_qty` (`:49`):
```
raw = max(0, S − on_hand − on_order)
qty = ceil(raw / lot_multiple) · lot_multiple        lot_multiple mặc định 5
```

> 🩹 **BUG đã sửa `T-CRITICAL-RATIO-ARGBUG` (2026-08-04)** — ghi tại
> `kinds.py:714–718`: lời gọi vị trí cũ đẩy `lead_time` vào tham số
> `holding_rate_annual`, **thổi phồng chi phí thừa ~28 lần** → CR tụt → **nhập
> thiếu triền miên**. Nay bắt buộc keyword: `critical_ratio(price, ewma_cost,
> review_period_days=review)`, `holding_rate_annual` giữ mặc định 0.25.

### 6.2 Newsvendor trên phân phối kịch bản CHUNG — `core/scenario_newsvendor.py`

Tier-0 (ưu tiên cao nhất). Artifact scenario của service `forecast` trả lời câu
hỏi **lead-time demand (LTD)** bằng cách cộng **TRONG từng kịch bản**, không bao
giờ cộng quantile của từng SKU (`:4–8`).

`interpolate_quantile_at` (`:19`):
```
marks = sorted( (mức_quantile, giá_trị_LTD) )
cr < mark thấp nhất → trả giá trị mark thấp nhất, cr_clamped = True
cr > mark cao nhất  → trả giá trị mark cao nhất, cr_clamped = True
trong khoảng        → nội suy tuyến tính giữa 2 mark kẹp, cr_clamped = False
```

`compute_scenario_order` (`:53`):
```
S*      = LTD_quantile(CR)                                     (nội suy như trên)
raw     = max(0, S* − inventory_position)      (position âm = backorder → đơn to hơn)
pack    = max(1, pack_size)
qty     = floor(raw/pack + 0.5) · pack                          ← LÀM TRÒN NỬA LÊN theo pack
nếu qty > 0 và qty < moq:  qty = ceil(moq/pack) · pack
qty làm tròn về 0 thì GIỮ 0 — MOQ không bao giờ ép mua
```
Trả `{target_level, order_qty, cr_used, cr_clamped, source:"scenario"}`.

**Điểm quantile yêu cầu forecast** (`decisions_run.py:894`):
`required = sorted({round(CR, 2), 0.5, 0.9})` — luôn xin thêm mốc 0.5 và 0.9.

**Điều kiện rơi về legacy** (`_scenario_replenishment`, `:860`): trả
`scenario_used=False` khi **bất kỳ** trong: không có forecast client / client
không có `fetch_lead_time_demand` / không có `forecast_api_key` / exception /
`ltd` rỗng hoặc thiếu `quantiles` (`:885–913`).
`scenario_used=True` + `decision=None` nghĩa là **phân phối chung nói không cần
nhập** — KHÔNG rơi về legacy (`:881–883`).

**Đường legacy khi không có scenario** — `_get_totals` (`:814`): nếu gọi
`forecast:query` được thì `degraded=False`; ngược lại **degrade về MA28**
(`:846–857`):
```
p50_day = units_30d / 28      (nếu units_30d > 0, ngược lại 0)
horizon = lead_time + review
p50 = p50_day · horizon;  p10 = 0.5·p50;  p90 = 1.5·p50;  model_used = "MA28"
degraded = True  →  confidence rơi xuống RUBRIC["forecast_degraded"] = 0.5
```
Quyết định legacy được gắn thêm `action_params.source = "quantile_legacy"` và
`degradation_mode = "NO_SCENARIO"` (`:1506–1507`).

### 6.3 ROP / Safety stock / DOI (F-DC-REPLENISH-2)

Bảng z (`replenish.py:87`): `{0.90: 1.28, 0.95: 1.65, 0.99: 2.33}`.
`DEFAULT_SERVICE_LEVEL = 0.90` (`:89`). `z_for_service_level` (`:92`) **snap về
mức gần nhất** trong 3 mức đó; `None` → 0.90.

```
daily_stats(units_by_day)  (:104)   → (avg_daily, sigma_daily)
    avg   = Σu / n
    sigma = sqrt( Σ(u−avg)² / (n−1) )        ← độ lệch chuẩn MẪU; n<2 → 0
    ⚠ caller PHẢI zero-fill: ngày không có dòng sales_daily là ngày bán 0,
      không phải dữ liệu thiếu  (:107-109)

safety_stock(avg_d, σ_d, LT, σ_LT, z)  (:120)
    SS = z · sqrt( LT·σ_d²  +  avg_d²·σ_LT² )
    σ_LT = 0  ⇒ rút gọn ĐÚNG về dạng kinh điển z·σ_d·√LT
    LT ≤ 0 → 0

reorder_point(avg_d, LT, SS)  (:139)
    ROP = avg_d · max(0, LT) + max(0, SS)

days_of_inventory(on_hand, avg_d)  (:148)
    DOI = max(0, on_hand) / avg_d ;  avg_d ≤ 0 → None

round_up_moq_pack(qty, moq, pack)  (:155)
    qty ≤ 0 → 0            (MOQ KHÔNG ép mua)
    qty < moq → qty = moq
    pack > 1  → qty = ceil(qty/pack)·pack
    trả ceil(qty)
```

`_get_daily_stats` (`decisions_run.py:183`) zero-fill đúng **30 ngày**:
`start = date.today() − 29 ngày`, series 30 phần tử.

**Mặc định nhà cung cấp** khi không có dòng `supplier_config`
(`decisions_run.py:1448–1454`): `lead_time = 7`, **`lead_time_std = 2.0`**,
`moq = 0`, `pack_size = 1`.
> ⚠ Lưu ý: `store/supplier.py:_DEFAULTS` cho `lead_time_std = 0.0` khi **ghi**
> config, nhưng `decisions_run.py:1451` dùng **2.0** khi **thiếu dòng**. Hai con
> số khác nhau, cố ý theo comment `:1449–1450` ("missing supplier row -> LT std
> default 2").

### 6.4 EOQ — `replenish.py:66`

```
EOQ = sqrt( 2·D·K / (cost · holding_rate_annual) )
fixed_order_cost ≤ 0  hoặc cost ≤ 0  hoặc rate ≤ 0  →  0.0
```
**CHƯA CHẮC**: hàm tồn tại và có test, nhưng không tìm thấy caller nào trong
`core/kinds.py` hay `jobs/` — xem §14.

### 6.5 Các kind tồn kho

`replenishment_advice` (`kinds.py:686`) / `replenishment_advice_scenario` (`:777`):
```
EV_legacy   = (price − ewma_cost) · max(0, totals["p50"] − on_hand)      (:753)
EV_scenario = (price − ewma_cost) · max(0, ltd_mean − on_hand)           (:846)
confidence  = 0.5 nếu degraded, ngược lại 0.9      (legacy, :754)
              scenario: LUÔN 0.9                    (:864)
```
Sau newsvendor còn một vòng MOQ/pack nữa ở `kinds.py:727–731` (legacy).

`stockout_warning` (`kinds.py:987`):
```
p50_day       = totals["p50"] / max(1, totals["horizon_days"])           (:1010)
expected_lead = p50_day · lead_time
trigger khi expected_lead > on_hand                                      (:1012)
urgency_days  = on_hand / p50_day  (0 nếu p50_day = 0)                   (:1015)
EV            = (price − ewma_cost) · max(0, expected_lead − on_hand)    (:1016)
order_qty_moq_pack = round_up_moq_pack(max(0, ROP − on_hand), moq, pack) (:1026)
```

---

## 7. MARKDOWN / HÀNG Ế

### 7.1 Ladder theo TUỔI TỒN (F-MARKDOWN-1) — `kinds.py:1139–1228`

Bài toán: *"hàng nằm kho 3 tháng không ai mua — giảm bao nhiêu để đẩy đi mà
không lỗ?"*

**Điều kiện kích hoạt** (`:1089`) — cả 3:
`on_hand > 0` **và** `units_30d == 0` **và** `history_days ≥ 30`.

**Vốn chôn** (`core/econ/slowmover.py`):
```
capital_locked      = on_hand · ewma_cost                        (:6)
holding_cost_month  = capital · holding_rate_annual / 12         (:11)   rate = 0.25
```
`expected_value.amount = capital_locked + holding_cost_month`, `basis="capital_locked"`
(`kinds.py:1221`, `:1226`) — **đây là ĐỘ LỚN CỦA VẤN ĐỀ, không phải giá trị của
hành động** (điểm mấu chốt cho §11).

**Bậc thang theo tuổi** — `age_tier` (`slowmover.py:19`):

| Tuổi tồn | tier | markdown |
|---|---|---|
| ≥ 90 ngày | 3 | 30% |
| ≥ 60 ngày | 2 | 20% |
| còn lại | 1 | 10% |

Comment `:22–26`: chọn 30/60/90 cho SME Việt Nam — *"Amazon FBA dùng 181/271/365
nhưng shop nhỏ VN quay vòng hàng nhanh hơn nhiều."*

**Đo tuổi tồn** — `_get_age_days` (`decisions_run.py:495`), thứ tự ưu tiên
(mọi cột đều verify bằng psql 2026-08-04):
1. `last_receipt` — số ngày từ `MAX(cost_ledger.recorded_at)` (ngày nhập thật);
2. `last_sale` — số ngày từ `MAX(sales_daily.day)` có `units > 0`;
3. `history_start` — số ngày từ `MIN(sales_daily.day)` (cận dưới);
4. `(None, "unknown")`.
`stock_state` chỉ có snapshot → proxy *"số ngày liên tiếp còn hàng bán chậm"*
**KHÔNG đo được** (`:507–509`).

**Tính từng bậc** (`:1147–1182`):
```
với step_pct ∈ (10, 20, 30):
    raw     = price · (1 − step_pct/100)
    floored = (raw < ewma_cost)
    p_step  = round( max(raw, ewma_cost) )        ← SÀN VỐN LUÔN THẮNG MODEL
    p_step  = apply_price_rules(p_step, price, pid, rules, hard_floor=ewma_cost)

    EV mô hình A (khi có eps và units_hist_30d > 0):
        d_eff   = max(0, 1 − p_step/price)
        q_new   = units_hist_30d · (1 − d_eff)^eps
        lift    = min( max(0, q_new − units_hist_30d), on_hand )
        ev_step = expected_profit_delta(cost, price, p_step, units_hist_30d, eps)
        ev_model = "elasticity"                    ← đơn vị: VND / 30 NGÀY

    EV mô hình B (thiếu eps hoặc thiếu baseline):
        clear   = clear_rates[idx],  clear_rates = (0.3, 0.6, 0.9)   (:1095)
        lift    = on_hand · clear
        ev_step = (p_step − ewma_cost·0.5) · lift   ← so với sàn thanh lý 50% vốn
        ev_model = "clear_rate_v1"                 ← đơn vị: TOÀN chiến dịch
```
**Bậc được CHỌN** = `schedule[tier − 1]` (`:1184`), tức bậc ứng với tuổi tồn
hiện tại, không phải bậc EV cao nhất.

**Baseline cầu cho SKU ế** — `_get_units_hist_30d` (`decisions_run.py:549`):
slowmover có `units_30d == 0` theo định nghĩa, nên EV elasticity cần baseline cũ:
```
units_hist_30d = SUM(units toàn lịch sử) / COUNT(DISTINCT day) · 30
```

**Guardrail ra** (`:1185–1190`): `MARKDOWN_COST_FLOOR` = `APPLIED` nếu bậc được
chọn bị sàn vốn cắt, ngược lại `PASS`; nối thêm guardrail của `apply_price_rules`
cho đúng bậc đó.

**`action_params.price_before`** (`:1211–1216`) — ghi lại giá TRƯỚC khi giảm:
> *"phép đo kết quả 90 ngày cần nó để dựng phản thực 'không giảm giá', và lượt
> chạy này là khoảnh khắc DUY NHẤT nó không mập mờ — ba tháng sau `price_history`
> có thể chứa chính các bậc markdown, hoặc không có gì cả trên tenant onboard
> bằng backfill. **Ghi lại, không suy lại.**"*

### 7.2 Ladder legacy theo VỐN — `slowmover.markdown_price` (`:38`)

Dùng khi thiếu `age_days` hoặc `price` (`kinds.py:1097–1137`):
```
step_idx 0/1/2 → markdown_step 0.1/0.2/0.3
salvage_floor = 0.5 · cost                          (salvage_floor_ratio = 0.5)
price = round( max(salvage_floor, cost·(1 − markdown_step)) )
step_idx ngoài {0,1,2} → raise ValueError
```
Lịch: `after_days = 30·(step_idx+1)` → 30/60/90; `expected_clear_rate` = 0.3/0.6/0.9;
`ev_vs_salvage = (price_step − cost·0.5) · (on_hand · clear)`.
`suggested_price` = bậc 1 (`:1119`).
Ở đường này `max_discount_pct` **không áp được** (không biết giá hiện tại) —
`apply_price_rules(price_step, None, ...)` (`:1105`), chỉ còn MAP + charm.

### 7.3 Promo candidate (A6) — `kinds.py:911` — **ĐƯỜNG CHẾT CÓ CHỦ ĐÍCH**

Công thức:
```
với disc_pct ∈ (10, 20, 30), bỏ nếu > promo_cap:
    d          = disc_pct/100
    new_price  = price·(1 − d)
    margin_new = new_price − ewma_cost;   nếu ≤ 0 → bỏ mức này
    q_promo    = units_30d · (1 − d)^eps
    EV         = margin_new·q_promo − (price − ewma_cost)·units_30d
chọn mức EV dương LỚN NHẤT; không mức nào EV > 0 → None
eps ≥ −0.2 → None ngay ("kém co giãn: giảm giá chỉ phá biên lãi", :933-935)
```
`STOCK_LIMITED` = APPLIED khi `on_hand < q_promo` — *tín hiệu kế hoạch, không
phải trần cứng: chủ shop nhập thêm trước cũng được* (`:958–963`).

> 🩹 **VÌ SAO ĐƯỜNG NÀY CHẾT** — `PROMO_ELASTICITY_METHODS = ("ols",)` (`:908`),
> **CỐ Ý hẹp hơn `OLS_METHODS`**. Comment `:870–907` ghi số đo: 1741 dòng
> elasticity trên 94 project = 1358 `pooled_prior` + 383 `ols_daily`, **ZERO
> `ols`** → A6 không sinh dòng nào; bảng `decisions` có **0 dòng promo_candidate
> từ trước tới nay**. Thêm nữa `run_decisions_once` gọi `refresh_all_elasticity`
> cho MỌI SKU trước khi đọc bảng, nên kể cả dòng `ols` cũ cũng bị ghi đè thành
> `ols_daily`/`pooled_prior` ngay đầu lượt.
>
> **Nới tuple này một mình sẽ tạo QUYẾT ĐỊNH SAI**, vì hai lý do đã đo:
> 1. `promo_candidate` được dựng **theo PROJECT sau vòng SKU** (`decisions_run.py:1804–1845`)
>    nên nó **không vào `plan_candidates`** và DecisionPlan không nhìn thấy → vỡ
>    bất biến "1 SKU 1 hành động giá/ngày": chủ shop sẽ đọc "đặt giá 92.000"
>    cạnh "chạy −20% = 80.000" cùng SKU cùng ngày.
> 2. `schedule_promo` là **một cú đổi giá 10/20/30% thật** nhưng **đi vòng qua
>    anti-oscillation hoàn toàn** (`promo_candidate` không nằm trong
>    `PRICE_CHANGE_KINDS`, và guard không được gọi ở nhánh này) — một cánh cửa
>    không ai gác cho cú cắt 30% trên SKU mà mọi hành động giá KHÁC bị chặn ở
>    ±15% và 1 lần/24h.
>
> Đây là **nợ WIRING, không phải nợ mô hình** (công thức EV lành mạnh):
> task `W-PROMO-PLAN-INTEGRATION`. Điều kiện đóng: một promo candidate và một
> price_suggestion trên cùng SKU phải giải về **đúng một dòng sống sót**.
> Mâu thuẫn này **không giả thuyết — nó được ĐO** bởi
> `tests/decision/test_ols_daily_residue.py::test_widening_the_one_tuple_opens_the_whole_path`,
> test này nới tuple ra và tìm thấy cả hai dòng trên cùng SKU cùng ngày.

### 7.4 Bundle / voucher — `kinds.py:1280`

Hằng số (`:1275–1277`): `BUNDLE_UPLIFT = 0.15`, `BUNDLE_VOUCHER_PCT = 0.05`,
`BUNDLE_MIN_MARGIN = 0.15`.

Khai thác cặp — `_find_bundle_pairs` (`decisions_run.py:1069`): đọc
`purchase.completed` **90 ngày**, gom theo `event_id` thành giỏ, đếm cặp:
```
lift = (cnt/total) / ((cnt_a/total)·(cnt_b/total)) = cnt·total / (cnt_a·cnt_b)
lọc: cnt ≥ 5 và lift ≥ 2.0;  sort lift giảm dần; limit 20
```
Builder:
```
điều kiện: lift ≥ 2.0, pair_cnt ≥ 5, margin_a > 15% và margin_b > 15%
raw_bundle   = 0.95 · (price_a + price_b)
bundle_price = floor(raw_bundle / 1000)·1000
sàn biên lãi: bp_min = sum_cost / (1 − 0.15)
    nếu bundle_price < bp_min:
        bundle_price = ceil(bp_min/1000)·1000
        nếu bundle_price ≥ sum_price → None  (không còn chỗ cho voucher ≥1000đ)
        VOUCHER_MARGIN_FLOOR = APPLIED
    ngược lại VOUCHER_MARGIN_FLOOR = PASS
voucher_amount    = sum_price − bundle_price
bundle_margin_pct = (bundle_price − sum_cost)/bundle_price · 100
EV = 0.15 · pair_cnt · ((price_a − cost_a) + (price_b − cost_b))
subject_id = "<min(a,b)>+<max(a,b)>"
```

---

## 8. GUARDRAILS

### 8.1 Bảng ĐẦY ĐỦ mọi mã guardrail

| Mã | Nơi sinh | Trạng thái có thể | Công thức / điều kiện |
|---|---|---|---|
| `PRICE_CLAMP_20PCT` | `pricing.py:69–77`, `price_optimizer.py:277` | APPLIED / PASS | giá đề xuất bị kẹp vào `[0.8P, 1.2P]` |
| `PRICE_FLOOR_ABOVE_COST` | `price_optimizer.py:279` | PASS (luôn) | bất biến do `feasible_band` bảo đảm (`lo ≥ 1.05c`) |
| `MAX_DISCOUNT_CAP` | `guardrails.py:248–250` | APPLIED / PASS | `price ≥ current·(1 − max_discount_pct/100)`; mặc định **50%** |
| `MAP_FLOOR` | `guardrails.py:258–260` | APPLIED / PASS | `price ≥ map_floors[khớp]` — khớp `product_id` chính xác trước, sau đó **tiền tố DÀI NHẤT** |
| `CHARM_PRICING` | `guardrails.py:274–276` | APPLIED / PASS | làm tròn xuống về đuôi `...900`; **bước LÊN +1000 nếu chạm sàn** |
| `COMPETITOR_MATCH` | `kinds.py:283, 296, 300` | APPLIED / PASS | APPLIED khi thật sự tạo được giá match/beat trên sàn vốn |
| `ANTI_OSC_CLAMP` | `kinds.py:288` | APPLIED | giá match bị kéo về mép dải ±15% (W-UNDERCUT-CLAMP-BAND) |
| `MARKDOWN_COST_FLOOR` | `kinds.py:1187–1189` | APPLIED / PASS | bậc markdown được chọn có bị `ewma_cost` cắt không |
| `PROMO_CAP` | `kinds.py:959` | PASS | mức giảm nằm trong `promo_cap` |
| `STOCK_LIMITED` | `kinds.py:962` | APPLIED | `on_hand < q_promo` (cảnh báo, không chặn) |
| `LEGAL_PROMO_CAP_50` | `kinds.py:1244` | **BLOCKED** | `discount_pct > promo_cap`; **vẫn `presentable=True`** nhờ `presentable_override` (`:1262`) |
| `VOUCHER_MARGIN_FLOOR` | `kinds.py:1339, 1341` | APPLIED / PASS | biên lãi combo ≥ 15% |
| `ROBUST_HOLD` | `kinds.py:390, 517` | APPLIED | optimizer robust chọn giữ giá — *cố ý APPLIED chứ không BLOCKED, vì BLOCKED sẽ giấu dòng khỏi merchant* |
| `ANTI_OSC_HOLD` | `kinds.py:394, 609` | APPLIED | guard chống dao động đã vứt một đề xuất |
| `MARKDOWN_BLOCKS_REORDER` | `decision_plan.py:146` | **BLOCKED** | luật R1 (§9.2) |
| `MARKDOWN_BEATS_REPRICE` | `decision_plan.py:146` | **BLOCKED** | luật R2 |
| `UNDERCUT_BEATS_BANDIT` | `decision_plan.py:146` | **BLOCKED** | luật R3 |
| `UNDERCUT_BEATS_MODEL` | `decision_plan.py:146` | **BLOCKED** | luật R3 |
| `MODEL_BEATS_BANDIT` | `decision_plan.py:146` | **BLOCKED** | luật R3 |
| `BELOW_COST` | `main.py:1093, 1095` | **FAIL** / PASS | `:price-preview`: `candidate_price < ewma_cost` → FAIL (sửa `b63938d`) |

**Ngữ nghĩa trạng thái**:
- `PASS` — luật đã xét, không phải can thiệp.
- `APPLIED` — luật ĐÃ ĐỔI kết quả (giá bị kẹp/nâng/làm tròn), quyết định vẫn hợp lệ.
- `BLOCKED` — **`presentable = False`** (`kinds.py:106`) → dòng bị ẩn khỏi
  `GET /v1/decisions`, trừ khi `presentable_override=True` (chỉ `promo_legal_alert`).
- `FAIL` — chỉ tồn tại ở `:price-preview`: câu trả lời "đừng làm thế".

### 8.2 Thứ tự áp luật giá — `apply_price_rules` (`guardrails.py:219`)

**GUARDRAIL LUÔN THẮNG MODEL** (`:183–185`): các SÀN (cost, MAP, max-discount)
thắng charm rounding — charm phải **bước LÊN** đuôi `...900` kế tiếp nếu làm tròn
xuống sẽ đâm thủng sàn.

```
1. thu thập sàn:
     hard_floor (ewma_cost) nếu > 0                                   (:238-239)
     disc_floor = current·(1 − max_discount_pct/100)  [cần current > 0] (:242-250)
     map_floor  = resolve_map_floor(product_id, map_floors)            (:253-260)
2. effective_floor = max(mọi sàn);  price = max(price, effective_floor) (:262-264)
3. final = round(price)
4. nếu charm_pricing:
     charmed = charm_down(final)
     while charmed < effective_floor:  charmed += 1000                 (:271-272)
     APPLIED nếu charmed ≠ final
```

`charm_down` (`:191`):
```
p < 1000 → trả nguyên (không có ...900 dưới 1000)
ngược lại: ((p + 100) // 1000)·1000 − 100
ví dụ: 125999 → 125900 ; 125000 → 124900 ; 124900 → 124900
```

`resolve_map_floor` (`:203`): khớp `product_id` **chính xác** trước; nếu không,
lấy **tiền tố DÀI NHẤT** khớp. Comment `:176–180`: DB decision **không có cột
brand** (verify migration V003–V006 ngày 2026-08-04) nên MAP floor phải khoá
theo `product_id`.

**Quan trọng**: `get_price_rules` (`store/config.py:111`) **luôn trả về
`max_discount_pct = 50.0`** ngay cả khi tenant chưa cấu hình gì → trong
`decisions_run.py` biến `price_rules` **luôn truthy**, tức `apply_price_rules`
**luôn chạy** và **sàn giảm giá 50% luôn có hiệu lực** cho mọi tenant.
Điều này khớp số đo: `MAX_DISCOUNT_CAP|PASS` xuất hiện 1261 lần trên DB live.

---

## 9. ANTI-OSCILLATION & DECISIONPLAN

### 9.1 Anti-oscillation — `core/guardrails.py:40` `anti_oscillation_verdict`

Bài toán: *"máy có nhảy giá lên xuống liên tục làm khách mất niềm tin không?"*
(câu hỏi nghiệp vụ gốc của `kb_feature:F-ANTIOSC-1`).

Khảo sát repricer Amazon/Shopee 2026-08-04: anti-oscillation là guardrail **bắt
buộc**, và **GUARDRAIL LUÔN THẮNG MODEL** (`:3–6`).

**Hằng số** (`:25–27`): `FREQ_WINDOW_HOURS = 24`, `CUM_BAND_PCT = 15.0`,
`PINGPONG_WINDOW_HOURS = 48`.
`PRICE_CHANGE_KINDS = ("price_suggestion", "slow_mover_alert")` (`:31`) — chỉ hai
kind này vừa **bị gác** vừa **tính vào lịch sử**.

**Ba luật, theo đúng thứ tự** (`:87–131`):

| # | mã `rule` | Điều kiện chặn | `reopen_at` |
|---|---|---|---|
| a | `frequency_24h` | có **bất kỳ** đề xuất nào trong 24h qua | `max(created_at trong cửa sổ) + 24h` |
| b | `band_15pct` | `\|cand − P\|/P · 100 > 15.0` (chỉ khi `single_change_band=True`) | **`None`** — *"chờ không đổi được gì, hứa giờ mở lại là nói dối"* (`:108–110`) |
| c | `pingpong_48h` | `cand > P` **và** có đề xuất GIẢM (`suggested_price < P`) trong 48h qua | `max(created_at của các lần giảm) + 48h` |

Vì luật (a) đã chặn mọi đề xuất thứ hai trong 24h, luật (b) thực chất rút gọn
thành **dải ±15% trên ứng viên duy nhất** (`:98–100`).

`band_used_pct` = tổng biên độ **đã dùng** trong 24h, tính theo % giá hiện tại
(`:72–75`) — "ngân sách đã tiêu".

**Miễn dải cho markdown**: `single_change_band=False` cho `slow_mover_alert`
(`decisions_run.py:705`).
> 📌 **Quyết định `D-M14-ANTIOSC-BAND-EXEMPT-MARKDOWN`** (rail DB): *"nếu dải áp
> cả markdown thì tier 2/3 (20/30%) chết vĩnh viễn — mâu thuẫn trực tiếp
> F-MARKDOWN-1 vs F-ANTIOSC-1. Ladder tự pace 30/60/90 ngày nên không phải dao
> động. `below_cost_alert` KHÔNG bị gate (safety > smoothing)."*
> Code khớp: `below_cost_alert` không nằm trong `PRICE_CHANGE_KINDS`, và
> `decisions_run.py` không gọi guard cho nó (`:1270–1279`).
> Docstring `:17–18`: *"bán dưới vốn là khoản lỗ đang diễn ra; làm mượt không
> được phép che nó."*

**Lịch sử được đếm** — `_get_recent_price_proposals` (`decisions_run.py:573`):
```sql
kind = ANY(PRICE_CHANGE_KINDS)  AND  status <> 'superseded'
AND created_at >= now() − INTERVAL '48 hours'
```
> 🩹 **BUG đã sửa `W-ANTIOSC-COUNTS-SUPERSEDED`** (tồn tại từ V4-M18, sửa
> 2026-08-05, `:580–589`): dòng `status='superseded'` là **kẻ thua của
> DecisionPlan** — chủ shop chưa bao giờ nhìn thấy (`presentable=false`) và
> **chưa có giá nào được đề xuất cho ai cả**. Đếm chúng vào lịch sử khiến một
> dòng audit nội bộ **tiêu mất ngân sách 24h và dải ±15% của SKU**, chặn đề xuất
> THẬT của ngày hôm sau cho một thay đổi chưa từng xảy ra. *"Guard tồn tại để
> bảo vệ NGƯỜI MUA khỏi biến động giá NHÌN THẤY ĐƯỢC, nên lịch sử của nó phải
> đúng bằng những gì đã nhìn thấy."* Các status khác (accepted/rejected/dismissed)
> vẫn tính — chúng đã tới tay merchant.

**Nhường đường cho tín hiệu thị trường** — `_price_proposal_verdict` (`:632`):
```
verdict = _anti_osc_verdict(...)
nếu verdict là None HOẶC đây không phải undercut thật → trả nguyên
undercut THẬT = có competitor_ref  VÀ  COMPETITOR_MATCH APPLIED     (:618-629)
non_bandit = lịch sử bỏ các dòng source == "bandit_shadow"
nếu lịch sử không có dòng bandit nào → trả nguyên
ngược lại: chấm lại verdict trên non_bandit;  bandit_yielded = (verdict mới là None)
```
Và `_markdown_verdict` (`:664`) làm y hệt cho markdown.
> 📌 **`D-BANDIT-YIELD-MD`** (rail DB, 2026-08-05T21:30): *"markdown là bậc 0 của
> price ladder: không được thua ở chỗ undercut (bậc 1) đã thắng. Va chạm thực
> 0/14 nhưng giá **877k VND/ngày/lần** + nhất quán hệ thống."*
> Dải và ping-pong **vẫn áp đủ** trong cả hai trường hợp.

**Cập nhật lịch sử TRONG lượt**: mỗi đề xuất sống sót được **append ngay vào
`recent_props` trong bộ nhớ** (`:1380–1390`, `:1433–1443`) → đề xuất model chặn
được đề xuất bandit ngay trong cùng một lượt chạy.

### 9.2 DecisionPlan — `core/decision_plan.py`

Nguyên tắc (`:7–12`): *"một số hành động mâu thuẫn nhau ở mức nghiệp vụ, và chủ
shop KHÔNG BAO GIỜ được nhìn thấy cả hai phía của một mâu thuẫn ('giảm giá tống
hàng' cạnh 'nhập thêm hàng')."*

**Thang ưu tiên MỘT hành động giá / SKU / ngày** — `price_action_rank` (`:109`):

| rank | Cái gì | Vì sao |
|---|---|---|
| **0** | `slow_mover_alert` + action `markdown` | chương trình thanh lý có chủ đích trên SKU **0 bán/30 ngày**; vốn chôn là chi phí trội |
| **1** | `price_suggestion` có `COMPETITOR_MATCH` APPLIED | **tín hiệu thị trường thật** |
| **2** | `price_suggestion` (model) **hoặc `price_hold`** | tối ưu elasticity nội bộ; "giữ giá, và đây là lý do" **LÀ** câu trả lời giá của mô hình hôm nay |
| **3** | `price_suggestion` `source=bandit_shadow` | **exploration không bao giờ vượt mặt một cam kết** |

`price_hold` ở rank 2 (`:113–118`): nó thua tín hiệu thị trường thật và thua
chương trình thanh lý (*"cả hai đều có căn cứ vững hơn một lời thú nhận bất
định"*), và **thắng exploration** (*"bandit không được lặng lẽ đổi một cái giá mà
mô hình vừa từ chối đổi"*). Rank này không bao giờ bị tranh chấp từ bên trong vì
`price_hold` chỉ phát sinh khi `price_suggestion` đã từ chối.

**Luật liên-kind**:

| Mã | Nội dung | Lý do nghiệp vụ (`RULE_REASONS`, `:72–95`) |
|---|---|---|
| R1 `MARKDOWN_BLOCKS_REORDER` | markdown thắng `replenishment_advice` **và** `stockout_warning` | *"không vừa giảm giá tống hàng vừa nhập thêm; vốn đã kẹt, đơn hàng chờ hết đợt thanh lý"*. Đánh đổi VIẾT RA (`:37–41`): reorder do forecast trên SKU 0-bán **nhiều khả năng là artifact của mô hình** hơn là cầu hồi phục thật; v1 luôn đứng về phía thanh lý. Nếu cầu hồi phục thật thì `units_30d` tăng, SKU thôi trigger slowmover và replenish nối lại ở lượt sau — **tự lành, trễ tối đa 1 ngày**. |
| R2 `MARKDOWN_BEATS_REPRICE` | markdown thắng mọi `price_suggestion` | *"reprice trên hàng chết không tạo ra cầu; ladder làm chủ giá"* |
| R3 `UNDERCUT_BEATS_BANDIT` / `UNDERCUT_BEATS_MODEL` / `MODEL_BEATS_BANDIT` | thang trên chọn 1 | `_price_rule` (`:128–134`) suy mã từ 2 rank |

**Thuật toán** — `resolve_decision_plan` (`:167`):
```
price_actions = [d | price_action_rank(d) is not None]
price_winner  = min(price_actions, key=rank)          ← hoà thì thứ tự dựng quyết định
mọi price_action khác → _supersede(loser, winner, rule)
nếu winner là markdown → mọi kind ∈ REORDER_KINDS chưa thua cũng bị supersede (R1)
```

`_supersede` (`:137`) **ghi cả hai phía** (auditability — mọi quyết định bị đè
đều trả lời được "sao không phải tôi?"):
- kẻ thua: `status="superseded"`, `presentable=False`,
  guardrail `{code: <RULE>, status: "BLOCKED"}`,
  `action_params.plan = {superseded_by_kind, superseded_by_id, rule, reason}`,
  trace nối thêm `"; PLAN: superseded by <kind> [<rule>] <reason>"`;
- kẻ thắng: `action_params.plan.supersedes[] = {kind, decision_id, rule}` (`:210–220`).

Kind **cảnh báo** (`below_cost_alert`, `cost_increase_alert`, `promo_legal_alert`)
đi qua nguyên vẹn (`:50–52`): *"chúng thông báo, chúng không hành động, nên không
thể mâu thuẫn."*

**Rollback story** (`:55–58`): dòng superseded là **trơ** — `outcome_ledger` chỉ
đo `status='accepted'`, API merchant ẩn `presentable=false`.

### 9.3 Explain khi guard làm câm — `anti_osc_hold` (`kinds.py:522`)

> 🩹 **W-ANTIOSC-SILENCE**, đo live 2026-08-05 trên các tenant pmode: **15/15 đề
> xuất Lerner bị dải ±15% vứt bỏ ở CẢ HAI chế độ giá và KHÔNG GÌ tới tay chủ
> shop** — guard là nguồn im lặng **lớn hơn** cả robust-hold mà V4-M18-PMODE
> được xây để giải thích (`:531–537`).

**Tái dùng carrier `price_hold` một cách cố ý** (`:539–552`) — kind `price_hold`,
action `hold_price`, phân biệt bằng `action_params.hold_reason`:
- dedup `price_hold:<sku>:<ngày>` → **tối đa 1 dòng hold/SKU/ngày**, không spam;
- `price_hold` **không** nằm trong `PRICE_CHANGE_KINDS` → lời giải thích cho một
  thay đổi bị chặn **không được tự tiêu ngân sách 48h** (nếu không sẽ chặn đề
  xuất thật của ngày mai);
- ladder xếp nó ở bậc mô hình → undercut thật hay markdown sống sót vẫn thắng.

**Điều kiện phát ra** (`decisions_run.py:1646–1706`) — cả 4:
```
1. có ít nhất 1 đề xuất bị guard vứt (antiosc_blocked không rỗng)
2. blocked_rank ≤ 2      ← chỉ CAM KẾT (markdown/undercut/model) mới được giải
                            thích; bandit_shadow (rank 3) là exploration,
                            "không ai đang chờ nó" (:1636-1639)
3. survivor_rank > 2     ← không còn hành động giá cam kết nào sống sót
                            (bandit sống sót KHÔNG tính là câu trả lời, :1640-1644)
4. not already_told      ← chưa có đề xuất non-bandit CÙNG NGÀY với giá chênh <1đ
                            (dedup giữ 1 gợi ý/SKU/ngày nên chạy lại cùng ngày sẽ
                             đề xuất đúng con số cũ; một dòng hold cạnh nó đọc
                             ra như mâu thuẫn, không phải giải thích) (:1664-1680)
```
Đề xuất được chọn để giải thích = `min(antiosc_blocked, key=rank)` (`:1655–1662`).

**Payload explain** — `build_anti_osc_explain` (`price_explain.py:309`):
```
band_remaining_pct = max(0, 15.0 − band_used_pct)
allowed_price_min  = P·(1 − 0.15);   allowed_price_max = P·(1 + 0.15)
steps_needed       = ceil(|move_pct| / 15.0)   ← CHỈ cho rule band_15pct
reopen_in_hours    = giờ từ now tới reopen_at (None với rule band)
recent_changes     = tối đa 5 dòng (_MAX_RECENT_IN_EXPLAIN, :298)
```
Ba câu tiếng Việt riêng cho 3 rule (`:375–421`), ví dụ rule `band_15pct`:
> *"Đề xuất giảm giá 1.221.000 → 900.000 (−26.3%) vượt biên độ cho phép ±15% mỗi
> 24h (khoảng giá được phép: 1.037.850 – 1.404.150 VND). Hệ thống GIỮ giá
> 1.221.000 VND; muốn tới 900.000 phải đi 2 bước, mỗi bước cách nhau 24h — chờ
> không làm biên độ rộng ra."*

`confidence = 0.9` cho `anti_osc_hold` (`:608`) — *"chắc chắn cơ học như
`below_cost_alert`: trạng thái guard được ĐO, không ước lượng — không rubric
elasticity nào áp vào đây."*

### 9.4 Dedup

`dedup_key(kind, subject_id)` = `"{kind}:{subject_id}:{YYYY-MM-DD UTC}"`
(`kinds.py:1266`). Ràng buộc DB: `UNIQUE (project_id, dedup_key)`.

`_insert_decision` (`decisions_run.py:935`):
- dòng `status='superseded'` được thêm hậu tố `":superseded"` + `":<source>"`
  (`:953–955`) — *"dòng audit không được va vào dòng hành động"*.
- **Đếm dedup đúng** (`:940–948`): với `ON CONFLICT DO NOTHING`, asyncpg **không
  raise**; command tag là `"INSERT 0 0"`. Parse tag là cách DUY NHẤT để đếm đúng.
  🩹 Bug tìm ra 2026-08-04: **mọi xung đột đều bị báo là "created"**.

---

## 10. EXPERIMENT GATE / AUTO-APPLY (M20)

*(chi tiết grader ở §10.3)*

### 10.1 Máy quyết định — `core/experiment_gate.py`

Câu hỏi nghiệp vụ (`:4–6`): *"một thay đổi (promote LTR model, bật
price-optimizer, canary policy mới) có được TỰ ĐỘNG áp dụng không?"* — trả lời
bằng **BẰNG CHỨNG anytime-valid**, không bằng cảm giác.

**Ba phán quyết**:
- **FIRE** — treatment hơn control ĐÃ CHỨNG MINH (`CS lower bound > min_effect`)
  VÀ đủ mẫu/chu kỳ ⇒ auto-apply.
- **BLOCK** — chưa đủ bằng chứng, **máy in ra lý do cụ thể**.
- **KILL** — arm vi phạm: (a) CS chứng minh HẠI (`ci_hi < 0`), hoặc (b) tổng
  thiệt hại ước tính vượt `loss_budget` ⇒ cắt NGAY trong chính chu kỳ phát hiện.

**Cấu hình `GateConfig`** (`:61`, sống trong `experiment_registry.config["gate"]`):

| Trường | Mặc định | Dòng | Ý nghĩa |
|---|---|---|---|
| `alpha` | **0.05** | `:70` | mức sai lầm của confidence sequence |
| `min_cycles` | **8** | `:71` | độ phủ mùa vụ/segment tối thiểu |
| `min_samples_per_arm` | **400** | `:72` | chống FIRE non trên vài chu kỳ may mắn |
| `min_effect` | **0.0** | `:73` | hiệu ứng tối thiểu đáng FIRE |
| `loss_budget` | **10.0** | `:74` | trần thiệt hại tuyệt đối (đơn vị reward) |
| `loss_trigger_frac` | **0.8** | `:75` | kích hoạt SỚM tại 0.8×budget |
| `reward_lo` / `reward_hi` | **0.0 / 1.0** | `:76–77` | biên reward (CS cần chặn) |

Validate ở `__post_init__` (`:79–96`): `0 < alpha < 1`, `loss_budget > 0`,
`0 < loss_trigger_frac ≤ 1`, `min_cycles ≥ 1`, `min_samples_per_arm ≥ 1`,
`reward_lo < reward_hi`, `min_effect ≥ 0`. `from_dict` **bỏ qua key lạ** (`:98`).

**Loss budget SIGNED** (`:26–36`, `:178`):
```
_loss_signed += (mean_control − mean_treat) · n_treat        mỗi chu kỳ
cum_loss     = max(0, _loss_signed)                          đọc ra ngoài
```
Cố ý dùng dấu: *"nhiễu hai chiều tự triệt dưới null, tránh kill oan vì cộng dồn
một chiều."*
Trigger SỚM tại `0.8 × budget` vì gate chỉ cắt được **ở ranh giới chu kỳ** — nếu
trigger đúng tại budget thì chu kỳ vượt đã tràn qua trần.

**Hai quy tắc sizing caller phải giữ** (`:28–36`):
- **S1**: `budget ≥ 5 × thiệt-hại-tệ-nhất-một-chu-kỳ` — một chu kỳ nhảy quá 20%
  budget thì không máy per-cycle nào cứu được (lỗi chọn cỡ chu kỳ).
- **S2**: `trigger ≥ z_{1−α} · σ_cycle · √T_horizon` — budget-guard đo ƯỚC LƯỢNG
  nhiễu (random walk dưới null, law-of-iterated-logarithm rồi cũng chạm mọi
  ngưỡng cố định); phần kill-oan do budget là **phí bảo hiểm trần-thiệt-hại,
  KHÔNG được α của CS bảo kê**. `α` chỉ bảo kê nhánh `harm_proven`.

**Thứ tự quyết định mỗi chu kỳ** — `_decide` (`:201`) — **KILL trước FIRE trước
BLOCK** (fail-closed):
```
1. if cum_loss > loss_budget·loss_trigger_frac  → KILL "loss_budget_exceeded"
2. if ci_hi < 0.0                                → KILL "harm_proven"
3. FIRE khi KHÔNG thiếu điều nào:
     cycles_seen ≥ min_cycles
     n_treat     ≥ min_samples_per_arm
     n_control   ≥ min_samples_per_arm
     ci_lo       >  min_effect
   → "improvement_proven: ci_lo=… > min_effect=…, cycles=…, n=(…,…)"
4. ngược lại BLOCK với danh sách lý do nối bằng "; ":
     insufficient_cycles / insufficient_treat_samples /
     insufficient_control_samples / ci_not_conclusive
```
FIRE/KILL là **TERMINAL** — nạp thêm chu kỳ ⇒ `RuntimeError` (`:156–159`).
Thiếu event ở một arm ⇒ `ValueError` (`:161–167`): *"thiếu arm là bug pipeline dữ
liệu, gate không đoán bù."*

**Cơ chế thống kê**: `DiffConfSeq` (`libs/common/confseq.py`) — empirical-Bernstein
confidence sequence per-arm trên **từng event reward bị chặn** `[reward_lo,
reward_hi]`, CI của hiệu ghép bằng **union bound**. Nhờ đó **nhìn-mỗi-chu-kỳ-
quyết-ngay** vẫn giữ `P(FIRE oan | không hơn) ≤ α` và `P(KILL oan | không hại) ≤ α`.
`CycleObservation` mang **reward từng EVENT**, không phải aggregate — *"aggregate
mean/chu-kỳ là mất power"* (`:107–118`).

`GateDecision` (`:120`) có 11 field, **mọi field đi thẳng vào audit row**:
`cycle, decision, reason, diff_mean, ci_lo, ci_hi, n_treat, n_control, cum_loss,
loss_budget, terminal`.

### 10.2 Runner + scheduler — `jobs/experiment_gate_run.py`

**Luồng** (`:3–6`): `experiment_registry` (arms + `config["gate"]`) →
`impression_log` (event/chu-kỳ/arm) → replay `ExperimentGate` theo thứ tự chu kỳ
→ mỗi quyết định 1 row `experiment_gate_audit` (**UPSERT — replay idempotent**)
→ FIRE/KILL đổi `status` registry.

**Flag `MINIAI_EXPERIMENT_AUTO_APPLY`** (mặc định `"0"` = OFF):
- **OFF**: gate **vẫn chạy + audit đầy đủ** (kể cả verdict FIRE — dry-run có vết),
  nhưng KHÔNG apply: `applied=false`, `auto_apply_enabled=false`.
- **ON** (hoặc `auto_apply=True` tường minh): FIRE ⇒ `status='applied'`.
- **KILL KHÔNG bị flag gate** (`:15–17`): kill-switch là hành-động-an-toàn
  fail-closed; `status='killed'` luôn được ghi khi runner được GỌI.
  *"Một cơ chế an toàn nằm sau flag tiện-nghi là cơ chế an toàn chết."*

**Scheduler** `start_experiment_gate_loop` (`:312`) trong worker process, mỗi
`EXPERIMENT_GATE_INTERVAL_S` giây (**mặc định 300**, `0` = tắt). Ba tính chất
BẮT BUỘC giữ (`:22–42`):
1. Flag tổng ≠ `"1"` ⇒ scheduler **không chạy vòng nào** (log 1 dòng lúc boot rồi
   `return`). *"Bật máy tự quyết là hành-động-có-chủ-ý của người vận hành, không
   phải hệ quả của việc deploy worker."*
2. **Một experiment hỏng KHÔNG giết loop**: mỗi experiment bọc `try/except`
   riêng, đếm vào `failed`; `CancelledError` **không bị nuốt**.
3. **Hai worker KHÔNG double-FIRE**: mỗi experiment chấm dưới **advisory lock
   Postgres** + **đọc LẠI status TRONG khoá** (double-checked). Khoá mang cả
   `project_id` → experiment trùng tên của 2 tenant không chặn nhau
   (`W-EXPREG-PK-TENANT`).

Ba lý do skip (`run_gate_scheduled`, `:177`, rẻ→đắt): `locked` /
`not-active(status=…)` / `no-new-evidence(cycle<=N)`.

Quan sát: mỗi tick ghi 1 dòng `job_runs` job `experiment_gate_sweep`
(success bị throttle 300s, failure **luôn** ghi). *"Scheduler có sống không"* =
`SELECT * FROM job_runs WHERE job_name='experiment_gate_sweep'`.

> 📌 `D-M20-GATE-SEED-NOT-WAIT` (`:49–51`): **KHÔNG chờ click thật** — chứng minh
> bằng seed ~5k event đáp-án-cài-sẵn (`scripts/seed_m20_experiment.py` +
> `scripts/grade_m20_gate.py`). Chỉ PROMOTE-lên-traffic-thật mới cần số thật.

### 10.3 Vai trò các grader

<!-- GRADERS_PLACEHOLDER -->

---

## 11. OUTCOME LEDGER — VÒNG PHẢN HỒI (VAI C)

`jobs/outcome_ledger.py` — chạy mỗi `OUTCOME_INTERVAL` giây
(**mặc định 604800 = 1 tuần**, `:29`, `:495`), chạy ngay 1 lần lúc start.

### 11.1 Cửa sổ đo THEO KIND (không phải 30 ngày phẳng)

`OUTCOME_WINDOW_DAYS_BY_KIND` (`:34–41`):

| kind | cửa sổ | Lý do (`:7–14`) |
|---|---|---|
| `price_suggestion` | **14** ngày | co giãn cầu lộ ra nhanh; chờ cả tháng chỉ làm chậm việc học |
| `promo_candidate` | **14** ngày | như trên |
| `stockout_warning` | **14** ngày | như trên |
| `replenishment_advice` | **21** ngày | LT 7 + review 7 + bán-hết 7 |
| `slow_mover_alert` | **90** ngày | ladder 3 bậc cách nhau 30d — đo **cuối chiến dịch**, không giữa chừng |
| mặc định | **30** ngày | giữ hành vi cũ |

Cùng cửa sổ đó vừa **gate điều kiện đủ tuổi** vừa **là cửa sổ before/after được
đo** — không lệch nhau (`:31–33`).

SQL cutoff thô ở cửa sổ **NHỎ NHẤT** (14) để tránh quét dòng còn tươi (`:143–146`),
rồi kiểm tra tuổi chính xác từng dòng (`:171–175`).

**`OUTCOME_EXEMPT_KINDS = ("price_hold",)`** (`:59`) — 🩹 **W-HOLD-OUTCOME**:
hold là **PHI-hành-động**, `expected_value` NULL theo cấu tạo, nên scan cũ ghi 1
dòng trơ sau 30 ngày với **predicted NULL VÀ realized NULL** — nhiễu thuần trong
bảng mà mọi báo cáo EV đều đọc.
> **Đánh đổi VIẾT RA** (`:51–58`): phương án bị loại là outcome riêng cho hold
> (`"hold_confirmed"` khi giá thật sự không đổi trong cửa sổ). *"Nó đo sự KHÔNG
> hành động của merchant, không phải quyết định của ta: giá đứng yên là MẶC ĐỊNH,
> nên một dòng 'thành công' sẽ kiếm được bằng cách không làm gì và thổi phồng mọi
> metric độ chính xác tương lai."* Phép đo trung thực của một hold là EV/CVaR
> phản thực đã lưu sẵn trong `action_params.explain`.
> Điều kiện mở lại: outcome của hold phải so với giá blocked/lerner **thực sự
> được áp sau đó** (cần join `price_history`).

### 11.2 Công thức realized EV

**`price_suggestion`** (`:186–232`):
```
new_price  = action_params.suggested_price
old_price  = price_history mới nhất TRƯỚC created_at
before     = SUM(units) trong [created_at − window, created_at)
after      = SUM(units) trong [created_at, created_at + window)
cost       = cost_state.ewma_cost hiện tại

realized_ev = (new_price − cost)·after  −  (old_price − cost)·before
```
Thiếu bất kỳ đầu vào nào → `skipped_no_data`, `note = "missing price/units/cost data"`.

**`replenishment_advice`** (`:250–253`): `realized_ev = predicted_ev`,
`note = "proxy_v1"` — **CHƯA phải phép đo thật** (spec §4.7 v1).

**`slow_mover_alert`** — `_markdown_realized_ev` (`:280`), 🩹 **W-OUTCOME-MARKDOWN-EV**:
```
realized = (revenue_after − cost·units_after)              ← biên lãi kiếm được
         − (price_before − cost)·units_before              ← phản thực
         + units_extra · cost · (0.25/12) · (window/30)    ← phí giữ hàng tránh được

units_extra = max(0, units_after − units_before)
window = 90 ngày → hệ số (window/30) = 3 tháng
```
**Ba lựa chọn phản thực, chọn cái thứ ba, viết rõ lý do** (`:299–310`):
- *"nó sẽ chẳng bán được gì"* (hấp dẫn vì SKU trigger trên 0-bán-30d) → **ghi công
  cho markdown từng đồng biên lãi và biến ledger thành cỗ máy tự khen**;
- *"những gì forecast nói nó sẽ bán"* → cần join forecast và **chấm dự đoán của ta
  bằng chính dự đoán của ta**;
- ✅ **cửa sổ before/after đối xứng** — đo được từ mình `sales_daily`, và là **CÙNG
  quy ước `price_suggestion` đã dùng**: *"một quy ước cho cả ledger hơn một quy
  ước thông minh cho mỗi kind."*

**Vốn được giải phóng CỐ Ý không tính là lợi nhuận** (`:311–315`): chuyển hàng
thành tiền là nghiệp vụ **bảng cân đối**; biên lãi trên số hàng đó đã nằm trong
`margin_actual`. Cái thật sự kiếm được là **phí giữ hàng không còn phải trả**,
định giá bằng **CÙNG rate 25%/năm** mà alert đã dùng để tuyên bố EV của nó
(`HOLDING_RATE_ANNUAL = 0.25`, `:66`).

**Hai trường hợp trả NULL kèm lý do, KHÔNG ghi 0** (`:352–361`):
- `days_after == 0` → *"ZERO ROWS không phải 'thanh lý không bán được gì' mà là
  'rollup ngày chưa bao giờ phủ cửa sổ này'. Ghi realized_ev=0 cho nó sẽ lặng lẽ
  kéo mọi báo cáo EV về 0 bằng những dòng có nghĩa là 'chúng tôi chưa nhìn'."*
- `units_after > 0` nhưng `revenue_after ≤ 0` → *"units mà không có tiền = rollup
  chưa điền revenue. Suy revenue từ giá ladder là BỊA dữ liệu."*

**Predicted EV của markdown** — `markdown_predicted_ev` (`:74`), cũng W-OUTCOME-MARKDOWN-EV:
> `expected_value.amount` của `slow_mover_alert` là `capital_locked +
> holding_cost_month` — **ĐỘ LỚN CỦA VẤN ĐỀ, không phải giá trị của hành động**.
> So một profit-delta 90 ngày với nó (đúng cái `advice_scorecard` đang làm) là
> **khập khiễng đơn vị không bao giờ hội tụ**.

Tuyên bố EV thật nằm ở **bậc ĐƯỢC CHỌN** của ladder:
```
step = schedule[tier − 1]
nếu step.ev tồn tại:
    factor = window/30  nếu ev_model == "elasticity"   ← EV elasticity là /30 NGÀY
             1.0        nếu ev_model == "clear_rate_v1" ← đã là toàn chiến dịch
    predicted = step.ev · factor
    basis = "tier{tier}.ev[{ev_model}]x{factor}"
ngược lại nếu step.ev_vs_salvage:  predicted = ev_vs_salvage,  basis = "tier{tier}.ev_vs_salvage"
ngược lại: fallback expected_value.amount, basis = "expected_value.amount(capital_locked)"
```
Ladder legacy (không có `tier`) → `step = schedule[0]`, `tier = 1` (`:112–116`).
Mọi truy cập đều **type-check trước** (`:104–105`): *"một job hàng tuần không được
chết vì MỘT dòng dị dạng và bỏ mọi quyết định khác không đo."*

### 11.3 Idempotency + bảng đích

Truy vấn ứng viên (`:149–163`):
```sql
status = 'accepted'  AND  created_at <= cutoff
AND kind <> ALL(OUTCOME_EXEMPT_KINDS)
AND NOT EXISTS (SELECT 1 FROM outcome_ledger ol WHERE ol.decision_id = d.decision_id)
```
→ **một quyết định chỉ được đo đúng một lần**, chạy lại job là no-op.

Bảng `outcome_ledger(id, project_id, decision_id, predicted_ev BIGINT,
realized_ev BIGINT, measured_at, note, window_days)` — RLS theo tenant.
`predicted_ev`/`realized_ev` được `round()` trong Python (`:269–271`): *"công thức
float không bao giờ được phụ thuộc vào ép kiểu int của driver."*

Dòng outcome **luôn được ghi kể cả khi `realized_ev` NULL** (`:255–273`) — `note`
nói rõ thiếu đầu vào nào, để một dòng NULL vẫn chẩn đoán được thay vì câm.

### 11.4 Đọc ra: `advice_scorecard`

`GET /v1/decisions:insights?kind=advice_scorecard`
(`main.py:993` → `store/insights.py:95`):
```
theo kind: n_created, n_accepted (status='accepted'), n_rejected (status='dismissed'),
           n_measured, ev_predicted_sum, ev_measured_sum,
           ev_gap_abs = |predicted − measured|   (None khi n_measured = 0)
+ worst_kind = kind có ev_gap_abs lớn nhất
```
0 dòng outcome là **HỢP LỆ** (`:100–103`) — quyết định phải đủ tuổi cửa sổ của
kind mình mới được đo; API trả 0 kèm note giải thích.

---

## 12. BẢNG TỔNG HỢP

| Thuật toán | Chạy khi nào | Tham số chính | Guardrail liên quan | Cách đo |
|---|---|---|---|---|
| EWMA cost | mỗi `cost.recorded` (rollup 300s) | α = 0.3 | — | `tests/decision/test_state_rollup.py` |
| OLS daily elasticity | đầu mỗi `run_decisions_once`, mọi SKU | `MIN_DAYS_OLS=30`, ≥2 giá, kẹp `[−5,−0.2]` | — | `scripts/grade_elasticity.py`, `tests/decision/test_elasticity_ols.py` |
| EB shrinkage | pass 2 cùng lượt | `τ²` MoM, floor `1e-6` | — | `test_elasticity_refresh.py` |
| Lerner / cost-plus | mọi `price_suggestion` (mode lerner) | `target_margin=0.15`, `clamp_pct=0.20` | `PRICE_CLAMP_20PCT` | `make eval-decision` (kb 4,5,6) |
| Mean-CVaR optimizer | `pricing_mode=robust` + có `eps_sd` | λ=0.15, α=0.10, 161×161, `floor_margin=0.05` | `PRICE_CLAMP_20PCT`, `PRICE_FLOOR_ABOVE_COST` | `scripts/grade_optimizer.py`, `test_price_optimizer.py` |
| pricing_policy | mọi `price_suggestion` | env kill-switch + `project_config.pricing_mode` | — | `test_pricing_mode.py` (41), `test_price_optimizer_wire.py` (95) |
| explain-hold | robust giữ giá & Lerner muốn đổi | `MOVE_THRESHOLD=0.05`, CI 80%, 14 bisect | `ROBUST_HOLD` | `test_pricing_mode.py` |
| Thompson bandit | mọi SKU có cost+price+units+elasticity | grid ±10% 5 arm, σ₀ = max(0.5\|μ₀\|,1), ngưỡng 2% | mọi rule của `apply_price_rules` | `test_price_bandit.py` |
| Newsvendor legacy | replenish khi không có scenario | `holding_rate=0.25`, `review=7`, CR≤0.99, lot=5 | — | `scripts/grade_newsvendor.py`, `test_replenish.py` |
| Newsvendor scenario | có forecast LTD artifact | quantile `{CR,0.5,0.9}`, round-half-up theo pack | — | `test_scenario_newsvendor.py` |
| ROP/SS/DOI | luôn khi có `avg_daily` | z ∈ {1.28,1.65,2.33}, LT=7, σ_LT=2.0 | — | `test_replenish_plus.py`, `GET :replenish-plan` |
| Markdown ladder | `on_hand>0` ∧ `units_30d=0` ∧ `history≥30` | tier 30/60/90 → 10/20/30%, sàn `ewma_cost` | `MARKDOWN_COST_FLOOR` + rules | `scripts/grade_markdown_ev.py`, `test_markdown_ladder.py` |
| Promo EV (A6) | **không bao giờ** (tuple `("ols",)`) | 10/20/30%, cap 50% | `PROMO_CAP`, `STOCK_LIMITED` | `test_promo_candidate.py`, `test_ols_daily_residue.py` |
| apply_price_rules | mọi đề xuất giá | `max_discount 50%` (mặc định), MAP, charm | 3 mã tương ứng | `test_guard_ext.py` |
| Anti-oscillation | mọi `PRICE_CHANGE_KINDS` | 24h / ±15% / 48h | `ANTI_OSC_HOLD`, `ANTI_OSC_CLAMP` | `test_anti_oscillation.py`, `test_antiosc_explain.py` |
| DecisionPlan | cuối mỗi vòng SKU | rank 0/1/2/3 + R1/R2/R3 | 5 mã `*_BEATS_*` / `*_BLOCKS_*` | `test_decision_plan.py` |
| Experiment gate | mỗi 300s (nếu flag ON) | α=.05, cycles≥8, n≥400, budget 10, trigger .8 | — | `grade_m20_gate.py`, `grade_confseq.py`, `grade_canary.py` |
| Outcome ledger | mỗi tuần | 14/21/30/90d, rate 0.25/năm | — | `test_outcome_ledger.py`, `test_markdown_outcome.py` |

---

## 13. SỐ ĐO THẬT (đo 2026-08-07, DB `miniai_decision` live)

### 13.1 Gate vàng

```
$ .venv/bin/python eval/decision_eval.py
...
DECISION-EVAL 24/24
DECISION-EVAL PASS
```
24 kịch bản vàng (`eval/decision_eval.py`): 3 elasticity · 4 pricing/EV ·
5 newsvendor · 7 kind-phải-nổ · 5 kind-KHÔNG-được-nổ.

> ⚠ **`make eval-decision` HỎNG trên python hệ thống**: target gọi `python3`
> trần, mà `python3` global không có `numpy` → `ModuleNotFoundError`. Phải chạy
> `.venv/bin/python eval/decision_eval.py`. (Đo trực tiếp 2026-08-07.)

### 13.2 Phân bố `elasticity.method` (toàn DB, 94 project)

| method | số dòng | avg eps | avg eps_sd | avg r² | avg n_points |
|---|---|---|---|---|---|
| `pooled_prior` | **1605** | — | NULL | — | — |
| `ols_daily` | **421** | **−1.317** | **1.405** | **0.410** | — |
| `ols` | **0** | — | — | — | — |

`ols_daily` toàn cục: `min(eps) = −2.01`, `max(eps) = −0.57`.

**Phát hiện quan trọng**: `avg(eps_sd) = 1.405 > |avg(eps)| = 1.317` — **độ bất
định của eps LỚN HƠN chính giá trị eps**. Đây chính là lý do cơ học khiến chế độ
`robust` gần như luôn GIỮ GIÁ, và là bối cảnh của
`D-ROBUST-ON-WITH-EXPLAIN` (*"đo thật: OFF 6 gợi ý, ON 0"*).

Tenant `demoshop`: 109 `ols_daily` (avg eps **−0.574**, avg eps_sd 1.291,
avg r² 0.433, avg n_points **81.8**) + 15 `pooled_prior` (avg eps **−0.574** —
xác nhận §3.2: pooled_prior mang μ quần thể, không phải −1.3).

**Phát hiện thứ hai**: **124/124 SKU của `demoshop` có `eps > −1.0`** → nhánh
Lerner rơi hết về **`cost_plus`** (`P* = ewma_cost × 1.15`), không dùng công thức
`c·eps/(1+eps)` lần nào. Toàn DB: 126 dòng `eps > −1` / 1900 dòng `eps ≤ −1`.

### 13.3 Phân bố quyết định

| kind | status | số dòng |
|---|---|---|
| `replenishment_advice` | open | 1547 |
| `price_hold` | open | 1247 |
| `stockout_warning` | open | 1009 |
| `price_suggestion` | superseded | 757 |
| `price_suggestion` | open | 683 |
| `bundle_suggestion` | open | 35 |
| `promo_legal_alert` | open | 15 |
| `slow_mover_alert` | accepted | 12 |
| `stockout_warning` | accepted | 9 |
| `below_cost_alert` | open | 8 |
| `price_suggestion` | accepted | 7 |
| `bundle_suggestion` | accepted | 4 |
| `cost_increase_alert` | open | 3 |
| `slow_mover_alert` | open | 3 |
| `promo_legal_alert` | accepted | 2 |
| `price_hold` | accepted | 2 |
| `replenishment_advice` | accepted | 1 |
| **`promo_candidate`** | — | **0** ✅ (xác nhận §7.3: đường chết đúng thiết kế) |

`price_hold` theo `hold_reason`: **`anti_oscillation` 1191** · `robust_uncertainty` 43 ·
NULL (dòng cũ trước khi có field) 15.
→ **96,5% mọi lời giải thích giữ giá là do GUARD, không phải do optimizer** —
đúng như W-ANTIOSC-SILENCE đã dự báo.

`price_suggestion` theo `source`: `bandit_shadow` **1160** · NULL (model) 287.
`price_suggestion` theo `price_source`: NULL 1438 (fleet mode lerner, không gắn
key) · `robust_optimizer` 6 · `lerner` 3.

### 13.4 Guardrail thực tế

| code | status | số lần |
|---|---|---|
| `MAX_DISCOUNT_CAP` | PASS | 1261 |
| `ANTI_OSC_HOLD` | APPLIED | 1191 |
| `MODEL_BEATS_BANDIT` | BLOCKED | 757 |
| `PRICE_CLAMP_20PCT` | APPLIED | 205 |
| `CHARM_PRICING` | APPLIED | 115 |
| `PRICE_CLAMP_20PCT` | PASS | 81 |
| `ROBUST_HOLD` | APPLIED | 58 |
| `VOUCHER_MARGIN_FLOOR` | PASS | 39 |
| `COMPETITOR_MATCH` | APPLIED | 22 |
| `LEGAL_PROMO_CAP_50` | BLOCKED | 17 |
| `MARKDOWN_COST_FLOOR` | PASS | 15 |
| `CHARM_PRICING` | PASS | 8 |
| `PRICE_FLOOR_ABOVE_COST` | PASS | 6 |

`MODEL_BEATS_BANDIT` là luật DecisionPlan hoạt động nhiều nhất (757/757 dòng
`price_suggestion` superseded) — **mỗi ngày mỗi SKU, đề xuất model đè bandit**.

### 13.5 `skipped_by_reason` — danh sách ĐỦ 6 lý do

| Lý do | Sinh ở | Ý nghĩa |
|---|---|---|
| `no_cost` | `decisions_run.py:1282, 1314, 1393` | không có `cost_state.ewma_cost` → không tính được biên lãi. **Đếm tối đa 3 lần/SKU** (below_cost, cost_increase, price_suggestion) |
| `no_stock` | `:1512, 1547` | không có `stock_state.on_hand_qty`. Đếm tối đa 2 lần/SKU (replenish, stockout) |
| `insufficient_history` | `:1310` | `cost_ledger` có < 6 dòng → không dựng được `last3/prev3` |
| `anti_oscillation` | `:1368, 1421, 1579` | guard vứt đề xuất (model / bandit / markdown) |
| `bandit_yield_to_undercut` | `:1354` | undercut thật được thông qua sau khi bỏ qua lịch sử bandit |
| `plan_conflict` | `:1710` | DecisionPlan đè một ứng viên (mỗi resolution 1 đơn vị) |

**Đo thật `demoshop`** (`kv_state['decisions_last_skips:demoshop']`):
```json
{"no_cost": 30, "no_stock": 2, "plan_conflict": 89, "anti_oscillation": 127, "insufficient_history": 1}
```
Đọc ra: 127 lần guard chặn + 89 lần plan conflict trên 124 SKU — hệ thống **chặn
nhiều hơn phát**, đúng triết lý "guardrail luôn thắng model".

Các tenant khác: `demo` `{no_stock:2, plan_conflict:45, anti_oscillation:76, insufficient_history:1}`;
`p1` `{no_cost:1878, no_stock:1734, plan_conflict:1, anti_oscillation:1, insufficient_history:478}`.

> ⚠ Khoá `decisions_last_skips` (không hậu tố project) vẫn còn 1 dòng CŨ trong
> `kv_state` — `{"no_cost":972,"no_stock":780,"anti_oscillation":505,"insufficient_history":236}`.
> Comment `decisions_run.py:1859–1860` nói khoá này "đã bị bỏ"; code hiện tại
> không ghi nó nữa nhưng **không ai xoá dòng cũ**. Không có gì đọc nó
> (`main.py` `:decisions:stats` đọc khoá có hậu tố), nhưng nó là rác gây hiểu nhầm.

### 13.6 Outcome ledger

| window_days | số dòng | có `realized_ev` |
|---|---|---|
| 14 | 1 | 1 |
| 90 | 12 | 12 |

→ mọi dòng đã đo đều có số thật (không có dòng NULL câm).

### 13.7 Cấu hình tenant live

```
demo       : charm_pricing=true, max_discount_pct=40, service_level=0.95, map_floors={}
demoshop   : pricing_mode="lerner", promo_cap_pct=50
m14test    : charm_pricing=true, max_discount_pct=25
pmode-live-lerner / pmode-demo-lerner : pricing_mode="lerner"
pmode-live-robust / pmode-demo-robust / pmode-kill-robust : pricing_mode="robust"
```
Fact rail `demo.pmode.tenants`: *"pmode-live-lerner (mode1, 15 price_suggestion) +
pmode-live-robust (mode2, 14 price_hold + explain … mode1 đề xuất 1.341.000
EV+2,1tr; mode2 giữ 1.221.000 vì CVaR −15,2%/tháng, cần 417 quan sát)."*

### 13.8 Lệnh kiểm chứng

```bash
cd /home/hai-soft/projects/icpp/mecom/project

# gate vàng thuần offline (KHÔNG dùng `make eval-decision` — thiếu numpy)
.venv/bin/python eval/decision_eval.py            # kỳ vọng: DECISION-EVAL 24/24 PASS

# chuỗi sống (LUẬT-0: chưa đo chuỗi = chưa xong)
make check-apis PROJECT=demoshop

# suite đơn vị của service decision (456 test, commit b63938d)
.venv/bin/python -m pytest tests/decision -q

# API
curl -XPOST .../v1/decisions:run                  # trả created / skipped_by_reason /
                                                  # superseded_plan / price_source / price_hold /
                                                  # anti_osc_hold
curl -XPOST .../v1/decisions:price-preview -d '{"product_id":"X","candidate_price":90000}'
curl     .../v1/decisions:replenish-plan?product_id=X
curl     .../v1/decisions:insights?kind=advice_scorecard
curl -XPUT .../v1/config -d '{"pricing_mode":"robust"}'   # rollback = 1 API call, không deploy
```

<!-- GRADER_CMDS_PLACEHOLDER -->

---

## 14. CHƯA CHẮC (không đoán)

1. **CHƯA CHẮC: `eoq()` (`core/econ/replenish.py:66`) có caller sản phẩm nào
   không.** Grep `eoq(` trong `services/decision/app/core/kinds.py`, `jobs/`,
   `store/`, `main.py` không ra kết quả. Có thể chỉ là API cho tương lai hoặc chỉ
   dùng trong test. Không dám khẳng định "đang chạy".

2. **CHƯA CHẮC: `estimate_elasticity` (segment-based, `method="ols"`) có bao giờ
   chạy trong production không.** Comment `kinds.py:61–63` nói caller duy nhất là
   `_refresh_elasticity` và "hiện chỉ test dùng". DB xác nhận **0 dòng
   `method='ols'`**. Nhưng tôi không grep hết mọi entrypoint để chứng minh
   `_refresh_elasticity` không bao giờ được gọi ở runtime.

3. **CHƯA CHẮC: nội dung chính xác của `libs/common/confseq.py::DiffConfSeq`.**
   Tôi đọc hợp đồng qua docstring của `experiment_gate.py` (empirical-Bernstein
   per-arm + union bound) chứ chưa đọc trực tiếp file cài đặt trong lượt này.
   Công thức bound cụ thể xem §10.3.

4. **CHƯA CHẮC: `service_level` (config) vs `service_level_floor` (tham số
   `order_up_to`) có được nối với nhau không.** Trong `kinds.replenishment_advice`,
   tham số `service_level` (dùng làm sàn cho CR) và `service_level_cfg` (dùng cho
   bảng z của ROP) là **hai tham số khác nhau**; `decisions_run.py:1488–1502`
   truyền `service_level_cfg` nhưng **KHÔNG truyền `service_level`** → sàn mức
   phục vụ của `order_up_to` luôn là `None` ở đường batch. Tôi không rõ đây là cố
   ý hay là một dây chưa nối.

5. **CHƯA CHẮC: `lead_time_std` mặc định 2.0 (`decisions_run.py:1451`) vs 0.0
   (`store/supplier.py:_DEFAULTS`)** — hai con số khác nhau cho cùng khái niệm.
   Comment nói rõ 2.0 là chủ ý cho trường hợp "thiếu dòng supplier", nhưng tôi
   không tìm thấy quyết định nào ghi lại lý do chọn đúng 2.0 ngày.

6. **CHƯA CHẮC: `promo_candidate` được sinh `top-5 theo EV mỗi project`
   (`decisions_run.py:1839`) nhưng dedup key là `promo_candidate:<sku>:<ngày>`.**
   Vì đường này chưa từng chạy, hành vi dedup khi 2 SKU khác nhau cùng lọt top-5
   chưa bao giờ được quan sát thực tế.

---

## PHỤ LỤC A — Bản đồ tệp

| Tệp | Dòng | Vai trò |
|---|---|---|
| `services/decision/app/core/kinds.py` | 1378 | 13 builder thuần, rubric, dedup |
| `services/decision/app/jobs/decisions_run.py` | 1928 | vòng chạy chính, gate, cổng, dedup |
| `services/decision/app/main.py` | 1268 | API `/v1/*`, `:price-preview`, `/v1/config` |
| `services/decision/app/jobs/outcome_ledger.py` | 520 | vòng phản hồi vai C |
| `services/decision/app/core/econ/price_explain.py` | 452 | explain hold + explain guard |
| `services/decision/app/jobs/experiment_gate_run.py` | 398 | runner + scheduler M20 |
| `services/decision/app/jobs/state_rollup.py` | 381 | event → state |
| `services/decision/app/core/econ/price_optimizer.py` | 327 | mean-CVaR |
| `services/decision/app/core/econ/elasticity.py` | 303 | OLS + EB shrinkage |
| `services/decision/app/core/guardrails.py` | 279 | anti-osc + price rules |
| `services/decision/app/core/econ/pricing_policy.py` | 256 | chọn pricer |
| `services/decision/app/core/experiment_gate.py` | 248 | FIRE/BLOCK/KILL |
| `services/decision/app/core/decision_plan.py` | 227 | thang ưu tiên + supersede |
| `services/decision/app/core/econ/replenish.py` | 210 | newsvendor + ROP |
| `services/decision/app/core/econ/price_bandit.py` | 197 | Thompson |
| `services/decision/app/core/econ/pricing.py` | 105 | Lerner + EV |
| `services/decision/app/core/scenario_newsvendor.py` | 88 | newsvendor kịch bản |
| `services/decision/app/core/econ/slowmover.py` | 60 | vốn chôn + tier |

## PHỤ LỤC B — Bảng DB mà BT02 đọc/ghi

| Bảng | Ghi bởi | Đọc bởi | Cột chính |
|---|---|---|---|
| `raw_events` | `/v1/events:ingest` | `state_rollup`, `_get_promo_*`, `_find_bundle_pairs` | `event_type`, `payload`, `event_time`, `received_at` |
| `cost_ledger` | `_process_cost` | `cost_increase_alert`, `_get_age_days` | `unit_cost`, `qty`, `recorded_at` |
| `cost_state` | `_process_cost` | mọi kind | `ewma_cost`, `last3_avg`, `prev3_avg`, `n_receipts` |
| `price_state` | `_process_price` | mọi kind giá | `current_price` |
| `price_history` | `_process_price` | elasticity, outcome_ledger | `price`, `changed_at` |
| `stock_state` | `_process_stock` | replenish, stockout, slowmover | `on_hand_qty` (snapshot, **không lịch sử**) |
| `sales_daily` | `_process_purchase`, `_process_return` | mọi thứ | `day`, `units`, `revenue` |
| `competitor_price_state` | `_process_competitor_price` | `price_suggestion` | `competitor_price`, `competitor_ref` |
| `elasticity` | `refresh_all_elasticity` | `price_suggestion`, `price_hold`, `:price-preview`, A6 | `eps`, `eps_sd`, `n_points`, `r2`, `method` |
| `price_bandit_state` | `_run_price_bandit_shadow` | cùng | `arm_price`, `mu`, `sigma`, `n` |
| `supplier_config` | `PUT /v1/config:supplier` | replenish, stockout | `lead_time_days`, `lead_time_std`, `moq`, `pack_size` |
| `project_config` | `PUT /v1/config` | mọi nơi | `promo_cap_pct`, `service_level`, `max_discount_pct`, `charm_pricing`, `map_floors`, `pricing_mode`, `quota_*` |
| `decisions` | `_insert_decision` | `GET /v1/decisions`, anti-osc history, outcome | 15 cột, UNIQUE `(project_id, dedup_key)` |
| `outcome_ledger` | `run_outcome_ledger_once` | `advice_scorecard` | `predicted_ev`, `realized_ev`, `window_days`, `note` |
| `kv_state` | rollup watermark, skips per project | `:decisions:stats` | `k`, `v` (jsonb) |
| `job_runs` | mọi loop | quan sát | `job_name`, `status`, `stats` |
| `experiment_registry` / `impression_log` / `experiment_gate_audit` | M20 | gate | `config["gate"]`, `arms`, cycle |

*Mọi bảng đều có RLS policy `tenant_isolation` theo `current_setting('app.project_id')`.*
