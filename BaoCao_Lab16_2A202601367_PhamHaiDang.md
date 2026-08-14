# BÁO CÁO THỰC HÀNH LAB 16: Cloud AI Environment Setup

- **Họ và tên:** Phạm Hải Đăng
- **MSSV:** 2A202601367
- **Nền tảng Cloud:** Amazon Web Services (AWS) — Region `us-east-1`
- **Instance type:** `t3.micro` (2 vCPU / 1 GB RAM — Free Tier eligible)
- **Ghi chú cấu hình:** Biến `cpu_instance_type` trong `terraform/variables.tf` được chủ động đổi từ mặc định `t3.medium` sang `t3.micro` để nằm trong Free Tier. Mức 1 GB RAM vẫn đủ để chạy trọn benchmark (peak chỉ dùng ~176 MiB).

---

## 1. Kết quả Benchmark LightGBM (Credit Card Fraud Detection — 284,807 giao dịch)

| Metric | Kết quả |
|---|---|
| Thời gian load data | 2.6634 s |
| Thời gian training | 2.4069 s |
| Best iteration | 1 |
| AUC-ROC | 0.951654 |
| Accuracy | 0.998947 |
| F1-Score | 0.727273 |
| Precision | 0.655738 |
| Recall | 0.816327 |
| Inference latency (1 row) | 0.4455 ms |
| Inference throughput (1000 rows) | 1,432,287.19 rows/s |

---

## 2. Screenshot minh chứng

| # | Nội dung | File |
|---|---|---|
| 1 | Terminal chạy `python3 benchmark.py` — bảng kết quả benchmark (ảnh bị cắt phần trên của bảng; đầy đủ 10 metrics xem ở screenshot #2) | `screenshot_benchmark_results.png` |
| 2 | Nội dung file `benchmark_result.json` trên Compute Node — đầy đủ 10 metrics | `screenshot_benchmark_json.png` |
| 3 | Tài nguyên RAM (`free -h`) và Network (`ip -s link`) | `screenshot_resource_usage.png` |
| 4 | AWS Billing Dashboard — bill summary kỳ 08/2026 | `screenshot_aws_billing.png` |

---

## 3. Tài nguyên hệ thống (Compute Node `t3.micro`)

- **RAM:** Tổng 914 MiB, đã dùng 176 MiB (~19%), còn khả dụng 564 MiB. Swap = 0 B.
- **Network (ens5):** RX 286.2 MB (195,948 packets) / TX 915.6 KB (11,299 packets). 0 errors, 0 dropped.
- **CPU:** Không chụp được output `top` tại thời điểm chạy benchmark. Có thể suy ra gián tiếp từ thời gian training 2.41 s trên 2 vCPU — tải CPU chỉ tăng cao trong khoảng vài giây ngắn, phần lớn thời gian instance ở trạng thái idle.
- **Chi phí AWS:** Ảnh Billing Dashboard hiển thị `Estimated grand total = USD 0.00` kèm thông báo *"There is no data to display"* — đây là do **dữ liệu billing của AWS trễ khoảng 6–24 giờ**, chưa kịp cập nhật tại thời điểm chụp, **không phải** vì toàn bộ hạ tầng miễn phí. Chi phí thực tế được ước tính như sau:

| Dịch vụ | Free Tier? | Chi phí thực tế (ước tính) |
|---|---|---|
| EC2 Bastion `t3.micro` | Có (750 h/tháng) | $0.00 |
| EC2 Compute Node `t3.micro` | Có (dùng chung hạn mức 750 h) | $0.00 |
| **NAT Gateway** | **Không có Free Tier** | ~$0.045/giờ × ~1–2 h ≈ $0.05–0.09 |
| NAT data processing | Không | $0.045/GB × ~0.29 GB ≈ $0.01 |
| ALB | 750 h/tháng trong 12 tháng đầu | $0.00 |
| **Tổng** | | **≈ $0.06 – $0.10** |

---

## 4. Nhận xét (Báo cáo ngắn — Deliverable 6)

1. **Training time:** LightGBM huấn luyện trên bộ dữ liệu ~284K dòng chỉ mất **2.41 giây** trên CPU instance `t3.micro` (1 GB RAM), chứng tỏ thuật toán Gradient Boosting được tối ưu rất tốt cho dữ liệu dạng bảng mà không cần GPU.
2. **AUC-ROC đạt 0.9517**, cho thấy mô hình phân biệt giao dịch gian lận và hợp lệ ở mức tốt (xem thêm nhận xét số 5 về dư địa cải thiện). Accuracy đạt 99.89% nhưng do dữ liệu mất cân bằng nặng (chỉ 0.17% là fraud), chỉ số F1-Score (0.727) và Precision (0.656) phản ánh đúng hơn chất lượng phân loại lớp thiểu số.
3. **Recall đạt 0.816** — mô hình phát hiện được hơn 81% giao dịch gian lận thực tế, đây là chỉ số quan trọng nhất trong bài toán fraud detection vì bỏ sót gian lận gây thiệt hại lớn hơn cảnh báo nhầm.
4. **Inference cực nhanh:** Latency chỉ 0.45 ms/row, throughput đạt hơn 1.43 triệu rows/giây — hoàn toàn đáp ứng yêu cầu phục vụ real-time trên quy mô sản xuất mà chỉ cần CPU instance nhỏ.
5. **Best iteration = 1 — điểm đáng lưu ý:** Early stopping dừng ngay ở cây thứ nhất, nghĩa là mô hình cuối cùng gần như chỉ dùng một cây quyết định. Việc chỉ một cây đã cho AUC 0.9517 cho thấy dữ liệu có vài feature phân biệt rất mạnh (các thành phần PCA `V14`, `V17`...), nhưng con số này vẫn thấp hơn mức LightGBM thường đạt trên bộ dữ liệu này (~0.97–0.98). Nếu nới lỏng `early_stopping_rounds` và tăng `num_boost_round`, AUC gần như chắc chắn sẽ cải thiện — đây là hướng tối ưu tiếp theo, đổi lại thời gian training sẽ dài hơn.
6. **Chi phí thực tế ≈ $0.06–0.10, không phải $0.00:** EC2 (Bastion + Compute Node `t3.micro`) và ALB nằm trong Free Tier, nhưng **NAT Gateway không có Free Tier** (~$0.045/giờ + $0.045/GB) nên vẫn phát sinh vài cent. Nhờ chạy `terraform destroy` ngay sau khi thu thập xong số liệu, tổng thời gian tồn tại của hạ tầng chỉ khoảng 1–2 giờ nên chi phí giữ được ở mức không đáng kể. Con số $0.00 trên Billing Dashboard chỉ phản ánh việc dữ liệu billing chưa kịp cập nhật.

---

## 5. Dọn dẹp tài nguyên

- Đã chạy `terraform destroy` thành công: **27 resources destroyed**.
- Xác nhận terminal hiển thị `Destroy complete!`.
