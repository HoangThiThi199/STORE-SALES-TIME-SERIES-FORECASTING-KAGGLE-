# Store Sales — Time Series Forecasting

Dự án học máy dự báo doanh số của Corporación Favorita cho cuộc thi Kaggle [Store Sales - Time Series Forecasting](https://www.kaggle.com/competitions/store-sales-time-series-forecasting).

Bài toán dự báo doanh số (`sales`) theo ngày cho 54 cửa hàng × 33 nhóm sản phẩm (1.782 chuỗi Store-Family), trong 16 ngày. Metric Kaggle là RMSLE. Yêu cầu học phần gồm ít nhất một mô hình non-DL, một mô hình DL-based, giải thích XAI cho ít nhất một mô hình, mã/notebook tái chạy được và báo cáo trình bày.

## Cấu trúc dự án

```text
.
├── data/
│   ├── raw/                 # 7 CSV gốc từ Kaggle
│   └── DATA_GUIDE.md        # Mô tả dữ liệu và lưu ý
├── notebooks/               # Notebook theo từng giai đoạn
├── reports/                 # Báo cáo phân tích và kết quả
├── models/                  # Model artifacts (không commit)
├── submissions/             # Kaggle submission files (không commit)
└── requirements.txt         # Dependency cho toàn dự án
```

## Cài đặt môi trường

Mở thư mục repo trong PowerShell. Tạo môi trường ảo và cài các thư viện từ `requirements.txt`:

```powershell
if (-not (Test-Path .venv\Scripts\python.exe)) {
    python -m venv .venv
}
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

Trong VS Code, mở notebook cần chạy, bấm **Select Kernel**, rồi chọn Python trong `.venv` (`.venv\Scripts\python.exe`). `requirements.txt` bao gồm các thư viện dữ liệu, EDA, notebook, mô hình boosting, DL/time-series, XAI và tuning. Nhóm DL/time-series có thể tải về dung lượng lớn.

Nếu cài đặt lỗi do Python hoặc hệ điều hành chưa được một package hỗ trợ, hãy tạo `.venv` bằng một phiên bản Python được các package đó hỗ trợ rồi cài lại. Không cài package vào Python toàn hệ thống.

## Notebook 01 — Data Understanding

Notebook đầu tiên là [`notebooks/01_data_understanding.ipynb`](notebooks/01_data_understanding.ipynb). Nó kiểm tra dữ liệu thô và tạo các báo cáo trong `reports/understanding/`; không làm cleaning, merge modeling table, feature engineering hay train model.

Chạy trong VS Code/Jupyter bằng **Run All**. Hoặc kiểm tra notebook từ PowerShell ở thư mục repo:

```powershell
.\.venv\Scripts\python.exe -m nbconvert --to notebook --execute notebooks\01_data_understanding.ipynb --output 01_data_understanding_executed.ipynb --output-dir tmp\notebook_check --ExecutePreprocessor.timeout=600
```

Các kết quả chính cần thấy:

- Train: 3.000.888 dòng, từ 2013-01-01 đến 2017-08-15.
- Test: 28.512 dòng, từ 2017-08-16 đến 2017-08-31.
- 54 cửa hàng, 33 nhóm sản phẩm, 1.782 chuỗi Store-Family.
- 43 giá trị thiếu ở `oil.dcoilwtico`; chưa được tự động điền.

## Phase 2 — EDA và Preprocessing

Thực hiện theo thứ tự; hãy xem kết quả EDA trước khi chốt các quyết định xử lý dữ liệu.

1. Mở [`notebooks/02_eda.ipynb`](notebooks/02_eda.ipynb), chọn kernel `.venv` và tự bấm **Run All**. Notebook phân tích sales, trend/seasonality, Store/Family/series, promotion, transactions, oil, holidays và zero sales. Notebook chỉ trực quan hóa, không sửa dữ liệu. Các bảng tổng hợp được lưu trong `reports/eda/`.
2. Sau khi xem EDA, mở [`notebooks/03_data_preprocessing.ipynb`](notebooks/03_data_preprocessing.ipynb) và tự bấm **Run All**. Notebook kiểm tra keys, ghép metadata cửa hàng, giá dầu và holiday events đã tổng hợp theo Date-Store. Nó lưu `train_processed.csv`, `test_processed.csv` và `transactions_history.csv` trong `data/processed/`, cùng các integrity reports trong `reports/preprocessing/`.

Preprocessing giữ missing oil, không tạo lại các ngày 25/12 bị thiếu, và để transactions lịch sử riêng để tránh dùng số giao dịch cùng ngày làm feature dự báo sales. Các quyết định này được ghi trong notebook; dữ liệu trong `data/raw/` không bị ghi đè. File processed có thể lớn và notebook có thể cần thêm RAM khi tạo dữ liệu dẫn xuất.

## Phase 3 - Validation Framework & Baseline

[`notebooks/04_validation_baseline.ipynb`](notebooks/04_validation_baseline.ipynb) defines the fixed folds and integrity checks, then evaluates Seasonal Naive and Global LightGBM with recursive 16-day forecasts. It compares pooled OOF RMSLE and reports performance by horizon and family. The recorded reference scores are 0.573874 for Seasonal Naive and 0.472009 for Global LightGBM (17.75% pooled improvement). All later models use the same fixed fold dates.

## Phase 4 - Feature Engineering

[`notebooks/05_feature_engineering.ipynb`](notebooks/05_feature_engineering.ipynb) contains the full Stage 2 lag experiment pipeline: E1-E4, recursive validation, pooled/fold comparisons, family and horizon analysis, normalized feature importance, and final review. E1 has saved outputs; run E2, E3, and E4 individually after reviewing prior results. Avoid Run All because each experiment trains three LightGBM models. Keep the validation folds, model parameters, target transform, and recursive protocol fixed during feature ablations.

## Nguyên tắc thực nghiệm

- Thiết kế validation thời gian mô phỏng đúng horizon 16 ngày và đánh giá bằng RMSLE.
- Tính lag/rolling riêng cho từng Store-Family; không dùng `sales` thực tế của validation để dự báo các ngày validation tiếp theo.
- Không dùng giao dịch thực tế tương lai: `transactions.csv` chỉ có dữ liệu lịch sử tới cuối train.
- Giữ nguyên `id` của `test.csv` khi tạo submission gồm `id` và `sales` không âm.
- Coi zero sales là một giá trị quan sát, không mặc định là missing hay lỗi.

## Dữ liệu và artifacts

Không commit dữ liệu CSV gốc, môi trường `.venv`, model artifacts hay submissions. Các thư mục `data/`, `models/`, `submissions/` và môi trường được bỏ qua trong `.gitignore`.
