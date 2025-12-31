# Beijing Multi-Site Air Quality — Classification + Regression + Time Series (ARIMA)

Phân tích dữ liệu chất lượng không khí **Beijing Multi-Site Air Quality (12 stations)** để xây dựng một pipeline hoàn chỉnh gồm:

- **Phân lớp mức độ ô nhiễm (AQI level)**: tạo nhãn từ **PM2.5 rolling 24h**, nhưng **KHÔNG dùng PM2.5** trong tập đặc trưng đầu vào (tránh leakage).
- **Hồi quy (Regression)**: dự đoán **PM2.5 tương lai** theo horizon (ví dụ t+1, t+24…).
- **Chuỗi thời gian (Time Series)**: phân tích đặc điểm dữ liệu time series “đúng bài giảng” và dự báo **chỉ dùng ARIMA** (statsmodels).

Project triển khai theo pipeline notebook → module hoá trong `src/` → tự động chạy bằng **Papermill** để phục vụ giảng dạy & demo ra quyết định chọn mô hình.

---

## Features

### 1) Classification (No PM2.5 in features)
- Load & merge dữ liệu từ nhiều trạm
- Làm sạch dữ liệu: missing, kiểu thời gian, chuẩn hoá numeric/object
- Tạo nhãn **AQI class** từ `pm25_24h` (rolling mean 24h)
- **Không dùng PM2.5 / pm25_24h làm feature**
- Đánh giá: Accuracy, Precision/Recall/F1, Confusion Matrix
- Lưu artifacts: metrics + prediction sample

### 2) Regression (Supervised)
- Tạo bài toán hồi quy theo time-based split (tránh leakage)
- Feature engineering cho hồi quy:
  - time features (hour/day/month/…)
  - lag features (theo cấu hình)
- Dự đoán `PM2.5(t + horizon)`
- Đánh giá: RMSE, MAE, R2
- Lưu artifacts: model + metrics + prediction sample

### 3) Time Series Forecasting (ARIMA only)
- Xây dựng chuỗi đơn biến theo **1 trạm** (univariate PM2.5)
- Phân tích đặc điểm dữ liệu chuỗi thời gian “đúng bài giảng”:
  - missingness & resampling
  - rolling mean/std
  - stationarity tests (ADF/KPSS)
  - ACF/PACF để định hướng p,q
  - quyết định d (sai phân) theo kiểm định + quan sát
- Fit & chọn ARIMA theo AIC/BIC (grid nhỏ)
- Dự báo + lưu artifacts: summary, predictions, model

---

## Project Structure

```text
air_quality_timeseries/
├── data/
│   ├── raw/
│   │   └── PRSA2017_Data_20130301-20170228.zip
│   └── processed/
│       ├── cleaned.parquet
│       ├── dataset_for_clf.parquet
│       ├── metrics.json
│       ├── predictions_sample.csv
│       ├── dataset_for_regression.parquet
│       ├── regressor.joblib
│       ├── regression_metrics.json
│       ├── regression_predictions_sample.csv
│       ├── arima_pm25_summary.json
│       ├── arima_pm25_predictions.csv
│       └── arima_pm25_model.pkl
│
├── notebooks/
│   ├── preprocessing_and_eda.ipynb
│   ├── feature_preparation.ipynb
│   ├── classification_modelling.ipynb
│   ├── regression_modelling.ipynb
│   ├── arima_forecasting.ipynb
│   └── runs/
│       ├── preprocessing_and_eda_run.ipynb
│       ├── feature_preparation_run.ipynb
│       ├── classification_modelling_run.ipynb
│       ├── regression_modelling_run.ipynb
│       └── arima_forecasting_run.ipynb
│
├── src/
│   ├── classification_library.py
│   ├── regression_library.py
│   ├── timeseries_library.py
│   └── __init__.py
│
├── run_papermill.py
├── requirements.txt
└── README.md

```

## Installation

```bash
git clone <your_repo_url>
cd air_quality_timeseries
pip install -r requirements.txt
```

## Data Preparation

Đặt file gốc vào:
```

```bash
data/raw/PRSA2017_Data_20130301-20170228.zip
```
Hoặc tải dataset Beijing Multi-Site Air Quality Data (UCI) và đặt các file trạm vào:

```bash 
data/raw/
```
Ví dụ

```bash
data/raw/station_01.csv
data/raw/station_02.csv
...
data/raw/station_12.csv
```

File output sẽ được sinh tự động vào:
```bash
data/processed/
```



Run Pipeline (Recommended)
Chạy toàn bộ phân tích chỉ với 1 lệnh:

```bash
python run_papermill.py
```
Kết quả sinh ra:

```bash
data/processed/cleaned.parquet
data/processed/dataset_for_clf.parquet
data/processed/metrics.json
data/processed/predictions_sample.csv

data/processed/dataset_for_regression.parquet
data/processed/regressor.joblib
data/processed/regression_metrics.json
data/processed/regression_predictions_sample.csv

data/processed/arima_pm25_summary.json
data/processed/arima_pm25_predictions.csv
data/processed/arima_pm25_model.pkl

notebooks/runs/arima_forecasting_run.ipynb
```

### Changing Parameters
Các tham số có thể chỉnh trong run_papermill.py:

#### Preprocessing/EDA
```python
USE_UCIMLREPO = False
RAW_ZIP_PATH = "data/raw/PRSA2017_Data_20130301-20170228.zip"
LAG_HOURS = [1, 3, 24]
```

#### Classification
```python
CUTOFF = "2017-01-01"   # time-based split
# (PM2.5 bị loại khỏi features trong library để tránh leakage)
```

#### Regression
```python
HORIZON = 1                       # dự đoán PM2.5(t + HORIZON)
TARGET_COL = "PM2.5"
OUTPUT_REG_DATASET_PATH = "data/processed/dataset_for_regression.parquet"
CUTOFF = "2017-01-01"
MODEL_OUT = "regressor.joblib"
METRICS_OUT = "regression_metrics.json"
PRED_SAMPLE_OUT = "regression_predictions_sample.csv"
```

#### ARIMA 
```
STATION = "Aotizhongxin"
VALUE_COL = "PM2.5"
CUTOFF = "2017-01-01"

P_MAX = 3
Q_MAX = 3
D_MAX = 2
IC = "aic"                         # hoặc "bic"
ARTIFACTS_PREFIX = "arima_pm25"
```


Hoặc sửa trong cell PARAMETERS của mỗi notebook để chạy với cấu hình khác nhau.

### Visualization & Results

Notebook preprocessing_and_eda.ipynb:

  kiểm tra missingness, phân phối, xu hướng theo thời gian

  gợi ý seasonality (24h, tuần) để định hướng mô hình

Notebook regression_modelling.ipynb:

  dự đoán PM2.5(t+h), đánh giá RMSE/MAE/R2, minh hoạ leakage và lý do time-split

Notebook arima_forecasting.ipynb:

  ADF/KPSS, rolling mean/std, ACF/PACF

  chọn (p,d,q) theo AIC/BIC và dự báo ARIMA

Bạn có thể export notebook chạy ra HTML:

```bash
jupyter nbconvert notebooks/runs/03_classification_modelling_run.ipynb --to html
```

## Ứng dụng thực tế 

Thiết kế bài giảng “end-to-end”:

  phân lớp mức độ ô nhiễm (classification) + chống leakage

  hồi quy dự đoán chỉ số PM2.5 tương lai (regression)

  phân tích chuỗi thời gian và quyết định dùng ARIMA (time series)

Demo ra quyết định mô hình dựa trên:

  stationarity (ADF/KPSS), ACF/PACF

  tiêu chí IC (AIC/BIC) và kiểm tra sai số dự báo

### Tech Stack

| Công nghệ | Mục đích |
|----------|----------|
| Python | Ngôn ngữ chính |
| Pandas | Xử lý dữ liệu transaction |
| Scikit-learn | Modelling & metrics |
| Statsmodels  | ARIMA               |
| Papermill | Chạy pipeline notebook tự động |
| Matplotlib & Seaborn | Visualization biểu đồ tĩnh |
| Plotly | Dashboard / biểu đồ tương tác |
| Jupyter Notebook | Môi trường notebook |

### Author
Project được thực hiện bởi:
Trang Le

### License
MIT — sử dụng tự do cho nghiên cứu, học thuật và ứng dụng nội bộ.