
## [Q1] Khám phá & Kiểm tra sức khỏe dữ liệu (Data Understanding)

[Q1] Khám phá & Kiểm tra sức khỏe dữ liệu (Data Understanding)
1. Giới thiệu bài toán
Trong dự báo chuỗi thời gian (Time Series Forecasting), câu hỏi kinh điển luôn là: "Nên dùng một mô hình thống kê chặt chẽ như ARIMA hay một mô hình hồi quy đơn giản dựa trên đặc trưng (Feature-based Regression)?"
1.1. Kiểm tra phạm vi thời gian (Time Range)
Trước khi đi vào xây dựng mô hình dự báo, bước quan trọng nhất là "hiểu" dữ liệu. Trong phần này, chúng tôi thực hiện rà soát toàn diện để đảm bảo tính toàn vẹn, phát hiện ngoại lai và nắm bắt các đặc tính chuỗi thời gian quan trọng của biến mục tiêu (PM2.5).

Dựa trên log chạy make_hourly_station_series và kích thước dữ liệu, chúng tôi xác nhận:

Thời gian bắt đầu: 2013-05-04 00:00:00.

Thời gian kết thúc: 2017-07-21 23:00:00.

Độ dài chuỗi: 35,064 điểm dữ liệu (tương ứng trọn vẹn 4 năm).

Tần suất: Hourly (Hàng giờ).

Kết luận: Dữ liệu liên tục, đảm bảo tần suất theo giờ sau khi xử lý (reindex/interpolate).

1.2. Phân tích dữ liệu thiếu (Missing Values)
Quan sát bảng thống kê missing_rate từ quá trình phân tích dữ liệu (EDA):

Quan sát biểu đồ phân bố dữ liệu thiếu theo thời gian: Các vạch đỏ (dữ liệu thiếu) phân bố rải rác nhưng có một vài giai đoạn tập trung dày đặc.
Mất nhãn (Ground Truth): PM2.5 là mục tiêu dự báo. Thiếu nó đồng nghĩa với việc không thể tính toán sai số (Loss) để huấn luyện mô hình tại các khung giờ đó.

Đứt gãy chuỗi tự hồi quy (Autoregression): Các mô hình dự báo (ARIMA, Regression Lag) dựa vào giá trị quá khứ ($t-1, t-24$) để dự báo tương lai. Nếu PM2.5 bị thiếu một khoảng dài (ví dụ mất tín hiệu cảm biến trong 12h), chuỗi liên kết bị đứt, khiến mô hình mất thông tin đầu vào quan trọng nhất.

1.3. Phân phối và Ngoại lai (Outliers)
Thống kê: Min = 3.0, Max = 898.0, Mean = 82.5.

Phân phối: Biểu đồ Boxplot cho thấy phân phối lệch phải (Right-skewed) với đuôi rất dài.

Ngoại lai (Outliers): Có rất nhiều điểm dữ liệu nằm vượt xa "râu trên" của Boxplot (các giá trị > 400). Tuy nhiên, đây là đặc trưng của các đợt "bão bụi" tại Bắc Kinh nên chúng tôi giữ nguyên để mô hình học các sự kiện cực đoan này.

1.4. Trực quan hóa chuỗi thời gian
Toàn cảnh (2013-2017): Dữ liệu thể hiện tính mùa vụ rõ rệt theo năm (ô nhiễm thường cao vào mùa đông và thấp vào mùa hè).

Phóng to (Zoom 2 tháng): Quan sát đoạn dữ liệu tháng 1-2/2016 cho thấy rõ nhịp dao động theo ngày (Daily cycle) và sự đồng pha giữa các trạm.

1.5. Kiểm tra Tự tương quan (Autocorrelation)Kết quả tính toán hệ số tương quan (Pearson) với các độ trễ:Lag 24h (1 ngày): Hệ số tương quan $r \approx 0.4085$.Lag 168h (1 tuần): Hệ số tương quan $r \approx 0.0297$.
Nhận xét: $r_{24h} \gg r_{168h}$. Tính chu kỳ theo NGÀY mạnh hơn rất nhiều so với chu kỳ tuần. Điều này gợi ý việc sử dụng đặc trưng trễ Lag_24 sẽ hiệu quả hơn Lag_168.

1.6. Kiểm định tính dừng (Stationarity Test)
Chúng tôi sử dụng kiểm định Augmented Dickey-Fuller (ADF).

ADF Statistic: -19.5261

p-value: 0.000000
Kết luận: Vì p-value $< 0.05$, chuỗi dữ liệu PM2.5 CÓ TÍNH DỪNG (Stationary) về mặt thống kê. Mô hình ARIMA có thể bắt đầu với tham số sai phân $d=0$.

## [Q2] Giải mã Mô hình Hồi quy (Regression Baseline)
Khi so sánh chỉ số sai số trên toàn bộ tập kiểm thử (Test set), kết quả cho thấy:
2.1. Tại sao Lag 24h (Độ trễ 1 ngày) lại quan trọng nhất?
Trong quá trình Feature Engineering, chúng tôi nhận thấy đặc trưng PM2.5_Lag24 (Nồng độ bụi tại cùng giờ ngày hôm qua) đóng vai trò quan trọng hàng đầu.
Lý do khoa học:

Nhịp sinh hoạt con người: Hoạt động gây ô nhiễm như giao thông, đun nấu tuân theo quy luật 24 giờ chặt chẽ.

Chu kỳ khí tượng: Nhiệt độ, độ ẩm và gió cũng thay đổi theo chu kỳ ngày đêm.

2.2. Tại sao phải chia tập dữ liệu bằng CUTOFF (Time-based Split)?
Khác với các bài toán thông thường, với dữ liệu chuỗi thời gian, chúng tôi chia Train/Test theo mốc thời gian cố định (Cutoff Date: 2017-01-01).

Lý do:

Tránh "Nhìn thấy tương lai" (Data Leakage): Không dùng dữ liệu ngày mai để dự báo cho hôm nay.

Đánh giá thực tế: Mô phỏng kịch bản vận hành thực tế là dùng quá khứ dự báo tương lai.

2.3. Phân biệt RMSE và MAE: Khi nào RMSE cao vọt?
Khi đánh giá sai số trên tập Test (2017), ghi nhận sự chênh lệch đáng kể:

RMSE: 25.57

MAE: 12.52

Tại sao RMSE lại lớn gấp đôi MAE?
RMSE sử dụng bình phương sai số ($\text{error}^2$), nghĩa là nó "phạt" rất nặng các sai số lớn. Sự chênh lệch này chỉ ra rằng mô hình dự báo tốt ở mức trung bình, nhưng gặp sai số lớn tại các điểm dị biệt (Spikes/Outliers) - những thời điểm "bão bụi" nồng độ tăng vọt.

## [Q3] Quy trình ra quyết định lựa chọn mô hình ARIMA

3.1. Quan sát chuỗi gốc (Identification)

Quan sát: Dữ liệu PM2.5 không có xu hướng (Trend) tăng/giảm dài hạn rõ rệt nhưng tính mùa vụ (Seasonality) rất mạnh.

Định hướng: Việc không có trend gợi ý tham số sai phân $d$ có thể thấp ($d=0$ hoặc $d=1$).

3.2. Kiểm định tính dừng để chọn tham số $d$Như đã kiểm định ở phần Q1, chuỗi có tính dừng với p-value = 0.000000.-> Quyết định: Chọn $d=0$ (hoặc thử $d=1$ để chắc chắn).

33.3. Phân tích ACF/PACF để ước lượng p và q 
PACF (Partial Autocorrelation): Các cọc trồi lên rõ rệt ở các lag đầu (1, 2, 3) rồi tắt dần \rightarrow Gợi ý p \in [1, 3].ACF (Autocorrelation): Cũng giảm dần hoặc cắt đuôi \rightarrow Gợi ý q \in [1, 3].

3.4. Grid Search tìm mô hình tối ưu
Nhóm em thực hiện vét cạn (Grid Search) trong không gian tham số nhỏ ($p, q \in [0..3]$) và chọn mô hình có chỉ số AIC thấp nhất.

Kết quả:ARIMA(0,0,0): AIC rất cao (Tệ nhất).ARIMA(1,0,3): Có AIC thấp nhất (~294,792).\rightarrow Quyết định: Chọn mô hình ARIMA(1, 0, 3).

3.5. Chẩn đoán phần dư (Residual Diagnostics)
Kiểm tra phần dư (Thực tế - Dự báo) để đảm bảo mô hình đã khai thác hết thông tin.

Kết quả: Trung bình phần dư xấp xỉ 0; biểu đồ ACF của phần dư nằm gọn trong khoảng tin cậy.

Kết luận: Phần dư là Nhiễu trắng (White Noise). Mô hình ARIMA(1, 0, 3) hợp lệ.

## Chủ đề 1: Regression vs ARIMA – Khi nào chọn cái nào?

1. Mô hình nào tốt hơn cho Horizon=1?
Dựa trên kết quả thực nghiệm, Regression Baseline là người chiến thắng áp đảo.

Tại sao?

Regression: Nhờ biến Lag 1 (giá trị giờ trước), mô hình "sao chép" được tính quán tính của bụi, giúp bám sát thực tế rất tốt.

ARIMA: Cấu trúc toán học phức tạp khiến mô hình đôi khi phản ứng không nhanh nhạy bằng, và sai số dễ cộng dồn.

2. Mô hình nào ổn hơn khi có Spike (Đột biến)
Regression: Phản ứng nhanh, leo được tới đỉnh của đợt ô nhiễm (dù bị trễ 1 nhịp).
ARIMA: Có xu hướng bị "mượt hóa" (Over-smoothing), đường dự báo thường đi ngang, không bắt được các đỉnh nhọn cực đoan \rightarrow Dẫn đến RMSE cao vọt.

3. Lựa chọn triển khai thực tế (Deployment Choice)
Nếu phải xây dựng hệ thống cảnh báo sớm, nhóm quyết định chọn: BASELINE REGRESSION.
Lý do:
Khả năng mở rộng: Dễ dàng thêm các biến Gió, Mưa, Độ ẩm vào Regression. ARIMA đơn biến không làm được điều này dễ dàng.
Tốc độ: Regression train nhanh, dự báo tức thì. ARIMA tốn tài nguyên để tìm tham số $(p,d,q)$.
Giá trị cảnh báo: Thà cảnh báo chậm 1 giờ nhưng đúng mức độ "Nguy hại", còn hơn dự báo "mượt mà" ở mức "Trung bình" trong khi thực tế đang ô nhiễm nặng
