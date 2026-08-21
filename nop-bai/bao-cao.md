# Báo Cáo Lab Day 21 - CI/CD cho AI Systems

| | |
|---|---|
| Họ và tên | Đào Minh Chiến |
| MSSV | 01184 |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/ChienhocIT/K4-Track2-Day21-CI-CD-for-AI-Systems-01184-DaoMinhChien |
| Ngày nộp | 21/08/2026 |

---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

| Lần chạy | n_estimators | learning_rate | max_depth | f1_score | accuracy |
|---|---|---|---|---|---|
| 1 | 100 | 0.1 | 3 | 0.7109 | 0.8780 |
| 2 | 50 | 0.05 | 2 | 0.6051 | 0.8460 |
| 3 | 200 | 0.1 | 5 | 0.7149 | 0.8740 |

**Bộ siêu tham số đã chọn:** `n_estimators=100`, `learning_rate=0.1`, `max_depth=3`.

**Lý do:** Bộ siêu tham số này đạt điểm `f1_score = 0.7109` trên tập holdout, vượt xa ngưỡng chất lượng yêu cầu (0.65) và đạt độ chính xác tổng thể cao nhất (0.8780). Qua các lần chạy, ta nhận thấy sự đánh đổi rõ rệt: giảm `learning_rate` xuống 0.05 kết hợp `max_depth=2` và ít cây (`n_estimators=50`) khiến mô hình bị underfitting nặng, `f1_score` tụt xuống 0.6051 và sẽ bị Quality Gate chặn. Ngược lại, việc tăng lên 200 cây với độ sâu 5 chỉ cải thiện F1 rất nhỏ (0.7149) nhưng tăng đáng kể chi phí tính toán và thời gian huấn luyện. Lần chạy 1 mang lại sự cân bằng tối ưu giữa hiệu năng dự đoán và tốc độ huấn luyện.

---

## 2. Vì Sao Ngưỡng Chất Lượng Đặt Trên F1 Chứ Không Phải Accuracy

Tập dữ liệu điều tra dân số Adult có sự mất cân bằng lớp rõ rệt: chỉ 24.8% số mẫu thuộc lớp thu nhập cao (>50K USD). Trong điều kiện này, một mô hình tầm thường (luôn đoán nhãn 0 "thu nhập thấp" cho mọi trường hợp) vẫn dễ dàng đạt được Accuracy lên tới 0.752 (75.2%), nhưng hoàn toàn vô dụng vì điểm `f1_score` lớp dương bằng 0.0 do không phát hiện được bất kỳ ai có thu nhập cao. 

Do đó, chỉ số Accuracy gây hiểu nhầm nghiêm trọng. `f1_score` của lớp dương (harmonic mean giữa Precision và Recall) đo lường chính xác năng lực nhận diện lớp thiểu số mục tiêu. Ta tuyệt đối không sử dụng `average="weighted"` hay `average="macro"` vì các trọng số này sẽ bị lớp đa số (75.2%) kéo lên cao giả tạo, làm mất đi tính nghiêm ngặt của Quality Gate.

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

| Khó khăn | Nguyên nhân | Cách giải quyết |
|---|---|---|
| Lỗi unpickle model trên VM (`AttributeError`) | Phiên bản scikit-learn trên VM (1.7.2) bị lệch so với phiên bản huấn luyện (1.4.2). | Cài đặt chính xác `scikit-learn==1.4.2` đồng bộ trên cả máy local và VM. |
| SSH deploy action thất bại do format key | Action `appleboy/ssh-action` yêu cầu private key định dạng chuẩn OpenSSH/PEM. | Tạo khóa RSA 2048-bit định dạng PEM và cập nhật lại vào GitHub Secret `SERVER_SSH_KEY`. |
| Pipeline CI/CD lỗi thiếu dữ liệu khi dvc pull | Thực hiện `git push` trước khi đẩy dữ liệu lên Cloud Storage. | Luôn tuân thủ quy trình `dvc push` dữ liệu lên S3 trước khi thực hiện `git push`. |

---

## 4. So Sánh Bước 2 và Bước 3

| | f1_score | accuracy |
|---|---|---|
| Bước 2 (chỉ `train_batch1` - 22.361 mẫu) | 0.7109 | 0.8780 |
| Bước 3 (thêm `train_batch2` - 44.722 mẫu) | 0.7014 | 0.8740 |

**Nhận xét:** Khi bổ sung thêm 22.361 mẫu ở Bước 3, `f1_score` dao động nhẹ (-0.0095) do toàn bộ dữ liệu ban đầu được trích xuất ngẫu nhiên từ cùng một nguồn nên có cùng phân phối xác suất; mô hình GradientBoosting đã học hầu hết đặc trưng từ nửa dữ liệu đầu tiên. Điểm quan trọng nhất ở Bước 3 là tính tự động hóa: chỉ bằng một commit cập nhật dữ liệu, GitHub Actions đã tự động kích hoạt, pull dữ liệu mới từ S3, huấn luyện lại, vượt qua Quality Gate và deploy thành công lên EC2 mà không cần bất kỳ can thiệp thủ công nào.
