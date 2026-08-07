# DEMO 1 — SẢN PHẨM MỚI TINH: từ 0 dữ liệu đến ra quyết định
> Kịch bản 20 API, chạy trọn 1 vòng: **smart search → recommend → forecast → decision → phản hồi**.
> Mọi INPUT/OUTPUT dưới đây là **kết quả chạy thật trên demoshop lúc 2026-08-07 00:16-00:20**, không phải ví dụ.
> SKU demo đã được dọn sạch khỏi hệ sau khi đo ⇒ chạy lại sáng mai sẽ ra đúng như tài liệu.

## THÔNG ĐIỆP BÁN HÀNG CỦA MÀN NÀY
Khách hàng thấy tận mắt: hàng mới lên kệ **tìm được sau vài giây**, nhưng hệ **thẳng thắn từ chối** dự báo và
khuyên giá khi chưa có lịch sử bán — rồi sau khi nạp 21 ngày dữ liệu thật, cũng những API đó **trả lời đầy đủ
và tự khai đang dùng phương pháp nào, tin cậy tới đâu**. Đó là khác biệt giữa một hệ thống trung thực và một
hệ thống đoán bừa.

## CHUẨN BỊ (chạy 1 lần, trước khi khách vào phòng)
```bash
cd /home/hai-soft/projects/icpp/mecom/project
SKEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['search'])")
DKEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['decision'])")
FKEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['forecast'])")
ITOK=$(docker exec miniai-smartsearch printenv MINIAI_INTERNAL_TOKEN)
SKU=demo-mi-omachi
echo "keys ${SKEY:0:6}/${DKEY:0:6}/${FKEY:0:6} | internal ${ITOK:0:4}"
```
**Header dùng cho MỌI API nghiệp vụ:** `Authorization: Bearer <key đúng service>` + `X-Project-Id: demoshop`.
API nội bộ (`/internal/...`) dùng `X-Internal-Token` thay cho 2 header trên.
Cổng: smartsearch **16021** · decision **16022** · forecast **16023**.

---
# MÀN 1 — KHAI SINH SẢN PHẨM (1 API)

### [01] POST /v1/products:upsert — đưa hàng mới lên kệ
**Ý nghĩa:** khai sinh SKU. Đây là cửa duy nhất để sản phẩm vào hệ; ghi vào Postgres rồi đẩy sang Vespa qua
outbox (nhất quán cuối, độ trễ ~6-9 giây).

**INPUT — giải thích từng trường**

| Trường | Bắt buộc | Ý nghĩa |
|---|---|---|
| `id` | ✔ | mã SKU, khoá chính trong tenant |
| `title` | ✔ | tên hiển thị — **nguồn chính cho tìm kiếm** (BM25 + vector) |
| `description` | | mô tả, cũng được đánh chỉ mục |
| `categories` | ✔ | đường dẫn ngành `"Cha > Con"`; phần trước dấu `>` thành `category_l1` dùng cho lọc/gộp |
| `brands` | | thương hiệu, dùng cho lọc và khớp chính xác |
| `price_info.price` | ✔ | giá bán hiện tại (VND, số nguyên) |
| `price_info.original_price` | | giá gốc, để tính % giảm |
| `availability` | ✔ | `IN_STOCK` / `OUT_OF_STOCK` — hàng hết sẽ **biến mất khỏi kết quả tìm kiếm** |
| `available_quantity` | | tồn hiển thị |
| `publish_time` | ✔ | thời điểm lên kệ (ISO-8601 UTC) |

```bash
curl -s localhost:16021/v1/products:upsert -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"products":[{"id":"demo-mi-omachi","title":"Thùng 30 gói mì Omachi sườn hầm ngũ quả 80g","description":"Mì ăn liền Omachi sợi khoai tây, vị sườn hầm ngũ quả, thùng 30 gói 80g","categories":["Bách hóa > Mì ăn liền"],"brands":["Omachi"],"price_info":{"currency_code":"VND","price":145000,"original_price":165000},"availability":"IN_STOCK","available_quantity":40,"attributes":{},"images":[],"publish_time":"2026-08-07T00:00:00Z"}]}'
```

**OUTPUT thật**
```json
{"upserted": 1, "queued_for_index": 1}
```
**Đọc kết quả:** `upserted=1` = đã ghi vào cơ sở dữ liệu (bền, không mất). `queued_for_index=1` = đã xếp hàng
đẩy sang công cụ tìm kiếm. Hai số tách nhau **có chủ đích**: ghi bền và đánh chỉ mục là hai việc khác nhau,
nếu Vespa chết thì dữ liệu vẫn còn và tự bù khi sống lại.

> ⏱ **Chờ 6-9 giây** rồi mới sang màn 2 (thời gian outbox đẩy sang Vespa). Nói với khách: *"đây là độ trễ
> thật của mọi sàn thương mại điện tử — ghi một nơi, tìm kiếm một nơi."*

---
# MÀN 2 — TÌM ĐƯỢC NGAY, NHƯNG CHƯA CÓ TRÍ KHÔN (6 API)

### [02] POST /v1/search — hàng mới đã tìm thấy
**Ý nghĩa:** tìm kiếm lai — BM25 (khớp chữ) + vector ngữ nghĩa (BGE-M3), trộn điểm bằng RRF.

**INPUT:** `query` (chuỗi khách gõ, chịu được không dấu và gõ sai) · `page_size` (số kết quả).
```bash
curl -s localhost:16021/v1/search -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"query":"omachi","page_size":5}' | python3 -m json.tool
```
**OUTPUT thật**
```json
{"items": [{"product_id": "demo-mi-omachi",
            "title": "Thùng 30 gói mì Omachi sườn hầm ngũ quả 80g",
            "price_info": {"currency_code": "VND", "price": 145000},
            "availability": "IN_STOCK", "score": 10.1054, "source": "rrf_fusion"}]}
```
**Đọc kết quả:** `score` = điểm khớp sau khi trộn (càng cao càng khớp) · `source: "rrf_fusion"` = kết quả đến
từ việc **trộn hai đường** chứ không chỉ khớp chữ. Điểm 10.1 với đúng 1 kết quả = khớp thương hiệu chính xác.
**Sản phẩm vừa tạo 9 giây trước đã tìm được** — đó là điều cần nói với khách.

> ⚠ **Truy vấn nên dùng khi demo:** `omachi` (tên thương hiệu). **Tránh** gõ `mi omachi` hay `mi an lien`:
> tiếng Việt bỏ dấu, "mi" 2 ký tự khớp cả "sơ **mi**", "hâm"/"máy hâm sữa" nên hàng mới bị đẩy xuống #3.
> Đây là điểm yếu đã đo và đã ghi sổ (nợ `W-SEARCH-CONCEPT-NEGATION`), không giấu.

### [03] GET /v1/suggest — gợi ý gõ phím đã có từ mới
**INPUT:** `q` (tiền tố khách đang gõ) · `limit`.
```bash
curl -s "localhost:16021/v1/suggest?q=omachi&limit=5" -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
```
**OUTPUT thật**
```json
{"items": [{"text": "omachi", "weight": 1.0},
           {"text": "omachi sườn", "weight": 1.0},
           {"text": "omachi sườn hầm", "weight": 1.0}],
 "consistency": {"projection_watermark": 929996, "is_caught_up": true, "ledger_head": 929996}}
```
**Đọc kết quả:** `weight` = độ phổ biến của cụm từ (hàng mới nên = 1.0, hàng bán chạy sẽ vài trăm — xem file
DEMO-2 để so sánh). Khối `consistency` là điểm khác biệt kỹ thuật đáng khoe: **`is_caught_up: true`** nghĩa là
dữ liệu trả về đã bắt kịp sổ cái (`projection_watermark == ledger_head`) — khách hàng biết mình đang nhìn số
mới nhất chứ không phải số cũ.

### [04] POST /v1/recommend (context=pdp) — gợi ý trên trang sản phẩm MỚI
**Ý nghĩa:** sản phẩm 0 hành vi thì lấy gì mà gợi ý? Hệ đi **bậc thang cold-start**: cùng ngành theo nội dung
→ phổ biến cùng ngành → phổ biến toàn shop (chỉ là lưới cuối).

**INPUT:** `context` (`pdp` | `similar` | `cart` | `home`) · `product_id`.
```bash
curl -s localhost:16021/v1/recommend -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"context":"pdp","product_id":"demo-mi-omachi"}' | python3 -c "import json,sys; [print(i['product_id'],'|',i['title']) for i in json.load(sys.stdin)['items'][:5]]"
```
**OUTPUT thật**
```
bh-mi-haohao        | Thùng 30 gói mì Hảo Hảo tôm chua cay
bh-nuocmam-namngu   | Nước mắm Nam Ngư 3 trong 1 750ml
bh-gao-st25         | Gạo ST25 thượng hạng túi 5kg
...
```
**Đọc kết quả:** vị trí #1 là **mì Hảo Hảo — đúng ngành mì ăn liền**, các vị trí sau là hàng tạp hoá cùng giỏ.
Trước bản vá 06/08, chỗ này trả kính cường lực / kem chống nắng / sữa bột (hàng bán chạy toàn shop) — nay đã
đi đúng ngành. `score` cao (291) là điểm phổ biến trong ngành, `source: reco_pdp`.

### [05] POST /v1/ask — hỏi tự nhiên, có chặn bịa đặt
**INPUT:** `question` (câu hỏi tiếng Việt tự do).
```bash
curl -s localhost:16021/v1/ask -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"question":"shop co ban mi an lien khong?"}' | python3 -m json.tool
```
**OUTPUT thật** (rút gọn)
```json
{"answer": "Gợi ý cho bạn:\n1. Thùng 30 gói mì Omachi sườn hầm ngũ quả 80g (145.000đ)",
 "answer_source": "template",
 "grounding_guard": {"blocked": false, "ungrounded_ids": [], "findings": {}},
 "answer_coherence": {"filtered": true,
   "dropped_ids": ["dt-banphim-akko", "mb-yem-andam", "tt-quanjean-nam-slim", "bh-dauan-neptune"]}}
```
**Đọc kết quả — đây là API đáng khoe nhất màn này:**
- `answer` — câu trả lời cho khách, **chỉ chứa hàng có thật trong kho**.
- `answer_source` — `template` (khuôn máy) hay `llm` (mô hình ngôn ngữ). Khai thật, không giấu.
- `grounding_guard.blocked` — cổng chống bịa: nếu mô hình bịa ra mã hàng không tồn tại, câu trả lời **bị chặn**
  và tự rơi về khuôn an toàn. `false` = lần này không phải chặn.
- `answer_coherence` — **bộ lọc ngành mới (vá 06/08)**: đã **loại 4 sản phẩm lệch ngành** (bàn phím, yếm ăn dặm,
  quần jean, dầu ăn) khỏi câu trả lời, chỉ giữ đúng mì. Trước bản vá, hỏi "mì" mà trả bàn phím là chuyện thường.

### [06] GET /internal/similar-products — hàng xóm gần nhất (API nội bộ)
**Ý nghĩa:** nền cho cold-start dự báo — sản phẩm mới sẽ **mượn lịch sử của hàng xóm** để đoán nhu cầu.
**INPUT:** `project_id`, `product_id`, `limit` (query string) + header `X-Internal-Token`.
```bash
curl -s "localhost:16021/internal/similar-products?project_id=demoshop&product_id=demo-mi-omachi&limit=5" -H "X-Internal-Token: $ITOK" | python3 -m json.tool
```
**OUTPUT thật**
```json
{"items": []}
```
**Đọc kết quả:** rỗng — vector ngữ nghĩa của SKU vừa tạo chưa đủ neo để tìm hàng xóm ở ngưỡng tin cậy hiện tại.
**Nói thẳng với khách:** hệ thà trả rỗng còn hơn trả bừa một hàng xóm sai, vì hàng xóm sai sẽ kéo theo dự báo
sai. (Nợ đã ghi sổ: nhánh này sẽ mạnh lên khi có tầng thuộc tính sản phẩm — xem `SOLUTION-TRI-THUC-AN-V3`.)

### [07] POST /v1/forecast:query — **HỆ TỪ CHỐI DỰ BÁO**
**INPUT:** `product_id` · `horizon_days` (số ngày muốn nhìn tới).
```bash
curl -s -w "\nstatus: %{http_code}\n" localhost:16023/v1/forecast:query -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"demo-mi-omachi","horizon_days":14}'
```
**OUTPUT thật**
```json
{"error": {"code": "NOT_FOUND", "message": "no forecast for product 'demo-mi-omachi'"}}
status: 404
```
**Đọc kết quả — điểm chốt của màn:** sản phẩm chưa bán ngày nào ⇒ **404, máy KHÔNG bịa ra con số**.
Câu nói cho khách: *"Nếu một hệ thống dự báo trả cho anh con số ngay khi sản phẩm chưa bán ngày nào, thì con
số đó là bịa. Hệ này từ chối."*

### [08] POST /v1/decisions:price-preview — **HỆ NÓI ĐÍCH DANH THIẾU GÌ**
**INPUT:** `product_id` · `candidate_price` (giá muốn thử, VND).
```bash
curl -s -w "\nstatus: %{http_code}\n" localhost:16022/v1/decisions:price-preview -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"demo-mi-omachi","candidate_price":129000}'
```
**OUTPUT thật**
```json
{"error": {"code": "FAILED_PRECONDITION", "message": "no sales in last 30d"}}
status: 412
```
**Đọc kết quả:** `412` không phải lỗi hệ thống — là **lỗi dẫn đường**: nói đúng đang thiếu *doanh số 30 ngày*.
Ba cổng dữ liệu của quyết định giá là **doanh số → giá vốn → giá hiện tại**; thiếu cổng nào nó gọi tên cổng đó.

---
# MÀN 3 — NẠP DỮ LIỆU BÁN THẬT (2 API + 1 lệnh vận hành)

### [09] POST /v1/events:backfill (forecast) — đổ 21 ngày lịch sử bán
**Ý nghĩa:** `:backfill` là đường **nạp lịch sử khi onboard khách mới** (miễn hạn ngạch, cho phép mốc thời gian
quá khứ). Khác `:ingest` là đường sự kiện phát sinh hằng ngày.

**INPUT — cấu trúc một sự kiện (bắt buộc đúng, sai tên trường là bị từ chối):**

| Trường | Ý nghĩa |
|---|---|
| `event_id` | mã sự kiện — **khoá chống trùng**, gửi lại 2 lần chỉ tính 1 |
| `schema_version` | phiên bản hợp đồng, hiện `"1.0"` |
| `event_type` | `purchase.completed` (đơn hàng) |
| `event_time` | thời điểm xảy ra (ISO-8601 UTC), được phép ở quá khứ |
| `user_pseudo_id` | mã khách ẩn danh |
| `payload.order_ref` | mã đơn hàng |
| `payload.items[]` | `{product_id, qty, unit_price}` — **tên trường đúng là `qty` và `unit_price`** |

```bash
# sinh 21 ngày đơn hàng (bán ~4 thùng/ngày thường, ~7 thùng cuối tuần) rồi nạp
python3 - <<'PY'
import json,random,subprocess
from datetime import UTC, datetime, timedelta
now=datetime.now(UTC); rng=random.Random(2026); SKU="demo-mi-omachi"; ev=[]
for d in range(21,0,-1):
    day=now-timedelta(days=d); q=max(1,int(rng.gauss(4 if day.weekday()<5 else 7,1.5)))
    ev.append({"event_id":f"{SKU}-ord-{d}","schema_version":"1.0","event_type":"purchase.completed",
      "event_time":day.strftime("%Y-%m-%dT10:30:00Z"),"user_pseudo_id":f"buyer-{d%7}",
      "payload":{"order_ref":f"{SKU}-o{d}","items":[{"product_id":SKU,"qty":q,"unit_price":145000}]}})
open("/tmp/ev.json","w").write(json.dumps({"events":ev}))
print("da sinh", len(ev), "don")
PY
curl -s localhost:16023/v1/events:backfill -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d @/tmp/ev.json | python3 -m json.tool
```
**OUTPUT thật**
```json
{"accepted": 21, "deduped": 0, "skipped": 0, "errors": [], "conflicted": 0, "ledger_position": 930017}
```
**Đọc kết quả:** `accepted` = nhận mới · `deduped` = trùng `event_id` nên bỏ (gửi lại an toàn) ·
`errors` = từng sự kiện sai kèm lý do đích danh · **`ledger_position`** = vị trí trong sổ cái —
số này chính là thứ API khác đối chiếu để biết đã "bắt kịp" chưa.

### [10] POST /v1/events:backfill (decision) — thêm giá vốn + tồn kho
**INPUT bổ sung 3 loại sự kiện** (ngoài đơn hàng ở trên):

| Loại | Payload | Dùng để |
|---|---|---|
| `cost.recorded` | `{product_id, unit_cost, qty}` | tính giá vốn bình quân (EWMA) — nền của mọi guardrail lãi/lỗ |
| `price.changed` | `{product_id, new_price}` | mốc giá hiện tại + lịch sử để ước lượng độ co giãn |
| `stock.level` | `{product_id, on_hand_qty}` | tồn kho hiện có — nền của kế hoạch nhập hàng |

```bash
curl -s localhost:16022/v1/events:backfill -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d "$(python3 - <<'PY'
import json
ev=json.load(open("/tmp/ev.json"))["events"]
ev+=[{"event_id":"demo-mi-omachi-cost-1","schema_version":"1.0","event_type":"cost.recorded","event_time":"2026-07-17T08:00:00Z","user_pseudo_id":"system","payload":{"product_id":"demo-mi-omachi","unit_cost":98000,"qty":100}},
     {"event_id":"demo-mi-omachi-price-1","schema_version":"1.0","event_type":"price.changed","event_time":"2026-07-17T08:00:00Z","user_pseudo_id":"system","payload":{"product_id":"demo-mi-omachi","new_price":145000}},
     {"event_id":"demo-mi-omachi-stock-1","schema_version":"1.0","event_type":"stock.level","event_time":"2026-08-07T00:00:00Z","user_pseudo_id":"system","payload":{"product_id":"demo-mi-omachi","on_hand_qty":40}}]
print(json.dumps({"events":ev}))
PY
)" | python3 -m json.tool
```
**OUTPUT thật**
```json
{"accepted": 23, "deduped": 1, "skipped": 0, "errors": [], "conflicted": 0, "ledger_position": 930041}
```
**Đọc kết quả:** `deduped: 1` — một sự kiện đã có từ trước nên bị bỏ đúng luật, **không nhân đôi doanh số**.
Đây là tính chất *gửi lại bao nhiêu lần cũng ra một kết quả* (idempotency) — điều bắt buộc với hệ thống tiền bạc.

### [BƯỚC VẬN HÀNH] Kích rollup (không phải API)
Sự kiện thô cần được cộng dồn thành chuỗi bán theo ngày. Vòng tự động chạy **mỗi 1 giờ** — quá chậm cho demo,
nên kích tay:
```bash
.venv/bin/python -c "
import sys; sys.path.insert(0,'scripts')
import seed_demoshop as sd; sd.run_rollups(window_days=40)"
```
**Kết quả mong đợi:** in `forecast rollup: {'products': N, 'days': M}` và `decision rollup: {...}` cho tới khi
hết sự kiện tồn. Nói với khách: *"đây là nhịp cộng sổ; ngoài đời nó tự chạy, ở đây tôi kích tay cho nhanh."*

---
# MÀN 4 — GIỜ ĐÃ CÓ TRÍ KHÔN (9 API)

### [11] POST /v1/forecast:run — chạy lại dự báo (bất đồng bộ)
**INPUT:** body rỗng `{}` (chạy cho toàn tenant).
```bash
curl -s -w "\nstatus: %{http_code}\n" -X POST localhost:16023/v1/forecast:run -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{}'
```
**OUTPUT thật**
```json
{"status": "queued", "run_id": "r_2026-08-07", "job_id": "fr-demoshop-r_2026-08-07"}
status: 202
```
**Đọc kết quả:** `202` = **đã nhận việc** (không phải đã làm xong). Huấn luyện mô hình mất vài chục giây tới
vài phút nên hệ trả ngay `job_id` để theo dõi — gọi lại khi job đang chạy sẽ trả về **cùng một job**, không
nhân đôi việc.

### [12] GET /v1/projections/status — theo dõi job tới khi xong
**INPUT:** `job_id` (query string).
```bash
curl -s "localhost:16023/v1/projections/status?job_id=fr-demoshop-r_$(date -u +%F)" -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
```
**OUTPUT thật** (khi xong)
```json
{"consumer": "forecast", "projection_watermark": 930041, "is_caught_up": true, "ledger_head": 930041,
 "job": {"job_id": "fr-demoshop-r_2026-08-07", "status": "done", "run_id": "r_2026-08-07",
         "attempt": 0, "error_code": null, "updated_at": "2026-08-07T00:19:14+00:00"}}
```
**Đọc kết quả:** `job.status` đi `queued → running → done` (hoặc `failed` kèm `error_code` — **lỗi nhìn thấy
được, không nuốt**). `attempt` = số lần thử lại. Thực đo lần này: **xong sau ~20 giây**.

### [13] POST /v1/forecast:query — **GIỜ ĐÃ CÓ SỐ**
```bash
curl -s localhost:16023/v1/forecast:query -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"demo-mi-omachi","horizon_days":14}' | python3 -m json.tool | head -30
```
**OUTPUT thật** (rút gọn)
```json
{"product_id": "demo-mi-omachi", "run_id": "r_2026-08-07",
 "daily": [{"day": "2026-08-08", "p10": 8.3,  "p50": 9.0,  "p90": 10.7},
           {"day": "2026-08-09", "p10": 9.0,  "p50": 10.3, "p90": 12.7},
           {"day": "2026-08-10", "p10": 5.0,  "p50": 5.2,  "p90": 6.8},
           {"day": "2026-08-11", "p10": 0.0,  "p50": 1.0,  "p90": 1.0},
           {"day": "2026-08-12", "p10": 4.0,  "p50": 4.0,  "p90": 4.0}, ...]}
```
**Đọc kết quả — 3 con số, 3 mục đích khác nhau (nói chậm chỗ này):**
- **`p50`** = kịch bản giữa → dùng để **lập kế hoạch** ("bình thường bán chừng này").
- **`p90`** = kịch bản cao → dùng để **nhập hàng** (chuẩn bị cho ngày đông khách, tránh cháy hàng).
- **`p10`** = kịch bản thấp → dùng để **giữ dòng tiền** (kịch bản xấu vẫn sống được).
So sánh 08-09 (Chủ nhật, p50=10.3) với 08-11 (Thứ ba, p50=1.0): **mô hình đã học được nhịp cuối tuần**
chỉ từ 21 ngày dữ liệu. Ngoài ra còn `model_used` khai đúng đang ở nấc nào của thang mô hình, và
`data_window` cho biết học trên bao nhiêu ngày.

### [14] POST /v1/scenarios:build — dựng kịch bản Monte Carlo
**Ý nghĩa:** dự báo cho từng ngày là chưa đủ — nhập hàng cần biết **tổng cầu trong cả kỳ chờ hàng**, mà tổng
của các phân vị thì không cộng được. Nên hệ sinh nhiều kịch bản tương lai rồi tính trên tập kịch bản.

**INPUT:** `product_ids[]` · `horizon_days` · `scenario_count` (số kịch bản).
```bash
curl -s localhost:16023/v1/scenarios:build -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_ids":["demo-mi-omachi"],"horizon_days":7,"scenario_count":128}' | python3 -m json.tool
```
**OUTPUT thật**
```json
{"run_id": "sc_46f4cb9b1a4d",
 "manifest": {"scenario_count": 128, "horizon_days": 7,
   "rng_algorithm": "philox", "rng_version": "1", "scenario_index_contract": "v1",
   "reference_price_mode": "FROZEN_AT_REFERENCE",
   "files": {"marginals.npz": "57c8830a…", "factors.npz": "7aa2c0a0…"},
   "sku_ids": ["demo-mi-omachi"],
   "marginals": {"demo-mi-omachi": {"marginal_source": "history_empirical",
                                    "tail": "none", "demand_class": "intermittent"}}}}
```
**Đọc kết quả — đây là chỗ khoe tính nghiêm túc kỹ thuật:**
- `rng_algorithm: philox` + `rng_version` — bộ sinh số ngẫu nhiên **có hạt giống, tái lập được**: chạy lại ra
  đúng bộ kịch bản cũ, kiểm toán được (khác hẳn "random cho vui").
- `files` kèm **mã băm SHA-256** của từng tệp kịch bản — bằng chứng chống sửa lén.
- `marginal_source: history_empirical` = phân phối lấy từ **lịch sử thật của SKU này**, không phải giả định
  hình chuông. `demand_class: intermittent` = hệ tự nhận đây là hàng bán lai rai (có ngày 0).

### [15] POST /v1/decisions:run — chạy bộ não quyết định
**INPUT:** `{}`.
```bash
curl -s -X POST localhost:16022/v1/decisions:run -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{}' | python3 -m json.tool
```
**OUTPUT thật**
```json
{"created": 2, "skipped_dedup": 155,
 "skipped_by_reason": {"anti_oscillation": 129, "plan_conflict": 95,
                       "insufficient_history": 2, "no_stock": 2, "no_cost": 33},
 "superseded_plan": 1, "price_hold": 1, "anti_osc_hold": 1}
```
**Đọc kết quả — mỗi con số là một lời hứa với chủ shop:**

| Trường | Nghĩa |
|---|---|
| `created` | số lời khuyên **mới** sinh ra lần này |
| `skipped_dedup` | đã có lời khuyên y hệt đang mở → không spam lại |
| `anti_oscillation` | **chặn đổi giá liên tục** — SKU vừa đổi giá gần đây thì khoá lại, không cho giật lên giật xuống |
| `plan_conflict` | cùng SKU đã có hành động giá khác trong kế hoạch → tránh 2 lệnh mâu thuẫn cùng ngày |
| `insufficient_history` | chưa đủ lịch sử để nói chắc |
| `no_stock` / `no_cost` | thiếu tồn kho / thiếu giá vốn → **không khuyên bừa** |
| `price_hold` | số quyết định "GIỮ NGUYÊN GIÁ" — máy nói rõ vì sao im lặng, thay vì im lặng khó hiểu |

### [16] GET /v1/decisions — danh sách lời khuyên
**INPUT:** `page_size`, `kind` (lọc theo loại), `status`. ⚠ **Không có tham số `product_id`** — muốn lọc theo
SKU thì lọc phía client.
```bash
curl -s "localhost:16022/v1/decisions?page_size=50" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | python3 -c "
import json,sys
for x in json.load(sys.stdin)['items']:
    if 'demo-mi-omachi' in json.dumps(x): print(x['decision_id'],'|',x['kind'],'|',x['status'])"
```
**OUTPUT thật:** 2 lời khuyên liên quan SKU mới. Mỗi lời khuyên gồm:

| Trường | Nghĩa |
|---|---|
| `decision_id` | mã quyết định — dùng để phản hồi ở bước cuối |
| `kind` | loại: `price_suggestion`, `price_hold`, `replenishment`, `bundle_suggestion`, `slow_mover_alert`… |
| `action` + `action_params` | hành động cụ thể và tham số (giá đề xuất, số lượng nhập…) |
| `expected_value` | **lợi ích kỳ vọng bằng tiền/tháng** — cơ sở để xếp ưu tiên |
| `confidence` | độ tin (0.9 / 0.7 / 0.5 theo chất lượng bằng chứng) |
| `guardrails` | các chốt an toàn đã kiểm và kết quả |
| `trace` | **toàn bộ phép tính viết ra bằng chữ** — chủ shop tự kiểm được, không phải hộp đen |

### [17] POST /v1/decisions:price-preview — **GIỜ TRẢ LỜI ĐƯỢC** (trước đó 412)
```bash
curl -s localhost:16022/v1/decisions:price-preview -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"demo-mi-omachi","candidate_price":129000}' | python3 -m json.tool
```
**OUTPUT thật**
```json
{"current":   {"price": 145000.0, "est_units_30d": 89.0,  "est_profit_30d": 4183000.0},
 "candidate": {"price": 129000,   "est_units_30d": 95.17, "est_profit_30d": 2950372.3},
 "delta_profit_30d": -1232627.68,
 "elasticity_used": {"eps": -0.5736, "method": "pooled_prior", "n_points": 21, "r2": null},
 "guardrails": [{"code": "BELOW_COST", "status": "PASS"}],
 "confidence": 0.7,
 "explanation": "Q(P)=Q0·(P/P0)^eps; profit=(P-c)·Q"}
```
**Đọc kết quả — giải thích từng dòng cho khách:**
- `current` vs `candidate`: giảm giá từ 145.000 → 129.000 thì **bán thêm** (89 → 95 thùng/tháng)…
- …nhưng `delta_profit_30d = **−1.232.627đ/tháng**` ⇒ **lãi GIẢM**. Máy can bằng số, chặn trực giác
  "giảm giá cho chạy hàng".
- `elasticity_used.method = **pooled_prior**` — **khai thật là đang mượn** độ co giãn trung bình của shop,
  vì SKU mới chỉ có 21 điểm dữ liệu, chưa đủ ước lượng riêng. `r2: null` = chưa có độ khớp riêng.
- `confidence: 0.7` (không phải 0.9) — **tự hạ điểm tin cậy vì bằng chứng yếu hơn**. So sánh với file DEMO-2:
  cùng API đó trên hàng bán lâu trả `method: ols_daily, n_points: 119, r2: 0.447, confidence: 0.9`.
- `explanation` — công thức lộ thiên để tự kiểm: cầu theo hàm luỹ thừa, lãi = (giá − vốn) × lượng.

### [18] POST /v1/decisions:price-preview (giá dưới vốn) — guardrail phải chặn
```bash
curl -s localhost:16022/v1/decisions:price-preview -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"demo-mi-omachi","candidate_price":80000}' | python3 -c "import json,sys; d=json.load(sys.stdin); print('guardrails:', d['guardrails']); print('delta_profit_30d:', round(d['delta_profit_30d']))"
```
**OUTPUT thật**
```
guardrails: [{'code': 'BELOW_COST', 'status': 'FAIL'}]
delta_profit_30d: -6436223
```
**Đọc kết quả:** giá thử 80.000đ trong khi **giá vốn 98.000đ** ⇒ guardrail `BELOW_COST` trả **FAIL** và lãi
tháng âm 6,4 triệu. Đây là chốt an toàn vừa được sửa ngày 06/08 (trước đó cả hai nhánh đều báo PASS — bug do
chính buổi tập của chúng tôi tìm ra và đã có test hồi quy khoá lại).

### [19] GET /v1/decisions:replenish-plan — nhập bao nhiêu, khi nào
**INPUT:** `product_id` (query, bỏ trống = cả shop).
```bash
curl -s "localhost:16022/v1/decisions:replenish-plan?product_id=demo-mi-omachi" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
```
**OUTPUT thật**
```json
{"items": [{"product_id": "demo-mi-omachi",
  "avg_daily_units": 2.967, "sigma_daily": 2.773,
  "lead_time_days": 7.0, "lead_time_std": 2.0,
  "service_level": 0.9, "z": 1.28,
  "safety_stock": 12.08, "reorder_point": 32.84,
  "on_hand": 40.0, "days_of_inventory": 13.5,
  "below_reorder_point": false,
  "moq": 0.0, "pack_size": 1.0, "order_qty_moq_pack": 0,
  "formula": "ROP = avg_daily*LT + z*sqrt(LT*sigma_d^2 + avg_d^2*sigma_LT^2); DOI = on_hand/avg_daily"}],
 "n": 1, "window_days": 30}
```
**Đọc kết quả — dịch sang lời chủ shop:**
- Bán trung bình **2,97 thùng/ngày**, dao động ±2,77 (`sigma_daily`).
- Đặt hàng về mất **7 ngày** (`lead_time_days`), độ trễ này cũng dao động ±2 ngày.
- Muốn **90% không cháy hàng** (`service_level`) ⇒ hệ số an toàn `z = 1.28` ⇒ cần trữ thêm
  **12,08 thùng dự phòng** (`safety_stock`).
- ⇒ **Điểm đặt hàng lại = 32,84 thùng.** Đang còn 40 ⇒ `below_reorder_point: false` = **chưa cần đặt**,
  còn đủ bán **13,5 ngày** (`days_of_inventory`).
- `formula` in ra ngay trong kết quả — chủ shop hoặc kế toán tự kiểm được từng bước.

### [20] POST /v1/decisions/{id}:feedback — chủ shop phán, khép vòng
**Ý nghĩa:** đây là **vòng phản hồi** — lời khuyên phải được chấm điểm bằng kết quả thật, không phải khuyên xong bỏ đó.
**INPUT:** `action` (`accepted` | `rejected` | `deferred`) · `note` (lý do, tuỳ chọn).
```bash
DID=<decision_id lấy ở bước 16>
curl -s -X POST "localhost:16022/v1/decisions/$DID:feedback" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"action":"accepted","note":"demo doi tac"}' | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['decision_id'], '->', d.get('status'))"
```
**OUTPUT thật:** trả về nguyên quyết định với trạng thái đã cập nhật. Sau 30 ngày, hệ đối chiếu **lãi thực tế**
với `expected_value` đã hứa để tự chấm điểm mình (sổ kết quả — `outcome ledger`).

---
# DỌN SÂN (bắt buộc, để lần demo sau còn nguyên kịch bản)
```bash
curl -s -X DELETE "localhost:16021/v1/products/demo-mi-omachi" -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -w "status: %{http_code}\n"
```
**OUTPUT thật:** `status: 204` (xoá xong, không có nội dung trả về).
> Nếu muốn xoá cả dữ liệu bán đã nạp (để lần sau lại thấy 404/412 như màn 2):
> ```bash
> for db in miniai_forecast miniai_decision; do docker exec miniai-postgres psql -U miniai -d $db -tAc \
>  "DELETE FROM demand_daily WHERE product_id='demo-mi-omachi'; DELETE FROM sales_daily WHERE product_id='demo-mi-omachi';" 2>/dev/null; done
> ```

---
# BẢNG TỔNG KẾT MÀN DEMO (chiếu lên màn hình lúc chốt)

| Câu hỏi | Trước khi có dữ liệu | Sau 21 ngày dữ liệu |
|---|---|---|
| Tìm thấy hàng không? | ✅ sau 9 giây | ✅ |
| Gợi ý hàng liên quan? | ✅ đúng ngành (bậc thang cold-start) | ✅ |
| Dự báo nhu cầu? | ❌ **404 — không bịa số** | ✅ p10/p50/p90 có nhịp cuối tuần |
| Khuyên giá? | ❌ **412 — nói rõ thiếu doanh số** | ✅ kèm cảnh báo lãi giảm 1,23 triệu/tháng |
| Chặn bán dưới vốn? | — | ✅ `BELOW_COST: FAIL` |
| Nhập hàng bao nhiêu? | ❌ chưa đủ cơ sở | ✅ ROP 32,8 — còn 40, đủ bán 13,5 ngày |
| Độ tin cậy khai báo? | — | ✅ `pooled_prior`, `confidence 0.7` — **tự khai đang mượn** |

**Câu chốt:** *"Mọi câu trả lời của hệ đều tự khai nó dựa trên cái gì và tin tới đâu. Khi chưa đủ cơ sở, nó nói
thiếu gì chứ không đoán. Đó là thứ anh chị cần ở một hệ thống ra quyết định về tiền."*
