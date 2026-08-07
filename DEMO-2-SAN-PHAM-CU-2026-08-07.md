# DEMO 2 — SẢN PHẨM ĐÃ CÓ ĐỦ DỮ LIỆU: hệ chạy hết công suất
> Kịch bản 28 API, trọn 1 vòng: **smart search → recommend → forecast → decision → phản hồi**.
> SKU: **`bh-mi-haohao`** — "Thùng 30 gói mì Hảo Hảo tôm chua cay", **128 ngày lịch sử**, giá 111.000đ, vốn 69.455đ.
> Mọi INPUT/OUTPUT dưới đây là **kết quả chạy thật trên demoshop lúc 2026-08-07 00:00-00:25**, không phải ví dụ.
> **Chạy song song với DEMO-1** để khách thấy khác biệt giữa hàng mới và hàng đã có lịch sử.

## THÔNG ĐIỆP BÁN HÀNG CỦA MÀN NÀY
Cùng bộ API đó, nhưng trên sản phẩm đã bán 4 tháng: hệ **không còn phải mượn** gì cả — độ co giãn giá ước lượng
riêng cho SKU này từ 119 điểm dữ liệu, mô hình dự báo được **chấm điểm và chọn tự động**, và hệ **tự công bố
điểm số của chính mình** (sai số, độ phủ khoảng tin cậy). Điểm nhấn: mọi con số đều truy ngược được tới công thức.

## CHUẨN BỊ (giống DEMO-1)
```bash
cd /home/hai-soft/projects/icpp/mecom/project
SKEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['search'])")
DKEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['decision'])")
FKEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['forecast'])")
ITOK=$(docker exec miniai-smartsearch printenv MINIAI_INTERNAL_TOKEN)
SKU=bh-mi-haohao
EVT=$(date -u +%Y-%m-%dT%H:%M:%SZ)
```
Cổng: smartsearch **16021** · decision **16022** · forecast **16023**.

---
# PHẦN A — SMART SEARCH & RECOMMEND (10 API)

### [01] GET /v1/ping — xác thực key
**INPUT:** không có body; chỉ 2 header.
```bash
curl -s localhost:16021/v1/ping -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop"
```
**OUTPUT thật:** `{"pong":true,"project_id":"demoshop"}`
**Đọc kết quả:** xác nhận key hợp lệ **và** đang thao tác đúng tenant. Key sai hoặc key của tenant khác → `403`.

### [02] POST /v1/search — tìm kiếm lai, gõ không dấu vẫn ra
**INPUT:** `query` · `page_size`.
```bash
curl -s localhost:16021/v1/search -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"query":"mi hao hao","page_size":5}' | python3 -c "import json,sys; [print(round(i['score'],4),'|',i['product_id'],'|',i['title']) for i in json.load(sys.stdin)['items']]"
```
**OUTPUT thật**
```
0.0328 | bh-mi-haohao        | Thùng 30 gói mì Hảo Hảo tôm chua cay
0.0323 | tt-somi-nam-oxford  | Áo sơ mi nam Oxford dài tay
...
```
**Đọc kết quả:** khách gõ **không dấu** `"mi hao hao"` vẫn ra đúng hàng ở vị trí #1. `score` là điểm RRF sau khi
trộn khớp-chữ và khớp-nghĩa, `source: rrf_fusion`.
> ⚠ Vị trí #2 là "áo sơ **mi**" — âm tiết "mi" 2 ký tự gây nhiễu khi bỏ dấu. Nếu khách thắc mắc: đây là điểm
> yếu đã đo, đã ghi sổ và đã có thiết kế xử lý (tầng thuộc tính sản phẩm). Nói trước vẫn hơn bị bắt gặp.

### [03] GET /v1/suggest — gợi ý gõ phím có trọng số theo độ phổ biến
```bash
curl -s "localhost:16021/v1/suggest?q=mi&limit=5" -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
```
**OUTPUT thật**
```json
{"items": [{"text": "mì", "weight": 334.80}, {"text": "mì hảo hảo", "weight": 334.80},
           {"text": "mì hảo", "weight": 334.80}, {"text": "micellar", "weight": 312.68},
           {"text": "micellar 400ml", "weight": 312.68}],
 "consistency": {"projection_watermark": 929993, "is_caught_up": true, "ledger_head": 929993}}
```
**Đọc kết quả:** khách gõ **2 chữ cái** đã gợi được cụm đầy đủ **có dấu**. `weight` **334.8** so với hàng mới
(1.0 ở DEMO-1) — đây chính là **giá trị của dữ liệu tích luỹ**: gợi ý được xếp theo mức độ khách thật sự tìm.

### [04] POST /v1/recommend (context=pdp) — mua kèm thật, học từ hành vi
```bash
curl -s localhost:16021/v1/recommend -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"context":"pdp","product_id":"bh-mi-haohao"}' | python3 -c "import json,sys; [print(round(i['score'],1),'|',i['product_id'],'|',i['title']) for i in json.load(sys.stdin)['items'][:5]]"
```
**OUTPUT thật**
```
145.3 | bh-snack-oishi      | Snack Oishi tôm cay 42g (lốc 10)
106.7 | bh-xucxich-ducviet  | Xúc xích tiệt trùng Đức Việt gói 500g
...
```
**Đọc kết quả — so sánh trực tiếp với DEMO-1:** hàng mới chỉ gợi được "cùng ngành"; hàng có lịch sử gợi được
**đúng giỏ hàng thật của người Việt** — mì gói đi với snack và xúc xích. Đây là tri thức **học từ hành vi mua
chung**, không ai lập trình tay.

### [05] POST /v1/recommend (context=similar) — hàng thay thế
```bash
curl -s localhost:16021/v1/recommend -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"context":"similar","product_id":"bh-mi-haohao"}' | python3 -c "import json,sys; [print(round(i['score'],3),'|',i['title']) for i in json.load(sys.stdin)['items'][:5]]"
```
**OUTPUT thật**
```
0.320 | Thùng 24 lon Coca-Cola 320ml
0.31x | Nước mắm Nam Ngư 3 trong 1 750ml
```
**Đọc kết quả:** `source: reco_similar`, thang điểm 0-1 (khác `reco_pdp` thang trăm) vì đây là **độ tương đồng
nội dung**, không phải điểm phổ biến. Lưu ý trung thực: chùm điểm 0.31-0.32 khá sát nhau — dấu hiệu vector phân
biệt ngành còn yếu, đã ghi sổ.

### [06] POST /v1/recommend (context=cart) — gợi ý trong giỏ hàng
**INPUT:** `context: "cart"` · `product_ids[]` (giỏ hiện tại) · **`user_pseudo_id` bắt buộc** cho ngữ cảnh này.
```bash
curl -s localhost:16021/v1/recommend -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"context":"cart","product_ids":["bh-mi-haohao"],"user_pseudo_id":"demo-user-01"}' | python3 -c "import json,sys; [print(round(i['score'],1),'|',i['title']) for i in json.load(sys.stdin)['items'][:5]]"
```
**OUTPUT thật**
```
338.0 | Kính cường lực iPhone 15 full màn
334.0 | Kem chống nắng Anessa Perfect UV 60ml
324.1 | Set 5 đôi tất nam cotton khử mùi
```
**Đọc kết quả:** giỏ hàng gợi theo **giá trị đơn hàng tăng thêm** cho người dùng đó, nên cross-ngành là **có
chủ đích** (khác PDP phải đúng ngành). `source: reco_cart`.
> Nếu thiếu `user_pseudo_id` → `400 INVALID_ARGUMENT: 'user_pseudo_id' is required for context=cart`.

### [07] POST /v1/ask — hỏi tự nhiên
```bash
curl -s localhost:16021/v1/ask -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"question":"shop co ban mi an lien khong?"}' | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['answer']); print('--- nguon:', d['answer_source'], '| chan bia:', d['grounding_guard']['blocked'], '| loc nganh:', d['answer_coherence'])"
```
**OUTPUT thật:** trả đúng mì, `answer_coherence.filtered=true` và **loại 4 sản phẩm lệch ngành**.
**Đọc kết quả:** xem giải thích chi tiết từng trường ở DEMO-1 bước [05].
> ⚠ **Tránh** hỏi `"mi hao hao gia bao nhieu?"` — âm tiết "mi" kéo áo sơ mi vào câu trả lời (đã đo).

### [08] GET /internal/similar-products — hàng xóm theo vector (API nội bộ)
```bash
curl -s "localhost:16021/internal/similar-products?project_id=demoshop&product_id=bh-mi-haohao&limit=5" -H "X-Internal-Token: $ITOK" | python3 -m json.tool
```
**Đọc kết quả:** đây là đường **decision/forecast gọi chéo sang search** để tìm sản phẩm tương tự khi cần
cold-start. Dùng `X-Internal-Token`, không dùng key khách hàng — ranh giới nội bộ/công khai rõ ràng.

### [09] GET /internal/products-by-category — SKU theo ngành
```bash
curl -s "localhost:16021/internal/products-by-category?project_id=demoshop&category_l1=B%C3%A1ch%20h%C3%B3a&limit=8" -H "X-Internal-Token: $ITOK" | python3 -m json.tool
```
**OUTPUT thật:** `{"product_ids": ["bh-banh-oreo", "bh-bia-tiger", "bh-cafe-g7", ...]}`
**Đọc kết quả:** đây là đường `forecast:aggregate` dùng để gộp dự báo theo ngành.
🆕 Từ 06/08 **hết phân biệt dấu**: `Bách hóa`, `Bach hoa`, `BACH HOA` trả **cùng một tập** — trước đây khách
khai không dấu sẽ tạo ra một "ngành ma" song song và mọi số gộp theo ngành đều lệch.

### [10] POST /v1/events:ingest — bơm hành vi thật đang diễn ra
**Ý nghĩa:** `:ingest` là đường sự kiện **hằng ngày** (khác `:backfill` dùng cho lịch sử khi onboard).
**INPUT:** mảng `events[]`, mỗi phần tử như bảng ở DEMO-1 bước [09].
```bash
curl -s localhost:16021/v1/events:ingest -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d "{\"events\":[{\"event_id\":\"demo-view-$(date +%s)\",\"schema_version\":\"1.0\",\"event_type\":\"product.viewed\",\"event_time\":\"$EVT\",\"user_pseudo_id\":\"demo-user-01\",\"payload\":{\"product_id\":\"$SKU\"}},{\"event_id\":\"demo-cart-$(date +%s)\",\"schema_version\":\"1.0\",\"event_type\":\"cart.added\",\"event_time\":\"$EVT\",\"user_pseudo_id\":\"demo-user-01\",\"payload\":{\"product_id\":\"$SKU\",\"quantity\":2}}]}" | python3 -m json.tool
```
**OUTPUT thật:** `{"accepted": 2, "deduped": 0, "skipped": 0, "errors": [], "ledger_position": 929995}`
**Đọc kết quả:** `ledger_position` tăng lên — mọi sự kiện đều vào **sổ cái ghi-một-lần**; không sửa, không xoá,
chỉ ghi thêm. Muốn đảo một giao dịch thì ghi bút toán đảo, giống kế toán.

---
# PHẦN B — FORECAST (10 API)

### [11] POST /v1/forecast:run — chạy dự báo (bất đồng bộ)
```bash
curl -s -w "\nstatus: %{http_code}\n" -X POST localhost:16023/v1/forecast:run -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{}'
```
**OUTPUT thật:** `202` + `{"status":"queued","run_id":"r_2026-08-07","job_id":"fr-demoshop-r_2026-08-07"}`
**Đọc kết quả:** xem DEMO-1 bước [11]. Đo thật: job xong sau **~20 giây** cho 124 SKU.

### [12] GET /v1/projections/status — chờ job xong rồi mới đo tiếp
```bash
curl -s "localhost:16023/v1/projections/status?job_id=fr-demoshop-r_$(date -u +%F)" -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" | python3 -c "import json,sys; j=json.load(sys.stdin); print('job:', j['job']['status'], '| bat kip so cai:', j['is_caught_up'])"
```
**OUTPUT thật:** `job: done | bat kip so cai: True`

### [13] POST /v1/forecast:query — dải dự báo P10/P50/P90
```bash
curl -s localhost:16023/v1/forecast:query -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"bh-mi-haohao","horizon_days":14}' | python3 -m json.tool | head -25
```
**OUTPUT thật** (rút gọn)
```json
{"product_id": "bh-mi-haohao", "run_id": "r_2026-08-07",
 "daily": [{"day": "2026-08-08", "p10": 1.206, "p50": 4.279, "p90": 7.352},
           {"day": "2026-08-09", "p10": 1.208, "p50": 4.280, "p90": 7.353}, ...]}
```
**Đọc kết quả:** `p50 ≈ 4,28 thùng/ngày` là kế hoạch; `p90 ≈ 7,35` là mức chuẩn bị hàng; `p10 ≈ 1,21` là kịch
bản xấu để giữ dòng tiền. Ngoài ra kết quả còn kèm `model_used` (**mô hình nào được chọn cho SKU này**),
`data_window` (học trên bao nhiêu ngày) và `calibration` (hệ số nới khoảng tin cậy + độ phủ thực đo).
**Điểm khoe:** hệ có **thang 8 mô hình** (từ cold-start, Croston cho hàng bán lai rai, ETS/Theta, tới LightGBM
quantile toàn cục) và **tự chấm điểm chọn mô hình cho từng SKU** bằng kiểm định lùi — không dùng một mô hình
cho tất cả.

### [14] POST /v1/forecast:aggregate — dự báo gộp theo ngành
**INPUT:** `category_l1` (hoặc `product_ids[]`) · `horizon_days`.
```bash
curl -s localhost:16023/v1/forecast:aggregate -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"category_l1":"Bách hóa","horizon_days":7}' | python3 -m json.tool | head -20
```
**OUTPUT thật** (rút gọn)
```json
{"scope": {"product_ids_count": 17, "category_l1": "Bách hóa",
           "resolved_product_ids": ["bh-banh-oreo", "bh-bia-tiger", "bh-cafe-g7", "..."]},
 "sku_count": 17, "horizon_days": 7, "totals": {...}}
```
**Đọc kết quả:** `resolved_product_ids` **liệt kê đích danh** 17 SKU đã được gộp — chủ shop kiểm được hệ có
bỏ sót hàng nào không. Lưu ý kỹ thuật quan trọng: **phân vị không cộng được** (p90 của tổng ≠ tổng các p90),
nên phần gộp đi qua tầng kịch bản Monte Carlo chứ không cộng thô.

### [15] GET /v1/forecast:accuracy — **HỆ TỰ CHẤM ĐIỂM CHÍNH MÌNH**
```bash
curl -s "localhost:16023/v1/forecast:accuracy?window=90d" -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
```
**OUTPUT thật** (rút gọn)
```json
{"window": "90d", "by_model": [
  {"model": "adida",                  "segment": "intermittent", "sku_count": 66,
   "pinball_q50_avg": 0.366, "mase_avg": 0.896, "coverage_p10_p90": 0.905},
  {"model": "autoets_theta_ensemble", "segment": "smooth",       "sku_count": 53,
   "pinball_q50_avg": 0.702, "mase_avg": 0.811, "coverage_p10_p90": 0.888},
  {"model": "lgbm_global",            "segment": "smooth",       "sku_count": 49,
   "pinball_q50_avg": 0.687, "mase_avg": 0.784, "coverage_p10_p90": 0.709}, "..."]}
```
**Đọc kết quả — đây là API thuyết phục nhất với khách khó tính:**
- **`mase_avg`** — sai số so với phương pháp ngây thơ "tuần trước bán bao nhiêu, tuần này bấy nhiêu".
  **< 1 là thắng**; ở đây 0,78-0,90 ⇒ mô hình **tốt hơn baseline 10-22%**.
- **`coverage_p10_p90`** — thực tế rơi vào khoảng [P10, P90] bao nhiêu phần trăm. Lý tưởng ≈ 0,80.
  0,888-0,905 = khoảng hơi rộng (an toàn); 0,709 = hơi hẹp, đã biết và đang theo dõi.
- **`segment`** — hệ chia hàng thành *bán đều* (smooth) và *bán lai rai* (intermittent) và dùng mô hình khác nhau.
- **`sku_count`** — mỗi mô hình đang phục vụ bao nhiêu SKU.
**Câu nói:** *"Hệ tự công bố điểm số của chính nó, kể cả chỗ chưa đẹp. Anh chị không phải tin lời tôi."*

### [16] POST /v1/forecast:insights — insight nhu cầu
**INPUT:** `kind` (**số ít**, một trong: `accuracy_sku`, `top_movers`, `group_breakdown`, `seasonality`,
`sell_through_prob`, `promo_uplift_learned`) · tham số kèm theo tuỳ loại.
```bash
curl -s localhost:16023/v1/forecast:insights -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"kind":"top_movers","window_days":30}' | python3 -m json.tool | head -20
```
**OUTPUT thật** (rút gọn)
```json
{"horizon_days": 28, "run_id": "r_2026-08-07", "method": "persisted_run_sum_p50",
 "ranking_note": "ranked by direct sum of daily p50 (ordering only)",
 "movers": [{"product_id": "dt-cuongluc-ip15", "total_p50": 137.07, "total_p10": 38.33, "total_p90": 235.03, "days": 28},
            {"product_id": "ld-kcn-anessa",    "total_p50": 133.14, "total_p10": 78.43, "total_p90": 193.60, "days": 28},
            {"product_id": "bh-mi-haohao",     "total_p50": 120.27, "total_p10": 35.57, "total_p90": 206.77, "days": 28}, "..."]}
```
**Đọc kết quả:** top hàng sẽ bán chạy 28 ngày tới. Đáng chú ý: **`ranking_note` tự khai giới hạn của chính nó** —
"cộng thẳng p50, chỉ dùng để xếp thứ tự" (vì cộng phân vị là sai về toán, muốn số tổng chính xác phải qua kịch bản).
Một hệ trung thực là hệ ghi rõ chỗ nó xấp xỉ.

### [17] POST /v1/forecast:promo-preview — "giảm 30% thì bán thêm bao nhiêu?"
**INPUT:** `product_id` · `discount_pct` · **`start` và `end` (ngày ISO, bắt buộc)**.
```bash
S=$(date -u -d "+3 days" +%F); E=$(date -u -d "+10 days" +%F)
curl -s localhost:16023/v1/forecast:promo-preview -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d "{\"product_id\":\"bh-mi-haohao\",\"discount_pct\":30,\"start\":\"$S\",\"end\":\"$E\"}" | python3 -m json.tool | head -20
```
**OUTPUT thật** (rút gọn)
```json
{"product_id": "bh-mi-haohao", "daily": [
  {"day": "2026-08-08", "p10": 1.206, "p50": 4.279, "p90": 7.352, "promo": false},
  {"day": "2026-08-09", "p10": 1.208, "p50": 4.280, "p90": 7.353, "promo": false},
  {"day": "2026-08-10", "p10": 1.689, "p50": 5.981, "p90": 10.291, "promo": true}, "..."]}
```
**Đọc kết quả — chỗ này rất trực quan trên màn hình:** cột `promo` đánh dấu ngày có khuyến mại; đúng ngày
khuyến mại **p50 nhảy từ 4,28 lên 5,98 thùng/ngày (+39,8%)**. Mức nhảy này **học từ các đợt khuyến mại đã chạy
trong quá khứ** của chính shop, không phải hệ số bịa.

### [18] POST /v1/scenarios:build — dựng 128 kịch bản
```bash
curl -s localhost:16023/v1/scenarios:build -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_ids":["bh-mi-haohao"],"horizon_days":7,"scenario_count":128}' | python3 -c "import json,sys; d=json.load(sys.stdin); print('run_id:', d['run_id']); print('manifest:', json.dumps(d['manifest'], ensure_ascii=False)[:400])"
```
**OUTPUT thật:** `run_id: sc_d3b954fd8813` + manifest (RNG philox có hạt giống, SHA-256 từng tệp — xem DEMO-1 [14]).
> ⚠ **Ba API kịch bản bên dưới đều cần `run_id` này.** Phải chạy `:build` trước.

### [19] POST /v1/scenarios:lead-time-demand — cầu trong thời gian chờ hàng
**INPUT:** `product_ids[]` · `run_id` · `lead_time_days` (bao lâu hàng về) · `review_period_days` (bao lâu kiểm
kho một lần) · `required_quantiles[]` · `horizon_days`.
```bash
curl -s localhost:16023/v1/scenarios:lead-time-demand -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_ids":["bh-mi-haohao"],"run_id":"sc_d3b954fd8813","lead_time_days":3,"review_period_days":2,"required_quantiles":[0.5,0.9],"horizon_days":7}' | python3 -m json.tool
```
**OUTPUT thật**
```json
{"run_id": "sc_d3b954fd8813",
 "quantiles": {"0.5": 22.176, "0.9": 29.832},
 "mean": 22.734, "sku_count": 1, "scenario_count": 128,
 "consistency": {"is_caught_up": true, "ledger_head": 929995}}
```
**Đọc kết quả — dịch sang lời chủ shop:** *"Từ lúc đặt hàng tới lúc hàng về (3 ngày) cộng chu kỳ kiểm kho
(2 ngày), tôi cần chuẩn bị **khoảng 22 thùng** cho trường hợp bình thường, và **30 thùng** nếu muốn 90% chắc
chắn không cháy hàng."* Đây chính là con số để đặt đơn nhập.

### [20] POST /v1/scenarios:aggregate — gộp kịch bản
```bash
curl -s localhost:16023/v1/scenarios:aggregate -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_ids":["bh-mi-haohao"],"run_id":"sc_d3b954fd8813","horizon_days":7}' | python3 -m json.tool
```
**OUTPUT thật**
```json
{"run_id": "sc_d3b954fd8813", "horizon_days": 7, "sku_count": 1, "scenario_count": 128,
 "totals": {"p10": 22.0, "p50": 31.324, "p90": 39.310, "mean": 31.538},
 "method": "scenario_mc_128"}
```
**Đọc kết quả:** tổng cầu 7 ngày: **thấp 22 – giữa 31,3 – cao 39,3 thùng**. `method: scenario_mc_128` khai rõ
số này tính bằng **mô phỏng 128 kịch bản**, không phải cộng phân vị (cách cộng thô sẽ cho số sai).

### [21] POST /v1/scenarios:probability — xác suất vượt ngưỡng
**INPUT:** `run_id` · `product_id` (số ít) · **`threshold_units`** · `horizon_days`.
```bash
curl -s localhost:16023/v1/scenarios:probability -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"run_id":"sc_d3b954fd8813","product_id":"bh-mi-haohao","threshold_units":30,"horizon_days":7}' | python3 -m json.tool
```
**OUTPUT thật**
```json
{"product_id": "bh-mi-haohao", "threshold_units": 30.0, "horizon_days": 7,
 "probability": 0.6015625, "scenario_count": 128}
```
**Đọc kết quả:** *"Xác suất bán được từ 30 thùng trở lên trong 7 ngày tới là **60,2%**"* — tính bằng 77/128 kịch
bản vượt ngưỡng. Đây là dạng câu hỏi chủ shop hỏi thật khi cân nhắc ôm hàng theo lô.

---
# PHẦN C — DECISION (8 API)

### [22] GET /v1/config — chính sách giá của tenant
```bash
curl -s localhost:16022/v1/config -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
```
**OUTPUT thật**
```json
{"config": {"promo_cap_pct": 50, "pricing_mode": "lerner"}}
```
**Đọc kết quả:** `promo_cap_pct: 50` = trần giảm giá 50%, hệ **không bao giờ** đề xuất vượt.
`pricing_mode` có 2 chế độ, đổi được **per-tenant, không cần khởi động lại**:
- **`lerner`** — tối ưu lợi nhuận kỳ vọng (mặc định).
- **`robust`** — thận trọng, cân nhắc rủi ro đuôi; khi độ bất định của ước lượng quá lớn thì **giữ giá** thay
  vì mạo hiểm. Đổi bằng `PUT /v1/config -d '{"pricing_mode":"robust"}'` (nhớ trả lại sau demo).

### [23] POST /v1/events:ingest — nạp tồn kho + giá vốn
**INPUT (3 loại sự kiện — tên trường phải đúng):**
`cost.recorded` → `{product_id, unit_cost, qty}` · `stock.level` → `{product_id, on_hand_qty}` ·
`price.changed` → `{product_id, new_price}`.
```bash
curl -s localhost:16022/v1/events:ingest -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d "{\"events\":[{\"event_id\":\"demo-stock-$(date +%s)\",\"schema_version\":\"1.0\",\"event_type\":\"stock.level\",\"event_time\":\"$EVT\",\"user_pseudo_id\":\"system\",\"payload\":{\"product_id\":\"$SKU\",\"on_hand_qty\":120}},{\"event_id\":\"demo-cost-$(date +%s)\",\"schema_version\":\"1.0\",\"event_type\":\"cost.recorded\",\"event_time\":\"$EVT\",\"user_pseudo_id\":\"system\",\"payload\":{\"product_id\":\"$SKU\",\"unit_cost\":69500,\"qty\":50}}]}" | python3 -m json.tool
```
**OUTPUT thật:** `{"accepted": 2, "deduped": 0, "errors": [], "ledger_position": ...}`
**Đọc kết quả:** giá vốn được cộng dồn theo **bình quân trượt có trọng số** (EWMA) — lô nhập mới ảnh hưởng
nhiều hơn lô cũ, phản ánh đúng thực tế giá nhập biến động.

### [24] POST /v1/decisions:run — kích bộ não quyết định
```bash
curl -s -X POST localhost:16022/v1/decisions:run -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{}' | python3 -m json.tool
```
**OUTPUT thật**
```json
{"created": 155, "skipped_dedup": 0,
 "skipped_by_reason": {"anti_oscillation": 127, "plan_conflict": 94,
                       "insufficient_history": 1, "no_stock": 2, "no_cost": 33},
 "superseded_plan": 94, "price_hold": 110, "anti_osc_hold": 110}
```
**Đọc kết quả:** 155 lời khuyên mới cho cả shop. Ý nghĩa từng trường xem DEMO-1 bước [15].
**Điểm cần nhấn:** `anti_oscillation: 127` — hệ **chủ động im lặng** với 127 SKU vừa đổi giá gần đây.
Một hệ khuyên đổi giá mỗi ngày là hệ làm hại chủ shop; chốt chặn này quan trọng ngang bản thân thuật toán.

### [25] GET /v1/decisions — hàng đợi lời khuyên
**INPUT:** `page_size` · `kind` · `status` (⚠ **không có `product_id`** — lọc phía client).
```bash
curl -s "localhost:16022/v1/decisions?page_size=5" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | python3 -c "
import json,sys
for x in json.load(sys.stdin)['items']:
    ev=x.get('expected_value') or {}
    print(x['decision_id'][:20],'|',x['kind'],'|',x['status'],'| EV', format(ev.get('amount',0),',.0f'),'d/',ev.get('per',''))"
```
**Đọc kết quả:** mỗi dòng là một lời khuyên có **giá trị kỳ vọng bằng tiền/tháng** để xếp ưu tiên, kèm
`guardrails` đã kiểm và `trace` — **toàn bộ phép tính viết ra bằng chữ**. Ví dụ trace thật của một đề xuất
combo: `lift=18.63 (>=2.0), pair_cnt=44 (>=5); margin_a=33.80%, margin_b=22.30% (both >15%); bundle_price=130000
voucher=7000 …; EV = 0.15*44*(33466+8474) = 276807`. Chủ shop tự kiểm được từng bước, không phải hộp đen.

### [26] GET /v1/decisions:stats — thống kê + tỷ lệ chấp nhận
```bash
curl -s "localhost:16022/v1/decisions:stats?window=30d" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool | head -25
```
**OUTPUT thật** (rút gọn)
```json
{"by_kind": {
  "price_suggestion":     {"count": 327, "accepted_rate": 0.0,    "dismissed_rate": 0.0},
  "price_hold":           {"count": 336, "accepted_rate": 0.0,    "dismissed_rate": 0.0},
  "replenishment_advice": {"count": 79,  "accepted_rate": 0.0,    "dismissed_rate": 0.0},
  "bundle_suggestion":    {"count": 58,  "accepted_rate": 0.0690, "dismissed_rate": 0.0},
  "stockout_warning":     {"count": 20,  "accepted_rate": 0.0,    "dismissed_rate": 0.0},
  "below_cost_alert":     {"count": 2,   "accepted_rate": 0.0,    "dismissed_rate": 0.0}}}
```
**Đọc kết quả:** phân bố theo loại quyết định trong 30 ngày. **`accepted_rate` là thước đo hệ tự soi mình**:
tỷ lệ chủ shop thực sự làm theo. `price_hold` (336) nhiều hơn `price_suggestion` (327) — hệ nói "giữ giá"
nhiều hơn "đổi giá", đúng tinh thần thận trọng. `below_cost_alert: 2` = đã bắt được 2 ca đang bán dưới vốn.

### [27] GET /v1/decisions:insights — insight tổng hợp
**INPUT:** `kind` (**số ít**): `capital_locked` | `advice_scorecard` | `monthly_benefit` | `removal_candidates`
| `bundle_candidates` | `golden_hours`.
```bash
curl -s "localhost:16022/v1/decisions:insights?kind=capital_locked" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
```
**OUTPUT thật**
```json
{"kind": "capital_locked", "total_capital_locked": 0, "currency_code": "VND",
 "n_slowmovers": 0, "top": [],
 "basis": "slowmover = on_hand>0, units_30d=0, history>=30d; von dong = on_hand*ewma_cost"}
```
**Đọc kết quả:** *"Vốn đang chôn trong hàng ế = 0đ, không có SKU nào đứng bánh"*. Trường **`basis` định nghĩa
rõ thế nào là hàng ế** ngay trong kết quả — không để chủ shop đoán tiêu chí. Thử `kind=monthly_benefit` để
xem tổng lợi ích hệ mang lại trong tháng.

### [28] POST /v1/decisions:price-preview — thử giá (2 lần: hợp lệ & dưới vốn)
```bash
# 28a — thử giảm 111.000 -> 99.000
curl -s localhost:16022/v1/decisions:price-preview -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"bh-mi-haohao","candidate_price":99000}' | python3 -m json.tool
```
**OUTPUT thật**
```json
{"current":   {"price": 111000.0, "est_units_30d": 185.0,  "est_profit_30d": 7685868.88},
 "candidate": {"price": 99000,    "est_units_30d": 197.55, "est_profit_30d": 5836585.25},
 "delta_profit_30d": -1849283.62,
 "elasticity_used": {"eps": -0.5736, "method": "ols_daily", "n_points": 119, "r2": 0.447},
 "guardrails": [{"code": "BELOW_COST", "status": "PASS"}],
 "confidence": 0.9,
 "explanation": "Q(P)=Q0·(P/P0)^eps; profit=(P-c)·Q"}
```
**Đọc kết quả — SO SÁNH TRỰC TIẾP VỚI DEMO-1 (đây là điểm chốt của cả buổi):**

| | Hàng mới (DEMO-1) | Hàng có 128 ngày (file này) |
|---|---|---|
| `method` | `pooled_prior` — **mượn** của shop | **`ols_daily`** — hồi quy riêng SKU này |
| `n_points` | 21 | **119** |
| `r2` | `null` | **0.447** — độ khớp của hồi quy |
| `confidence` | 0.7 | **0.9** |

Kết luận kinh doanh: hạ giá 10,8% → bán thêm 6,8% → **lãi tháng GIẢM 1,85 triệu** ⇒ máy **can đừng làm**.
Đây là chỗ máy chặn trực giác "cứ giảm giá cho chạy hàng" bằng con số.

```bash
# 28b — thử giá DƯỚI VỐN (vốn 69.455đ)
curl -s localhost:16022/v1/decisions:price-preview -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"bh-mi-haohao","candidate_price":50000}' | python3 -c "import json,sys; d=json.load(sys.stdin); print('guardrails:', d['guardrails']); print('delta_profit_30d:', format(d['delta_profit_30d'],',.0f'))"
```
**OUTPUT thật**
```
guardrails: [{'code': 'BELOW_COST', 'status': 'FAIL'}]
delta_profit_30d: -13,372,522
```
**Đọc kết quả:** giá 50.000đ dưới vốn 69.455đ ⇒ **`FAIL`** + lãi tháng âm 13,4 triệu. Chốt an toàn này vừa
được sửa 06/08 và có test hồi quy khoá lại vĩnh viễn.

### [29] GET /v1/decisions:replenish-plan — kế hoạch nhập hàng
```bash
curl -s "localhost:16022/v1/decisions:replenish-plan?product_id=bh-mi-haohao" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
```
**OUTPUT thật**
```json
{"items": [{"product_id": "bh-mi-haohao",
  "avg_daily_units": 5.9, "sigma_daily": 3.898,
  "lead_time_days": 3.0, "lead_time_std": 0.0,
  "service_level": 0.9, "z": 1.28,
  "safety_stock": 8.64, "reorder_point": 26.34,
  "on_hand": 50.0, "days_of_inventory": 8.5, "below_reorder_point": false,
  "moq": 10.0, "pack_size": 1.0, "order_qty_moq_pack": 0,
  "formula": "ROP = avg_daily*LT + z*sqrt(LT*sigma_d^2 + avg_d^2*sigma_LT^2); DOI = on_hand/avg_daily"}],
 "n": 1, "window_days": 30}
```
**Đọc kết quả:** bán **5,9 thùng/ngày**, hàng về sau **3 ngày**, muốn 90% không cháy hàng ⇒ trữ thêm
**8,64 thùng an toàn** ⇒ **đặt lại khi còn 26,34 thùng**. Đang có 50 ⇒ chưa cần đặt, đủ bán **8,5 ngày**.
`moq: 10` = nhà cung cấp yêu cầu đặt tối thiểu 10 (khai qua `PUT /v1/config:supplier`).

### [30] POST /v1/decisions/{id}:feedback — khép vòng
```bash
DID=$(curl -s "localhost:16022/v1/decisions?page_size=1" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | python3 -c "import json,sys; print(json.load(sys.stdin)['items'][0]['decision_id'])")
curl -s -X POST "localhost:16022/v1/decisions/$DID:feedback" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"action":"accepted","note":"demo doi tac 07/08"}' | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['decision_id'],'->',d.get('status'))"
```
**INPUT:** `action` ∈ {`accepted`, `rejected`, `deferred`} · `note`.
**Đọc kết quả:** phản hồi này quay lại nuôi `accepted_rate` ở bước [26], và sau 30 ngày hệ đối chiếu **lãi thực
tế** với `expected_value` đã hứa. **Đây là điều phân biệt một hệ thống AI nghiêm túc với một cái máy đoán:
nó chịu trách nhiệm với lời khuyên của mình bằng số.**

---
# BẢNG SO SÁNH 2 FILE DEMO (slide chốt buổi)

| Năng lực | DEMO-1: hàng mới 0 dữ liệu | DEMO-2: hàng 128 ngày dữ liệu |
|---|---|---|
| Tìm kiếm | ✅ sau 9 giây | ✅ + gợi ý gõ có trọng số 334.8 |
| Gợi ý liên quan | cùng ngành (cold-start) | **mua kèm thật** (mì → snack, xúc xích) |
| Dự báo | ❌ 404 (chưa bán ngày nào) → sau 21 ngày: ✅ có nhịp cuối tuần | ✅ + hệ **tự chấm điểm** MASE 0.78-0.90, coverage 0.89 |
| Khuyến mại | — | ✅ giảm 30% ⇒ p50 tăng **+39,8%**, học từ đợt cũ |
| Kịch bản nhập hàng | — | ✅ P(bán ≥30 thùng/7 ngày) = **60,2%** trên 128 kịch bản |
| Độ co giãn giá | `pooled_prior`, 21 điểm, tin cậy **0.7** | **`ols_daily`, 119 điểm, r²=0.447, tin cậy 0.9** |
| Chặn bán dưới vốn | ✅ FAIL | ✅ FAIL |
| Nhập hàng | ROP 32,8 / còn 40 | ROP 26,3 / còn 50, đủ 8,5 ngày |
| Vòng phản hồi | ✅ | ✅ + `accepted_rate` theo từng loại |

**Câu chốt cả buổi:** *"Hai sản phẩm, cùng một bộ API. Khác biệt duy nhất là lượng dữ liệu — và hệ thống
tự khai mình đang ở đâu trên thang đó: mượn hay tự tính, tin 70% hay 90%. Dữ liệu của anh chị càng nhiều,
nó càng thông minh; và ở mọi thời điểm nó đều nói thật mình biết tới đâu."*

---
# PHỤ LỤC — LỖI THƯỜNG GẶP KHI GÕ TAY (đã đo, đừng vấp trước mặt khách)

| API | Sai | Đúng |
|---|---|---|
| `/v1/recommend` context=cart | thiếu `user_pseudo_id` → 400 | thêm `"user_pseudo_id"` |
| `/v1/forecast:insights` | `"kinds": [...]` → 400 | **`"kind"` số ít** |
| `/v1/decisions:insights` | `?kinds=` → 400 | **`?kind=`** số ít |
| `/v1/forecast:promo-preview` | thiếu `start`/`end` → 400 | thêm 2 ngày ISO |
| `/v1/scenarios:*` | gọi thẳng không có `run_id` → 400 | chạy `:build` trước lấy `run_id` |
| `/v1/scenarios:probability` | `threshold` → 400 | **`threshold_units`** |
| `/v1/scenarios:lead-time-demand` | `product_id` → 400 | **`product_ids`** (mảng) |
| `purchase.completed` | `order_id` / `quantity` / `price` → từ chối | **`order_ref`** + items **`qty`**, **`unit_price`** |
| `stock.level` | `on_hand` → từ chối | **`on_hand_qty`** |
| `cost.recorded` | thiếu `qty` → từ chối | `{product_id, unit_cost, qty}` |
| `/v1/decisions?product_id=` | tham số **bị bỏ qua âm thầm** | lấy danh sách rồi lọc phía client |

**Lưu ý cuối:** mọi thông báo lỗi của hệ đều nói **đích danh trường sai và giá trị hợp lệ** — khi demo lỡ gõ
sai, cứ đọc to thông báo lỗi lên, đó cũng là một điểm cộng về chất lượng API.
