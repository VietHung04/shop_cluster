# README – Phân khúc khách hàng dựa trên Luật kết hợp & RFM
## 👥 Thông tin Nhóm

- **Nhóm:** Nhóm 5
- **Thành viên:**
  - Nguyễn Nam Cường
  - Nguyễn Văn Đạt
  - Trần Trường Giang
## 🎯 Mục tiêu

Mini Project này nhằm xây dựng một **pipeline phân khúc khách hàng hoàn chỉnh** theo hướng:

**Luật kết hợp → Đặc trưng hành vi mua kèm → Phân cụm khách hàng → Diễn giải → Đề xuất chiến lược marketing**.

Cụ thể, nhóm hướng tới các mục tiêu sau:

* Khai phá các **mối quan hệ mua kèm (association rules)** có ý nghĩa từ dữ liệu giao dịch.
* Biến các luật kết hợp thành **đặc trưng hành vi** cho từng khách hàng.
* So sánh **nhiều biến thể feature engineering** (baseline vs nâng cao).
* Đánh giá và trực quan hóa kết quả phân cụm bằng các chỉ số và PCA 2D.
* Thực hiện **profiling và diễn giải cụm** gắn với giá trị kinh doanh.
* Đề xuất **chiến lược marketing cụ thể** cho từng nhóm khách hàng.

---

## 1. Khai phá luật kết hợp (Association Rule Mining)

### 1.1 Phương pháp

Nhóm sử dụng thuật toán **FP-Growth** để khai phá luật kết hợp từ bộ dữ liệu **Online Retail**. Dữ liệu giao dịch được tiền xử lý và chuyển sang dạng **basket format** trước khi sinh luật.

Tổng số luật sinh ra ban đầu: **3,856 luật**.

### 1.2 Quy trình lọc luật

Để đảm bảo chất lượng và khả năng diễn giải, nhóm áp dụng các tiêu chí lọc:

* **min_support**: loại bỏ các luật xuất hiện quá ít.
* **min_confidence**: đảm bảo xác suất xảy ra consequent đủ lớn khi antecedent xuất hiện.
* **min_lift > 1**: chỉ giữ các luật có mối quan hệ mua kèm có ý nghĩa.
* Giới hạn độ dài antecedent/consequent để tránh luật quá phức tạp.

Sau lọc, số luật giảm từ **3,856 → 1,794 luật**, giữ lại phần lớn các quan hệ mua kèm quan trọng.

### 1.3 Tiêu chí lựa chọn luật cho phân cụm

Từ tập luật đã lọc, nhóm:

* Sắp xếp theo **lift giảm dần**.
* Chọn **Top-100 luật** làm đầu vào cho phân cụm.

**Lý do lựa chọn lift & Top-100:**

* Lift phản ánh độ mạnh thực sự của mối quan hệ mua kèm.
* Tránh đưa quá nhiều luật gây nhiễu và tăng số chiều không cần thiết.
* Đảm bảo cân bằng giữa **độ phong phú hành vi** và **khả năng diễn giải**.

### 1.4 Các luật tiêu biểu

| Antecedents                               | Consequents          | Support | Confidence | Lift  |
| ----------------------------------------- | -------------------- | ------- | ---------- | ----- |
| HERB MARKER PARSLEY, HERB MARKER ROSEMARY | HERB MARKER THYME    | 0.0109  | 0.9517     | 74.57 |
| HERB MARKER MINT, HERB MARKER THYME       | HERB MARKER ROSEMARY | 0.0106  | 0.9550     | 74.50 |
| HERB MARKER BASIL, HERB MARKER THYME      | HERB MARKER ROSEMARY | 0.0107  | 0.9507     | 74.17 |

👉 Các luật đều có **lift rất cao**, cho thấy mối liên kết mua kèm mạnh mẽ giữa các sản phẩm cùng dòng.

---



## 2. Feature Engineering cho phân cụm

Nhóm xây dựng **hai biến thể đặc trưng** để so sánh.

### 2.1 Biến thể 1 – Baseline (Rule-based Binary Features)

**Cấu hình:**

```bash
RULE_FEATURE_TYPE=binary   # chỉ đánh dấu có / không thỏa luật
TOP_K_RULES=200
USE_RFM=false
RFM_SCALE=false
RULE_SCALE=false
MIN_ANTECEDENT_LEN=2
```

**Không gian đặc trưng:**

* Shape X: **(3921 × 175)**
* Chỉ sử dụng rule-features dạng nhị phân

**Kết quả phân cụm:**

* Silhouette cao nhất tại **k = 2**, score ≈ **0.56**
* Phân tách được nhóm mua nhiều và mua ít, nhưng mức độ chưa rõ ràng

---

### 2.2 Biến thể 2 – Rule + RFM (Weighted Features)

Đây là **biến thể được lựa chọn chính thức** cho các bước phân tích tiếp theo.

**Cấu hình:**

```bash
RULE_FEATURE_TYPE=weighted  # lift × confidence
TOP_K_RULES=200
USE_RFM=true
RFM_SCALE=true
RULE_SCALE=false
MIN_ANTECEDENT_LEN=2
```

**Không gian đặc trưng:**

* Shape X: **(3921 × 203)**
* Rule-features có trọng số + RFM chuẩn hóa
  
**Kết quả phân cụm:**

* Silhouette cao nhất tại **k = 2**, score ≈ **0.96**
* Phân tách được nhóm mua nhiều và mua ít, phân cụm rõ ràng
  
**Ưu điểm:**

* Giữ được cường độ hành vi mua kèm (thông qua lift & confidence)
* Kết hợp giá trị khách hàng (RFM) → tăng khả năng diễn giải
* Phù hợp cho profiling & marketing action

---

## 3. Lựa chọn số cụm K (Sử dụng biến thể 2 – Rule + RFM)

Nhóm khảo sát K từ **2 → 10** bằng **Silhouette score** trên **biến thể 2 (Rule + RFM)**.

**Kết quả:**

* Silhouette tối đa ≈ **0.96** tại **k = 2**
* Các giá trị k lớn hơn cho silhouette giảm mạnh

👉 **Chọn k = 2** vì:

* Silhouette cao vượt trội
* Cụm tách biệt rõ ràng trên PCA
* Các cụm có **ý nghĩa hành động marketing rõ ràng** (VIP vs Mass)

---

## 4. Trực quan hóa & đánh giá cụm

* Phương pháp giảm chiều: **PCA 2D**
* Mỗi điểm: một khách hàng
* Màu sắc: nhãn cụm

**Nhận xét:**

* Hai cụm được tách biệt rất rõ trên trục Component 1.
* Cụm nhỏ nằm xa trung tâm → nhóm khách hàng giá trị cao.
* Phù hợp với profiling RFM.

---

## 5. So sánh có hệ thống các biến thể

| Tiêu chí            | Baseline     | Rule + RFM   |
| ------------------- | ------------ | ------------ |
| Rule-features       | Nhị phân     | Weighted     |
| RFM                 | ❌            | ✅            |
| Shape X             | (3921 × 175) | (3921 × 203) |
| K tối ưu            | 2            | 2            |
| Silhouette max      | ~0.56        | ~0.96 ⭐      |
| PCA separation      | Vừa          | Rất rõ       |
| Marketing usability | Thấp         | Cao hơn      |

👉 Biến thể **Rule + RFM** vượt trội về chất lượng và khả năng diễn giải.

---

## 6. Profiling & diễn giải cụm (Quan trọng nhất)

### 6.1 Thống kê RFM theo cụm

| Cluster | Số KH | Recency | Frequency | Monetary |
| ------- | ----- | ------- | --------- | -------- |
| 0       | 3797  | 93.2    | 4.05      | 1,809.8  |
| 1       | 124   | 60.5    | 21.3      | 17,365.5 |

→ Phân hóa giá trị rất rõ ràng.

### 6.2 Đặt tên & Persona

| Cluster | English name               | Vietnamese name                    |
| ------- | -------------------------- | ---------------------------------- |
| 0       | Occasional Shoppers        | Khách mua lẻ, không thường xuyên   |
| 1       | High-Value Loyal Customers | Khách hàng trung thành giá trị cao |

**Persona:**

* Cluster 0: Khách mua không thường xuyên, chủ yếu mua quà tặng đơn giản.
* Cluster 1: Khách hàng VIP, mua thường xuyên, giá trị rất cao.

### 6.3 Chiến lược marketing

**Cluster 0 – Bundle & Cross-sell:**

* Gợi ý sản phẩm cùng bộ / cùng màu
* Combo quà tặng, Buy-more-save-more

**Cluster 1 – VIP Care & Upsell:**

* Gắn nhãn VIP
* Upsell full-set, limited edition
* Chăm sóc riêng, ưu đãi độc quyền

---
## 7.Dashboard Streamlit

<img width="1919" height="909" alt="image" src="https://github.com/user-attachments/assets/68dd576b-1a5e-4c9e-aec0-bdca64ec09da" />


## Project Structure
```text
SHOP_CLUSTER/
├── data/
│   ├── raw/
│   │   └── online_retail.csv
│   └── processed/
│       ├── basket_bool.parquet
│       ├── cleaned_uk_data.csv
│       ├── customer_clusters_from_rules.csv
│       ├── rules_apriori_filtered.csv
│       └── rules_fpgrowth_filtered.csv
│
├── notebooks/
│   ├── preprocessing_and_eda.ipynb
│   ├── basket_preparation.ipynb
│   ├── apriori_modelling.ipynb
│   ├── fp_growth_modelling.ipynb
│   ├── compare_apriori_fpgrowth.ipynb
│   ├── clustering_from_rules.ipynb
│   └── runs/
│       ├── preprocessing_and_eda_run.ipynb
│       ├── basket_preparation_run.ipynb
│       ├── apriori_modelling_run.ipynb
│       ├── fp_growth_modelling_run.ipynb
│       ├── compare_apriori_fpgrowth_run.ipynb
│       ├── clustering_from_rules_run.ipynb
│       └── visualize_rules.py
│
├── src/
│   ├── cluster_library.py
│   └── __pycache__/
│       ├── apriori_library.cpython-311.pyc
│       ├── apriori_library.cpython-312.pyc
│       ├── cluster_library.cpython-310.pyc
│       └── cluster_library.cpython-311.pyc
│── Dashboard.py
├── run_papermill.py
├── requirements.txt
├── README.md
├── LICENSE.txt
└── gitignore.txt
```

## Tech Stack

- Python, Pandas, MLxtend (Apriori/FP-Growth)  
- Matplotlib, Seaborn, Plotly (visualization)  
- Streamlit (dashboard)  
- Papermill (pipeline tự động)  
- Jupyter Notebook

---

## 🔚 Kết luận

Pipeline đề xuất không chỉ đạt hiệu quả kỹ thuật cao mà còn mang **ý nghĩa kinh doanh rõ ràng**. Việc kết hợp **luật kết hợp + RFM** giúp phát hiện và khai thác hiệu quả các nhóm khách hàng giá trị, hỗ trợ ra quyết định marketing có mục tiêu và hành động được.

### Author
Project được thực hiện bởi:
Trang Le

📄 License
MIT — sử dụng tự do cho nghiên cứu, học thuật và ứng dụng nội bộ.
