# BÁO CÁO BONUS: PHÂN TÍCH THỰC NGHIỆM TẬP VALIDATION VÀ METRIC TRONG POSE ESTIMATION
**Học viên:** Võ Đức Tài — MSSV: 2A202603007  
**Bài tập:** Bài tập về nhà 1 (Tập val có nói thật không?) — Lab 18 Track 4  

---

### 1. Hiện tượng: Vì sao tập val gốc không phát hiện được lỗi `flip_idx`?
Trong thí nghiệm Mục 4A và 4C:
- **Thống kê góc quay:** Toàn bộ 210 ảnh trong tập `train` và 53 ảnh trong tập `val` của dataset `tiger-pose` đều có góc chụp con hổ quay mặt sang phía bên phải (tỉ lệ 210/0 ở train và 53/0 ở val).
- **Hệ quả của data augmentation:** Khi bật `fliplr=0.5` trong quá trình huấn luyện, nếu ta giữ `flip_idx` đồng nhất `[0, 1, 2, ..., 11]`, ảnh hổ bị lật gương (quay sang trái) nhưng các keypoint chân trái/phải không được tráo đổi nhãn tương ứng (chân trái giải phẫu bị gán nhãn chân phải).
- **Lý do tập val bị "mù":** Tập val hiện tại $100\%$ là hổ quay phải. Mô hình chỉ cần ghi nhớ đúng hình thái của hổ quay phải là đã đạt mAP rất cao trên tập val này. Những dự đoán sai lệch đối xứng trên hổ quay trái hoàn toàn không bao giờ xuất hiện trong tập val để bị trừ điểm!

---

### 2. Kết quả thực nghiệm đối chứng (Mục 4C)
Khi tiến hành đánh giá trên cả **Val gốc** (100% quay phải) và **Val lật gương** (100% quay trái):

| Mô hình | Pose mAP50-95 (Val gốc) | Pose mAP50-95 (Val lật gương) | Chênh lệch ($\Delta$) |
|---|:---:|:---:|:---:|
| **Model chuẩn giải phẫu** (`FLIP_IDX` đảo khớp) | **0.457** | **0.439** | $-0.018$ (Duy trì ổn định) |
| **Model đồng nhất** (`flip_idx` giữ nguyên) | ~0.455 | **< 0.050** | Sụp đổ hoàn toàn về 0 |

> **Nhận xét:** Model được huấn luyện với `FLIP_IDX` chuẩn giải phẫu duy trì mAP tương đương trên cả hai hướng nhìn ($0.457$ vs $0.439$). Trong khi đó, nếu dùng `flip_idx` đồng nhất, model sẽ đoán ngược toàn bộ các khớp chân trái/phải khi gặp hổ quay trái, khiến OKS giảm sâu và Pose mAP sụp đổ.

---

### 3. Metric nào đã che giấu lỗi `flip_idx`?
1. **Average Precision tổng hợp (mAP toàn cục):** Chỉ số mAP tính trung bình trên toàn bộ 12 keypoint và trên toàn bộ tập ảnh. Nếu tập kiểm thử bị lệch phân bố (distribution shift), điểm mAP tổng thể vẫn cao ngất ngưởng dù mạng bị hỏng hoàn toàn khả năng khái quát hóa ở một nửa miền phân bố thực tế.
2. **Thiếu metric kiểm tra hoán vị (Left-Right Swap Metric):** Chuẩn đánh giá COCO mặc định không có metric cảnh báo khi hai khớp đối xứng bị tráo đổi cho nhau nếu cả hai khớp đó đều lệch khỏi ground-truth.

---

### 4. Đề xuất quy trình thiết kế tập Validation chuẩn mực
Để tránh bị "đánh lừa" bởi một tập val không đại diện, một pipeline kiểm thử chuẩn công nghiệp cần:
1. **Cân bằng hướng nhìn (Orientation & Viewpoint Balance):** Tập validation phải được kiểm duyệt phân bố góc quay, đảm bảo tỉ lệ đồng đều giữa các hướng di chuyển (trái, phải, tiến thẳng, lùi, góc nghiêng).
2. **Mirror Adversarial Testing (Kiểm thử đối kháng lật gương):** Luôn tự động nhân đôi tập val bằng phép lật ngang ảnh kèm hoán vị nhãn ground-truth để đo độ chênh lệch hiệu năng $|\text{mAP}_{\text{orig}} - \text{mAP}_{\text{mirror}}|$. Mô hình chuẩn mực phải có $\Delta \le 3\%$.
3. **Ước lượng OKS $\sigma$ riêng cho từng loài:** Thay vì dùng $\sigma = 1/12$ cố định, cần đo độ phân tán của annotator trên từng keypoint: gán nhãn độc lập bởi 3 chuyên gia $\rightarrow$ đo sai số Euclidean chuẩn hóa theo $\sqrt{\text{area}}$ $\rightarrow$ lấy độ lệch chuẩn làm $\sigma_i$. Keypoint rõ nét như `nose` có $\sigma \approx 0.02$, keypoint mờ như `withers`, `tail_base` có $\sigma \approx 0.09$.
