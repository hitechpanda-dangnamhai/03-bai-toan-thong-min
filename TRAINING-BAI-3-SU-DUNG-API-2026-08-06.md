# BÀI 3 — SỬ DỤNG API TRỌN CHUỖI: ĐÓNG VAI KHÁCH TÍCH HỢP

> Giáo trình training human (xuất 2026-08-06). Bài 0 = bản đồ API · Bài 1 = vận hành · Bài 2 = cấu hình & key.
> Điều kiện vào bài: stack xanh (Bài 1) + hiểu 2 header xác thực (Bài 2).
> Mọi lệnh chạy từ: `cd /home/hai-soft/projects/icpp/mecom/project` · tenant: **demoshop** (đã đăng ký đủ 3 key).
> Mọi payload trong bài lấy NGUYÊN VĂN từ bộ probe thật (`seedtool/checker.py`) — đây là các request đã chạy 42/42.

**Mục tiêu:** tự tay đi trọn vòng đời nghiệp vụ với MỘT sản phẩm CỦA BẠN: tạo nó → tìm thấy nó → bơm hành vi
mua nó → xem dự báo của nó → xin lời khuyên → phản hồi → dọn sân. Đi hết là "hiểu bằng cơ bắp" sơ đồ chuỗi
sống A → B → C của Bài 0.

**Sơ đồ hành trình (dán lên tường):**
```
3.1 upsert (vai A) → 3.2 search → 3.3 suggest/recommend → 3.4 ask   (vai B - BT01)
       │
3.5 events: viewed + purchase (vai A/C)
       │
3.6 forecast:query dải P10/P50/P90 (vai B - BT03)
       │
3.7 decisions:run + list (vai B - BT02) → 3.8 price-preview → 3.9 feedback (vai C)
       │
3.10 dọn sân (delete 204/404)
```

---

## 3.0 — CHUẨN BỊ: nạp 3 chìa vào biến (chạy 1 lần cho cả buổi)

```bash
cd /home/hai-soft/projects/icpp/mecom/project
SKEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['search'])")
DKEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['decision'])")
FKEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['forecast'])")
echo "da nap 3 key (khong in)"
```
**Ghi chú:** biến shell chỉ sống trong terminal hiện tại — mở tab mới phải nạp lại. Từ đây: cửa search dùng
`$SKEY`+:16021 · decision `$DKEY`+:16022 · forecast `$FKEY`+:16023.

---

## 3.1 — VAI A: KHAI SINH SẢN PHẨM CỦA BẠN (products:upsert)

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
    "availability": "IN_STOCK",
    "available_quantity": 25,
    "attributes": {},
    "images": [],
    "publish_time": "2026-08-06T09:00:00Z"
  }]
}' | python3 -m json.tool
```
**Kỳ vọng:** `{"upserted": 1}`.
**Giải nghĩa:** đây là API vai A quan trọng nhất — catalog là nguồn sống của cả BT01. Body là mảng `products`
(một lần đẩy được nhiều SKU — khách thật đồng bộ cả nghìn SKU/lượt). `id` là SKU do KHÁCH đặt (hệ không tự sinh
— để khớp với hệ quản kho của khách). Upsert = có thì cập nhật, chưa có thì tạo — idempotent, đẩy lại không sao.

## 3.2 — VAI B: TÌM THẤY CHÍNH NÓ (search) + bài học EVENTUAL CONSISTENCY

**Bản THÍ NGHIỆM (khuyên dùng khi học — quan sát hàng đợi VÀ kết quả search cùng một khoảnh khắc):**
```bash
docker exec miniai-postgres psql -U miniai -d miniai_search -c "SELECT count(*) FROM catalog_outbox;" \
&& curl -s localhost:16021/v1/search -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{"query":"tai nghe mecom"}' \
  | python3 -c "import json,sys; d=json.load(sys.stdin); items=d.get('items',[]); print(f'tim thay {len(items)} ket qua:'); [print(' ', i.get('product_id'), '|', i.get('title')) for i in items[:5]]"
```
**Cách đọc — dù rơi vào nhánh nào cũng tự giải được:**
- `count = 0` → worker đã đẩy xong hàng đợi → search PHẢI thấy `hoc-sp-01` trong danh sách.
- `count > 0` → còn việc đang chờ → search chưa thấy sản phẩm thì KHÔNG phải lỗi — chờ 3-5 giây chạy lại cả
  cụm, xem count tụt về 0 và sản phẩm hiện ra. (Chứng kiến cảnh "chưa thấy → thấy" còn quý hơn thấy ngay —
  đó là eventual consistency bằng xương thịt.)

**Bản QUY TRÌNH (khi tích hợp thật, chỉ cần search):**
```bash
curl -s localhost:16021/v1/search -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{"query":"tai nghe mecom"}' \
  | python3 -c "import json,sys; d=json.load(sys.stdin); [print(i.get('product_id'), '|', i.get('title')) for i in d.get('items',[])[:5]]"
```
Search chưa thấy → mới quay lại soi outbox như trên (soi-queue là công cụ chẩn đoán, không phải bước bắt buộc).

**Giải nghĩa kiến trúc:** đường đi của sản phẩm là
`upsert → bảng catalog (PG) → catalog_outbox → worker đẩy vào Vespa → search thấy`. Mẫu **outbox** là chuẩn
ngành: ghi sản phẩm + ghi việc-cần-đẩy trong CÙNG 1 transaction PG (không bao giờ "ghi DB xong mà quên đẩy
index"); Vespa chết thì việc nằm chờ, sống lại đẩy tiếp — không mất, chỉ muộn. Chính vì thế response 3.1 có
`queued_for_index: 1` — API tự khai 2 thì: đã-ghi-sổ (upserted) và sẽ-đánh-chỉ-mục (queued).

**Bài học xếp hạng (kết quả thật 2026-08-06):** `hoc-sp-01` đứng **#2**, thua `Sony WH-CH520` dù query có chữ
"mecom" — KHÔNG phải lỗi. Ranking lai = khớp chữ (BM25) + khớp nghĩa (vector) + **tín hiệu hành vi**
(click/mua lịch sử). Sony là hàng seed có lịch sử click thật; sản phẩm mới toanh tín hiệu hành vi = 0. Hệ nói:
"khớp query thì có, nhưng chưa ai chứng thực". Muốn lên hạng → cần vai C (bơm event 3.5), không phải sửa title.

## 3.3 — VAI B: AUTOCOMPLETE + GỢI Ý NGỮ CẢNH

```bash
# suggest: nguoi dung dang go "tai ng..."
curl -s "localhost:16021/v1/suggest?q=tai%20ng" -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
# recommend theo ngu canh trang san pham (pdp) cua chinh hoc-sp-01
curl -s localhost:16021/v1/recommend -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{"context":"pdp","product_id":"hoc-sp-01"}' \
  | python3 -c "import json,sys; d=json.load(sys.stdin); [print(i.get('product_id'), '|', i.get('title')) for i in d.get('items',[])[:5]]"
```
**Kỳ vọng:** suggest trả mảng gợi ý chữ · recommend trả sản phẩm "tương tự/mua kèm" cho trang hoc-sp-01.
**Giải nghĩa:** 4 ngữ cảnh recommend: `home` (đầu trang, cần `user_pseudo_id`) · `pdp` (trang sản phẩm, cần
`product_id`) · `similar` · `cart`. Suggest học từ query THẬT của người dùng trộn n-gram title — shop mới ít
query thì gợi ý còn thô, càng chạy càng khôn (vai C nuôi).

## 3.4 — VAI B: HỎI TỰ NHIÊN (ask — có guard chống bịa)

```bash
curl -s localhost:16021/v1/ask -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{"question":"co tai nghe nao chong on duoi 300 nghin khong?"}' | python3 -m json.tool
```
**Kỳ vọng:** JSON có trường `answer` — câu trả lời phải bám vào catalog thật (nhắc được sản phẩm của bạn nếu khớp).
**Giải nghĩa:** ask = compose tư vấn grounded vào catalog; có guard chặn echo id lạ/bịa sản phẩm (đã fuzz 104 ca,
1 ca compound-id còn tracked `W-ASK-GUARD-COMPOUND-ID`). Không có LLM key thì fallback template — minh bạch.

## 3.5 — VAI A/C: BƠM HÀNH VI — XEM và MUA sản phẩm của bạn

⚠ Chú ý đúng CỬA: cùng một "phong bì" event nhưng mỗi loại nộp cho service chủ quản:
`product.viewed` → search :16021 · `purchase.completed` → forecast :16023 · `stock.level` → decision :16022.

```bash
# 0) sinh event_time = GIO UTC HIEN TAI (bug mui gio kinh dien: "Z" = UTC, VN = UTC+7 —
#    ghi gio-dia-phuong + Z se thanh "tuong lai" va bi validator chan: future >5min = INVALID)
EVT=$(date -u +%Y-%m-%dT%H:%M:%SZ) && echo "event_time=$EVT"

# 1) khach XEM san pham (nuoi search/LTR/user_profile)
curl -s localhost:16021/v1/events:ingest -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{
  "events": [{
    "event_id": "hoc-ev-view-001",
    "schema_version": "1.0",
    "event_type": "product.viewed",
    "event_time": "'$EVT'",
    "user_pseudo_id": "hoc-user-1",
    "session_id": null, "attribution_token": null,
    "payload": {"product_id": "hoc-sp-01"}
  }]
}' | python3 -m json.tool

# 2) khach MUA san pham (nuoi chuoi thoi gian forecast)
curl -s localhost:16023/v1/events:ingest -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{
  "events": [{
    "event_id": "hoc-ev-buy-001",
    "schema_version": "1.0",
    "event_type": "purchase.completed",
    "event_time": "'$EVT'",
    "user_pseudo_id": "hoc-user-1",
    "session_id": null, "attribution_token": null,
    "payload": {"order_ref": "hoc-order-001", "items": [{"product_id": "hoc-sp-01", "qty": 2, "unit_price": 250000}]}
  }]
}' | python3 -m json.tool
```
**Kỳ vọng:** mỗi cú trả `{"accepted": 1, "deduped": 0, "skipped": 0}`.
**Thí nghiệm idempotency (chạy lại NGUYÊN VĂN cú số 2):** lần 2 trả `accepted: 0, deduped: 1` — hệ nhận ra
`event_id` đã thấy, không đếm đơn 2 lần. Đây là lý do khách phải sinh `event_id` bền vững (không random mỗi lần
gửi): mạng chập gửi lại không làm sai số liệu. **Bất biến tiền/kho không vỡ nhờ dedup theo event_id.**

## 3.6 — VAI B: DỰ BÁO CHÍNH SKU CỦA BẠN (forecast:query)

```bash
curl -s localhost:16023/v1/forecast:query -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{"product_id": "hoc-sp-01", "horizon_days": 7}' | python3 -m json.tool | head -50
```
**Kỳ vọng:** 200, JSON có dải theo ngày + `model_used` + `data_window`.
**Cách đọc (quan trọng nhất bài):**
- `p10/p50/p90` mỗi ngày: "bán chắc ít nhất / kỳ vọng / hiếm khi vượt". P10 dùng cho cam kết an toàn (không hết
  hàng), P90 cho phòng rủi ro vốn đọng.
- `model_used` = SỰ TRUNG THỰC của hệ: SKU của bạn mới có 1 đơn → kỳ vọng thấy model đơn giản (naive/analog
  cold-start), KHÔNG phải LightGBM — hệ khai đúng "tôi biết ít nên dùng công cụ thô". SKU dày data mới được
  model xịn. Dải rộng = trung thực về độ bất định, không phải dở.
- `data_window`: hệ nhìn thấy bao nhiêu ngày lịch sử của SKU này — đối chiếu được vì bạn biết chính xác mình
  vừa bơm gì.
(Ghi chú: projections nền được tính khi `forecast:run` — check-apis đã chạy nó nhiều lần hôm nay; query SKU quá
mới có thể rơi vào nhánh cold-start/fallback — chính là điều đáng quan sát.)

## 3.7 — VAI B: XIN LỜI KHUYÊN (decisions:run + list)

```bash
# 1) kich hoat sinh quyet dinh
curl -s -X POST localhost:16022/v1/decisions:run -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{}' | python3 -m json.tool
# 2) doc danh sach quyet dinh dang mo
curl -s "localhost:16022/v1/decisions?page_size=5" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); [print(x.get('decision_id'),'|',x.get('kind'),'|',(x.get('explain') or x.get('title') or '')[:80]) for x in d.get('decisions', d.get('items', []))[:5]]"
```
**Kỳ vọng:** run trả `{"created": N}` · list in tối đa 5 quyết định: id | kind | giải thích tiếng Việt.
**Giải nghĩa:** 7 kind (below_cost, cost_increase, price_suggestion, replenishment, stockout_warning,
slow_mover, promo_legal). Mỗi quyết định kèm giải thích tiếng Việt + EV — người bán không cần hiểu ML vẫn
hành động được. `:run` chạy trên TOÀN BỘ data tenant (không riêng SKU của bạn). GHI LẠI 1 `decision_id` cho 3.9.
⚠ Bài học từ Bài 1: `:run` là endpoint nặng — lần đầu sau restart có thể chậm/timeout; phản xạ đã biết.

## 3.8 — VAI B: THỬ GIÁ TRƯỚC KHI ÁP (price-preview) + bài học mã 412

```bash
curl -s -w "\nstatus: %{http_code}\n" localhost:16022/v1/decisions:price-preview \
  -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{"product_id": "hoc-sp-01", "candidate_price": 199000}'
```
**Kỳ vọng: 200 HOẶC 412 — cả hai đều là bài học:**
- `200` = trả preview tác động (demand/doanh thu/margin dự kiến ở giá 199k).
- `412 PRECONDITION_FAILED` = hệ nói thẳng: "SKU này tôi CHƯA đủ dữ liệu (cost/elasticity) để ước lượng tử tế
  — tôi từ chối đoán bừa." SKU của bạn mới 1 đơn, chưa có giá vốn → 412 là câu trả lời TRUNG THỰC. So với hệ
  trả đại một con số đẹp: 412 chính là điểm cộng bán hàng ("hệ không bịa khi thiếu dữ liệu" — cùng triết lý
  guard của /v1/ask). Probe của nhà cũng chấp nhận `200 or 412 expected`.

## 3.9 — VAI C: KHÉP VÒNG PHẢN HỒI (feedback)

```bash
# thay <ID> bang decision_id ghi lai o 3.7
curl -s localhost:16022/v1/decisions/<ID>:feedback -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" -d '{"action": "accepted"}' | python3 -m json.tool
```
**Kỳ vọng:** 200.
**Giải nghĩa:** đây là mắt xích vai C của BT02: chủ shop chấp nhận/từ chối lời khuyên → hệ ghi nhận → sau 30
ngày outcome-loop đối chiếu lời khuyên với KẾT QUẢ THẬT (bán được không, margin ra sao) → đo được "lời khuyên
của AI đúng bao nhiêu %". Không có feedback = không bao giờ biết mình khuyên hay hay dở. (Hàng chờ theo thiết
kế: `T-OUTCOME-30D` — row hợp lệ đầu tiên ~09/2026 khi decision đủ 30 ngày tuổi.)

## 3.10 — DỌN SÂN (delete + bài học 204/404)

```bash
curl -s -o /dev/null -w "xoa lan 1: %{http_code}\n" -X DELETE localhost:16021/v1/products/hoc-sp-01 -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop"
curl -s -o /dev/null -w "xoa lan 2: %{http_code}\n" -X DELETE localhost:16021/v1/products/hoc-sp-01 -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop"
```
**Kỳ vọng:** lần 1 `204` (No Content — xóa xong) · lần 2 `404` (Not Found — đã không còn).
**Giải nghĩa:** cặp 204/404 là hợp đồng API chuẩn mực: xóa thành công không cần body; xóa thứ không tồn tại phải
nói thật 404 (không giả vờ 204). Probe của nhà kiểm đúng cặp này (`products:delete 204+404`).
(Event `hoc-ev-*` đã bơm thì Ở LẠI — event là sự kiện lịch sử, đúng thiết kế không có API xóa; 2 event học tập
không ảnh hưởng số liệu demo.)

---

## SỔ TAY — GHI CHÚ CHỐT BÀI 3
1. Chuỗi sống bạn vừa đi bằng tay: upsert → (outbox → Vespa) → search/suggest/recommend/ask → events → forecast
   → decisions → feedback. Đúng sơ đồ Bài 0, giờ là trải nghiệm cơ bắp.
2. Eventual consistency: upsert xong search chưa thấy ngay = bình thường; soi `catalog_outbox` count = 0 là đã đẩy hết.
3. Idempotency 2 tầng: upsert theo `id` sản phẩm · event dedup theo `event_id` (`deduped: 1` khi gửi lại) — mạng
   chập không làm sai số.
4. Mỗi loại event nộp đúng cửa service chủ quản; cùng phong bì `{event_id, event_type, event_time, payload}`.
5. Đọc forecast: P10/P50/P90 + `model_used` trung thực (SKU mỏng → model thô, dải rộng = trung thực).
6. 412 = "tôi thiếu dữ liệu, không đoán bừa" — cùng DNA với guard chống bịa của ask.
7. 204 rồi 404 khi xóa 2 lần = hợp đồng API tử tế.

## BÀI TẬP TỰ LUYỆN
1. Tạo thêm 2 sản phẩm cùng brand MECOM, bơm 5-6 purchase rải nhiều ngày (`event_time` khác nhau) cho 1 SKU,
   rồi `forecast:query` lại — quan sát `data_window` và dải thay đổi.
2. Dùng `recommend context=similar` với 1 sản phẩm demoshop có sẵn — so kết quả với `pdp`.
3. Thử ask một câu KHÔNG có trong catalog ("co ban o to dien khong?") — quan sát guard trả lời thế nào.
4. Gửi 1 event với `event_time` tương lai hoặc `payload` thiếu trường — xem hệ chặn ra sao (`events:dead` có gì?).
5. Dọn: xóa các sản phẩm học tập đã tạo thêm (nhớ cặp 204/404).

## CÂU PHỎNG VẤN TỰ LUYỆN (dẫn số thật)
1. "Khách upsert xong search chưa thấy — bug không? Kiến trúc nào đứng sau?" (outbox pattern, eventual
   consistency, cách kiểm count=0)
2. "Làm sao hệ không đếm trùng đơn khi client retry?" (event_id dedup — demo `deduped:1` tự tay đo)
3. "SKU mới toanh thì forecast trả gì? Vì sao đó là thiết kế tốt?" (model_used trung thực, cold-start analog,
   dải rộng = trung thực về bất định)
4. "Vì sao price-preview trả 412 thay vì một con số?" (precondition, thà từ chối còn hơn bịa — triết lý xuyên
   suốt: ask guard, SKIP-có-lý-do của probe)
5. "Vòng C của BT02 đo bằng gì?" (feedback → outcome-loop 30 ngày, T-OUTCOME-30D ~09/2026)
