# DEMO 1 — SẢN PHẨM MỚI TINH: từ 0 dữ liệu đến ra quyết định
> Kịch bản 20 API, chạy trọn 1 vòng: **smart search → recommend → forecast → decision → phản hồi**.
> **Số liệu đo lại toàn bộ ngày 2026-08-12** trên demoshop (bản trước đo 07/08 đã lệch — xem mục
> "ĐÃ ĐỔI SO VỚI BẢN 07/08" ở cuối file).
>
> ⭐ **MỖI API ĐỀU CÓ 4 BƯỚC:** ① **ĐO TRƯỚC** (chụp trạng thái) → ② **GỌI API** → ③ **ĐO SAU**
> (chứng minh dữ liệu đã đổi) → ④ **LUỒNG** (dữ liệu chảy qua bảng nào, job nào).
> Đây là thứ biến buổi demo từ *"tin tôi đi"* thành *"anh chị tự nhìn số"*.

---
## ⛔⛔ LUẬT NGHIỆM THU (human chốt 2026-08-12) — ĐỌC TRƯỚC KHI NÓI "XONG"
**Hai kịch bản này chỉ được coi là HOÀN THIỆN khi chạy end-to-end đủ 4 LƯỢT LIÊN TIẾP không ra lỗi,
trên CÙNG MỘT BẢN CODE.** Deploy bất kỳ thay đổi nào ở giữa ⇒ 4 lượt trước đó mất giá trị, **đếm lại từ 1**.

*Vì sao:* một lượt không đủ để lộ lỗi phụ thuộc trạng thái. Đo 12/08: lượt 1 xanh, **lượt 2 mới lộ** việc
`reset1` xoá sản phẩm là xoá luôn vector ⇒ `[06]` rỗng ⇒ `[07]` ra 404 thay vì `cold_start_analog`.
Lượt 1 chỉ xanh vì vector còn sót từ lần chạy trước.

**Log mỗi lượt lưu ở `icpp/demo-e2e-runs/`** (ngoài repo) — để mất ngữ cảnh phiên vẫn còn bằng chứng.

## THÔNG ĐIỆP BÁN HÀNG CỦA MÀN NÀY
Hàng mới lên kệ **tìm được sau vài giây**. Khi chưa có lịch sử bán, hệ **không bịa số** — nó nói rõ đang
**mượn lịch sử của 5 mặt hàng tương tự nào**, và **từ chối thẳng** việc khuyên giá vì thiếu doanh số. Sau khi
nạp 21 ngày dữ liệu thật, cũng những API đó trả lời đầy đủ và **tự khai đang dùng phương pháp nào, tin tới đâu**.

---
## CHUẨN BỊ (chạy 1 lần, trước khi khách vào phòng)

```bash
cd /home/hai-soft/projects/icpp/mecom/project
SKEY=$(.venv/bin/python -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['search'])")
DKEY=$(.venv/bin/python -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['decision'])")
FKEY=$(.venv/bin/python -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['forecast'])")
ITOK=$(docker exec miniai-smartsearch printenv MINIAI_INTERNAL_TOKEN)
SKU=demo-mi-omachi
echo "keys ${SKEY:0:6}/${DKEY:0:6}/${FKEY:0:6} | internal ${ITOK:0:4}"
```

### ⭐ BỘ ĐO — dán 1 lần, dùng cho cả buổi
```bash
# chạy một câu SQL trên 1 trong 4 kho: miniai_search | miniai_decision | miniai_forecast | miniai_ledger
# (kho thứ 4 = sổ cái chung, `reset1` có dùng — xem giải thích `conflicted` ở [09])
q(){ docker exec miniai-postgres psql -U miniai -d "$1" -tAc "$2"; }

# ⭐ RESET — gõ MỘT CHỮ là sân sạch, chạy lại kịch bản từ đầu bao nhiêu lần cũng được.
# (Đã chạy kiểm chứng 12/08: mọi bảng về 0, KỂ CẢ sổ cái chung — xem giải thích `conflicted` ở [09].)
reset1(){ P=demo-mi-omachi
  curl -s -X DELETE "localhost:16021/v1/products/$P" -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -o /dev/null
  for db in miniai_forecast miniai_decision miniai_search; do docker exec miniai-postgres psql -U miniai -d $db -tAc "DO \$\$ DECLARE t text; BEGIN FOR t IN SELECT table_name FROM information_schema.columns WHERE table_schema='public' AND column_name='product_id' LOOP EXECUTE format('DELETE FROM %I WHERE product_id=%L', t, '$P'); END LOOP; END \$\$;" >/dev/null 2>&1; done
  q miniai_decision "DELETE FROM feedback  WHERE project_id='demoshop' AND decision_id IN (SELECT decision_id FROM decisions WHERE subject_id='$P');" >/dev/null 2>&1
  q miniai_decision "DELETE FROM decisions WHERE project_id='demoshop' AND (subject_id='$P' OR trace LIKE '%$P%');" >/dev/null
  for db in miniai_forecast miniai_decision; do q $db "DELETE FROM raw_events WHERE project_id='demoshop' AND payload::text LIKE '%$P%';" >/dev/null; done
  q miniai_search "DELETE FROM query_log     WHERE project_id='demoshop' AND query_norm LIKE '%omachi%';" >/dev/null
  q miniai_search "DELETE FROM suggest_terms WHERE project_id='demoshop' AND term       LIKE '%omachi%';" >/dev/null
  q miniai_ledger "DELETE FROM event_ledger  WHERE project_id='demoshop' AND payload::text LIKE '%$P%';" >/dev/null
  rm -f /tmp/ev.json /tmp/fq.json
  echo "  RESET xong: products=$(q miniai_search "SELECT count(*) FROM products WHERE project_id='demoshop' AND product_id='$P'") demand=$(q miniai_forecast "SELECT count(*) FROM demand_daily WHERE project_id='demoshop' AND product_id='$P'") forecasts=$(q miniai_forecast "SELECT count(*) FROM forecasts WHERE project_id='demoshop' AND product_id='$P'") sales=$(q miniai_decision "SELECT count(*) FROM sales_daily WHERE project_id='demoshop' AND product_id='$P'") decisions=$(q miniai_decision "SELECT count(*) FROM decisions WHERE project_id='demoshop' AND subject_id='$P'") ledger=$(q miniai_ledger "SELECT count(*) FROM event_ledger WHERE project_id='demoshop' AND payload::text LIKE '%$P%'") query_log=$(q miniai_search "SELECT count(*) FROM query_log WHERE project_id='demoshop' AND query_norm LIKE '%omachi%'") suggest=$(q miniai_search "SELECT count(*) FROM suggest_terms WHERE project_id='demoshop' AND term LIKE '%omachi%'")"
}

# ảnh chụp toàn cảnh 1 SKU — gõ trước và sau mỗi API để thấy cái gì đã đổi
snap(){ P="${1:-$SKU}"
 echo "── ẢNH CHỤP [$P] lúc $(date +%H:%M:%S)"
 echo "  search  : products=$(q miniai_search "SELECT count(*) FROM products WHERE project_id='demoshop' AND product_id='$P'")  outbox_đang_chờ=$(q miniai_search "SELECT count(*) FROM catalog_outbox WHERE project_id='demoshop'")  reco_exposure=$(q miniai_search "SELECT count(*) FROM reco_exposure WHERE project_id='demoshop'")"
 echo "  forecast: raw_events=$(q miniai_forecast "SELECT count(*) FROM raw_events WHERE project_id='demoshop'")  demand_daily=$(q miniai_forecast "SELECT count(*) FROM demand_daily WHERE project_id='demoshop' AND product_id='$P'")  forecasts=$(q miniai_forecast "SELECT count(*) FROM forecasts WHERE project_id='demoshop' AND product_id='$P'")"
 echo "  decision: raw_events=$(q miniai_decision "SELECT count(*) FROM raw_events WHERE project_id='demoshop'")  sales_daily=$(q miniai_decision "SELECT count(*) FROM sales_daily WHERE project_id='demoshop' AND product_id='$P'")  decisions=$(q miniai_decision "SELECT count(*) FROM decisions WHERE project_id='demoshop'")  tồn=$(q miniai_decision "SELECT coalesce(max(on_hand_qty)::text,'—') FROM stock_state WHERE project_id='demoshop' AND product_id='$P'")  vốn=$(q miniai_decision "SELECT coalesce(round(max(ewma_cost))::text,'—') FROM cost_state WHERE project_id='demoshop' AND product_id='$P'")"
}
```

### ⛔ CỔNG BẮT BUỘC — kiểm sân sạch trước khi khách vào
Cả kịch bản dựa trên việc SKU demo **chưa tồn tại**. Lần demo trước quên dọn ⇒ màn 2 sẽ ra số thay vì ra
"chưa có dữ liệu" và **hỏng toàn bộ mạch kể**. (Đo 12/08: đúng lỗi này đã xảy ra — còn tồn 27 ngày bán,
98 dòng dự báo.)
```bash
snap $SKU          # PHẢI thấy: products=0 · demand_daily=0 · forecasts=0 · sales_daily=0 · tồn=— · vốn=—
```
**Nếu chưa sạch → gõ đúng một chữ:**
```bash
reset1
```
> 💡 `reset1` chạy được **bao nhiêu lần cũng được** — cứ thực hành xong lại gõ, sân về đúng vạch xuất phát.

**Header cho MỌI API nghiệp vụ:** `Authorization: Bearer <key đúng service>` + `X-Project-Id: demoshop`.
API nội bộ (`/internal/...`) dùng `X-Internal-Token`.
Cổng: smartsearch **16021** · decision **16022** · forecast **16023**.

---
# MÀN 1 — KHAI SINH SẢN PHẨM (1 API)

## [01] POST /v1/products:upsert — đưa hàng mới lên kệ
**Ý nghĩa:** cửa duy nhất để sản phẩm vào hệ. Ghi Postgres **ngay**, rồi đẩy sang Vespa **qua hàng đợi**
(nhất quán cuối). Đây là API cho thấy rõ nhất "dữ liệu chảy đi đâu".

### ① ĐO TRƯỚC
```bash
echo "products = $(q miniai_search "SELECT count(*) FROM products WHERE project_id='demoshop' AND product_id='$SKU'")"
echo "outbox   = $(q miniai_search "SELECT count(*) FROM catalog_outbox WHERE project_id='demoshop'")"
```
**Đo thật 12/08:** `products = 0` · `outbox = 0`

### ② GỌI API
| Trường | Bắt buộc | Ý nghĩa |
|---|---|---|
| `id` | ✔ | mã SKU, khoá chính trong tenant |
| `title` | ✔ | tên hiển thị — **nguồn chính cho tìm kiếm** (BM25 + vector) |
| `categories` | ✔ | `"Cha > Con"`; phần trước `>` thành `category_l1` để lọc/gộp |
| `price_info.price` | ✔ | giá bán (VND, số nguyên) |
| `availability` | ✔ | `IN_STOCK`/`OUT_OF_STOCK` — hết hàng sẽ **biến mất khỏi kết quả tìm** |
| `publish_time` | ✔ | thời điểm lên kệ (ISO-8601 UTC) |

```bash
curl -s localhost:16021/v1/products:upsert -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"products":[{"id":"demo-mi-omachi","title":"Thùng 30 gói mì Omachi sườn hầm ngũ quả 80g","description":"Mì ăn liền Omachi sợi khoai tây, vị sườn hầm ngũ quả, thùng 30 gói 80g","categories":["Bách hóa > Mì ăn liền"],"brands":["Omachi"],"price_info":{"currency_code":"VND","price":145000,"original_price":165000},"availability":"IN_STOCK","available_quantity":40,"attributes":{},"images":[],"publish_time":"2026-08-13T00:00:00Z"}]}'
```
**OUTPUT thật:** `{"upserted":1,"queued_for_index":1}`

### ③ ĐO SAU — **gõ NGAY, rồi gõ lại sau 10 giây**
```bash
echo "NGAY SAU : products=$(q miniai_search "SELECT count(*) FROM products WHERE project_id='demoshop' AND product_id='$SKU'")  outbox=$(q miniai_search "SELECT count(*) FROM catalog_outbox WHERE project_id='demoshop'")"
sleep 10
echo "SAU 10s  : products=$(q miniai_search "SELECT count(*) FROM products WHERE project_id='demoshop' AND product_id='$SKU'")  outbox=$(q miniai_search "SELECT count(*) FROM catalog_outbox WHERE project_id='demoshop'")"
```
**Đo thật 12/08 — đây là khoảnh khắc đắt nhất của màn này:**
```
TRƯỚC    : products=0  outbox=0
NGAY SAU : products=1  outbox=1     ← BẮT ĐƯỢC HÀNG ĐỢI
SAU 10s  : products=1  outbox=0     ← indexer đã rút hàng đợi
```

### ④ LUỒNG DỮ LIỆU
```
API :upsert ──ghi ngay──► Postgres products      (bền, không mất)
            └─xếp hàng──► catalog_outbox ──indexer──► Vespa (tìm kiếm)
```
**Nói với khách:** *"Hai con số `upserted` và `queued_for_index` tách nhau có chủ đích. Ghi bền và đánh chỉ mục
là hai việc khác nhau — Vespa chết thì dữ liệu vẫn còn trong hàng đợi và tự bù khi sống lại. Anh chị vừa nhìn
thấy hàng đợi lên 1 rồi về 0."*

### ⏱ CỔNG CHỜ BẮT BUỘC trước khi sang màn 2
```bash
until curl -s localhost:16021/v1/ask -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"question":"co mi omachi khong?"}' | grep -q demo-mi-omachi; do echo "dang danh chi muc..."; sleep 3; done; echo "== DA SAN SANG =="
```
> Không đếm nhẩm — đo bằng máy. Hỏi `/v1/ask` quá sớm sẽ ra hàng lung tung.

### ⛔⛔ CỔNG THỨ HAI — KÍCH VECTOR (🆕 12/08, **bỏ qua là hỏng màn 2**)
Chỉ mục chữ (BM25) có ngay, nhưng **vector ngữ nghĩa** do job `embed_backfill` sinh, chạy mỗi **300 giây**.
Chưa có vector thì `[06] similar-products` trả **rỗng** ⇒ `[07]` ra **404** thay vì `cold_start_analog` —
đúng mắt xích đắt nhất của màn 2. (Đo 12/08: chạy lại kịch bản lần 2 vấp đúng chỗ này, vì `reset1` xoá sản
phẩm là xoá luôn vector.)

```bash
docker exec miniai-smartsearch python3 -c "
import asyncio, asyncpg, os
from app.jobs.embed_backfill import run_embed_backfill_once
from app.core.embedding import BgeM3Embedder
from app.store.vespa_client import VespaClient
async def m():
    p = await asyncpg.create_pool(os.environ.get('SEARCH_DSN') or os.environ.get('DATABASE_URL'), min_size=1, max_size=2)
    v = VespaClient(os.environ.get('VESPA_URL','http://vespa:8080'))
    print('embed job:', await run_embed_backfill_once(p, v, BgeM3Embedder()))
    await p.close()
asyncio.run(m())"

# đo bằng máy: phải thấy 5 hàng xóm, KHÔNG được rỗng
until curl -s "localhost:16021/internal/similar-products?project_id=demoshop&product_id=demo-mi-omachi&k=5" -H "X-Internal-Token: $ITOK" | grep -q product_id; do echo "dang sinh vector..."; sleep 3; done; echo "== VECTOR DA SAN SANG =="
```
**Đo thật 12/08:** `embed job: {'embedded': 1}` → similar-products trả 5 SKU.
*"Đây cũng là một nhịp job nền — ngoài đời tự chạy mỗi 5 phút, ở đây tôi kích tay cho nhanh."*

---
# MÀN 2 — TÌM ĐƯỢC NGAY, NHƯNG CHƯA CÓ TRÍ KHÔN (6 API)

## [02] POST /v1/search — hàng mới đã tìm thấy
**Ý nghĩa:** tìm kiếm lai — BM25 (khớp chữ) + vector ngữ nghĩa, trộn bằng RRF.

### ① ĐO TRƯỚC
```bash
q miniai_search "SELECT coalesce(sum(cnt),0) AS lan_tim FROM query_log WHERE project_id='demoshop' AND query_norm='omachi';"
```
**Đo thật:** `0` (đã dọn sân)

### ② GỌI API
```bash
curl -s localhost:16021/v1/search -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"query":"omachi","page_size":5}' | .venv/bin/python -c "import json,sys; [print(round(i['score'],4),'|',i['product_id'],'|',i.get('source')) for i in json.load(sys.stdin)['items']]"
```
**OUTPUT thật 12/08**
```
10.2236 | demo-mi-omachi | vespa_bm25
```

### ③ ĐO SAU
```bash
sleep 2; q miniai_search "SELECT cnt AS lan_tim, results_count_last AS so_kq FROM query_log WHERE project_id='demoshop' AND query_norm='omachi';"
```
**Đo thật:** `1 | 1` — **truy vấn của khách đã được ghi sổ**.

### ④ LUỒNG
```
/v1/search ──► Vespa (BM25 + vector) ──► RRF trộn điểm ──► trả kết quả
           └──► ghi query_log (cnt++)  ──► nuôi gợi ý gõ phím + học xếp hạng
```
**Nói với khách:** *"Sản phẩm tạo chưa tới 1 phút đã tìm được. Và mỗi lần khách tìm, hệ ghi lại — đó là nguyên
liệu để ngày mai gợi ý thông minh hơn."*

> ⚠ **Truy vấn nên dùng:** `omachi` (tên thương hiệu). **Tránh** `mi omachi` / `mi an lien`: bỏ dấu, "mi"
> 2 ký tự khớp cả "sơ **mi**" nên hàng mới bị đẩy xuống. Điểm yếu đã đo, đã ghi sổ (`W-SEARCH-CONCEPT-NEGATION`).

---
## [03] GET /v1/suggest — gợi ý gõ phím đã có từ mới

### ① ĐO TRƯỚC
```bash
q miniai_search "SELECT count(*) AS so_cum_tu FROM suggest_terms WHERE project_id='demoshop' AND term LIKE '%omachi%';"
```
**Đo thật ngay sau [01]:** `0` — **cụm từ CHƯA có**.

> ⛔ **BẮT BUỘC KÍCH JOB, nếu không bước này RỖNG trước mặt khách.**
> Job `suggest_terms` chạy mỗi **3.600 giây (1 giờ)** — quá chậm cho demo:
> ```bash
> docker exec miniai-smartsearch python3 -c "
> import asyncio, asyncpg, os
> from app.jobs.suggest_terms import run_suggest_terms_once
> async def m():
>     p=await asyncpg.create_pool(os.environ.get('SEARCH_DSN') or os.environ.get('DATABASE_URL'),min_size=1,max_size=2)
>     print('suggest job:', await run_suggest_terms_once(p)); await p.close()
> asyncio.run(m())"
> q miniai_search "SELECT count(*) AS so_cum_tu FROM suggest_terms WHERE project_id='demoshop' AND term LIKE '%omachi%';"
> ```
> **Đo thật sau khi kích (12/08):** bảng có **6 dòng** (mỗi cụm có thêm bản không dấu), **API trả 3 cụm**.
> *"Đây là nhịp cộng sổ của gợi ý gõ phím — ngoài đời chạy mỗi giờ, ở đây tôi kích tay cho nhanh."*

### ② GỌI API
```bash
curl -s "localhost:16021/v1/suggest?q=omachi&limit=5" -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" | .venv/bin/python -m json.tool
```
**OUTPUT thật 12/08**
```json
{"items": [{"text": "omachi", "weight": 1.0},
           {"text": "omachi sườn", "weight": 1.0},
           {"text": "omachi sườn hầm", "weight": 1.0}],
 "consistency": {"projection_watermark": 1079633, "data_as_of": "2026-08-12T08:55:04Z",
                 "is_caught_up": true, "ledger_head": 1079633}}
```

### ③ ĐO SAU — đối chiếu số của API với số trong kho
```bash
q miniai_search "SELECT term, round(weight::numeric,2) FROM suggest_terms WHERE project_id='demoshop' AND term LIKE '%omachi%' ORDER BY weight DESC;"
```
**Đo thật:** 3 dòng, `weight = 1.0` — **khớp đúng con số API vừa trả**.

> 🆕 **Đã vá 13/08 — bản cũ viết `round(weight,2)` và lệnh NỔ TRƯỚC MẶT KHÁCH.**
> Cột `suggest_terms.weight` kiểu `double precision`, mà Postgres chỉ có `round(numeric, int)` —
> không có bản 2 tham số cho `double`. Lỗi tái lập 100%:
> ```
> ERROR:  function round(double precision, integer) does not exist
> ```
> Phải ép `::numeric` trước. Cùng lỗi này còn ở **DEMO-2 bước [03] và [04]** — đã vá cả hai.

### ④ LUỒNG
```
products.title ──job suggest_terms (mỗi 1 GIỜ)──► suggest_terms ──► /v1/suggest
query_log.cnt  ─────────────────────────────────┘  (độ phổ biến nâng weight lên)
```
**Điểm khoe:** `is_caught_up: true` nghĩa là **dữ liệu trả về đã bắt kịp sổ cái**
(`projection_watermark == ledger_head`) — khách biết mình đang nhìn số mới nhất.
`weight = 1.0` = hàng mới chưa ai tìm. **So với DEMO-2: cụm "mì hảo hảo" có weight ~400** — đó là giá trị
của dữ liệu tích luỹ, nhìn thấy bằng một con số.

> ⚠ **`weight` là số ĐANG LỚN DẦN, đừng đọc thuộc.** Đo được: `334.8` (07/08) → `376.96` (12/08) →
> **`401.28` (13/08)**. Cứ đọc từ màn hình. Con số bên hàng mới thì luôn là `1.0` — đó mới là điểm so sánh.

---
## [04] POST /v1/recommend (context=pdp) — gợi ý trên trang sản phẩm MỚI
**Ý nghĩa:** sản phẩm 0 hành vi thì lấy gì gợi ý? Hệ đi **bậc thang cold-start**: cùng ngành theo nội dung →
phổ biến cùng ngành → phổ biến toàn shop.

### ① ĐO TRƯỚC
```bash
q miniai_search "SELECT count(*) AS luot_hien_thi FROM reco_exposure WHERE project_id='demoshop';"
```
**Đo thật:** `1562`

### ② GỌI API
```bash
curl -s localhost:16021/v1/recommend -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"context":"pdp","product_id":"demo-mi-omachi"}' | .venv/bin/python -c "import json,sys; d=json.load(sys.stdin); [print(round(i['score'],2),'|',i['product_id'],'|',i['title'][:40]) for i in d['items'][:3]]; print('source:', d['items'][0].get('source'))"
```
**OUTPUT thật 12/08**
```
0.30 | bh-mi-haohao     | Thùng 30 gói mì Hảo Hảo tôm chua cay
0.30 | bh-hatdieu-500g  | Hạt điều rang muối Bình Phước 500g
0.30 | bh-giay-vesinh   | Giấy vệ sinh Bless You 3 lớp (10 cuộn)
source: reco_pdp
```

### ③ ĐO SAU
```bash
sleep 2; q miniai_search "SELECT count(*) AS luot_hien_thi FROM reco_exposure WHERE project_id='demoshop';"
q miniai_search "SELECT product_id, position FROM reco_exposure WHERE project_id='demoshop' ORDER BY ts DESC LIMIT 3;"
```
**Đo thật:** `1562 → 1574` (**+12 dòng**, đúng bằng số sản phẩm vừa gợi ý), 3 dòng mới nhất là
`bh-mi-haohao|1`, `bh-hatdieu-500g|2`, `bh-giay-vesinh|3`.

### ④ LUỒNG
```
/v1/recommend ──► bậc thang cold-start ──► trả gợi ý
              └──► ghi reco_exposure (1 dòng/sản phẩm, kèm VỊ TRÍ)
                          └──► ghép với click_log ──► khử thiên lệch vị trí khi học xếp hạng
```
**Nói với khách:** *"Vị trí #1 là mì Hảo Hảo — **đúng ngành mì ăn liền**. Và hệ ghi lại chính xác nó đã hiện
cái gì ở vị trí nào; không có con số này thì mọi phép học sau đó đều thiên lệch."*

---
## [05] POST /v1/ask — hỏi tự nhiên, có chặn bịa đặt

### ① ĐO TRƯỚC
```bash
q miniai_search "SELECT count(*) FROM query_log WHERE project_id='demoshop' AND query_norm LIKE '%mi an lien%';"
```

### ② GỌI API
```bash
curl -s localhost:16021/v1/ask -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"question":"shop co ban mi an lien khong?"}' | .venv/bin/python -m json.tool
```
**OUTPUT thật 12/08** (rút gọn)
```json
{"answer": "Gợi ý cho bạn:\n1. Thùng 30 gói mì Omachi sườn hầm ngũ quả 80g (145.000đ)",
 "answer_source": "template",
 "grounding_guard": {"blocked": false, "ungrounded_ids": [], "findings": {}},
 "answer_coherence": {"filtered": true,
   "dropped_ids": ["dt-banphim-akko", "mb-yem-andam", "tt-quanjean-nam-slim", "bh-dauan-neptune"]}}
```

### ③ ĐO SAU — chứng minh 4 sản phẩm bị loại là có thật
```bash
q miniai_search "SELECT product_id, category_l1 FROM products WHERE project_id='demoshop' AND product_id IN ('dt-banphim-akko','mb-yem-andam','tt-quanjean-nam-slim','bh-dauan-neptune');"
sleep 2; q miniai_search "SELECT query_norm, cnt FROM query_log WHERE project_id='demoshop' AND query_norm LIKE '%mi an lien%';"
```
**Đo thật:** 4 SKU đó thuộc **4 ngành khác nhau** (Điện tử, Mẹ & bé, Thời trang, Bách hóa-dầu ăn) — bộ lọc
đã loại đúng hàng lệch ngành, chỉ giữ mì. Câu hỏi cũng được ghi vào `query_log`.

### ④ LUỒNG — 3 tầng bảo vệ
```
câu hỏi ──► retrieval (tìm ứng viên)
        ──► grounding_guard : mã hàng không có thật ⇒ CHẶN, rơi về khuôn an toàn
        ──► answer_coherence: sản phẩm lệch ngành  ⇒ LOẠI khỏi câu trả lời
        ──► answer + tự khai answer_source (template | llm)
```
**Nói với khách:** *"Câu trả lời chỉ chứa hàng có thật trong kho, và hệ **tự khai** nó dùng khuôn máy hay mô
hình ngôn ngữ. Trước bản vá 06/08, hỏi 'mì' mà trả bàn phím là chuyện thường."*

---
## [06] GET /internal/similar-products — hàng xóm gần nhất (API nội bộ)
**Ý nghĩa:** nền cho cold-start dự báo — SKU mới sẽ **mượn lịch sử của hàng xóm**.

### ① ĐO TRƯỚC
```bash
q miniai_search "SELECT count(*) AS co_vector FROM products WHERE project_id='demoshop' AND product_id='$SKU' AND embedding_version > 0;"
```
**Cách đọc:** `1` = đã có vector, sang bước ② được. `0` = **chưa có** — quay lại chạy cổng kích vector
ở cuối `[01]`, nếu không thì `[07]` sẽ ra 404 thay vì `cold_start_analog`.

> 🆕 **Đã vá 13/08 — phép đo cũ cho XANH GIẢ.** Bản trước viết `AND embedding_version IS NOT NULL`.
> Cột này khai báo là `NOT NULL DEFAULT 0` (đo từ `information_schema`), nên vế `IS NOT NULL`
> **luôn đúng** — kể cả với SKU có `embedding_version = 0` tức chưa hề sinh vector. Chứng minh:
> ```
> sku-chua-co-vector | IS NOT NULL -> true | > 0 -> false
> sku-da-co-vector   | IS NOT NULL -> true | > 0 -> true
> ```
> Tức câu cũ chỉ đếm *"sản phẩm có tồn tại không"*, không đếm *"đã có vector chưa"* như tên biến
> `co_vector` hứa hẹn. **Chỉ tài liệu sai — mã nguồn ĐÚNG** (`store/products.py:251` dùng
> `WHERE embedding_version < $1`, so sánh số, không dùng `IS NOT NULL` ở bất kỳ đâu).
> Đo thật 13/08 trên demoshop: 114/114 SKU có `embedding_version = 1`.

### ② GỌI API
```bash
curl -s "localhost:16021/internal/similar-products?project_id=demoshop&product_id=demo-mi-omachi&limit=5" -H "X-Internal-Token: $ITOK" | .venv/bin/python -m json.tool
```
**OUTPUT thật 12/08**
```json
{"items": [{"product_id": "bh-mi-haohao"}, {"product_id": "bh-nuocgiat-omo"},
           {"product_id": "bh-banh-oreo"}, {"product_id": "bh-snack-oishi"},
           {"product_id": "bh-suatuoi-th"}]}
```

### ③ ĐO SAU — 5 hàng xóm này có lịch sử bán không?
```bash
q miniai_forecast "SELECT product_id, count(*) AS ngay_lich_su FROM demand_daily WHERE project_id='demoshop' AND product_id IN ('bh-mi-haohao','bh-nuocgiat-omo','bh-banh-oreo','bh-snack-oishi','bh-suatuoi-th') GROUP BY 1 ORDER BY 2 DESC;"
```
**Đo thật:** cả 5 đều có **~130 ngày lịch sử** — tức có thật thứ để mượn. Đây chính là đầu vào của bước [07].

### ④ LUỒNG
```
forecast ──gọi chéo (X-Internal-Token)──► smartsearch /internal/similar-products
                                                  └──► vector Vespa ──► 5 hàng xóm
         ◄──── dùng làm nền cho cold_start_analog ở [07]
```
**Nói với khách:** *"Đây là ranh giới nội bộ/công khai: hai service nói chuyện với nhau bằng token riêng,
không dùng key của khách hàng."*

---
## [07] POST /v1/forecast:query — **HỆ KHÔNG BỊA, NÓ NÓI ĐANG MƯỢN CỦA AI**

> 🆕 **ĐỔI SO VỚI BẢN 07/08.** Bản cũ ghi API này trả **404 "từ chối dự báo"**. Từ khi tính năng
> **cold-start analog** (`F-COLDSTART-ANALOG-1`) lên, hệ **trả lời được** — nhưng **khai rõ** là đang mượn.
> Đây là câu chuyện MẠNH HƠN: không phải "tôi từ chối", mà **"tôi trả lời được, và đây là 5 mặt hàng tôi
> dựa vào"**.

### ① ĐO TRƯỚC — chứng minh SKU này chưa bán ngày nào
```bash
echo "demand_daily = $(q miniai_forecast "SELECT count(*) FROM demand_daily WHERE project_id='demoshop' AND product_id='$SKU'")"
echo "forecasts    = $(q miniai_forecast "SELECT count(*) FROM forecasts WHERE project_id='demoshop' AND product_id='$SKU'")"
echo "raw_events   = $(q miniai_forecast "SELECT count(*) FROM raw_events WHERE project_id='demoshop' AND payload::text LIKE '%$SKU%'")"
```
**Đo thật:** `0 · 0 · 0` — **không một dòng dữ liệu bán nào**.

### ② GỌI API
```bash
curl -s -w "\nstatus: %{http_code}\n" localhost:16023/v1/forecast:query -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"demo-mi-omachi","horizon_days":14}' -o /tmp/fq.json; .venv/bin/python -c "
import json; d=json.load(open('/tmp/fq.json'))
print('run_id     =', d['run_id']); print('model_used =', d['model_used'])
print('analog_of  =', d['analog_of']); print('data_window=', d['data_window'])
print('3 ngay dau =', [(x['day'], round(x['p50'],2), round(x['p90'],2)) for x in d['daily'][:3]])"
```
**OUTPUT thật 12/08**
```
status: 200
run_id     = analog_2026-08-12
model_used = cold_start_analog
analog_of  = ['bh-mi-haohao', 'bh-nuocgiat-omo', 'bh-banh-oreo', 'bh-snack-oishi', 'bh-suatuoi-th']
data_window= None
3 ngay dau = [('2026-08-13', 1.69, 3.08), ('2026-08-14', 1.54, 3.55), ('2026-08-15', 2.41, 4.34)]
```

### ③ ĐO SAU — số này KHÔNG được ghi vào kho (chỉ tính lúc hỏi)
```bash
q miniai_forecast "SELECT count(*) AS van_la_0 FROM forecasts WHERE project_id='demoshop' AND product_id='$SKU';"
```
**Đo thật:** vẫn `0`. **Rất quan trọng:** dự báo analog là **câu trả lời tạm lúc đọc**, hệ **không đóng dấu**
nó vào sổ dự báo chính thức — vì nó không phải lịch sử của chính SKU này.

### ④ LUỒNG
```
:query ──không thấy demand_daily──► gọi similar-products ──► 5 hàng xóm
       ──► lấy mẫu mùa vụ của hàng xóm, quy về cùng mức ──► trả p10/p50/p90
       ──► run_id = analog_*  ·  model_used = cold_start_analog  ·  data_window = null
       ──✗ KHÔNG ghi vào bảng forecasts
```
**Câu nói cho khách (chậm rãi):** *"Ba chỗ hệ tự khai mình đang đoán: `run_id` bắt đầu bằng `analog_`,
`model_used` ghi thẳng `cold_start_analog`, và `data_window` để **trống** vì SKU này chưa có ngày dữ liệu nào.
Nó còn liệt kê đúng 5 mặt hàng nó dựa vào để anh chị tự phán xem có hợp lý không. Một hệ thống bịa số sẽ không
bao giờ tự khai như vậy."*

---
## [08] POST /v1/decisions:price-preview — **HỆ NÓI ĐÍCH DANH THIẾU GÌ**

### ① ĐO TRƯỚC — chứng minh thiếu đúng 3 thứ
```bash
echo "doanh so 30d = $(q miniai_decision "SELECT count(*) FROM sales_daily WHERE project_id='demoshop' AND product_id='$SKU' AND day >= CURRENT_DATE-30")"
echo "gia von      = $(q miniai_decision "SELECT coalesce(round(max(ewma_cost))::text,'CHUA CO') FROM cost_state WHERE project_id='demoshop' AND product_id='$SKU'")"
echo "gia hien tai = $(q miniai_decision "SELECT coalesce(max(current_price)::text,'CHUA CO') FROM price_state WHERE project_id='demoshop' AND product_id='$SKU'")"
```
**Đo thật:** `0 · CHUA CO · CHUA CO`

### ② GỌI API
```bash
curl -s -w "\nstatus: %{http_code}\n" localhost:16022/v1/decisions:price-preview -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"demo-mi-omachi","candidate_price":129000}'
```
**OUTPUT thật 12/08**
```json
{"error": {"code": "FAILED_PRECONDITION", "message": "no sales in last 30d"}}
status: 412
```

### ③ ĐO SAU — không có gì được ghi (API chỉ đọc)
```bash
q miniai_decision "SELECT count(*) AS quyet_dinh_moi FROM decisions WHERE project_id='demoshop' AND subject_id='$SKU';"
```
**Đo thật:** `0`

### ④ LUỒNG
```
:price-preview ──► kiểm 3 cổng dữ liệu: doanh số 30d → giá vốn → giá hiện tại
               ──► thiếu cổng nào ⇒ 412 + GỌI TÊN đúng cổng đó
```
**Nói với khách:** *"`412` không phải lỗi hệ thống — là **lỗi dẫn đường**. Nó nói đúng đang thiếu doanh số
30 ngày. Anh chị vừa tự kiểm bằng SQL: đúng là 0 dòng. Máy không nói dối."*

---
# MÀN 3 — NẠP DỮ LIỆU BÁN THẬT (2 API + 1 lệnh vận hành)

## [09] POST /v1/events:backfill (forecast) — đổ 21 ngày lịch sử bán
**Ý nghĩa:** `:backfill` là đường **nạp lịch sử khi onboard khách mới** (cho phép mốc thời gian quá khứ).
Khác `:ingest` là đường sự kiện phát sinh hằng ngày.

### ① ĐO TRƯỚC
```bash
snap $SKU
```
**Đo thật:** `raw_events(forecast)=13604 · demand_daily=0 · sales_daily=0`

### ② GỌI API
| Trường | Ý nghĩa |
|---|---|
| `event_id` | **khoá chống trùng** — gửi lại 2 lần chỉ tính 1 |
| `event_type` | `purchase.completed` |
| `event_time` | ISO-8601 UTC, được phép ở quá khứ |
| `payload.items[]` | `{product_id, qty, unit_price}` — **đúng tên `qty`/`unit_price`** |

```bash
.venv/bin/python - <<'PY'
import json,random
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
curl -s localhost:16023/v1/events:backfill -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d @/tmp/ev.json | .venv/bin/python -m json.tool
```
**OUTPUT thật (sau `reset1`):** `{"accepted": 21, "deduped": 0, "skipped": 0, "errors": [], "conflicted": 0}`

> ⚠ **`conflicted` là gì — và vì sao nó có thể KHÁC 0.**
> Ngoài kho riêng của mỗi service còn có **sổ cái chung** `miniai_ledger.event_ledger` (ghi bóng).
> `conflicted` = số event sổ cái chung **đã có rồi**. Chạy demo lần 2 mà **không** `reset1` thì số này sẽ
> khác 0 (đo thật: `conflicted: 12`) — **vô hại**, chỉ nghĩa là sổ cái nhớ dai hơn kho service.
> `reset1` đã dọn cả sổ cái này nên chạy lại luôn ra `0`.

### ③ ĐO SAU — **sổ cái tăng NGAY, chuỗi ngày thì CHƯA**
```bash
echo "raw_events   = $(q miniai_forecast "SELECT count(*) FROM raw_events WHERE project_id='demoshop' AND payload::text LIKE '%$SKU%'")"
echo "demand_daily = $(q miniai_forecast "SELECT count(*) FROM demand_daily WHERE project_id='demoshop' AND product_id='$SKU'")"
```
**Đo thật:** `raw_events = 21` ✅ · `demand_daily = 0` ⛔

### ④ LUỒNG — **đây là chỗ giải thích kiến trúc hay nhất**
```
:backfill ──ghi NGAY──► raw_events (sổ cái, chỉ ghi thêm, không sửa/xoá)
                            │
                            └──job rollup (mỗi 1 giờ)──► demand_daily (chuỗi theo ngày)
```
**Nói với khách:** *"Sự kiện vào sổ cái ngay lập tức — 21 dòng, anh chị vừa đếm. Nhưng chuỗi bán theo ngày
vẫn là 0, vì việc cộng sổ do một job nền làm theo nhịp. **Ghi nhanh, tính chậm** — đó là lý do cửa vào chịu
được giờ cao điểm."*

### ⑤ Gửi lại lần 2 để chứng minh chống trùng (tuỳ chọn, rất thuyết phục)
```bash
curl -s localhost:16023/v1/events:backfill -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d @/tmp/ev.json | .venv/bin/python -c "import json,sys; d=json.load(sys.stdin); print('accepted =',d['accepted'],'| deduped =',d['deduped'])"
echo "raw_events VAN LA = $(q miniai_forecast "SELECT count(*) FROM raw_events WHERE project_id='demoshop' AND payload::text LIKE '%$SKU%'")"
```
**Đo thật:** `accepted = 0 | deduped = 21`, `raw_events vẫn = 21` — **doanh số không bị nhân đôi**.

---
## [10] POST /v1/events:backfill (decision) — thêm giá vốn + tồn kho

### ① ĐO TRƯỚC
```bash
q miniai_decision "SELECT 'ton='||coalesce(max(on_hand_qty)::text,'—') FROM stock_state WHERE project_id='demoshop' AND product_id='$SKU';"
q miniai_decision "SELECT 'von='||coalesce(round(max(ewma_cost))::text,'—') FROM cost_state WHERE project_id='demoshop' AND product_id='$SKU';"
```
**Đo thật:** `ton=—` · `von=—`

### ② GỌI API
| Loại | Payload | Dùng để |
|---|---|---|
| `cost.recorded` | `{product_id, unit_cost, qty}` | giá vốn bình quân (EWMA) — nền của mọi chốt lãi/lỗ |
| `price.changed` | `{product_id, new_price}` | mốc giá + lịch sử để ước lượng độ co giãn |
| `stock.level` | `{product_id, on_hand_qty}` | tồn kho — nền của kế hoạch nhập hàng |

```bash
EVT=$(date -u +%Y-%m-%dT%H:%M:%SZ)
curl -s localhost:16022/v1/events:backfill -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d "$(EVT="$EVT" .venv/bin/python - <<'PY'
import json, os
ev=json.load(open("/tmp/ev.json"))["events"]
EVT=os.environ["EVT"]
ev+=[{"event_id":"demo-mi-omachi-cost-1","schema_version":"1.0","event_type":"cost.recorded","event_time":"2026-07-24T08:00:00Z","user_pseudo_id":"system","payload":{"product_id":"demo-mi-omachi","unit_cost":98000,"qty":100}},
     {"event_id":"demo-mi-omachi-price-1","schema_version":"1.0","event_type":"price.changed","event_time":"2026-07-24T08:00:00Z","user_pseudo_id":"system","payload":{"product_id":"demo-mi-omachi","new_price":145000}},
     {"event_id":"demo-mi-omachi-stock-1","schema_version":"1.0","event_type":"stock.level","event_time":EVT,"user_pseudo_id":"system","payload":{"product_id":"demo-mi-omachi","on_hand_qty":40}}]
print(json.dumps({"events":ev}))
PY
)" | .venv/bin/python -m json.tool
```
**OUTPUT thật:** `{"accepted": 3, "deduped": 21, "skipped": 0, "errors": [], "conflicted": 0}`

> 🆕 **Vì sao `accepted` chỉ là 3 chứ không phải 24 — và đây là ĐIỂM KHOE, không phải lỗi** (đo 12/08).
> 21 đơn hàng ở bước [09] gửi vào **forecast**, nhưng `purchase.completed` là loại sự kiện **cả decision
> cũng tiêu thụ**, nên chúng đã tự sang decision qua **sổ cái chung + projector**. Lần gửi này chỉ còn 3 sự
> kiện mới (giá vốn · giá · tồn kho), 21 cái kia bị nhận ra là trùng.
> Kiểm ngay bằng SQL — bảng vẫn đủ 24 dòng:
> ```bash
> q miniai_decision "SELECT event_type, count(*) FROM raw_events WHERE project_id='demoshop' AND payload::text LIKE '%demo-mi-omachi%' GROUP BY 1 ORDER BY 2 DESC;"
> ```
> **Đo thật:** `purchase.completed=21 · cost.recorded=1 · price.changed=1 · stock.level=1`.
> *"Anh chị vừa thấy một sự kiện gửi vào một service tự chảy sang service khác cần nó — và không bị đếm hai lần."*
> ⚠ Nếu projector **chưa kịp** chạy thì con số sẽ là `accepted: 24, deduped: 0` — cả hai đều đúng, đừng đọc
> thuộc số.

> ⛔ **`stock.level` PHẢI dùng thời điểm HIỆN TẠI (`$EVT`), không được dùng ngày tương lai.**
> Đo thật 12/08: bản trước ghi `event_time: "2026-08-13"` (ngày mai) ⇒ `state_rollup` **bỏ qua**, và
> `stock_state` mãi rỗng ⇒ bước `[19] replenish-plan` không có tồn kho để tính. Lỗi im lặng, không báo gì.

### ③ ĐO SAU — **hình chiếu cập nhật theo nhịp job, KHÔNG tức thì**
```bash
echo "ngay sau: ton=$(q miniai_decision "SELECT coalesce(max(on_hand_qty)::text,'—') FROM stock_state WHERE project_id='demoshop' AND product_id='$SKU'")  von=$(q miniai_decision "SELECT coalesce(round(max(ewma_cost))::text,'—') FROM cost_state WHERE project_id='demoshop' AND product_id='$SKU'")"
# vòng state_rollup chạy mỗi 300 giây — demo thì kích tay:
docker exec miniai-decision python3 -c "
import asyncio, asyncpg, os, json
from app.jobs.state_rollup import run_state_rollup_once
async def m():
    p=await asyncpg.create_pool(os.environ.get('DECISION_DSN') or os.environ.get('DATABASE_URL'),min_size=1,max_size=3)
    print('state_rollup:', json.dumps(await run_state_rollup_once(p))); await p.close()
asyncio.run(m())"
echo "sau job : ton=$(q miniai_decision "SELECT coalesce(max(on_hand_qty)::text,'—') FROM stock_state WHERE project_id='demoshop' AND product_id='$SKU'")  von=$(q miniai_decision "SELECT coalesce(round(max(ewma_cost))::text,'—') FROM cost_state WHERE project_id='demoshop' AND product_id='$SKU'")"
```
**Đo thật:** `ngay sau: ton=— von=—` → `sau job: ton=40 von=98000`

### ④ LUỒNG
```
:backfill ──► raw_events (NGAY)
                 └──job state_rollup (mỗi 300s)──► stock_state · cost_state · price_state
```
**Nói với khách:** *"Đây là **tách sổ cái khỏi hình chiếu**. Sổ cái là sự thật, ghi ngay và không bao giờ sửa.
Các bảng trạng thái chỉ là ảnh chụp được dựng lại từ sổ — hỏng thì dựng lại từ đầu được."*

---
## [BƯỚC VẬN HÀNH] Kích rollup (không phải API)
```bash
.venv/bin/python -c "
import sys; sys.path.insert(0,'scripts')
import seed_demoshop as sd; sd.run_rollups(window_days=40)"
```
### ĐO SAU
```bash
q miniai_forecast "SELECT count(*) AS ngay, round(avg(units_sold),2) AS ban_tb FROM demand_daily WHERE project_id='demoshop' AND product_id='$SKU';"
q miniai_decision "SELECT count(*) AS ngay, sum(units) AS tong_ban FROM sales_daily WHERE project_id='demoshop' AND product_id='$SKU';"
```
**Đo thật:** `demand_daily` ~**22 ngày**, bán TB ~**4,4/ngày** · `sales_daily` ~**21 ngày**.
**Nói với khách:** *"Đây là nhịp cộng sổ; ngoài đời nó tự chạy mỗi giờ, ở đây tôi kích tay cho nhanh.
21 sự kiện thô vừa biến thành 21 dòng lịch — đó là tầng chuẩn bị dữ liệu."*

---
# MÀN 4 — GIỜ ĐÃ CÓ TRÍ KHÔN (9 API)

## [11] POST /v1/forecast:run — chạy lại dự báo (bất đồng bộ)

### ① ĐO TRƯỚC
```bash
q miniai_forecast "SELECT count(*) AS job_dang_co FROM job_run WHERE tenant_id='demoshop' AND job_type='forecast_run';"
q miniai_forecast "SELECT count(*) AS dong_du_bao FROM forecasts WHERE project_id='demoshop' AND product_id='$SKU';"
```
**Đo thật:** `job = 1` (của hôm qua) · `dòng dự báo = 0`

### ② GỌI API
```bash
curl -s -w "\nstatus: %{http_code}\n" -X POST localhost:16023/v1/forecast:run -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{}'
```
**OUTPUT thật:** `202` + `{"status":"queued","run_id":"r_2026-08-13","job_id":"fr-demoshop-r_2026-08-13"}`

### ③ ĐO SAU — **việc đã vào hàng đợi, chưa làm xong**
```bash
q miniai_forecast "SELECT job_id, status, attempt FROM job_run WHERE tenant_id='demoshop' AND job_type='forecast_run' ORDER BY updated_at DESC LIMIT 1;"
```
**Đo thật:** `fr-demoshop-r_2026-08-13 | queued | 0` — **nhìn thấy việc nằm trong hàng đợi**.

### ④ LUỒNG
```
:run ──► ghi 1 dòng job_run (status=queued) ──► trả 202 NGAY
                    └──forecast-worker nhận việc──► queued → running → done
                                                        └──► ghi bảng forecasts
```
**Nói với khách:** *"`202` = **đã nhận việc**, không phải đã làm xong. Huấn luyện mất vài chục giây nên hệ trả
ngay mã việc. Gọi lại khi việc đang chạy sẽ trả **cùng một mã** — không nhân đôi công."*

---
## [12] GET /v1/projections/status — theo dõi job tới khi xong

### ② GỌI API (vòng lặp bắt buộc)
```bash
JOB="fr-demoshop-r_$(date -u +%F)"
until curl -s "localhost:16023/v1/projections/status?job_id=$JOB" -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" | .venv/bin/python -c "
import json,sys; s=(json.load(sys.stdin).get('job') or {}).get('status'); print('   trang thai:',s); sys.exit(0 if s in ('done','failed') else 1)"; do sleep 5; done
```
**OUTPUT thật:** `queued → running → done` (~30-60 giây)

### ③ ĐO SAU — đối chiếu API với kho
```bash
q miniai_forecast "SELECT status, attempt, coalesce(error_code,'-') FROM job_run WHERE job_id='$JOB';"
q miniai_forecast "SELECT count(*) AS dong_du_bao FROM forecasts WHERE project_id='demoshop' AND product_id='$SKU';"
```
**Đo thật:** `done | 0 | -` · **dòng dự báo: 0 → 28** (đúng 28 ngày tầm nhìn)

### ④ LUỒNG
```
job_run.status ◄── worker cập nhật ──► forecasts (28 dòng/SKU)
projections/status đọc CẢ HAI: trạng thái việc + đã bắt kịp sổ cái chưa
```
**Nói với khách:** *"`is_caught_up: true` = hình chiếu đã bắt kịp sổ cái. Nếu `failed` thì có `error_code`
kèm — **lỗi nhìn thấy được, không nuốt**."*

---
## [13] POST /v1/forecast:query — **GIỜ ĐÃ CÓ SỐ CỦA CHÍNH NÓ**

### ① ĐO TRƯỚC — số nằm sẵn trong kho
```bash
q miniai_forecast "SELECT model_used, data_window, count(*) FROM forecasts WHERE project_id='demoshop' AND product_id='$SKU' GROUP BY 1,2;"
```

### ② GỌI API
```bash
curl -s localhost:16023/v1/forecast:query -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"demo-mi-omachi","horizon_days":14}' | .venv/bin/python -c "
import json,sys; d=json.load(sys.stdin)
print('run_id     =', d['run_id']); print('model_used =', d['model_used']); print('data_window=', d['data_window'])
print('5 ngay dau =', [(x['day'][5:], round(x['p10'],1), round(x['p50'],1), round(x['p90'],1)) for x in d['daily'][:5]])"
```

### ③ ĐO SAU — **API trả đúng số đang nằm trong bảng**
```bash
q miniai_forecast "SELECT horizon_day, round(p10,2), round(p50,2), round(p90,2) FROM forecasts WHERE project_id='demoshop' AND product_id='$SKU' ORDER BY horizon_day LIMIT 5;"
```
**Điểm chốt:** hai bảng số **trùng khít**. API tầng đọc **không tính lại gì** — nó đọc kết quả đã đông lạnh.

### ④ LUỒNG + cách đọc 3 con số
```
forecasts (đã đông lạnh theo run_id) ──:query──► nhân hệ số lịch (Tết/lễ) ──► trả về
```
- **`p50`** = kịch bản giữa → **lập kế hoạch**
- **`p90`** = kịch bản cao → **nhập hàng** (tránh cháy hàng)
- **`p10`** = kịch bản thấp → **giữ dòng tiền**

⭐ **So sánh với bước [07]:** giờ `run_id` là `r_2026-08-13` (không còn `analog_`), `model_used` là mô hình
thật, `data_window` **có giá trị** thay vì `null`. *"Cùng một API, nhưng giờ nó đứng trên dữ liệu của chính
sản phẩm này — và nó nói cho anh chị biết điều đó."*

---
## [14] POST /v1/scenarios:build — dựng kịch bản Monte Carlo

### ① ĐO TRƯỚC
```bash
q miniai_forecast "SELECT count(*) AS so_bo_kich_ban FROM scenario_manifest WHERE project_id='demoshop';"
```

### ② GỌI API
```bash
curl -s localhost:16023/v1/scenarios:build -H "Authorization: Bearer $FKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_ids":["demo-mi-omachi"],"horizon_days":7,"scenario_count":128}' | .venv/bin/python -m json.tool
```
**OUTPUT thật** (rút gọn)
```json
{"run_id": "sc_…", "manifest": {"scenario_count": 128, "horizon_days": 7,
  "rng_algorithm": "philox", "rng_version": "1", "scenario_index_contract": "v1",
  "files": {"marginals.npz": "57c8830a…", "factors.npz": "7aa2c0a0…"},
  "marginals": {"demo-mi-omachi": {"marginal_source": "history_empirical",
                                   "tail": "none", "demand_class": "intermittent"}}}}
```

### ③ ĐO SAU
```bash
q miniai_forecast "SELECT run_id, created_at FROM scenario_manifest WHERE project_id='demoshop' ORDER BY created_at DESC LIMIT 1;"
# ⚠ tệp nằm TRONG CONTAINER (MINIAI_ARTIFACT_DIR=/srv/data/artifacts), KHÔNG nằm ở data/ trên máy host
docker exec miniai-forecast ls -la /srv/data/artifacts/scenario/demoshop/ | tail -3
```
**Đo thật:** `scenario_manifest` +1 dòng, và trong container có thư mục `sc_…` chứa `marginals.npz` ·
`factors.npz` · `manifest.json`.
> ⚠ Bản trước ghi `ls data/artifacts/...` trên host — **sai**, chạy sẽ báo `No such file or directory`.

### ④ LUỒNG
```
:build ──► fit phân phối từ demand_daily ──► ghi 2 tệp .npz + manifest.json (SHA-256 từng tệp)
       ──► ghi scenario_manifest (Postgres) ──► 3 API kịch bản sau dùng run_id này
```
**Điểm khoe:** `rng_algorithm: philox` **có hạt giống, tái lập được** — chạy lại ra đúng bộ kịch bản cũ.
`files` kèm **mã băm SHA-256** = bằng chứng chống sửa lén. `marginal_source: history_empirical` = phân phối
lấy từ **lịch sử thật**, không phải giả định hình chuông.

---
## [15] POST /v1/decisions:run — chạy bộ não quyết định

### ① ĐO TRƯỚC
```bash
q miniai_decision "SELECT count(*) AS tong, count(*) FILTER (WHERE created_at::date=CURRENT_DATE) AS hom_nay FROM decisions WHERE project_id='demoshop';"
```

### ② GỌI API
```bash
curl -s -X POST localhost:16022/v1/decisions:run -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{}' | .venv/bin/python -m json.tool
```
**OUTPUT thật 12/08** (số sẽ đổi theo ngày)
```json
{"created": 2, "skipped_dedup": 149,
 "skipped_by_reason": {"anti_oscillation": 143, "plan_conflict": 83,
                       "insufficient_history": 2, "no_stock": 2, "no_cost": 63},
 "superseded_plan": 0, "price_hold": 1, "anti_osc_hold": 1}
```
> 🆕 Có thêm 2 trường `price_hold` / `anti_osc_hold` — số lời khuyên "giữ giá" và số ca bị khoá vì vừa đổi giá.

### ③ ĐO SAU
```bash
q miniai_decision "SELECT kind, count(*) FROM decisions WHERE project_id='demoshop' AND created_at > now()-interval '5 min' GROUP BY 1;"
q miniai_decision "SELECT decision_id, kind, status FROM decisions WHERE project_id='demoshop' AND subject_id='$SKU';"
```
**Đo thật:** đúng bằng `created` ở trên, và thấy được lời khuyên nào thuộc SKU demo.

### ④ LUỒNG + ý nghĩa từng con số
```
:run ──► với MỖI SKU: đọc sales_daily + cost_state + stock_state + forecasts
     ──► sinh ứng viên ──► DecisionPlan giải xung đột ──► guardrails ──► ghi decisions
```
| Trường | Nghĩa |
|---|---|
| `created` | số lời khuyên **mới** |
| `skipped_dedup` | đã có lời khuyên y hệt đang mở → **không spam lại** |
| `anti_oscillation` | **chặn đổi giá liên tục** — SKU vừa đổi giá thì khoá |
| `plan_conflict` | cùng SKU đã có hành động giá khác → tránh 2 lệnh mâu thuẫn |
| `no_stock` / `no_cost` | thiếu tồn / thiếu vốn → **không khuyên bừa** |

---
## [16] GET /v1/decisions — danh sách lời khuyên

### ② GỌI API — 🆕 lọc thẳng theo SKU (vá 12/08)
```bash
curl -s "localhost:16022/v1/decisions?product_id=demo-mi-omachi&page_size=50" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | .venv/bin/python -c "
import json,sys
for x in json.load(sys.stdin)['items']: print(x['decision_id'],'|',x['kind'],'|',x['status'])"
```
**OUTPUT thật 12/08:** đúng **2 dòng** — `price_hold` và `replenishment_advice`, không lẫn SKU nào khác.

### ③ ĐO SAU — đối chiếu API với kho (phải khớp)
```bash
q miniai_decision "SELECT decision_id, kind, status, presentable FROM decisions WHERE project_id='demoshop' AND subject_id='$SKU' ORDER BY created_at DESC;"
```
> 🆕 **Đã vá 12/08:** trước đây `?product_id=` bị **bỏ qua im lặng** (FastAPI lờ mọi query param không khai
> báo) nên API trả nguyên danh sách cả shop trong khi người gọi tưởng đã lọc — phải lọc tay phía client.
> Nay `product_id` là bí danh của `subject_id` và lọc thật; truyền cả hai mà khác giá trị thì báo lỗi 400
> thay vì tự chọn hộ.

### ④ Mỗi lời khuyên gồm
| Trường | Nghĩa |
|---|---|
| `expected_value` | **lợi ích kỳ vọng bằng tiền/tháng** — cơ sở xếp ưu tiên |
| `confidence` | độ tin theo chất lượng bằng chứng |
| `guardrails` | các chốt an toàn đã kiểm và kết quả |
| `trace` | **toàn bộ phép tính viết ra bằng chữ** — tự kiểm được, không phải hộp đen |

---
## [17] POST /v1/decisions:price-preview — **GIỜ TRẢ LỜI ĐƯỢC** (trước đó 412)

### ① ĐO TRƯỚC — 3 cổng dữ liệu giờ đã đủ
```bash
echo "doanh so 30d = $(q miniai_decision "SELECT count(*) FROM sales_daily WHERE project_id='demoshop' AND product_id='$SKU' AND day >= CURRENT_DATE-30")"
echo "gia von      = $(q miniai_decision "SELECT round(max(ewma_cost)) FROM cost_state WHERE project_id='demoshop' AND product_id='$SKU'")"
echo "do co gian   = $(q miniai_decision "SELECT coalesce(round(max(eps)::numeric,4)::text,'chua co') FROM elasticity WHERE project_id='demoshop' AND product_id='$SKU'")"
```
**Đo thật:** `21 · 98000 · <có giá trị>` — **so với màn 2 là `0 · CHUA CO · CHUA CO`**.

### ② GỌI API
```bash
curl -s localhost:16022/v1/decisions:price-preview -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"demo-mi-omachi","candidate_price":129000}' | .venv/bin/python -m json.tool
```
**OUTPUT thật** (hình dạng — số đổi theo dữ liệu sinh ngẫu nhiên)
```json
{"current":   {"price": 145000.0, "est_units_30d": 89.0,  "est_profit_30d": 4183000.0},
 "candidate": {"price": 129000,   "est_units_30d": 95.17, "est_profit_30d": 2950372.3},
 "delta_profit_30d": -1232627.68,
 "elasticity_used": {"eps": -0.5736, "method": "pooled_prior", "n_points": 21, "r2": null},
 "guardrails": [{"code": "BELOW_COST", "status": "PASS"}],
 "confidence": 0.7,
 "explanation": "Q(P)=Q0·(P/P0)^eps; profit=(P-c)·Q"}
```

### ③ ĐO SAU — đối chiếu độ co giãn API dùng với bảng
```bash
q miniai_decision "SELECT eps, n_points, r2, method FROM elasticity WHERE project_id='demoshop' AND product_id='$SKU';"
```
**Điểm chốt:** `method = pooled_prior` = **khai thật đang MƯỢN** độ co giãn trung bình của shop vì SKU mới
chỉ có 21 điểm. `r2 = null` = chưa có độ khớp riêng. `confidence 0.7` (không phải 0.9) — **tự hạ điểm tin cậy
vì bằng chứng yếu hơn**.

### ④ Đọc kết quả cho khách
Giảm 145.000 → 129.000 thì **bán thêm** (89 → 95 thùng/tháng)… nhưng
`delta_profit_30d = **−1,23 triệu/tháng**` ⇒ **lãi GIẢM**. *"Máy can bằng con số, chặn trực giác 'giảm giá
cho chạy hàng'."*

---
## [18] POST /v1/decisions:price-preview (giá dưới vốn) — guardrail phải chặn

### ① ĐO TRƯỚC
```bash
q miniai_decision "SELECT round(max(ewma_cost)) AS gia_von FROM cost_state WHERE project_id='demoshop' AND product_id='$SKU';"
```
**Đo thật:** `98000` — **giá thử 80.000 sẽ nằm DƯỚI vốn**.

### ② GỌI API
```bash
curl -s localhost:16022/v1/decisions:price-preview -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"product_id":"demo-mi-omachi","candidate_price":80000}' | .venv/bin/python -c "import json,sys; d=json.load(sys.stdin); print('guardrails:', d['guardrails']); print('delta_profit_30d:', round(d['delta_profit_30d']))"
```
**OUTPUT thật:** `guardrails: [{'code': 'BELOW_COST', 'status': 'FAIL'}]` · lãi tháng **âm ~6,4 triệu**

### ④ Nói với khách
*"Giá thử 80.000 trong khi giá vốn 98.000 — anh chị vừa tự truy vấn con số vốn đó. Guardrail trả **FAIL**.
Chốt an toàn này được sửa ngày 06/08 (trước đó cả hai nhánh đều báo PASS) — lỗi do chính buổi tập của chúng
tôi tìm ra, và đã có test hồi quy khoá lại."*

---
## [19] GET /v1/decisions:replenish-plan — nhập bao nhiêu, khi nào

### ① ĐO TRƯỚC — nguyên liệu của phép tính
```bash
q miniai_decision "SELECT round(avg(units),3) AS ban_tb_ngay, round(stddev_samp(units),3) AS dao_dong FROM sales_daily WHERE project_id='demoshop' AND product_id='$SKU' AND day >= CURRENT_DATE-30;"
q miniai_decision "SELECT on_hand_qty AS ton_kho FROM stock_state WHERE project_id='demoshop' AND product_id='$SKU';"
```

### ② GỌI API
```bash
curl -s "localhost:16022/v1/decisions:replenish-plan?product_id=demo-mi-omachi" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | .venv/bin/python -m json.tool
```
**OUTPUT thật** (hình dạng)
```json
{"items": [{"product_id": "demo-mi-omachi",
  "avg_daily_units": 2.967, "sigma_daily": 2.773,
  "lead_time_days": 7.0, "lead_time_std": 2.0,
  "service_level": 0.9, "z": 1.28,
  "safety_stock": 12.08, "reorder_point": 32.84,
  "on_hand": 40.0, "days_of_inventory": 13.5, "below_reorder_point": false,
  "formula": "ROP = avg_daily*LT + z*sqrt(LT*sigma_d^2 + avg_d^2*sigma_LT^2); DOI = on_hand/avg_daily"}]}
```

### ③ ĐO SAU — **tự kiểm phép tính bằng tay**
```bash
.venv/bin/python -c "
import math
avg, sig, LT, sLT, z, on_hand = 2.967, 2.773, 7.0, 2.0, 1.28, 40.0
ss  = z*math.sqrt(LT*sig**2 + avg**2*sLT**2)
rop = avg*LT + ss
print(f'safety_stock  = {ss:.2f}   (API nói 12.08)')
print(f'reorder_point = {rop:.2f}  (API nói 32.84)')
print(f'days_of_inv   = {on_hand/avg:.1f}   (API nói 13.5)')"
```
**Đây là khoảnh khắc mạnh nhất của màn 4:** công thức in ngay trong kết quả API, khách **tự bấm máy tính ra
đúng con số đó**.

### ④ Dịch sang lời chủ shop
- Bán TB **2,97 thùng/ngày**, dao động ±2,77 · hàng về mất **7 ngày** (±2)
- Muốn **90% không cháy hàng** ⇒ trữ thêm **12,08 thùng dự phòng**
- ⇒ **Đặt lại khi còn 32,84 thùng.** Đang có 40 ⇒ **chưa cần đặt**, đủ bán **13,5 ngày**

---
## [20] POST /v1/decisions/{id}:feedback — chủ shop phán, khép vòng

### ① ĐO TRƯỚC
```bash
q miniai_decision "SELECT count(*) AS so_phan_hoi FROM feedback WHERE project_id='demoshop';"
```

### ② GỌI API
```bash
DID=$(curl -s "localhost:16022/v1/decisions?page_size=50" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" | .venv/bin/python -c "
import json,sys
mine=[x for x in json.load(sys.stdin)['items'] if 'demo-mi-omachi' in json.dumps(x)]
print(mine[0]['decision_id'] if mine else '')")
echo "decision_id = $DID"
curl -s -X POST "localhost:16022/v1/decisions/$DID:feedback" -H "Authorization: Bearer $DKEY" -H "X-Project-Id: demoshop" -H "Content-Type: application/json" -d '{"action":"accepted","note":"demo doi tac"}' | .venv/bin/python -c "import json,sys; d=json.load(sys.stdin); print(d['decision_id'], '->', d.get('status'))"
```

### ③ ĐO SAU
```bash
q miniai_decision "SELECT count(*) AS so_phan_hoi FROM feedback WHERE project_id='demoshop';"
q miniai_decision "SELECT decision_id||' | '||action||' | '||coalesce(outcome_note,'(khong ghi chu)') FROM feedback WHERE project_id='demoshop' ORDER BY ts DESC LIMIT 1;"
q miniai_decision "SELECT status FROM decisions WHERE decision_id='$DID';"
```
**Đo thật 12/08:** `feedback +1`, dòng mới đúng `decision_id` vừa phản hồi, `decisions.status = accepted`,
và `outcome_note = "demo doi tac"` — **ghi chú của chủ shop được lưu**.
> ⚠ Dùng `coalesce(...)` như trên — nối chuỗi với cột NULL trong Postgres cho ra **NULL**, tức dòng in ra
> **rỗng trơn** và trông như lệnh hỏng (bản trước vấp đúng chỗ này).
> 🆕 **Đã vá 12/08:** trường `note` (đúng thứ mọi ví dụ ở đây gửi) trước kia bị **nuốt im lặng** — handler
> chỉ đọc `outcome_note`, nên API vẫn trả 200, dòng feedback vẫn vào bảng, **chỉ mất chữ**. Nay `note` là bí
> danh hợp lệ; gửi cả hai tên với giá trị khác nhau thì báo 400 thay vì tự chọn hộ.

### ④ LUỒNG — vòng khép kín
```
decisions ──► chủ shop phán ──► feedback ──► accepted_rate (bước [26] DEMO-2)
                                        └──► sau 30 ngày: outcome_ledger
                                              so LÃI THỰC TẾ với expected_value đã hứa
```
**Câu chốt:** *"Đây là thứ phân biệt một hệ thống AI nghiêm túc với một cái máy đoán: **nó chịu trách nhiệm
với lời khuyên của mình bằng số**."*

---
# DỌN SÂN (BẮT BUỘC — chạy ngay sau khi khách rời phòng)

> ⚠ **Bản 07/08 dọn THIẾU 2 bảng** (`query_log`, `suggest_terms`) nên lần demo sau `weight` không còn bằng
> 1.0. Khối dưới đây đã bổ sung và **đã chạy kiểm chứng 12/08**.

```bash
curl -s -X DELETE "localhost:16021/v1/products/demo-mi-omachi" -H "Authorization: Bearer $SKEY" -H "X-Project-Id: demoshop" -w "xoa san pham: status %{http_code}\n"

for db in miniai_forecast miniai_decision miniai_search; do docker exec miniai-postgres psql -U miniai -d $db -tAc "DO \$\$ DECLARE t text; BEGIN FOR t IN SELECT table_name FROM information_schema.columns WHERE table_schema='public' AND column_name='product_id' LOOP EXECUTE format('DELETE FROM %I WHERE product_id=%L', t, 'demo-mi-omachi'); END LOOP; END \$\$;" >/dev/null 2>&1; done
q miniai_decision "DELETE FROM decisions WHERE project_id='demoshop' AND (subject_id='demo-mi-omachi' OR trace LIKE '%demo-mi-omachi%');"
q miniai_forecast "DELETE FROM raw_events WHERE project_id='demoshop' AND payload::text LIKE '%demo-mi-omachi%';"
q miniai_decision "DELETE FROM raw_events WHERE project_id='demoshop' AND payload::text LIKE '%demo-mi-omachi%';"
# 🆕 hai bảng bản cũ bỏ sót — không xoá thì suggest weight sẽ KHÁC 1.0 lần sau
q miniai_search  "DELETE FROM query_log     WHERE project_id='demoshop' AND query_norm LIKE '%omachi%';"
q miniai_search  "DELETE FROM suggest_terms WHERE project_id='demoshop' AND term       LIKE '%omachi%';"
rm -f /tmp/ev.json /tmp/fq.json
```

**Kiểm lại đã sạch chưa — phải ra 0 hết:**
```bash
snap demo-mi-omachi
q miniai_search "SELECT 'query_log='||count(*) FROM query_log WHERE project_id='demoshop' AND query_norm LIKE '%omachi%';"
q miniai_search "SELECT 'suggest_terms='||count(*) FROM suggest_terms WHERE project_id='demoshop' AND term LIKE '%omachi%';"
```

---
# BẢNG TỔNG KẾT MÀN DEMO (chiếu lên màn hình lúc chốt)

| Câu hỏi | Trước khi có dữ liệu | Sau 21 ngày dữ liệu |
|---|---|---|
| Tìm thấy hàng không? | ✅ sau ~10 giây (nhìn thấy hàng đợi 0→1→0) | ✅ |
| Gợi ý gõ phím? | ✅ `weight 1.0` | (DEMO-2: **334.8**) |
| Gợi ý hàng liên quan? | ✅ đúng ngành (bậc thang cold-start) | ✅ |
| Dự báo nhu cầu? | ⚠️ **có số, nhưng khai `cold_start_analog` + liệt kê 5 hàng nó mượn** | ✅ mô hình riêng, `data_window` có giá trị |
| Khuyên giá? | ❌ **412 — nói rõ thiếu doanh số** (tự kiểm bằng SQL: 0 dòng) | ✅ kèm cảnh báo lãi giảm ~1,23 triệu/tháng |
| Chặn bán dưới vốn? | — | ✅ `BELOW_COST: FAIL` |
| Nhập hàng bao nhiêu? | ❌ chưa đủ cơ sở | ✅ ROP 32,8 — còn 40, đủ 13,5 ngày, **tự bấm máy ra đúng số** |
| Độ tin cậy khai báo? | — | ✅ `pooled_prior`, `confidence 0.7` — **tự khai đang mượn** |

**Câu chốt:** *"Mọi câu trả lời của hệ đều tự khai nó dựa trên cái gì và tin tới đâu. Khi chưa đủ cơ sở, nó
nói thiếu gì hoặc nói đang mượn của ai — chứ không đoán bừa. Và anh chị vừa tự kiểm từng con số bằng SQL,
không phải tin lời tôi."*

---
# VÁ NGÀY 13/08 — chỉ sửa TÀI LIỆU, KHÔNG đụng mã nguồn

> ⛔ **Không deploy gì, nên 4 lượt nghiệm thu e2e đã chạy VẪN CÒN GIÁ TRỊ** (luật đầu file chỉ reset
> khi thay đổi **bản code**). Hai sửa đổi dưới đây nằm hoàn toàn trong file `.md` này.

| Chỗ | Bản cũ | Nay | Vì sao |
|---|---|---|---|
| ⛔ `[03]` ③ ĐO SAU | `round(weight,2)` | **`round(weight::numeric,2)`** | **LỆNH NỔ TRƯỚC MẶT KHÁCH.** `suggest_terms.weight` kiểu `double precision`; Postgres **không có** `round(double, int)`, chỉ có `round(numeric, int)`. Tái lập 100%: `ERROR: function round(double precision, integer) does not exist`. Cùng lỗi ở **DEMO-2** `[03]`①③ và `[04]`① — đã vá cả 4 chỗ |
| `[06]` ① ĐO TRƯỚC | `embedding_version IS NOT NULL` | **`embedding_version > 0`** | cột là `NOT NULL DEFAULT 0` ⇒ vế cũ **luôn đúng**, cổng canh vector cho **xanh giả**. Mã nguồn không sai — `store/products.py:251` dùng `WHERE embedding_version < $1` |
| `[03]` điểm khoe | `weight 334.8` | **`~400`** + cảnh báo | số này **lớn dần theo ngày**: 334.8 (07/08) → 376.96 (12/08) → 401.28 (13/08) |
| chú thích hàm `q()` | "1 trong **3** kho" | "1 trong **4** kho" | `reset1` có gọi `q miniai_ledger`, chú thích cũ bỏ sót sổ cái chung |

**Đã chạy kiểm chứng 13/08 — cả 4 câu sửa đều ra kết quả, không còn `ERROR`:**
```
DEMO-2 [03]① →  mì|401.28   mì hảo|401.28   mì hảo hảo|401.28
DEMO-2 [04]① →  bh-snack-oishi|71|34.70     bh-xucxich-ducviet|69|28.21
```

⚠ **Vì sao 4 lượt e2e trước KHÔNG bắt được lỗi này:** đây là các câu SQL ở mục "① ĐO TRƯỚC / ③ ĐO SAU" —
phần *người dẫn gõ tay để chứng minh*, không nằm trong đường chạy API mà bộ e2e đo. Bài học: **lệnh trong
tài liệu cũng là deliverable, cũng phải chạy thật ≥1 lần** (LUẬT-0 mục 4).

**Ba điểm ĐÃ RÀ nhưng CỐ Ý KHÔNG SỬA trước demo** (đổi hành vi lệnh ⇒ phải đếm lại 4 lượt; ghi ra đây để quyết sau):

1. Vòng lặp `DO $$` trong `reset1` xoá **không kèm `project_id`** ⇒ về nguyên tắc là xoá **xuyên tenant**.
   Vô hại hiện tại vì mã `demo-mi-omachi` chỉ tồn tại ở `demoshop`.
2. Cùng vòng lặp đó có `2>&1` ⇒ **giấu cả luồng lỗi**. Postgres chết cũng không hiện gì; dòng `echo`
   cuối hàm `reset1` là phép kiểm chứng độc lập duy nhất.
3. `LIKE '%omachi%'` ở 2 dòng dọn `query_log`/`suggest_terms` là tìm **chuỗi con**, sẽ quét cả SKU thật
   nào có chứa chữ "omachi" nếu shop phát sinh sau này.

> 📘 Lược đồ CSDL đầy đủ (70 bảng · từng cột · ERD · ai ghi bảng nào): `icpp/db-docs/MINIAI-DB-SCHEMA.md`

---

# ĐÃ ĐỔI SO VỚI BẢN 07/08 (đo lại 12/08 — đọc trước khi lên sân)

| Chỗ | Bản 07/08 | Thực tế 12/08 | Ảnh hưởng demo |
|---|---|---|---|
| **[07] forecast:query** | `404` từ chối | **`200`** + `cold_start_analog` + `analog_of` 5 SKU | **ĐỔI MẠCH KỂ** — xem lại lời thoại ở [07] |
| [02] search | `score 10.1054`, `source rrf_fusion` | `score 10.2236`, **`source vespa_bm25`** | chỉ đổi số |
| [03] suggest | 3 trường `consistency` | thêm **`data_as_of`** | chỉ thêm trường |
| [04] recommend | `score 291` (thang trăm) | **`score 0.30`** (thang 0-1, nấc nội dung) | đổi lời giải thích thang điểm |
| [06] similar-products | `{"items": []}` rỗng | **5 hàng xóm** | đổi từ "thà rỗng còn hơn sai" sang "đã có nền cho analog" |
| Dọn sân | thiếu `query_log` + `suggest_terms` | đã bổ sung | không dọn ⇒ `weight ≠ 1.0` lần sau |
| Lệnh `python3` | `python3 -m json.tool` | **`.venv/bin/python`** | `python3` hệ thống thiếu thư viện |
