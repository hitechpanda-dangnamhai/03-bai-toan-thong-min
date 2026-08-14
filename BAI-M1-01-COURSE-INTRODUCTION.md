# BÀI M1-01 — GIỚI THIỆU KHOÁ MACHINE LEARNING WITH PYTHON

> **Gia sư:** SLEEP · **Học viên:** 1 người · **Ngày soạn:** 2026-08-14
> **Bài giảng gốc:** `lectures/Machine Learning with Python/m1/m1 - 01 - Course Introduction.txt`
> (file bạn đưa từ `~/Downloads/` — đã đo `diff`: giống hệt)
>
> **Cách tôi soạn bài này:** đọc bài giảng → gạch ra **mọi khái niệm nó nhắc tới** → giải thích từng cái
> cho dễ hiểu. Định nghĩa lấy từ sổ tri thức đã kiểm chứng của bạn, không lấy từ trí nhớ.

---

## CÁCH ĐỌC FILE NÀY

| Dấu | Nghĩa | Bạn phải làm gì |
|---|---|---|
| 🧠 **CẦN NHỚ** | Sự thật, định nghĩa | Đọc, nhắc lại được là đủ |
| 🛠 **CẦN HỌC** | Kỹ năng | Phải **làm được** — nhắc lại định nghĩa không tính |
| 💡 **VÍ DỤ ĐỜI THỰC** | Chuyện đời để dễ nhớ | Đọc cho ngấm |
| ⚠ **BẪY** | Chỗ dễ hiểu sai | Đọc kỹ, đây là chỗ mất điểm |

**Lưu ý quan trọng:** bài giảng gốc chỉ là bài **giới thiệu** — nó **nêu tên** rất nhiều khái niệm nhưng
**không giải thích** cái nào. Việc của file này: mỗi cái tên nó thả ra, tôi giải thích **nó là gì, dùng làm
gì, ví dụ đời thực, bẫy ở đâu**.

Cuối file có mục **HỎI & ĐÁP** — bạn thắc mắc gì, tôi ghi câu trả lời vào đó, để lần sau đọc lại file là đủ.

---

## BẢN ĐỒ — BÀI GIẢNG CÓ 5 ĐOẠN, THẢ RA 20 KHÁI NIỆM

| Đoạn | Nói gì | Khái niệm cần giải thích |
|---|---|---|
| 1 | Vì sao học ML bằng Python | applied ML · data science · analytics · Scikit-Learn |
| 2 | Dành cho ai, cần biết sẵn gì | Pandas · NumPy · data preparation · AI |
| 3 | **Bạn sẽ học gì** ← quan trọng nhất | ML lifecycle · mô hình học kiểu gì · classification · regression · clustering · supervised · unsupervised · reinforcement learning · deep learning |
| 4 | Bạn sẽ tự tay làm gì | building · assessing · validating · mã nguồn mở |
| 5 | Học và chấm thế nào | multiple linear regression · logistic regression · prediction · fraud detection · KNN · SVM |

---

# ĐOẠN 1 — VÌ SAO HỌC MACHINE LEARNING BẰNG PYTHON

> *"Hello and welcome to this course on Machine Learning with Python, where you will learn about **applied
> machine learning** using Python tools. **Python dominates the machine learning landscape today.** According
> to recent surveys, it is the most widely used language for **machine learning, data science, and analytics**.
> Libraries, such as **Scikit-Learn**, are especially popular with practitioners in this field."*

## 1.1 Machine Learning là gì — và chữ "applied" nghĩa là gì

**Định nghĩa:**
> **Machine Learning (Học máy)** = tập con của AI, dùng thuật toán để máy **HỌC TỪ DỮ LIỆU**, nhận ra mẫu
> và ra quyết định **mà không cần con người viết luật tường minh**.

Điểm mấu chốt nằm ở vế cuối. So sánh hai cách viết phần mềm:

```
CÁCH THƯỜNG (lập trình cũ)          CÁCH MACHINE LEARNING
──────────────────────────          ─────────────────────────────────
NGƯỜI viết luật:                    NGƯỜI đưa ví dụ:
                                    
  if thu_nhap < 5_000_000:            10.000 hồ sơ vay cũ
      return "từ chối"                + kết quả thật của từng hồ sơ
  if no / thu_nhap > 0.5:               (trả được / vỡ nợ)
      return "từ chối"                          ↓
  return "duyệt"                       MÁY tự tìm ra luật
        ↓                                       ↓
  Máy chỉ làm theo luật               Máy áp luật đó cho hồ sơ mới
```

💡 **VÍ DỤ ĐỜI THỰC:** dạy trẻ con phân biệt chó với mèo. Bạn **không** đọc cho nó nghe *"mèo có tai nhọn,
ria dài 5cm, đồng tử dọc..."*. Bạn chỉ vào 100 con vật và nói "chó", "mèo", "chó"... Đến con thứ 101 nó
tự nhận ra. **Đó chính là machine learning.**

Còn chữ **"applied"** (ứng dụng) là lời hứa về **cách dạy**: khoá này thiên về *dùng công cụ có sẵn để giải
bài toán*, chứ không thiên về *chứng minh công thức toán đằng sau*.

🧠 **CẦN NHỚ:** ML = **học luật từ ví dụ**, thay vì được người viết luật cho.

## 1.2 Ba nghề bị nhắc chung một câu — chúng KHÁC nhau

Bài giảng viết *"machine learning, data science, and analytics"* liền một mạch, dễ tưởng là một thứ.
Thực ra là ba nghề, phân biệt bằng **câu hỏi mà mỗi nghề trả lời**:

| Nghề | Trả lời câu gì | Nhìn về đâu | Ví dụ câu trả lời |
|---|---|---|---|
| **Analytics** (phân tích dữ liệu) | *"Chuyện gì ĐÃ xảy ra?"* | quá khứ | "Quý trước doanh thu giảm 12%, giảm mạnh nhất ở miền Bắc" |
| **Data Science** (khoa học dữ liệu) | *"VÌ SAO nó xảy ra?"* | quá khứ → nguyên nhân | "Giảm vì đối thủ mở 40 cửa hàng ở miền Bắc" |
| **Machine Learning** | *"Ca NÀY thì sao? Sắp tới thế nào?"* | tương lai, từng ca cụ thể | "Khách này 87% khả năng sẽ bỏ đi trong 3 tháng tới" |

🧠 **CẦN NHỚ:** analytics nhìn **quá khứ, nhìn tổng thể** · ML nhìn **tương lai, nhìn từng ca**.

## 1.3 Scikit-Learn là gì

**Định nghĩa:**
> **scikit-learn** (đọc: *xai-kít-lơn*) = thư viện Python **miễn phí** cho **ML cổ điển**: phân loại, hồi quy,
> gom cụm, giảm chiều. Xây trên NumPy/SciPy/Matplotlib. Có sẵn: tiền xử lý, chia train/test, `fit`/`predict`,
> kiểm chứng chéo, lưu mô hình.

💡 **VÍ DỤ ĐỜI THỰC:** scikit-learn là **bộ dao đã mài sẵn**. Bạn không phải rèn dao — bạn học **chọn dao nào
cho món nào** và **cầm cho đúng**. Gần như cả khoá học này là học đúng hai việc đó.

Cái hay nhất của nó: **mọi thuật toán đều dùng chung một nghi thức 3 nhịp.**

```python
model = <TênThuậtToán>()   # nhịp 1: chọn dao
model.fit(X, y)            # nhịp 2: mài theo dữ liệu của bạn   ← HỌC
model.predict(X_moi)       # nhịp 3: thái                        ← DÙNG
```

Trong đó: `X` = **các đặc trưng** (dữ liệu đầu vào, ví dụ diện tích/số phòng/tuổi nhà) ·
`y` = **đáp án** (ví dụ giá nhà thật).

Đổi từ cây quyết định sang SVM sang KNN — **hai dòng cuối không đổi một chữ**, chỉ đổi dòng đầu.
Đây là lý do khoá dạy được nhiều thuật toán trong thời gian ngắn.

🧠 **CẦN NHỚ:** nghi thức scikit-learn = **khởi tạo → `fit` (học) → `predict` (dùng)**.

⚠ **BẪY:** scikit-learn chỉ dành cho **ML cổ điển**, **KHÔNG** dùng cho deep learning. Mạng nơ-ron sâu
dùng thư viện khác (TensorFlow/Keras, PyTorch) — đó là nội dung hai khoá tiếp theo trong lộ trình của bạn.

---

# ĐOẠN 2 — KHOÁ DÀNH CHO AI, CẦN BIẾT SẴN GÌ

> *"Whether you're starting your career or enhancing your skills, this **intermediate-level** course will
> provide you with a solid foundation in applied machine learning... a working knowledge of **Python, Pandas,
> NumPy**, and **data preparation and data analysis** with Python is recommended. Familiarity with the field
> of machine learning will be an added advantage."*

## 2.1 "Intermediate-level" — trung cấp cái gì?

Ghép với câu cuối (*biết ML từ trước chỉ là "lợi thế"*, không bắt buộc), ta đọc ra định vị thật:

> **Trung cấp về PYTHON — nhập môn về MACHINE LEARNING.**

Tức: nó giả định bạn gõ Python được rồi, còn ML thì sẽ dạy từ đầu. Không cần lo.

## 2.2 Ba thư viện — mỗi cái một việc, không thay nhau được

Đây là chỗ hay lẫn nhất với người mới:

| Thư viện | Lo việc gì | Hình dung |
|---|---|---|
| **NumPy** | mảng số nhiều chiều, tính toán nhanh | **bảng số thuần** — chỉ có số, không tên cột |
| **Pandas** | bảng dữ liệu có tên cột: đọc, lọc, ghép, nhóm | **file Excel viết bằng code** |
| **scikit-learn** | các thuật toán học máy | **bộ dao** |

Dòng chảy thực tế của một dự án:

```
file .csv  ──▶  Pandas  ──▶  NumPy  ──▶  scikit-learn  ──▶  kết quả
               (đọc vào,     (biến      (fit / predict)
                làm sạch)     thành số)
```

💡 **VÍ DỤ ĐỜI THỰC:** Pandas là **người đi chợ và sơ chế** · NumPy là **thớt và nguyên liệu đã thái** ·
scikit-learn là **đầu bếp**. Ba vai riêng biệt.

🧠 **CẦN NHỚ:** **Pandas = bảng có tên cột · NumPy = mảng số · scikit-learn = thuật toán.**

## 2.3 "Data preparation" — món nặng nhất, bị giấu ở cuối câu

Ba thư viện trên là *công cụ*. Nhưng **chuẩn bị dữ liệu** mới là món tốn thời gian nhất trong dự án thật.
Nó gồm những việc rất "tay chân":

| Vấn đề | Ví dụ cụ thể | Phải làm gì |
|---|---|---|
| **Thiếu dữ liệu** | cột thu nhập có 300 ô trống | điền trung bình? bỏ dòng? bỏ cả cột? |
| **Sai đơn vị** | giá nhà chỗ ghi `2.5` (tỉ), chỗ ghi `2500000000` (đồng) | quy về một đơn vị |
| **Chữ không phải số** | máy không hiểu `"Hà Nội"` | mã hoá thành số |
| **Chênh thang đo** | tuổi (20–70) vs thu nhập (5tr–50tr) | **chuẩn hoá**, nếu không thu nhập sẽ lấn át tuổi |
| **Ngoại lai (outlier)** | một căn 200 tỉ lọt vào danh sách nhà phố | xem xét loại bỏ |

Việc này có tên nghề là **ETL** (Extract – Transform – Load): *thu thập từ nhiều nguồn → làm sạch, biến đổi
→ lưu về một chỗ thống nhất để dùng cho ML*.

🧠 **CẦN NHỚ — câu quan trọng nhất đoạn này:**
> **Lỗi cú pháp thì máy báo. Lỗi dữ liệu thì KHÔNG AI BÁO** — mô hình vẫn chạy, số vẫn ra, chỉ là sai.

💡 **VÍ DỤ ĐỜI THỰC:** dùng dao sai → đứt tay, biết ngay. Chọn phải thịt ôi → món vẫn nấu xong, vẫn thơm,
khách vẫn ăn — rồi mới ngộ độc. **Lỗi dữ liệu là lỗi kiểu thịt ôi.**

---

# ĐOẠN 3 — BẠN SẼ HỌC GÌ *(đoạn quan trọng nhất — 9 khái niệm)*

> *"...You will gain a foundational knowledge of the **machine learning lifecycle** and **how machine learning
> models work**. Throughout the course, you will focus on machine learning modeling techniques, such as
> **classification, regression, and clustering**. Using real-world data, you will learn how these models fit
> within the broader **supervised and unsupervised learning** frameworks. You'll also get a brief introduction
> to advanced methods, such as **reinforcement learning, deep learning**, and the world of **artificial
> intelligence**."*

## 3.1 AI, ML, Deep Learning — ba chữ này xếp thế nào?

Bài giảng ném cả ba chữ ra mà không nói quan hệ. Quan hệ là **lồng nhau như búp bê Nga**:

```
┌───────────────────────────────────────────────────────┐
│  AI — TRÍ TUỆ NHÂN TẠO                                │
│  Làm máy mô phỏng năng lực nhận thức của con người    │
│  (thị giác máy, xử lý ngôn ngữ, AI sinh nội dung…)    │
│                                                       │
│   ┌─────────────────────────────────────────────┐     │
│   │  ML — HỌC MÁY                               │     │
│   │  Học luật từ DỮ LIỆU, không được lập trình  │     │
│   │  luật tường minh                            │     │
│   │  → CON NGƯỜI phải chọn đặc trưng            │     │
│   │                                             │     │
│   │    ┌───────────────────────────────────┐    │     │
│   │    │  DL — HỌC SÂU                     │    │     │
│   │    │  Mạng nơ-ron nhiều lớp,           │    │     │
│   │    │  MÁY TỰ trích đặc trưng           │    │     │
│   │    │  Hợp dữ liệu lớn, phi cấu trúc    │    │     │
│   │    └───────────────────────────────────┘    │     │
│   └─────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────┘

           ← KHOÁ NÀY nằm ở vòng giữa (ML cổ điển) →
```

**Khác biệt cốt lõi giữa ML và DL nằm ở một câu hỏi: AI CHỌN ĐẶC TRƯNG?**

| | ML cổ điển (khoá này) | Deep Learning |
|---|---|---|
| Ai tìm ra "dấu hiệu cần nhìn"? | **Con người** (gọi là *feature engineering*) | **Máy tự tìm** |
| Cần bao nhiêu dữ liệu | ít cũng chạy (vài nghìn dòng) | rất nhiều (chục nghìn → triệu) |
| Hợp loại dữ liệu nào | **bảng biểu** (có cột rõ ràng) | **ảnh, âm thanh, văn bản** |
| Giải thích được vì sao không? | thường **được** | thường **rất khó** |

💡 **VÍ DỤ ĐỜI THỰC — phân biệt ảnh chó / mèo:**
- **ML cổ điển:** *bạn* phải tự nghĩ ra và tự đo các đặc trưng — độ nhọn của tai, độ dài ria, tỉ lệ mắt/mặt —
  rồi đưa **các con số bạn chọn** cho máy học.
- **Deep learning:** bạn ném thẳng **ảnh thô** vào. Máy tự phát hiện ra rằng "tai nhọn" là dấu hiệu đáng chú ý.

🧠 **CẦN NHỚ:** **AI ⊃ ML ⊃ DL.** ML = *người chọn đặc trưng* · DL = *máy tự chọn đặc trưng*.

## 3.2 Machine learning lifecycle — vòng đời của một mô hình

**Định nghĩa:**
> **Chuỗi LẶP:** Định nghĩa bài toán → Thu thập dữ liệu → Chuẩn bị dữ liệu → Phát triển & Đánh giá mô hình
> → Triển khai. **Có thể quay lại pha trước bất cứ lúc nào.**

```
 ┌───────────┐   ┌──────────┐   ┌──────────┐   ┌─────────────┐   ┌───────────┐
 │ Định nghĩa│──▶│ Thu thập │──▶│ Chuẩn bị │──▶│ Phát triển &│──▶│ Triển khai│
 │  bài toán │   │  dữ liệu │   │  dữ liệu │   │   Đánh giá  │   │  (deploy) │
 └───────────┘   └──────────┘   └──────────┘   └─────────────┘   └───────────┘
       ▲                                              │                │
       │                                              │ chưa đạt       │ dữ liệu đời đổi
       └──────────────────────────────────────────────┴────────────────┘
                        QUAY LẠI — đây KHÔNG phải đường thẳng
```

**Vì sao nó LẶP?** Ba lý do rất đời:

1. Làm tới bước 4 mới phát hiện **dữ liệu không đủ để trả lời câu hỏi** → quay về bước 1 sửa lại câu hỏi.
2. Mô hình chấm điểm kém → quay về bước 3 làm sạch kỹ hơn, hoặc bước 2 lấy thêm dữ liệu.
3. **Quan trọng nhất:** triển khai chạy ngon rồi, nhưng **thế giới đổi** → mô hình cũ đi.

💡 **VÍ DỤ ĐỜI THỰC — vì sao pha cuối phải quay về đầu:**
Một ngân hàng dựng mô hình chấm rủi ro cho vay. Chạy tốt, nên **giải tán team**. Hai năm sau kinh tế đổi,
thói quen trả nợ của người dân đổi. Mô hình **vẫn tự tin phán** — nhưng phán sai, và **không còn ai ngồi đó
để phát hiện**. Hiện tượng này có tên nghề: **data drift** (dữ liệu trôi).

🧠 **CẦN NHỚ:** vòng đời ML là **vòng tròn**, không phải đường thẳng. **Triển khai xong không phải là hết việc.**

🛠 **CẦN HỌC:** kể lại 5 pha đúng thứ tự + chỉ ra ETL nằm ở pha nào.
*(Đáp án: ETL nằm ở pha 2–3, tức thu thập + chuẩn bị.)*

## 3.3 "How machine learning models work" — mô hình học kiểu gì

Nói gọn, mọi mô hình trong khoá này đều làm đúng một việc:

> **Tìm bộ tham số sao cho dự đoán lệch ít nhất so với đáp án thật đã biết.**

Ba nhịp, lặp đi lặp lại:

```
1. ĐOÁN BỪA     máy khởi tạo tham số lung tung → đoán sai tè le
2. ĐO ĐỘ SAI    so với đáp án đúng → ra MỘT CON SỐ "sai bao nhiêu"   ← gọi là LOSS (hoặc COST)
3. SỬA          chỉnh tham số cho con số đó nhỏ lại  →  quay về bước 2
```

Lặp tới khi không cải thiện nữa. **Toàn bộ ba nhịp đó gói trong một dòng `model.fit(X, y)`.**

💡 **VÍ DỤ ĐỜI THỰC:** tập ném phi tiêu bịt mắt, có người đứng bên nói "chệch phải 20cm". Bạn chỉnh tay,
ném lại — "chệch phải 5cm". Chỉnh tiếp. **"Chệch bao nhiêu cm" chính là loss.**

🧠 **CẦN NHỚ:** học = **giảm dần độ sai**. **Loss / cost** = con số đo mức sai, **càng nhỏ càng tốt**.

## 3.4 BA HỌ MÔ HÌNH — lõi của cả khoá học

Ba chữ này bạn sẽ gặp mỗi ngày. Phân biệt chúng bằng **câu trả lời trông như thế nào**:

| Họ | Trả lời ra cái gì | Câu hỏi mẫu | Trong project của bạn |
|---|---|---|---|
| **Classification**<br>(phân loại) | **một nhãn** trong danh sách có sẵn | *"Khối u này lành hay ác?"* | chẩn đoán khối u · duyệt vay · khách rời bỏ |
| **Regression**<br>(hồi quy) | **một con số** liên tục | *"Căn nhà này giá bao nhiêu?"* | định giá nhà |
| **Clustering**<br>(gom cụm) | **các nhóm** máy tự chia, không có tên sẵn | *"Khách của tôi chia làm mấy kiểu?"* | phân nhóm khách hàng |

**Mẹo phân biệt classification vs regression** — rất hay nhầm:

> ### 👉 Nhìn vào **ĐÁP ÁN**, đừng nhìn vào câu hỏi.
> Đáp án **đếm được, hữu hạn** → *classification* · Đáp án **là số đo, liên tục** → *regression*.

Thử nhanh:

| Bài toán | Đáp án trông thế nào | Họ nào |
|---|---|---|
| Email này có phải spam không? | spam / không spam | **classification** |
| Ngày mai nhiệt độ bao nhiêu? | 31.4 °C | **regression** |
| Bệnh nhân sẽ nằm viện mấy ngày? | 5 · 7 · 12.5 ngày… | **regression** |
| Khách này có trả được nợ không? | trả được / vỡ nợ | **classification** |
| Chia 50.000 khách thành các nhóm giống nhau | *không có đáp án cho trước* | **clustering** |

⚠ **BẪY:** *"nằm viện mấy ngày"* nghe như đếm nên dễ tưởng phân loại. Nhưng nó là **số đo trên thang liên tục**
(5.5 ngày vẫn có nghĩa) → **regression**.

🛠 **CẦN HỌC:** đưa một bài toán bất kỳ → xếp đúng 1 trong 3 họ + **giải thích được vì sao**.

## 3.5 HAI KHUNG LỚN — supervised và unsupervised

Ba họ trên nằm gọn trong hai cái khung lớn hơn. Phân biệt bằng **đúng một câu hỏi**:

> # ❓ *Dữ liệu của tôi có kèm ĐÁP ÁN không?*

```
      CÓ đáp án sẵn                        KHÔNG có đáp án
      ─────────────────                    ──────────────────
      SUPERVISED                           UNSUPERVISED
      (học CÓ giám sát)                    (học KHÔNG giám sát)

  10.000 hồ sơ vay                     10.000 khách hàng,
  + kết quả thật của từng hồ sơ        không nhãn gì cả
    (trả được / vỡ nợ)                        
           ↓                                     ↓
  Máy học ánh xạ: hồ sơ → kết quả      Máy tự tìm cấu trúc ẩn
           ↓                                     ↓
  • Classification                     • Clustering (gom cụm)
  • Regression                         • Giảm chiều dữ liệu
```

**Định nghĩa gọn:**
- **Supervised learning:** huấn luyện trên dữ liệu **CÓ NHÃN**, để suy ra nhãn cho dữ liệu mới.
- **Unsupervised learning:** học trên dữ liệu **KHÔNG nhãn**, tìm mẫu / cấu trúc ẩn.

Bài giảng chỉ nhắc hai khung này. Thực tế có **bốn kiểu học** — hai cái còn lại bạn nên biết tên:

| Kiểu | Dữ liệu thế nào | Ví dụ đời thực |
|---|---|---|
| **Supervised** | có nhãn đầy đủ | Gmail học lọc spam từ email người dùng đã bấm "báo spam" |
| **Unsupervised** | không nhãn | siêu thị tự chia khách thành các nhóm mua sắm |
| **Semi-supervised** | nhãn **một phần** — học trên phần có nhãn rồi tự gán nhãn cho phần còn lại | 1.000 ảnh X-quang bác sĩ đã đọc + 50.000 ảnh chưa ai đọc |
| **Reinforcement** | **không có nhãn, chỉ có thưởng/phạt** | AI chơi Mario: qua màn = thưởng, chết = phạt |

🧠 **CẦN NHỚ — câu hỏi vàng:**
> **"Dữ liệu có kèm đáp án không?"** — Có → **supervised** · Không → **unsupervised** ·
> Không có đáp án mà chỉ có thưởng/phạt → **reinforcement**.

## 3.6 Reinforcement learning — học bằng thưởng và phạt

**Định nghĩa:**
> Một *tác nhân* (**agent**) tương tác với *môi trường* (**environment**), chọn *hành động* (**action**),
> nhận *phần thưởng* (**reward**) — thường **trễ**, và **không ai nói trước hành động nào là đúng**.
> Nó học dần để tối đa **tổng** phần thưởng.

Khác supervised ở chỗ này:

| | Supervised | Reinforcement |
|---|---|---|
| Máy có được biết đáp án đúng không? | **có**, ngay lập tức | **không** — chỉ biết điểm sau cả một chuỗi hành động |
| Phản hồi tới lúc nào | tức thì | **trễ** (đi 50 nước cờ mới biết thắng hay thua) |

💡 **VÍ DỤ ĐỜI THỰC:** dạy chó ngồi. Bạn không nói được tiếng chó để giải thích *"ngồi nghĩa là hạ mông xuống"*.
Bạn chỉ **thưởng bánh mỗi khi nó tình cờ ngồi đúng**. Sau vài chục lần, nó tự hiểu.
**Đó chính xác là reinforcement learning.**

Bài giảng ghi *"brief introduction"* (giới thiệu sơ qua) — đúng vậy, khoá này chỉ nêu tên. Nó là nội dung
đầy đủ của khoá thứ ba trong lộ trình của bạn.

---

# ĐOẠN 4 — BẠN SẼ TỰ TAY LÀM GÌ

> *"You will gain **hands-on** experience with **building, assessing, and validating** various classification,
> regression, and clustering machine learning models using Python and popular **open-source libraries**,
> such as Pandas, NumPy, and Scikit-Learn."*

## 4.1 Ba động từ — đây là cam kết thật của khoá học

Cả bài giới thiệu dài dòng, nhưng phần "bạn **tự tay làm được**" rút gọn chỉ còn:

> # 3 động từ × 3 họ mô hình
> **build (dựng) · assess (chấm) · validate (kiểm chứng)**
> trên **classification · regression · clustering**

Nếu chỉ được nhớ một dòng trong cả file này, hãy nhớ dòng đó.

## 4.2 "Assess" và "validate" khác nhau chỗ nào? *(rất hay nhầm)*

Hai chữ nghe na ná, nhưng khác nhau ở **dữ liệu dùng để chấm**:

| | **Assess** (chấm) | **Validate** (kiểm chứng) |
|---|---|---|
| Chấm trên dữ liệu nào | dữ liệu mô hình **ĐÃ nhìn thấy** khi học | dữ liệu mô hình **CHƯA từng thấy** |
| Trả lời câu gì | *"Học thuộc bài chưa?"* | *"Ra đời có dùng được không?"* |
| Điểm cao có đáng tin không | **KHÔNG** | **CÓ** |

💡 **VÍ DỤ ĐỜI THỰC:** cho học sinh làm lại **đúng đề nó đã có đáp án** → 10 điểm. Đó là *assess*.
Cho nó **đề mới chưa từng thấy** → mới biết thật sự học được gì. Đó là *validate*.

Vì vậy mới có nghi thức **train-test split**: chia dữ liệu ra **~70–80% để học**, phần còn lại **giấu đi**,
chỉ lôi ra lúc chấm cuối cùng.

⚠ **BẪY LỚN NHẤT CỦA CẢ NGHỀ:** mô hình đạt **99% trên dữ liệu đã học** thì **không có nghĩa gì cả** —
nó đã biết trước đáp án rồi. Bệnh này tên là **overfitting** (quá khớp):

> **Overfitting** = mô hình quá phức tạp, học thuộc cả **nhiễu** của bộ dữ liệu thay vì học quy luật.
> Kết quả: khớp dữ liệu cũ rất tốt, nhưng gặp dữ liệu mới thì tệ.
>
> Bệnh ngược lại là **underfitting** (chưa khớp): mô hình quá đơn giản (vẽ đường thẳng cho dữ liệu cong),
> không nắm được cả xu hướng chính.

🧠 **CẦN NHỚ:** **điểm trên dữ liệu đã học KHÔNG TÍNH.** Chỉ điểm trên dữ liệu chưa từng thấy mới tính.

## 4.3 "Open-source libraries" — mã nguồn mở, vì sao đáng quan tâm

**Mã nguồn mở** = mã công khai, ai cũng đọc / dùng / sửa được, miễn phí.

Với người học thì đó là *"đỡ tốn tiền"*. Với người đi làm, ý nghĩa lớn hơn:

- Bạn **đọc được** mô hình thực sự tính gì → khi nó ra kết quả lạ, bạn truy được nguyên nhân.
- Kỹ năng bạn học **mang đi được** sang công ty khác, không bị khoá vào một sản phẩm trả tiền.
- Cả thế giới cùng dùng nên lỗi được phát hiện và vá rất nhanh.

---

# ĐOẠN 5 — HỌC VÀ CHẤM BẰNG CÁCH NÀO

> *"...through **instructional videos** and then practice in **hands-on labs**. You will assess your learning
> with **practice and graded quizzes**... A concluding **final project** in the last module... **Lab topics
> include multiple linear regression and logistic regression, prediction, fraud detection, KNN, and SVM.**"*

## 5.1 Bốn tầng học — xếp theo độ tin cậy

| Tầng | Hình thức | Chứng minh được gì | Tin được không |
|---|---|---|---|
| 1 | Video | bạn đã **nghe** | thấp nhất — nghe ≠ hiểu |
| 2 | Lab | bạn đã **gõ theo** | gõ theo ≠ tự làm được |
| 3 | Quiz | bạn **nhớ** | nhớ ≠ vận dụng |
| 4 | Final project | bạn **tự ghép được** | **cao nhất** |

🧠 **CẦN NHỚ:** **nghe < gõ theo < nhớ < tự ghép được.** Chỉ tầng 4 là bằng chứng thật.

## 5.2 SÁU CHỦ ĐỀ LAB — giải thích từng cái

Bài giảng liệt kê 6 cái tên mà không giải thích. Đây là chúng.

---

### ① Multiple Linear Regression — hồi quy tuyến tính bội

**Định nghĩa:**
> Mô hình hoá quan hệ **tuyến tính** giữa một mục tiêu **liên tục** và các đặc trưng:
> `ŷ = θ₀ + θ₁·x₁ + θ₂·x₂ + …`

Đọc công thức cho dễ hiểu:

```
giá nhà  =  số nền  +  (hệ số₁ × diện tích)
                    +  (hệ số₂ × số phòng ngủ)
                    +  (hệ số₃ × tuổi nhà)
                    +  (hệ số₄ × khoảng cách vào trung tâm)
```

- **"Linear"** (tuyến tính) = quan hệ **đường thẳng**: tăng 1 đơn vị đầu vào → tăng **đều** một lượng đầu ra.
- **"Multiple"** (bội) = dùng **nhiều** đặc trưng, không phải một.
- Việc học của máy chính là **tìm ra các hệ số** θ₀, θ₁, θ₂… sao cho sai số nhỏ nhất.

💡 **VÍ DỤ ĐỜI THỰC:** đây đúng là dịch vụ **định giá nhà** trong project của bạn — nhận diện tích, số phòng,
tuổi nhà, vị trí → trả về giá ước tính.

⚠ **BẪY — collinearity (đa cộng tuyến):** nếu hai đặc trưng **nói cùng một chuyện** — ví dụ *"diện tích m²"*
và *"diện tích ft²"*, hoặc *"số phòng ngủ"* và *"tổng số phòng"* — mô hình sẽ loạn vì không biết chia công lao
cho ai. Phải bỏ bớt một.

---

### ② Logistic Regression — hồi quy logistic

**Định nghĩa:**
> Kỹ thuật dự đoán **XÁC SUẤT** một quan sát thuộc **một trong hai lớp**.
> Trong ML, nó là một **bộ phân loại nhị phân** (binary classifier).

> ## ⚠ BẪY TÊN GỌI — gần như chắc chắn sẽ bị hỏi
> Tên nó có chữ **"regression"** (hồi quy), nhưng nó là **CLASSIFICATION** (phân loại) — **không phải** regression!

**Vì sao tên lạ vậy?** Vì bên trong nó **có** làm hồi quy — nhưng hồi quy ra một **xác suất**, rồi cắt ngưỡng:

```
hồ sơ vay ──▶ [tính toán kiểu hồi quy] ──▶ 0.87 ──▶ so với ngưỡng 0.5 ──▶ "VỠ NỢ"
                                        xác suất                          một NHÃN
                                                                     (nên là phân loại)
```

Cái hàm ép mọi con số về khoảng 0–1 để thành xác suất tên là **sigmoid**:

```
σ(x) = 1 / (1 + e⁻ˣ)        σ(0) = 0.5

  1 ┤                    ╭──────────
    │                ╭───╯
0.5 ┤ ─ ─ ─ ─ ─ ╭───╯ ─ ─ ─ ─ ─ ─ ─    ← ngưỡng quyết định
    │       ╭───╯
  0 ┤───────╯
    └────────────┴─────────────────
                 0
```

Con số ngưỡng **0.5** gọi là **decision boundary** (ranh giới quyết định) — và **nó chỉnh được**.
Đây là quyết định **kinh doanh**, không phải hằng số toán học: ngân hàng sợ rủi ro có thể hạ ngưỡng xuống 0.3
để từ chối nhiều hơn.

💡 **VÍ DỤ ĐỜI THỰC:** đúng là dịch vụ **chấm rủi ro tín dụng** trong project của bạn — nó trả về
`default_probability` (xác suất vỡ nợ) rồi mới quy ra `approved` (duyệt hay không).

---

### ③ Prediction — "dự đoán"

⚠ **Chỗ này bản gốc viết cẩu thả.** Năm cái tên kia đều là **kỹ thuật**; *"prediction"* thì **không** —
nó là **hành động** mà mọi mô hình đều làm, chính là `model.predict()`.

Xếp nó chung danh sách giống như liệt kê: *"dao, thớt, chảo, **nấu ăn**, muỗng"*.

🧠 **CẦN NHỚ:** *prediction* **không phải một loại mô hình** — nó là **việc dùng** mô hình đã học xong.

---

### ④ Fraud Detection — phát hiện gian lận

Đây **không** phải một thuật toán, mà là một **bài toán ứng dụng** (thường giải bằng classification):
*giao dịch thẻ này là thật hay gian lận?*

Nó được lôi ra làm lab riêng vì có một đặc điểm khiến người mới **ngã sấp mặt**: **dữ liệu mất cân bằng**.

```
1.000.000 giao dịch  →  999.000 thật  ·  1.000 gian lận   (chỉ 0.1%)
```

Tôi viết cho bạn một "mô hình" đạt **99.9% chính xác**:

```python
def mo_hinh_sieu_viet(giao_dich):
    return "thật"          # luôn luôn trả lời "thật"
```

Nó đúng 999.000/1.000.000 = **99.9%** — và bắt được **0** vụ gian lận. Hoàn toàn vô dụng.

🧠 **CẦN NHỚ:** khi dữ liệu **mất cân bằng**, **accuracy (độ chính xác) là chỉ số LỪA ĐẢO.**

Phải dùng chỉ số khác:

| Chỉ số | Trả lời câu hỏi | Quan trọng khi nào |
|---|---|---|
| **Precision** (độ chuẩn xác) | *"Trong những ca tôi BÁO là gian lận, bao nhiêu đúng?"* | khi báo nhầm gây tốn kém (làm phiền khách) |
| **Recall** (độ bao phủ) | *"Trong những ca gian lận THẬT, tôi bắt được bao nhiêu?"* | khi bỏ sót gây tốn kém (**y tế!**) |

💡 **VÍ DỤ ĐỜI THỰC:** đây cũng là chuyện của **chẩn đoán ung thư** trong project bạn — bỏ sót một ca ác tính
(recall thấp) nguy hiểm hơn nhiều so với báo động nhầm một ca lành.

---

### ⑤ KNN — K-Nearest Neighbors (K láng giềng gần nhất)

**Định nghĩa:**
> Gán nhãn cho điểm cần đoán theo **k điểm có nhãn gần nó nhất**. Là một *"lazy learner"* (kẻ học lười).
> Dùng được cho cả phân loại lẫn hồi quy.

Cơ chế đơn giản đến bất ngờ — **không có công thức nào cả**:

```
Cần đoán điểm  ✚  :   tìm k điểm gần nhất  →  bên nào đông hơn thì theo bên đó

     với k = 3:        ●   ●
                          ✚          →  2 hình tròn, 1 hình vuông
                        ■              →  ✚ là hình TRÒN
```

**Vì sao gọi là "học lười"?** Vì lúc `fit` nó **không học gì cả** — nó chỉ **ghi nhớ toàn bộ dữ liệu**.
Mọi tính toán bị hoãn đến lúc `predict`.

💡 **VÍ DỤ ĐỜI THỰC:** *"5 căn giống nhà này nhất trong khu vực bán trung bình 3 tỉ, vậy nhà này chắc ~3 tỉ."*
→ Môi giới bất động sản làm KNN bằng tay mỗi ngày.

⚠ **BẪY:** vì KNN đo **khoảng cách**, nếu các cột chênh thang đo (tuổi 20–70 vs thu nhập hàng chục triệu),
cột số lớn sẽ **nuốt chửng** cột số nhỏ → **bắt buộc chuẩn hoá dữ liệu trước.**
👉 Đây chính là chỗ *"data preparation"* ở Đoạn 2 quay lại đòi nợ.

---

### ⑥ SVM — Support Vector Machine (máy vector hỗ trợ)

**Định nghĩa:**
> Kỹ thuật supervised dựng mô hình phân loại/hồi quy bằng cách **ánh xạ điểm vào không gian nhiều chiều**
> rồi tìm **siêu phẳng** phân tách các lớp.

Nghe đáng sợ, nhưng chỉ có **hai ý tưởng**:

**Ý tưởng 1 — kẻ ranh giới có LỀ RỘNG NHẤT.**
Có vô số đường tách được hai lớp; SVM chọn đường **cách xa cả hai bên nhất**:

```
  ●  ●                     ●  ●
    ● ● ┊                    ● ●  ║   ║   ║
─────────┊──────         ─────────────────────
    ■ ■  ┊                    ■ ■  ║   ║   ║
  ■   ■                    ■   ■

  đường sát rìa            đường có LỀ RỘNG   ← SVM chọn cái này
  (dữ liệu mới lệch        (dữ liệu mới lệch
   một tí là sai)           một tí vẫn đúng)
```

- Khoảng trống hai bên gọi là **margin** (lề).
- Các điểm nằm sát mép, quyết định vị trí đường kẻ, gọi là **support vectors** (vector hỗ trợ) —
  **tên thuật toán đến từ đây.**

**Ý tưởng 2 — kernel trick: nâng chiều cho cái không tách được thành tách được.**

```
Trên mặt phẳng: KHÔNG kẻ nổi          Nâng lên 3 chiều: một LÁT PHẲNG là tách được
đường thẳng nào tách được
                                                ╱───────────╱
      ■ ■ ■ ■ ■                                ╱   ● ● ●   ╱   ← ● được nhấc lên cao
    ■   ● ● ●   ■           ──▶               ╱───────────╱
    ■   ● ● ●   ■                            ■ ■ ■ ■ ■ ■       ← ■ nằm dưới
      ■ ■ ■ ■ ■
```

Phép nâng chiều đó gọi là **kerneling**; các kiểu kernel hay dùng: *linear, polynomial, RBF, sigmoid*.

💡 **VÍ DỤ ĐỜI THỰC:** trên một tờ giấy, hai đàn kiến trộn vào nhau — không kẻ nổi một đường thẳng để tách.
Nhưng nếu **nhấc một đàn lên khỏi mặt giấy**, chỉ cần luồn một tờ bìa phẳng vào giữa là xong.
**Kernel trick làm đúng việc "nhấc lên" đó — bằng toán.**

---

# TỔNG KẾT

## 🧠 CẦN NHỚ — 12 câu

1. **ML = học luật từ ví dụ**, thay vì được người viết luật cho.
2. **Analytics** nhìn quá khứ, nhìn tổng thể · **ML** nhìn tương lai, nhìn từng ca.
3. Nghi thức scikit-learn: **khởi tạo → `fit` (học) → `predict` (dùng)** — mọi thuật toán đều giống nhau.
4. **Pandas** = bảng có tên cột · **NumPy** = mảng số · **scikit-learn** = thuật toán.
5. **Lỗi cú pháp thì máy báo, lỗi dữ liệu thì không ai báo.**
6. **AI ⊃ ML ⊃ DL.** ML = người chọn đặc trưng · DL = máy tự chọn đặc trưng.
7. Vòng đời ML là **vòng tròn**, không phải đường thẳng — triển khai xong chưa phải hết việc.
8. Mô hình học = **giảm dần độ sai (loss)**.
9. Ba họ: **classification** (ra nhãn) · **regression** (ra số) · **clustering** (ra nhóm).
   Phân biệt bằng cách nhìn **đáp án**, không nhìn câu hỏi.
10. Câu hỏi vàng chia hai khung: **"Dữ liệu có kèm đáp án không?"** Có → supervised · Không → unsupervised.
11. Cam kết thật của khoá = **3 động từ (build · assess · validate) × 3 họ mô hình.**
12. **Điểm trên dữ liệu đã học không tính.** Chỉ dữ liệu chưa từng thấy mới tính.

## ⚠ BỐN CÁI BẪY — nhớ riêng

| Bẫy | Sự thật |
|---|---|
| "Logistic **regression** chắc là hồi quy" | **KHÔNG** — nó là **phân loại** nhị phân |
| "Accuracy 99% là mô hình giỏi" | **KHÔNG** — với dữ liệu mất cân bằng (gian lận, ung thư) accuracy là chỉ số lừa đảo |
| "Prediction là một loại mô hình" | **KHÔNG** — nó là *việc dùng* mô hình đã học xong |
| "Chuẩn bị dữ liệu là việc lặt vặt" | **KHÔNG** — tốn thời gian nhất, và là loại lỗi máy không báo |

## 🛠 CẦN HỌC — 4 kỹ năng (phải LÀM ĐƯỢC)

1. Cho một bài toán bất kỳ → xếp đúng vào **classification / regression / clustering** + giải thích vì sao.
2. Cho một tình huống → xếp đúng vào **supervised / unsupervised / semi / reinforcement**.
3. Kể lại **5 pha vòng đời ML** đúng thứ tự + chỉ ra ETL nằm ở đâu + nói được vì sao nó lặp.
4. Tự chạy trọn nghi thức scikit-learn: **chuẩn hoá → chia train/test → `fit` → `predict` → chấm điểm**.

### Bốn kỹ năng này nối với sổ tri thức của bạn

| Kỹ năng trong sổ | Trạng thái |
|---|---|
| `F-ml-vs-ai` — phân biệt ML / AI / DL & 4 kiểu học | `đang-học` |
| `F-ml-technique-select` — chọn đúng họ kỹ thuật | `đang-học` |
| `F-ml-lifecycle` — vòng đời mô hình | `đang-học` |
| `F-sklearn-workflow` — chạy workflow scikit-learn | `đang-học` |

Cả bốn đang ở `đang-học` chứ chưa `đã-làm-được` — vì bạn mới trả lời đúng quiz (tầng 3),
chưa tự làm biến thể trên project (tầng 4).

---

# BÀI TẬP — làm được mới tính là hiểu

Mỗi bài trả lời vài dòng là đủ. Không cần viết dài.

**Bài 1 — xếp họ mô hình** *(kỹ năng 🛠 #1)*
Xếp 4 bài toán sau vào classification / regression / clustering:
- a) Dự đoán số lượng đơn hàng ngày mai
- b) Nhận diện chữ số viết tay 0–9
- c) Chia 50.000 khách thành các nhóm hành vi
- d) Đoán một người có mắc tiểu đường hay không

**Bài 2 — bẫy tên gọi** *(kỹ năng 🛠 #1)*
Đồng nghiệp nói: *"Dùng logistic regression đi, mình cần đoán **giá** nhà mà."*
Câu này sai ở đâu? Nên dùng cái gì?

**Bài 3 — bẫy accuracy** *(kỹ năng 🛠 #1, khó)*
Mô hình phát hiện gian lận báo cáo **accuracy 99.8%**. Sếp rất vui.
Bạn cần hỏi **đúng một câu** để biết mô hình có thật sự dùng được không. Câu đó là gì?

**Bài 4 — vòng đời** *(kỹ năng 🛠 #3)*
Một công ty triển khai mô hình gợi ý sản phẩm, chạy tốt 2 năm rồi hiệu quả tụt dần
dù **không ai đụng vào code**. Chuyện gì đã xảy ra? Theo vòng đời, họ phải quay lại pha nào?

**Bài 5 — chạy thật** *(kỹ năng 🛠 #4, cần gõ lệnh)*
```bash
cd /home/hai-soft/projects/icpp/sleep/project/machine-learning-with-python
.venv/bin/python -m mlstudio data
```
Nhìn kết quả in ra và cho biết: nó thuộc động từ nào trong **build / assess / validate**?

---

# HỎI & ĐÁP

> Mục này sẽ **dài dần**. Bạn hỏi gì, tôi ghi câu trả lời vào đây kèm ngày,
> để lần sau đọc lại file là đủ, không phải lục lại chat.

*(chưa có câu hỏi nào)*

---

## PHỤ LỤC — tra lại nguồn của mọi định nghĩa

Mọi định nghĩa in trong khung trích dẫn đều lấy từ sổ tri thức đã kiểm chứng, tra lại được:

```bash
cd /home/hai-soft/projects/icpp/sleep
python3 rail.py q "Supervised learning"
python3 rail.py q "K-Nearest"
python3 rail.py q "Logistic regression"
python3 rail.py q "ML Model Lifecycle"
python3 rail.py q "Support Vector Machine"
```

Chỗ tôi **không đồng ý với bản gốc** đã ghi rõ: mục 5.2-③ (*"prediction"* bị xếp nhầm vào danh sách kỹ thuật).

**Ngày đo:** 2026-08-14.

## LỊCH SỬ CẬP NHẬT

| Ngày | Thay đổi |
|---|---|
| 2026-08-14 | Bản đầu — **viết lệch**: dạy *cách đọc tài liệu* thay vì dạy *nội dung tài liệu*. Human bác. |
| 2026-08-14 | **Viết lại đúng ý:** đi theo 5 đoạn của bài giảng, giải thích **toàn bộ 20 khái niệm** nó nhắc tới. Bệnh cũ đã khắc thành luật kernel BỔ SUNG 10 + `SK-TUTOR-CLARITY` để không tái phạm. |
