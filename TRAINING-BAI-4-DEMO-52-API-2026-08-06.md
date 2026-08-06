# BÀI 4 — SỔ TAY DEMO 52 API: COPY → CHẠY → ĐỌC KẾT QUẢ

> Giáo trình training human (2026-08-06). Khác Bài 0-3 (lý thuyết + hành trình), Bài 4 là **SỔ TAY TRA CỨU**:
> đủ 52 endpoint, mỗi cái MỘT LỆNH 1-DÒNG copy-là-chạy + kỳ vọng + cách đọc 1 câu. Payload gặt từ probe
> check-apis (`seedtool/checker.py`) + code thật — không lệnh nào bịa.
> ⚠ Mọi lệnh cố ý viết 1 DÒNG (tránh bug dán-thiếu-`\` đã dính ở Bài 3). Lệnh có nhãn **[GHI]** = thay đổi
> dữ liệu (còn lại chỉ đọc, bắn thoải mái).

## CHUẨN BỊ (chạy 1 lần)
```bash
cd /home/hai-soft/projects/icpp/mecom/project
SKEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['search'])")
DKEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['decision'])")
FKEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['forecast'])")
EVT=$(date -u +%Y-%m-%dT%H:%M:%SZ)
echo "keys: ${SKEY:0:6}/${DKEY:0:6}/${FKEY:0:6} | EVT=$EVT"
```
SKU mẫu dùng xuyên bài: `bh-mi-haohao` (128 ngày lịch sử sau SEED-120D 2026-08-06 — đủ điều kiện mọi API
forecast, kể cả nấc LightGBM global) · sản phẩm tập: `hoc-sp-52` (tạo ở S5, xóa ở S17). Port: search 16021 ·
decision 16022 · forecast 16023.
> 🆕 **BẢN VÁ 2026-08-06 TỐI (chiến dịch T-PREP-DEMO-0807)** — 6 hành vi ĐÃ ĐỔI so với buổi sáng, mục nào
> có nhãn 🆕 bên dưới thì kỳ vọng MỚI là chuẩn: F8 (:run trả 202+job_id, poll status) · D15 (BELOW_COST
> giờ FAIL khi giá dưới vốn) · S16 (ngành hết phân biệt dấu) · S9 (answer tự lọc item lệch ngành) ·
> S8 (PDP sản phẩm mới gợi ý cùng ngành thay vì popular) · preflight (`python -m seedtool check` tự từ
> chối đo khi máy nghẹt, exit 3).

═══════════════════════════════════════════════
# SERVICE 01 — SMARTSEARCH :16021 (17 endpoint)
═══════════════════════════════════════════════

### S1. GET /healthz — process còn thở?
```bash
curl -s localhost:16021/healthz
```
→ `{"status":"ok"}`. Docker dùng mạch này quyết restart. Không cần auth.

### S2. GET /readyz — sẵn sàng phục vụ?
```bash
curl -s localhost:16021/readyz
```
→ `{"status":"ok"}` = đã nối DB/Vespa. Load-balancer tin mạch này, KHÔNG tin healthz.

### S3. GET /metrics — số đo Prometheus
```bash
curl -s localhost:16021/metrics | head -15
```
→ Text định dạng Prometheus (`miniai_smartsearch_...`). Grafana :16020 vẽ từ đây.

### S4. GET /v1/ping — thử auth nhẹ nhất
```bash
curl -s localhost:16021/v1/ping -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop"
```
→ pong/ok. Lệnh chẩn đoán "key còn sống không" rẻ nhất.

### S5. POST /v1/products:upsert [GHI] — khai sinh/cập nhật sản phẩm
```bash
curl -s localhost:16021/v1/products:upsert -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"products":[{"id":"hoc-sp-52","title":"Loa bluetooth MECOM demo 52","description":"Loa demo bai 4, pin 12h","categories":["Dien tu > Am thanh"],"brands":["MECOM"],"price_info":{"currency_code":"VND","price":390000},"availability":"IN_STOCK","available_quantity":10,"attributes":{},"images":[],"publish_time":"2026-08-06T09:00:00Z"}]}' | python3 -m json.tool
```
→ `{"upserted": 1, "queued_for_index": 1}` — 2 THÌ: đã-ghi-sổ PG + đã-xếp-việc-đẩy-Vespa (outbox pattern).

### S6. POST /v1/search — tìm kiếm hybrid
```bash
curl -s localhost:16021/v1/search -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"query":"loa bluetooth"}' | python3 -c "import json,sys; [print(i['product_id'],'|',i['title']) for i in json.load(sys.stdin).get('items',[])[:5]]"
```
→ Danh sách xếp hạng (BM25 chữ + vector nghĩa + hành vi, trộn RRF). `hoc-sp-52` xuất hiện sau vài giây
(eventual consistency — chưa thấy thì soi `catalog_outbox` count).

### S7. GET /v1/suggest — autocomplete
```bash
curl -s "localhost:16021/v1/suggest?q=loa" -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
```
→ `items[{text, weight}]` học từ query người dùng thật + n-gram title, kèm khối `consistency` (độ tươi data).

### S8. POST /v1/recommend — gợi ý 4 ngữ cảnh
```bash
curl -s localhost:16021/v1/recommend -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"context":"pdp","product_id":"dt-tainghe-baseus"}' | python3 -c "import json,sys; [print(i['product_id'],'|',i['title']) for i in json.load(sys.stdin).get('items',[])[:5]]"
```
→ Món liên quan trang sản phẩm. Đổi `context`: `home`+`user_pseudo_id` · `similar` · `cart`.
🆕 SKU mới toanh (0 hành vi) giờ đi bậc thang cold-start: cùng-ngành theo nội dung → popular cùng-ngành →
popular toàn shop (chỉ còn là lưới cuối). Thử lại ca đo sáng nay: upsert 1 SKU tai nghe mới rồi gọi pdp —
kỳ vọng ≥3/5 item cùng ngành Điện tử (sáng nay: tất/kem chống nắng/sữa bột — đã hết). Nợ W-RECO-PDP-COLDSTART ĐÓNG.

### S9. POST /v1/ask — hỏi tự nhiên có guard chống bịa
```bash
curl -s localhost:16021/v1/ask -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"question":"co loa nao duoi 500 nghin khong?"}' | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['answer']); print('--- llm:', d['llm_used'], '| guard blocked:', d['grounding_guard']['blocked'])"
```
→ Câu trả lời grounded catalog. 4 chặng: parse luật (giá<500k) → RRF → template/LLM → guard B−A.
🆕 Thêm trường `answer_coherence`: câu hỏi nêu ngành rõ ("tai nghe chống ồn") → item lệch ngành bị LOẠI
khỏi answer (đo tối nay: tự loại chảo + ốp lưng, `dropped_ids` khai đủ); answer được phép <3 dòng; câu
mơ hồ/đa-ngành → không lọc (giữ hành vi cũ). `items` retrieval vẫn trả đủ — chỉ ANSWER bị siết.

### S10. POST /v1/events:ingest [GHI] — bơm hành vi xem
```bash
curl -s localhost:16021/v1/events:ingest -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"events":[{"event_id":"b4-view-'$RANDOM'","schema_version":"1.0","event_type":"product.viewed","event_time":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","user_pseudo_id":"b4-user","session_id":null,"attribution_token":null,"payload":{"product_id":"dt-tainghe-baseus"}}]}' | python3 -m json.tool
```
→ `accepted: 1` + `ledger_position` (sổ cái đánh số tuần tự). Gửi lại CÙNG event_id → `deduped: 1`.

### S11. GET /v1/events:dead — nhà xác event (DLQ)
```bash
curl -s "localhost:16021/v1/events:dead?limit=5" -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool | head -20
```
→ Event payload-hỏng nằm đây kèm lý do — chẩn đoán tích hợp của khách ("event tôi gửi đi đâu mất?").

### S12. POST /v1/merch:rules [GHI] — ghim/boost/chặn thủ công
```bash
curl -s localhost:16021/v1/merch:rules -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"rule_id":"b4-rule-pin","rule_type":"pin","match_query_norm":"loa bluetooth","target_product_ids":["hoc-sp-52"]}' | python3 -m json.tool
```
→ 200. Từ giờ query "loa bluetooth" sẽ GHIM hoc-sp-52 lên #1 bất chấp điểm — quyền lực người bán đè thuật toán.
(Chạy lại S6 để chứng kiến!)

### S13. GET /v1/merch:rules — xem rule đang chạy
```bash
curl -s localhost:16021/v1/merch:rules -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
```
→ `rules[]` gồm `b4-rule-pin` vừa tạo.

### S14. DELETE /v1/merch:rules/{id} [GHI] — gỡ rule
```bash
curl -s -o /dev/null -w "status: %{http_code}\n" -X DELETE localhost:16021/v1/merch:rules/b4-rule-pin -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop"
```
→ `204`. Search trở về xếp hạng tự nhiên.

### S15. GET /internal/similar-products — nội bộ: ai giống ai (embedding)
```bash
curl -s "localhost:16021/internal/similar-products?project_id=demoshop&product_id=dt-tainghe-baseus&k=5" -H "X-Internal-Token: dev-internal-token" | python3 -m json.tool | head -20
```
→ 5 hàng xóm gần nhất theo vector. KHÔNG dùng Bearer — dùng `X-Internal-Token` (chỉ service nội bộ gọi nhau;
forecast dùng chính API này cho cold-start analog).

### S16. GET /internal/products-by-category — nội bộ: SKU theo ngành
```bash
curl -s "localhost:16021/internal/products-by-category?project_id=demoshop&category_l1=Dien%20tu" -H "X-Internal-Token: dev-internal-token" | python3 -m json.tool | head -15
```
→ Danh sách SKU ngành — forecast:aggregate theo category dùng đường này.
🆕 Hết phân biệt dấu/hoa-thường: `Dien tu` = `Điện tử` = `dIEN Tu` trả CÙNG tập (đo tối nay: 17=17=17 SKU,
sáng nay "Dien tu" chỉ ra đúng hoc-sp-52 tự tập). Ngành-ma do khai không dấu đã hết đường sinh — W-CAT-L1-DIACRITICS ĐÓNG.

### S17. DELETE /v1/products/{id} [GHI] — xóa sản phẩm (dọn sân, chạy CUỐI bài)
```bash
curl -s -o /dev/null -w "lan 1: %{http_code}\n" -X DELETE localhost:16021/v1/products/hoc-sp-52 -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop"; curl -s -o /dev/null -w "lan 2: %{http_code}\n" -X DELETE localhost:16021/v1/products/hoc-sp-52 -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop"
```
→ `204` rồi `404` — hợp đồng xóa tử tế (không giả vờ xóa được thứ không còn).

═══════════════════════════════════════════════
# SERVICE 02 — DECISION :16022 (17 endpoint)
═══════════════════════════════════════════════

### D1-D4. healthz · readyz · metrics · ping (bộ tứ hạ tầng — như S1-S4, đổi port + $DKEY)
```bash
curl -s localhost:16022/healthz && echo && curl -s localhost:16022/readyz && echo && curl -s localhost:16022/v1/ping -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" && echo && curl -s localhost:16022/metrics | head -5
```

### D5. GET /v1/config — đọc chính sách tenant
```bash
curl -s localhost:16022/v1/config -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
```
→ CHỈ trả ghi-đè đã PUT (`promo_cap_pct: 50, pricing_mode: lerner`); rỗng = chạy mặc định trong code.

### D6. PUT /v1/config [GHI] — đổi chính sách sống (nhớ TRẢ LẠI sau khi demo!)
```bash
curl -s -X PUT localhost:16022/v1/config -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"pricing_mode":"robust"}' | python3 -m json.tool && curl -s -X PUT localhost:16022/v1/config -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"pricing_mode":"lerner"}' | python3 -m json.tool
```
→ `{"updated": ["pricing_mode"]}` ×2 (đổi robust rồi trả lerner ngay trong 1 lệnh — demo an toàn).
lerner = tối ưu kỳ vọng · robust = CVaR thận trọng. Per-tenant, không restart.

### D7. GET /v1/config:supplier — cấu hình nhà cung cấp
```bash
curl -s "localhost:16022/v1/config:supplier" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool | head -20
```
→ lead-time, MOQ theo supplier — nguyên liệu cho replenish-plan.

### D8. PUT /v1/config:supplier [GHI] — khai điều kiện cung ứng THEO SẢN PHẨM
```bash
curl -s -X PUT "localhost:16022/v1/config:supplier" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"bh-mi-haohao","lead_time_days":3,"moq":10}' | python3 -m json.tool
```
→ 200. ⚠ Khóa là **`product_id`** (không phải supplier_ref!) — nhìn D7: mỗi dòng config = 1 sản phẩm với
lead_time_days/lead_time_std/moq/pack_size. Bản đầu sổ tay viết sai `supplier_ref` → API dẫn đường bằng 400
"'product_id' must be a non-empty string" (đã sửa 2026-08-06). Ý nghĩa: "món này đặt nhà cung cấp thì 3 ngày
mới về, mỗi lần đặt tối thiểu 10" — nguyên liệu trực tiếp cho replenish-plan (D16).

### D9. POST /v1/events:ingest [GHI] — nạp tồn kho (+ họ hàng: cost.recorded, price.changed)
```bash
curl -s localhost:16022/v1/events:ingest -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"events":[{"event_id":"b4-stock-'$RANDOM'","schema_version":"1.0","event_type":"stock.level","event_time":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","user_pseudo_id":"b4-user","session_id":null,"attribution_token":null,"payload":{"product_id":"bh-mi-haohao","on_hand_qty":50}}]}' | python3 -m json.tool
```
→ `accepted: 1`. Onboard đủ BT02 cho 1 SKU = 3 event: purchase.completed (→forecast) + `cost.recorded`
{unit_cost, qty} + `price.changed` {new_price} (→decision, worker cuộn nhịp 300s vào cost_state/price_state).

### D10. GET /v1/events:dead — DLQ của decision
```bash
curl -s "localhost:16022/v1/events:dead?limit=5" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool | head -15
```

### D11. POST /v1/decisions:run [GHI] — kích bộ não quyết định
```bash
curl -s -X POST localhost:16022/v1/decisions:run -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{}' | python3 -m json.tool
```
→ `{"created": N}` + BẢN KHAI 4 cửa kỷ luật: `skipped_dedup` (chống lặp) · `anti_oscillation` (cấm lật kèo giá)
· `plan_conflict` (không đá kế hoạch) · `no_cost/no_stock` (thiếu data = im). created:0 + bản khai = kết quả TỐT.
⚠ Endpoint nặng — lần đầu sau restart có thể chậm.

### D12. GET /v1/decisions — hàng đợi lời khuyên
```bash
curl -s "localhost:16022/v1/decisions?page_size=5" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | python3 -c "
import json,sys
for x in json.load(sys.stdin).get('items', [])[:5]:
    ev = (x.get('expected_value') or {}); print(x['decision_id'],'|',x['kind'],'|',x['status'],'|',f\"EV {ev.get('amount',0):,.0f}d/{ev.get('per','')}\")"
```
→ id | kind | status (open=chờ phán, accepted=đã nhận) | **EV = cột xếp việc** (làm con to trước). Mở nguyên
con bằng `page_size=1 | python3 -m json.tool` để đọc `trace` — chứng minh có số audit được từng phép nhân.

### D13. GET /v1/decisions:stats — thống kê
```bash
curl -s "localhost:16022/v1/decisions:stats?window=30d" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
```
→ Đếm theo status/kind + accepted_rate — "chủ shop có nghe lời AI không?".

### D14. GET /v1/decisions:insights — insight tổng hợp
```bash
curl -s "localhost:16022/v1/decisions:insights?kind=capital_locked" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool | head -25
```
→ Vốn đang bị khóa trong hàng ế. `kind` ∈ {capital_locked, advice_scorecard, monthly_benefit, removal_candidates}.

### D15. POST /v1/decisions:price-preview — thử giá trước khi áp
```bash
curl -s -w "\nstatus: %{http_code}\n" localhost:16022/v1/decisions:price-preview -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"bh-mi-haohao","candidate_price":9000}'
```
→ `200`: current vs candidate (units/profit 30d) + `delta_profit_30d` (âm = máy CAN đừng làm) + elasticity_used
khai nguồn (n_points=0 → mượn pooled_prior). HOẶC `412` nêu đích danh cổng thiếu (sales/cost/price) — lỗi dẫn đường.
🆕 Với chính lệnh trên (mì 9.000đ vs vốn ~70.458đ): `guardrails` giờ trả `{"code":"BELOW_COST","status":"FAIL"}`
— sáng nay 2 nhánh đều PASS oan (bug D15 bạn tìm ra), đã fix + test hồi quy ghim đúng ca này. Giá trên vốn → PASS.

### D16. GET /v1/decisions:replenish-plan — kế hoạch nhập hàng
```bash
curl -s "localhost:16022/v1/decisions:replenish-plan" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool | head -30
```
→ Toàn bộ SKU (cap 100; 1 SKU: thêm `?product_id=...`) — điểm đặt hàng + số lượng, ăn P90/lead-time-demand
từ forecast qua scenario fabric.

### D17. POST /v1/decisions/{id}:feedback [GHI] — chủ shop phán (vai C)
```bash
DID=$(curl -s "localhost:16022/v1/decisions?page_size=10" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | python3 -c "import json,sys; L=[x['decision_id'] for x in json.load(sys.stdin).get('items',[]) if x.get('status')=='open']; print(L[0] if L else '')") && echo "decision: $DID" && curl -s -o /dev/null -w "status: %{http_code}\n" "localhost:16022/v1/decisions/$DID:feedback" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"action":"accepted"}'
```
→ Tự nhặt con `open` đầu tiên rồi accept (action ∈ accepted/dismissed). Từ đây outcome-loop 30 ngày chấm
lời khuyên bằng tiền thật.

═══════════════════════════════════════════════
# SERVICE 03 — FORECAST :16023 (18 endpoint)
═══════════════════════════════════════════════

### F1-F4. healthz · readyz · metrics · ping
```bash
curl -s localhost:16023/healthz && echo && curl -s localhost:16023/readyz && echo && curl -s localhost:16023/v1/ping -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" && echo && curl -s localhost:16023/metrics | head -5
```

### F5. POST /v1/events:ingest [GHI] — nạp đơn hàng (nuôi chuỗi thời gian)
```bash
curl -s localhost:16023/v1/events:ingest -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"events":[{"event_id":"b4-buy-'$RANDOM'","schema_version":"1.0","event_type":"purchase.completed","event_time":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","user_pseudo_id":"b4-user","session_id":null,"attribution_token":null,"payload":{"order_ref":"b4-order-'$RANDOM'","items":[{"product_id":"bh-mi-haohao","qty":1,"unit_price":9500}]}}]}' | python3 -m json.tool
```
→ `accepted: 1` + ledger_position. Sửa sai KHÔNG xóa — bút toán đảo `order.returned` {order_ref, items, reason}.

### F6. POST /v1/events:backfill [GHI] — đổ lịch sử cũ (onboard khách)
```bash
curl -s localhost:16023/v1/events:backfill -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"events":[{"event_id":"b4-backfill-001","schema_version":"1.0","event_type":"purchase.completed","event_time":"2026-04-01T10:00:00Z","user_pseudo_id":"b4-user","session_id":null,"attribution_token":null,"payload":{"order_ref":"b4-old-order","items":[{"product_id":"bh-mi-haohao","qty":2,"unit_price":9000}]}}]}' | python3 -m json.tool
```
→ Cùng phong bì ingest nhưng SÀN -90 ngày được dỡ (event 04/2026 vẫn nhận) — cửa riêng cho "đổ sử ngày-1".

### F7. GET /v1/events:dead — DLQ forecast
```bash
curl -s "localhost:16023/v1/events:dead?limit=5" -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool | head -15
```

### F8. POST /v1/forecast:run [GHI] — train + sinh projections
```bash
curl -s -w "\nstatus: %{http_code}\n" -X POST localhost:16023/v1/forecast:run -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{}'
```
🆕 → `202` NGAY LẬP TỨC: `{"status":"queued","run_id":"r_...","job_id":"fr-demoshop-r_..."}` — đo "nhận việc",
worker làm nặng phía sau (nợ W-RUN-ASYNC-202 ĐÓNG, hết cảnh 4 lần timeout như Bài 1 sáng nay). Gọi lại khi
job đang chạy = trả CÙNG job (idempotent, không nhân đôi việc). Theo dõi tới xong:
```bash
curl -s "localhost:16023/v1/projections/status?job_id=fr-demoshop-r_$(date -u +%F)" -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
```
→ `job.status`: `queued` → `running` → `done` (hoặc `failed` + `error_code` — lỗi nhìn thấy được, không nuốt).
Máy nguội train từ 0 có thể vài phút — F9/F10/F11 chỉ đo SAU khi `done`. Router phân loại từng SKU → thang model
(cold_start_analog → similar_item_transfer → croston/SBA → seasonal_naive → ETS+Theta → LightGBM ≥120d);
sau SEED-120D kỳ vọng `model_used="lgbm_global"` xuất hiện ở ~66 SKU bán đều (soi qua F9).

### F9. POST /v1/forecast:query — hỏi dải dự báo
```bash
curl -s localhost:16023/v1/forecast:query -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"bh-mi-haohao","horizon_days":7}' | python3 -m json.tool | head -30
```
→ p10/p50/p90 theo ngày (P50 kế hoạch · P90 nhập hàng · P10 dòng tiền) + `model_used` TRUNG THỰC +
`data_window` + `totals` (Monte Carlo 2000 — quantile không cộng được) + `calibration` (backtest tự chấm).

### F10. POST /v1/forecast:aggregate — dự báo gộp nhóm
```bash
curl -s localhost:16023/v1/forecast:aggregate -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"category_l1":"Bách hóa","horizon_days":7}' | python3 -m json.tool | head -20
```
→ Tổng dải theo ngành (hoặc `{"product_ids":[...]}`). Hier-reconcile ép tổng SKU khớp tổng ngành.

### F11. GET /v1/forecast:accuracy — hệ tự chấm điểm
```bash
curl -s "localhost:16023/v1/forecast:accuracy?window=90d" -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool | head -20
```
→ MASE (so seasonal-naive; <1 = thắng baseline) + coverage_p10_p90 (chuẩn 0.8) — số để trả lời "forecast
anh CÓ GIỎI không?" bằng đo lường, không bằng lời.

### F12. POST /v1/forecast:insights — insight nhu cầu
```bash
curl -s localhost:16023/v1/forecast:insights -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"kind":"top_movers"}' | python3 -m json.tool | head -25
```
→ SKU tăng/giảm mạnh nhất. `kind` ∈ {accuracy_sku, top_movers, group_breakdown, seasonality} — sai kind thì
400 liệt kê đủ lựa chọn (validation dẫn đường).

### F13. POST /v1/forecast:promo-preview — "giảm giá X% thì bán thêm bao nhiêu?"
```bash
curl -s localhost:16023/v1/forecast:promo-preview -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"bh-mi-haohao","discount_pct":30,"start":"2026-08-10","end":"2026-08-15"}' | python3 -m json.tool | head -20
```
→ Uplift dự kiến của chương trình giảm 30% (persisted:false — chỉ xem thử, không ghi). Từng là ca bug 56s
→ 0.03s nhờ cache model (W-PROMO-PREVIEW-CACHE).

### F14. GET /v1/projections/status — trạng thái projections
```bash
curl -s "localhost:16023/v1/projections/status" -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
```
→ Projection đã cuộn tới ledger_position nào — "số tôi đang xem tươi đến đâu". (Op đang tracked
W-SURFACE-FC-PROJSTATUS về chủ business.)

### F15. POST /v1/scenarios:build [GHI] — dựng kịch bản xác suất (scenario fabric, ADR-009)
```bash
curl -s localhost:16023/v1/scenarios:build -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_ids":["bh-mi-haohao"],"horizon_days":7,"scenario_count":500}' | python3 -m json.tool | head -12
```
→ `run_id` + manifest — 500 "tương lai giả lập" cho SKU. Decision gọi nội bộ đường này (tier-0).

### F16. POST /v1/scenarios:lead-time-demand — cầu trong thời gian chờ hàng
```bash
curl -s localhost:16023/v1/scenarios:lead-time-demand -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_ids":["bh-mi-haohao"],"lead_time_days":3,"review_period_days":2,"required_quantiles":[0.5,0.9],"horizon_days":7}' | python3 -m json.tool
```
→ q50/q90 của tổng cầu trong (lead-time + review) — CON SỐ trực tiếp quyết định "đặt bao nhiêu" ở
replenish-plan (D16). Cầu nối forecast→decision.

### F17. POST /v1/scenarios:aggregate — gộp kịch bản
```bash
curl -s localhost:16023/v1/scenarios:aggregate -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_ids":["bh-mi-haohao"],"horizon_days":7}' | python3 -m json.tool
```
→ totals p10/p50/p90 từ tập kịch bản (thay vì công thức) — nhất quán với mọi phép hỏi khác trên cùng fabric.

### F18. POST /v1/scenarios:probability — xác suất theo ngưỡng
```bash
curl -s localhost:16023/v1/scenarios:probability -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_ids":["bh-mi-haohao"],"product_id":"bh-mi-haohao","threshold_units":10,"horizon_days":7}' | python3 -m json.tool
```
→ `P(tổng cầu ≥ 10)` — trả lời kiểu "khả năng bán hết 10 thùng trong tuần là bao nhiêu %?" — ngôn ngữ
xác suất mà chủ shop hiểu được.

═══════════════════════════════════════════════
# DỌN SÂN SAU BÀI (bắt buộc nếu đã chạy các lệnh [GHI])
═══════════════════════════════════════════════
1. Gỡ rule ghim: S14 (nếu chưa).  2. Xóa sản phẩm tập: S17 (204/404).
3. Trung hòa đơn tập (nếu muốn số ròng sạch): bút toán đảo `order.returned` cho từng `b4-order-*` đã bơm
   (xem Bài 3 mục 3.10b).  4. Gate chốt: `make check-apis PROJECT=demoshop` → 42/42.

═══════════════════════════════════════════════
# 🆕 MÀN DEMO 2-LINE — SP MỚI vs SP ĐẦY ĐỦ DATA (human đề xuất tối 06/08, số đo thật kèm dưới)
═══════════════════════════════════════════════
> Ý tưởng bán hàng: chạy SONG SONG cùng 4 API trên 2 sản phẩm — khách thấy NGAY hệ trả lời được cả hàng
> mới toanh (thang cold-start) lẫn hàng có 128 ngày lịch sử (trí khôn đầy đủ), và LUÔN KHAI THẬT nó đang
> ở nấc nào. Line A = `dt-tainghe-baseus` (tai nghe, 128d). Line B = tạo sống 1 tai nghe mới trước mặt khách.

**B-0. Tạo SP mới (chạy trước mặt khách):**
```bash
curl -s localhost:16021/v1/products:upsert -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"products":[{"id":"demo-tn-moi","title":"Tai nghe chup tai X-Sound Pro 2026","description":"Tai nghe bluetooth chong on chu dong, pin 40h","categories":["Điện tử > Âm thanh"],"brands":["XSound"],"price_info":{"currency_code":"VND","price":350000},"availability":"IN_STOCK","available_quantity":20,"attributes":{},"images":[],"publish_time":"2026-08-06T18:00:00Z"}]}'
```
Chờ ~6-9s (outbox → Vespa) rồi chạy 4 cặp:

| API | Line B — `demo-tn-moi` (0 hành vi) | Line A — `dt-tainghe-baseus` (128d) | Câu chuyện |
|---|---|---|---|
| S6 search "tai nghe chong on" | CÓ MẶT trong kết quả sau vài giây | top đầu (điểm hành vi tích lũy) | hàng mới lên kệ tức thì, thứ hạng phải NUÔI bằng hành vi |
| S8 recommend pdp | 5/5 CÙNG NGÀNH điện tử (bậc thang cold-start — đo thật: cường lực/iPhone/sạc, `fallback:"popularity"` cùng-ngành) | phụ kiện mua-kèm thật (Sony/bàn phím/SSD — bought_together) | chưa có hành vi thì mượn ngành, có hành vi thì dùng hành vi |
| F9 forecast:query | `404 NOT_FOUND` — chưa có đơn nào, MÁY KHÔNG BỊA SỐ | `200 model_used="lgbm_global"`, coverage 0.857 tự chấm | trung thực > ảo thuật; nạp vài đơn (F5) là leo thang cold_start→analog |
| D15 price-preview 300k/180k | `412 no sales in last 30d` — lỗi DẪN ĐƯỜNG nói thiếu gì | `200` eps −0.57 `ols_daily n=113, conf 0.9` + guardrail | 3 cổng dữ liệu bảo vệ khách khỏi lời khuyên thiếu căn cứ |

**Chốt màn:** mọi response đều tự khai nguồn (`model_used` · `method/n_points` · `fallback`) — "hệ không bao giờ
giả vờ biết". **Dọn sân:** `curl -s -X DELETE localhost:16021/v1/products/demo-tn-moi -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop"` (204).

# ⚠ VÙNG TRÁNH KHI DEMO (điểm yếu ĐO ĐƯỢC tối 06/08 — W-SEARCH-CONCEPT-NEGATION)
Truy vấn KHÁI NIỆM + PHỦ ĐỊNH hiện thua: "đồ uống không cồn" → chuột KHÔNG dây, nồi chiên KHÔNG dầu (máy khớp
chữ "không"); "nước giải khát" → nước GIẶT, nước rửa chén. `/v1/ask` cũng thua ca này. ĐỪNG gõ dạng này khi demo.
Nếu khách tự thử → trả lời chuẩn bị sẵn: "đúng, truy vấn thuộc-tính-ẩn là hạng mục roadmap (concept-mapping +
attribute enrichment — nền intent_lexicon/enrich_attrs có sẵn); hệ này mạnh ở truy vấn danh-từ-ngành + typo +
không-dấu (S6 demo được ngay)". Điểm yếu nói trước mặt luôn ăn điểm hơn bị bắt quả tang.

## GHI NHỚ CHUNG
- 2 header mọi API nghiệp vụ: `Authorization: Bearer <key-đúng-service>` + `X-Project-Id` · nội bộ dùng
  `X-Internal-Token`.
- Đọc = bắn thoải mái; [GHI] = để lại vết → dọn sân + trả nguyên trạng.
- Mọi con số lạ trong response đều tra được: sổ (`demand_daily`, `cost_state`...) hoặc code (hằng có tên).
- Lệnh nghi ngờ nhất luôn là: `make check-apis PROJECT=demoshop` — 42/42 mới tin.
