# 🛡️ DDoS Detection with SDN Floodlight + ML (2025)

Một hướng dẫn ngắn gọn, đẹp và nhất quán để chạy demo phát hiện & chặn DDoS theo thời gian thực bằng SDN Floodlight, Mininet và mô hình Machine Learning huấn luyện từ CICIDS2017.

---

## 📚 Tổng quan hệ thống

- **Kiến trúc**: Mininet sinh traffic → `tcpdump` quay PCAP theo cửa sổ thời gian → CICFlowMeter chuyển PCAP thành CSV đặc trưng → mô hình ML dự đoán → controller Floodlight nhận cảnh báo và đẩy flow rule chặn IP tấn công.
- **Mục tiêu**: Quan sát traffic bình thường vs. DDoS, đánh giá mô hình ML và chứng minh khả năng chặn tấn công tự động.

## 🧩 Yêu cầu môi trường

- OS khuyến nghị: Ubuntu 22.04 (máy ảo được khuyên dùng).
- Python ≥ 3.10, `pip` đầy đủ.
- JDK 1.8 cho Floodlight.
- Quyền `sudo` cho `tcpdump`, Mininet, Wireshark.
- Cài đặt CICFlowMeter, build nó thành 1 thư mục và cho vào thư mục root của dự án.
- Giải nén thư mục .venv.zip thành .venv để đảm bảo chứa các thư viện cần thiết để chạy dự án

> Ghi chú: Repo có sẵn script cho Linux. Trên Windows, hãy chạy qua WSL/VM Ubuntu để tương thích hoàn toàn.

---

## 🚀 Quy trình chạy nhanh

Thư mục làm việc: `source/`

### 1) ⚙️ Khởi động Floodlight

```bash
sudo java -jar target/floodlight.jar
```

Kiểm tra controller thấy switch:

```
http://localhost:8080/wm/core/controller/switches/json
```

### 2) 🧠 Huấn luyện mô hình ML với CICIDS2017

```bash
python3 machinelearning/ML_trainer.py --csv dataset/CICIDS2017_processed.csv
```

Kết quả sinh ra:
- `model.pkl`: mô hình đã huấn luyện.
- `metadata.pkl`: danh sách đặc trưng, medians, scaler, threshold.
- `model_eval_roc.png`, `model_eval_pr.png`, `model_eval_cm.png`: ảnh đánh giá hiệu năng.

> Sau khi train, hãy xem các ảnh để đánh giá AUC/PR và confusion matrix.

### 3) 🌐 Khởi chạy topology Mininet và host

```bash
python3 mininet/topology.py
```

Trong CLI Mininet:

```bash
xterm h1 h2 h3
```

Trên từng xterm:
- `h1`: chạy HTTP server
  ```bash
  python3 -m http.server 80
  ```
- `h2`: ping đều về `h1`
  ```bash
  csh mininet/ping.csh
  ```

### 4) 👀 Quan sát traffic trước khi DDoS (tuỳ chọn)

Mở Wireshark để theo dõi:

```bash
sudo wireshark
```

### 5) 🔥 Chạy DDoS không có ML (để quan sát baseline)

Trên `h3`:

```bash
sh mininet/ddos.sh
```

Chạy khoảng 300s để dễ quan sát biểu đồ traffic.

### 6) 📥 Bật bắt gói tin và chuyển đổi PCAP → CSV realtime

Trong `source/`:

```bash
./processing/capture_tcpdump.sh
./processing/pcap_processor.sh
```

- `capture_tcpdump.sh`: quay PCAP theo cửa sổ (mặc định 15s) vào `output/pcap_in`.
- `pcap_processor.sh`: dùng CICFlowMeter chuyển PCAP sang CSV; gộp vào `output/final_csv/Predict.csv`.

### 7) 🛡️ Chạy ML realtime để chặn DDoS

```bash
python3 controller/realtime_floodlight_ML.py --csv output/final_csv/Predict.csv --threshold 0.07
```

- Đọc CSV realtime, tiền xử lý theo `metadata.pkl`, dự đoán và chặn IP nguồn khi xác suất ≥ ngưỡng.

### 8) 🔁 Chạy lại DDoS trên `h3` để kiểm chứng chặn realtime

```bash
sh mininet/ddos.sh
```

Khi phát hiện tấn công, terminal ML sẽ in thông báo kiểu “Block src…”, và Floodlight sẽ cài flow rule drop, khiến host tấn công không thể truy cập `h1`.

---

## 🧱 Cấu trúc dữ liệu & mô hình

- `feature_schema.py`: định nghĩa `FEATURE_NAMES` (bộ đặc trưng đầu vào), ánh xạ cột từ CICFlowMeter và loại bỏ nhóm Active/Idle nhiễu.
- `ML_trainer.py`: tiền xử lý (impute median, chuẩn hoá), train `RandomForest + CalibratedClassifierCV`, lưu `model.pkl` và `metadata.pkl`.
- `realtime_floodlight_ML.py`: đọc CSV, đồng bộ schema, chuẩn hoá theo `metadata`, dự đoán, gọi Floodlight API để push static flow chặn IP.
- `processing/`: `capture_tcpdump.sh` (quay PCAP), `pcap_processor.sh` (CICFlowMeter → CSV, gộp `Predict.csv`).

---

## 🛠️ Gợi ý khắc phục sự cố

- CICFlowMeter yêu cầu đúng JAR và `jnetpcap`. Kiểm tra biến trong `processing/pcap_processor.sh` nếu lỗi phụ thuộc.
- Nếu `Predict.csv` chưa được cập nhật, xác nhận `capture_tcpdump.sh` đang tạo PCAP và script processor đang chạy.
- Floodlight API khác nhau theo phiên bản; script tự phát hiện endpoint push-flow. Nếu không thấy `dpid`, kiểm tra kết nối switch Mininet.

---

## 📝 Tham khảo lệnh nhanh

```bash
# Floodlight
sudo java -jar target/floodlight.jar

# Train ML
python3 machinelearning/ML_trainer.py --csv dataset/CICIDS2017_processed.csv

# Topology + hosts
python3 mininet/topology.py
xterm h1 h2 h3
h1: python3 -m http.server 80
h2: csh mininet/ping.csh

# Quan sát traffic
sudo wireshark

# DDoS baseline
sh mininet/ddos.sh

# Realtime capture + features
./processing/capture_tcpdump.sh
./processing/pcap_processor.sh

# Realtime ML + block
python3 controller/realtime_floodlight_ML.py --csv output/final_csv/Predict.csv --threshold 0.07
```

---

## 📦 Đầu ra chính

- `output/final_csv/Predict.csv`: CSV đặc trưng thời gian thực từ PCAP.
- `model.pkl`, `metadata.pkl`: mô hình và meta cho suy luận realtime.
- `model_eval_roc.png`, `model_eval_pr.png`, `model_eval_cm.png`: ảnh đánh giá mô hình.

Chúc bạn demo thành công!
