# DEMO 2 — SẢN PHẨM ĐÃ CÓ ĐỦ DỮ LIỆU: hệ chạy hết công suất
> Kịch bản 30 API, trọn 1 vòng: **smart search → recommend → forecast → decision → phản hồi**.
> SKU: **`bh-mi-haohao`** — "Thùng 30 gói mì Hảo Hảo tôm chua cay", **~132 ngày lịch sử**.
> **Số liệu đo lại toàn bộ ngày 2026-08-12** (bản trước đo 07/08 đã lệch — xem mục cuối file).
> **Chạy song song với DEMO-1** để khách thấy khác biệt giữa hàng mới và hàng đã có lịch sử.
>
> ⭐ **MỖI API ĐỀU CÓ 4 BƯỚC:** ① **ĐO TRƯỚC** → ② **GỌI API** → ③ **ĐO SAU** (chứng minh dữ liệu đã đổi)
> → ④ **LUỒNG** (dữ liệu chảy qua bảng nào, job nào).
>
> ⛔ **LUẬT NGHIỆM THU (human chốt 2026-08-12):** hai kịch bản chỉ HOÀN THIỆN khi chạy end-to-end đủ
> **4 LƯỢT LIÊN TIẾP** không lỗi, trên **cùng một bản code**; deploy giữa chừng ⇒ **đếm lại từ 1**.
> Log mỗi lượt lưu ở `icpp/demo-e2e-runs/`. Xem giải thích đầy đủ ở đầu **DEMO-1**.

## THÔNG ĐIỆP BÁN HÀNG CỦA MÀN NÀY
Cùng bộ API đó, trên sản phẩm đã bán 4 tháng: hệ **không còn phải mượn** gì cả — độ co giãn giá ước lượng
riêng cho SKU này từ **132 điểm dữ liệu**, mô hình dự báo được **chấm điểm và chọn tự động**, và hệ **tự công
bố điểm số của chính mình**. Mọi con số đều truy ngược được tới công thức và tới bảng dữ liệu.

---
## CHUẨN BỊ
```bash
cd /home/hai-soft/projects/icpp/mecom/project
SKEY=$(.venv/bin/python -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['search'])")
DKEY=$(.venv/bin/python -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['decision'])")
FKEY=$(.venv/bin/python -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['forecast'])")
ITOK=$(docker exec miniai-smartsearch printenv MINIAI_INTERNAL_TOKEN)
SKU=bh-mi-haohao
EVT=$(date -u +%Y-%m-%dT%H:%M:%SZ)
```
**BỘ ĐO** — dán cả khối `q()` và `snap()` từ **DEMO-1 mục CHUẨN BỊ** (dùng chung).

### Ảnh chụp mở màn — chiếu lên để khách thấy "đây là hàng có lịch sử"
```bash
snap bh-mi-haohao
```
**Đo thật 12/08:**
```
search  : products=1  outbox_đang_chờ=0  reco_exposure=1574
forecast: raw_events=13604  demand_daily=134  forecasts=168
decision: raw_events=17519  sales_daily=132  decisions=1587  tồn=137  vốn=70145
```
**Nói ngay:** *"So với DEMO-1: chỗ đó `demand_daily=0`, ở đây **134 ngày**. Toàn bộ khác biệt của buổi hôm nay
nằm ở hai con số này."*

Cổng: smartsearch **16021** · decision **16022** · forecast **16023**.

---
# PHẦN A — SMART SEARCH & RECOMMEND (10 API)

## [01] GET /v1/ping — xác thực key
### ② GỌI API
```bash
curl -s localhost:16021/v1/ping -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop"
```
**OUTPUT thật:** `{"pong":true,"project_id":"demoshop"}`

### ③ ĐO SAU — chứng minh key này gắn đúng tenant
```bash
q miniai_search "SELECT key_id, project_id, state FROM api_keys WHERE project_id='demoshop' AND state='active' LIMIT 3;"
```
### ④ LUỒNG
```
Bearer key ──băm + pepper──► api_keys ──► suy ra project_id ──► RLS Postgres khoá theo project_id
```
**Nói với khách:** *"Khách **không tự khai** mình là ai — `X-Project-Id` phải KHỚP với key. Cầm key shop A mà
khai shop B là bị chặn. Đây là lớp khoá thứ nhất; lớp thứ hai là RLS trong Postgres."*

---
## [02] POST /v1/search — tìm kiếm lai, gõ không dấu vẫn ra
### ① ĐO TRƯỚC
```bash
q miniai_search "SELECT coalesce(sum(cnt),0) AS lan_tim FROM query_log WHERE project_id='demoshop' AND query_norm='mi hao hao';"
```
### ② GỌI API
```bash
curl -s localhost:16021/v1/search -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"query":"mi hao hao","page_size":3}' | .venv/bin/python -c "import json,sys; [print(round(i['score'],4),'|',i['product_id'],'|',i.get('source')) for i in json.load(sys.stdin)['items']]"
```
**OUTPUT thật 12/08**
```
0.0328 | bh-mi-haohao       | rrf_fusion
0.0323 | tt-somi-nam-oxford | rrf_fusion
```
### ③ ĐO SAU
```bash
sleep 2; q miniai_search "SELECT cnt, results_count_last, user_cnt FROM query_log WHERE project_id='demoshop' AND query_norm='mi hao hao';"
```
**Đo thật:** `cnt` tăng đúng 1.
### ④ LUỒNG
```
/v1/search ──► Vespa BM25 + vector ──► RRF ──► kết quả
           └──► query_log ──job suggest_terms──► gợi ý gõ phím có trọng số ([03])
```
> ⚠ Vị trí #2 là "áo sơ **mi**" — âm tiết "mi" 2 ký tự gây nhiễu khi bỏ dấu. Điểm yếu đã đo, đã ghi sổ. Nói
> trước vẫn hơn bị khách bắt gặp.

---
## [03] GET /v1/suggest — gợi ý gõ phím có trọng số theo độ phổ biến
### ① ĐO TRƯỚC — trọng số nằm sẵn trong bảng
```bash
q miniai_search "SELECT term, round(weight,2) FROM suggest_terms WHERE project_id='demoshop' AND term LIKE 'mì%' ORDER BY weight DESC LIMIT 4;"
```
### ② GỌI API
```bash
curl -s "localhost:16021/v1/suggest?q=mi&limit=4" -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" | .venv/bin/python -m json.tool
```
**OUTPUT thật 12/08**
```json
{"items": [{"text": "mì hảo hảo", "weight": 376.96}, {"text": "mì", "weight": 376.96},
           {"text": "mì hảo", "weight": 376.96}, {"text": "miếng", "weight": 353.00}],
 "consistency": {"projection_watermark": …, "data_as_of": "…", "is_caught_up": true, "ledger_head": …}}
```
### ③ ĐO SAU — số của API **phải khớp** số trong bảng
```bash
q miniai_search "SELECT term, round(weight,2) FROM suggest_terms WHERE project_id='demoshop' AND term IN ('mì','mì hảo hảo') ORDER BY weight DESC;"
```
### ④ LUỒNG + điểm khoe
```
products.title ─┐
query_log.cnt  ─┴─job suggest_terms (mỗi 1 GIỜ)──► suggest_terms.weight ──► /v1/suggest
```
> SKU này đã có sẵn cụm từ nên **không cần kích job** — khác DEMO-1 [03], hàng mới tinh phải kích tay.
⭐ **`weight = 376.96` so với `1.0` của hàng mới ở DEMO-1** — khách gõ **2 chữ cái** đã gợi được cụm đầy đủ
**có dấu**. Đây chính là **giá trị của dữ liệu tích luỹ**, nhìn thấy bằng một con số.

---
## [04] POST /v1/recommend (context=pdp) — mua kèm thật, học từ hành vi
### ① ĐO TRƯỚC — tri thức "mua chung" nằm ở đâu
```bash
q miniai_search "SELECT product_b, cnt, round(lift,2) FROM co_occurrence WHERE project_id='demoshop' AND product_a='$SKU' ORDER BY lift DESC LIMIT 3;"
q miniai_search "SELECT count(*) FROM reco_exposure WHERE project_id='demoshop';"
```
### ② GỌI API
```bash
curl -s localhost:16021/v1/recommend -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"context":"pdp","product_id":"bh-mi-haohao"}' | .venv/bin/python -c "import json,sys; [print(round(i['score'],1),'|',i['product_id'],'|',i['title'][:40]) for i in json.load(sys.stdin)['items'][:3]]"
```
**OUTPUT thật 12/08**
```
148.4 | bh-snack-oishi     | Snack Oishi tôm cay 42g (lốc 10)
119.8 | bh-xucxich-ducviet | Xúc xích tiệt trùng Đức Việt gói 500g
  0.3 | bh-hatdieu-500g    | Hạt điều rang muối Bình Phước 500g
```
### ③ ĐO SAU
```bash
sleep 2; q miniai_search "SELECT count(*) FROM reco_exposure WHERE project_id='demoshop';"
q miniai_search "SELECT product_id, position FROM reco_exposure WHERE project_id='demoshop' ORDER BY ts DESC LIMIT 3;"
```
**Đo thật:** `+12 dòng`, 3 dòng mới nhất đúng thứ tự vừa hiện.
### ④ LUỒNG — **so sánh trực tiếp với DEMO-1**
```
purchase.completed ──job co_occurrence (24h)──► co_occurrence(lift) ──► reco_pdp thang TRĂM
                                              (hàng mới chưa có ⇒ rơi xuống nấc nội dung, thang 0-1)
```
⭐ **Nhìn thang điểm là biết đang ở nấc nào:** 148.4 / 119.8 = **điểm mua-kèm thật** (thang trăm);
0.3 = nấc nội dung (thang 0-1) — đúng bằng thang mà DEMO-1 nhận được. *"Mì gói đi với snack và xúc xích —
tri thức **học từ hành vi mua chung**, không ai lập trình tay."*

---
## [05] POST /v1/recommend (context=similar) — hàng thay thế
### ② GỌI API
```bash
curl -s localhost:16021/v1/recommend -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"context":"similar","product_id":"bh-mi-haohao"}' | .venv/bin/python -c "import json,sys; d=json.load(sys.stdin); [print(round(i['score'],3),'|',i['title'][:40]) for i in d['items'][:4]]; print('source:',d['items'][0].get('source'))"
```
### ③ ĐO SAU
```bash
q miniai_search "SELECT context, count(*) FROM reco_exposure WHERE project_id='demoshop' AND ts > now()-interval '2 min' GROUP BY 1;"
```
**Đo thật:** thấy đúng `context=similar` vừa được ghi.
### ④ Đọc kết quả
`source: reco_similar`, thang **0-1** (khác `reco_pdp` thang trăm) vì đây là **độ tương đồng nội dung**.
Trung thực: chùm điểm sát nhau ⇒ vector phân biệt ngành còn yếu, đã ghi sổ.

---
## [06] POST /v1/recommend (context=cart) — gợi ý trong giỏ hàng
### ② GỌI API
```bash
curl -s localhost:16021/v1/recommend -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"context":"cart","product_ids":["bh-mi-haohao"],"user_pseudo_id":"demo-user-01"}' | .venv/bin/python -c "import json,sys; [print(round(i['score'],1),'|',i['title'][:40]) for i in json.load(sys.stdin)['items'][:3]]"
```
### ③ ĐO SAU — hồ sơ người dùng có tồn tại không?
```bash
q miniai_search "SELECT user_pseudo_id, events_count FROM user_profile WHERE project_id='demoshop' ORDER BY events_count DESC LIMIT 3;"
```
### ④ Đọc kết quả
Giỏ hàng gợi theo **giá trị đơn hàng tăng thêm**, nên cross-ngành là **có chủ đích** (khác PDP phải đúng ngành).
> Thiếu `user_pseudo_id` → `400 INVALID_ARGUMENT: 'user_pseudo_id' is required for context=cart`.

---
## [07] POST /v1/ask — hỏi tự nhiên, có chặn bịa + lọc lệch ngành
### ② GỌI API
```bash
curl -s localhost:16021/v1/ask -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"question":"tai nghe chong on tot khong?"}' | .venv/bin/python -c "
import json,sys; d=json.load(sys.stdin)
print(d['answer'])
print('--- nguon:', d['answer_source'], '| chan bia:', d['grounding_guard']['blocked'])
print('--- loc lech nganh:', d['answer_coherence'])"
```
**OUTPUT thật (ổn định 3/3 lần đo)**
```
Gợi ý cho bạn:
1. Tai nghe Bluetooth chụp tai Sony WH-CH520 (990.000đ)
2. Tai nghe Bluetooth TWS Baseus Bowie E2 (195.000đ)
3. Tai nghe gaming Havit H2002D có mic (449.000đ)
--- nguon: template | chan bia: False
--- loc lech nganh: {'filtered': True, 'dropped_ids': ['gd-chao-locklock', 'th-gangtay-gym']}
```
### ③ ĐO SAU — chứng minh 2 sản phẩm bị loại thuộc ngành khác
```bash
q miniai_search "SELECT product_id, category_l1 FROM products WHERE project_id='demoshop' AND product_id IN ('gd-chao-locklock','th-gangtay-gym');"
q miniai_search "SELECT product_id, category_l1 FROM products WHERE project_id='demoshop' AND product_id LIKE 'dt-tainghe%' LIMIT 3;"
```
**Đo thật:** 2 SKU bị loại thuộc **Gia dụng** và **Thể thao**; 3 SKU được giữ thuộc **Điện tử — tai nghe**.
### ④ LUỒNG — 3 tầng bảo vệ
```
câu hỏi ──► retrieval ──► grounding_guard (mã hàng bịa ⇒ CHẶN)
                     ──► answer_coherence (lệch ngành ⇒ LOẠI)  ──► answer + khai answer_source
```
> 💡 **Chọn câu hỏi khi demo:** dùng câu có **danh từ ngành rõ ràng** (tai nghe / kem chống nắng / ốp lưng).
> **Tránh** hỏi về mì: âm tiết "mi" quá ngắn khi bỏ dấu, vị trí 2-3 dễ lẫn hàng khác.

---
## [08] GET /internal/similar-products — hàng xóm theo vector
### ② GỌI API
```bash
curl -s "localhost:16021/internal/similar-products?project_id=demoshop&product_id=bh-mi-haohao&limit=5" -H "X-Internal-Token: $ITOK" | .venv/bin/python -m json.tool
```
### ③ ĐO SAU — hàng xóm có lịch sử để mượn không
```bash
q miniai_forecast "SELECT product_id, count(*) AS ngay FROM demand_daily WHERE project_id='demoshop' GROUP BY 1 ORDER BY 2 DESC LIMIT 5;"
```
### ④ LUỒNG
```
forecast/decision ──X-Internal-Token──► smartsearch /internal/* ──► vector Vespa
```
**Nói với khách:** *"Ranh giới nội bộ/công khai rõ ràng: hai service nói chuyện bằng token riêng, không dùng
key của khách hàng."*

---
## [09] GET /internal/products-by-category — SKU theo ngành
### ② GỌI API (thử **cả 3 cách viết dấu**)
```bash
for c in "B%C3%A1ch%20h%C3%B3a" "Bach%20hoa" "BACH%20HOA"; do
  echo -n "  $c -> "; curl -s "localhost:16021/internal/products-by-category?project_id=demoshop&category_l1=$c&limit=50" -H "X-Internal-Token: $ITOK" | .venv/bin/python -c "import json,sys; print(len(json.load(sys.stdin)['product_ids']),'SKU')"
done
```
### ③ ĐO SAU — đối chiếu với kho
```bash
q miniai_search "SELECT count(*) FROM products WHERE project_id='demoshop' AND category_l1='Bách hóa';"
```
**Đo thật:** cả 3 cách viết trả **CÙNG một con số**, khớp với `count(*)` trong bảng.
### ④ Điểm khoe
🆕 Từ 06/08 **hết phân biệt dấu**. Trước đây khách khai không dấu sẽ tạo ra một **"ngành ma"** song song và
mọi số gộp theo ngành đều lệch — lỗi im lặng, không ai phát hiện.

---
## [10] POST /v1/events:ingest — bơm hành vi thật đang diễn ra
### ① ĐO TRƯỚC
```bash
q miniai_search "SELECT count(*) AS raw FROM raw_events WHERE project_id='demoshop';"
q miniai_search "SELECT coalesce(max(events_count),0) FROM user_profile WHERE project_id='demoshop' AND user_pseudo_id='demo-user-01';"
```
### ② GỌI API
```bash
curl -s localhost:16021/v1/events:ingest -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d "{\"events\":[{\"event_id\":\"demo-view-$(date +%s)\",\"schema_version\":\"1.0\",\"event_type\":\"product.viewed\",\"event_time\":\"$EVT\",\"user_pseudo_id\":\"demo-user-01\",\"payload\":{\"product_id\":\"$SKU\"}},{\"event_id\":\"demo-cart-$(date +%s)\",\"schema_version\":\"1.0\",\"event_type\":\"cart.added\",\"event_time\":\"$EVT\",\"user_pseudo_id\":\"demo-user-01\",\"payload\":{\"product_id\":\"$SKU\",\"qty\":2}}]}" | .venv/bin/python -m json.tool
```
**OUTPUT thật:** `{"accepted": 2, "deduped": 0, "skipped": 0, "errors": [], "ledger_position": …}`
### ③ ĐO SAU
```bash
q miniai_search "SELECT count(*) AS raw FROM raw_events WHERE project_id='demoshop';"
q miniai_search "SELECT event_type, event_time FROM raw_events WHERE project_id='demoshop' ORDER BY received_at DESC LIMIT 2;"
```
**Đo thật:** `raw +2`, và thấy đúng 2 loại `product.viewed` / `cart.added` vừa bắn.
### ④ LUỒNG
```
:ingest ──► raw_events (sổ cái ghi-một-lần) ──job learning (5 phút / 1 giờ / 24 giờ)──►
            user_profile · popularity · co_occurrence ──► nuôi [04][06]
```
> ⚠ `cart.added` dùng trường **`qty`** (không phải `quantity`) — gõ sai thì **chỉ sự kiện đó** bị từ chối
> (`errors[].index` chỉ đúng vị trí), các sự kiện còn lại vẫn nhận. Thiết kế **từ chối từng dòng**.
**Nói với khách:** *"`ledger_position` tăng — mọi sự kiện vào **sổ cái ghi-một-lần**; không sửa, không xoá.
Muốn đảo một giao dịch thì ghi bút toán đảo, giống kế toán."*

---
# PHẦN B — FORECAST (11 API)

## [11] POST /v1/forecast:run — chạy dự báo (bất đồng bộ)
### ① ĐO TRƯỚC
```bash
q miniai_forecast "SELECT run_id, count(*) FROM forecasts WHERE project_id='demoshop' GROUP BY 1 ORDER BY 1 DESC LIMIT 3;"
```
### ② GỌI API
```bash
curl -s -w "\nstatus: %{http_code}\n" -X POST localhost:16023/v1/forecast:run -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{}'
```
**OUTPUT thật:** `202` + `{"status":"queued","run_id":"r_2026-08-13","job_id":"fr-demoshop-r_2026-08-13"}`
### ③ ĐO SAU
```bash
q miniai_forecast "SELECT job_id, status, attempt FROM job_run WHERE tenant_id='demoshop' AND job_type='forecast_run' ORDER BY updated_at DESC LIMIT 1;"
```
**Đo thật:** dòng `queued` — **nhìn thấy việc nằm trong hàng đợi**.
### ④ LUỒNG: xem DEMO-1 [11]. Đo thật: **~60 giây** cho 134 SKU.

---
## [12] GET /v1/projections/status — chờ job xong rồi mới đo tiếp
### ② GỌI API (vòng lặp **bắt buộc**)
```bash
JOB="fr-demoshop-r_$(date -u +%F)"
until curl -s "localhost:16023/v1/projections/status?job_id=$JOB" -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" | .venv/bin/python -c "
import json,sys; s=(json.load(sys.stdin).get('job') or {}).get('status'); print('   trang thai:',s); sys.exit(0 if s in ('done','failed') else 1)"; do sleep 5; done
```
**OUTPUT thật:** `queued → running → done`
### ③ ĐO SAU
```bash
q miniai_forecast "SELECT status, attempt, coalesce(error_code,'-') FROM job_run WHERE job_id='$JOB';"
q miniai_forecast "SELECT run_id, count(*) AS dong, count(DISTINCT product_id) AS sku FROM forecasts WHERE project_id='demoshop' GROUP BY 1 ORDER BY 1 DESC LIMIT 2;"
```
**Đo thật:** mẻ mới có **≈134 SKU × 28 ngày ≈ 3.752 dòng**.
### ④ Nói với khách
*"`is_caught_up: true` = hình chiếu đã bắt kịp sổ cái; `job.status = done` = **giờ mới được đo** các API dưới.
Nếu `failed` thì có `error_code` kèm — lỗi nhìn thấy được, không nuốt."*

---
## [13] POST /v1/forecast:query — dải dự báo P10/P50/P90
### ① ĐO TRƯỚC — số nằm sẵn trong bảng
```bash
q miniai_forecast "SELECT model_used, data_window, count(*) FROM forecasts WHERE project_id='demoshop' AND product_id='$SKU' GROUP BY 1,2;"
```
### ② GỌI API
```bash
curl -s localhost:16023/v1/forecast:query -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"bh-mi-haohao","horizon_days":14}' | .venv/bin/python -c "
import json,sys; d=json.load(sys.stdin)
print('run_id     =', d['run_id']); print('model_used =', d['model_used']); print('data_window=', d['data_window'])
print('calibration=', json.dumps(d['calibration']))
print('3 ngay dau =', [(x['day'][5:], round(x['p10'],2), round(x['p50'],2), round(x['p90'],2)) for x in d['daily'][:3]])"
```
**OUTPUT thật 12/08** (đo lại sau bản vá chiều 12/08)
```
run_id     = r_2026-08-12
model_used = lgbm_global
data_window= 2026-04-01..2026-08-11     ← kết ở HÔM QUA, xem ghi chú dưới
calibration= {"width_factor": 1.2004, "empirical_coverage": 0.7143}
3 ngay dau = [('08-13', 1.65, 3.93, 7.69), ('08-14', 1.59, 4.19, 7.59), ('08-15', 1.42, 5.09, 8.25)]
```
> 🆕 **`data_window` kết ở HÔM QUA chứ không phải hôm nay — và đó là điểm đúng đắn** (vá 12/08).
> `rollup` sinh dòng cho cả ngày hôm nay, mà ngày đó **chưa đóng sổ** (9 giờ sáng mới có doanh số 9 tiếng).
> Trước bản vá, mẫu mùa vụ lấy nhầm ngày dở dang nên **cứ 7 ngày dự báo có 1 ngày bị ép xuống gần 0**.
> *"Hệ chỉ học từ những ngày đã chốt sổ — hôm nay còn đang bán thì chưa tính."*
### ③ ĐO SAU — **API trả đúng số trong bảng**
```bash
q miniai_forecast "SELECT horizon_day, round(p10,2), round(p50,2), round(p90,2) FROM forecasts WHERE project_id='demoshop' AND product_id='$SKU' ORDER BY horizon_day LIMIT 3;"
```
**Hai bảng số phải TRÙNG KHÍT** — tầng đọc **không tính lại gì**, chỉ đọc kết quả đã đông lạnh.
### ④ Điểm khoe
- `model_used = lgbm_global` — SKU này được **chấm điểm và chọn** mô hình LightGBM quantile, không phải
  dùng một mô hình cho tất cả. Hệ có **thang 9 mô hình**.
- `data_window = 2026-04-01..2026-08-11` — **học trên 133 ngày đã chốt sổ** (DEMO-1 hàng mới: `null`).
- `calibration.empirical_coverage = 0.7143` — hệ **đo được** khoảng của nó bao 71,4% thực tế (hứa 80%),
  nên `width_factor = 1.20` **nới khoảng ra** cho trung thực. *"Nó tự biết mình đang hẹp và tự sửa."*

---
## [14] POST /v1/forecast:aggregate — dự báo gộp theo ngành
### ① ĐO TRƯỚC
```bash
q miniai_search "SELECT count(*) AS sku_trong_nganh FROM products WHERE project_id='demoshop' AND category_l1='Bách hóa';"
```
### ② GỌI API
```bash
curl -s localhost:16023/v1/forecast:aggregate -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"category_l1":"Bách hóa","horizon_days":7}' | .venv/bin/python -m json.tool | head -20
```
### ③ ĐO SAU — số SKU gộp **phải khớp** số SKU trong ngành
```bash
# so sánh resolved_product_ids của API với danh sách trong kho
q miniai_search "SELECT string_agg(product_id, ' ' ORDER BY product_id) FROM products WHERE project_id='demoshop' AND category_l1='Bách hóa';"
```
### ④ Đọc kết quả
`resolved_product_ids` **liệt kê đích danh** từng SKU đã gộp — chủ shop kiểm được có bỏ sót hàng nào không.
⚠ Kỹ thuật: **phân vị không cộng được** (p90 của tổng ≠ tổng các p90) nên phần gộp đi qua tầng mô phỏng,
không cộng thô.

---
## [15] GET /v1/forecast:accuracy — **HỆ TỰ CHẤM ĐIỂM CHÍNH MÌNH**
### ① ĐO TRƯỚC — bảng chấm điểm nằm ở đâu
```bash
q miniai_forecast "SELECT model, count(*) AS lan_cham, round(avg(mase)::numeric,3) AS mase FROM backtest_results WHERE project_id='demoshop' GROUP BY 1 ORDER BY 3;"
```
### ② GỌI API
```bash
curl -s "localhost:16023/v1/forecast:accuracy?window=90d" -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" | .venv/bin/python -c "
import json,sys; d=json.load(sys.stdin)
for m in d['by_model']: print(f\"  {m['model']:<24} {str(m.get('segment')):<13} sku={m['sku_count']:<4} mase={m['mase_avg']} cov={m['coverage_p10_p90']}\")"
```
**OUTPUT thật 12/08**
```
  adida                    intermittent  sku=74   mase=0.891  cov=0.909
  autoets_theta_ensemble   smooth        sku=71   mase=0.811  cov=0.867
  croston_auto             intermittent  sku=74   mase=0.897  cov=0.943
  imapa                    intermittent  sku=74   mase=0.891  cov=0.908
  lgbm_global              intermittent  sku=74   mase=0.782  cov=0.844
```
### ③ ĐO SAU — API và bảng chấm điểm **phải khớp**
```bash
q miniai_forecast "SELECT model, round(avg(mase)::numeric,3), round(avg(coverage_p10_p90)::numeric,3) FROM backtest_results WHERE project_id='demoshop' AND created_at > now()-interval '90 days' GROUP BY 1 ORDER BY 2;"
```
### ④ Đọc kết quả — API thuyết phục nhất với khách khó tính
- **`mase`** — sai số so với cách ngây thơ "tuần trước bán bao nhiêu, tuần này bấy nhiêu". **< 1 là thắng**;
  ở đây **0,78-0,90** ⇒ tốt hơn baseline **10-22%**.
- **`coverage_p10_p90`** — thực tế rơi vào khoảng [P10,P90] bao nhiêu %. Lý tưởng ≈ **0,80**.
  0,844 gần đích nhất; 0,943 = **hơi rộng** (an toàn nhưng nhập dư).
  ⚠ **Coverage cao KHÔNG phải thành tích** — chệch lên cũng là lỗi hiệu chuẩn.
- **`segment`** — hệ chia hàng *bán đều* / *bán lai rai* và dùng mô hình khác nhau.

**Câu nói:** *"Hệ tự công bố điểm số của chính nó, **kể cả chỗ chưa đẹp**. Anh chị không phải tin lời tôi —
số này lấy thẳng từ bảng `backtest_results`, tôi vừa truy vấn trước mặt anh chị."*

---
## [16] POST /v1/forecast:insights — insight nhu cầu
### ② GỌI API
```bash
curl -s localhost:16023/v1/forecast:insights -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"kind":"top_movers","window_days":30}' | .venv/bin/python -m json.tool | head -18
```
⚠ `kind` **số ít**, một trong: `accuracy_sku` · `top_movers` · `group_breakdown` · `seasonality` ·
`sell_through_prob` · `promo_uplift_learned`.
### ③ ĐO SAU — tự kiểm bảng xếp hạng
```bash
q miniai_forecast "SELECT product_id, round(sum(p50),2) AS tong_p50 FROM forecasts WHERE project_id='demoshop' AND run_id=(SELECT max(run_id) FROM forecasts WHERE project_id='demoshop') AND horizon_day > CURRENT_DATE GROUP BY 1 ORDER BY 2 DESC LIMIT 3;"
```
**Số của API và của SQL phải khớp** — vì API cũng chỉ cộng đúng cột đó.
### ④ Điểm trung thực
`ranking_note` **tự khai giới hạn của chính nó**: *"cộng thẳng p50, chỉ dùng để xếp thứ tự"* — vì cộng phân vị
là sai về toán; muốn số tổng chính xác phải qua tầng kịch bản. *"Một hệ trung thực là hệ ghi rõ chỗ nó xấp xỉ."*

🆕 **Thử luôn `kind=promo_uplift_learned`** (vá 12/08 — trước đây gộp chung mọi SKU nên có tenant khai
`uplift = 0.0`, tức "giảm giá làm bán 0 cái"):
```bash
curl -s localhost:16023/v1/forecast:insights -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"kind":"promo_uplift_learned"}' | .venv/bin/python -m json.tool
```
Nay báo **trung vị theo từng SKU** + **biên độ min/max giữa các SKU** + số SKU đủ dữ liệu.

---
## [17] POST /v1/forecast:promo-preview — "giảm 30% thì bán thêm bao nhiêu?"
### ① ĐO TRƯỚC — hệ số uplift học từ đâu
```bash
q miniai_forecast "SELECT v FROM kv_state WHERE k='promo_uplift_k:demoshop';"
q miniai_forecast "SELECT count(*) AS ngay_da_tung_sale FROM demand_daily WHERE project_id='demoshop' AND product_id='$SKU' AND promo_pct > 0;"
```
**Đo thật:** `k ≈ 0.947` học từ **48 SKU / 719 ngày sale** — *"hệ số này **học từ chính shop này**, không phải
hằng số bịa."*
### ② GỌI API
```bash
S=$(date -u -d "+3 days" +%F); E=$(date -u -d "+10 days" +%F)
curl -s localhost:16023/v1/forecast:promo-preview -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d "{\"product_id\":\"bh-mi-haohao\",\"discount_pct\":30,\"start\":\"$S\",\"end\":\"$E\"}" | .venv/bin/python -c "
import json,sys; d=json.load(sys.stdin)
for x in d['daily'][:8]: print(f\"  {x['day'][5:]}  p50={x['p50']:.2f}  promo={x['promo']}\")"
```
### ③ ĐO SAU — tự kiểm mức nhảy
```bash
# so p50 ngày KHÔNG promo với ngày CÓ promo, đối chiếu với công thức (1 + k*0.30)
.venv/bin/python -c "k=0.947; print(f'thua so ky vong = 1 + {k}*0.30 = {1+k*0.30:.4f}')"
```
**Đo thật:** đúng ngày khuyến mại `p50` **nhảy ~+28-40%**, khớp thừa số `(1 + k×0,30)`.
⚠ **Không ghi vào DB** — đây là API *thử-nếu-thì*, `forecasts` không đổi:
```bash
q miniai_forecast "SELECT count(*) FROM forecasts WHERE project_id='demoshop' AND product_id='$SKU';"
```
### ④ LUỒNG
```
promo.scheduled quá khứ ──► demand_daily.promo_pct ──học k──► kv_state
                                                          └──:promo-preview nhân (1+k·giảm) ──► trả về
```

---
## [18] POST /v1/scenarios:build — dựng 128 kịch bản
### ① ĐO TRƯỚC
```bash
q miniai_forecast "SELECT count(*) FROM scenario_manifest WHERE project_id='demoshop';"
```
### ② GỌI API
```bash
RUN=$(curl -s localhost:16023/v1/scenarios:build -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_ids":["bh-mi-haohao"],"horizon_days":7,"scenario_count":128}' | .venv/bin/python -c "import json,sys; d=json.load(sys.stdin); print(d['run_id'])")
echo "RUN=$RUN"
```
### ③ ĐO SAU — manifest + tệp trên đĩa
```bash
q miniai_forecast "SELECT run_id, created_at FROM scenario_manifest WHERE project_id='demoshop' ORDER BY created_at DESC LIMIT 1;"
# ⚠ tệp nằm TRONG CONTAINER (MINIAI_ARTIFACT_DIR=/srv/data/artifacts), KHÔNG ở data/ trên host
docker exec miniai-forecast ls -la /srv/data/artifacts/scenario/demoshop/$RUN/
```
**Đo thật:** 3 tệp `marginals.npz` · `factors.npz` · `manifest.json`, mỗi tệp có **SHA-256 trong manifest**.
### ④ Điểm khoe
`rng_algorithm: philox` **có hạt giống** ⇒ chạy lại ra **đúng bộ kịch bản cũ**, kiểm toán được.

> 🆕 **ĐÍNH CHÍNH (đo 12/08):** ba API kịch bản bên dưới **KHÔNG bắt buộc** truyền `run_id` — thiếu thì chúng
> **tự lấy mẻ mới nhất** và trả `200` kèm `run_id` đã dùng (bản trước ghi "thiếu `run_id` → 400", **sai**).
> Vẫn nên truyền `$RUN` tường minh khi demo để **chỉ đích danh** mẻ đang nói tới; và phải có **ít nhất một
> mẻ đã `:build`** thì mới có gì để lấy.

---
## [19] POST /v1/scenarios:lead-time-demand — cầu trong thời gian chờ hàng
### ② GỌI API
```bash
curl -s localhost:16023/v1/scenarios:lead-time-demand -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d "{\"product_ids\":[\"bh-mi-haohao\"],\"run_id\":\"$RUN\",\"lead_time_days\":3,\"review_period_days\":2,\"required_quantiles\":[0.5,0.9],\"horizon_days\":7}" | .venv/bin/python -m json.tool
```
### ③ ĐO SAU — đối chiếu với dự báo ngày
```bash
q miniai_forecast "SELECT round(sum(p50),2) AS tong_p50_5ngay FROM forecasts WHERE project_id='demoshop' AND product_id='$SKU' AND horizon_day > CURRENT_DATE AND horizon_day <= CURRENT_DATE + 5;"
```
**Điểm nhấn:** con số kịch bản **gần** tổng p50 nhưng **không bằng** — vì `q0.9` của tổng ≠ tổng các `p90`.
### ④ Dịch sang lời chủ shop
*"Từ lúc đặt hàng tới lúc hàng về (3 ngày) cộng chu kỳ kiểm kho (2 ngày), tôi cần chuẩn bị khoảng **q0.5**
thùng cho trường hợp bình thường, và **q0.9** thùng nếu muốn 90% chắc chắn không cháy hàng."* Đây chính là
con số để đặt đơn nhập.

---
## [20] POST /v1/scenarios:aggregate — gộp kịch bản
```bash
curl -s localhost:16023/v1/scenarios:aggregate -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d "{\"product_ids\":[\"bh-mi-haohao\"],\"run_id\":\"$RUN\",\"horizon_days\":7}" | .venv/bin/python -m json.tool
```
**Đọc kết quả:** `method: scenario_mc_128` khai rõ số này tính bằng **mô phỏng 128 kịch bản**, không phải
cộng phân vị.

---
## [21] POST /v1/scenarios:probability — xác suất vượt ngưỡng
```bash
curl -s localhost:16023/v1/scenarios:probability -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d "{\"run_id\":\"$RUN\",\"product_id\":\"bh-mi-haohao\",\"threshold_units\":30,\"horizon_days\":7}" | .venv/bin/python -m json.tool
```
**Đọc kết quả:** *"Xác suất bán được từ 30 thùng trở lên trong 7 ngày tới là **X%**"* — tính bằng số kịch bản
vượt ngưỡng / 128. Dạng câu hỏi chủ shop hỏi thật khi cân nhắc ôm hàng theo lô.
⚠ Tham số là **`threshold_units`** (không phải `threshold`) và **`product_id`** số ít.

---
# PHẦN C — DECISION (9 API)

## [22] GET /v1/config — chính sách giá của tenant
### ① ĐO TRƯỚC
```bash
q miniai_decision "SELECT key, value FROM project_config WHERE project_id='demoshop';"
```
### ② GỌI API
```bash
curl -s localhost:16022/v1/config -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | .venv/bin/python -m json.tool
```
**OUTPUT thật:** `{"config": {"promo_cap_pct": 50, "pricing_mode": "lerner"}}`
### ③ ĐO SAU — **API đọc đúng bảng** (2 số phải khớp)
### ④ Đọc kết quả
`promo_cap_pct: 50` = trần giảm giá 50% (đúng NĐ 81/2018), hệ **không bao giờ** đề xuất vượt.
`pricing_mode` đổi được **per-tenant, không cần khởi động lại**:
- **`lerner`** — tối ưu lợi nhuận kỳ vọng (mặc định)
- **`robust`** — thận trọng, cân nhắc rủi ro đuôi; bất định lớn thì **giữ giá** thay vì mạo hiểm

---
## [23] POST /v1/events:ingest — nạp tồn kho + giá vốn
### ① ĐO TRƯỚC
```bash
q miniai_decision "SELECT on_hand_qty FROM stock_state WHERE project_id='demoshop' AND product_id='$SKU';"
q miniai_decision "SELECT round(ewma_cost) AS von, n_receipts FROM cost_state WHERE project_id='demoshop' AND product_id='$SKU';"
```
**Đo thật 12/08:** `tồn = 137` · `vốn = 70145`
### ② GỌI API
```bash
curl -s localhost:16022/v1/events:ingest -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d "{\"events\":[{\"event_id\":\"demo-stock-$(date +%s)\",\"schema_version\":\"1.0\",\"event_type\":\"stock.level\",\"event_time\":\"$EVT\",\"user_pseudo_id\":\"system\",\"payload\":{\"product_id\":\"$SKU\",\"on_hand_qty\":120}},{\"event_id\":\"demo-cost-$(date +%s)\",\"schema_version\":\"1.0\",\"event_type\":\"cost.recorded\",\"event_time\":\"$EVT\",\"user_pseudo_id\":\"system\",\"payload\":{\"product_id\":\"$SKU\",\"unit_cost\":69500,\"qty\":50}}]}" | .venv/bin/python -m json.tool
```
### ③ ĐO SAU — **⚠ ĐÂY LÀ ĐIỂM DỄ VẤP NHẤT CỦA CẢ BUỔI**
```bash
echo "ngay sau : ton=$(q miniai_decision "SELECT on_hand_qty FROM stock_state WHERE project_id='demoshop' AND product_id='$SKU'")"
# vòng state_rollup chạy mỗi 300 GIÂY — demo thì kích tay cho nhanh:
docker exec miniai-decision python3 -c "
import asyncio, asyncpg, os, json
from app.jobs.state_rollup import run_state_rollup_once
async def m():
    p=await asyncpg.create_pool(os.environ.get('DECISION_DSN') or os.environ.get('DATABASE_URL'),min_size=1,max_size=3)
    print('state_rollup:', json.dumps(await run_state_rollup_once(p))); await p.close()
asyncio.run(m())"
echo "sau job  : ton=$(q miniai_decision "SELECT on_hand_qty FROM stock_state WHERE project_id='demoshop' AND product_id='$SKU'")  von=$(q miniai_decision "SELECT round(ewma_cost) FROM cost_state WHERE project_id='demoshop' AND product_id='$SKU'")"
```
**Đo thật:** `ngay sau: ton=137` (**CHƯA đổi**) → `sau job: ton=120` ✅
### ④ LUỒNG
```
:ingest ──► raw_events (NGAY, sổ cái) ──job state_rollup (300s)──► stock_state · cost_state · price_state
```
**Nói với khách:** *"Sổ cái ghi ngay; **hình chiếu** cập nhật theo nhịp. Nếu tôi không nói trước, anh chị sẽ
tưởng hệ hỏng khi thấy tồn kho chưa đổi. Đây là **tách sổ cái khỏi hình chiếu** — sổ là sự thật, các bảng
trạng thái chỉ là ảnh chụp dựng lại từ sổ, hỏng thì dựng lại được."*
Giá vốn cộng dồn theo **bình quân trượt có trọng số (EWMA)** — lô mới ảnh hưởng nhiều hơn lô cũ.

---
## [24] POST /v1/decisions:run — kích bộ não quyết định
### ① ĐO TRƯỚC
```bash
q miniai_decision "SELECT count(*) AS tong, count(*) FILTER (WHERE status='open') AS dang_mo FROM decisions WHERE project_id='demoshop';"
```
### ② GỌI API
```bash
curl -s -X POST localhost:16022/v1/decisions:run -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{}' | .venv/bin/python -m json.tool
```
**OUTPUT thật 12/08** (số đổi theo ngày)
```json
{"created": 2, "skipped_dedup": 149,
 "skipped_by_reason": {"anti_oscillation": 142, "plan_conflict": 84,
                       "insufficient_history": 2, "no_stock": 2, "no_cost": 57},
 "superseded_plan": 1}
```
### ③ ĐO SAU
```bash
q miniai_decision "SELECT kind, count(*) FROM decisions WHERE project_id='demoshop' AND created_at > now()-interval '5 min' GROUP BY 1 ORDER BY 2 DESC;"
```
**Số dòng mới phải bằng đúng `created`.**
### ④ Điểm cần nhấn
`anti_oscillation: 142` — hệ **chủ động im lặng** với 142 SKU vừa đổi giá gần đây.
*"Một hệ khuyên đổi giá mỗi ngày là hệ làm hại chủ shop. Chốt chặn này quan trọng ngang bản thân thuật toán."*

---
## [25] GET /v1/decisions — hàng đợi lời khuyên
### ② GỌI API
```bash
curl -s "localhost:16022/v1/decisions?page_size=5" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | .venv/bin/python -c "
import json,sys
for x in json.load(sys.stdin)['items']:
    ev=x.get('expected_value') or {}
    print(x['decision_id'][:20],'|',x['kind'],'|',x['status'],'| EV', format(ev.get('amount',0),',.0f'),'d/',ev.get('per',''))"
```
### ③ ĐO SAU — đọc trọn `trace` của 1 quyết định (điểm khoe)
```bash
q miniai_decision "SELECT trace FROM decisions WHERE project_id='demoshop' AND kind='bundle_suggestion' ORDER BY created_at DESC LIMIT 1;"
```
**Đo thật — ví dụ `trace` thật:**
```
lift=18.63 (>=2.0), pair_cnt=44 (>=5); margin_a=33.80%, margin_b=22.30% (both >15%);
bundle_price=130000 voucher=7000 …; EV = 0.15*44*(33466+8474) = 276807
```
### ④ Nói với khách
*"**Toàn bộ phép tính viết ra bằng chữ** — chủ shop tự kiểm được từng bước, không phải hộp đen. Anh chị vừa
thấy cả ngưỡng (`>=2.0`, `>=5`, `>15%`) lẫn phép nhân cuối cùng."*
🆕 **Lọc theo SKU được rồi** (vá 12/08): `?product_id=bh-mi-haohao`. Trước đây tham số này bị **bỏ qua im
lặng** nên phải lọc tay phía client.

---
## [26] GET /v1/decisions:stats — thống kê + tỷ lệ chấp nhận
### ① ĐO TRƯỚC
```bash
q miniai_decision "SELECT kind, count(*) FROM decisions WHERE project_id='demoshop' AND created_at > now()-interval '30 days' GROUP BY 1 ORDER BY 2 DESC;"
q miniai_decision "SELECT count(*) AS so_phan_hoi FROM feedback WHERE project_id='demoshop';"
```
### ② GỌI API
```bash
curl -s "localhost:16022/v1/decisions:stats?window=30d" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | .venv/bin/python -m json.tool | head -25
```
### ③ ĐO SAU — **`count` của API phải khớp SQL ở bước ①**
### ④ Đọc kết quả
**`accepted_rate` là thước đo hệ tự soi mình** — tỷ lệ chủ shop thực sự làm theo.
`price_hold` nhiều hơn `price_suggestion` = hệ nói "giữ giá" nhiều hơn "đổi giá", **đúng tinh thần thận trọng**.
`below_cost_alert` = số ca đang bán dưới vốn đã bắt được.

---
## [27] GET /v1/decisions:insights — insight tổng hợp
### ② GỌI API
```bash
curl -s "localhost:16022/v1/decisions:insights?kind=capital_locked" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | .venv/bin/python -m json.tool
```
⚠ `kind` **số ít**: `capital_locked` | `advice_scorecard` | `monthly_benefit` | `removal_candidates` |
`bundle_candidates` | `golden_hours`.
### ③ ĐO SAU — tự kiểm định nghĩa "hàng ế" bằng SQL
```bash
q miniai_decision "SELECT count(*) AS hang_e FROM stock_state s WHERE s.project_id='demoshop' AND s.on_hand_qty>0 AND NOT EXISTS (SELECT 1 FROM sales_daily d WHERE d.project_id='demoshop' AND d.product_id=s.product_id AND d.day >= CURRENT_DATE-30 AND d.units>0);"
```
**Số này phải khớp `n_slowmovers` của API.**
### ④ Điểm khoe
Trường **`basis` định nghĩa rõ thế nào là hàng ế ngay trong kết quả** — không để chủ shop đoán tiêu chí,
và anh chị vừa **tự viết lại đúng định nghĩa đó bằng SQL** để đối chiếu.

---
## [28] POST /v1/decisions:price-preview — thử giá (2 lần: hợp lệ & dưới vốn)
### ① ĐO TRƯỚC — 3 nguyên liệu của phép tính
```bash
q miniai_decision "SELECT current_price FROM price_state WHERE project_id='demoshop' AND product_id='$SKU';"
q miniai_decision "SELECT round(ewma_cost) AS von FROM cost_state WHERE project_id='demoshop' AND product_id='$SKU';"
q miniai_decision "SELECT round(eps::numeric,4) AS eps, n_points, round(r2::numeric,3) AS r2, method FROM elasticity WHERE project_id='demoshop' AND product_id='$SKU';"
```
**Đo thật 12/08:** `giá 112.000` · `vốn 70.145` · `eps −0.4641 · n=132 · r²=0.417 · ols_daily`

### ② GỌI API — 28a: thử giảm 112.000 → 99.000
```bash
curl -s localhost:16022/v1/decisions:price-preview -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"bh-mi-haohao","candidate_price":99000}' | .venv/bin/python -m json.tool
```
**OUTPUT thật 12/08**
```json
{"current":   {"price": 112000.0, "est_units_30d": 171.0,    "est_profit_30d": 7157260.66},
 "candidate": {"price": 99000,    "est_units_30d": 181.08,   "est_profit_30d": 5225040.83},
 "delta_profit_30d": -1932220,
 "elasticity_used": {"eps": -0.4641, "method": "ols_daily", "n_points": 132, "r2": 0.4172},
 "guardrails": [{"code": "BELOW_COST", "status": "PASS"}],
 "confidence": 0.9,
 "explanation": "Q(P)=Q0·(P/P0)^eps; profit=(P-c)·Q"}
```
### ③ ĐO SAU — **tự bấm máy tính ra đúng con số**
```bash
.venv/bin/python -c "
P0,P1,c,Q0,eps = 112000.0, 99000.0, 70145.0, 171.0, -0.4641
Q1 = Q0*(P1/P0)**eps
print(f'  est_units_30d moi = {Q1:.2f}      (API noi 181.08)')
print(f'  lai hien tai      = {(P0-c)*Q0:,.0f}   (API noi 7.157.261)')
print(f'  lai neu giam gia  = {(P1-c)*Q1:,.0f}   (API noi 5.225.041)')
print(f'  delta             = {(P1-c)*Q1-(P0-c)*Q0:,.0f}  (API noi -1.932.220)')"
```
⭐ **Khoảnh khắc mạnh nhất của cả buổi:** công thức `Q(P)=Q0·(P/P0)^eps` in ngay trong kết quả API, và khách
**tự bấm ra đúng từng con số**.

### ④ SO SÁNH TRỰC TIẾP VỚI DEMO-1 — điểm chốt
| | Hàng mới (DEMO-1) | Hàng ~132 ngày (file này) |
|---|---|---|
| `method` | `pooled_prior` — **mượn** của shop | **`ols_daily`** — hồi quy riêng SKU này |
| `n_points` | 21 | **132** |
| `r2` | `null` | **0.417** |
| `confidence` | 0.7 | **0.9** |

**Kết luận kinh doanh:** hạ giá 11,6% → bán thêm 5,9% → **lãi tháng GIẢM 1,93 triệu** ⇒ máy **can đừng làm**.

### ② GỌI API — 28b: thử giá **DƯỚI VỐN**
```bash
curl -s localhost:16022/v1/decisions:price-preview -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"bh-mi-haohao","candidate_price":50000}' | .venv/bin/python -c "import json,sys; d=json.load(sys.stdin); print('guardrails:', d['guardrails']); print('delta_profit_30d:', format(d['delta_profit_30d'],',.0f'))"
```
**OUTPUT thật:** `[{'code': 'BELOW_COST', 'status': 'FAIL'}]` · lãi tháng **âm hàng chục triệu**
**Nói với khách:** *"Giá 50.000 dưới vốn 70.145 — anh chị vừa tự truy vấn con số vốn đó ở bước ①."*

---
## [29] GET /v1/decisions:replenish-plan — kế hoạch nhập hàng
### ① ĐO TRƯỚC — nguyên liệu
```bash
q miniai_decision "SELECT round(avg(units),3) AS ban_tb, round(stddev_samp(units),3) AS dao_dong FROM sales_daily WHERE project_id='demoshop' AND product_id='$SKU' AND day >= CURRENT_DATE-30;"
q miniai_decision "SELECT on_hand_qty FROM stock_state WHERE project_id='demoshop' AND product_id='$SKU';"
q miniai_decision "SELECT * FROM supplier_config WHERE project_id='demoshop' LIMIT 1;"
```
> ⚠ **Bước này PHỤ THUỘC bước [23].** Nếu vừa chạy [23] (nạp tồn 120) thì `on_hand` ở đây là **120**, không
> phải con số cũ — và `days_of_inventory` nhảy theo. Đo thật 12/08 sau [23]: `avg 5.433 · sigma 3.35 ·
> ss 7.43 · ROP 23.73 · on_hand 120 · DOI 22.1 · moq 10`. Đừng đọc thuộc số, hãy đọc từ màn hình.
### ② GỌI API
```bash
curl -s "localhost:16022/v1/decisions:replenish-plan?product_id=bh-mi-haohao" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | .venv/bin/python -m json.tool
```
### ③ ĐO SAU — **tự kiểm công thức**
```bash
.venv/bin/python -c "
import math
avg, sig, LT, sLT, z, on_hand = 5.9, 3.898, 3.0, 0.0, 1.28, 50.0   # thay bằng số API vừa trả
ss  = z*math.sqrt(LT*sig**2 + avg**2*sLT**2); rop = avg*LT + ss
print(f'  safety_stock  = {ss:.2f}'); print(f'  reorder_point = {rop:.2f}'); print(f'  days_of_inv   = {on_hand/avg:.1f}')"
```
**Đã kiểm chứng 12/08:** công thức tay ra **đúng** con số API (sai lệch chỉ do làm tròn).
### ④ Dịch sang lời chủ shop
Bán **~5,9 thùng/ngày**, hàng về sau **3 ngày**, muốn 90% không cháy hàng ⇒ trữ thêm **~8,6 thùng an toàn**
⇒ **đặt lại khi còn ~26 thùng**. `moq` = nhà cung cấp yêu cầu đặt tối thiểu (khai qua `PUT /v1/config:supplier`).

---
## [30] POST /v1/decisions/{id}:feedback — khép vòng
### ① ĐO TRƯỚC
```bash
q miniai_decision "SELECT count(*) AS so_phan_hoi FROM feedback WHERE project_id='demoshop';"
```
### ② GỌI API
```bash
DID=$(curl -s "localhost:16022/v1/decisions?page_size=1" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | .venv/bin/python -c "import json,sys; print(json.load(sys.stdin)['items'][0]['decision_id'])")
curl -s -X POST "localhost:16022/v1/decisions/$DID:feedback" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"action":"accepted","note":"demo doi tac"}' | .venv/bin/python -c "import json,sys; d=json.load(sys.stdin); print(d['decision_id'],'->',d.get('status'))"
```
### ③ ĐO SAU
```bash
q miniai_decision "SELECT count(*) AS so_phan_hoi FROM feedback WHERE project_id='demoshop';"
q miniai_decision "SELECT decision_id, action, outcome_note, ts FROM feedback WHERE project_id='demoshop' ORDER BY ts DESC LIMIT 1;"
q miniai_decision "SELECT status FROM decisions WHERE decision_id='$DID';"
```
**Đo thật:** `feedback +1`, dòng mới đúng `decision_id`, `decisions.status` đã đổi.
### ④ LUỒNG — vòng khép kín
```
decisions ──► feedback ──► accepted_rate ([26])
                      └──sau 30 ngày──► outcome_ledger: LÃI THỰC TẾ vs expected_value đã hứa
```
```bash
q miniai_decision "SELECT count(*) AS so_dong_da_cham FROM outcome_ledger WHERE project_id='demoshop';"
```
**Câu chốt:** *"Đây là điều phân biệt một hệ thống AI nghiêm túc với một cái máy đoán: **nó chịu trách nhiệm
với lời khuyên của mình bằng số**."*

---
# BẢNG SO SÁNH 2 FILE DEMO (slide chốt buổi)

| Năng lực | DEMO-1: hàng mới 0 dữ liệu | DEMO-2: hàng ~132 ngày |
|---|---|---|
| Tìm kiếm | ✅ sau ~10 giây (**nhìn thấy hàng đợi 0→1→0**) | ✅ `rrf_fusion` |
| Gợi ý gõ phím | `weight` **1.0** | **376.96** |
| Gợi ý liên quan | nấc nội dung, thang **0-1** | **mua kèm thật**, thang **trăm** (148.4) |
| Dự báo | `cold_start_analog`, **khai mượn của 5 SKU**, `data_window=null` | `lgbm_global`, `data_window` 134 ngày |
| Tự chấm điểm | — | ✅ MASE **0,78-0,90**, coverage 0,84-0,94 |
| Khuyến mại | — | ✅ `k` học từ 48 SKU/719 ngày sale của chính shop |
| Kịch bản nhập hàng | — | ✅ 128 kịch bản, RNG có hạt giống + SHA-256 |
| Độ co giãn giá | `pooled_prior`, 21 điểm, tin **0.7** | **`ols_daily`, 132 điểm, r²=0.417, tin 0.9** |
| Chặn bán dưới vốn | ✅ FAIL | ✅ FAIL |
| Vòng phản hồi | ✅ | ✅ + `accepted_rate` theo từng loại |

**Câu chốt cả buổi:** *"Hai sản phẩm, cùng một bộ API. Khác biệt duy nhất là lượng dữ liệu — và hệ thống
**tự khai mình đang ở đâu** trên thang đó: mượn hay tự tính, tin 70% hay 90%. Và mọi con số anh chị vừa nghe,
anh chị đều đã tự truy vấn thẳng vào cơ sở dữ liệu để đối chiếu."*

---
# PHỤ LỤC A — LỖI THƯỜNG GẶP KHI GÕ TAY (đã đo, đừng vấp trước mặt khách)

| API | Sai | Đúng |
|---|---|---|
| lệnh python | `python3 -m json.tool` | **`.venv/bin/python -m json.tool`** (python3 hệ thống thiếu thư viện) |
| `/v1/recommend` context=cart | thiếu `user_pseudo_id` → 400 | thêm `"user_pseudo_id"` |
| `/v1/forecast:insights` | `"kinds": [...]` → 400 | **`"kind"` số ít** |
| `/v1/decisions:insights` | `?kinds=` → 400 | **`?kind=`** số ít |
| `/v1/forecast:promo-preview` | thiếu `start`/`end` → 400 | thêm 2 ngày ISO |
| `/v1/scenarios:*` | ~~thiếu `run_id` → 400~~ **SAI** — đo 12/08 trả **200**, tự lấy mẻ mới nhất | truyền `run_id` khi muốn chỉ đích danh một mẻ |
| `/v1/scenarios:probability` | `threshold` → 400 | **`threshold_units`** |
| `/v1/scenarios:lead-time-demand` | `product_id` → 400 | **`product_ids`** (mảng) |
| `purchase.completed` | `order_id`/`quantity`/`price` → từ chối | **`order_ref`** + **`qty`**, **`unit_price`** |
| `stock.level` | `on_hand` → từ chối | **`on_hand_qty`** |
| `cost.recorded` | thiếu `qty` → từ chối | `{product_id, unit_cost, qty}` |
| `/v1/decisions?product_id=` | ~~bị bỏ qua âm thầm~~ **ĐÃ VÁ 12/08** | `?product_id=` lọc thật (bí danh của `subject_id`) |
| `:feedback` body | ~~`note` bị nuốt~~ **ĐÃ VÁ 12/08** | `note` hoặc `outcome_note` đều được lưu |
| DEMO-1 sau `reset1` | gọi `[06]/[07]` ngay → similar rỗng → **404** | **kích job embedding** rồi mới đi tiếp (cổng thứ hai ở DEMO-1) |
| `make check-apis PROJECT=` | `PROJECT=forecast` (tên service) | **`PROJECT=demoshop`** (mã shop) |

**Lưu ý:** mọi thông báo lỗi đều nói **đích danh trường sai** — lỡ gõ sai, cứ **đọc to thông báo lỗi**,
đó cũng là điểm cộng về chất lượng API.

---
# PHỤ LỤC B — ĐÃ ĐỔI SO VỚI BẢN 07/08 (đo lại 12/08)

| Chỗ | Bản 07/08 | Thực tế 12/08 |
|---|---|---|
| [03] suggest `weight` | 334.80 | **376.96** |
| [04] recommend pdp | 145.3 / 106.7 | **148.4 / 119.8** |
| [13] forecast:query | không nêu model | **`lgbm_global`**, calib `width 1.20 / cov 0.714` |
| [15] accuracy | mase 0.784-0.896 | **0.782-0.897**, autoets cov 0.888 → **0.867** |
| [28] giá hiện tại | 111.000 | **112.000** |
| [28] `n_points` / `r2` | 119 / 0.447 | **132 / 0.417** |
| [28] `delta_profit_30d` | −1.849.284 | **−1.932.220** |
| [29] vốn | 69.455 | **70.145** |
| Số API | 28 | **30** (đánh lại số cho khớp mục lục) |

> ⚠ **Số của [24] `decisions:run` và [26] `stats` đổi mỗi ngày** — đừng đọc thuộc lòng, hãy đọc từ màn hình.
> Cấu trúc và ý nghĩa từng trường thì không đổi.
