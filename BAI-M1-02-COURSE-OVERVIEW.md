# BÀI M1-02 — COURSE OVERVIEW: BẢN ĐỒ TOÀN KHOÁ

> **Gia sư:** SLEEP · **Học viên:** 1 người · **Ngày soạn:** 2026-08-14
> **Bài giảng gốc:** `lectures/Machine Learning with Python/m1/m1 - 00 - Course Overview (reading).txt`
> (bạn đưa từ `~/Downloads/Course Overview.txt` — đã lưu vào nhà)
> **Bài trước:** [`BAI-M1-01-COURSE-INTRODUCTION.md`](./BAI-M1-01-COURSE-INTRODUCTION.md) — bài này **dùng lại**
> 5 câu hạt nhân ở đó, nên nếu quên thì mở lại xem trước.

---

## CÁCH ĐỌC FILE NÀY

| Dấu | Nghĩa | Bạn phải làm gì |
|---|---|---|
| 🧠 **CẦN NHỚ** | Sự thật, định nghĩa | Đọc, nhắc lại được là đủ |
| 🛠 **CẦN HỌC** | Kỹ năng | Phải **làm được** |
| 💡 **VÍ DỤ ĐỜI THỰC** | Chuyện đời để dễ nhớ | Đọc cho ngấm |
| ⚠ **BẪY** | Chỗ dễ hiểu sai | Đọc kỹ |
| 🔭 **ĐỂ SAU** | Khái niệm chỉ được **nêu tên** ở bài này, sẽ có bài riêng | Biết nó là gì là đủ, **đừng cố hiểu sâu bây giờ** |

> **Bài này là BẢN ĐỒ, không phải bài kỹ thuật.** Nó nhắc tên rất nhiều thuật toán (KNN, SVM, DBSCAN, PCA,
> t-SNE, UMAP…) mà chưa dạy cái nào. Việc của bạn ở bài này: **biết mỗi thuật toán nằm ở đâu trên đường đi
> và giải bài toán gì** — không phải hiểu cách nó chạy. Cái đó có bài riêng.

---

# PHẦN 1 — KHOÁ NÀY NẰM Ở ĐÂU TRONG LỘ TRÌNH

> *"This course is a part of both the **IBM AI Engineering** Professional Certificate and the **IBM Data
> Science** Professional Certificate."*

Khoá này là **giao điểm của hai lộ trình nghề**:

```
        LỘ TRÌNH DATA SCIENCE                    LỘ TRÌNH AI ENGINEERING
        (làm việc với dữ liệu)                   (xây hệ thống thông minh)
                  │                                        │
                  └──────────────┬─────────────────────────┘
                                 ▼
                   ⭐ MACHINE LEARNING WITH PYTHON
                        (khoá bạn đang học)
                                 │
                  ┌──────────────┴─────────────────────────┐
                  ▼                                        ▼
        dừng ở đây → làm data scientist        đi tiếp → deep learning,
                                                 mạng nơ-ron, AI engineer
```

Bản gốc gợi ý: học xong Data Science PC thì **nên học tiếp AI Engineering PC**. Trong lộ trình của bạn,
đó đúng là hai khoá tiếp theo đã nằm sẵn trong sổ: **DL-KERAS** và **DL-ADV**.

🧠 **CẦN NHỚ:** đây là khoá **bản lề** — ai làm dữ liệu cũng phải qua, ai làm AI cũng phải qua.
Nó là phần chung của hai nghề.

---

# PHẦN 2 — CẦN BIẾT SẴN GÌ *(bổ sung so với bài M1-01)*

> *"This course requires a working knowledge of Python and Python libraries, such as Pandas, NumPy, and more,
> to perform data preparation and data analysis. **High School level mathematics is an asset.**"*

Ba thứ đầu (Python, Pandas, NumPy, chuẩn bị dữ liệu) đã giải thích kỹ ở **bài M1-01 mục 2.2–2.3**.
Đây là **cái mới** của bài này:

## 2.1 "Toán cấp ba là một lợi thế" — cụ thể là toán gì?

Chữ **"asset"** = *tài sản / lợi thế*, **không phải bắt buộc**. Bạn không cần giỏi toán để học khoá này.
Nhưng để bạn yên tâm, đây là chính xác những mảnh toán cấp 3 sẽ xuất hiện — chỉ có 4 mảnh, và đều nhẹ:

| Mảnh toán | Xuất hiện ở đâu | Bạn thật sự cần biết gì |
|---|---|---|
| **Phương trình đường thẳng** `y = ax + b` | Module 2 — hồi quy tuyến tính | hiểu `a` là độ dốc, `b` là điểm cắt trục tung |
| **Hệ toạ độ, điểm trên mặt phẳng** | Module 3, 4 — KNN, SVM, gom cụm | hiểu "hai điểm gần nhau" nghĩa là gì |
| **Trung bình, phần trăm** | khắp nơi — mọi chỉ số đánh giá | cộng chia lấy trung bình |
| **Hàm số, đồ thị hình chữ S** | Module 2 — hàm sigmoid | nhìn đồ thị biết nó tăng/giảm, chặn ở 0 và 1 |

**Không cần:** đạo hàm, tích phân, đại số tuyến tính, xác suất thống kê nâng cao. Thư viện lo hết.

> **Bản gốc còn liệt kê 4 khoá bổ trợ** nếu bạn thấy hụt Python: *Python for Data Science AI & Development ·
> Python Projects for Data Science · Data Analysis with Python · Data Visualization with Python*.
> Với bạn thì **không cần** — bạn đang học qua project có sẵn code chạy được, không phải học chay.

---

# PHẦN 3 — NĂM MỤC TIÊU CỦA KHOÁ

Bản gốc ghi rõ 5 mục tiêu. Ta đọc chúng bằng **mẹo bắt động từ** đã học ở bài M1-01:

| # | Nguyên văn | Động từ | Mức |
|---|---|---|---|
| 1 | *"**Describe** how machine learning plays a pivotal role in various career paths"* | mô tả | 🔵 **BIẾT** |
| 2 | *"**Articulate** the various stages involved in the machine learning lifecycle"* | nói rành mạch | 🔵 **BIẾT** |
| 3 | *"**Discuss** how various machine learning models work"* | bàn được | 🔵 **BIẾT** |
| 4 | *"**Implement** machine learning models using Python and scikit-learn"* | **cài đặt được** | 🟢 **LÀM ĐƯỢC** |
| 5 | *"**Solve** data-related problems using machine-learning methods"* | **giải được bài toán** | 🟢 **LÀM ĐƯỢC** (cao nhất) |

**Ba mục tiêu đầu là BIẾT. Chỉ hai mục tiêu cuối là LÀM ĐƯỢC** — và hai cái đó mới là phần có giá trị.

## Phân biệt mục tiêu 4 và 5 — chỗ này quan trọng

Hai cái nghe giống nhau nhưng khác hẳn về độ khó:

| | **④ Implement** (cài đặt được) | **⑤ Solve** (giải được bài toán) |
|---|---|---|
| Đề bài đưa cho bạn | *"Dùng decision tree phân loại bộ dữ liệu này"* | *"Công ty đang mất khách, làm gì đó đi"* |
| Bạn phải tự quyết cái gì | không — đã chỉ rõ dùng gì | **mọi thứ**: đây là bài toán gì, dùng họ nào, dữ liệu có đủ không, chấm bằng chỉ số nào |
| Kỹ năng thật sự | gõ đúng cú pháp | **phán đoán** |

💡 **VÍ DỤ ĐỜI THỰC:** ④ là *biết dùng dao*. ⑤ là *nhìn tủ lạnh rồi quyết định nấu món gì*.
Người mới ra trường thường rất giỏi ④ và rất kém ⑤ — và ⑤ mới là thứ công ty trả tiền.

🧠 **CẦN NHỚ:** *implement* = làm theo chỉ dẫn · *solve* = **tự quyết định** phải làm gì. Bậc trên là **solve**.

---

# PHẦN 4 — BẢN ĐỒ 6 MODULE *(phần chính của bài này)*

> Khoá có **6 module**, học trong **4–6 tuần**, mỗi tuần vài giờ.

Điều quan trọng nhất bạn phải thấy: **thứ tự 6 module KHÔNG ngẫu nhiên**. Đó là một **câu chuyện**, trong đó
**mỗi module sinh ra để vá đúng cái lỗ hở mà module trước để lại.**

```
M1 ─────▶ M2 ─────▶ M3 ─────▶ M4 ─────▶ M5 ─────▶ M6
NỀN       ĐƠN GIẢN  MẠNH HƠN  KHÔNG     BIẾT MÌNH  THI +
          NHẤT                NHÃN      CÓ TỰ LỪA  ĐỒ ÁN
                                        KHÔNG

          ↑ lỗ hở:  ↑ lỗ hở:  ↑ lỗ hở:  ↑
          chỉ vẽ    dễ học    (nhánh    câu hỏi cuối:
          được      thuộc,    khác hẳn) mọi số ở trên
          ĐƯỜNG     dễ overfit           là THẬT hay ẢO?
          THẲNG
```

---

## Module 1 — Introduction to Machine Learning *(bạn đang ở đây)*

> *"...foundational machine learning concepts... machine learning modeling is an **iterative process** with
> various lifecycle stages... daily activities of a machine-learning engineer... **scikit-learn**."*

**Dạy gì:** ML là gì · **vòng đời mô hình** (và vì sao nó **lặp**, không phải đường thẳng) · một ngày của
kỹ sư ML · các công cụ mã nguồn mở, đặc biệt là scikit-learn.

**Vì sao đứng đầu:** phải biết mình đang làm nghề gì trước khi gõ dòng code đầu tiên.

*(Toàn bộ nội dung này đã có trong bài M1-01 — mục 1.1, 1.3, 3.1, 3.2.)*

---

## Module 2 — Linear and Logistic Regression

> *"...two **classical statistical methods** foundational to machine learning... linear regression,
> **pioneered in the 1800s**, models linear relationships while logistic regression serves as a classifier.
> Through implementing these models, you'll understand **their limitations** and gain insight into
> **why modern machine-learning models are often preferred**."*

**Dạy gì:** hồi quy tuyến tính (đơn & bội) · hồi quy đa thức/phi tuyến · hồi quy logistic · cách huấn luyện chúng.

**⭐ Vì sao đứng thứ hai — và đây là câu quan trọng nhất của cả bản đồ:**

Đọc kỹ câu bản gốc: học hai mô hình này để **"hiểu GIỚI HẠN của chúng"** và **"vì sao mô hình hiện đại
thường được ưa dùng hơn"**.

> **Tức là: khoá cố tình dạy bạn hai mô hình YẾU trước — để bạn TỰ THẤY chỗ chúng gãy.**
> Thấy chỗ gãy rồi thì mọi thứ ở Module 3 mới có lý do tồn tại.

Hai mô hình này ra đời từ **thế kỷ 19** — trước cả máy tính. Chúng đơn giản đến mức bạn **đọc được từng hệ số**
(*"cứ thêm 1m² thì giá tăng 30 triệu"*), điều mà mô hình hiện đại rất khó làm.

**Lỗ hở chúng để lại → chính là lý do có Module 3:**

| Giới hạn | Nghĩa là gì |
|---|---|
| Chỉ mô hình được quan hệ **đường thẳng** | dữ liệu cong thì nó bó tay |
| Ranh giới phân loại là một **đường thẳng** | hai lớp trộn xoắn vào nhau thì chịu |
| Nhạy với **ngoại lai** (outlier) | một điểm dị thường kéo lệch cả đường |

💡 **VÍ DỤ ĐỜI THỰC:** giống như học vẽ bằng thước kẻ trước. Vẽ được nhà, được cầu thang. Đến lúc phải vẽ
khuôn mặt thì bạn **tự nhận ra** thước kẻ không đủ — và lúc đó bạn mới thật sự hiểu vì sao cần bút cong.

---

## Module 3 — Building Supervised Learning Models

> *"...**binary classification**... construct a **multiclass classifier** from binary classification
> components... **decision trees**, how they learn, and how to build them... **regression trees**...
> **KNN** and **SVM**... what **bias and variance** are in model fitting and the **tradeoff** between them
> that is **inherent to all learning models**... strategies for **mitigating** this tradeoff."*

**Dạy gì — module dày nhất, 5 nhóm nội dung:**

| Nội dung | Là gì (một dòng) | Vá lỗ hở nào của M2 |
|---|---|---|
| **Binary → multiclass** | ghép nhiều bộ phân loại 2-lớp thành bộ nhiều lớp | M2 chỉ làm được 2 lớp |
| 🔭 **Decision tree** (cây quyết định) | lưu đồ hỏi-đáp: mỗi nút một câu hỏi, mỗi lá một kết luận | ranh giới không cần thẳng nữa |
| 🔭 **Regression tree** (cây hồi quy) | cùng cấu trúc cây nhưng lá trả về **một con số** | cây làm được cả bài toán ra số |
| 🔭 **KNN · SVM** | KNN: hỏi k hàng xóm gần nhất · SVM: kẻ ranh giới có lề rộng nhất, nâng chiều được | xử được dữ liệu không tách thẳng |
| ⭐ **Bias–variance tradeoff + ensemble** | đánh đổi có mặt trong **MỌI** mô hình + cách giảm nhẹ | mô hình mạnh lên thì **dễ học thuộc** — phải có cách trị |

*(KNN và SVM đã được giải thích ở bài M1-01 mục 5.2-⑤ và ⑥ — mở lại xem nếu quên.)*

**Lỗ hở M3 để lại:** mọi thứ ở đây đều cần **dữ liệu có nhãn**. Nếu không có nhãn thì sao? → **Module 4**.

---

## Module 4 — Building Unsupervised Learning Models

> *"...algorithms uncover patterns in data **without labeled examples**... **hierarchical clustering,
> k-means**, and advanced methods such as **DBSCAN and HDBSCAN**... **dimension reduction** algorithms like
> **PCA, t-SNE, and UMAP**... integrate them with **feature engineering**."*

**Đây không phải "module khó hơn" — nó là NHÁNH KHÁC HẲN.** Nó trả lời câu hỏi vàng ❸ ở vế "KHÔNG có đáp án".

Module này có **hai nửa**, phục vụ hai mục đích khác nhau:

### Nửa 1 — GOM CỤM (clustering): chia dữ liệu thành nhóm

| Thuật toán | Ý tưởng một dòng |
|---|---|
| 🔭 **K-means** | bạn **báo trước có k nhóm**, máy tìm k tâm cụm rồi kéo mỗi điểm về tâm gần nhất |
| 🔭 **Hierarchical clustering** | dựng **cây quan hệ** giữa các nhóm (nhóm nhỏ gộp thành nhóm lớn) — không cần báo trước số nhóm |
| 🔭 **DBSCAN** | gom theo **vùng đông đúc**; điểm lẻ loi bị gắn nhãn *"nhiễu"* — bắt được cụm hình thù bất kỳ |
| 🔭 **HDBSCAN** | bản nâng cấp của DBSCAN, chịu được các cụm có mật độ khác nhau |

### Nửa 2 — GIẢM CHIỀU (dimension reduction): bớt số cột đi

**Giảm chiều là gì?** Dữ liệu có 200 cột thì con người không nhìn nổi, và mô hình cũng dễ nhiễu.
Giảm chiều = **ép 200 cột xuống còn 2–10 cột** mà vẫn giữ tối đa thông tin quan trọng.

| Thuật toán | Ý tưởng một dòng |
|---|---|
| 🔭 **PCA** | giảm chiều kiểu **tuyến tính** — tìm vài hướng "trải rộng" nhất của dữ liệu và giữ lại |
| 🔭 **t-SNE** | ép xuống 2 chiều **để vẽ ra xem**, giữ tốt quan hệ gần-gũi cục bộ |
| 🔭 **UMAP** | như t-SNE nhưng nhanh hơn và thường giữ được bức tranh tổng thể tốt hơn |

💡 **VÍ DỤ ĐỜI THỰC — giảm chiều là gì:** mô tả một người bằng 200 số đo cơ thể thì không ai hình dung nổi.
Nén xuống 2 con số — **chiều cao** và **cân nặng** — thì mất chi tiết, nhưng **nhìn phát là hiểu**.
Đó chính là giảm chiều.

**Feature engineering** (đã gặp ở bài M1-01 mục 3.1): tạo/chọn đặc trưng tốt hơn cho mô hình.
Bản gốc nói rõ nó **đi kèm** hai nửa trên — vì gom cụm và giảm chiều thường là bước **chuẩn bị** cho mô hình chính.

---

## Module 5 — Evaluating and Validating Machine Learning Models

> *"...how to assess model performance **on unseen data**... key **evaluation metrics** for classification
> and regression... **hyperparameter tuning** to optimize models while **avoiding overfitting** using
> **cross-validation**... **regularization** in linear regression... to handle overfitting due to **outliers**."*

**Đây là module quan trọng nhất của cả khoá** — và nó trả lời đúng một câu hỏi:

> ## ❓ *"Mọi con số tôi tạo ra ở Module 2, 3, 4 — là THẬT hay là ẢO?"*

Bốn nhóm nội dung:

| Nội dung | Trả lời câu gì |
|---|---|
| **Evaluation metrics** (chỉ số đánh giá) | *"Chấm mô hình này bằng thước nào cho đúng?"* — accuracy, precision, recall, MAE, RMSE, R²… |
| 🔭 **Cross-validation** (kiểm chứng chéo) | *"Chia train/test một lần có thể ăn may. Làm sao chắc?"* → chia nhiều lần, xoay vòng, lấy trung bình |
| 🔭 **Hyperparameter tuning** | *"Các nút vặn của mô hình (k của KNN, độ sâu của cây) đặt bao nhiêu?"* — dò tìm có hệ thống |
| 🔭 **Regularization** | *"Mô hình học thuộc quá thì phanh lại kiểu gì?"* — phạt mô hình khi hệ số quá lớn |

⚠ **Nhận xét thẳng của tôi (góc người làm nghề):** module này **đứng cuối là hợp lý về mặt dạy** (phải có mô hình
rồi mới chấm được), **nhưng trong công việc thật thì bạn phải nghĩ tới nó TỪ NGÀY ĐẦU**. Người mới hay dựng
mô hình suốt 3 tuần rồi mới đi đánh giá — và phát hiện toàn bộ số liệu 3 tuần đó là số ảo.
👉 Đây chính là hạt nhân ❺ của bài M1-01: *điểm trên dữ liệu đã học không tính.*

---

## Module 6 — Final Exam and Project

> *"...review the course content, complete a **final exam**, and work on a hands-on project... a course summary
> **cheat sheet**, apply your skills in a project on **Rain Prediction in Australia**, and participate in
> **peer reviews**."*

**Gồm 4 thứ:** ôn tập + tờ tóm tắt (cheat sheet) · thi cuối khoá · **đồ án** · chấm chéo giữa học viên.

---

# PHẦN 5 — ĐỒ ÁN CUỐI KHOÁ: DỰ ĐOÁN MƯA Ở ÚC

Đây là chỗ hay nhất của bài hôm nay: **bạn đã đủ kiến thức để phân tích đề bài này rồi**, dù chưa học module nào.
Lấy hạt nhân bài M1-01 ra dùng thử:

| Câu hỏi | Áp hạt nhân nào | Trả lời |
|---|---|---|
| **Có cần ML không?** | ❶ | **Có.** Không ai viết tay nổi luật "khi nào trời mưa"; và ta có sẵn hàng chục năm dữ liệu thời tiết |
| **ML cổ điển hay deep learning?** | ❷ | **ML cổ điển.** Dữ liệu là bảng (nhiệt độ, độ ẩm, áp suất, gió…) — mở được bằng Excel |
| **Có đáp án không?** | ❸ | **Có** → *supervised.* Mỗi ngày trong lịch sử đều có cột "hôm sau có mưa hay không" |
| **Họ mô hình nào?** | ❹ | Viết đáp án mẫu: `"có mưa"` → **một nhãn** → **CLASSIFICATION** (nhị phân) |
| **Cẩn thận điều gì?** | ❺ | Phải chấm trên dữ liệu **chưa từng thấy** |

⚠ **Và đây là cái bẫy tôi muốn bạn để ý sẵn từ bây giờ:**

Nước Úc phần lớn **khô**. Nghĩa là trong dữ liệu, số ngày **không mưa** nhiều hơn hẳn số ngày mưa.
Đó chính là **dữ liệu mất cân bằng** — đúng cái bẫy đã học ở bài M1-01 mục 5.2-④ (phát hiện gian lận).

> Một mô hình chỉ việc **luôn trả lời "không mưa"** sẽ có accuracy rất cao và **hoàn toàn vô dụng**.
> Nên khi làm đồ án này, **accuracy là chỉ số phải nghi ngờ đầu tiên** — phải xem **recall**:
> *trong những ngày thật sự có mưa, mô hình bắt được bao nhiêu?*

🛠 **Việc cần làm khi tới đồ án:** đếm tỉ lệ ngày mưa / không mưa **trước khi** chấm bất kỳ con số nào.
Đây là bước người mới hay bỏ qua.

---

# TỔNG KẾT

## 🧠🧠 HẠT NHÂN — 4 CÂU. NHỚ 4 CÂU NÀY LÀ NẮM CẢ BẢN ĐỒ

> Mỗi câu gồm: **HIỂU** → **⚙ VẬN DỤNG** → **✏ THỬ NGAY**.

---

### ❶ Thứ tự các module là một CÂU CHUYỆN: mỗi module sinh ra để **vá lỗ hở của module trước**.

**HIỂU.** Không phải danh sách chủ đề xếp ngẫu nhiên:

```
M2 mô hình yếu     →  lỗ hở: chỉ vẽ được ĐƯỜNG THẲNG
M3 mô hình mạnh    →  vá lỗ đó, nhưng sinh lỗ mới: DỄ HỌC THUỘC (overfit)
M4 không cần nhãn  →  vá một lỗ khác hẳn: KHÔNG CÓ ĐÁP ÁN thì làm gì
M5 đánh giá        →  vá lỗ chí mạng: mọi số ở trên là THẬT hay ẢO?
```

**⚙ VẬN DỤNG — mỗi khi bắt đầu một module mới, hỏi đúng 2 câu:**

> **1. Module trước để lại lỗ hở gì?**
> **2. Module này vá lỗ đó bằng cách nào?**

Trả lời được 2 câu này thì bạn **nhớ được cả module mà không cần học thuộc**, vì mọi kỹ thuật trong đó
đều có **một lý do tồn tại** rõ ràng. Không trả lời được ⇒ bạn đang học thuộc danh sách thuật toán,
và sẽ quên sạch sau hai tuần.

Mẹo này dùng được **ngoài khoá học**: đọc tài liệu kỹ thuật nào cũng hỏi *"cái này sinh ra để chữa bệnh gì?"*

**✏ THỬ NGAY.**

| Câu hỏi | Đáp án |
|---|---|
| Vì sao Module 3 dạy decision tree **ngay sau** hồi quy tuyến tính? | Vì hồi quy tuyến tính chỉ vẽ được **ranh giới thẳng**; cây quyết định cắt được ranh giới **gấp khúc tuỳ ý** |
| Vì sao Module 3 phải dạy thêm **bias–variance và ensemble**, không chỉ dạy các thuật toán mạnh? | Vì mô hình càng mạnh càng **dễ học thuộc** (overfit) — dạy dao sắc thì phải dạy luôn cách cầm |
| Module 4 vá lỗ hở của Module 3 à? | **Không hẳn** — nó rẽ sang **nhánh khác**: khi dữ liệu **không có nhãn**. Đây là ngoại lệ của quy luật vá-lỗ |

---

### ❷ Đọc mục tiêu khoá học theo **ĐỘNG TỪ**: chỉ **2 trong 5** là "làm được".

**HIỂU.** *Describe · Articulate · Discuss* = **BIẾT** (3 mục tiêu) ·
*Implement · Solve* = **LÀM ĐƯỢC** (2 mục tiêu).
Và trong hai cái đó, **Solve đứng trên Implement**: *implement* = làm theo chỉ dẫn, *solve* = **tự quyết định**.

**⚙ VẬN DỤNG — dùng để tự chấm bản thân, không chờ ai chấm hộ.**

Sau mỗi module, tự hỏi theo thang này:

| Bạn làm được tới đâu | Mức | Đủ chưa |
|---|---|---|
| Nhắc lại được định nghĩa | 🔵 biết | **chưa đủ** |
| Đọc code có sẵn thì hiểu | 🔵 biết | **chưa đủ** |
| Đề bảo "dùng decision tree", bạn gõ ra được | 🟢 implement | tạm được |
| Đề chỉ nói "công ty mất khách", bạn **tự chọn** cách làm | 🟢 **solve** | ✅ **đây mới là đích** |

**✏ THỬ NGAY.** Bạn đang ở mức nào với kiến thức Module 1?
*(Gợi ý tự chấm: bạn đã đọc hiểu — mức "biết". Muốn lên "implement" thì phải tự gõ được một workflow
scikit-learn từ đầu tới cuối. Đó chính là bài tập số 5 của bài M1-01.)*

---

### ❸ Module 5 (đánh giá) **học cuối cùng, nhưng phải nghĩ tới từ ngày đầu**.

**HIỂU.** Đây là chỗ trình tự dạy học và trình tự làm việc **ngược nhau**:

```
TRÌNH TỰ DẠY (hợp lý):    dựng mô hình  ────────────────▶  rồi mới học cách chấm
TRÌNH TỰ LÀM VIỆC:        quyết định CÁCH CHẤM  ────────▶  rồi mới dựng mô hình
                          ↑
                     phải làm TRƯỚC, nếu không
                     mọi thứ dựng ra đều là số ảo
```

Nối thẳng với hạt nhân ❺ bài M1-01: *điểm trên dữ liệu đã học không tính.*

**⚙ VẬN DỤNG — quy tắc "chốt thước đo trước khi làm".**

Trước khi gõ dòng code mô hình đầu tiên, trả lời **3 câu**:

1. **Tôi sẽ chấm bằng chỉ số nào?** (accuracy? recall? MAE?) — và **vì sao chỉ số đó** hợp bài này?
2. **Tôi giấu bao nhiêu dữ liệu để chấm?** — chia ngay từ đầu, đừng chia sau.
3. **Bao nhiêu thì coi là ĐẠT?** — con số này phải chốt **trước khi nhìn kết quả**.

> **Câu 3 quan trọng nhất và hay bị bỏ nhất.** Nếu chốt sau, con người sẽ tự nhiên hạ chuẩn cho vừa
> kết quả mình vừa thấy — *"à 72% cũng ổn mà"*. Chốt trước thì không tự lừa được.

**✏ THỬ NGAY.** Bạn sắp làm đồ án dự đoán mưa ở Úc. Ba câu trên trả lời thế nào?

| | Trả lời mẫu |
|---|---|
| 1. Chỉ số | **Recall** (và precision) — **không** dùng accuracy, vì dữ liệu mất cân bằng |
| 2. Giấu bao nhiêu | 20% dữ liệu, tách ngay từ đầu, không đụng tới cho tới lần chấm cuối |
| 3. Ngưỡng đạt | phải **chốt trước**, ví dụ *"bắt được ít nhất 70% số ngày thật sự có mưa"* |

---

### ❹ **Bias–variance tradeoff** — sự đánh đổi có mặt trong **MỌI** mô hình, không trừ cái nào.

**HIỂU.** Bản gốc nói rõ nó *"inherent to all learning models"* — **cố hữu ở mọi mô hình**. Đây là hai kiểu sai
hoàn toàn khác nhau:

| | **BIAS cao** (thiên lệch) | **VARIANCE cao** (phương sai) |
|---|---|---|
| Mô hình thế nào | **quá đơn giản** | **quá phức tạp** |
| Nó bỏ sót gì | bỏ sót cả quy luật chính | học thuộc luôn cả nhiễu |
| Tên bệnh | **underfitting** (chưa khớp) | **overfitting** (quá khớp) |
| Dấu hiệu nhận ra | sai **cả** trên dữ liệu học **lẫn** dữ liệu mới | đúng trên dữ liệu học, **sai trên dữ liệu mới** |

💡 **VÍ DỤ ĐỜI THỰC — tập bắn bia:**

```
BIAS cao                    VARIANCE cao              TỐT
(chụm nhưng lệch tâm)       (quanh tâm nhưng tản)     (chụm và trúng tâm)

      ╭───────╮                 ╭───────╮              ╭───────╮
      │   ●●  │                 │ ●   ● │              │       │
      │   ●●  │                 │   ⊕   │              │  ●●●  │
      │   ⊕   │                 │ ●   ● │              │  ●⊕●  │
      ╰───────╯                 ╰───────╯              ╰───────╯
  ngắm sai chỗ,             ngắm đúng chỗ,        vừa đúng vừa ổn định
  nhưng tay rất vững        nhưng tay run
```

**Đánh đổi:** làm mô hình phức tạp hơn → **bias giảm** nhưng **variance tăng**. Không có cách nào giảm cả hai
xuống 0 — chỉ có tìm **điểm cân bằng**.

**⚙ VẬN DỤNG — chẩn bệnh bằng cách so HAI con số.**

Đây là thao tác bạn sẽ làm suốt đời làm nghề. Chấm mô hình **hai lần** — trên dữ liệu học và trên dữ liệu chưa
từng thấy — rồi đọc bảng:

| Điểm trên dữ liệu **đã học** | Điểm trên dữ liệu **chưa thấy** | Chẩn đoán | Thuốc |
|---|---|---|---|
| **thấp** (60%) | **thấp** (58%) | **BIAS cao** — underfitting | mô hình **mạnh hơn**, thêm đặc trưng |
| **cao** (99%) | **thấp** (65%) | **VARIANCE cao** — overfitting | đơn giản bớt · thêm dữ liệu · **regularization** · **ensemble** |
| **cao** (92%) | **cao** (89%) | ✅ **khoẻ mạnh** | giữ nguyên |
| thấp | **cao hơn hẳn** | 🚨 **sai ở đâu đó** — chia dữ liệu sai, hoặc rò rỉ dữ liệu | đi kiểm lại quy trình |

> **Chỉ cần nhìn KHOẢNG CÁCH giữa hai con số.** Khoảng cách lớn = variance cao. Hai số cùng thấp = bias cao.
> Đơn giản vậy thôi, và nó đúng cho **mọi** mô hình bạn sẽ gặp.

**✏ THỬ NGAY.** Ba mô hình, chẩn bệnh nào?

| Kết quả | Chẩn đoán |
|---|---|
| a) Học 98% · Mới 61% | **Overfitting** (variance cao) — chênh 37 điểm. Đơn giản bớt hoặc thêm dữ liệu |
| b) Học 64% · Mới 62% | **Underfitting** (bias cao) — hai số cùng thấp, mô hình quá yếu |
| c) Học 91% · Mới 88% | **Khoẻ** — chênh ít, cả hai đều cao |

---

## 🧠 BỔ TRỢ — 5 câu

5. Khoá này là **bản lề** của hai lộ trình: Data Science và AI Engineering.
6. **Toán cấp 3 là lợi thế, không bắt buộc** — chỉ cần đường thẳng, toạ độ, trung bình, đồ thị hàm số.
7. Module 2 dạy hai mô hình **yếu, cổ (thế kỷ 19)** — cố ý, để bạn tự thấy giới hạn của chúng.
8. Module 4 (không nhãn) là **nhánh rẽ**, không phải bậc cao hơn Module 3.
9. Khoá học trong **4–6 tuần**, kết bằng **thi + đồ án dự đoán mưa ở Úc + chấm chéo**.

## ⚠ BA CÁI BẪY CỦA BÀI NÀY

| Bẫy | Sự thật |
|---|---|
| "Module 4 khó hơn Module 3" | **KHÔNG** — nó là **nhánh khác** (không có nhãn), không phải bậc cao hơn |
| "Học đánh giá (M5) sau cùng cũng được" | **KHÔNG** — phải chốt thước đo **trước** khi dựng mô hình, nếu không mọi số đều ảo |
| "Đồ án dự đoán mưa: accuracy cao là ngon" | **KHÔNG** — Úc khô, dữ liệu mất cân bằng, accuracy là chỉ số lừa đảo. Nhìn **recall** |

## 🛠 CẦN HỌC — 3 kỹ năng

1. **Đọc bản đồ theo lỗ hở:** cho một chương trình học/tài liệu bất kỳ → nói được mỗi phần **vá lỗ hở gì**
   của phần trước.
2. **Chốt thước đo trước khi làm:** trước khi dựng bất kỳ mô hình nào → trả lời được 3 câu
   *(chỉ số nào · giấu bao nhiêu dữ liệu · ngưỡng đạt là bao nhiêu)*.
3. **Chẩn bias/variance:** cho hai con số (điểm trên dữ liệu đã học · điểm trên dữ liệu mới) → gọi đúng bệnh
   và kê đúng thuốc.

---

# TỪ ĐIỂN TRA NHANH — các thuật toán bài này chỉ NÊU TÊN

> Bảng này để **tra**, không phải để học thuộc. Mỗi cái sẽ có bài riêng.
> Việc của bạn bây giờ: nhìn tên **biết nó nằm ở module nào và giải bài toán gì**.

| Tên | Module | Nhóm | Một dòng |
|---|---|---|---|
| Linear regression | M2 | hồi quy | vẽ đường thẳng khớp dữ liệu, trả về **một con số** |
| Logistic regression | M2 | **phân loại** | trả về xác suất rồi cắt ngưỡng → **một nhãn** (⚠ tên gây nhầm) |
| Polynomial regression | M2 | hồi quy | dùng hồi quy tuyến tính để khớp đường **cong** |
| Decision tree | M3 | phân loại | lưu đồ hỏi–đáp, mỗi lá là một kết luận |
| Regression tree | M3 | hồi quy | cùng cấu trúc cây, lá trả về **một con số** |
| KNN | M3 | cả hai | hỏi **k hàng xóm gần nhất** rồi theo số đông |
| SVM | M3 | phân loại | kẻ ranh giới có **lề rộng nhất**; nâng chiều được (kernel) |
| Ensemble (bagging/boosting) | M3 | cả hai | **ghép nhiều mô hình** để giảm sai — trị bias/variance |
| K-means | M4 | gom cụm | báo trước **k nhóm**, máy tìm tâm cụm |
| Hierarchical clustering | M4 | gom cụm | dựng **cây quan hệ** nhóm, không cần báo trước số nhóm |
| DBSCAN | M4 | gom cụm | gom theo **vùng đông đúc**, điểm lẻ = nhiễu |
| HDBSCAN | M4 | gom cụm | DBSCAN nâng cấp, chịu được mật độ khác nhau |
| PCA | M4 | giảm chiều | nén cột theo kiểu **tuyến tính**, giữ hướng trải rộng nhất |
| t-SNE | M4 | giảm chiều | ép xuống 2 chiều **để vẽ ra nhìn** |
| UMAP | M4 | giảm chiều | như t-SNE, nhanh hơn, giữ bức tranh tổng thể tốt hơn |
| Cross-validation | M5 | đánh giá | chia train/test **nhiều lần xoay vòng** rồi lấy trung bình |
| Hyperparameter tuning | M5 | đánh giá | dò tìm giá trị tốt nhất cho các **nút vặn** của mô hình |
| Regularization | M5 | chống overfit | **phạt** mô hình khi hệ số quá lớn → buộc nó đơn giản lại |

---

# BÀI TẬP

**Bài 1 — đọc bản đồ theo lỗ hở** *(kỹ năng 🛠 #1)*
Module 2 để lại **ba** giới hạn (xem lại Phần 4, Module 2). Chọn **một** giới hạn và chỉ ra
**thuật toán nào ở Module 3** vá đúng giới hạn đó, giải thích ngắn vì sao.

**Bài 2 — chẩn bias/variance** *(kỹ năng 🛠 #3)*
Bạn dựng mô hình dự đoán mưa. Kết quả: **dữ liệu đã học 96% · dữ liệu chưa thấy 68%.**
a) Bệnh gì? b) Kê hai thuốc.

**Bài 3 — chốt thước đo trước** *(kỹ năng 🛠 #2, quan trọng nhất)*
Sếp giao: *"Dựng mô hình phát hiện đơn hàng có nguy cơ bị hoàn trả."*
Trước khi gõ dòng code nào, bạn phải trả lời 3 câu gì? Trả lời luôn cả ba cho bài toán này.

**Bài 4 — áp hạt nhân bài trước** *(ôn tập M1-01)*
Đồ án cuối khoá là *"Rain Prediction in Australia"*. Không nhìn Phần 5, tự trả lời:
a) Supervised hay unsupervised? b) Classification hay regression? c) Vì sao **không** được tin accuracy?

**Bài 5 — nhìn bản đồ** *(dễ)*
Trong 6 module, module nào bạn nghĩ sẽ **khó nhất với bản thân**, và vì sao?
*(Không có đáp án đúng/sai — câu này để tôi biết cần chuẩn bị kỹ chỗ nào cho bạn.)*

---

# HỎI & ĐÁP

> Bạn hỏi gì, tôi ghi câu trả lời vào đây kèm ngày, để lần sau đọc lại file là đủ.

*(chưa có câu hỏi nào)*

---

## PHỤ LỤC — tra lại nguồn

Mọi định nghĩa thuật toán trong bài lấy từ sổ tri thức đã kiểm chứng:

```bash
cd /home/hai-soft/projects/icpp/sleep
python3 rail.py q "Bias-variance"
python3 rail.py q "DBSCAN"
python3 rail.py q "PCA"
python3 rail.py q "Cross-validation"
python3 rail.py q "Regularization"
```

**Đã kiểm trước khi viết (2026-08-14):** file bạn đưa là tài liệu **MỚI**, chưa từng có trong `lectures/`
(khác 8 file transcript của m1) — đã lưu vào `lectures/Machine Learning with Python/m1/m1 - 00 - Course
Overview (reading).txt`. Mọi thuật toán nó nêu tên **đều đã có trong sổ và đã verify**; thứ duy nhất mới là
**đồ án cuối khoá "Rain Prediction in Australia"**.

**Nhận xét riêng đã ghi rõ trong bài** (không phải nội dung bản gốc): mục Module 5 — *đánh giá dạy cuối
nhưng phải nghĩ tới từ đầu*; mục Phần 5 — *cảnh báo mất cân bằng dữ liệu của đồ án mưa* (suy ra từ khí hậu Úc,
**cần đếm lại tỉ lệ thật khi tới đồ án**, tôi chưa đo bộ dữ liệu đó).

## LỊCH SỬ CẬP NHẬT

| Ngày | Thay đổi |
|---|---|
| 2026-08-14 | Soạn bài. Viết theo kernel BỔ SUNG 10 (điều 4b + 4c): 4 câu hạt nhân, mỗi câu có HIỂU → ⚙ VẬN DỤNG (bảng quyết định) → ✏ THỬ NGAY (ca cụ thể + đáp án). |
