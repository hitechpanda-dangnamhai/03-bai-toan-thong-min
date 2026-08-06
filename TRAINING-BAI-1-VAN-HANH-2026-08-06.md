# BÀI 1 — VẬN HÀNH STACK: DỰNG → BẮT MẠCH → NGHIỆM THU TẤT ĐỊNH

> Giáo trình training human (xuất 2026-08-06, đã thực hành đạt cùng ngày).
> Nguồn sự thật = DB tri thức: `RB-DEMO-PREFLIGHT` (phòng) + `RB-HOST-OVERLOAD-COLDSTART` (chữa) — `python3 rail.py q "preflight"`.
> Bài 0 (bản đồ 52 API) = `TRAINING-API-MAP-2026-08-06.md`.

**Mục tiêu:** tự tay dựng hệ miniAI từ số 0, chứng minh nó sống bằng SỐ (42/42), và chẩn đoán được khi số đỏ.

---

## PHẦN A — KIẾN THỨC NỀN (đọc 3 phút)

### A1. Container vs Volume — "vỏ" và "ruột"
- **Container** = vỏ chạy code, đập đi xây lại thoải mái.
- **Volume** = ruột chứa dữ liệu (Postgres data, Vespa index) — sống NGOÀI container.
- Hệ quả: `down` rồi `up` không mất dữ liệu.

### A2. Ba cấp độ "xóa" — thuộc lòng
| Lệnh | Xóa gì | Dữ liệu | Dùng khi |
|---|---|---|---|
| `docker compose stop` | không xóa, chỉ dừng | ✅ nguyên | tạm dừng, mai `start` lại |
| `docker compose down` | container + network | ✅ nguyên (volume còn) | làm lại sạch sẽ |
| `docker compose down -v` | cả volume | ❌ **MẤT SẠCH** | ⛔ gần như không bao giờ |

### A3. Hai mạch của mỗi service
- `/healthz` = "process còn thở" — Docker dùng để quyết restart.
- `/readyz` = "**đã nối được DB, sẵn sàng nhận request**" — load-balancer dùng để rót traffic.
- Vận hành luôn tin `readyz`. `Started` ≠ ready.

### A4. Load average — nhiệt kế máy
- `uptime` in load 1/5/15 phút = số process đang chạy + chờ CPU + kẹt chờ đĩa (D-state).
- Chỉ có nghĩa khi so với `nproc` (máy này: **16**). Load ≥ 16 = nghẹt; 110 = quá tải ~7 lần.
- So 3 số đọc xu hướng: `110, 50, 20` = đang bùng · `6, 54, 74` = đã xử, đang hạ nhiệt.
- Load là **trung bình trượt**: tắt nguồn đốt xong cần ~2 phút số mới phản ánh thật.

### A5. Ba port thuộc lòng
**16021 = smartsearch · 16022 = decision · 16023 = forecast** (đuôi 1/2/3 = BT01/02/03).
Hạ tầng: pg 16024 · vespa 16025 · grafana 16020 · prometheus 16029.

---

## PHẦN B — TIỀN XỬ LÝ P1–P6 (khi máy có nhiều project khác)

> Chạy TRƯỚC Bài 1 mỗi khi: máy vừa reboot · sắp demo/đo đạc · nghi máy chậm.

### P1. Điểm danh cả máy theo chủ sở hữu
```bash
docker ps -a --format 'table {{.Label "com.docker.compose.project"}}\t{{.Names}}\t{{.Status}}' | sort
```
- `-a` = gồm cả container đã tắt. Nhãn `com.docker.compose.project` = "container này của ai".
- Đọc 3 câu: máy cõng project nào? · ai `Restarting` (cờ đỏ crash-loop)? · ai `Up` mà không phục vụ hôm nay?

### P2. Đo nền
```bash
uptime && nproc
```
- Load 1-phút **< ~10**/16 core → sạch, bỏ qua P3–P5.
- Load ≥ nproc → máy nghẹt, mọi số đo lúc này là số VỨT (vòng-0). Đi P3.

### P3. Tìm kẻ ăn tài nguyên
```bash
# 3a. tầng container
docker stats --no-stream --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}' | sort -t$'\t' -k2 -hr | head -15
# 3b. tầng host (bắt cả kẻ ngoài docker)
ps --sort=-pcpu -eo pid,pcpu,pmem,args | head -12
# 3c. kiểm nghẽn đĩa
ps -eo stat,comm | awk '$1 ~ /^D/' | sort | uniq -c | sort -rn | head
```
- ⚠ Sort theo **CPU**, đừng lấy PID nhỏ (PID nhỏ = process già, chưa chắc thủ phạm — bài học soi nhầm 2026-08-06).
- Cột `args` = căn cước tự khai (chuỗi `debezium`/`/opt/vespa`/`elasticsearch` nằm trong dòng lệnh).
- STAT: `R` chạy · `S` ngủ · `D` kẹt chờ đĩa. 3c ra đàn D đông = nghẽn ĐĨA; rỗng = CPU thuần bị chen.

### P4. Truy căn cước kẻ lạ
```bash
docker inspect <container> --format \
'project: {{index .Config.Labels "com.docker.compose.project"}}
dir:     {{index .Config.Labels "com.docker.compose.project.working_dir"}}
files:   {{index .Config.Labels "com.docker.compose.project.config_files"}}'
```
- Ra đủ: tên project + thư mục + **danh sách file compose**. GHI LẠI cả 3 = lệnh dựng lại cho mai.
- ⚠ Bẫy đa-file: stack `sicp/infra` dùng 3 file compose — thao tác thiếu `-f` là sót service.

### P5. Tắt láng giềng — 2 nhát, có đường lui
```bash
cd <dir-từ-P4> && docker compose -f <file1> -f <file2> ... stop   # nhát 1: theo sổ sách
docker stop $(docker ps -q --filter "name=^icp-")                  # nhát 2: quét vét theo tên
```
- `stop` = tạm dừng hoàn nguyên 100%. Không `down`, tuyệt đối không `down -v` nhà người khác.
- `stop` thắng restart-policy (policy chỉ hồi sinh container *crash*); ngoại lệ: reboot máy → `restart: always` dựng lại tất → chạy lại P1.
- Kỷ luật: tắt nhà người khác phải có chủ nhà gật + ghi lại đã tắt gì.

### P6. Xác nhận sân sạch
```bash
docker ps --format '{{.Names}}\t{{.Status}}' | grep -v miniai
uptime    # chờ ~2 phút sau khi tắt đàn lớn rồi mới đọc
```
Ngưỡng qua cổng: load 1-phút một chữ số + không container lạ Up/Restarting.

---

## PHẦN C — BÀI 1 v2: QUY TRÌNH TẤT ĐỊNH (chạy giống nhau mọi lần)

> Nguyên tắc thiết kế: **chất keo giữa các bước là PHÉP ĐO CÓ NGƯỠNG (until-ready / until-idle), không phải thời gian + hy vọng.** Mọi trạng thái ẩn phải thành một bước đo tường minh.

### ĐIỀU KIỆN VÀO (không đạt thì KHÔNG bắt đầu)
```bash
uptime && nproc        # load1 < ~10/16; cao hơn → xử P1-P6 trước
```

### B0 — Dọn sân (nếu muốn làm lại từ đầu)
```bash
cd /home/hai-soft/projects/icpp/mecom/project && docker compose down
```
Đạt: 11 container + 1 network `Removed`. (Vài container mất ~10s = docker chờ app tắt êm SIGTERM — bình thường.)

### B1 — Dựng
```bash
docker compose up -d
```
Đạt: 11/11 ✔, **`miniai-postgres Healthy`** (không chỉ Started — compose chờ PG khỏe thật rồi mới thả API nhờ `depends_on: service_healthy`).
- Sau `down`: có pha `Created` trước `Started` (xác cũ đã xóa, phải tạo mới). Sau `stop`: chỉ `Started`.
- Idempotent: chạy lại bao nhiêu lần cũng an toàn.

### B2 — Bắt mạch có vòng chờ
```bash
for p in 16021 16022 16023; do until curl -sf localhost:$p/readyz >/dev/null; do sleep 3; done; echo "port $p READY"; done
```
Lặp đến khi ready — loại biến ẩn "gõ lệnh sớm hay muộn".

### B3a — Khởi động máy (KHÔNG chấm điểm)
```bash
make check-apis PROJECT=demoshop ; echo "== WARM-UP XONG (khong cham diem) =="
```
Sau `down/up`, lần gọi `:run` đầu bắt buộc train model từ 0 → có thể FAIL timeout — **là kỳ vọng, kệ nó**.

### B3b — Cổng chờ worker nghỉ (bước biến hên-xui thành tất định)
```bash
until [ "$(docker stats --no-stream --format '{{.CPUPerc}}' miniai-forecast-worker | cut -d. -f1)" -lt 50 ]; do echo "worker dang train... cho 15s"; sleep 15; done; echo "== WORKER NGHI — duoc phep do =="
```
Đo trực tiếp CPU worker thay vì `sleep` đoán mò — cổng chỉ mở khi train xong thật.

### B3c — PHÁN QUYẾT
```bash
make check-apis PROJECT=demoshop
```
Kỳ vọng: **API CHECK 42/42 PASS — mọi lần.** FAIL tại đây = sự cố thật 100%, xứng đáng chẩn đoán.

**Rút gọn hợp lệ:** hệ đang chạy sẵn + lần đo gần nhất xanh (model ấm) → chạy thẳng B3c. Cứ dựng lại stack là đủ 3a→3b→3c.

**Vì sao cần 3a/3b (nợ thiết kế có tên):** `forecast:run` làm việc nặng ĐỒNG BỘ trong request → kết quả probe = cuộc đua (thời gian việc vs trần chờ), phụ thuộc 3 biến ẩn: độ ấm model · CPU rảnh · hàng đợi worker. Đã ghi nợ `W-RUN-ASYNC-202` (chuyển sang 202+job_id, đo "nhận việc" thay vì "làm xong việc") — land xong thì bài này rút còn 4 bước.

> 🆕 **CẬP NHẬT TỐI 2026-08-06 — W-RUN-ASYNC-202 ĐÃ LAND:** `:run` giờ trả `202+job_id` ngay, probe đo
> "nhận việc" tất định; cái "cuộc đua" ở trên chuyển hết về phía worker và theo dõi được bằng
> `GET /v1/projections/status?job_id=...` (`queued→running→done/failed`). B3a/B3b vẫn giữ khi dựng lại
> stack (train nguội vẫn tốn phút — chỉ là không còn rơi vào mặt client). Thêm 2 vũ khí mới:
> 1. **Preflight máy:** `python -m seedtool check` tự TỪ CHỐI đo khi máy nghẹt — exit `3` in
>    `MOI-TRUONG-BAN: load1=... nproc=...` (khác 0=PASS, 1=FAIL nghiệp vụ). Hết cảnh 39/42 oan như sáng nay.
> 2. **GOTCHA backtest-on-boot (đo thật 17h):** restart container forecast ⇒ `start_backtest_loop` chạy
>    FULL backtest mọi project ngay khi boot (CPU 90%+ có thể vài chục phút) — **ĐỪNG restart forecast ngay
>    trước demo**; lỡ restart thì chờ bằng B3b (đo CPU <20-50%), không chờ bằng đồng hồ.

---

## PHẦN D — CHẨN ĐOÁN: 3 LOẠI TIMEOUT (case study thật 2026-08-06)

Cùng triệu chứng `timed out`, ba bệnh khác nhau — chỉ phép đo phân biệt được:

| Loại | Nhận diện (`docker stats` + chạy lại) | Xử |
|---|---|---|
| **Người ngoài chen** | kẻ ăn CPU là stack KHÁC; chạy lại VẪN fail | dọn sân P1–P6 |
| **Nhà mình đang cày** | kẻ ăn CPU là worker miniai (vd forecast-worker ~1500% khi train); chạy lại HẾT fail | chờ worker rảnh (B3b) rồi đo lại |
| **Hỏng thật** | fail dạng 4xx/5xx, hoặc máy yên + hệ ấm vẫn fail | đọc log: `docker logs <svc> --since 15m 2>&1 \| grep -iE "error\|traceback\|timeout" \| tail -30` |

Case study sáng 2026-08-06 (đủ chuỗi nhân quả): máy reboot → stack miniai tắt + 16 container CDC debezium (sicp) crash-loop nuốt ~11/16 core, load 110 → Postgres miniai bị chen → `ledger unavailable TimeoutError` + `decisions:run`/`forecast:run` timeout → check-apis 39/42 → tắt icp (human gật) → load 6.58 → **42/42**. Thí nghiệm đối chứng cùng ngày: cold-start hoàn toàn trên máy yên → 42/42 ngay lần đầu ⇒ biến số quyết định là TẢI MÁY, không phải "lạnh".

Manh mối vàng khi đọc:
- Cả loạt FAIL đều `timed out`, 0 cái 4xx/5xx → nghi tài nguyên, không nghi logic.
- `ledger unavailable (TimeoutError)` trong log decision = **Postgres chậm** (ledger chạy trên PG, không phải Redis — `libs/common/quota_store.py`).
- Load cao + CPU container thấp + D-state rỗng = CPU thuần bị chen bởi kẻ ngoài docker hoặc đàn đông con.

---

## SỔ TAY — GHI CHÚ CHỐT BÀI 1
1. Volume giữ ruột — `down` không mất data; `-v` mới mất. 3 cấp xóa: stop < down < down -v ⛔.
2. `Started` ≠ ready ≠ nghiệp vụ đúng: `up` → `readyz` → `check-apis` là 3 tầng chứng minh khác nhau.
3. Quy trình tất định = mọi chỗ chờ đều là vòng `until` đo ngưỡng, không `sleep` đoán mò.
4. Phán quyết ở B3c; warm-up B3a không chấm điểm.
5. Timeout: hỏi ngay "AI đang ăn CPU — người ngoài / nhà mình / không ai?" → 3 đường xử khác nhau.
6. Tắt nhà người khác: đo trước, chủ nhà gật, chỉ `stop`, đủ `-f`, ghi lại để dựng lại.
7. Lệnh thuộc lòng số 1: `make check-apis PROJECT=demoshop` — trước mọi demo, sau mọi restart. Không tin cảm giác, tin 42/42.

## CÂU PHỎNG VẤN TỰ LUYỆN (dẫn số thật)
1. Phân biệt healthz/readyz — hệ của bạn ai tiêu thụ từng mạch?
2. Load average 110 trên máy 16 core nhưng tổng CPU container chỉ ~190% — giải thích? (đáp: chen CPU bởi process ngoài docker/đàn đông con; kiểm D-state loại trừ nghẽn đĩa)
3. Vì sao quy trình nghiệm thu của bạn "chạy giống nhau mọi lần"? (đáp: gate bằng phép đo có ngưỡng; warm-up tách khỏi phán quyết; kể nợ W-RUN-ASYNC-202)
4. Kể một sự cố thật bạn tự xử: (kể case study phần D — có số từng bước).
