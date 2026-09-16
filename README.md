# Document Restoration with Restormer

Phục hồi ảnh tài liệu bị hư hỏng bằng mô hình **Restormer**, huấn luyện tuần tự qua **6 bước degradation** với kỹ thuật **replay dataset** để tránh catastrophic forgetting.

---

## 📋 Tổng quan

Project huấn luyện một mô hình duy nhất có khả năng phục hồi nhiều loại hư hỏng trên ảnh tài liệu:

| Bước | Loại hư hỏng | Mô tả |
|:---:|---|---|
| 1 | **Noise** | Nhiễu hạt, nhiễu Gaussian |
| 2 | **Blur** | Mờ do mất nét, motion blur |
| 3 | **Broken** | Nét chữ bị đứt, gãy |
| 4 | **Missing** | Mất chữ, mất dòng |
| 5 | **Occlusion** | Che khuất (con dấu, vết bẩn) |
| 6 | **Final** | Tổng hợp mọi loại hư hỏng |

### 🎯 Chiến lược train

- **Progressive training**: Train tuần tự từ bước 1 → 6, mỗi bước kế thừa weight từ bước trước.
- **Replay dataset**: Mỗi bước mới pha thêm dữ liệu của các bước cũ để mô hình không quên kiến thức đã học.
- **Tỷ lệ replay giảm dần**: Bước càng cao, tỷ lệ dữ liệu mới càng giảm.

| Bước | Tỷ lệ dữ liệu mới | Tỷ lệ replay |
|:---:|:---:|:---:|
| 1 - Noise | 100% | 0% |
| 2 - Blur | 70% | 30% |
| 3 - Broken | 50% | 50% |
| 4 - Missing | 40% | 60% |
| 5 - Occlusion | 30% | 70% |
| 6 - Final | 20% | 80% |

---

## 🏗️ Kiến trúc

**Base model**: [Restormer](https://github.com/swz30/Restormer) — Transformer cho image restoration.

**Input**: Ảnh tài liệu degraded (LQ)  
**Output**: Ảnh tài liệu sạch (GT)

---

## 📂 Cấu trúc thư mục
