# BÀI 2 — CẤU HÌNH & XÁC THỰC PER-TENANT: KEY → HEADER → CHÍNH SÁCH → VÒNG ĐỜI KEY

> Giáo trình training human (xuất 2026-08-06). Bài 0 = `TRAINING-API-MAP-2026-08-06.md` · Bài 1 = `TRAINING-BAI-1-VAN-HANH-2026-08-06.md`.
> Điều kiện vào bài: stack đang xanh (Bài 1 B3c = 42/42).
> Mọi lệnh chạy từ: `cd /home/hai-soft/projects/icpp/mecom/project`

**Mục tiêu:** hiểu tenant/key/header; đọc-đổi chính sách kinh doanh qua API; nắm trọn vòng đời key (cấp → dùng → lộ → xoay → thu hồi) và cơ chế hash một chiều đứng sau nó.

---

## KIẾN THỨC NỀN (đọc 3 phút)

- **Tenant (= project = shop):** một khách thuê hệ — `demoshop`, `seedtest`, `hocdemo`… Dữ liệu cách ly tuyệt đối theo tenant (RLS tầng DB — bằng chứng chạy hàng ngày: probe `rls-fuzz 28 cases, 0 leak` trong check-apis).
- **Mỗi shop 3 KEY** (không phải 1): mỗi service một key riêng (search/decision/forecast) — lộ key search không mở được cửa decision.
- **Mọi request nghiệp vụ cần đúng 2 header:**
  ```
  Authorization: Bearer <key-của-đúng-service>
  X-Project-Id: <tenant>
  ```
- **Key demo/dev** gói sẵn tại `data/seed_keys_<tenant>.json` (seedtool tạo). Production không có file này.
- **Phân biệt lệnh ĐỌC vs GHI bằng mắt:** có `-X PUT/POST/DELETE` hoặc `-d '{...}'` = có thể thay đổi hệ · chỉ `curl` + `-H` = GET, an toàn tuyệt đối.

---

## BÀI NHỎ 2.1 — XEM CHÙM CHÌA KHÓA (không in key trần)

```bash
python3 -c "import json; d=json.load(open('data/seed_keys_demoshop.json')); print({k: v[:8]+'...' for k,v in d.items()})"
```
**Kỳ vọng:** `{'search': 'jCwHxCJl...', 'decision': 'hB6_WaDE...', 'forecast': 'gUMBQrI9...'}` — đúng 3 chìa.
**Ghi chú:** key không bao giờ `cat` trần ra terminal/log (scrollback, screen-share). Xem hình dạng đủ hiểu; dùng thì nạp biến.

## BÀI NHỎ 2.2 — REQUEST CÓ XÁC THỰC ĐẦU TIÊN: ĐỌC CHÍNH SÁCH

```bash
KEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['decision'])") \
&& curl -s localhost:16022/v1/config -H "Authorization: Bearer $KEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
```
**Kỳ vọng:** `{"config": {"promo_cap_pct": 50, "pricing_mode": "lerner"}}`
**Đọc nghĩa kinh doanh:**
- `pricing_mode: lerner` = định giá tối ưu kỳ vọng theo elasticity (mặc định) · `robust` = CVaR thận trọng, tối ưu kịch bản xấu (tenant ghét rủi ro/data mỏng).
- `promo_cap_pct: 50` = trần khuyến mãi (chốt pháp lý VN — liên quan kind `promo_legal`).
**Ghi chú:** key nạp vào biến `KEY=$(...)`, không echo. Port theo service (16022=decision) — key cũng phải của decision.

## BÀI NHỎ 2.3 — ĐỔI CHÍNH SÁCH SỐNG (PUT rồi GET verify)

```bash
KEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['decision'])") \
&& curl -s -X PUT localhost:16022/v1/config -H "Authorization: Bearer $KEY" -H "X-Project-Id: demoshop" \
   -H "Content-Type: application/json" -d '{"pricing_mode":"robust"}' | python3 -m json.tool \
&& echo "--- doc lai xac nhan:" \
&& curl -s localhost:16022/v1/config -H "Authorization: Bearer $KEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
```
**Kỳ vọng:** PUT trả `{"updated": ["pricing_mode"]}` (khai đúng cái đã đổi) · GET thấy `robust`, còn `promo_cap_pct` giữ nguyên 50 (partial update chuẩn).
**Ghi chú:** đổi "tính cách" engine per-tenant bằng 1 request, không restart. Phản xạ: PUT xong LUÔN GET đọc lại — không tin lệnh ghi, tin lệnh đọc.

## BÀI NHỎ 2.4 — TEST PHÁ: DÙNG NHẦM CHÌA

```bash
WRONGKEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['search'])") \
&& curl -s -o /dev/null -w "sai-service: %{http_code}\n" localhost:16022/v1/config -H "Authorization: Bearer $WRONGKEY" -H "X-Project-Id: demoshop" \
&& curl -s -o /dev/null -w "khong-header: %{http_code}\n" localhost:16022/v1/config
```
**Kỳ vọng:** cả hai `401`. Bất kỳ `200` nào = lỗ hổng nghiêm trọng, báo ngay.
**Ghi chú:** kiểm cả đường CẤM, không chỉ đường CHO. 401 im lặng không tiết lộ "key này tồn tại ở service khác" — im lặng là đức tính bảo mật. `-o /dev/null -w "%{http_code}"` = chỉ đọc mã, vứt body.

## BÀI NHỎ 2.5 — TRẢ NGUYÊN TRẠNG

```bash
KEY=$(python3 -c "import json;print(json.load(open('data/seed_keys_demoshop.json'))['decision'])") \
&& curl -s -X PUT localhost:16022/v1/config -H "Authorization: Bearer $KEY" -H "X-Project-Id: demoshop" \
   -H "Content-Type: application/json" -d '{"pricing_mode":"lerner"}' | python3 -m json.tool \
&& curl -s localhost:16022/v1/config -H "Authorization: Bearer $KEY" -H "X-Project-Id: demoshop" | python3 -m json.tool
```
**Kỳ vọng:** về `lerner` + xác nhận bằng GET.
**Ghi chú:** thí nghiệm xong trả hệ đúng trạng thái tìm thấy (demoshop là sân demo). Xác nhận bằng đọc lại, không tin trí nhớ.

---

## BÀI NHỎ 2.6 — VÒNG ĐỜI KEY: CẤP KEY CHO SHOP MỚI

### 2.6a. Cấp (mint) — shop khai sinh bằng chiếc key đầu tiên
```bash
.venv/bin/python scripts/keys.py new --project hocdemo \
  --dsn postgresql://miniai:miniai@localhost:16024/miniai_decision
```
**Kỳ vọng:** 3 dòng — `api_key=...` (IN ĐÚNG 1 LẦN, không xem lại được) + `key_id=k_...` + WARNING.
**Cơ chế bên trong (`scripts/keys.py`):** sinh `secrets.token_urlsafe(32)` → lưu **hash(key+pepper)** vào bảng `api_keys` của đúng DB service, state `active` → in key thô 1 lần rồi server QUÊN LUÔN. Pepper = bí mật server (`MINIAI_KEY_PEPPER`), nằm NGOÀI DB.

### 2.6b. Tự kiểm chứng "DB chỉ giữ hash"
```bash
docker exec miniai-postgres psql -U miniai -d miniai_decision \
  -c "SELECT project_id, key_id, left(key_hash,20) || '...' AS hash, state FROM api_keys WHERE project_id='hocdemo';"
```
**Kỳ vọng:** cột hash là chuỗi KHÁC HOÀN TOÀN key thô — server không biết key của khách, chỉ biết cách nhận ra nó.

### 2.6c. Hiểu hash một chiều — "máy xay sinh tố" (chạy để cảm 3 tính chất)
```bash
echo -n "peMb3n-vi-du" | sha256sum && echo -n "peMb3n-vi-du" | sha256sum && echo -n "peMb3n-vi-dU" | sha256sum
```
1. Cùng đầu vào → LUÔN cùng đầu ra (dòng 1 = dòng 2) — nền tảng của phép so.
2. Một chiều tuyệt đối — từ hash không lắp ngược ra key.
3. Đổi 1 ký tự (u→U) → hash khác hoàn toàn.

**Luồng xác thực mỗi request:** khách gửi key thô trong header → server XAY LẠI tại chỗ hash(key+pepper) → so với hash trong DB → khớp = cho vào. Server không "nhớ" key — chỉ nhớ "ly sinh tố mẫu". DB bị trộm → hacker chỉ có hash: không đảo ngược được, cầm hash đi gọi cũng bị xay-của-xay → 401.

**Key thô lưu ở đâu?** KHÔNG ĐÂU CẢ — trách nhiệm cất giữ thuộc người nhận (password manager/vault). Mất = cấp lại. File `seed_keys_*.json` chỉ là bao thư tiện lợi môi trường dev do seedtool ghi.

---

## BÀI NHỎ 2.7 — CẤP KEY MỚI CHO SHOP CŨ (rotation / xử lý key lộ)

**3 tình huống:** khách mất key (thường) · key LỘ (gấp — ví dụ key dán vào chat/scrollback) · xoay vòng định kỳ (chuẩn an ninh).

**Nguyên tắc vàng:** một shop được phép NHIỀU key active cùng lúc ⇒ rotation không-gián-đoạn:
```
cấp key MỚI (cũ vẫn sống) → khách đổi xong → revoke key CŨ theo key_id
```
⚠ Làm ngược (revoke trước, cấp sau) = shop chết API trong lúc chờ — lỗi vận hành kinh điển. Chỉ revoke-trước khi key đang bị kẻ xấu dùng.

### Nước 1 — Cấp key thứ 2 + nhìn DB thấy 2 dòng cùng active
```bash
.venv/bin/python scripts/keys.py new --project hocdemo \
  --dsn postgresql://miniai:miniai@localhost:16024/miniai_decision
docker exec miniai-postgres psql -U miniai -d miniai_decision \
  -c "SELECT key_id, left(key_hash,16) || '...' AS hash, state FROM api_keys WHERE project_id='hocdemo' ORDER BY key_id;"
```
**Kỳ vọng:** bộ `api_key`/`key_id` mới + bảng 2 dòng đều `active`.

### Nước 2 — Chứng minh cả 2 chìa cùng mở cửa (test xác thực thuần ĐỌC)
```bash
NEWKEY='<key-mới>' && OLDKEY='<key-cũ>' \
&& curl -s -w "\nkey moi: %{http_code}\n" localhost:16022/v1/config -H "Authorization: Bearer $NEWKEY" -H "X-Project-Id: hocdemo" \
&& curl -s -w "\nkey cu:  %{http_code}\n" localhost:16022/v1/config -H "Authorization: Bearer $OLDKEY" -H "X-Project-Id: hocdemo"
```
**Kỳ vọng:** cả hai `200` (giai đoạn chuyển tiếp). Body là `{"config":{}}` — xem giải thích dưới.

### Nước 3 — Thu hồi key cũ theo key_id (KHÔNG cần key thô) + xác nhận chết/sống
```bash
.venv/bin/python scripts/keys.py revoke --project hocdemo --key-id <key_id-cũ> \
  --dsn postgresql://miniai:miniai@localhost:16024/miniai_decision
curl -s -o /dev/null -w "key cu sau revoke: %{http_code}\n" localhost:16022/v1/config -H "Authorization: Bearer $OLDKEY" -H "X-Project-Id: hocdemo"
curl -s -o /dev/null -w "key moi van song:  %{http_code}\n" localhost:16022/v1/config -H "Authorization: Bearer $NEWKEY" -H "X-Project-Id: hocdemo"
```
**Kỳ vọng:** key cũ `401` TỨC KHẮC · key mới vẫn `200` — shop không gián đoạn giây nào.
(Hệ có fix `auth reload-on-miss`: key vừa cấp sống ngay, key vừa revoke chết ngay — không chờ cache 60s.)

---

## KHÁI NIỆM RÚT RA TỪ THỰC HÀNH

### `{"config":{}}` = "chưa có GHI ĐÈ", không phải "chưa có chính sách"
`GET /v1/config` chỉ trả các dòng đã PUT tường minh (bảng `project_config`). Cơ chế 2 tầng:
- **Mặc định trong CODE** (vd `lerner`) — áp ngầm lúc ra quyết định; tenant mới zero-config chạy đủ ngay.
- **Ghi đè per-tenant trong DB** — chỉ hiện khi đã PUT; tenant đã chọn thì "đóng đinh" theo lựa chọn.
Ngành gọi: convention over configuration. (demoshop "trông có config" vì probe check-apis từng PUT.)

### Đăng ký shop trọn vẹn = checklist 5 mục
1. Cấp **đủ 3 key** — 3 lần `keys.py new`, DSN lần lượt: `miniai_search` · `miniai_decision` · `miniai_forecast` (cùng postgres :16024).
2. Trao key qua kênh an toàn + dặn "in 1 lần, tự cất, mất thì xoay".
3. Khách đổ catalog (`products:upsert`) + nối ống event (`events:ingest`) — vai A/C, thiếu là hệ không khôn lên.
4. (Tùy chọn) `PUT /v1/config` nếu khách có khẩu vị riêng.
5. Nghiệm thu chuỗi bằng key của chính shop.
Không có "nghi lễ create shop" riêng — shop khai sinh bằng key đầu tiên; dữ liệu tự "mọc" nhờ cột project_id + RLS.

---

## SỔ TAY — GHI CHÚ CHỐT BÀI 2
1. 1 shop = 3 key theo service; 2 header bắt buộc mọi request.
2. Key thô in đúng 1 lần, không lưu đâu cả; DB chỉ giữ hash(key+pepper); pepper ngoài DB.
3. PUT xong luôn GET verify; partial update không phá trường khác.
4. Kiểm cả đường CẤM (sai chìa 401, tay không 401).
5. Rotation: cấp trước — thu sau; revoke theo key_id; nhiều key active cùng lúc là tính năng.
6. Config rỗng = đang chạy mặc định code; chỉ ghi đè mới hiện.
7. Thí nghiệm xong trả nguyên trạng + dọn key học tập (revoke).

## BÀI TẬP TỰ LUYỆN (làm không cần hướng dẫn)
1. Cấp nốt 2 key còn lại cho `hocdemo` (search + forecast) — hoàn tất checklist mục 1.
2. Đổi `promo_cap_pct` của hocdemo thành 30, verify, rồi xóa ghi đè về mặc định (gợi ý: đọc API xem có cách xóa key config không — nếu không có, đó là 1 GAP đáng ghi nhận).
3. Diễn lại trọn 2.7 với shop `hocdemo2` từ số 0, không mở giáo trình.
4. Revoke MỌI key học tập (`hocdemo*`) khi luyện xong — dọn sân.

## CÂU PHỎNG VẤN TỰ LUYỆN (dẫn số thật)
1. "Chứng minh tenant A không đọc được dữ liệu tenant B?" (2 tầng: 401 key/header + RLS tầng DB; số: rls-fuzz 28 ca 0 leak + fuzz 96 ca read+write chạy trong mọi check-apis)
2. "Server không lưu key thì xác thực kiểu gì?" (hash một chiều + pepper; xay-lại-tại-chỗ rồi so; DB bị trộm vẫn an toàn)
3. "Quy trình xử lý key lộ của bạn?" (cấp trước-thu sau theo key_id; zero-downtime; kể ca thật: key dán vào chat → xoay ngay trong buổi)
4. "lerner vs robust — khuyên khách nào dùng cái nào?" (kỳ vọng vs CVaR kịch bản xấu; độ dày data; khẩu vị rủi ro; đổi bằng 1 PUT per-tenant)
