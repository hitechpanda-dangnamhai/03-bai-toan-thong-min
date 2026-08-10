# BÀI 3 — SỬ DỤNG API TRỌN CHUỖI: ĐÓNG VAI KHÁCH TÍCH HỢP (bản chuẩn v2 — sau buổi thực hành 2026-08-06)

> Giáo trình training human. Bài 0 = bản đồ API · Bài 1 = vận hành · Bài 2 = cấu hình & key.
> Điều kiện vào: stack xanh (Bài 1 B3c 43/43) + thuộc 2 header xác thực (Bài 2).
> Mọi lệnh chạy từ `cd /home/hai-soft/projects/icpp/mecom/project` · tenant **demoshop**.
> Bản v2 này đã hấp thụ TOÀN BỘ kết quả đo thật + bài học phát sinh của buổi thực hành đầu (human tự tay chạy).
> ⚠ Lệnh nhiều dòng có dấu `\` cuối dòng — copy TRỌN KHỐI; dán thiếu `\` là `-H` rơi mất tham số (đã dính 1 lần).

**Mục tiêu:** đi trọn vòng đời nghiệp vụ với MỘT sản phẩm của mình: tạo → tìm thấy → hỏi tư vấn → bơm hành vi
→ xem dự báo tiến hóa → xin lời khuyên → thử giá → phản hồi → dọn sân.

**Sơ đồ hành trình:**
```
3.1 upsert ─► 3.2 search ─► 3.3 suggest/recommend ─► 3.4 ask        (BT01 — vai B)
      │
3.5 events viewed+purchase (vai A/C)  ─►  3.6 forecast:query        (BT03 — vai B)
      │
3.7 decisions:run + đọc lời khuyên ─► 3.8 price-preview (3 cổng 412) ─► 3.9 feedback   (BT02 — B rồi C)
      │
3.10 dọn sân (delete 204/404)
```

---

## 3.0 — NẠP 3 CHÌA VÀO BIẾN (1 lần/buổi, đúng terminal đang dùng)

```bash
cd /home/hai-soft/projects/icpp/mecom/project
SKEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['search'])")
DKEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['decision'])")
FKEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['forecast'])")
echo "SKEY=${SKEY:0:6}... DKEY=${DKEY:0:6}... FKEY=${FKEY:0:6}..."   # kiem bien, khong in key tran
```
- Quy ước: **S**KEY↔search:16021 · **D**KEY↔decision:16022 · **F**KEY↔forecast:16023.
- Biến chết khi đổi tab/terminal → gặp **401 khó hiểu** thì phản xạ đầu tiên: `echo ${SKEY:0:6}` kiểm biến.

## 3.1 — VAI A: KHAI SINH SẢN PHẨM (products:upsert)

```bash
curl -s localhost:16021/v1/products:upsert -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{
  "products": [{
    "id": "hoc-sp-01",
    "title": "Tai nghe MECOM hoc bai chong on",
    "description": "Tai nghe thuc hanh Bai 3 - chong on, pin 30h",
    "categories": ["Dien tu > Am thanh"],
    "brands": ["MECOM"],
    "price_info": {"currency_code": "VND", "price": 250000},
    "availability": "IN_STOCK", "available_quantity": 25,
    "attributes": {}, "images": [], "publish_time": "2026-08-06T09:00:00Z"
  }]
}' | python3 -m json.tool
```
**Kết quả thật:** `{"upserted": 1, "queued_for_index": 1}` — API khai 2 THÌ:
- `upserted` (hoàn thành): đã nằm chắc trong Postgres (sổ cái catalog).
- `queued_for_index` (tương lai gần): 1 việc-cần-đẩy-sang-Vespa đã xếp vào hàng đợi `catalog_outbox`.
**Ghi chú:** `id` = SKU do KHÁCH đặt (khớp hệ quản kho) · body là MẢNG (sync nghìn SKU/lượt) · upsert
idempotent theo `id` · title/description tử tế = SEO nội bộ (nuôi cả BM25 lẫn vector — xem 3.4).

## 3.2 — VAI B: TÌM THẤY CHÍNH NÓ + EVENTUAL CONSISTENCY

**Bản thí nghiệm (học — nhìn hàng đợi VÀ kết quả cùng lúc):**
```bash
docker exec miniai-postgres psql -U miniai -d miniai_search -c "SELECT count(*) FROM catalog_outbox;" \
&& curl -s localhost:16021/v1/search -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{"query":"tai nghe mecom"}' \
  | python3 -c "import json,sys; d=json.load(sys.stdin); items=d.get('items',[]); print(f'{len(items)} ket qua:'); [print(' ', i.get('product_id'), '|', i.get('title')) for i in items[:5]]"
```
- `count=0` → đã đẩy hết → search PHẢI thấy. `count>0` → chờ 3-5s chạy lại — KHÔNG phải lỗi.
- **Outbox pattern:** ghi sản phẩm + ghi việc-cần-đẩy trong CÙNG 1 transaction PG; worker đẩy dần sang Vespa.
  Vespa chết → việc nằm chờ — không mất, chỉ muộn. Đổi lại: search thấy TRỄ vài giây (eventual consistency).
**Bài học xếp hạng (đo thật):** `hoc-sp-01` đứng **#2 thua Sony CH520** dù query có chữ "mecom" — ranking lai
= chữ (BM25) + nghĩa (vector) + **HÀNH VI** (Sony có lịch sử click thật, hàng mới = 0). Muốn lên hạng → vai C
(bơm event), không phải sửa title.

## 3.3 — VAI B: AUTOCOMPLETE + GỢI Ý NGỮ CẢNH

```bash
curl -s "localhost:16021/v1/suggest?q=tai%20ng" -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
curl -s localhost:16021/v1/recommend -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{"context":"pdp","product_id":"hoc-sp-01"}' \
  | python3 -c "import json,sys; [print(' ', i.get('product_id'), '|', i.get('title')) for i in json.load(sys.stdin).get('items',[])[:5]]"
```
**Kết quả thật + bài học:**
- Suggest trả cụm NGƯỜI THẬT gõ kèm `weight` ("tai nghe bluetooth" 275.2...) + khối **`consistency`**
  (`is_caught_up`, `ledger_head`) — API tự khai độ tươi dữ liệu (chính cái ledger từng ốm hôm 06/08).
- Recommend 4 context: `home` (cần user_pseudo_id) · `pdp` (cần product_id) · `similar` · `cart`.
- **PHÁT HIỆN THẬT (đã ghi nợ `W-RECO-PDP-COLDSTART`):** pdp cho SKU mới trả kem chống nắng/tất/sữa bột —
  cold-start rơi thẳng nấc fallback thô nhất (popular toàn shop) thay vì cùng-category. Gate đóng nợ: ≥3/5
  item cùng ngành. **Meta-bài-học: test bằng sản phẩm MÌNH TỰ TẠO nên mình biết đáp án đúng — mới chấm được máy.**

## 3.4 — VAI B: HỎI TỰ NHIÊN (/v1/ask) — cỗ máy 4 CHẶNG + guard chống bịa

```bash
curl -s localhost:16021/v1/ask -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{"question":"co tai nghe nao chong on duoi 300 nghin khong?"}' | python3 -m json.tool
```
**Kết quả thật:** answer nêu đúng `hoc-sp-01` #1 (250.000đ khớp catalog) · `llm_used: false` ·
`answer_source: "template"` · `grounding_guard: {blocked: false, ungrounded_ids: []}`.

**4 chặng bên trong (mổ từ code):**
1. **HIỂU câu hỏi = MÁY LUẬT, không LLM** (`core/query_parse.py`): regex + từ điển đơn vị
   `{k/nghin/ngan: ×1.000, tr/trieu: ×1.000.000}` + bỏ dấu → "duoi 300 nghin" → `price_max=300000`.
   Vì sao luật: chạy MỌI request (rẻ, tức thời) · tất định · miền hẹp đếm được · không phụ thuộc LLM sống chết.
2. **TÌM = RRF fusion**: BM25 (chữ) + vector (nghĩa) chạy song song → trộn điểm `Σ 1/(60+rank)`.
   Đọc điểm: 0.0328 = 1/61+1/61 = đứng **#1 cả 2 bảng**. Vector nén từ `title+brands+category_l1+description`
   (hàm `build_doc_text`) thành 1 vector 1024 chiều — cả title LẪN description đều được so.
   **Phân biệt LỌC vs KHỚP:** giá = LỌC CỨNG (loại hẳn ai >300k) · "chống ồn" = ĐIỂM MỀM (chấm cao thấp,
   không loại) → ốp lưng "chống sốc" vớt điểm chữ "chống" lết vào top-5.
3. **VIẾT**: có LLM key → LLM viết văn tư vấn NHƯNG chỉ về danh sách đã tìm; không key → template đổ khuôn
   (đúng tuyệt đối, khô). LLM = gia vị, không phải xương sống — hệ không chết theo nhà cung cấp LLM.
4. **SOÁT = grounding guard** (`core/grounding.py`, máy luật thuần): **B − A** với
   A = "danh sách khách mời" (product_id do chặng 2 trả) · B = mã-hàng-hình-dạng-`xx-yyy` mà REGEX quét được
   trong CHÍNH VĂN BẢN answer (sản phẩm của chặng 3). B−A ≠ ∅ = có "khách không mời" = bịa → chặn cả câu.
   A và B sinh từ 2 nguồn ĐỘC LẬP → người viết không vừa-đá-bóng-vừa-thổi-còi. +3 cánh tay: echoed_markers
   (nhại token lạ từ câu hỏi — chống injection) · meta_vocabulary (lộ từ hậu trường) · prompt_echo (chép prompt).
   Giới hạn thật: guard giữ DANH TÍNH, không thẩm định từng mệnh đề văn xuôi (pin 50h vs 30h không bắt được).
**PHÁT HIỆN THẬT (đã ghi nợ `W-ASK-COMPOSE-RELEVANCE`):** template lấy top-3 đổ khuôn KHÔNG lọc ngành →
ốp lưng lọt câu trả lời tai nghe. Gate: item trong answer phải cùng ngành với intent.

## 3.5 — VAI A/C: BƠM HÀNH VI (XEM + MUA) + 2 BÀI HỌC LỚN

⚠ Đúng CỬA: `product.viewed`→search:16021 · `purchase.completed`→forecast:16023 · `stock.level`/`cost.recorded`/`price.changed`→decision:16022. Cùng phong bì `{event_id, schema_version, event_type, event_time, payload}`.

```bash
# 0) BUG MUI GIO kinh dien (dinh that trong buoi hoc): "Z" = UTC, VN = UTC+7. Ghi gio-dia-phuong+Z
#    = event "tuong lai" -> validator chan (future >5min = INVALID; qua khu >90d phai di events:backfill).
EVT=$(date -u +%Y-%m-%dT%H:%M:%SZ) && echo "event_time=$EVT"

# 1) khach XEM
curl -s localhost:16021/v1/events:ingest -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{"events":[{"event_id":"hoc-ev-view-001","schema_version":"1.0",
  "event_type":"product.viewed","event_time":"'$EVT'","user_pseudo_id":"hoc-user-1","session_id":null,
  "attribution_token":null,"payload":{"product_id":"hoc-sp-01"}}]}' | python3 -m json.tool

# 2) khach MUA 2 chiec (1 event = 1 DON, don nay 2 CHIEC — phan biet event/don/chiec!)
curl -s localhost:16023/v1/events:ingest -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{"events":[{"event_id":"hoc-ev-buy-001","schema_version":"1.0",
  "event_type":"purchase.completed","event_time":"'$EVT'","user_pseudo_id":"hoc-user-1","session_id":null,
  "attribution_token":null,"payload":{"order_ref":"hoc-order-001","items":[{"product_id":"hoc-sp-01","qty":2,"unit_price":250000}]}}]}' | python3 -m json.tool

# 3) chay lai NGUYEN VAN lenh 2 → thi nghiem dedup
```
**Kết quả thật:** (1)(2) `accepted: 1` + **`ledger_position` 864895/864896** — sổ cái sự kiện toàn hệ đánh số
tuần tự (số này khớp `ledger_head` trong consistency của suggest!). (3) `accepted: 0, deduped: 1` — và bài
đo được còn mạnh hơn dự kiến: lần chạy lại có `event_time` MỚI mà vẫn dedup ⇒ **dedup khóa theo DANH TÍNH
(`event_id`), không theo nội dung.** Trả 200 + errors rỗng = idempotent ACK lịch sự, client retry vô tư.
**Quy tắc:** event_id sinh từ nghiệp vụ thật (`order-<mã đơn>`), KHÔNG random mỗi lần gửi. Mua lần 2 thật =
id mới. Idempotent 3 tầng đã gặp: compose up (Bài 1) · upsert theo id (3.1) · ingest theo event_id (3.5).

## 3.6 — VAI B: DỰ BÁO — VÀ CHỨNG KIẾN MODEL TIẾN HÓA THEO DỮ LIỆU CỦA MÌNH

```bash
curl -s localhost:16023/v1/forecast:query -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{"product_id": "hoc-sp-01", "horizon_days": 7}' | python3 -m json.tool
```
**Chuyện thật của buổi học — chạy 2 lần, 2 bộ não:**

| | Lần 1 (SKU 0 ngày sổ) | Lần 2 (sau khi event mua vào demand_daily) |
|---|---|---|
| `model_used` | `cold_start_analog` | `similar_item_transfer` |
| `data_window` | null (khai thật: 0 sử) | `2026-08-06..2026-08-06` (1 ngày) |
| CỠ (bán bao nhiêu/ngày) | ĐOÁN = mực chung nhóm hàng xóm 3.563 | **ĐO = 2.0 thật của chính nó** |
| totals p50 tuần | 25.7 | 12.4 (dè dặt hơn vì neo số thật) |

**Cơ chế `cold_start_analog` (giải phẫu khớp từng chữ số với API):**
1. Hỏi smartsearch 5 hàng xóm gần nhất theo embedding (thật: mì Hảo Hảo, Baseus, áo thun, ốp lưng, sạc —
   1/5 đúng ngành! cùng gốc bệnh W-RECO-PDP-COLDSTART). Lọc: cần ≥7 ngày sổ (`MIN_ANALOG_DAYS=7` — chưa sống
   đủ 1 tuần thì không có nhịp thứ-trong-tuần để mượn) + từng bán >0.
2. Sổ thật mỗi anh (bảng `demand_daily` — tổng kết tự động của event mua): **mực riêng = tổng ÷ số ngày**
   (Hảo Hảo 297÷62=4.790). Sổ dài ngắn khác nhau OK — ai chia theo sổ nấy.
3. **Mực chung = trung bình 5 mực riêng** = 17.813÷5 = 3.563 · **hệ số mỗi anh = 3.563 ÷ mực riêng**
   (Hảo Hảo ×0.744 bị nén, Baseus ×1.292 được kéo) — quy đồng "âm lượng", mượn NHỊP không mượn CỠ.
4. Ngày tương lai soi ngày CÙNG-THỨ tuần chót (seasonal-naive): **p50 = trung bình 5 phiếu-đã-nhân-hệ-số ·
   p10 = phiếu MIN × 0.8 · p90 = phiếu MAX × 1.2** (BAND_WIDEN=0.2 — số NGƯỜI CHỌN, hardcode, prior cho
   "người lạ"). Đã tính tay đủ 7 ngày khớp API từng chữ số (07/08: 0.805/3.215/7.140).
5. Điều CỐ TÌNH không làm: scale theo GIÁ — SKU mới chưa đo được phản ứng giá; thêm vào = "nhiễu mặc áo
   tín hiệu". Kỹ sư giỏi biết TỪ CHỐI yếu tố chưa đo được.

**Cơ chế `similar_item_transfer` (nấc kế — khớp từng chữ số):** chọn 1 hàng xóm tốt nhất, lấy nhịp tuần chót,
nhưng **hệ số = mực-riêng-CỦA-CHÍNH-SKU ÷ mực-riêng-hàng-xóm** = 2.0÷4.790 = 0.4175 → cả dải p50 = tuần
Hảo Hảo × 0.4175 (8→3.340, 7→2.923...). CỠ THẬT của mình + nhịp mượn. Nới cánh ×1.3 (lại số-người-chọn).

**Cái thang model đầy đủ (router phân loại chuỗi trước, chọn model sau):**
```
0 ngày → cold_start_analog (mượn cả cỡ lẫn nhịp) → vài ngày → similar_item_transfer (cỡ thật + nhịp mượn)
→ bán-lắt-nhắt → Croston/SBA (tách "bao lâu 1 đơn × đơn to cỡ nào", dải NBD)
→ đều, ≤56 ngày → seasonal_naive → đều, >56 → AutoETS+Theta (fallback naive)
→ cả shop ≥50 SKU × ≥120 ngày → LightGBM global (demoshop 62 ngày: SKIP trung thực trong check-apis)
```
Triết lý: **data mỏng thì model đơn giản THẮNG model xịn** — model_used khai thật đang đứng nấc nào.

**Đọc output:** dải p10/p50/p90 = "chắc ít nhất / kỳ vọng / hiếm khi vượt" — P50 kế hoạch · P90 nhập hàng ·
P10 dòng tiền. `totals` KHÔNG cộng quantile ngày (sai toán — ngày đen đỏ bù trừ) mà Monte Carlo 2000 lượt
(`triangular_mc_2000`). `calibration`: backtest = THI THỬ TRÊN QUÁ KHỨ (cần ≥14 ngày) → `empirical_coverage`
so chuẩn 0.8 → `width_factor` tự nới/thắt CÁNH quanh p50 (z-score, kẹp [0.5, 3.0]) — vòng tự-chấm-tự-chỉnh
chạy bằng job nền, không ai sửa code. **3 loại số trong mọi hệ ML: đếm-từ-sổ / suy-ra / người-chọn-có-tên.**

## 3.7 — VAI B: XIN LỜI KHUYÊN (decisions:run) — 3 KHOANG

```bash
# 1) kich hoat
curl -s -X POST localhost:16022/v1/decisions:run -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{}' | python3 -m json.tool
# 2) danh sach (truong that: items[]; EV o expected_value; giai thich CO SO o trace)
curl -s "localhost:16022/v1/decisions?page_size=5" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" \
  | python3 -c "
import json,sys
for x in json.load(sys.stdin).get('items', [])[:5]:
    ev = (x.get('expected_value') or {})
    print(x.get('decision_id'), '|', x.get('kind'), '|', x.get('status'), '|', f\"EV {ev.get('amount',0):,.0f} {ev.get('currency_code','')}/{ev.get('per','')}\")"
# 3) mo nguyen con 1 quyet dinh
curl -s "localhost:16022/v1/decisions?page_size=1" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
```
**Khoang 1 — RÀ 7+1 lăng kính:** below_cost · cost_increase · price_suggestion (pricing_mode Bài 2 làm việc
ở đây) · replenishment (ăn P90+lead-time từ forecast) · stockout_warning · slow_mover · promo_legal (trần
promo_cap Bài 2) · bundle_suggestion (mua-chung từ đơn thật).
**Khoang 2 — 4 CỬA KỶ LUẬT (kết quả thật `created: 0` + bản khai):** `skipped_dedup 158` (không spam lời
khuyên trùng đang mở) · `anti_oscillation 114` (thời gian nguội — cấm sáng khuyên tăng chiều khuyên giảm) ·
`plan_conflict 110` (không đá kế hoạch đang chạy) · `no_cost 27 / no_stock 2` (thiếu dữ liệu = im, không đoán).
created:0 kèm bản khai = kết quả TỐT — AI có kỷ luật im lặng.
**Khoang 3 — ĐÓNG GÓI (giải phẫu thật con bundle):** `subject` (cặp SKU) · `action_params` (bundle_price/
voucher/margin — làm được ngay) · `expected_value` (177.464đ/tháng, basis profit_delta) · `confidence` ·
`guardrails` (VOUCHER_MARGIN_FLOOR PASS) · **`trace` = chứng minh có số, audit từng phép nhân:**
"lift=19.42 (>=2.0), pair_cnt=26 (>=5)... EV = 0.15*26*(11819+33684) = 177464".
**Status:** open = chờ chủ shop · accepted (probe check-apis tự accept con đầu — đọc trạng thái phải hỏi
"ai gây ra") · **EV là cột XẾP VIỆC**: dải thật 177k→2.5M (gấp 14 lần) — làm con to trước.

## 3.8 — THỬ GIÁ (price-preview): CHUỖI 3 CỔNG 412 → 200 "ĐỪNG GIẢM GIÁ"

```bash
curl -s -w "\nstatus: %{http_code}\n" localhost:16022/v1/decisions:price-preview \
  -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{"product_id": "hoc-sp-01", "candidate_price": 199000}'
```
**Chuỗi 3 cổng (đo thật — mỗi 412 NÊU ĐÍCH DANH cổng kế = lỗi dẫn đường):**
1. `no sales in last 30d` → cần `purchase.completed` (có từ 3.5) ✓
2. `no cost recorded` → event **`cost.recorded`** `{product_id, unit_cost: 150000, qty: 25, supplier_ref}`
   (cửa decision) → worker `state_rollup` cuộn vào `cost_state.ewma_cost` — nhịp `STATE_ROLLUP_INTERVAL=300s`!
   Chờ bằng CỔNG ĐO (không sleep mò): `until psql ... cost_state ... | grep -q "[0-9]"; do sleep 30; done`
3. `no current price recorded` → event **`price.changed`** `{product_id, new_price: 250000}` → `price_state`.
   (Giá khai với smartsearch lúc upsert KHÔNG tự chảy sang decision — 3 DB tách biệt; giá-tính-lãi cần vết
   thời gian qua event, nuôi price_history/elasticity.)
→ **Onboard 1 SKU vào BT02 = 3 event: BÁN + VỐN + GIÁ.**

**Đọc response 200 (kết quả thật — máy CAN giảm giá):**
```
current   {price 250000, est_units_30d 2.0,   est_profit_30d 200000}    = (250k−150k)×2
candidate {price 199000, est_units_30d 2.735, est_profit_30d 134031}    = (199k−150k)×2.735
delta_profit_30d = −65.968đ/tháng  → ĐỪNG LÀM
elasticity_used {eps −1.372, method pooled_prior, n_points: 0}  ← khai thật: 0 điểm đo giá riêng, mượn của shop
explanation "Q(P)=Q0·(P/P0)^eps; profit=(P−c)·Q"  ← công thức lộ thiên; tự kiểm: 2×(199/250)^−1.372 = 2.735 ✓
guardrails [BELOW_COST PASS] · confidence 0.7
```
Nghĩa: hạ giá 20% → bán thêm ~37% nhưng lãi/chiếc 100k→49k → tổng lãi GIẢM. Máy can bằng số, chặn trực giác
"giảm giá cho chạy hàng" TRƯỚC khi mất tiền thật. Triết lý thiếu-thì-từ-chối xuất hiện 3 nơi: guard ask (3.4)
· cửa no_cost :run (3.7) · 412 preview (3.8) — một DNA xuyên hệ.

## 3.9 — VAI C: FEEDBACK (khép vòng — lời khuyên có KPI)

```bash
# chon 1 decision_id dang OPEN tu 3.7 (con EV cao nhat truoc)
curl -s -w "\nstatus: %{http_code}\n" "localhost:16022/v1/decisions/<DECISION_ID>:feedback" \
  -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{"action": "accepted"}'
```
**Đã làm thật:** accept con combo Nike+vớ (EV 2.500.808đ/tháng, lift 20, 20 đơn mua chung) → response trả
nguyên hồ sơ với `status: "accepted"`. Action chỉ nhận `accepted`/`dismissed`. Từ lúc accept, **outcome-loop
30 ngày** bắt đầu đếm — đầu tháng 9 hệ đối chiếu lời hứa 2.5M với sổ thật và tự chấm điểm (T-OUTCOME-30D).
Lời khuyên = cam kết có hồ sơ theo dõi, không phải chatbot phủi tay.

## 3.10 — DỌN SÂN (mượn sân thì trả nguyên trạng) — LỆNH 1 DÒNG chống ngắt-dòng

```bash
curl -s -o /dev/null -w "xoa lan 1: %{http_code}\n" -X DELETE localhost:16021/v1/products/hoc-sp-01 -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop"; curl -s -o /dev/null -w "xoa lan 2: %{http_code}\n" -X DELETE localhost:16021/v1/products/hoc-sp-01 -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop"
```
**Kỳ vọng:** lần 1 `204` (xóa xong, không cần body) · lần 2 `404` (nói thật "đã không còn", không giả vờ
thành công) — đúng probe `products:delete 204+404` của nhà.

### 3.10b — VÌ SAO KHÔNG XÓA ĐƯỢC EVENT + cách dọn ĐÚNG (bút toán đảo — human chất vấn, bài học đắt)
Event = **sổ cái kế toán**: không tẩy dòng, sai thì ghi dòng ĐẢO. 4 lý do immutable: (1) kiểm toán tiền/kho/
thuế; (2) replay/watermark — projection cuộn theo ledger_position, rút dòng giữa = số hạ nguồn mồ côi;
(3) dedup — xóa event = giải phóng event_id = mở cửa đếm trùng; (4) forecast/decision đã "ăn" event — chỉ
sự kiện mới dạy lại được chúng.
**Bút toán đảo cho từng loại:** đơn sai/dọn training → **`order.returned`** `{order_ref, items, reason}` (có
trường reason để ghi lý do vào sổ!) · vốn sai → `cost.recorded` giá trị đúng (EWMA trôi theo) · giá sai →
`price.changed` giá trị đúng (ghi đè current_price).
```bash
EVT=$(date -u +%Y-%m-%dT%H:%M:%SZ) && curl -s localhost:16023/v1/events:ingest -H "Authorization: Bearer $FKEY" \
 -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"events":[{"event_id":"hoc-ev-return-001",
 "schema_version":"1.0","event_type":"order.returned","event_time":"'$EVT'","user_pseudo_id":"hoc-user-1",
 "session_id":null,"attribution_token":null,"payload":{"order_ref":"hoc-order-001","items":[{"product_id":
 "hoc-sp-01","qty":2,"unit_price":250000}],"reason":"don tap huan - trung hoa truoc demo"}}]}' | python3 -m json.tool
# xac nhan trung hoa (cho nhip rollup ~5 phut):
docker exec miniai-postgres psql -U miniai -d miniai_forecast -c "SELECT COALESCE(sum(adjusted_units),0) FROM demand_daily WHERE project_id='demoshop' AND product_id='hoc-sp-01';"   # ky vong 0
```
**Dọn sân TRỌN VẸN = xóa catalog (204/404) + trung hòa event bằng sự kiện đảo + đo lại (demand ròng=0 ·
check-apis 43/43).** Chưa đo thấy 0 = chưa được tuyên xong. Bonus đo thật buổi học: event product.viewed
của buổi tập làm chain `user_profile rows` chuyển SKIP→PASS count=1 — dữ liệu tập cũng nuôi hệ khôn lên.

---

## SỔ TAY CHỐT BÀI 3
1. Chuỗi sống đi bằng tay: upsert → outbox → Vespa → search/ask → events → demand_daily → forecast (tiến hóa
   analog→transfer) → decisions (7 lăng kính, 4 cửa) → preview (3 cổng) → feedback (outcome-loop).
2. Eventual consistency 2 nơi: outbox→Vespa (giây) · event→state_rollup (nhịp 300s) — chờ bằng CỔNG ĐO.
3. Idempotent 3 tầng: upsert theo id · ingest dedup theo event_id (danh tính, không nội dung) · compose up.
4. UTC mọi nơi (`date -u`); "Z"=UTC; VN=UTC+7; validator chặn future>5min, quá khứ>90d đi backfill.
5. Sổ cái ledger đánh số tuần tự toàn hệ — `ledger_position` lúc ghi khớp `ledger_head` lúc đọc.
6. 3 loại số: đếm-từ-sổ / suy-ra / người-chọn-có-tên (BAND_WIDEN 0.2, widen 1.3, MIN_ANALOG_DAYS 7,
   ROLLUP 300s...) — số người-chọn phải có tên + có cơ chế đo lại (calibration).
7. DNA trung thực xuyên hệ: model_used/data_window khai thật · SKIP-có-lý-do · 412 nêu đích danh ·
   created:0 kèm bản khai · n_points:0 nói rõ mượn prior.
8. Lỗi tử tế dẫn đường: 3 cổng 412 = checklist onboard SKU (bán+vốn+giá).
9. Test bằng dữ liệu MÌNH TỰ TẠO (biết đáp án) → chấm được máy → 2 phát hiện thật đã thành nợ có tên.

## SỔ NỢ PHÁT HIỆN TRONG BÀI (đã ghi DB, có gate đóng)
- `W-RECO-PDP-COLDSTART` — recommend pdp SKU mới fallback popular toàn-shop (kem chống nắng trên trang tai nghe).
- `W-ASK-COMPOSE-RELEVANCE` — ask template đổ top-3 không lọc ngành (ốp lưng trong câu tai nghe).
- (Bài 1: `W-RUN-ASYNC-202` · `W-PREFLIGHT-LOAD-GUARD` — cùng chiến dịch sửa sau training.)

## BÀI TẬP TỰ LUYỆN
1. Bơm 5-6 đơn mua rải nhiều ngày cho 1 SKU mới → xem forecast leo nấc (`model_used` đổi, dải hẹp lại).
2. Ask câu KHÔNG có trong catalog ("co ban o to dien khong?") → quan sát guard/template trả lời.
3. Gửi event thiếu trường payload → đọc `events:dead` xem xác event nằm đâu.
4. Làm lại chuỗi 3 cổng 412 cho một SKU mới khác, không mở giáo trình.
5. Preview với candidate_price 300000 (TĂNG giá) — delta dương hay âm? Vì sao? (elasticity −1.372)

## CÂU PHỎNG VẤN TỰ LUYỆN (dẫn số thật)
1. "Upsert xong search chưa thấy — bug?" (outbox, 2-thì upserted/queued, count=0)
2. "Chống đếm trùng đơn khi retry?" (dedup theo event_id — tự đo `deduped:1` với event_time khác)
3. "SKU 0 lịch sử thì forecast kiểu gì?" (analog: mượn nhịp quy đồng cỡ, min/max ±20%; lên nấc transfer khi
   có sử: cỡ thật 2÷4.79=0.4175; khai model_used/data_window thật)
4. "Vì sao ask không chết khi LLM outage?" (4 chặng: luật/RRF/template-fallback/guard B−A)
5. "AI của anh có spam chủ shop không?" (4 cửa: dedup/anti-osc/plan-conflict/no-data + created:0 kèm bản khai)
6. "Làm sao biết lời khuyên AI đáng tin?" (trace có số audit được · guardrails · EV xếp việc · outcome-loop
   30 ngày chấm bằng tiền thật · preview can giảm giá −66k/tháng)
