# BÀI HỌC: BẢN ĐỒ API 3 SERVICE miniAI (xuất 2026-08-06)

> ⚠ File này là BẢN XUẤT ĐỂ ĐỌC (training human, demo đối tác). NGUỒN SỰ THẬT = DB tri thức
> (`python3 rail.py q/graph` — kb_surface 58 · kb_feature 46 · kb_domain 3) + `project/openapi/*.json` (codegen từ code).
> Số đo tại thời điểm xuất: check-apis **42/42 PASS, 0 SKIP** · QC V4 16/16 · 1.263 test per-service.

---

## 1. TỔNG QUAN: 52 ENDPOINT, 3 VAI TRÒ A/B/C

Tổng **52 endpoint** (smartsearch 17 · decision 17 · forecast 18). Mỗi service có 4 endpoint hạ tầng
(`/healthz` `/readyz` `/metrics` `/v1/ping` — cho Docker/Prometheus, không phải nghiệp vụ).
Còn lại **~40 endpoint nghiệp vụ**, mỗi API thuộc 1 trong 3 vai:

| Vai | Nghĩa | Ví dụ |
|---|---|---|
| **A = NUÔI DATA** | đưa dữ liệu vào hệ | `products:upsert`, `events:ingest`, `events:backfill` |
| **B = TRẢ KẾT QUẢ** | khách hỏi, hệ trả lời | `search`, `recommend`, `forecast:query`, `decisions:run` |
| **C = VÒNG PHẢN HỒI** | kết quả tốt/xấu quay về nuôi hệ | event `product.clicked`/`purchase.completed`, `decisions:feedback` |

**Câu chốt cho đối tác:** Không có A thì B trả rỗng. Không có C thì hệ không tự khôn lên.
Tích hợp đủ A+B+C mới hưởng trọn giá trị.

---

## 2. SERVICE 01 — smartsearch (BT01 Tìm kiếm + Gợi ý): 17 endpoint

**Bài toán:** khách gõ tiếng Việt (có dấu / không dấu / lỗi gõ nhẹ) phải ra đúng sản phẩm;
gợi ý theo ngữ cảnh; trả lời câu hỏi tự nhiên — KHÔNG bịa.

| Nhóm | Endpoint | Input → Output |
|---|---|---|
| Nuôi data (A) | `POST /v1/products:upsert` · `DELETE /v1/products/{id}` | catalog (id, title, price, category, attrs) → index vào Vespa |
| Nuôi hành vi (A/C) | `POST /v1/events:ingest` | `product.clicked`, `purchase.completed` → nuôi query_log, user_profile, LTR dataset |
| Trả kết quả (B) | `POST /v1/search` | query + filters → danh sách xếp hạng (hybrid BM25 + vector + RRF, học từ click thật) |
| | `GET /v1/suggest` | prefix đang gõ → autocomplete (học từ query người dùng thật trộn title n-gram) |
| | `POST /v1/recommend` | context (home / pdp / similar / cart) + user/product id → gợi ý theo ngữ cảnh |
| | `POST /v1/ask` | câu hỏi tự nhiên ("có tai nghe nào dưới 500k?") → trả lời grounded vào catalog, guard chống bịa |
| Merchandising | `GET/POST/DELETE /v1/merch:rules` | rule ghim / chặn / boost → can thiệp thứ hạng bằng tay |
| Nội bộ | `GET /internal/similar-products` · `/internal/products-by-category` | decision/forecast gọi chéo (cold-start analog) — không public |
| Vận hành | `GET /v1/events:dead` | → hàng đợi event chết (DLQ) để chẩn đoán ingest |

---

## 3. SERVICE 02 — decision (BT02 Quyết định nhập–giá): 17 endpoint

**Bài toán:** biến data cost / price / stock / sales thành **lời khuyên hành động được** — 7 kind:
bán-dưới-vốn · vốn-tăng · đề-xuất-giá · nhập-hàng · cảnh-báo-hết-hàng · hàng-ế · promo-hợp-pháp.
Mỗi quyết định kèm **giải thích tiếng Việt + EV (giá trị kỳ vọng)**.

| Nhóm | Endpoint | Input → Output |
|---|---|---|
| Nuôi data (A) | `POST /v1/events:ingest` | biến động giá vốn, tồn kho, đơn hàng → price_history, inventory |
| Cấu hình | `GET/PUT /v1/config` | chính sách per-tenant: margin sàn, **pricing_mode** (lerner mặc định / robust CVaR opt-in), khẩu vị rủi ro |
| | `GET/PUT /v1/config:supplier` | lead-time, MOQ, giá theo nhà cung cấp |
| Trả kết quả (B) | `POST /v1/decisions:run` | (trigger) → lô quyết định 7 kind + giải thích + EV |
| | `GET /v1/decisions` · `:stats` · `:insights` | filter → danh sách quyết định mở / thống kê / insight tổng hợp |
| | `POST /v1/decisions:price-preview` | giá định thử → preview tác động (demand, doanh thu, margin) TRƯỚC khi áp |
| | `GET /v1/decisions:replenish-plan` | SKU/kho → kế hoạch nhập (điểm đặt, số lượng — ăn lead-time-demand từ forecast) |
| Vòng phản hồi (C) | `POST /v1/decisions/{id}:feedback` | chấp nhận / từ chối / kết quả thật → outcome-loop 30 ngày (đo lời khuyên đúng thật không) |
| Vận hành | `GET /v1/events:dead` | → DLQ event chết |

---

## 4. SERVICE 03 — forecast (BT03 Dự báo): 18 endpoint

**Bài toán:** dự báo nhu cầu **theo xác suất** per-SKU — không trả 1 con số, trả **dải P10/P50/P90**
("bán chắc ít nhất 10, kỳ vọng 25, hiếm khi quá 60") + `model_used` trung thực + data_window + calibration.
Engine = model-router tự chọn (naive / snaive / ETS / LightGBM) theo độ dày data từng SKU.

| Nhóm | Endpoint | Input → Output |
|---|---|---|
| Nuôi data (A) | `POST /v1/events:ingest` · `:backfill` | đơn hàng ngày / lịch sử bán cũ → chuỗi thời gian per-SKU |
| Chạy + hỏi (B) | `POST /v1/forecast:run` | (trigger) → train + sinh projections |
| | `POST /v1/forecast:query` | SKU + horizon → dải P10/P50/P90 theo ngày + totals + model_used + data_window |
| | `POST /v1/forecast:aggregate` | nhóm SKU/category → dự báo gộp |
| | `GET /v1/forecast:accuracy` | → MASE / coverage so thực tế (hệ tự chấm điểm mình) |
| | `POST /v1/forecast:insights` | → insight nhu cầu (tăng/giảm, mùa vụ) |
| | `POST /v1/forecast:promo-preview` | "nếu giảm giá X%" → bán thêm bao nhiêu (dùng cache model, 0.03s) |
| Scenario fabric (nội bộ, ADR-009) | `POST /v1/scenarios:build` · `:probability` · `:aggregate` | decision gọi nội bộ tier-0 — dựng kịch bản xác suất |
| | `POST /v1/scenarios:lead-time-demand` | SKU + lead-time → phân phối nhu cầu trong thời gian chờ hàng → **đầu vào trực tiếp replenish-plan** |
| Vận hành | `GET /v1/events:dead` · `GET /v1/projections/status` | DLQ · trạng thái projections (op này đang tracked W-SURFACE-FC-PROJSTATUS) |

---

## 5. QUY TRÌNH SỬ DỤNG (CHUỖI SỐNG — cái mà check-apis 42/42 + 6 chain đo)

```
NGÀY 1 — NUÔI:  products:upsert (catalog) ──► events:backfill (lịch sử bán)
                        │
HÀNG NGÀY:      events:ingest (đơn hàng, click, giá vốn, tồn kho)
                — MỘT cửa ingest, 3 service cùng nhận
                        │
HỎI:            search / suggest / recommend / ask   (khách mua hàng)
                forecast:run ──► forecast:query      (dải nhu cầu P10/P50/P90)
                decisions:run ──► GET /v1/decisions  (lời khuyên — đã ăn forecast
                                                      qua scenario fabric nội bộ)
                        │
PHẢN HỒI:       click / purchase events ──► LTR dataset, user_profile (search khôn lên)
                decisions:feedback ──► outcome-loop 30 ngày (đo lời khuyên đúng/sai THẬT)
```

**3 service không rời rạc:**
- forecast nuôi decision: `scenarios:lead-time-demand` → `decisions:replenish-plan`
- search nuôi forecast + decision: event mua hàng đi chung 1 cửa ingest
- mọi chuỗi có phép đo máy: `make check-apis PROJECT=demoshop` = 42 probe API + 6 chain probe
  (rls-fuzz cách ly tenant · user_profile · elasticity · …) — đang **42/42 PASS, 0 SKIP**

---

## 6. CÂU PHỎNG VẤN TỰ LUYỆN (senior AI engineer — trả lời phải dẫn SỐ THẬT)

1. Vì sao forecast trả dải P10/P50/P90 thay vì 1 con số? Chủ shop dùng P10 và P90 vào việc gì khác nhau?
2. Nếu đối tác chỉ tích hợp `products:upsert` + `search` mà bỏ `events:ingest`, 3 tháng sau họ thua gì? (vai C)
3. `decisions:price-preview` vs `forecast:promo-preview` khác nhau chỗ nào — cái nào trả lời "nên đặt giá
   bao nhiêu", cái nào trả lời "giảm giá thì bán thêm bao nhiêu"?
4. Khi bị hỏi "hệ thống có bịa không?", dẫn cơ chế nào của `/v1/ask` và số đo nào?
   (gợi ý: grounding guard + fuzz 104 ca; 1 ca compound-id tracked W-ASK-GUARD-COMPOUND-ID)
5. Vì sao mọi quyết định của BT02 phải kèm EV + giải thích tiếng Việt? (gợi ý: trust + outcome-loop đo được)
