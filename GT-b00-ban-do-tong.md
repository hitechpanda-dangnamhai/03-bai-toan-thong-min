# BT03 — FORECAST · BÀI 0: BẢN ĐỒ TỔNG

> **Giáo trình đọc từ code thật.** Bản giảng theo file `mecom/ALGO-FORECAST-BT03-2026-08-07.md` (1958 dòng, 14 mục).
> Mọi con số kèm phép tính và vị trí `file.py:dòng`. Chỗ nào code không nói rõ thì ghi **CHƯA CHẮC** — không đoán.
>
> Nguồn: code trong `mecom/project/services/forecast/` · Số đo: Postgres `localhost:16024`, đo 2026-08-07
> Bài này giảng ngày 2026-08-10.

---

> **Cách dùng tài liệu này (luật human ban 2026-08-10).** Anh hỏi bất cứ mục nào chưa rõ, tôi giải thích xong
> sẽ **viết thẳng phần giải thích đó vào đúng mục** trong file này — để tài liệu giàu thêm và dễ đọc hơn sau mỗi
> câu hỏi, và để anh chỉ cần đọc file, không phải lội lại chat. Chỗ nào câu hỏi lộ ra lỗi của bản trước thì
> đính chính được ghi ngay tại đó, giữ nguyên cả vết sai.

## LỘ TRÌNH — 10 BÀI TỪ 14 MỤC

| Bài | Mục | Nội dung | Trạng thái |
|---|---|---|---|
| **0** | §0 | Bản đồ tổng: 4 tầng · 3 job nền · 33 thuật toán | **đang học** |
| 1 | §1 | Chuẩn bị dữ liệu: rollup · censored demand · promo deflation · học hệ số `k` | kế tiếp |
| 2 | §2 | Phân loại chuỗi Syntetos–Boylan (ADI / CV²) | |
| 3 | §3.1–3.3 | Chuỗi trơn: `cold_start` · `seasonal_naive` · `autoets_theta_ensemble` | |
| 4 | §3.4–3.6 | Chuỗi gián đoạn: Croston/SBA + NBD · ADIDA · IMAPA | |
| 5 | §3.7–3.9 | `similar_item_transfer` · `cold_start_analog` · LightGBM global quantile | |
| 6 | §4 | Router · backtest rolling-origin · `choose_model` · `calibration_factor` | |
| 7 | §5–6 | Lịch Tết VN 3 pha · Hierarchical reconciliation MinT/WLS | |
| 8 | §7–8 | Scenario Fabric (Philox · copula · TailCalibrator) · khoảng tin cậy | |
| 9 | §9–14 | Insights · số đo thật · bất biến & cạm bẫy · 3 phát hiện code · phần "vì sao" | |

Thứ tự cứng: không có bản đồ thì mọi công thức chỉ là mảnh rời; không hiểu tầng chuẩn bị dữ liệu thì không hiểu vì sao model đọc `adjusted_units` chứ không đọc `units_sold`.

---

## 0.1 · MỘT CÂU ĐỊNH NGHĨA BT03

> BT03 nhận **event bán hàng lẻ tẻ theo giây**, trả về **phân phối cầu tương lai theo ngày cho từng SKU** — không phải một con số, mà là ba con số p10/p50/p90 (kèm p95/p99 khi được hỏi).

Đây là ranh giới giữa "làm forecast" nghiệp dư và chuyên nghiệp: **đầu ra là phân phối, không phải điểm**. Lý do nghiệp vụ: người mua hàng không cần "trung bình bán 10 cái", họ cần "nhập bao nhiêu để 90% khả năng không đứt hàng".

Ba cụm từ trong câu này được mổ kỹ ở phần **HỎI & ĐÁP** phía dưới.

---

## 0.2 · BỐN TẦNG — ĐỌC THEO CHIỀU DỮ LIỆU CHẢY

### Trước hết: "tầng" ở đây nghĩa là gì?

Bốn tầng **không phải bốn phần mềm**, cũng không phải bốn máy chủ. Chúng là **bốn trạng thái mà cùng một
thông tin đi qua**, mỗi trạng thái nằm ở một chỗ khác nhau và do một người khác nhau tạo ra:

| Tầng | Thông tin ở dạng gì | Nằm ở đâu | Ai tạo ra |
|---|---|---|---|
| 1 — THU | *sự việc lẻ* ("lúc 07:15 có người mua 2 hộp") | bảng `raw_events` | cửa hàng bắn vào qua API |
| 2 — CHUẨN BỊ | *chuỗi theo ngày* ("ngày 09/08 bán 6 hộp") | bảng `demand_daily` | job `rollup` chạy mỗi giờ |
| 3 — GHI | *dự báo đã chốt* ("11/08 nhiều khả năng bán 2 hộp") | bảng `forecasts` | job `forecast_run` chạy mỗi ngày |
| 4 — ĐỌC | *câu trả lời cho người dùng* ("nhập 5 hộp là đủ") | không lưu — sinh lúc gọi | API khi có người hỏi |

Nói ngắn: **tầng 1 là cái đã xảy ra · tầng 2 là cái đã xảy ra được sắp gọn · tầng 3 là cái sắp xảy ra ·
tầng 4 là cái nên làm.**

### Đi theo MỘT sự việc có thật, từ đầu đến cuối

Toàn bộ ví dụ dưới đây là **dữ liệu thật lấy ra từ DB lúc 03:10 ngày 2026-08-10**, không phải số bịa để minh
hoạ. SKU: **`ld-srm-cerave`** (sữa rửa mặt CeraVe) của shop `demoshop`. Anh gõ lại lệnh nào cũng ra đúng số đó.

---

#### TẦNG 1 — ngày 31/05, bốn lượt mua rời rạc

Bốn **khách khác nhau**, bốn **đơn hàng khác nhau**, mỗi đơn là một dòng trong `raw_events`. Đây là bốn dòng
thật, đủ mọi cột nhận dạng:

| Giờ | `event_id` | `order_ref` | khách | qty | unit_price |
|---|---|---|---|---|---|
| 10:04:41 | `ds8-ld-srm-cerave-…-buy` | `ds-ord-ld-srm-cerave-2026-05-31` | `u350` | 5 | 355.000 |
| 11:29:24 | `ds8-ld-taytrang-garnier-…-buy` | `ds-ord-ld-taytrang-garnier-2026-05-31` | `u124` | 1 | 355.000 |
| 11:41:42 | `ds8-ld-kcn-anessa-…-buy` | `ds-ord-ld-kcn-anessa-2026-05-31` | `u319` | 1 | 355.000 |
| 12:59:13 | `ds8-ld-toner-klairs-…-buy` | `ds-ord-ld-toner-klairs-2026-05-31` | `u155` | 1 | 355.000 |

> ⚠ **Câu hỏi rất đúng của học viên: "4 lượt mua mà sao chỉ thấy 1 đơn?"**
> Đó là lỗi trình bày của bản trước — nó rút gọn `order_ref` thành `"..."` nên đọc ra như thể cả bốn lượt
> chung một đơn. Sự thật: **bốn `event_id` khác nhau, bốn `order_ref` khác nhau, bốn khách khác nhau**
> (`u350` · `u124` · `u319` · `u155`). Bảng trên là dữ liệu nguyên trạng, không rút gọn nữa.

#### Một event = một ĐƠN HÀNG = một GIỎ nhiều mặt hàng

Đây là chỗ dễ hiểu nhầm nhất, và nhìn payload thật là thông ngay. Toàn văn đơn lúc 11:29:24:

```json
{
  "order_ref": "ds-ord-ld-taytrang-garnier-2026-05-31",
  "items": [
    { "product_id": "ld-taytrang-garnier", "qty": 4, "unit_price": 156000 },
    { "product_id": "ld-kcn-anessa",       "qty": 1, "unit_price": 553000 },
    { "product_id": "ld-srm-cerave",       "qty": 1, "unit_price": 355000 }
  ]
}
```

Chị `u124` đi mua **nước tẩy trang Garnier** (4 chai) — đó là món chính, và tên đơn được đặt theo nó. Nhưng
trong cùng giỏ hàng chị lấy thêm 1 kem chống nắng Anessa và **1 chai CeraVe**. Ba đơn 11:29 · 11:41 · 12:59
đều là kiểu đó: **đơn của SKU khác, nhưng trong giỏ có kèm CeraVe**. Chỉ đơn 10:04 mới thực sự là "đơn CeraVe".

Rút ra ba điều, đều quan trọng cho các bài sau:

1. **`items` là một MẢNG.** Một đơn chứa nhiều dòng hàng. Muốn biết một SKU bán bao nhiêu thì phải **mở mảng
   ra** (`jsonb_array_elements`) rồi cộng theo `product_id` — không đếm số đơn.
2. **Đếm đơn ≠ đếm hàng.** Ngày 31/05 CeraVe có mặt trong **4 đơn** nhưng bán **8 chai**. Ai đếm đơn sẽ ra 4
   và sai gấp đôi.
3. **`order_ref` chỉ là tên gọi của đơn**, không phải nhãn phân loại mặt hàng. Đơn tên "garnier" vẫn chứa
   CeraVe. Tin vào tên đơn thay vì đọc `items` là một lỗi dễ mắc — và im lặng.

Hình dạng chung của gói tin, viết gọn lại để nhớ:

```json
{ "order_ref": "<mã đơn>",
  "items": [ { "product_id": "<SKU>", "qty": <số lượng>, "unit_price": <giá 1 đơn vị> }, … ] }
```

#### Định dạng ĐẦY ĐỦ của một event — và vì sao `event_id` không nằm trong payload

> ⚠ **Câu hỏi của học viên: "trong JSON kia không thấy `event_id`, nó được tạo thế nào?"**
> Vì khối JSON đó **chỉ là cột `payload`** — tức phần *ruột*. `event_id` nằm ở lớp ngoài, gọi là **phong bì**
> (envelope). Bản trước chỉ in ruột nên nhìn thiếu. Dưới đây là toàn bộ hai lớp.

**Một event có hai lớp:**

```
PHONG BÌ (envelope) — ai · loại gì · lúc nào · số hiệu gì   → thành CÁC CỘT của raw_events
   └── RUỘT (payload) — chi tiết riêng của từng loại        → thành MỘT CỘT jsonb duy nhất
```

Lớp phong bì, đúng theo hợp đồng `libs/common/contracts/events.py:151` (`EventEnvelope`):

| Trường | Bắt buộc | Ràng buộc thật trong code | Nghĩa |
|---|---|---|---|
| `event_id` | ✅ | 1–128 ký tự | **số hiệu do NGƯỜI GỬI đặt** — khoá chống trùng |
| `event_type` | ✅ | 1 trong 13 giá trị đóng | loại sự việc |
| `event_time` | ✅ | **phải có múi giờ**; không quá tương lai 5 phút; không cũ quá 90 ngày | lúc việc xảy ra |
| `user_pseudo_id` | ✅ | 1–128 ký tự | khách ẩn danh |
| `schema_version` | — | mặc định `"1.0"` | phiên bản định dạng |
| `session_id` | — | ≤128 ký tự | phiên duyệt web |
| `attribution_token` | — | ≤128 ký tự | vết nguồn traffic |
| `payload` | — | dict, kiểm theo `event_type` | ruột |

Thân request gửi lên `POST /v1/events:ingest` là **một lô**, không phải một cái:

```json
{ "events": [ <phong bì 1>, <phong bì 2>, … ] }
```

Giới hạn thật: tối đa **1.000 event/lô** (`MAX_BATCH_SIZE`, `ingest.py:40`) và thân request ≤ **5 MB**
(`main.py:439`).

**Đơn hàng lúc 11:29 ở trên, viết ĐẦY ĐỦ cả hai lớp** (đây mới là thứ shop thật sự gửi đi):

```json
{
  "events": [
    {
      "event_id": "ds8-ld-taytrang-garnier-2026-05-31-buy",
      "event_type": "purchase.completed",
      "event_time": "2026-05-31T11:29:24+00:00",
      "user_pseudo_id": "u124",
      "schema_version": "1.0",
      "payload": {
        "order_ref": "ds-ord-ld-taytrang-garnier-2026-05-31",
        "items": [
          { "product_id": "ld-taytrang-garnier", "qty": 4, "unit_price": 156000 },
          { "product_id": "ld-kcn-anessa",       "qty": 1, "unit_price": 553000 },
          { "product_id": "ld-srm-cerave",       "qty": 1, "unit_price": 355000 }
        ]
      }
    }
  ]
}
```

#### `event_id` được tạo như thế nào?

**Hệ KHÔNG tự sinh. Người gửi đặt, và bắt buộc phải có** — đây là chủ ý, không phải thiếu sót.

Nhìn `event_id` thật trong dữ liệu demo sẽ thấy quy tắc đặt:

```
ds8 - ld-taytrang-garnier - 2026-05-31 - buy
 │            │                  │        └── hành động
 │            │                  └── ngày
 │            └── SKU chính của đơn
 └── nhãn bộ dữ liệu seed
```

Điểm mấu chốt: id này **suy ra được từ chính sự việc** (tất định), nên gửi lại lần thứ hai vẫn ra **đúng chuỗi
đó**. Bảng `raw_events` có khoá chính `(project_id, event_id)` và ingest ghi bằng
**`ON CONFLICT DO NOTHING`** (`ingest.py:231`):

| Lần gửi | Kết quả |
|---|---|
| Lần 1 | ghi 1 dòng → đếm vào `accepted` |
| Lần 2, 3, … cùng `event_id` | **không ghi gì** → đếm vào `deduped` |

Nhờ vậy shop cứ **thử lại thoải mái khi mạng lỗi** mà không sợ nhân đôi doanh số.

> ⛔ **Lỗi tích hợp phổ biến nhất ở chỗ này:** người gửi đặt `event_id` **ngẫu nhiên** (kiểu `uuid4()` mới mỗi
> lần gửi). Khi đó lần thử lại tạo ra một id khác ⇒ hệ coi là sự việc mới ⇒ **doanh số nhân đôi**, âm thầm,
> không có lỗi nào nổ ra. Quy tắc: **`event_id` phải suy ra được từ sự việc, không được sinh ngẫu nhiên.**

Đáp lại, ingest trả về **thành công một phần** chứ không phải được-ăn-cả-ngã-về-không:

```json
{ "accepted": 3, "deduped": 1, "skipped": 0, "errors": [ { "index": 4, "code": "...", "message": "..." } ] }
```

- `accepted` — ghi mới · `deduped` — đã có rồi · `skipped` — loại event này service khác lo
- `errors` — sai phong bì, **có chỉ số dòng** để biết cái nào hỏng; sai ruột thì rơi vào bảng `dead_events`
  (không mất, mổ xẻ được sau).

#### Đủ 13 loại event — ruột là gì, ai ăn, dùng làm gì

> ⚠ **Câu hỏi của học viên: "phong bì nói `event_type` có 13 giá trị, sao bảng payload chỉ liệt kê 5?"**
> Vì bảng trước chỉ lọc **5 loại mà BT03 nhận**. Dưới đây là **đủ 13**, đúng theo `PAYLOAD_MODELS`
> (`libs/common/contracts/events.py:130`) — enum này **đóng**, gửi loại thứ 14 là bị từ chối ngay ở cửa.

**Nhóm 1 — HÀNH VI NGƯỜI DÙNG** (chỉ BT01 ăn; đây là thứ dạy máy biết cái gì đáng hiện lên trước)

| `event_type` | Ruột `payload` | Ràng buộc | Dùng làm gì (đọc từ code) |
|---|---|---|---|
| `product.viewed` | `product_id` | | tín hiệu phổ biến; hồ sơ người dùng |
| `product.clicked` | `product_id`, `position`, `source` | `position ≥ 1`; `source` chỉ `"search"` hoặc `"reco"` | học xếp hạng (LTR), bandit, hồ sơ người dùng — `learning_jobs.py`, `bandit.py`, `user_profile.py` |
| `cart.added` | `product_id`, `qty` | `qty ≥ 0` | tín hiệu ý định mạnh hơn click — `reco.py`, `learning_jobs.py` |
| `review.submitted` | `product_id`, `rating` (+ `review_id`, `review_text`) | `1 ≤ rating ≤ 5`; text ≤ 2000 ký tự | tín hiệu chất lượng — `learning_jobs.py` |

`position` và `source` tồn tại vì một cú click ở **vị trí 1** không cùng giá trị với click ở **vị trí 20**:
không ghi vị trí thì không khử được thiên lệch vị trí, và mô hình sẽ học nhầm "cái nào tôi để lên đầu thì cái
đó tốt".

**Nhóm 2 — GIAO DỊCH** (cả ba bài toán cùng ăn)

| `event_type` | Ruột `payload` | Ràng buộc | Dùng làm gì |
|---|---|---|---|
| `purchase.completed` | `order_ref`, `items[]` | mỗi item `product_id` · `qty ≥ 0` · `unit_price` **int** ≥ 0; `items` ≥ 1 dòng | **xương sống**: cầu cho BT03, doanh thu cho BT02, tín hiệu mua cho BT01 |
| `order.returned` | `order_ref`, `items[]` (+ `reason` ≤ 256 ký tự) | cùng hình dạng `items` | trừ vào cầu **ngày trả** (BT03), điều chỉnh doanh thu (BT02) |

**Nhóm 3 — TRẠNG THÁI HÀNG HOÁ**

| `event_type` | Ruột `payload` | Ràng buộc | Dùng làm gì |
|---|---|---|---|
| `stock.out` | `product_id` | | **BT03**: bật cờ `stockout` → bù cầu bị che (mục E). **BT01**: đặt `products.availability='OUT_OF_STOCK'` (`state_apply.py:11`) — hết hàng thì đừng hiện lên đầu |
| `stock.level` | `product_id`, `on_hand_qty` | `qty ≥ 0` | **BT01**: `on_hand_qty ≤ 0` cũng coi là hết hàng (`state_apply.py:12`). **BT02**: tồn kho để tính nhập bao nhiêu. BT03 **không** nhận |
| `price.changed` | `product_id`, `new_price` (+ `old_price`) | giá là **int** ≥ 0 | **BT03**: cột `price`. **BT02**: học co giãn giá. **BT01**: cập nhật `products.price` (`state_apply.py:14`) |
| `cost.recorded` | `product_id`, `unit_cost`, `qty` (+ `supplier_ref`) | tiền **int** ≥ 0 | **BT02**: giá vốn → biên lãi → chốt chặn *bán-dưới-vốn*. BT03 **không** nhận (dự báo cầu không cần biết vốn) |

**Nhóm 4 — THƯƠNG MẠI & VÒNG PHẢN HỒI**

| `event_type` | Ruột `payload` | Ràng buộc | Dùng làm gì |
|---|---|---|---|
| `promo.scheduled` | `product_ids[]`, `discount_pct`, `start`, `end` | `0 ≤ discount_pct ≤ 100`; **`end` phải sau `start`** | **BT03**: học hệ số uplift `k`, tô `promo_pct` cho từng ngày. **BT02**: chốt chặn khuyến mãi. Loại **duy nhất** mang một **khoảng thời gian** thay vì một thời điểm |
| `competitor.price` | `product_id`, `competitor_price` (+ `competitor_ref`) | giá **int** ≥ 0 | **BT02**: giá đối thủ vào bài toán định giá — `state_rollup.py`, `decisions_run.py` |
| `decision.feedback` | `decision_id`, `action` (+ `outcome_note`) | `action` chỉ `"accepted"` hoặc `"dismissed"` | **vòng phản hồi**: người ta có nghe khuyến nghị của máy không, và kết quả ra sao |

`decision.feedback` là loại đặc biệt: mọi loại khác kể chuyện **cửa hàng**, riêng nó kể chuyện **về chính
miniAI** — máy khuyên gì và người có nghe không. Không có nó thì hệ không bao giờ biết mình khuyên đúng hay sai.

#### 13 khuôn mẫu ĐẦY ĐỦ — mỗi loại một tình huống thật

Bảng ở trên nói *có những trường gì*. Phần này cho thấy **thật sự gửi đi trông ra sao** — đủ cả hai lớp, đúng
thứ shop bắn lên. **12/13 khuôn dưới đây là dòng nguyên trạng lấy từ DB** (`raw_events` của 3 service, đọc lúc
03:40 ngày 2026-08-10); riêng loại cuối chưa từng có dòng nào nên được đánh dấu rõ là **khuôn mẫu**.

Nhắc lại lớp ngoài để khỏi phải lật lại: mọi khuôn đều bọc trong `{"events": [ … ]}`, và mọi phong bì đều có
`event_id` · `event_type` · `event_time` · `user_pseudo_id` (+ `schema_version` mặc định `"1.0"`).

---

##### Nhóm 1 — HÀNH VI NGƯỜI DÙNG

**1. `product.viewed`** — *"có người mở xem trang nước tẩy trang Garnier"*

```json
{ "events": [ {
  "event_id": "ds8-ld-taytrang-garnier-2026-04-08-v11",
  "event_type": "product.viewed",
  "event_time": "2026-04-08T08:09:07+00:00",
  "user_pseudo_id": "u691",
  "schema_version": "1.0",
  "payload": { "product_id": "ld-taytrang-garnier" }
} ] }
```
Ruột đơn giản nhất trong 13 loại: đúng một trường. Xem là tín hiệu **yếu nhất** — nhiều lượt xem mà không ai
mua là dấu hiệu tiêu đề hấp dẫn nhưng hàng không hợp.

**2. `product.clicked`** — *"người ta bấm vào CeraVe ở vị trí 10 của khối gợi ý"*

```json
{ "events": [ {
  "event_id": "ds8-ld-srm-cerave-2026-04-08-c1",
  "event_type": "product.clicked",
  "event_time": "2026-04-08T08:10:55+00:00",
  "user_pseudo_id": "u140",
  "schema_version": "1.0",
  "payload": { "product_id": "ld-srm-cerave", "position": 10, "source": "reco" }
} ] }
```
`position: 10` và `source: "reco"` là hai trường **đắt giá nhất** ở đây: bấm được ở tận vị trí 10 nghĩa là món
đó thực sự hấp dẫn (người ta phải cuộn xuống mới thấy). Thiếu `position` thì mô hình học nhầm *"cái gì tôi để
lên đầu thì cái đó tốt"*. `source` chỉ nhận `"search"` hoặc `"reco"` — để chấm điểm hai cỗ máy riêng biệt.

**3. `cart.added`** — *"bỏ 1 chai Garnier vào giỏ"*

```json
{ "events": [ {
  "event_id": "ds8-ld-taytrang-garnier-2026-04-08-k1",
  "event_type": "cart.added",
  "event_time": "2026-04-08T08:16:11+00:00",
  "user_pseudo_id": "u463",
  "schema_version": "1.0",
  "payload": { "product_id": "ld-taytrang-garnier", "qty": 1.0 }
} ] }
```
Ý định mạnh hơn click nhưng **chưa phải doanh thu** — bỏ giỏ rồi rời đi là chuyện thường. Vì thế `cart.added`
nuôi gợi ý (BT01) nhưng **không** được tính là cầu (BT03 không nhận loại này).

**4. `review.submitted`** — *"khách chấm 4,5 sao cho ốp lưng iPhone 15"*

```json
{ "events": [ {
  "event_id": "ds8-dt-oplung-ip15-2026-04-08-rev11",
  "event_type": "review.submitted",
  "event_time": "2026-04-08T15:31:21+00:00",
  "user_pseudo_id": "u189",
  "schema_version": "1.0",
  "payload": {
    "product_id": "dt-oplung-ip15",
    "rating": 4.5,
    "review_id": "rv-dt-oplung-ip15-11",
    "review_text": "Chất lượng tuyệt vời, gia đình rất thích."
  }
} ] }
```
`rating` bắt buộc và bị kẹp trong `[1 ; 5]` — gửi 0 sao hay 6 sao là bị từ chối. `review_text` tuỳ chọn, tối đa
2000 ký tự.

---

##### Nhóm 2 — GIAO DỊCH

**5. `purchase.completed`** — đơn 11:29 đã mổ ở trên; đây là bản đầy đủ, để ngay đây cho tiện đối chiếu

```json
{ "events": [ {
  "event_id": "ds8-ld-taytrang-garnier-2026-05-31-buy",
  "event_type": "purchase.completed",
  "event_time": "2026-05-31T11:29:24+00:00",
  "user_pseudo_id": "u124",
  "schema_version": "1.0",
  "payload": {
    "order_ref": "ds-ord-ld-taytrang-garnier-2026-05-31",
    "items": [
      { "product_id": "ld-taytrang-garnier", "qty": 4, "unit_price": 156000 },
      { "product_id": "ld-kcn-anessa",       "qty": 1, "unit_price": 553000 },
      { "product_id": "ld-srm-cerave",       "qty": 1, "unit_price": 355000 }
    ]
  }
} ] }
```

**6. `order.returned`** — *"khách trả 2 sản phẩm, có ghi lý do"*

```json
{ "events": [ {
  "event_id": "hoc-ev-return-001",
  "event_type": "order.returned",
  "event_time": "2026-08-06T07:38:56+00:00",
  "user_pseudo_id": "hoc-user-1",
  "schema_version": "1.0",
  "payload": {
    "order_ref": "hoc-order-001",
    "items": [ { "product_id": "hoc-sp-01", "qty": 2, "unit_price": 250000 } ],
    "reason": "don tap huan - trung hoa du lieu training truoc demo"
  }
} ] }
```
Cùng hình dạng `items` như đơn mua — cố ý, để tái dùng đúng một bộ kiểm. `reason` tuỳ chọn (≤256 ký tự).
Nhớ luật ở mục D: hàng trả trừ vào **ngày trả**, không phải ngày mua.

---

##### Nhóm 3 — TRẠNG THÁI HÀNG HOÁ

**7. `stock.out`** — *"áo thun nữ form rộng hết hàng"*

```json
{ "events": [ {
  "event_id": "ds8-tt-aothun-nu-form-2026-04-13-so",
  "event_type": "stock.out",
  "event_time": "2026-04-13T15:40:14+00:00",
  "user_pseudo_id": "u321",
  "schema_version": "1.0",
  "payload": { "product_id": "tt-aothun-nu-form" }
} ] }
```
Ruột chỉ một trường, nhưng đây là loại **có ảnh hưởng lớn nhất trên mỗi byte**: nó bật cờ `stockout` để BT03
biết con số bán ra hôm đó **bị tồn kho che**, và cho BT01 hạ món hàng xuống khỏi kết quả.

**8. `stock.level`** — *"còn 10 cái trong kho"*

```json
{ "events": [ {
  "event_id": "fb17ad506ef944a5",
  "event_type": "stock.level",
  "event_time": "2026-08-01T10:00:00+00:00",
  "user_pseudo_id": "u_test",
  "schema_version": "1.0",
  "payload": { "product_id": "sku-1", "on_hand_qty": 10 }
} ] }
```
Khác `stock.out` ở chỗ nó nói **còn bao nhiêu** chứ không chỉ *"hết rồi"*. BT01 coi `on_hand_qty ≤ 0` cũng là
hết hàng. **BT03 không nhận loại này** — dự báo cầu không được phép nhìn tồn kho, nếu không sẽ học thành dự
báo *doanh số* (thứ đã bị tồn kho chặn) thay vì dự báo *cầu*.

**9. `price.changed`** — *"iPhone 15 128GB đổi giá niêm yết"*

```json
{ "events": [ {
  "event_id": "ds8-dt-iphone15-128-2026-04-08-price0",
  "event_type": "price.changed",
  "event_time": "2026-04-08T00:01:00+00:00",
  "user_pseudo_id": "seed",
  "schema_version": "1.0",
  "payload": { "product_id": "dt-iphone15-128", "new_price": 16348000 }
} ] }
```
`16348000` = 16.348.000đ, kiểu **`int`**, không dấu chấm không đơn vị. `old_price` tuỳ chọn (dòng thật này
không có). Chú ý `user_pseudo_id: "seed"` — trường này bắt buộc, nên **việc do hệ thống làm** vẫn phải khai
một danh tính; ở đây quy ước là `"seed"`.

**10. `cost.recorded`** — *"nhập 10 cái, giá vốn 100đ/cái"*

```json
{ "events": [ {
  "event_id": "evt_cost_0_3f52f2",
  "event_type": "cost.recorded",
  "event_time": "2026-08-01T00:00:00+00:00",
  "user_pseudo_id": "u_test",
  "schema_version": "1.0",
  "payload": { "product_id": "prod_903f1abb", "unit_cost": 100, "qty": 10 }
} ] }
```
Đây là loại **BT02 sống chết phải có**: không biết vốn thì không biết lãi, và chốt chặn *bán-dưới-vốn* không
có gì để so. `supplier_ref` tuỳ chọn. **BT03 không nhận** — dự báo cầu không quan tâm mua vào bao nhiêu.

---

##### Nhóm 4 — THƯƠNG MẠI & VÒNG PHẢN HỒI

**11. `promo.scheduled`** — *"phấn nước Clio giảm 21% từ 11/04 đến hết 15/04"*

```json
{ "events": [ {
  "event_id": "ds8-ld-phannuoc-clio-2026-04-11-promo",
  "event_type": "promo.scheduled",
  "event_time": "2026-04-11T00:05:00+00:00",
  "user_pseudo_id": "seed",
  "schema_version": "1.0",
  "payload": {
    "product_ids": ["ld-phannuoc-clio"],
    "discount_pct": 21.0,
    "start": "2026-04-11T00:00:00Z",
    "end":   "2026-04-15T23:59:59Z"
  }
} ] }
```
**Loại duy nhất mang một KHOẢNG thời gian.** Ba khác biệt so với mọi loại còn lại: (a) `product_ids` là **mảng**
— một đợt sale áp cho nhiều SKU; (b) có `start`/`end`, và `end` **phải** sau `start`; (c) `event_time` (lúc
*khai báo*) khác hoàn toàn `start` (lúc *có hiệu lực*). Rollup phải "tô" `discount_pct` ra **từng ngày** trong
khoảng — ở đây là 5 ngày 11→15/04. Nhiều đợt chồng nhau thì lấy **giảm sâu nhất**.

**12. `competitor.price`** — *"đối thủ trên Shopee bán 170.000đ"*

```json
{ "events": [ {
  "event_id": "cp-comp-1",
  "event_type": "competitor.price",
  "event_time": "2026-08-04T09:00:00+00:00",
  "user_pseudo_id": "probe",
  "schema_version": "1.0",
  "payload": {
    "product_id": "cp-sku-1",
    "competitor_price": 170000,
    "competitor_ref": "shopee-probe"
  }
} ] }
```
`competitor_ref` tuỳ chọn nhưng nên có — để biết giá đó của sàn nào. `user_pseudo_id: "probe"` lại là một
"danh tính máy": dữ liệu này do robot dò giá sinh ra, không phải người dùng.

**13. `decision.feedback`** — *"người mua hàng ĐỒNG Ý với khuyến nghị của máy"*

> ⚠ **Khuôn mẫu, KHÔNG phải dòng thật.** Đây là loại **duy nhất trong 13 chưa có dòng nào** trong cả ba DB
> (đếm lúc 03:40 ngày 10/08 = 0). Không phải hệ thiếu chức năng — vòng phản hồi đã nối, nhưng cần quyết định
> đủ 30 ngày tuổi mới có dòng hợp lệ đầu tiên (nợ có tên trong DB: `T-OUTCOME-30D`, dự kiến ~09/2026).
> Vì chưa có số thật nên khuôn dưới đây dựng **theo đúng hợp đồng** `DecisionFeedbackPayload`, không phải đo được.

```json
{ "events": [ {
  "event_id": "<mã do người gửi đặt, tất định>",
  "event_type": "decision.feedback",
  "event_time": "2026-08-10T09:00:00+00:00",
  "user_pseudo_id": "buyer-01",
  "schema_version": "1.0",
  "payload": {
    "decision_id": "<id của khuyến nghị mà máy đã đưa ra>",
    "action": "accepted",
    "outcome_note": "nhap 5 thung theo goi y, het hang truoc cuoi tuan"
  }
} ] }
```
`action` là enum đóng, chỉ `"accepted"` hoặc `"dismissed"`. Đây là loại **kể chuyện về chính miniAI**: máy
khuyên gì và người có nghe không. Không có nó thì hệ không bao giờ tự biết mình khuyên đúng hay sai.

---

##### Ba nhận xét khi nhìn cả 13 khuôn cạnh nhau

1. **Phong bì luôn giống hệt nhau, chỉ ruột đổi.** Đó là lý do một cửa `POST /v1/events:ingest` phục vụ được
   cả 13 loại và cả 3 bài toán — thêm loại thứ 14 chỉ cần thêm một model ruột, không đụng gì tới cửa.
2. **`user_pseudo_id` bắt buộc kể cả khi không có người thật.** Việc do máy làm vẫn phải khai danh tính —
   dữ liệu thật dùng `"seed"` (dữ liệu dựng sẵn), `"probe"` (robot dò giá), `"u_test"` (bộ kiểm). Trường này
   luôn trả lời được câu *"ai gây ra dòng này?"*.
3. **Trường bắt buộc rất ít, trường tuỳ chọn khá nhiều.** Chủ ý: hạ rào cho khách tích hợp — bắn được cái tối
   thiểu là đã dùng được ngay, `session_id` · `review_text` · `old_price` · `supplier_ref` · `competitor_ref`
   bổ sung sau mà không phá dữ liệu cũ.

#### Ba điều rút ra từ bảng 13 loại

1. **`unit_price` · `new_price` · `unit_cost` · `competitor_price` đều khai kiểu `int`** ngay tại hợp đồng —
   tiền vào hệ đã là **số nguyên (đồng)** từ ngoài cửa, chứ không phải đến `demand_daily` mới ép kiểu. Đây là
   bất biến tiền tệ, chặn từ cổng.
2. **Enum đóng + ruột kiểm theo loại.** Sai loại → từ chối ở cửa. Đúng loại nhưng ruột sai (thiếu `qty`, bịa
   tên cột) → rơi vào bảng `dead_events`, **không mất**, mổ xẻ được sau. Không có đường nào để dữ liệu rác
   lặng lẽ chui vào `demand_daily`.
3. **BT03 chỉ ăn 5/13.** Ba loại hành vi và `review.submitted` là chuyện của BT01; `stock.level`,
   `cost.recorded`, `competitor.price`, `decision.feedback` là chuyện của BT02. Cùng một cửa, ai việc nấy —
   nên `skipped` là hoạt động bình thường, không phải lỗi.

#### Gửi thật trông như thế nào — hai header bắt buộc

```bash
curl -X POST http://localhost:16023/v1/events:ingest \
  -H "Authorization: Bearer <API_KEY>" \
  -H "X-Project-Id: demoshop" \
  -H "Content-Type: application/json" \
  -d '{ "events": [ { "event_id": "...", "event_type": "purchase.completed",
                      "event_time": "2026-05-31T11:29:24+00:00",
                      "user_pseudo_id": "u124",
                      "payload": { "order_ref": "...", "items": [ ... ] } } ] }'
```

Hai header, thiếu cái nào cũng **401 `UNAUTHENTICATED`** (`main.py:356-370`):

| Header | Vai trò |
|---|---|
| `Authorization: Bearer <key>` | chứng minh anh là ai |
| `X-Project-Id: <shop>` | khai anh đang thao tác trên shop nào |

Vì sao cần **cả hai** chứ không chỉ mỗi key? Vì key và project phải **khớp nhau**: cầm key của shop A mà khai
`X-Project-Id: B` thì bị chặn. Đây là lớp khoá nằm trước cả RLS trong Postgres — hai lớp độc lập, cùng bảo vệ
một thứ: **shop này không bao giờ thấy dữ liệu shop kia**.

Mỗi response đều mang `X-Request-ID` (`main.py:259`) — có sự cố thì dò log theo mã đó.

#### Vì sao có `skipped` — mỗi service chỉ nhận phần việc của mình

13 loại event dùng chung một cửa, nhưng **mỗi service chỉ nhận loại nó cần**
(`libs/common/contracts/routing.py:9`). Riêng **forecast (BT03) nhận đúng 5 loại**:

| Loại event | smartsearch (BT01) | decision (BT02) | **forecast (BT03)** |
|---|---|---|---|
| `purchase.completed` | ✅ | ✅ | ✅ |
| `stock.out` | ✅ | ✅ | ✅ |
| `price.changed` | ✅ | ✅ | ✅ |
| `promo.scheduled` | ✅ | ✅ | ✅ |
| `order.returned` | — | ✅ | ✅ |
| `stock.level` | ✅ | ✅ | — |
| `cost.recorded` | — | ✅ | — |
| `decision.feedback` · `competitor.price` | — | ✅ | — |
| `product.viewed` · `product.clicked` · `cart.added` · `review.submitted` | ✅ | — | — |

Bắn `product.viewed` vào cửa của forecast thì **không phải lỗi** — nó chỉ được đếm vào `skipped`, vì hành vi
xem/click là chuyện của BT01. Đúng 5 loại ở cột phải chính là 5 loại đã học ở mục C.

> **Đọc ra một tính chất kiến trúc:** khách hàng bắn **một luồng event duy nhất** cho cả ba bài toán, mỗi
> service tự nhặt phần của mình. Nếu mỗi bài đòi một định dạng nhập riêng thì chi phí tích hợp của khách
> nhân ba — và họ sẽ không mua.

---

miniAI **không tính toán gì** ở bước này: kiểm định dạng → tra API key để biết là shop nào → cất nguyên văn →
trả lời ngay. Chú ý *"lẻ tẻ"* hiện ra rất rõ: 5 chai lúc 10 giờ, rồi 1-1-1 rải rác trưa. Không nhịp nào cả.

> **Vì sao cửa vào không được phép tính toán?** Vì nó phải chịu được lúc đông khách nhất. Nếu mỗi lần bán hàng
> mà hệ thống dừng lại dự báo thì giờ cao điểm sẽ nghẽn ngay tại quầy thu ngân. Nguyên tắc: **cửa vào phải nhẹ**.

---

#### TẦNG 2 — đầu giờ kế tiếp, "người thư ký" gom sổ

Cứ mỗi 3.600 giây, job `rollup` thức dậy và **cuộn** mọi sự việc thành **một dòng cho mỗi SKU mỗi ngày**.

##### Trước hết: bảng `demand_daily` trông như thế nào?

> ⚠ **Câu hỏi của học viên: "mục này là gộp data vào bảng `demand_daily` đúng không, cấu trúc bảng thế nào?"**
> Đúng — tầng 2 chỉ có đúng một việc: **ghi vào `demand_daily`**. Đây là cấu trúc thật, đọc bằng
> `\d demand_daily` trên DB đang chạy:

```
                Table "public.demand_daily"
     Column     |  Type   | Nullable | Default
----------------+---------+----------+---------
 project_id     | text    | not null |            ← shop nào
 product_id     | text    | not null |            ← SKU nào
 day            | date    | not null |            ← ngày nào
 units_sold     | numeric | not null | 0          ← bán ra (đã trừ hàng trả)
 stockout       | boolean | not null | false      ← ngày đó có hết hàng không
 price          | bigint  |          |            ← giá hiệu lực, ĐỒNG, số nguyên
 promo_pct      | numeric | not null | 0          ← % giảm giá
 adjusted_units | numeric |          |            ← CẦU đã bù ngày hết hàng
Indexes:
    "demand_daily_pkey" PRIMARY KEY, btree (project_id, product_id, day)
Policies:
    POLICY "tenant_isolation" USING (project_id = current_setting('app.project_id', true))
```

Hình dung cho dễ: bảng này là **một tờ lịch cho mỗi SKU**. Mỗi ô là một ngày, ô nào cũng phải có (kể cả ngày
bán 0), và trong ô ghi 5 con số — bán bao nhiêu · có hết hàng không · giá bao nhiêu · giảm bao nhiêu % · cầu
thật ước tính bao nhiêu.

##### Mổ TỪNG CỘT — nghĩa là gì, ví dụ thật, và cạm bẫy

> ⚠ **Câu hỏi của học viên: "các trường field ghi tóm tắt quá, giải thích từng trường và ví dụ được không?"**
> Dưới đây là từng cột một. Mọi ví dụ và mọi con số đếm được đều **truy thẳng từ DB** lúc 03:55 ngày 12/08.

---

**1. `project_id` · kiểu `text` · BẮT BUỘC**

Shop nào sở hữu dòng này. Trong dữ liệu thật đang có 8 giá trị: `demoshop` (shop demo tiếng Việt), `p1`,
`simworld1..4` (thế giới mô phỏng để chấm điểm), `bulktest`, `seedtest`.

- **Không phải cột thường** — nó là **cột cách ly**. Cùng một `product_id` ở hai shop khác nhau là **hai mặt
  hàng hoàn toàn khác nhau**, không bao giờ được lẫn số.
- Khách **không tự khai** cột này; nó suy ra từ API key (mục B). Kể cả khách có cố gửi `project_id` khác thì
  cũng vô ích.
- Đây là cột mà `POLICY tenant_isolation` bám vào (giải thích ở cuối mục).

---

**2. `product_id` · kiểu `text` · BẮT BUỘC**

Mã SKU **do shop tự đặt**, miniAI không sinh. Ví dụ thật trong `demoshop`: `ld-srm-cerave` ·
`bh-cafe-g7` · `dt-iphone15-128` · `tt-aothun-nu-form`. Shop `p1` thì đặt kiểu máy: `p_a88a66e84614`.

- miniAI **không hiểu ý nghĩa** của chuỗi này — với nó `ld-srm-cerave` chỉ là một nhãn. Cách đặt tên đẹp là
  để **người** đọc, không phải để máy suy luận.
- Cặp `(project_id, product_id)` là danh tính đầy đủ của một mặt hàng.

---

**3. `day` · kiểu `date` · BẮT BUỘC**

Ngày lịch — **chỉ ngày, không có giờ, không có múi giờ**. `2026-05-31`, hết.

- Đây là chỗ **đổi đơn vị**: `raw_events.event_time` chính xác tới giây và có múi giờ; sang đây bị "nén" thành
  một ngày. Sau bước này, mọi thứ trong BT03 chạy theo nhịp NGÀY.
- Chính vì nén nên câu hỏi *"23:59:59 thuộc ngày nào"* là chuyện có thật — cảnh báo §11.5 về "kiểu ngày" của
  tenant `p1` sinh ra từ đây.

---

**4. `units_sold` · kiểu `numeric` · BẮT BUỘC · mặc định `0`**

Số lượng **bán ra** trong ngày, đã trừ hàng trả.

```
ld-srm-cerave · 2026-05-31 · units_sold = 8      ← 5 + 1 + 1 + 1 (bốn đơn)
```

Ba điều dễ hiểu nhầm:

- **`numeric` chứ không phải số nguyên.** Số lẻ là hợp lệ và **có thật**: đếm được **301 dòng** có
  `units_sold` không nguyên, ví dụ tenant mô phỏng `m17d-wire-async` ngày 12/06 có
  `units_sold = 6,87196683917459…`. Lý do: hợp đồng khai `qty: float` (bán theo cân, theo lít, hoặc dữ liệu mô
  phỏng), và `numeric` là kiểu **thập phân chính xác** — không phải `float` — nên cộng hàng triệu dòng không
  trôi số lẻ.
- **Mặc định `0`, không phải NULL.** Ngày không bán gì ⇒ `0`. Đây là con số **thật**, không phải "không biết".
- **Đã trừ hàng trả, và bị KẸP ở 0** — xem ngay mục dưới, vì chỗ này nghe vô lý cho tới khi thấy code.

---

**4b. Vì sao "trả nhiều hơn bán" lại ghi `0` chứ không ghi số âm?**

> ⚠ **Câu hỏi của học viên: "hàng trả trừ đi hàng bán, nếu trả nhiều hơn thì ra số âm mà, sao lại ghi 0?"**
> Đúng — phép trừ **thật sự ra số âm**. Nhưng có một bước nữa sau phép trừ mà bản trước tôi viết gộp thành một
> dòng nên đọc ra vô lý. Đây là toàn bộ sự thật, ba bước.

**Bước 1 — cộng hàng bán** (`rollup.py:99`):
```python
units[key] += qty        # mỗi dòng hàng trong purchase.completed
```

**Bước 2 — trừ hàng trả** (`rollup.py:111`):
```python
units[key] -= qty        # mỗi dòng hàng trong order.returned
```
Tới đây `units[key]` **hoàn toàn có thể âm**. Không có gì chặn.

**Bước 3 — lúc GHI XUỐNG BẢNG mới kẹp** (`rollup.py:174`):
```python
u = max(0.0, units.get(key, 0.0))     # ← số âm bị nâng lên 0 ở ĐÂY
```

Nên câu đúng phải là: **phép trừ ra số âm thật, nhưng số âm không bao giờ được ghi vào bảng.** Nó bị chặn ở
cửa ghi, không phải ở phép tính.

**Ca thật, đọc được ngay trong DB:**

Shop `p1`, SKU `px`, ngày **2026-08-04** — có **20 event `order.returned`** (mỗi cái `qty: 1`) và **không có
lượt mua nào**:

```
raw_events   :  bán 0  ·  trả 20   ⇒  units[key] = 0 − 20 = −20
demand_daily :  units_sold = 0                    ← đã kẹp
```

| ngày | bán | trả | ròng (trước kẹp) | `units_sold` ghi vào bảng |
|---|---|---|---|---|
| 2026-08-02 | 0 | 4 | **−4** | **0** |
| 2026-08-03 | 0 | 14 | **−14** | **0** |
| 2026-08-04 | 0 | 20 | **−20** | **0** |
| 2026-08-05 | 0 | 20 | **−20** | **0** |

**Vì sao phải kẹp — bốn lý do, đều là hỏng thật nếu không kẹp:**

1. **"Cầu âm" không có nghĩa.** Cột này trả lời câu *"ngày đó thị trường muốn mua bao nhiêu cái"*. Câu trả lời
   nhỏ nhất có thể có là **không ai muốn mua = 0**. Không tồn tại "âm 20 người muốn mua".
2. **Toán học vỡ ngay.** Nhiều model trong thang lấy `log` của chuỗi (ETS, Theta), tính hệ số biến thiên, hoặc
   ước lượng phân phối đếm (Croston/NBD — vốn chỉ định nghĩa trên số **không âm**). `log(−20)` không tồn tại;
   phương sai tính ra số bịa; quantile trả về vô nghĩa. Một dòng âm làm **hỏng cả chuỗi của SKU đó**.
3. **Hàng trả là chuyện của NGÀY KHÁC.** 20 cái trả hôm 04/08 là hàng đã bán từ tuần trước. Trừ hết vào 04/08
   là gán hậu quả sai ngày. Nhưng dự án **cố ý chọn** trừ vào ngày trả (xem CỤM 1 §"LỆCH 2") vì phương án kia
   — trừ ngược vào ngày mua — sẽ **viết lại quá khứ** mà model đã học và đã ra quyết định trên đó.
4. **Nếu không kẹp thì phải kẹp ở chỗ khác.** Mỗi model lại tự xử số âm theo một kiểu ⇒ hai model ra hai con
   số khác nhau cho cùng một ngày. Kẹp **một lần, tại một chỗ** là rẻ nhất và dễ giải thích nhất.

**Cái gì bị mất, cái gì KHÔNG mất — đây mới là phần quan trọng:**

| | Có mất không? |
|---|---|
| Thông tin *"ngày 04/08 có 20 lượt trả hàng"* | **KHÔNG mất** — 20 dòng `order.returned` nằm nguyên trong `raw_events`, đọc lại bất cứ lúc nào |
| Mức độ âm (`−20` so với `−4`) | **MẤT** trong `demand_daily` — cả hai đều thành `0` |
| Khả năng phân tích tỷ lệ trả hàng | **KHÔNG mất** — tính thẳng từ `raw_events`, không cần cột này |

Nói cách khác: `demand_daily` **không phải sổ kế toán**, nó là **đầu vào cho model dự báo cầu**. Sổ gốc vẫn là
`raw_events` và không bị đụng tới. Đây là lý do kiến trúc giữ **cả hai tầng** thay vì chỉ giữ bảng đã gom.

> **Đánh đổi viết ra:** thà mất thông tin *"ngày này trả hàng nhiều"* trong bảng đã gom, còn hơn để một số âm
> chảy vào model và làm hỏng toàn bộ chuỗi. Ai cần số liệu trả hàng thì đọc `raw_events` — đúng nhà của nó.

📌 **Đính chính số dòng code:** tài liệu gốc `ALGO-FORECAST-BT03` (viết 07/08) ghi phép kẹp ở `rollup.py:172`.
Đọc lại code hôm nay là **`rollup.py:174`** — lệch 2 dòng vì bản vá `W-JOB-SCHEDULE-STATE-ANCHOR` (10/08) thêm
một dòng `import` vào đầu file. Bài học nhỏ nhưng thật: **số dòng là thứ hết hạn nhanh nhất trong mọi tài
liệu** — luôn kiểm lại bằng mắt trước khi tin.

---

**5. `stockout` · kiểu `boolean` · BẮT BUỘC · mặc định `false`**

Ngày đó SKU có **hết hàng** hay không. Bật lên khi trong ngày có ít nhất một event `stock.out`.

Đếm thật: **6.437 dòng** đang mang `stockout = true`.

- Đây là cột **nhỏ nhất mà nặng ký nhất**: chỉ 1 bit, nhưng nó là thứ phân biệt *"không ai muốn mua"* với
  *"muốn mua mà không có hàng"*.
- Không có mức độ: hết hàng 10 phút cuối ngày hay hết cả ngày đều là `true`. Muốn biết mức độ thì phải nhìn
  `stock.level` — nhưng BT03 cố ý **không nhận** loại đó.

---

**6. `price` · kiểu `bigint` · CÓ THỂ NULL**

Giá hiệu lực của một đơn vị hàng trong ngày, đơn vị **đồng**, **số nguyên**.

```
bh-cafe-g7 · price = 176000        →  đọc là 176.000đ
dt-iphone15-128 · new_price = 16348000  →  16.348.000đ
```

- **`bigint` chứ không phải `float`** — nếu để số thực, `176000` có thể thành `176000.00000000001`; cộng vài
  triệu dòng là lệch sổ. Bất biến tiền tệ, và nó được ép **từ ngoài cửa** (hợp đồng khai `unit_price: int`).
- Không có dấu chấm, không có ký hiệu tiền tệ, không có số thập phân. Đồng Việt Nam không có xu nên hợp.
- **Có thể NULL, và NULL rất phổ biến: 47.448 dòng.** Ví dụ thật: shop `p1`, SKU `p_a88a66e84614`, ngày
  04/07 → `price` trống. Nguyên nhân: shop đó **chưa từng bắn `price.changed` nào**, mà ngày đó cũng không
  có giao dịch để tính giá bình quân ⇒ không có gì để ghi. **Thà để trống còn hơn bịa số 0** — giá 0đ sẽ làm
  mô hình co giãn giá của BT02 điên loạn.

---

**7. `promo_pct` · kiểu `numeric` · BẮT BUỘC · mặc định `0`**

Phần trăm giảm giá đang áp cho SKU trong ngày. **Đơn vị là PHẦN TRĂM, không phải tỷ lệ**: `10` nghĩa là giảm
10%, không phải 0,1.

Ba dòng thật của `demoshop` ngày 11/08:

| product_id | units_sold | price | promo_pct |
|---|---|---|---|
| `gd-noichien-philips` | 2 | 2.116.000 | **15** |
| `mb-ta-bobby-m` | 5 | 277.000 | **12** |
| `bh-cafe-g7` | 3 | 176.000 | **10** |

Đếm thật: **6.718 dòng** đang có `promo_pct > 0`.

- Cột này **không đến từ giao dịch** mà đến từ event `promo.scheduled` — loại duy nhất mang một **khoảng**
  thời gian. Rollup "tô" `discount_pct` ra từng ngày trong `[start, end]`.
- Nhiều đợt chồng nhau ⇒ lấy **sâu nhất** (`max`), không cộng dồn — mổ kỹ ở mục 7b ngay dưới.
- Đây là đầu vào để BT03 học hệ số uplift `k` — *"giảm 1% thì bán tăng bao nhiêu"*.

---

**7b. Một SKU, một ngày, có 2 (hay 5) đợt khuyến mãi thì sao?**

> ⚠ **Câu hỏi của học viên: "nếu 1 SKU trong 1 ngày có sự thay đổi 2 lần thì thế nào?"**

Luật nằm gọn trong một dòng code (`rollup.py:131`):

```python
promo[key] = max(promo[key], discount_pct)
```

**LẤY SÂU NHẤT.** Không phải "cái sau đè cái trước", cũng không phải cộng dồn.

**Ca thật, đọc được ngay trong DB** — shop `bulktest`, SKU `sku-0015`, có **hai đợt chồng nhau**:

| Đợt | Giảm | Từ | Đến |
|---|---|---|---|
| A | **59,462 %** | 25/06 | 03/07 |
| B | **21,120 %** | 29/06 | 05/07 |

Vùng chồng lấn là **29/06 → 03/07** (5 ngày). Bảng `demand_daily` ghi thế này:

| ngày | đợt đang chạy | `promo_pct` trong bảng |
|---|---|---|
| 24/06 | — | `0` |
| 25/06 → 28/06 | chỉ A | `59,462` |
| **29/06 → 03/07** | **A và B cùng lúc** | **`59,462`** ← lấy sâu nhất, B bị bỏ qua |
| 04/07 → 05/07 | A hết, chỉ còn B | `21,120` |
| 06/07 | cả hai đã hết | `0` |

Để ý ngày 04/07: đợt sâu hơn hết hạn thì con số **tự tụt xuống** 21,120 — không "dính" lại. Vì `max` được tính
**cho từng ngày một**, không phải một lần cho cả đợt.

**Ba hệ quả phải nhớ:**

**(1) Không có khái niệm GIỜ.** `promo_pct` là con số của **cả ngày**. Code lấy `.date()` của `start` và `end`
(`rollup.py:125-126`) rồi lặp từng ngày, nên một đợt sale bắt đầu lúc **15:00** vẫn phủ **trọn ngày hôm đó**
trong bảng. Nghiệp vụ cần độ chính xác theo giờ thì cột này **không đáp ứng được** — giới hạn có chủ ý, vì cả
BT03 chạy theo nhịp ngày.

**(2) ⛔ KHÔNG có đường "huỷ" hay "sửa" khuyến mãi.** Trong 13 loại event **không có** `promo.cancelled` hay
`promo.updated`. Hệ quả trực tiếp:

> Shop khai nhầm **50%**, phát hiện ra, khai lại **10%** cho đúng ngày đó ⇒ bảng vẫn giữ **50%**.
> Lần khai sau **không đè** được lần khai trước, vì luật là `max` chứ không phải "mới nhất thắng".

Đây là cạm bẫy thật, và nó **im lặng** — không lỗi, không cảnh báo. Người vận hành sẽ thấy `promo_pct` không
chịu xuống mà không hiểu vì sao. Đường duy nhất hiện nay là sửa dưới DB, hoặc chờ đợt sai hết hạn.

**(3) Không cộng dồn — và cộng dồn cũng sai về kinh tế.** Hai đợt 20% và 30% cùng ngày ⇒ bảng ghi **30**,
không phải 50. Kể cả nếu muốn "chồng khuyến mãi thật" thì phép đúng cũng không phải phép cộng:

```
giảm 20% rồi giảm tiếp 30%  →  giá còn 0,8 × 0,7 = 0,56  →  giảm thực tế 44 %,  KHÔNG phải 50 %
```

**Vì sao chọn `max` mà không phải hai phương án kia?**

| Phương án | Vấn đề |
|---|---|
| **Cái sau đè cái trước** (last-write-wins) | Kết quả phụ thuộc **thứ tự đọc event**; hai đợt khai sát nhau thì thứ tự do DB quyết ⇒ **chạy lại có thể ra số khác** — phá tính tái lập, vốn là bất biến của cả tầng 2 |
| **Cộng dồn** | Sai về kinh tế (xem phép tính trên), và có thể vượt 100% ⇒ giá âm |
| **`max` (đang dùng)** | Tất định, không phụ thuộc thứ tự; và **khớp với thứ khách hàng thật sự nhìn thấy** — người mua luôn hưởng mức giảm tốt nhất đang có, chứ không cộng các mã lại |

Lý do cuối là lý do mạnh nhất: cột này tồn tại để trả lời *"ngày đó người mua nhìn thấy mức giảm bao nhiêu"* —
và người mua thấy **mức tốt nhất**, không thấy tổng.

**Nối sang việc BT03 dùng cột này làm gì.** `promo_pct` là đầu vào để học hệ số uplift `k` (*"giảm 1% thì bán
tăng bao nhiêu"*). Nếu một đợt khai nhầm quá sâu bị kẹt trong bảng, hệ sẽ thấy *"giảm 50% mà bán chẳng tăng
mấy"* ⇒ **`k` học bị thấp đi**, và mọi dự báo cho ngày sale tương lai hụt theo. Một dòng khai sai không chỉ sai
một ngày — nó lệch cả tham số của shop đó.

---

**8. `adjusted_units` · kiểu `numeric` · CÓ THỂ NULL (thực tế: chưa dòng nào NULL)**

**CẦU ước tính** sau khi bù cho ngày hết hàng. Đây là **cột mà mọi model đọc** — không phải `units_sold`.

Hai cột này trả lời **hai câu hỏi khác nhau**:

| Cột | Trả lời câu | Ai quan tâm |
|---|---|---|
| `units_sold` | *"Ngày đó **bán được** bao nhiêu?"* | kế toán, doanh thu |
| `adjusted_units` | *"Ngày đó thị trường **muốn mua** bao nhiêu?"* | **model dự báo** |

Hai câu chỉ khác nhau đúng vào **ngày hết hàng** — ngày mà tồn kho chặn bớt nhu cầu. Đếm thật: **4.669 dòng**
có `adjusted_units ≠ units_sold`; mọi dòng còn lại hai cột **bằng nhau y hệt**.

##### Công thức, đúng theo code (`rollup.py:218-228`)

```python
for r in rows:                              # rows đã sắp theo NGÀY tăng dần
    if r["stockout"]:
        if adjusted_prev:                                        # đã có ngày trước
            trailing = sum(adjusted_prev[-7:]) / len(adjusted_prev[-7:])
        else:                                                    # ngày đầu tiên của SKU
            trailing = 0.0
        adj = max(r["units_sold"], trailing)                     # ← NHÁNH HẾT HÀNG
    else:
        adj = r["units_sold"]                                    # ← NHÁNH BÌNH THƯỜNG
    adjusted_prev.append(adj)                                    # ← ĐỆ QUY: đẩy adj vào lịch sử
```

Đọc thành lời:

- Ngày **không** hết hàng ⇒ chép nguyên `units_sold`. Không đoán gì cả.
- Ngày **có** hết hàng ⇒ lấy **số lớn hơn** giữa *"số thật bán được"* và *"trung bình 7 ngày gần nhất"*.

##### Ca thật, tính tay từng bước — `ld-son-3ce` (son 3CE), shop `demoshop`

Đây là ca đẹp vì có **hai ngày hết hàng LIÊN TIẾP** (04/08 và 05/08):

| ngày | `units_sold` | `stockout` | `adjusted_units` |
|---|---|---|---|
| 26/07 | 1 | f | 1 |
| 27/07 | 3 | f | 3 |
| 28/07 | 2 | f | 2 |
| 29/07 | 1 | f | 1 |
| 30/07 | 3 | f | 3 |
| 31/07 | 1 | f | 1 |
| 01/08 | 1 | f | 1 |
| 02/08 | 2 | f | 2 |
| 03/08 | 1 | f | 1 |
| **04/08** | **0** | **t** | **1,571429** |
| **05/08** | **0** | **t** | **1,510204** |
| 06/08 | 1 | f | 1 |
| 07/08 | 2 | f | 2 |

**Ngày 04/08** — hết hàng, bán được 0. Bảy giá trị `adjusted` gần nhất là của 28/07 → 03/08:

```
[2, 1, 3, 1, 1, 2, 1]
trailing = (2+1+3+1+1+2+1) / 7 = 11 / 7 = 1,571428…
adj      = max(0 ; 1,571428…)   = 1,571429      ← đúng con số trong DB
```

**Ngày 05/08** — lại hết hàng. Bảy giá trị `adjusted` gần nhất là của 29/07 → **04/08**:

```
[1, 3, 1, 1, 2, 1, 1,571429]      ← phần tử cuối là adj của 04/08, KHÔNG phải units_sold=0
trailing = 10,571429 / 7 = 1,510204…
adj      = max(0 ; 1,510204…)     = 1,510204     ← đúng con số trong DB
```

##### ⭐ Điểm mấu chốt: đệ quy trên chính cột `adjusted`, không phải trên `units_sold`

Nhìn kỹ dòng cuối của phép tính ngày 05/08: phần tử thứ 7 là **`1,571429`** — tức giá trị **đã bù** của ngày
04/08, **không phải** `units_sold = 0` của ngày đó.

Nếu code lấy trung bình trên `units_sold` (tức dùng số `0` thật), kết quả sẽ là:

```
[1, 3, 1, 1, 2, 1, 0]  →  9 / 7 = 1,285714      ← THẤP HƠN 1,510204
```

Chênh lệch một ngày trông nhỏ, nhưng nó **tích luỹ**: hết hàng 5 ngày liên tiếp thì mỗi ngày lại kéo trung
bình xuống thêm, và cột "cầu" tụt dần về 0 — **đúng cái vòng xoáy mà nó sinh ra để chặn**:

```
hết hàng → bán 0 → cầu ước tính tụt → dự báo thấp → nhập ít → hết hàng sớm hơn → …
```

Đệ quy trên `adjusted` giữ mức nền **đứng yên** trong suốt đợt hết hàng, thay vì trôi xuống.

##### Bốn lựa chọn thiết kế trong công thức, và lý do từng cái

**(1) Vì sao `max()` mà không lấy thẳng `trailing`?**
Vì có ca hết hàng **vào cuối ngày**: bán được 15 cái rồi mới sạch kệ, trong khi trung bình 7 ngày chỉ 10. Nếu
lấy thẳng `trailing` thì **hạ số thật xuống** 10 — vứt mất thông tin có thật. `max` bảo đảm: **không bao giờ
ghi thấp hơn số đã bán được**.

**(2) Vì sao 7 ngày mà không phải 3 hay 30?**
7 ngày = **đúng một tuần**, nên trung bình phủ đủ cả thứ Hai lẫn Chủ nhật — không bị lệch vì rơi vào toàn ngày
cuối tuần. Đủ ngắn để bám mức gần đây, đủ dài để không giật theo một ngày bất thường.

**(3) Cửa sổ chưa đủ 7 ngày thì sao?**
`adjusted_prev[-7:]` lấy **tối đa** 7 phần tử, và mẫu số là `len()` của lát cắt đó — tức nếu SKU mới có 3 ngày
lịch sử thì chia cho **3**, không chia cứng cho 7. Không có chuyện pha loãng bằng số 0 ảo.

**(4) Ngày ĐẦU TIÊN của SKU mà đã hết hàng?**
`adjusted_prev` rỗng ⇒ `trailing = 0.0` ⇒ `adj = max(units_sold, 0) = units_sold`. Tức **không bù gì cả** —
đúng: chưa có lịch sử thì không có cơ sở nào để đoán, thà để nguyên số thật còn hơn bịa.

##### Giới hạn phải nói thẳng

Cột này là **ƯỚC LƯỢNG, không phải sự thật**. Nó giả định *"cầu hôm nay giống mức trung bình tuần vừa rồi"*.
Nếu cầu **thật sự sập** đúng vào đợt hết hàng (hết mốt, qua mùa, đối thủ phá giá), cột bù sẽ **giữ mức cao lâu
hơn thực tế** ⇒ dự báo hơi cao ⇒ nhập hơi dư.

Đây là đánh đổi có chủ ý: **nhập dư đắt, nhưng đứt hàng còn đắt hơn** (mất lãi + mất khách + vòng xoáy tự
xác nhận). Chọn sai theo hướng an toàn.

##### Về chuyện `NULL`

Kiểu cột cho phép `NULL`, nhưng **thực tế 0 dòng NULL** — vì rollup luôn chạy pass 2 và gán `adj` cho **mọi**
dòng trước khi ghi. Nên nếu có ngày anh thấy `adjusted_units IS NULL`, đó **không phải chuyện bình thường**:
nó nghĩa là dòng đó được ghi bởi một đường **không đi qua pass 2** — dấu hiệu rollup chết giữa chừng hoặc có
ai đó `INSERT` tay vào bảng. Đáng đi soi ngay.

##### Ai đọc cột này

`store/forecasts.py:26` — hàm nạp chuỗi lịch sử cho model **đọc `adjusted_units`**, không đọc `units_sold`.
Nghĩa là toàn bộ 9 model trong thang, backtest, và cả bước học hệ số promo `k` đều ăn cột này. `units_sold`
chỉ còn dùng để **đối chiếu và báo cáo**.

> Một câu đáng thuộc: **`units_sold` là quá khứ đã xảy ra; `adjusted_units` là quá khứ ĐÁNG LẼ đã xảy ra nếu
> kho không hết hàng.** Model học cái thứ hai.

---

##### Hai dòng cuối của schema — khoá chính và RLS

**`PRIMARY KEY (project_id, product_id, day)`**

Bộ ba này định nghĩa *"một dòng là gì"*: **một shop · một SKU · một ngày = đúng MỘT dòng, không bao giờ hai**.

Hệ quả nghe thì nhỏ mà rất lớn: rollup dùng `INSERT … ON CONFLICT (project_id, product_id, day) DO UPDATE`,
tức **gặp dòng cũ thì ghi đè chính nó**. Cho nên chạy rollup 1 lần hay 50 lần đều ra cùng kết quả — job chết
giữa chừng cứ chạy lại, không sợ nhân đôi số liệu.

**`POLICY tenant_isolation USING (project_id = current_setting('app.project_id', true))`**

Postgres **tự chèn** điều kiện `WHERE project_id = <shop đang đăng nhập>` vào **mọi** câu truy vấn chạm bảng
này. Lập trình viên quên viết `WHERE` thì thấy **0 dòng**, chứ không bao giờ thấy dữ liệu shop khác.

Đây là **lớp khoá thứ hai**, độc lập với lớp API key ở mục B. Hai lớp cùng bảo vệ một thứ: shop này không bao
giờ thấy dữ liệu shop kia. Muốn thủng thì phải thủng cả hai.

> ⚠ **Cạm bẫy khi tự kiểm tra RLS:** role `miniai` là **chủ bảng**, mà Postgres cho chủ bảng **bỏ qua RLS**.
> Nên gõ `psql -U miniai` rồi thấy đủ 8 tenant thì **không có nghĩa là RLS hỏng**. Muốn kiểm thật phải dùng
> role `miniai_app` — đúng role mà đường request của service dùng.

##### Bảng tra nhanh cả 8 cột

| Cột | Kiểu | NULL? | Đơn vị | Ví dụ thật | Đếm ca đặc biệt |
|---|---|---|---|---|---|
| `project_id` | text | không | — | `demoshop` | 8 tenant |
| `product_id` | text | không | — | `ld-srm-cerave` | — |
| `day` | date | không | ngày | `2026-05-31` | — |
| `units_sold` | numeric | không (mặc định 0) | cái | `8` | 301 dòng số lẻ |
| `stockout` | boolean | không (mặc định false) | — | `true` | 6.437 dòng `true` |
| `price` | bigint | **CÓ** | đồng | `176000` | **47.448 dòng NULL** |
| `promo_pct` | numeric | không (mặc định 0) | **%** | `15` | 6.718 dòng > 0 |
| `adjusted_units` | numeric | có (thực tế 0) | cái | `1,5102…` | 4.669 dòng ≠ `units_sold` |

##### Quá trình "gom theo SKU" — xem tận mắt bằng số thật

Lấy **hai SKU, hai ngày** để thấy đủ mọi phép gom cùng lúc.

**Bước 1 — nguyên liệu thô.** Tám dòng hàng có thật trong `raw_events` (đã mở mảng `items` ra):

| # | Lúc | `order_ref` | SKU | qty | unit_price |
|---|---|---|---|---|---|
| 1 | 05-31 10:04:41 | `ds-ord-ld-srm-cerave-2026-05-31` | `ld-srm-cerave` | 5 | 355.000 |
| 2 | 05-31 11:29:24 | `ds-ord-ld-taytrang-garnier-…` | `ld-kcn-anessa` | 1 | 553.000 |
| 3 | 05-31 11:29:24 | `ds-ord-ld-taytrang-garnier-…` | `ld-srm-cerave` | 1 | 355.000 |
| 4 | 05-31 11:41:42 | `ds-ord-ld-kcn-anessa-…` | `ld-kcn-anessa` | 7 | 553.000 |
| 5 | 05-31 11:41:42 | `ds-ord-ld-kcn-anessa-…` | `ld-srm-cerave` | 1 | 355.000 |
| 6 | 05-31 12:59:13 | `ds-ord-ld-toner-klairs-…` | `ld-srm-cerave` | 1 | 355.000 |
| 7 | 06-01 16:00:44 | `ds-ord-ld-kcn-anessa-2026-06-01` | `ld-kcn-anessa` | 3 | 553.000 |
| 8 | 06-01 16:22:12 | `ds-ord-ld-srm-cerave-2026-06-01` | `ld-srm-cerave` | 3 | 355.000 |

Để ý dòng 2-3 và 4-5: **cùng một giây, cùng một đơn**, nhưng là **hai SKU khác nhau** — vì một đơn là một giỏ
nhiều mặt hàng. Rollup **không quan tâm đơn nào**; nó chỉ quan tâm cặp **(SKU, ngày)**.

**Bước 2 — xếp vào ô.** Mỗi dòng hàng rơi vào đúng một ô `(SKU, ngày)`:

```
                     │  ngày 2026-05-31         │  ngày 2026-06-01
─────────────────────┼──────────────────────────┼──────────────────
 ld-srm-cerave       │  #1(5) #3(1) #5(1) #6(1) │  #8(3)
 ld-kcn-anessa       │  #2(1) #4(7)             │  #7(3)
```

**Bước 3 — cộng trong từng ô.**

```
ld-srm-cerave · 31/05 :  5 + 1 + 1 + 1  = 8
ld-kcn-anessa · 31/05 :  1 + 7          = 8
ld-srm-cerave · 01/06 :  3              = 3
ld-kcn-anessa · 01/06 :  3              = 3
```

**Bước 4 — kết quả thật trong `demand_daily`** (đọc thẳng từ DB, đối chiếu với phép cộng trên):

| product_id | day | units_sold | price | stockout | promo_pct | adjusted_units |
|---|---|---|---|---|---|---|
| `ld-kcn-anessa` | 2026-05-31 | **8** | 553.000 | f | 0 | 8 |
| `ld-srm-cerave` | 2026-05-31 | **8** | 355.000 | f | 0 | 8 |
| `ld-kcn-anessa` | 2026-06-01 | **3** | 553.000 | f | 0 | 3 |
| `ld-srm-cerave` | 2026-06-01 | **3** | 355.000 | f | 0 | 3 |

**8 dòng hàng thô → 4 dòng ngày.** Khớp từng con số.

##### Bốn phép gom xảy ra đồng thời — đừng nhầm chúng với nhau

| Phép | Gom cái gì | Công thức | Ví dụ ở trên |
|---|---|---|---|
| **Cộng lượng** | `qty` mọi dòng hàng cùng ô | `Σ qty` | 5+1+1+1 = 8 |
| **Bình quân giá** | tiền / lượng, **gia quyền** | `Σ(qty×price) / Σ qty` | cả 4 lượt cùng 355.000 ⇒ vẫn 355.000 |
| **HOẶC cờ hết hàng** | có **một** event `stock.out` là bật | `OR` | không có ⇒ `false` |
| **LẤY SÂU NHẤT** | nhiều đợt promo chồng nhau | `max(discount_pct)` | không có ⇒ 0 |

Bốn phép này **khác nhau về bản chất** — đây là chỗ dễ sai nhất nếu tự viết lại rollup:

- Lượng thì **cộng**: bán 2 lần, mỗi lần 3 cái, là 6 cái.
- Giá thì **không cộng**, và cũng **không lấy trung bình cộng đơn thuần**. Nếu ngày đó bán 5 cái giá 100.000 và
  1 cái giá 90.000 thì giá hiệu lực là `(5×100.000 + 1×90.000) / 6 = 590.000/6 = 98.333` —
  **không phải** `(100.000+90.000)/2 = 95.000`. Trung bình cộng đơn thuần cho hai mức giá cùng trọng số, sai
  về kinh tế.
- Cờ hết hàng thì **HOẶC**: hết hàng 10 phút cuối ngày cũng là `true`. Không có "hết hàng 30%".
- Promo chồng nhau lấy **sâu nhất**, không cộng dồn: hai đợt 20% và 30% cùng ngày ⇒ `30`, **không phải** 50.

##### Còn hai việc nữa rollup làm mà nhìn bảng không thấy

**(a) Điền ngày trống.** Rollup lặp **từng ngày** từ ngày đầu tiên của SKU tới hôm nay. Ngày không có event nào
vẫn đẻ ra một dòng `units_sold = 0`. Bỏ trống thì thư viện thống kê sẽ nội suy hoặc bỏ qua, và mức nền bị thổi
lên. **Vắng mặt là con số 0 thật, không phải "không biết".**

**(b) Nhớ giá của ngày không bán.** Ngày không ai mua thì không có giá bình quân để tính — rollup **mang giá gần
nhất sang** (carry-forward mốc `price.changed`). Vì thế cột `price` gần như không bao giờ trống, kể cả ngày bán 0.

Số đo thật một lượt rollup (2026-08-10, toàn bộ 8 tenant): **2 giây** cho 1.160 SKU / 49.640 ngày-SKU — khoảng
**25.000 ô lịch mỗi giây**.

##### Vì sao phải có tầng này? Sao model không đọc thẳng `raw_events`?

| Nếu bắt model đọc thẳng event thô | Hậu quả |
|---|---|
| Mỗi lần dự báo phải mở mảng `items` của hàng triệu dòng | chậm gấp hàng trăm lần, lặp lại y hệt mỗi ngày |
| Không có dòng cho ngày bán 0 | chuỗi thủng lỗ ⇒ mức nền bị thổi ⇒ dự báo cao giả |
| Không có chỗ nào ghi `adjusted_units` | không bù được ngày hết hàng ⇒ học nhầm "cầu đang giảm" |
| Mỗi model tự gom theo cách của mình | hai model ra hai con số khác nhau cho **cùng một ngày** — không ai gỡ nổi |

Dòng cuối là lý do mạnh nhất: `demand_daily` là **một bản sự thật duy nhất về quá khứ**, mọi model đọc cùng nó.
Tranh cãi về *dự báo* thì còn có chỗ; tranh cãi về *"hôm 31/05 bán mấy cái"* thì không.

##### Quay lại một dòng cụ thể

Dòng thật trong `demand_daily`:

```
project_id = demoshop · product_id = ld-srm-cerave · day = 2026-05-31
units_sold     = 8          ← 5 + 1 + 1 + 1
price          = 355000     ← bình quân gia quyền theo số lượng
stockout       = false
promo_pct      = 0
adjusted_units = 8          ← bằng units_sold vì ngày đó không hết hàng
```

Bốn dòng lẻ ở tầng 1 biến thành **một** dòng ở tầng 2. Ba việc quan trọng chỉ xảy ra ở tầng này:

1. **Đổi đơn vị**: từ *giây* sang *ngày*. Model làm việc theo ngày.
2. **Điền chỗ trống**: ngày không ai mua vẫn đẻ ra một dòng `units_sold = 0`. Vắng mặt là **con số thật**,
   không phải thiếu dữ liệu.
3. **Bù cái bị che**: ngày hết hàng, con số bán ra *không phải* nhu cầu thật.

Ví dụ thật cho việc thứ 3 — SKU khác, `tt-aothun-coolmate`, ngày 2026-08-05:

| 7 ngày trước | 29/07 | 30/07 | 31/07 | 01/08 | 02/08 | 03/08 | 04/08 |
|---|---|---|---|---|---|---|---|
| `adjusted_units` | 1 | 1 | 1 | 5 | 10 | 2 | 4 |

```
trung bình 7 ngày = (1+1+1+5+10+2+4) / 7 = 24 / 7 = 3,428571…

ngày 05/08: units_sold = 0 nhưng stockout = true
  ⇒ adjusted_units = max(0 ; 3,428571…) = 3,428571…      ← đúng bằng số trong DB
```

Bán 0 vì **hết hàng**, không phải vì hết người mua. Nếu để 0 chảy vào model thì nó học nhầm "cầu đang giảm"
→ dự báo thấp → nhập ít → hết hàng sớm hơn nữa. **Cột `adjusted_units` mới là cột model đọc**, không phải
`units_sold`.

Số đo thật một lượt rollup (2026-08-10, toàn bộ 8 tenant): **2 giây** cho 1.160 SKU / 49.640 ngày-SKU.

---

#### TẦNG 3 — nửa đêm, hệ tự ngồi tính dự báo

Mỗi 86.400 giây, job `forecast_run` chạy: học hệ số khuyến mãi của shop → gỡ ảnh hưởng sale ra khỏi lịch sử →
phân loại SKU bán đều hay bán lai rai → chọn model → hiệu chỉnh khoảng. Kết quả **ghi xuống bảng `forecasts`**,
mỗi ngày tương lai một dòng. Ba dòng đầu của mẻ `r_2026-08-10` cho chính SKU trên:

| `horizon_day` | p10 | p50 | p90 | `model_used` |
|---|---|---|---|---|
| 2026-08-11 | 0,531 | 1,503 | 3,444 | `autoets_theta_ensemble` |
| 2026-08-12 | 0,503 | 1,426 | 3,367 | `autoets_theta_ensemble` |
| 2026-08-13 | 0,476 | 1,348 | 3,289 | `autoets_theta_ensemble` |

kèm `data_window = 2026-04-08..2026-08-10` (124 ngày đã học) và
`calibration = {"width_factor": 0.647, "empirical_coverage": 0.952}`.

Đọc dòng đầu thành lời: *"Ngày 11/08, CeraVe nhiều khả năng bán 1,5 chai; kịch bản ế còn ~0,5 chai; muốn
90% chắc chắn không thiếu thì chuẩn bị ~3,4 chai."*

Hai điều đáng để ý ngay trong bảng này:

- **Số lẻ (1,503 chai) là bình thường, không phải lỗi.** Đây là *kỳ vọng* của một biến ngẫu nhiên, giống
  câu "trung bình mỗi hộ có 1,8 con". Người mua hàng làm tròn ở bước cuối, không phải hệ làm tròn ở bước này.
- **Càng xa càng thấp dần** (1,503 → 1,426 → 1,348): model đang kéo dự báo về mức nền dài hạn, vì càng xa thì
  thông tin của những ngày gần đây càng ít giá trị.

Và `empirical_coverage = 0,952` nghĩa là khoảng `[p10,p90]` **đo được** bao 95,2% giá trị thật trong khi nó chỉ
**hứa** 80% ⇒ khoảng quá rộng ⇒ hệ tự nén lại còn 64,7% bề rộng (`width_factor`). Xem mục G.

Số đo thật một lượt: **334 giây** cho 1.793 SKU, ghi 50.204 dòng dự báo.

> ⚠ **Ba câu hỏi của học viên về đúng đoạn này (2026-08-12):** *"hệ số khuyến mãi của shop là gì?"* ·
> *"gỡ ảnh hưởng sale ra khỏi lịch sử là gì?"* · *"job chỉ chạy 1 lần thì bảng `forecasts` trông ra sao,
> từng cột nghĩa là gì?"*
> Câu 1 và 2 được mổ đầy đủ ở **[§F](#f)** (có phép tính tay trên số thật), câu 3 ở **[§G](#g)**
> (12 cột, từng cột một). Ngay dưới đây trả lời gọn phần *"chỉ chạy 1 lần"*, vì nó là chỗ dễ hiểu nhầm nhất.

##### "Chạy 1 lần" nghĩa là gì — một lượt chạy đẻ ra bao nhiêu dòng?

Một lượt **không** đẻ ra một dòng. Nó đẻ ra **28 dòng cho mỗi SKU** — mỗi ngày tương lai một dòng
(`HORIZON_DEFAULT = 28`, `forecast_run.py:43`). Đo thật mẻ `r_2026-08-12` của `demoshop`:

```
132 SKU  ×  28 ngày  =  3.696 dòng      ← đúng bằng số đếm được trong DB
```

Và các mẻ **cộng dồn**, không đè nhau — mỗi mẻ có `run_id` riêng:

| `run_id` | dòng | SKU | ngày được dự báo | sinh lúc |
|---|---|---|---|---|
| `r_2026-08-05` | 3.164 | 113 | 06/08 → 02/09 | 05/08 08:27 |
| `r_2026-08-06` | 3.472 | 124 | 07/08 → 03/09 | 06/08 10:23 |
| `r_2026-08-07` | 3.598 | 129 | 08/08 → 04/09 | 07/08 02:56 |
| `r_2026-08-10` | 3.612 | 129 | 11/08 → 07/09 | 10/08 02:22 |
| `r_2026-08-11` | 3.696 | 132 | 12/08 → 08/09 | 11/08 04:28 |
| `r_2026-08-12` | 3.696 | 132 | 13/08 → 09/09 | 12/08 03:54 |

Ba điều đọc ra từ chính bảng này:

1. **Thiếu 08/08 và 09/08.** Không phải lỗi trình bày — hai ngày đó job **không chạy** (máy tắt cuối tuần).
   Bảng `forecasts` là sổ ghi *"đã thực sự chạy lúc nào"*, nên nó tố cáo được cả những ngày hệ nghỉ.
2. **Số SKU tăng dần** (113 → 132): shop bán thêm mặt hàng mới, mỗi ngày lại có thêm SKU đủ dữ liệu để dự báo.
3. **Cùng một ngày lịch được dự báo nhiều lần.** Ngày 13/08 có mặt trong cả 6 mẻ. Sáu con số cho cùng một ngày,
   sinh ở sáu thời điểm khác nhau — càng gần thì càng nhiều thông tin. Tầng 4 luôn đọc **mẻ mới nhất**;
   các mẻ cũ ở lại để đối chiếu *"hôm đó anh khuyên tôi nhập bao nhiêu?"*.

**Chạy hai lần trong CÙNG một ngày thì sao?** `run_id = "r_" + ngày hôm nay` (`forecast_run.py:1098`) nên hai
lượt cùng ngày dùng **chung một `run_id`**; `save_run` **xoá sạch dòng cùng `run_id` của SKU đó rồi ghi lại**
(`store/forecasts.py:96-104`). Tức lượt sau **đè** lượt trước, không nhân đôi. Đây là cùng một tính chất
*chạy-lại-được* của rollup, chỉ khác khoá.

---

#### TẦNG 4 — 8 giờ sáng, người mua hàng mở máy

Bây giờ mới có người hỏi. Anh ta gọi `POST /v1/forecast:query` cho SKU này, 14 ngày tới.

Hệ **không tính lại gì cả**. Nó đọc 14 dòng đã có sẵn trong `forecasts`, rồi làm ba việc nhẹ:

- nhân **hệ số lịch** (sắp Tết thì đội lên, mùng 1-2 Tết thì sụt);
- **cộng dồn** nếu người ta hỏi tổng 14 ngày (không phải cộng thẳng p90 — xem mục H);
- **hoà giải cấp bậc** nếu người ta hỏi cả ngành hàng.

Trả về trong vài chục mili-giây. Anh ta nhìn `p90 = 3,444` cho ngày mai, làm tròn lên, **nhập 4 chai**.

---

#### Tóm cả hành trình trong một bảng

| | Tầng 1 | Tầng 2 | Tầng 3 | Tầng 4 |
|---|---|---|---|---|
| Cùng một thông tin, ở dạng | 4 đơn hàng rời (4 khách) | 1 dòng/ngày | 1 dòng/ngày-tương-lai | câu trả lời |
| Con số | 5 · 1 · 1 · 1 | `units_sold = 8` | `p50 = 1,503` | "nhập 4 chai" |
| Ai làm | cửa hàng bắn vào | job `rollup` | job `forecast_run` | API khi có người hỏi |
| Khi nào | tức thì | mỗi 1 giờ | mỗi 1 ngày | lúc được gọi |
| Lưu ở đâu | `raw_events` | `demand_daily` | `forecasts` | không lưu |

### Vì sao phải chia bốn tầng — sao không tính thẳng lúc có người hỏi?

Đây là câu hỏi đúng, và có bốn câu trả lời, mỗi câu là một tính chất kỹ thuật thật:

| Nếu gộp lại tính lúc hỏi | Hậu quả |
|---|---|
| Cửa vào vừa nhận event vừa dự báo | Giờ cao điểm bán hàng = giờ hệ nghẽn. Việc **nặng** và việc **gấp** dẫm chân nhau |
| API đọc chạy model | Mỗi lần bấm F5 ra một con số khác (model có yếu tố ngẫu nhiên) ⇒ khách mất niềm tin ngay |
| Không lưu `forecasts` | Không trả lời được *"hôm qua anh khuyên tôi nhập bao nhiêu?"* — không có gì để đối chiếu khi sai |
| Không lưu `demand_daily` | Mỗi lần dự báo phải quét lại hàng triệu event thô ⇒ chậm gấp hàng trăm lần, và không ai bù được ngày hết hàng |

Cách chia này có một tên gọi chung trong ngành: **tách đường ghi khỏi đường đọc**. Đường ghi (tầng 1-3) làm
việc nặng theo lịch, âm thầm. Đường đọc (tầng 4) chỉ lấy kết quả đã có, nên luôn nhanh và luôn ra cùng một số.

### Bất biến phải nhớ

> **Tầng 3 là kết quả ĐÃ ĐÔNG LẠNH của một `run_id`. Tầng 4 KHÔNG chạy lại model.**

Hai hệ quả đi kèm — một tốt, một phải chấp nhận:

- **Tốt:** API đọc nhanh và **tất định** — hỏi 10 lần ra 10 lần giống nhau; và vì mỗi dòng mang theo
  `run_id`, `model_used`, `data_window`, `calibration`, ta luôn truy được *"con số này sinh lúc nào, bằng
  model gì, học trên khoảng dữ liệu nào"*. Sai thì mổ xẻ được, không phải cãi nhau bằng trí nhớ.
- **Phải chấp nhận:** số **cũ nhất bằng lần chạy job gần nhất**. Vừa nạp một đống dữ liệu mới xong mà gọi
  API ngay thì vẫn nhận số của mẻ cũ. Muốn số mới **phải chạy job**, gọi lại API bao nhiêu lần cũng vô ích.
  (Đây chính là lý do có `POST /v1/forecast:run` — nút bấm để ép chạy mẻ mới cho một shop.)

### Bản đồ rút gọn để nhớ

```
TẦNG 1 — THU        raw_events        ← POST /v1/events:ingest
   purchase.completed · order.returned · stock.out · price.changed · promo.scheduled
        │
        │  jobs/rollup.py :: run_rollup_once          (mỗi 1 giờ)
        ▼
TẦNG 2 — CHUẨN BỊ   demand_daily(project_id, product_id, day,
                                 units_sold, stockout, price, promo_pct, adjusted_units)
        │
        │  jobs/forecast_run.py                       (mỗi 1 ngày)
        │  học k → deflate promo → phân loại → chọn model → hiệu chỉnh
        ▼
TẦNG 3 — GHI        forecasts(project_id, product_id, run_id, horizon_day,
                              p10, p50, p90, model_used, data_window, calibration)
        │
        ▼   (không chạy model — chỉ đọc + biến đổi nhẹ)
TẦNG 4 — ĐỌC        main.py: forecast:query · aggregate · promo-preview · insights · scenarios:*
                    (+ apply_calendar, + tổng hợp horizon, + reconcile)
```

---

## 0.2b · GIẢI THÍCH TỪNG THÀNH PHẦN CỦA BỐN TẦNG

> Bổ sung theo câu hỏi trong lớp. Mọi schema và mọi dòng dữ liệu dưới đây **lấy trực tiếp từ DB đang chạy**
> (`localhost:16024/miniai_forecast`) lúc 2026-08-10 01:45–01:51, không viết từ trí nhớ.

Mục lục nhanh: [A raw_events](#a) · [B cửa ingest](#b) · [C 5 loại event](#c) · [D rollup](#d) · [E demand_daily](#e) · [F forecast_run](#f) · [G forecasts](#g) · [H tầng đọc](#h)

<a id="a"></a>

## A · `raw_events` — SỔ NHẬT KÝ GỐC

Hình dung: một **cuốn sổ chỉ được viết thêm, không được tẩy xoá**. Mỗi dòng là một sự việc đã xảy ra ở cửa hàng.

| Cột | Kiểu thật | Nghĩa dễ hiểu |
|---|---|---|
| `project_id` | text | **Khách hàng nào** của miniAI (mỗi shop = 1 tenant). Mọi thứ đều bị khoá theo cột này |
| `event_id` | text | Số hiệu sự việc **do người gửi đặt**. Cùng `(project_id, event_id)` = **khoá chính** |
| `schema_version` | text | Bản hợp đồng dữ liệu — để sau này đổi định dạng mà không vỡ hàng cũ |
| `event_type` | text | Loại sự việc (5 loại, mục C) |
| `event_time` | timestamptz | **Lúc việc xảy ra** (theo người gửi) |
| `received_at` | timestamptz | **Lúc hệ nhận được** — mặc định `now()` |
| `user_pseudo_id` | text | Người mua ẩn danh (BT01 cần; BT03 hầu như không dùng) |
| `session_id`, `attribution_token` | text | Phiên duyệt web / vết nguồn traffic (dành cho BT01) |
| `payload` | jsonb | **Ruột** của sự việc: `qty`, `unit_price`, `product_id`… tuỳ loại |

Hai chi tiết đáng để ý:

1. **`event_time` ≠ `received_at`.** Việc xảy ra 20:00 nhưng mạng shop chập chờn, 23:00 mới bắn lên. Có hai cột nên hệ phân biệt được "hàng đến trễ" với "hàng mới". Nếu chỉ có một cột thì mọi dữ liệu trễ sẽ bị ghi nhầm ngày.
2. **Khoá chính `(project_id, event_id)`** — gửi lại đúng sự việc đó 10 lần cũng chỉ có 1 dòng. Đây là **chống trùng ở tầng cửa**: shop cứ retry thoải mái khi mạng lỗi mà không sợ nhân đôi doanh số.

Chỉ mục thật trên bảng: `idx_raw_events_project_type_time (project_id, event_type, event_time)` — phục vụ đúng câu hỏi mà rollup hỏi: *"shop này, loại event này, trong khoảng thời gian này"*.

---

<a id="b"></a>

## B · `POST /v1/events:ingest` — CÁI CỬA

Đọc từng phần của địa chỉ:

| Phần | Nghĩa |
|---|---|
| `POST` | động từ HTTP nghĩa "tôi **gửi** dữ liệu vào" (khác `GET` = "cho tôi xem") |
| `/v1/` | **phiên bản 1** của hợp đồng API. Sau này đổi kiểu dữ liệu thì mở `/v2/`, khách cũ không vỡ |
| `events` | tài nguyên: sự việc |
| `:ingest` | **hành động** trên tài nguyên đó (kiểu đặt tên của Google API) — "nạp vào" |

Code tại `services/forecast/app/main.py:426`. Ba việc cửa này làm:

1. **Nhận diện khách qua API key** → suy ra `project_id`. Khách **không tự khai** mình là ai; khoá quyết định. Đây là lý do một shop không bao giờ thấy được dữ liệu shop khác.
2. **Kiểm định dạng** rồi ghi thẳng vào `raw_events`. Cửa **không tính toán gì** — nhận và cất.
3. **Trả lời ngay.** Mọi việc nặng để job nền làm sau.

> **Nguyên tắc: cửa vào phải nhẹ.** Nếu ingest mà đi tính dự báo luôn thì shop bắn 1000 event/phút là hệ sập.

Có anh em `POST /v1/events:backfill` (`main.py:485`) — cùng thân dữ liệu, cùng bộ gom, nhưng dành cho **nạp lịch sử cũ** và có luật riêng về mốc thời gian (sàn −90 ngày).

---

<a id="c"></a>

## C · NĂM LOẠI EVENT — AI BẮN, THIẾU THÌ HỎNG GÌ

| Loại | Ai bắn | Ruột `payload` | Thiếu nó thì hỏng gì |
|---|---|---|---|
| `purchase.completed` | hệ bán hàng, khi đơn **thanh toán xong** | `product_id`, `qty`, `unit_price` | Không có gì để dự báo — đây là xương sống |
| `order.returned` | khi khách trả hàng | `product_id`, `qty` | Cầu bị **thổi phồng**: hàng trả rồi vẫn tính là bán |
| `stock.out` | kho/POS khi hết hàng | `product_id` | **Nguy nhất**: ngày hết hàng bị đọc là "cầu giảm" → vòng xoáy đặt thiếu |
| `price.changed` | khi đổi giá niêm yết | `product_id`, `new_price` | Cột `price` rỗng → không học được co giãn giá, BT02 mất đầu vào |
| `promo.scheduled` | khi lên lịch khuyến mãi | `product_id(s)`, `discount_pct`, `start`, `end` | Không tách được **nền** khỏi **uplift** → dự báo ngày thường bị thổi theo ngày sale |

Khác biệt về **hình dạng thời gian**, nhớ kỹ:

- `purchase.completed`, `order.returned`, `stock.out`, `price.changed` = **một thời điểm**.
- `promo.scheduled` = **một khoảng** `[start, end]` ⇒ rollup phải "tô" nó ra thành nhiều ngày, và khi nhiều promo chồng nhau thì lấy **giảm giá sâu nhất** (`max`).

---

<a id="d"></a>

## D · `jobs/rollup.py :: run_rollup_once` — NGƯỜI THƯ KÝ

**"Job" khác "API"**: API do khách gọi; job **tự chạy theo lịch**, không ai bấm.

Tên hàm nói hết thiết kế:

- **`run`** — chạy
- **`rollup`** — "cuộn lên": gom nhiều dòng lẻ thành một dòng tổng theo ngày
- **`once`** — **một lượt duy nhất rồi thôi**. Vòng lặp lịch nằm ở chỗ khác (`start_rollup_loop`)

Tách `once` khỏi `loop` là chủ ý, và rất quan trọng:

| Người gọi | Gọi cái gì | Vì sao |
|---|---|---|
| Lịch nền | `start_rollup_loop` → gọi `once` mỗi 3600 s | vận hành bình thường |
| Test | gọi thẳng `once` | test không phải chờ 1 tiếng |
| Người vận hành / seed tool | gọi thẳng `once` | nạp xong dữ liệu, muốn thấy kết quả ngay |

Việc `once` làm, đúng thứ tự:

1. Đọc `raw_events` trong cửa sổ `window_days` (mặc định 120).
2. Cộng theo `(SKU, ngày)` theo bảng ở mục C.
3. **Điền đủ mọi ngày trống bằng 0** — vắng mặt là con số thật, không phải thiếu dữ liệu.
4. Tính giá bình quân **gia quyền theo số lượng**; ngày không bán thì carry-forward giá gần nhất.
5. **Pass 2: bù ngày hết hàng** → cột `adjusted_units`.
6. **UPSERT** vào `demand_daily` theo khoá `(project_id, product_id, day)`.

Số đo thật của một lượt (đo 2026-08-10 01:41, toàn bộ 8 tenant): **2 giây** cho `1160` SKU / `49.640` ngày-SKU.

---

<a id="e"></a>

## E · `demand_daily` — TỪNG CỘT, KIỂU THẬT, DÒNG THẬT

Một dòng thật lấy từ DB lúc 01:45 hôm nay:

```
product_id = bh-cafe-g7 · day = 2026-08-10 · units_sold = 0 · stockout = f
price = 176000 · promo_pct = 10 · adjusted_units = 0
```

| Cột | Kiểu thật | Đơn vị | Giải thích |
|---|---|---|---|
| `project_id` | text | — | shop nào |
| `product_id` | text | — | SKU nào (`bh-cafe-g7` = cà phê G7) |
| `day` | **date** | ngày | chỉ ngày, không có giờ — đơn vị của cả bài toán |
| `units_sold` | **numeric** | cái | số bán ra. Dùng `numeric` (thập phân **chính xác**), không phải `float` |
| `stockout` | boolean | — | ngày đó có hết hàng không |
| `price` | **bigint** | **đồng** | `176000` = 176.000đ. **Số nguyên** — tiền không bao giờ để kiểu thực |
| `promo_pct` | numeric | **phần trăm** | `10` = giảm 10% (không phải 0,1) |
| `adjusted_units` | numeric | cái | **cầu đã bù** ngày hết hàng — cột mà **mọi model** đọc |

Ba điều thiết kế nằm ngay trong bảng này:

1. **Khoá chính `(project_id, product_id, day)`** — mỗi SKU mỗi ngày đúng **một** dòng. Đây là cái làm rollup chạy lại được: `ON CONFLICT … DO UPDATE` = ghi đè chính nó, không đẻ thêm.
2. **`price` là `bigint`, không phải `float`.** Tiền để kiểu thực sẽ sinh `176000.00000000001`; cộng vài triệu dòng là lệch sổ. Đây là bất biến tiền tệ, không phải sở thích.
3. **`RLS POLICY tenant_isolation`** ngay trên bảng: `project_id = current_setting('app.project_id')`. Cách ly giữa các shop do **Postgres** ép, không phụ thuộc lập trình viên nhớ viết `WHERE`. Quên `WHERE` thì thấy **0 dòng**, chứ không thấy dữ liệu shop khác.
   *(Lưu ý vận hành: role `miniai` là **chủ bảng** nên Postgres cho phép nó bỏ qua RLS — muốn kiểm tra RLS thật phải dùng role `miniai_app` của đường request.)*

Đọc dòng trên thành lời:

> *"Ngày 10/08, cà phê G7 ở shop demoshop bán 0 cái, không hết hàng, giá niêm yết 176.000đ, đang giảm 10%; cầu đã bù = 0 (đúng bằng số bán, vì không hết hàng)."*

Số 0 ở đây **hợp lý**: 10/08 là ngày đang chạy, chưa có event bán nào.

---

<a id="f"></a>

## F · `jobs/forecast_run.py` — NĂM BƯỚC

| Bước | Việc | Vì sao phải có |
|---|---|---|
| 1. **Học `k`** | Từ lịch sử của **chính shop này**, ước lượng "giảm giá 1% thì bán tăng bao nhiêu" | Mỗi ngành hàng một khác; hằng số cứng 1.5 là nói bừa |
| 2. **Deflate promo** | Chia ngược cầu quá khứ cho `(1 + k·frac)` để lấy **mức nền không sale** | Nếu không: quá khứ có 20 ngày sale làm mức nền bị thổi, rồi ngày sale tương lai lại nhân uplift ⇒ **đếm hai lần** |
| 3. **Phân loại** | Đo `ADI` (bao lâu bán một lần) và `CV²` (mỗi lần bán lệch nhau bao nhiêu) | Chuỗi bán đều và chuỗi bán lai rai cần **hai họ thuật toán khác nhau** |
| 4. **Chọn model** | Ưu tiên `kv_state model_choice` do backtest ghi; không có thì router chọn theo phân loại | Bằng chứng đo được đè lên luật tay |
| 5. **Hiệu chỉnh** | Nong/nén khoảng bằng `width_factor`, rồi **nhân lại uplift** cho ngày sale tương lai | Model thô cho khoảng không trung thực; phải chỉnh cho coverage về đúng 80% |

**Bước 2 và bước 5 là một cặp đối xứng:** bước 2 gỡ ảnh hưởng khuyến mãi ra khỏi **quá khứ**, bước 5 lắp nó lại vào **tương lai**. Gỡ mà quên lắp = dự báo hụt mọi ngày sale.

Số đo thật một lượt batch toàn bộ tenant (2026-08-10 01:41→01:47): **334 giây** cho `1793` SKU, ghi `50.204` dòng dự báo.

---

### F.1 · "HỆ SỐ KHUYẾN MÃI CỦA SHOP" (`k`) — MỔ KỸ

> ⚠ **Câu hỏi của học viên: "hệ số khuyến mãi của shop là gì?"**
> Toàn bộ phần này đọc từ `core/promo_uplift.py` (132 dòng) + `jobs/forecast_run.py:333-388`, và mọi con số
> đều truy thẳng từ DB đang chạy lúc 2026-08-12.

#### Nó trả lời đúng MỘT câu hỏi

> *"Ở shop này, giảm giá thì bán tăng bao nhiêu?"*

Công thức dùng nó nằm ngay đầu file (`promo_uplift.py:5`):

```
số bán ngày sale  =  số bán ngày thường  ×  (1 + k × mức_giảm)
```

`mức_giảm` là **tỷ lệ** (0..1), không phải phần trăm: giảm 24,3% ⇒ `0,243`.

Cách nhớ nhanh: cho `mức_giảm = 0,01` (giảm 1%) thì thừa số là `1 + 0,01k` ⇒ **giảm 1% thì bán tăng khoảng `k` phần trăm.**

| `k` | Nghĩa nghiệp vụ |
|---|---|
| `k = 0` | giảm giá **không** làm bán thêm cái nào — sale vô ích |
| `k = 1` | giảm 10% thì bán tăng 10% (doanh thu gần như đứng yên) |
| `k = 3` | giảm 10% thì bán tăng 30% — hàng rất nhạy giá |

**Vì sao phải HỌC chứ không đặt hằng số?** Trong code vẫn có một hằng số `PROMO_UPLIFT_K = 1.5`
(`forecast_run.py:46`) — nhưng nó chỉ là **phao cứu sinh** khi không học nổi. Lý do: mỗi ngành hàng nhạy giá
một kiểu. Đo thật 10 shop trong DB, `k` chênh nhau **hơn 4 lần**:

| shop | `k` học được | số SKU góp | số ngày sale |
|---|---|---|---|
| `demoshop` | **0,9474** | 48 | 719 |
| `simworld4` | 1,9088 | 79 | 797 |
| `simworld1` | 2,1628 | 59 | 607 |
| `elastest` | 3,0775 | 15 | 216 |
| `simworld3` | 3,4020 | 37 | 407 |
| `simworld2` | 3,4937 | 20 | 236 |
| `bulktest` · `demo` · `seedtest` · `stafffull` | **4,0000** | 50 · 60 · 18 · 15 | 774 · 877 · 210 · 217 |

> ⛔ **Bốn shop cuối KHÔNG "học ra 4,0" — chúng ĐỤNG TRẦN.** `K_MAX = 4.0` (`promo_uplift.py:30`) và bước cuối
> kẹp `k = max(0, min(4, median))`. Con số 4,0000 tròn trịa ở bốn shop khác nhau là dấu hiệu bị cắt, không phải
> trùng hợp. Tệ hơn: payload lưu trong DB **không ghi lại việc đã cắt**, nên người đọc không phân biệt được
> *học-ra-4* với *bị-cắt-về-4*. Đây là một khuyết tật thật, đã đặt tên trong DB tri thức:
> **`W-UPLIFT-K-CLAMP-SATURATED`**.

#### Thuật toán, đúng 6 bước theo code

| Bước | Việc | Dòng code |
|---|---|---|
| 1 | **Vứt mọi ngày hết hàng** (`stockout = true`) | `promo_uplift.py:75` |
| 2 | Gom theo SKU; chia làm **ngày sale** (`promo_pct > 0`) và **ngày thường** (`promo_pct == 0`) | `:90-91` |
| 3 | Loại SKU có **< 3 ngày sale** hoặc **< 7 ngày thường** | `:93-96` |
| 4 | Mỗi SKU tính `implied_k = (bán_TB_ngày_sale / bán_TB_ngày_thường − 1) / mức_giảm_TB` | `:112` |
| 5 | **Dưới 3 SKU sống sót ⇒ trả về `None`** (không đủ căn cứ, dùng hằng số 1,5) | `:121-122` |
| 6 | `k = TRUNG VỊ` của các `implied_k`, kẹp vào `[0 ; 4]` | `:125-126` |

Ba ngưỡng ở bước 3 và 5 (`3` ngày sale · `7` ngày thường · `3` SKU) đều là **chống nhiễu**: một SKU có đúng
1 ngày sale mà hôm đó tình cờ bán gấp đôi sẽ đẻ ra `implied_k` khổng lồ và kéo lệch cả shop.

**Vì sao bước 1 vứt ngày hết hàng?** Vì ngày hết hàng, con số bán ra bị tồn kho chặn — đúng chuyện đã học ở
mục E. Nếu một đợt sale làm cháy hàng giữa chừng, ngày đó **bán ít đi** vì hết hàng chứ không phải vì sale
không hiệu quả. Để nó vào sẽ dạy máy rằng *"giảm giá làm bán ít đi"*.

#### Tính tay trên số thật — `demoshop`, 6 SKU đầu bảng chữ cái

Mọi con số dưới đây đọc thẳng từ `demand_daily` (bỏ ngày hết hàng, đúng bước 1):

| SKU | ngày sale | ngày thường | bán TB ngày thường | bán TB ngày sale | mức giảm TB | `implied_k` |
|---|---|---|---|---|---|---|
| `bh-bia-tiger` | 10 | 124 | 1,5968 | 2,4000 | 0,2280 | **2,2062** |
| `bh-cafe-g7` | 10 | 124 | 2,6613 | 2,9000 | 0,1000 | **0,8969** |
| `bh-dauan-neptune` | 13 | 121 | 1,6860 | 1,4615 | 0,3325 | **−0,4004** |
| `bh-duong-bienhoa` | 12 | 122 | 1,6639 | 1,7500 | 0,2260 | **0,2289** |
| `bh-mi-haohao` | 12 | 122 | 4,4836 | 6,5000 | 0,2490 | **1,8062** |
| `bh-nuocmam-namngu` | 10 | 124 | 3,8629 | 4,1000 | 0,1750 | **0,3507** |

Phép tính đầy đủ cho mì Hảo Hảo:

```
tỷ lệ bán tăng = 6,5000 / 4,4836        = 1,44975      (tăng 44,975 %)
trừ 1                                    = 0,44975
chia cho mức giảm TB                     = 0,44975 / 0,2490 = 1,80622
```

Đọc thành lời: *"Hảo Hảo giảm trung bình 24,9% thì bán tăng 45% ⇒ quy về 1% giảm giá thì bán tăng ~1,8%."*

> 🔴 **Dòng thứ ba là dòng đáng học nhất: `implied_k` ÂM.** Dầu ăn Neptune ngày sale bán **1,46** cái, ngày
> thường bán **1,69** — sale mà bán **ít hơn**. Nguyên nhân có thể là bất kỳ thứ gì code không nhìn thấy: đợt
> sale rơi vào giai đoạn ế, đối thủ giảm sâu hơn, hoặc đơn giản là ngẫu nhiên trên một SKU bán lai rai.
> Toàn shop có **5/48 SKU** mang `implied_k` âm, thấp nhất **−0,6633**; đầu kia có SKU tới **+5,3321**.

#### Vì sao bước 6 lấy TRUNG VỊ, không lấy trung bình cộng?

Đây là chỗ một dòng code chứa cả một quyết định kỹ thuật (`promo_uplift.py:125`):

```python
k = statistics.median(implied_ks)
```

| Phép | Với 48 SKU của `demoshop` | Vấn đề |
|---|---|---|
| Trung bình cộng | bị SKU **+5,33** và SKU **−0,66** kéo hai đầu | một SKU dị thường đủ sức lệch cả shop |
| **Trung vị** (đang dùng) | **0,9474** | nửa số SKU cao hơn, nửa thấp hơn — vài ca dị thường **không** đổi được kết quả |

Trung vị còn khiến việc kẹp `[0 ; 4]` ở bước 6 hầu như không phải hoạt động: nó chỉ chạm trần khi **quá nửa**
số SKU của shop thật sự có uplift ≥ 4 (đúng ca 4 shop dữ liệu dựng sẵn ở bảng trên).

#### `k` sống ở đâu, và ai đọc

Không sống trong file cấu hình — sống trong DB, một dòng cho mỗi shop (`forecast_run.py:362-370`):

```
bảng kv_state · khoá "promo_uplift_k:demoshop"
{"k": 0.947408367288503, "n_skus": 48, "n_promo_days": 719,
 "computed_at": "2026-08-12T03:54:49.493831+00:00"}
```

Ba trường phụ tồn tại để **cãi lại được**: con số này dựa trên bao nhiêu SKU, bao nhiêu ngày sale, tính lúc nào.
Một `k` học từ 3 SKU không đáng tin bằng `k` học từ 48 SKU, và payload nói thẳng ra điều đó.

- **Ghi:** mỗi lượt `forecast_run`, học lại một lần cho mỗi shop rồi cache trong lượt (`forecast_run.py:1123`).
- **Đọc:** đường request (`forecast:promo-preview`, forecast theo yêu cầu) đọc lại từ `kv_state`
  (`_load_uplift_k`, `:374`) — học nặng thì để job nền làm, đường đọc chỉ tra bảng. Đúng nguyên tắc tầng 3/tầng 4.
- **Hỏng thì sao:** `estimate_uplift_k` trả `None` (không đủ SKU) ⇒ **không ghi gì** vào DB và dùng `1.5`;
  đọc lỗi cũng trả `1.5` (`:380-387`). Degrade có tên, không nổ.

> ⚠ **`k` KHÔNG phải hằng số — nó TRÔI theo dữ liệu.** Đo thật cùng một shop trong cùng một ngày:
> lúc 03:54 `k = 0,9474` (719 ngày sale) → lúc 06:26 chạy lại chính hàm đó cho **`k = 0,8584`** (710 ngày sale).
> Cùng code, cùng shop, cách nhau 2,5 giờ, lệch **9,4%**. Câu chuyện đằng sau con số này là một **lỗi thật của
> sản phẩm** mà chính buổi học hôm nay lôi ra — kể đầy đủ ở phần cuối bài.

---

### F.2 · "GỠ ẢNH HƯỞNG SALE RA KHỎI LỊCH SỬ" (deflate) — MỔ KỸ

> ⚠ **Câu hỏi của học viên: "gỡ ảnh hưởng sale ra khỏi lịch sử là gì?"**

#### Bệnh cần chữa: ĐẾM HAI LẦN

Hình dung một SKU bán nền 3 cái/ngày. Trong 124 ngày lịch sử có 24 ngày chạy sale, những ngày đó bán 6 cái.

Nếu **không** gỡ gì cả:

```
① Model học trên chuỗi có cả ngày sale  →  nó thấy mức nền "trung bình" bị đội lên  ≈ 3,6
② Ngày sale tương lai lại được NHÂN thêm (1 + k·mức_giảm)                          × 1,23
                                                                    ⇒ 4,4 thay vì 3,7
```

Ảnh hưởng của sale bị tính **hai lần**: một lần chìm trong mức nền, một lần nhân tường minh. Và ngày **thường**
trong tương lai cũng bị thổi theo, vì mức nền đã cao sẵn. Đây là cái nối `W-BT23` trong code gọi là
**"promo seam"** — đường nối giữa lịch sử và tương lai.

#### Thuốc: chia ngược đúng cái đã nhân

`_deflate_promo_units` (`forecast_run.py:436-453`) — toàn bộ phép chữa nằm trong một dòng:

```python
u = u / (1.0 + k * min(max(frac, 0.0), 1.0))       # frac = promo_pct / 100
```

Nó **đảo ngược** đúng công thức của mục F.1: nhân lên bằng `(1 + k·frac)` thì chia lại bằng `(1 + k·frac)`.
Ngày không sale (`frac = 0`) thì thừa số là 1 — **không đụng gì tới nó**.

`min(max(frac, 0), 1)` kẹp mức giảm vào `[0 ; 1]`: một payload khai `discount_pct = 150` sẽ không tạo ra
thừa số quái đản; giảm quá 100% vô nghĩa về kinh tế.

#### Tính tay trên số thật — `ld-srm-cerave`, đợt sale tháng 6

SKU quen thuộc xuyên suốt bài này. Đợt sale 24,3% từ 02/06, và `k = 0,947408` của `demoshop`:

```
thừa số = 1 + 0,947408 × 0,243 = 1 + 0,230220 = 1,230220
```

| ngày | `promo_pct` | `adjusted_units` (cái đã bán) | **model thật sự học** |
|---|---|---|---|
| 02/06 | 24,3 | 3 | 3 / 1,23022 = **2,4386** |
| 03/06 | 24,3 | 4 | 4 / 1,23022 = **3,2515** |
| **04/06** | **24,3** | **8** | **8 / 1,23022 = 6,5029** |
| 05/06 | 24,3 | 5 | 5 / 1,23022 = **4,0643** |
| 06/06 | 24,3 | 5 | 5 / 1,23022 = **4,0643** |
| 07/06 | 24,3 | 1 | 1 / 1,23022 = **0,8129** |

Đọc dòng 04/06 thành lời: *"Hôm đó bán được 8 chai, nhưng đang giảm 24,3%. Nếu KHÔNG có sale thì ước chừng
chỉ bán 6,5 chai — và 6,5 mới là con số model được phép coi là mức nền."*

#### Cặp đối xứng: gỡ ở quá khứ, lắp lại ở tương lai

| | Hàm | Phép | Chạy trên |
|---|---|---|---|
| **Gỡ** (bước 2) | `_deflate_promo_units` (`:436`) | **chia** cho `(1 + k·frac)` | mọi ngày **lịch sử** có sale |
| **Lắp** (bước 5) | `_apply_promo_uplift` (`:307`) | **nhân** cả p10/p50/p90 với `(1 + k·frac)` | ngày **tương lai** đã lên lịch sale |

Hai hàm này phải luôn đi cùng nhau. Gỡ mà quên lắp ⇒ mọi ngày sale tương lai bị dự báo hụt. Lắp mà quên gỡ ⇒
đếm hai lần như trên. Trong code, `_apply_promo_uplift` còn **sắp lại thứ tự** ba phân vị sau khi nhân
(`sort_quantiles`, `:328`) để không bao giờ có chuyện `p10 > p50`.

> ⭐ **Ngoại lệ quan trọng: `lgbm_global` KHÔNG được nhân.** Điều kiện thật trong code là
> `if promo and model_used != "lgbm_global"` (`forecast_run.py:1308`). Lý do: model học máy này nhận
> `promo_pct` **làm đặc trưng đầu vào** — nó đã tự học ảnh hưởng của sale rồi. Nhân thêm lần nữa là đếm hai
> lần, y hệt bệnh ở trên nhưng ở một chỗ khác. Ở mẻ `r_2026-08-12` của `demoshop`, `lgbm_global` phục vụ
> **64/132 SKU** — tức gần một nửa số SKU đi đường "không nhân".
>
> Đây là cùng một loại cạm bẫy với `apply_calendar` ở mục H (lịch Tết cũng không nhân cho `lgbm_global`).
> Quy tắc chung đáng thuộc: **mỗi ảnh hưởng chỉ được vào con số ĐÚNG MỘT LẦN, bằng đúng MỘT đường.**

---

<a id="g"></a>

## G · `forecasts` — TỪNG CỘT, ĐỌC MỘT DÒNG THẬT

Dòng thật, sinh lúc 01:44 hôm nay:

```
product_id  = bh-suatuoi-th          run_id      = r_2026-08-10
horizon_day = 2026-08-11             p10 = 0,464   p50 = 2   p90 = 4,304
model_used  = seasonal_naive         data_window = 2026-04-08..2026-08-10
calibration = {"width_factor": 0.768, "empirical_coverage": 0.905}
```

| Cột | Kiểu thật | Giải thích |
|---|---|---|
| `id` | bigint | số thứ tự dòng, tự tăng |
| `project_id` · `product_id` | text | shop nào · SKU nào |
| `run_id` | text | **mẻ chạy**: `r_2026-08-10` = mẻ ngày 10/08. Cả mẻ chia sẻ một `run_id` ⇒ luôn tách được "số hôm nay" với "số hôm qua" |
| `horizon_day` | **date** | ⚠ **NGÀY ĐƯỢC DỰ BÁO** — một ngày lịch thật, **không phải** "ngày thứ mấy tính từ lần chạy". Mẻ 10/08 sinh các dòng cho 11/08, 12/08, … |
| `p10` · `p50` · `p90` | numeric | ba phân vị của **cầu ngày đó**, đơn vị **cái** |
| `model_used` | text | model nào thực sự đẻ ra ba số này |
| `data_window` | text | **cửa sổ dữ liệu đã học**: `2026-04-08..2026-08-10` = 124 ngày |
| `calibration` | jsonb | hai số của bước hiệu chỉnh (giải thích dưới) |
| `created_at` | timestamptz | lúc dòng được ghi |

### Đọc dòng này thành lời

> *"Sữa tươi TH: ngày mai (11/08) nhiều khả năng bán **2 hộp**. Kịch bản ế thì ~0,5 hộp; muốn 90% chắc chắn không thiếu thì chuẩn bị **~4,3 → 5 hộp**. Số này do `seasonal_naive` đẻ ra, học trên 124 ngày."*

`p50 = 2` mà `p90 = 4,3` ⇒ muốn phục vụ 90% phải chuẩn bị **hơn gấp đôi** trung vị. Bài học "điểm vs phân phối" hiện ra bằng số thật.

### Cột `calibration` — hai con số nói lên toàn bộ triết lý

| Khoá | Giá trị dòng này | Nghĩa |
|---|---|---|
| `empirical_coverage` | **0,905** | Đo trên backtest: khoảng `[p10,p90]` thực sự bao **90,5%** giá trị thật, trong khi nó chỉ **hứa 80%** ⇒ **khoảng quá rộng** |
| `width_factor` | **0,768** | Vì rộng quá nên hệ **nén** khoảng lại còn 76,8% bề rộng cũ |

So sánh với dòng `lgbm_global` sinh cùng lúc: `{"width_factor": 0.5, "empirical_coverage": 1.0}` — coverage **1,0** nghĩa là khoảng bao **100%** giá trị thật, rộng đến mức vô dụng, nên bị nén xuống **sàn 0,5** (mức nén tối đa cho phép).

> Đây là bằng chứng sống: **coverage cao không phải thành tích**. Hệ nhìn thấy 1,0 và phản ứng bằng cách **bóp khoảng lại**, chứ không đem khoe.

`cold_start` thì `calibration` để **NULL** — SKU mới toanh, chưa có lịch sử để đo coverage nên không có gì để hiệu chỉnh. Trung thực: không đo được thì để trống, không bịa số.

---

### G.1 · MỔ TỪNG CỘT — schema thật, ví dụ thật, cạm bẫy

> ⚠ **Câu hỏi của học viên: "cấu trúc bảng `forecasts` là gì, giải thích chi tiết từng field?"**
> Đây là schema nguyên trạng, đọc bằng `\d forecasts` trên DB đang chạy lúc 2026-08-12 — **12 cột**, nhiều hơn
> bảng tóm tắt phía trên vì bảng đó bỏ qua `id` và gộp vài cột.

```
                                       Table "public.forecasts"
   Column    |           Type           | Nullable |                Default
-------------+--------------------------+----------+---------------------------------------
 id          | bigint                   | not null | nextval('forecasts_id_seq'::regclass)
 project_id  | text                     | not null |
 product_id  | text                     | not null |
 run_id      | text                     | not null |
 horizon_day | date                     | not null |
 p10         | numeric                  |          |
 p50         | numeric                  |          |
 p90         | numeric                  |          |
 model_used  | text                     |          |
 data_window | text                     |          |
 calibration | jsonb                    |          |
 created_at  | timestamp with time zone | not null | now()
Indexes:
    "forecasts_pkey" PRIMARY KEY, btree (id)
    "idx_forecasts_project_product_created" btree (project_id, product_id, created_at DESC)
    "idx_forecasts_project_product_run"     btree (project_id, product_id, run_id)
Policies:
    POLICY "tenant_isolation" USING ((project_id = current_setting('app.project_id', true)))
```

Một dòng thật để bám theo suốt phần này — chính SKU CeraVe của bài, mẻ hôm nay:

```
id=…  project_id=demoshop  product_id=ld-srm-cerave  run_id=r_2026-08-12
horizon_day=2026-08-13   p10=1,092   p50=3,033   p90=4,974
model_used=autoets_theta_ensemble    data_window=2026-04-01..2026-08-12
calibration={"width_factor": 0.6470024056079998, "empirical_coverage": 0.9523809523809523}
created_at=2026-08-12 03:54:52+00
```

---

**1. `id` · `bigint` · tự tăng · BẮT BUỘC**

Số thứ tự dòng, Postgres tự cấp. Không mang nghĩa nghiệp vụ nào.

> ⚠ **Chi tiết dễ bỏ qua nhưng quan trọng: khoá chính là `id`, KHÔNG phải bộ nghiệp vụ.**
> Bảng `demand_daily` có `PRIMARY KEY (project_id, product_id, day)` nên DB **tự cấm** hai dòng cho cùng một
> ngày. Bảng `forecasts` **không có** ràng buộc tương ứng trên `(project_id, product_id, run_id, horizon_day)`.
> Nghĩa là **DB cho phép** hai dòng trùng — thứ giữ cho không trùng là **phần mềm**: `save_run` xoá sạch dòng
> cùng `run_id` trước khi ghi (`store/forecasts.py:96-104`), cộng với **khoá tiến trình** `pg_advisory_lock`
> theo từng shop (`forecast_run.py:1119`) chặn hai mẻ chạy đè nhau.
> Đánh đổi: linh hoạt hơn (ghi lại mẻ cũ, chèn thủ công) nhưng **kỷ luật nằm ở code, không nằm ở DB** — mất
> một lớp lưới an toàn so với `demand_daily`.

---

**2. `project_id` · `text` · BẮT BUỘC** — shop nào. Cùng vai trò cột cách ly như ở `demand_daily`, và cũng là
cột mà `POLICY tenant_isolation` bám vào. Hai lớp khoá y hệt mục E.

**3. `product_id` · `text` · BẮT BUỘC** — SKU nào. Cặp `(project_id, product_id)` là danh tính mặt hàng.

---

**4. `run_id` · `text` · BẮT BUỘC**

**Số hiệu MẺ CHẠY.** Sinh bằng đúng một dòng (`forecast_run.py:1098`):

```python
run_id = "r_" + date.today().isoformat()        # -> "r_2026-08-12"
```

Ba tính chất rơi ra từ chính dòng đó:

- **Cả mẻ dùng chung một `run_id`.** 133 SKU × 28 ngày = 3.724 dòng cùng mang `r_2026-08-12` ⇒ luôn tách được
  *"số hôm nay"* với *"số hôm qua"*.
- **Một ngày chỉ có một `run_id`.** Chạy lần thứ hai trong ngày ⇒ cùng tên ⇒ `save_run` xoá rồi ghi lại =
  **ghi đè**, không nhân đôi.
- **Không có giờ trong tên.** Muốn biết mẻ chạy lúc mấy giờ thì đọc `created_at`, không đọc `run_id`.

---

**5. `horizon_day` · `date` · BẮT BUỘC**

⚠ **NGÀY ĐƯỢC DỰ BÁO — một ngày lịch thật, KHÔNG phải "ngày thứ mấy kể từ lúc chạy".** Đây là lỗi đọc phổ biến
nhất ở bảng này. Mẻ 12/08 đẻ ra 28 dòng mang `horizon_day` = 13/08, 14/08, …, 09/09.

Sinh ra ở `forecast_run.py:1319-1321`:

```python
for i, (p10, p50, p90) in enumerate(daily_raw[:HORIZON_DEFAULT]):
    day = today + timedelta(days=i + 1)          # i+1: bắt đầu từ NGÀY MAI
```

`i + 1` nghĩa là **không bao giờ dự báo cho hôm nay** — hôm nay đã đang xảy ra, dự báo nó thì vô nghĩa.
Tầm nhìn cố định **28 ngày** (`HORIZON_DEFAULT = 28`, `:43`) — khoảng 4 tuần, đủ cho một chu kỳ đặt hàng.

---

**6-7-8. `p10` · `p50` · `p90` · `numeric` · CÓ THỂ NULL**

Ba phân vị của **cầu ngày đó**, đơn vị **cái**. Nghĩa xác suất đã mổ ở CỤM 2 mảnh 4.

- **`numeric` chứ không phải `float`** — cùng lý do với `units_sold`: thập phân chính xác, không trôi số.
- **Số lẻ là bình thường** (`p50 = 3,033`): đây là kỳ vọng của một biến ngẫu nhiên. Làm tròn là việc của người
  đặt hàng ở bước cuối.
- **Luôn thoả `p10 ≤ p50 ≤ p90`** — được ép bằng `sort_quantiles` sau mọi phép nhân (`:328`), không phải trông
  chờ model tự ngoan.
- **Cho phép NULL nhưng thực tế luôn có đủ ba** — model nào cũng phải trả về ba số; không có đường ghi thiếu.

---

**9. `model_used` · `text` · CÓ THỂ NULL**

Model nào **thực sự** đẻ ra ba con số này — không phải model được *chọn*, mà model đã *chạy* (hai thứ có thể
khác nhau: lgbm được chọn nhưng bundle chưa train xong thì rơi về đường thống kê).

Phân bố thật của mẻ `r_2026-08-12` trên `demoshop`:

| `model_used` | dòng | SKU | `calibration` NULL |
|---|---|---|---|
| `lgbm_global` | 1.792 | 64 | 0 |
| `autoets_theta_ensemble` | 532 | 19 | 0 |
| `cold_start` | 504 | 18 | **504 (toàn bộ)** |
| `croston_auto` | 476 | 17 | 0 |
| `seasonal_naive` | 252 | 9 | 28 (1 SKU) |
| `imapa` | 84 | 3 | 0 |
| `adida` | 56 | 2 | 0 |

Đọc ra ba điều: (a) **một shop dùng 7 model cùng lúc** — mỗi SKU một bản chất thống kê khác nhau, đúng tinh
thần §2; (b) `lgbm_global` phủ gần một nửa SKU; (c) **18 SKU vẫn là `cold_start`** — hàng mới, chưa đủ lịch sử.

---

**10. `data_window` · `text` · CÓ THỂ NULL**

Cửa sổ dữ liệu model đã học, viết thành chuỗi `"ngày_đầu..ngày_cuối"` (`forecast_run.py:1326`):

```
2026-04-01..2026-08-12        →  134 ngày lịch sử
```

Nó lấy thẳng ngày đầu và ngày cuối của chuỗi `demand_daily` đã nạp (`series[0]["day"]`, `series[-1]["day"]`).

> **Vì sao lưu một chuỗi thay vì hai cột `date`?** Đây là cột để **người đọc** truy vết, không phải để máy lọc.
> Đánh đổi có thật: muốn hỏi *"mẻ nào học ít hơn 60 ngày"* thì phải cắt chuỗi — bất tiện. Đổi lại, thêm dạng
> cửa sổ mới sau này (ví dụ có lỗ hổng ở giữa) không phải đổi schema.

---

**11. `calibration` · `jsonb` · CÓ THỂ NULL** — hai con số hiệu chỉnh, đã mổ kỹ ngay phía trên. Bổ sung ba điều
về **khi nào nó NULL** (`forecast_run.py:1328-1333`):

```python
calibration = None
if empirical_cov is not None:
    calibration = {"empirical_coverage": empirical_cov, "width_factor": width_factor}
```

`empirical_coverage` chỉ có khi **backtest đã chấm SKU đó**. Nên `calibration IS NULL` đọc thành:
*"SKU này chưa từng được backtest chấm"* — đúng 18 SKU `cold_start` (hàng mới, chưa đủ dữ liệu để backtest)
cộng 1 SKU `seasonal_naive` (router tự chọn, backtest chưa tới lượt).

**Cạm bẫy:** `calibration IS NULL` **không** có nghĩa "khoảng chuẩn". Nó nghĩa là **chưa ai đo** — mà chưa đo
thì không biết khoảng rộng hay hẹp. Trung thực hơn là bịa `width_factor = 1.0`, nhưng người đọc phải hiểu đúng.

---

**12. `created_at` · `timestamptz` · BẮT BUỘC · mặc định `now()`**

Lúc dòng được **ghi**. Khác `run_id` (chỉ có ngày) ở chỗ nó có giờ-phút-giây, và khác `horizon_day` (ngày được
dự báo) hoàn toàn. Ba mốc thời gian trong cùng một dòng, đừng lẫn:

| Cột | Trả lời câu |
|---|---|
| `run_id` = `r_2026-08-12` | mẻ **nào** |
| `created_at` = 12/08 03:54:52 | ghi xuống **lúc nào** |
| `horizon_day` = 13/08 | dự báo **cho ngày nào** |

`created_at` cũng là cột mà chỉ mục `idx_forecasts_project_product_created` bám vào — đúng câu hỏi mà tầng 4
hỏi mỗi lần có người gọi API: *"shop này, SKU này, mẻ mới nhất đâu?"*.

---

#### Bảng tra nhanh cả 12 cột

| Cột | Kiểu | NULL? | Nghĩa một dòng | Ví dụ thật |
|---|---|---|---|---|
| `id` | bigint | không | số thứ tự, không nghĩa nghiệp vụ | — |
| `project_id` | text | không | shop nào (cột cách ly + RLS) | `demoshop` |
| `product_id` | text | không | SKU nào | `ld-srm-cerave` |
| `run_id` | text | không | **mẻ chạy**, `r_` + ngày | `r_2026-08-12` |
| `horizon_day` | date | không | **ngày được dự báo** (không phải offset) | `2026-08-13` |
| `p10` | numeric | có | kịch bản ế | `1,092` |
| `p50` | numeric | có | trung vị | `3,033` |
| `p90` | numeric | có | kịch bản chuẩn bị hàng | `4,974` |
| `model_used` | text | có | model **đã chạy** thật | `autoets_theta_ensemble` |
| `data_window` | text | có | cửa sổ đã học, dạng chuỗi | `2026-04-01..2026-08-12` |
| `calibration` | jsonb | **có** | 2 số hiệu chỉnh; NULL = **chưa backtest chấm** | `{"width_factor":0.647,…}` |
| `created_at` | timestamptz | không | lúc ghi dòng | `2026-08-12 03:54:52+00` |

#### Bảng này lớn cỡ nào, và ai dọn?

Đo thật 2026-08-12: **277.487 dòng · 72 MB · 10 mẻ**.

> 🔴 **Không ai dọn cả.** `save_run` chỉ xoá dòng **cùng `run_id`** (tức ghi đè mẻ trong ngày); không có vòng
> nào xoá mẻ cũ. Trong khi đó bảng `job_runs` **có** vòng dọn riêng (`start_job_runs_retention_loop`,
> `main.py:179`). Tức dự án đã biết làm retention — chỉ là chưa làm cho `forecasts`.
> Phép tính tăng trưởng: `số SKU × 28 dòng × mỗi ngày một mẻ`. Với 1.793 SKU đang có, đó là **~50.000 dòng/ngày**
> ⇒ khoảng **18 triệu dòng/năm**. Đã đặt tên trong DB tri thức: **`W-FORECASTS-NO-RETENTION`** (đang mở —
> xoá dữ liệu là việc phải hỏi chủ dự án trước, không được tự quyết).

---

<a id="h"></a>

## H · TẦNG 4 — NĂM CỬA ĐỌC + BA PHÉP BIẾN ĐỔI

| Cửa | Câu hỏi nghiệp vụ nó trả lời |
|---|---|
| `forecast:query` | "SKU X, 14 ngày tới bán bao nhiêu?" — đọc thẳng `forecasts` |
| `forecast:aggregate` | "**Cả ngành hàng sữa** tháng tới bao nhiêu?" — cộng nhóm |
| `forecast:promo-preview` | "Nếu tôi giảm 20% tuần sau thì bán được bao nhiêu?" — thử **what-if**, **không ghi DB** |
| `forecast:insights` | "SKU nào đang lên/xuống? Thứ mấy bán chạy? Bao lâu hết hàng tồn?" |
| `scenarios:*` | "Cho tôi **1000 kịch bản** để tính rủi ro" — Monte Carlo (Bài 8) |

Ba phép biến đổi thêm khi đọc:

1. **`apply_calendar`** — nhân hệ số lịch Việt Nam (Tết 3 pha: trước Tết tăng, trong Tết sụt, sau Tết hồi). Làm **lúc đọc**, không nướng sẵn vào bảng, để sửa lịch không phải chạy lại toàn bộ dự báo. **Trừ `lgbm_global`** — model này đã mang lịch trong feature, nhân nữa là đếm hai lần.
2. **Tổng hợp horizon** — hỏi "14 ngày tới tổng bao nhiêu" thì **không được cộng p90 của 14 ngày**: cộng như vậy là giả định 14 ngày cùng xấu một lúc — cực hiếm, ra số thổi phồng. Phải mô phỏng: triangular Monte Carlo, hoặc NBD horizon-sim cho hàng bán lai rai.
3. **`reconcile`** — ép tổng các SKU khớp với dự báo cấp ngành hàng (MinT/WLS).

> Điểm chung của cả ba: chúng **biến đổi số đã có**, không gọi model. Đó là lý do tầng 4 nhanh và tất định.

---

---

## 0.3 · BA JOB NỀN — CHU KỲ VÀ VÌ SAO LÀ CHU KỲ ĐÓ

Đo lại trực tiếp trong code ngày 2026-08-10, không tin file:

| Job | Biến môi trường | Mặc định | Quy ra | Dòng code |
|---|---|---|---|---|
| Rollup | `ROLLUP_INTERVAL` | 3.600 s | mỗi **1 giờ** | `rollup.py:272` |
| Forecast run | `FORECAST_RUN_INTERVAL` | 86.400 s | mỗi **1 ngày** = 3.600×24 | `forecast_run.py:1666` |
| Backtest | `BACKTEST_INTERVAL` | 604.800 s | mỗi **7 ngày** = 86.400×7 | `backtest_run.py:514` |

- **Rollup 1 giờ** — đơn vị bài toán là NGÀY. Dày hơn 1 giờ không tạo thêm thông tin, chỉ tốn CPU; thưa hơn thì dữ liệu hôm nay chưa sẵn khi có người gọi API.
- **Forecast 1 ngày** — mỗi ngày mới chỉ có đúng 1 điểm dữ liệu mới trên mỗi SKU. Chạy 2 lần/ngày cho kết quả gần như y hệt với gấp đôi chi phí.
- **Backtest 7 ngày** — backtest không sinh dự báo cho khách; nó trả lời "model nào đang đúng nhất cho SKU này". Câu trả lời đó đổi rất chậm, mà chi phí lại đắt nhất trong ba.

Ba con số này là một **đánh đổi được viết ra**, không phải số ngẫu nhiên: chi phí tính toán ↔ độ tươi của lựa chọn model.

---

## 0.4 · THANG 9 MODEL VÀ THỨ TỰ AI THẮNG AI

```
kv_state model_choice   (do BACKTEST ghi)   ─── THẮNG TUYỆT ĐỐI
        │ không có / SKU chưa đủ dữ liệu để backtest
        ▼
core/router.py          (chọn theo phân loại ADI/CV²)
```

Chín model trên thang: `cold_start` · `seasonal_naive` · `autoets_theta_ensemble` · `croston/sba` · `adida` · `imapa` · `lgbm_global` · `similar_item_transfer` · `cold_start_analog`.

> **Bằng chứng đo được (backtest) đè lên luật tay (router).** Router chỉ là phương án khi chưa có bằng chứng.

Đây đúng kiểu tư duy mà LUẬT-0 của dự án áp cho con người — ở đây được cài thẳng vào máy.

---

## 0.5 · BA THUẬT TOÁN CÓ CODE NHƯNG KHÔNG CHẠY

| # | Thuật toán | Tình trạng |
|---|---|---|
| 31 | PSI drift — `libs/featurelib/drift.py:17` | **không nối vào BT03** |
| 32 | Data-quality gate — `libs/featurelib/data_quality.py:11` | **không nối vào BT03** (§13.1) |
| 33 | Holiday factor hardcode VN — `libs/featurelib/holidays_vn.py:69` | **đã nghỉ hưu**, bị `calendar_effect` thay |

Giá trị của ba dòng này: nó chứng minh tài liệu được viết **từ code** chứ không từ ý định. 30/33 thuật toán chạy thật; 3 cái không, và nói rõ. Khi phỏng vấn, đây là thứ phân biệt người đọc code với người đọc slide.

---
---

# HỎI & ĐÁP — MỔ BA CỤM TỪ TRONG CÂU ĐỊNH NGHĨA

---

## CỤM 1 · "event bán hàng lẻ tẻ theo giây"

### Event là gì — và vì sao không phải "dữ liệu bán hàng"

**Event = bản ghi một sự việc đã xảy ra, có dấu thời gian, không sửa được về sau.** Khác hẳn "bảng doanh số": bảng là *trạng thái tổng kết*, event là *sự việc đơn lẻ*.

miniAI nhận đúng **5 loại** (`jobs/rollup.py:88-130`):

| event_type | Kể chuyện gì | Rollup làm gì |
|---|---|---|
| `purchase.completed` | 1 đơn đã thanh toán | `units += qty`, cộng dồn tiền để tính giá bình quân |
| `order.returned` | khách trả hàng | `units -= qty` vào **ngày TRẢ** |
| `stock.out` | SKU hết hàng | bật cờ `stockout = true` |
| `price.changed` | đổi giá niêm yết | ghi mốc `(ngày, giá mới)` |
| `promo.scheduled` | lên lịch khuyến mãi | tô `promo_pct` cho mọi ngày trong `[start, end]` |

Vì sao chọn kiến trúc event chứ không bắt khách gửi báo cáo ngày:

1. **Khách không phải xử lý trước.** Họ bắn cái họ có sẵn tại thời điểm xảy ra. Bắt khách tổng hợp = bắt khách viết một nửa hệ thống của mình.
2. **Một luồng nuôi cả 3 bài toán.** Cùng dòng event đó: BT01 (smartsearch) học cái gì bán chạy, BT02 (decision) chấm kết quả quyết định giá, BT03 dự báo. Mỗi bài một định dạng nhập thì chi phí tích hợp nhân ba.
3. **Sửa được quá khứ mà không nói dối.** Event bất biến; muốn sửa thì bắn event mới (`order.returned`). Bảng tổng kết bị ghi đè thì lịch sử biến mất.
4. **Chạy lại được (idempotent).** Rollup UPSERT theo khoá `(project_id, product_id, day)` (`rollup.py:241-256`) ⇒ chạy 1 lần hay 5 lần cho **cùng một kết quả**. Đây là tính chất bắt buộc của mọi job nền: job chết giữa chừng, chạy lại, không được nhân đôi số liệu.

### "Lẻ tẻ" — ba kiểu bất quy tắc, mỗi kiểu một hậu quả

**(a) Bất quy tắc theo thời gian.** Một SKU có thể nhận 40 event trong 1 phút (giờ vàng), rồi im 3 ngày. Không có nhịp cố định.

**(b) Bất quy tắc theo SKU.** SKU hot có event mỗi ngày; SKU đuôi dài có 6 event trong 90 ngày. Cùng một bảng, hai bản chất thống kê khác nhau hoàn toàn — chính điều này đẻ ra §2 (phân loại ADI/CV²) và cả nhánh model gián đoạn (Croston/ADIDA/IMAPA).

**(c) Vắng mặt ≠ thiếu dữ liệu.** Đây là chỗ chết người nhất. Ngày không có event `purchase.completed` cho SKU X nghĩa là **bán 0 cái** — một con số THẬT — chứ không phải "không biết". Nếu để trống, thư viện thống kê sẽ nội suy hoặc bỏ qua; chuỗi bị bóp méo theo hướng **thổi phồng mức nền**.

Chính vì (c) mà `run_rollup_once` không chỉ nhóm event lại, mà **lặp từng ngày** từ `min_day` tới `today` cho từng cặp `(project, product)` và điền `u = max(0.0, units.get(key, 0.0))` (`rollup.py:165,172`).

> ⚠ **CHI TIẾT CHÌM TRONG DÒNG 172.** `max(0.0, …)` — **cầu âm bị kẹp về 0**. Hôm nay 5 khách trả hàng mua từ tuần trước và không ai mua mới ⇒ `units = −5` ⇒ ghi 0.
> **Đánh đổi:** thà mất thông tin "ngày trả hàng nhiều" còn hơn để số âm chảy vào model (log của số âm, phương sai bịa, quantile vô nghĩa).

### "Theo giây" — lệch đơn vị và ba phép vá

`raw_events.event_time` chính xác tới **giây**. Đơn vị bài toán là **ngày**.

**LỆCH 1 — lệch lưới (giây → ngày):** gộp. Ví dụ phép gộp giá (`rollup.py:88-99`):

| Giờ | qty | unit_price |
|---|---|---|
| 09:14:03 | 2 | 100.000 |
| 09:14:55 | 1 | 100.000 |
| 17:02:11 | 3 | 90.000 |

```
units_sold = 2 + 1 + 3                          = 6
price_sum  = 2·100.000 + 1·100.000 + 3·90.000   = 570.000
price_qty  = 6
price      = 570.000 / 6                        = 95.000   ← bình quân GIA QUYỀN theo số lượng
```

Không phải trung bình cộng của 3 mức giá (`(100+100+90)/3 = 96.667`). Gia quyền theo lượng mới đúng nghĩa "giá hiệu lực của một cái bán ra ngày đó".

Ngày **không có** giao dịch thì không có giá bình quân → **carry-forward** mốc `price.changed` gần nhất (`rollup.py:178-181`); chưa từng có mốc nào thì để `NULL`.

**LỆCH 2 — lệch ngữ nghĩa:** hàng trả bị trừ vào **ngày trả**, không phải ngày mua (`rollup.py:100-109`). Đây là **lựa chọn có chủ ý**, không phải lỗi:

- Trừ vào ngày mua = *đúng hơn về kế toán*, nhưng **viết lại quá khứ**: chuỗi hôm qua đổi số sau khi model đã học và đã ra quyết định nhập hàng. Không tái lập được, không audit được.
- Trừ vào ngày trả = *hơi sai về kế toán*, nhưng chuỗi **chỉ tiến về phía trước** (append-only). Đây cũng là lý do có `max(0.0, …)` ở trên.

**LỆCH 3 — ranh giới ngày.** Sự kiện lúc 23:59:59 thuộc ngày nào, theo múi giờ nào? File khai một cảnh báo thật ở §11.5 về "kiểu ngày" trong `demand_daily` của tenant `p1`. Loại lỗi này không nổ, chỉ làm lệch chuỗi 1 ngày cho một tenant.

> ⚠ **CẠM BẪY ĐÃ TRẢ GIÁ THẬT — CỬA SỔ 120 NGÀY.**
> `run_rollup_once(pool, window_days=120)` chỉ đọc `raw_events` có `event_time >= now − 120 ngày` (`rollup.py:57-66`). Comment `rollup.py:49-52` ghi lại vụ **SIM-WORLD SW-1**: đường backfill nạp lịch sử dài hơn 120 ngày, rollup chạy với mặc định ⇒ **cắt mất phần lịch sử vừa nạp**. Không exception, không log đỏ.
> **Dạng tổng quát (dùng được khi phỏng vấn):** tham số mặc định an toàn cho đường chạy hằng ngày lại là con dao cho đường nạp lại lịch sử. Hai đường dùng chung một hàm, mặc định chỉ đúng cho một đường.

---

## CỤM 2 · "phân phối cầu tương lai theo ngày cho từng SKU"

### Mảnh 1 — "cầu": chữ quan trọng nhất cả câu

> **Cầu (demand) ≠ Bán (sales).** Quan hệ thật: `bán = min(cầu, tồn kho)`.

Ngày hết hàng, `units_sold = 0` — nhưng cầu ngày đó **không** bằng 0. Số liệu bán bị **kiểm duyệt** (censored) bởi chính tồn kho của mình. Đây là bẫy kinh điển:

```
hết hàng → bán 0 → model học "cầu giảm" → dự báo thấp → nhập ít → hết hàng sớm hơn → …
```

Vòng xoáy tự xác nhận. Đó là lý do có §1.2 (`rollup.py:211-238`):

```
adj[t] = max( units_sold[t], mean(adj[t−7 .. t−1]) )   nếu stockout[t] = true
adj[t] = units_sold[t]                                 nếu không
```

Ví dụ số: 7 ngày trước có `adj = [12, 9, 11, 10, 13, 8, 10]` → `mean = 73/7 = 10,43`.

| Tình huống hôm nay | Phép tính | adjusted_units |
|---|---|---|
| Hết hàng, bán được 3 rồi sạch kệ | `max(3 , 10,43)` | **10,43** |
| Hết hàng vào cuối ngày, vẫn bán 15 | `max(15 , 10,43)` | **15** (không hạ số thật xuống) |

Hai chi tiết dễ bỏ sót:

- Trung bình lấy trên **`adj` đã tính** (`adjusted_prev[-7:]`, `rollup.py:220`), tức **đệ quy trên chính cột bù** — 3 ngày hết hàng liên tiếp thì ngày thứ 3 lấy trung bình gồm cả hai ngày bù trước đó. Ưu điểm: không bị kéo tụt dần. Rủi ro: nếu cầu thật sự giảm đúng lúc hết hàng, cột bù giữ mức cao lâu hơn thực tế.
- **Mọi model đọc `adjusted_units`, không đọc `units_sold`** (`store/forecasts.py:26`). Câu này nên thuộc lòng — nó là ranh giới "dự báo cầu" vs "dự báo doanh số".

### Mảnh 2 — "tương lai theo ngày"

Bước thời gian = **1 ngày**, tầm nhìn `H` ngày. Bảng `forecasts` lưu **một dòng cho mỗi `(product_id, run_id, horizon_day)`**. Nghĩa là luôn trả lời được: con số này sinh lúc nào, bởi lần chạy nào, với model nào (`model_used`), cửa sổ dữ liệu nào (`data_window`), hệ số hiệu chỉnh nào (`calibration`).

Đây là **tái lập được (reproducibility)** ở mức cột bảng, không phải lời hứa.

### Mảnh 3 — "cho từng SKU"

Dự báo ở **cấp thấp nhất** (SKU), rồi mới cộng lên. Lựa chọn này đẻ ra một bài toán mới, giải ở §6:

> Tổng dự báo 100 SKU của ngành hàng **không** bằng dự báo trực tiếp của ngành hàng đó.

Người mua hàng nhìn số SKU, giám đốc nhìn số ngành hàng — hai số không khớp là mất niềm tin ngay. Đó là **hierarchical reconciliation** (MinT/WLS, `core/scenario/reconcile.py`).

**Chi phí của việc dự báo cấp SKU là phải hoà giải cấp bậc.**

### Mảnh 4 — "phân phối": chính xác nó là cái gì trong DB

Bảng `forecasts` lưu **đúng 3 con số**: `p10`, `p50`, `p90`.

| Ký hiệu | Đọc là | Nghĩa xác suất |
|---|---|---|
| `p10` | phân vị 10% | 10% khả năng cầu ≤ p10 — kịch bản ế |
| `p50` | trung vị | 50/50 — cầu ở trên hay dưới đều 50% |
| `p90` | phân vị 90% | **90% khả năng cầu ≤ p90** — kịch bản chuẩn bị hàng |

Đọc thành lời: `p90 = 25` ⇒ *"trong 10 ngày giống ngày này, khoảng 9 ngày cầu không vượt 25 cái."*

Còn "**kèm p95/p99 khi được hỏi**" phải hiểu thật thà: **p95/p99 KHÔNG được lưu**, chúng được **ngoại suy** từ 3 điểm bằng giả định đuôi lognormal (`quantiles.py:80`):

```
z(0,90)=1,2816   z(0,95)=1,6449   z(0,99)=2,3263
σ_log = ln(p90 / p50) / 1,2816
p_q   = p50 · exp( z_q · σ_log )
```

Chạy số với `p50 = 10`, `p90 = 18`:

```
σ_log = ln(18/10) / 1,2816 = ln(1,8) / 1,2816 = 0,58779 / 1,2816 = 0,45864
p95   = 10 · exp(1,6449 × 0,45864) = 10 · exp(0,75447) = 10 × 2,1265 = 21,27
p99   = 10 · exp(2,3263 × 0,45864) = 10 · exp(1,06699) = 10 × 2,9067 = 29,07
```

Ba điều rút ra:

1. **p99 ≈ 2,9 × p50** — đuôi dài hơn nhiều so với trực giác tuyến tính. Ai đặt hàng theo "p50 nhân đôi cho chắc" đang đặt **dưới** p99.
2. API **chỉ cho** hỏi `q ∈ {0,95 ; 0,99}` (`main.py:598-608`) — càng xa 3 điểm neo thì càng là **giả định**, không phải đo.
3. Response gắn nhãn `quantile_method = "lognormal_tail_extrapolation"` (`main.py:741`). **Hệ thống tự khai con số này là suy ra, không phải đo ra.** Chuẩn mực đáng học: đừng để người dùng không phân biệt được số đo với số suy.

Tương tự, p25/p75 là **nội suy tuyến tính** giữa 3 điểm (`quantile_interp`, `quantiles.py:12`).

> **GIỚI HẠN THẬT THÀ.** Cái BT03 trả về là **phác thảo 3 điểm của một phân phối** cộng luật nội suy/ngoại suy công khai — **không phải** phân phối đầy đủ. Muốn phân phối đầy đủ + **quan hệ phụ thuộc giữa các SKU và các ngày** (hôm nay ế thì mai có ế theo không?) thì phải sang **Scenario Fabric** §7 (Monte Carlo, copula). Đó là lý do §7 tồn tại — không phải đồ trang trí.

---

## CỤM 3 · Ranh giới nghiệp dư / chuyên nghiệp: "đầu ra là phân phối, không phải điểm"

Đây là luận điểm, nên chứng minh bằng 4 lập luận có số.

### Lập luận 1 — dự báo điểm **giấu** rủi ro 50%

Model nói "bán khoảng 10 cái/ngày", bạn nhập 10.

Con số 10 đó là **trung vị**. Theo đúng định nghĩa trung vị: **50% số ngày cầu sẽ vượt 10** ⇒ bạn **đứt hàng 50% số ngày**. Không ai chấp nhận mức phục vụ 50%, nhưng đó chính xác là điều xảy ra khi đặt hàng theo dự báo điểm.

Dự báo điểm không sai — nó **không có chỗ để chứa mức phục vụ**. Đơn vị thông tin bị thiếu.

### Lập luận 2 — chi phí bất đối xứng: **p90 không phải con số tuỳ tiện**

Bài toán người bán báo (newsvendor):
- `Cu` = chi phí thiếu 1 đơn vị (mất lãi + mất khách)
- `Co` = chi phí thừa 1 đơn vị (vốn đọng + lưu kho + rủi ro hỏng/lỗi mốt)

Mức phục vụ tối ưu = **tỷ số tới hạn**:

```
CR = Cu / (Cu + Co)   →  đặt hàng ở phân vị CR
```

| Ca | Cu | Co | Phép tính | Đặt ở phân vị |
|---|---|---|---|---|
| **A** — hàng lâu hỏng, biên lãi tốt | 30.000 | 3.000 | `30/(30+3) = 0,909` | ~91% ≈ **p90** |
| **B** — hàng tươi, hỏng trong ngày | 10.000 | 40.000 | `10/(10+40) = 0,20` | 20% = **p20**, *dưới* trung vị |

Ca A chính là **nguồn gốc thật của con số p90** — nó rơi ra từ tỷ lệ chi phí, không phải quy ước. Ca B: với rau tươi, quyết định đúng là nhập **dưới** trung vị và chấp nhận hết sớm.

> Một con số điểm **không thể** diễn đạt được cả ca A lẫn ca B. Cùng một dự báo "10 cái", hai ngành hàng phải đặt hai lượng khác nhau. Chỉ **phân phối** mới cho khách tự chọn phân vị theo cấu trúc chi phí của họ.

Đây cũng là chỗ BT03 nối vào BT02: forecast đưa phân phối, decision đưa chính sách chọn phân vị.

### Lập luận 3 — dự báo điểm **giấu độ rộng bất định**

| SKU | p10 | p50 | p90 | p90 − p10 |
|---|---|---|---|---|
| A | 8 | **10** | 12 | 4 |
| B | 0 | **10** | 40 | 40 |

Cùng dự báo điểm **10**. Nhưng:
- SKU A: nhập 12 là gần như an toàn, tồn kho an toàn nhỏ, ít vốn chôn.
- SKU B: muốn 90% phục vụ phải nhập 40 — gấp 4 lần trung vị. Vốn chôn khổng lồ; đây là SKU đáng cân nhắc **đổi chiến lược** (đặt theo đơn, hoặc chấp nhận mức phục vụ thấp hơn).

Dự báo điểm làm hai SKU này **trông y hệt nhau**. Mất mát thông tin dẫn thẳng tới quyết định sai.

### Lập luận 4 — hệ quả **đo lường**: phân phối buộc phải chấm bằng thước khác

| Kiểu đầu ra | Chấm bằng | Trả lời câu hỏi |
|---|---|---|
| Điểm | MAE, RMSE, **MASE** | "Dự báo lệch bao nhiêu?" |
| Phân phối | **Coverage**, **Pinball loss** | "Khoảng có **trung thực** không? p90 tôi hứa có đúng 90% không?" |

Vì thế dự án **tách hai họ metric** (§14.4) và ghim **hai** baseline chứ không một (§11.0, DB tri thức `facts`):

| fact key | Giá trị | Đọc là |
|---|---|---|
| `eval.forecast.mase.baseline` | **0,827** | sai số bằng 82,7% so với thước nền `snaive7` (naive mùa vụ tuần) |
| `eval.forecast.coverage.baseline` | **0,815** | 81,5% giá trị thật rơi vào `[p10, p90]`; đích lý thuyết 0,80 |
| `eval.baseline.tolerance` | 0,10 | dung sai chung của gate eval |

Bây giờ đọc bảng thật §11.2 bằng con mắt vừa học:

| project | model | coverage | MASE | Đọc ra |
|---|---|---|---|---|
| demoshop | `croston_auto` | **0,938** | 0,905 | khoảng **QUÁ RỘNG** — hứa 80% mà bao 93,8% |
| demoshop | `lgbm_global` | **0,784** | **0,782** | gần đích 0,80 nhất, sai số thấp nhất |
| demoshop | `seasonal_naive` | 0,692 | (nền) | khoảng **QUÁ HẸP** — hứa 80% chỉ bao 69,2% |

> ⚠ **CHỖ DỄ LỘ NGHIỆP DƯ NHẤT.** Người mới nhìn `coverage = 0,938` sẽ khoe "bao phủ 93,8%, tốt hơn 80%!". **Sai.** Coverage cao hơn đích là **lỗi hiệu chuẩn**, không phải thành tích: khoảng quá rộng ⇒ p90 bị thổi ⇒ khách nhập thừa ⇒ chôn vốn. Một khoảng `[0, ∞)` có coverage 100% và vô dụng 100%.
> Coverage là chỉ số **hai chiều đều xấu** — chệch lên cũng sai, chệch xuống cũng sai; đích là **đúng bằng lời hứa**. Chính vì `croston_auto` rộng như vậy nên nó là nhóm bị `width_factor` **nén** xuống sàn 0,5 ở §8.3.

### Tóm tắt ranh giới

| | Nghiệp dư | Chuyên nghiệp (BT03 đang làm) |
|---|---|---|
| Đầu ra | 1 con số | p10/p50/p90 (+p95/p99 khai rõ là ngoại suy) |
| Học từ | `units_sold` | `adjusted_units` — bù kiểm duyệt bởi tồn kho |
| Chấm bằng | MAE/MASE | MASE **và** coverage **và** pinball |
| "Coverage 0,94" | khoe | **báo động khoảng quá rộng** |
| Chọn model | 1 model cho mọi SKU | backtest per-SKU; bằng chứng đè luật tay |
| Số suy ra | trộn lẫn với số đo | gắn nhãn `lognormal_tail_extrapolation` |

---
---

# PHÁT SINH TRONG BUỔI HỌC — KHUYẾT TẬT "CHẠY LẠI TỪ ĐẦU MỖI LẦN KHỞI ĐỘNG"

Câu hỏi trong lớp — *"không lẽ mỗi lần demo lại phải mất công như vậy?"* — dẫn thẳng tới một lỗi sản phẩm thật, ghi trong DB tri thức là `W-JOB-SCHEDULE-STATE-ANCHOR`.

Cả ba loop nền đều viết theo khuôn *chạy ngay rồi mới ngủ*:

```python
while not stop_event.is_set():
    await run_backtest_once(pool)          # ← CHẠY NGAY, không hỏi gì
    await asyncio.wait_for(stop_event.wait(), timeout=interval)   # ← rồi mới ngủ
```

`backtest_run.py:514-522` · `forecast_run.py:1666-1674` · `rollup.py:272-280`, được `main.py:159,165,171` bật khi service khởi động.

**Điểm gốc của lịch là "lúc tiến trình sinh ra", không phải "lúc việc được làm xong lần cuối".** Container không nhớ gì về quá khứ — mỗi lần boot, nó tin rằng chưa ai từng chạy backtest.

| | Hiện tại | Đúng ra phải là |
|---|---|---|
| Backtest chạy lần cuối | 2026-08-06 10:20 (đọc `job_runs`) | — |
| Chu kỳ | 604.800 s = 7 ngày | 7 ngày |
| Boot bây giờ ⇒ | **chạy lại full** (đo thật 3.754 s / 180 SKU simworld1) | **không chạy gì**, ngủ tới đúng hạn |

Nó tệ hơn "bất tiện lúc demo": mỗi lần **deploy** là tự nã CPU vào chính mình; container bị OOM-kill lúc 2 giờ sáng là tự gây tải đỉnh đúng lúc không ai trực; restart 5 lần trong 1 giờ là chạy backtest 5 lần cho cùng một kết quả.

**CÁCH SỬA — neo lịch vào TRẠNG THÁI, không neo vào tiến trình.** Nguyên liệu đã có sẵn: bảng `job_runs` và index `idx_job_runs_status_finished_at`. Trước khi chạy, loop hỏi Postgres còn bao lâu tới hạn; chưa tới hạn thì ngủ, không làm gì.

- Toàn bộ phép tính thời gian nằm **trong Postgres** — một đồng hồ duy nhất, lệch giờ container không park nổi loop.
- Đọc lỗi thì **fail-open** (chạy) — hành vi degrade đúng bằng hành vi cũ, không bao giờ làm dữ liệu cũ đi.
- Kill-switch `FORECAST_JOB_SCHEDULE_ANCHOR=0` trả về hành vi cũ mà không cần build lại.
- Marker của loop (`<job>_loop`) **tách khỏi** dòng `job_runs` của job sản phẩm, vì `forecast_run`/`backtest_run` cũng được ghi bởi lần chạy **một tenant** từ API — một tenant xong không được phép thoả mãn lịch của loop phủ mọi tenant.

### ĐÃ SỬA VÀ ĐÃ ĐO — 2026-08-10

Fix đã land trong `services/forecast/app/jobs/schedule.py` (+ 3 loop chuyển sang `run_scheduled_loop`).

**Lần boot ĐẦU sau khi deploy** (chưa có marker nào ⇒ đúng thiết kế: "chưa từng chạy thì chạy") —
đây cũng là lần đầu tiên `job_runs` ghi được **thời lượng thật** thay vì 0 giây:

| Job | Thời lượng thật | Khối lượng |
|---|---|---|
| `rollup_loop` | **2 s** | 1.160 SKU / 49.640 ngày-SKU |
| `forecast_run_loop` | **334 s** (5,6 phút) | 1.793 SKU → 50.204 dòng dự báo |
| `backtest_run_loop` | **593 s** (9,9 phút) | 1.146 SKU → 4.448 dòng |

Ba job chạy song song ⇒ wall-clock boot đầu = 01:41:24 → 01:51:21 = **9 phút 57 giây**.
Đây chính là con số "vài chục phút" mà trước nay **không ai đo được**, vì mọi dòng `job_runs` đều khai 0 giây.

**Lần restart THỨ HAI** (marker còn tươi) — phép đo quyết định:

| | Trước fix | Sau fix (đo 01:56–01:59) |
|---|---|---|
| Restart khi chưa tới hạn | chạy lại full, CPU 782–1294% | **không chạy gì**, CPU **0,24–0,52%** |
| Số marker `*_loop` sau restart | — | vẫn **3**, không sinh thêm dòng nào |
| Thời gian tới `healthz` 200 | vài chục phút mới hết tải | **14 giây** |

Kiểm chứng bằng test: 13/13 xanh trong `tests/forecast/test_job_schedule.py`, gồm **một test tự-phá**
(`test_skip_is_caused_by_the_gate_not_by_luck`): cùng marker tươi đó, tắt cổng bằng
`FORECAST_JOB_SCHEDULE_ANCHOR=0` thì job **có** chạy — chứng minh việc bỏ qua là do cổng, không phải do may.
Suite forecast: 264 pass / 0 fail.

Nợ kèm theo, đã đặt tên `W-JOBRUNS-DURATION-ZERO`: `backtest_run.py` đặt `started_at` và `finished_at` **cùng lúc ở cuối job** ⇒ mọi dòng `job_runs` khai thời lượng ≈ 0; cộng với `except Exception: pass` ở cả ba loop, một job có thể **chết cả tuần mà không ai biết**.

---

# PHÁT SINH TRONG BUỔI HỌC (2) — NGÀY SALE BỊ XOÁ ÂM THẦM

> Câu hỏi *"hệ số khuyến mãi của shop là gì?"* dẫn tới một lỗi sản phẩm thật, tìm ra **trong lúc soạn mục F.1**
> của chính bài này. Đã vá và đã đo xong cùng ngày. Tên trong DB tri thức: **`W-ROLLUP-PROMO-WINDOW-ERASE`**.

## Con số không khớp

Đang viết mục F.1, tôi tái lập lại `k` của `demoshop` bằng SQL để đối chiếu với số đã lưu trong `kv_state`:

| | `k` | số SKU | số ngày sale |
|---|---|---|---|
| Đã lưu trong DB, ghi lúc **03:54** | **0,947408** | 48 | **719** |
| Chạy lại chính hàm đó lúc **06:26** | **0,858374** | 48 | **710** |

Cùng shop, cùng code, cách nhau 2,5 giờ. **Chín ngày sale biến mất** và `k` tụt 9,4%.

Bước đầu tiên là loại trừ khả năng SQL của tôi viết sai — chạy thẳng hàm thật của sản phẩm trong container:

```
estimate_uplift_k = {"k": 0.8583739626929002, "n_skus": 48, "n_promo_days": 710}
```

Khớp SQL tới chữ số thứ 12 ⇒ phép đo đúng, **dữ liệu thật sự đã đổi**.

## Truy ra thủ phạm

Nhìn một đợt sale cụ thể. Event khai giảm 21% cho phấn nước Clio, **từ 11/04 đến hết 15/04**:

| ngày | 11/04 | 12/04 | 13/04 | **14/04** | **15/04** |
|---|---|---|---|---|---|
| `promo_pct` trong bảng | 21 | 21 | 21 | **0** | **0** |

Đợt sale bị **cụt đuôi**. Và mốc cắt 120 ngày của rollup hôm đó rơi đúng vào **14/04**.

Cơ chế, đọc thẳng từ code:

1. Rollup chỉ đọc event trong cửa sổ `event_time >= now − 120 ngày` (`rollup.py:59-68`).
2. Event `promo.scheduled` khai **một lần** ở đầu đợt, nhưng **sơn nhiều ngày về sau** (`:122-132`).
3. Rollup ghi lại mọi ngày từ `min_day` của SKU tới hôm nay (`:158,167`) — với Clio, `min_day` đo được đúng
   bằng **14/04**.
4. UPSERT ghi đè thẳng: `promo_pct = EXCLUDED.promo_pct` (`:254`) — không phải `GREATEST`.

Ghép lại: event khai ngày 11/04 đã **rơi khỏi cửa sổ**, nhưng hai ngày nó sơn (14, 15/04) **vẫn nằm trong tầm
ghi lại** ⇒ rollup ghi đè chúng về 0. Rồi biên trôi qua, hai ngày đó **không bao giờ được ghi lại nữa** —
**đóng băng sai vĩnh viễn**.

Đếm toàn hệ trước khi vá: **55 ngày-SKU** mất sale — cụm dày đúng tại biên `now−120d` (14→18/04), cộng các cụm
cũ đã đóng băng từ những lần biên quét trước (2025-07, 2026-03).

## Vì sao nó nguy hiểm hơn vẻ ngoài

Chuỗi hậu quả đi **xuôi** đúng những gì bài này đã dạy:

```
ngày sale bị xoá nhãn
   → nó thành "ngày thường bán cao"
   → base_mean (bán TB ngày thường) bị THỔI LÊN
   → implied_k = (promo_mean/base_mean − 1)/mức_giảm  TỤT XUỐNG
   → k của cả shop tụt
   → mọi ngày sale TƯƠNG LAI bị dự báo HỤT
   → khách nhập thiếu đúng vào đợt sale — lúc đông khách nhất
```

Không exception, không log đỏ, không cảnh báo. Chỉ là con số **xấu dần theo thời gian**. Đây đúng loại lỗi mà
LUẬT-0 của dự án sinh ra để bắt: gate đo *hàm* thì vẫn xanh, vì mỗi hàm đều đúng.

> **Câu hỏi của học viên đáng ghi lại: "thuật toán bị sai hay dữ liệu bị sai vậy?"**
> **Thuật toán ghi dữ liệu sai ⇒ dữ liệu sai theo.** Ba tầng, phân định rạch ròi:
>
> | | Đúng hay sai | Bằng chứng |
> |---|---|---|
> | `estimate_uplift_k` (công thức học `k`) | **ĐÚNG** | vá xong nó trả lại **đúng từng bit** `0.947408367288503` / 719 ngày |
> | `rollup.py` (code ghi dữ liệu) | **SAI** | đọc promo trong 120 ngày nhưng ghi lại xa hơn thế |
> | `demand_daily` (dữ liệu) | **SAI — là hậu quả** | 55 ngày-SKU, và **đóng băng**, không tự lành |
>
> Model dự báo không hề sai — nó tính trung thực trên đầu vào đã hỏng. Bài học chung: **khi một con số suy ra
> trông lạ, đừng sửa công thức trước; đi ngược về dữ liệu nuôi nó.**

## Thuật toán đúng là gì

Chỗ sai nằm ở **lệch đơn vị**, cùng họ với ba phép lệch đã học ở CỤM 1:

> Cửa sổ `window_days` là tham số của **đường ĐỌC** (để khỏi quét lại cả bảng mỗi giờ), **không phải** định
> nghĩa nghiệp vụ của một đợt sale. Tuổi của **EVENT** và tuổi của **NGÀY BỊ SƠN** là hai thứ khác nhau — lấy
> tuổi event để quyết định ngày đó có sale hay không là **sai đơn vị**.

Cách vá đã chọn: nạp **riêng** những promo cũ hơn cửa sổ, sơn vào một bản đồ riêng, và chỉ **hợp nhất lúc GHI**
bằng `max()` — giữ đúng luật *"lấy sâu nhất"* của promo chồng nhau đã học ở mục 7b.

⛔ **Và một điều tuyệt đối không được làm**, đáng học ngang bản vá: **không** được đưa những ngày promo cũ đó
vào tập tính `min_day`. Nếu đưa, `min_day` lùi về quá khứ ⇒ rollup ghi lại những ngày mà **purchase event của
chúng đã ngoài cửa sổ** ⇒ `units.get(key, 0)` trả 0 ⇒ **xoá sạch doanh số cũ**. Đó đúng là tai nạn SIM-WORLD
SW-1 đã ghi trong docstring của chính hàm này. Sửa một lỗi im lặng bằng cách tạo ra một lỗi im lặng to hơn là
lối mòn rất dễ sa vào.

Hai phương án bị loại, viết ra để sau này khỏi thử lại:

| Phương án loại | Vì sao sai |
|---|---|
| Nới `window_days` cho **mọi** thứ | `min_day` lùi theo ⇒ xoá sạch doanh số cũ (đúng tai nạn SW-1) |
| `GREATEST` trong UPSERT (promo không bao giờ giảm) | không bao giờ **sửa** được khai nhầm, và phá luật `max` theo từng ngày |

Kèm theo bản vá: kill-switch `ROLLUP_PROMO_FULL_HISTORY=0` trả về hành vi cũ không cần build lại · trần
`MAX_PROMO_SPAN_DAYS = 400` chặn payload lỗi làm nở vòng lặp ngày · fail-open nếu đường đọc phụ hỏng (rollup
degrade đúng bằng hành vi cũ, không bao giờ chết) · thêm số đếm `promo_carried` trong kết quả để **hỏng thì
biết bằng số**.

## Đã đo sau khi vá — 2026-08-12

| Phép đo | Trước | Sau |
|---|---|---|
| Ngày sale bị xoá (11 tenant) | **55** | **0** |
| `k` của `demoshop` | 0,858374 (710 ngày) | **0,947408367288503 (719 ngày)** — đúng từng bit giá trị gốc |
| `k` của `simworld2` | 2,9944 | **3,4937** (+16,7%) |
| `k` của `simworld1` | 2,0735 | **2,1628** |
| Test `tests/forecast/test_rollup.py` | 4 pass | **7 pass** (+3, có 1 test **tự-phá**) |
| Toàn suite `tests/forecast` | — | **282 pass / 0 fail** |

Test tự-phá tên `test_promo_carry_is_caused_by_the_gate_not_by_luck`: cùng dữ liệu đó, tắt kill-switch thì
`promo_pct` **về 0 trở lại** — chứng minh việc sửa được là **do bản vá**, không phải do may. Gate không tự phá
được thì gate không tồn tại.

> **Điều đáng nhớ nhất của ca này:** con bug lộ ra không phải nhờ test, không phải nhờ alert, mà nhờ **một câu
> hỏi của học viên** buộc người viết tài liệu phải **tự tính lại một con số bằng tay** rồi đối chiếu với số hệ
> thống tự khai. Đó chính là "phân tích ngược" trong LUẬT-0, làm thủ công.

---

# KIỂM TRA BÀI 0 — 7 CÂU, TRẢ LỜI BẰNG SỐ CỦA CHÍNH DỰ ÁN

1. Khách hỏi: *"Sao không chạy backtest mỗi ngày cho model luôn tươi?"* — trả lời bằng **đánh đổi** kèm con số chu kỳ, và nói rõ backtest ghi cái gì vào đâu.
2. Một SKU đang được phục vụ bởi `croston`. Muốn biết **vì sao là croston** chứ không phải router chọn — tìm ở bảng nào, key nào?
3. Tầng 4 (API đọc) có được phép gọi model để tính lại dự báo không? Trả lời có/không **và** hai hệ quả nghiệp vụ của thiết kế đó: một tốt, một xấu.

**Ba câu mới, theo mục F và G (thêm 2026-08-12):**

4. Shop A có `k = 0,95`, shop B có `k = 3,49`. Cùng giảm **20%** cho một SKU mà mức nền là 10 cái/ngày — mỗi shop dự báo bán bao nhiêu vào ngày sale? Viết ra **phép tính**, rồi nói con số nào **đáng nghi** và vì sao (gợi ý: nhìn lại bảng 10 shop ở F.1).
5. Một SKU ngày sale bán được **8** cái, đang giảm **24,3%**, shop có `k = 0,947`. Model được cho học con số nào? **Vì sao không cho nó học thẳng số 8?** Nếu bỏ hẳn bước này thì dự báo sai theo hướng nào — cao lên hay thấp đi?
6. Mở bảng `forecasts` thấy một dòng `calibration IS NULL`. Đọc ra được điều gì về SKU đó? Và vì sao để **trống** lại trung thực hơn là điền `width_factor = 1.0`?
7. Ba mốc `run_id = r_2026-08-12` · `horizon_day = 2026-08-13` · `created_at = 12/08 03:54` khác nhau chỗ nào? Nếu chạy job **thêm một lần nữa lúc 15:00 cùng ngày** thì bảng có thêm bao nhiêu dòng — và vì sao?
