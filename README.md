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
Document_Restoration_Project/
├── checkpoints/ # Checkpoint backup từ Colab
│ ├── 0_pretrained/ # Model gốc ban đầu
│ ├── 1_noise/
│ ├── 2_blur/
│ ├── 3_broken/
│ ├── 4_missing/
│ ├── 5_occlusion/
│ └── 6_final/
├── configs/ # YAML config cho mỗi bước
│ ├── step1_noise.yml
│ ├── step2_blur.yml
│ └── ...
├── notebooks/ # Notebook Colab
│ └── train_pipeline.ipynb
└── README.md

---

## 🔄 Quy trình huấn luyện

Mỗi bước thực hiện 5 giai đoạn:

### 1. Detect + Copy checkpoint

Tự động phát hiện checkpoint cao nhất từ bước trước và copy sang runtime.

**Hai chế độ:**
- `MODE = 'new'` — Train bước mới, lấy weight từ bước trước, **không** load optimizer state.
- `MODE = 'resume'` — Train tiếp bước đang dở, load cả weight + optimizer state.

### 2. Tạo dataset replay

Gộp dữ liệu degradation mới + mẫu replay từ các bước cũ theo tỷ lệ định sẵn.

### 3. Sinh YAML config

Tự động sinh file config với đường dẫn pretrain, resume, dataset phù hợp.

### 4. Train

Chạy `basicsr/train.py` với config đã sinh.

### 5. Backup

Copy checkpoint mới nhất + log + config lên Google Drive.

---

## 🚀 Hướng dẫn sử dụng

### Yêu cầu

- Google Colab (khuyến nghị GPU T4/A100)
- Google Drive để lưu checkpoint
- Dataset theo cấu trúc:
data/
├── train/
│ ├── clean/ # Ảnh sạch
│ └── degraded/
│ ├── noise/ # Ảnh nhiễu
│ ├── blur/ # Ảnh mờ
│ ├── broken/ # Ảnh đứt nét
│ ├── missing/ # Ảnh mất chữ
│ └── occlusion/ # Ảnh bị che
└── val/
└── ... (tương tự train)


### Setup lần đầu

```python
from google.colab import drive
drive.mount('/content/drive')
!git clone https://github.com/swz30/Restormer.git /content/Restormer
Pipeline huấn luyện

Mỗi bước thực hiện 5 cell theo thứ tự:
📌 Cell 1 — Detect + Copy checkpoint

Tự động phát hiện checkpoint cao nhất từ bước trước và copy sang runtime.

Chỉ sửa 2 dòng ở đầu cell:
python

STEP_NAME = 'broken'    # bước hiện tại: 'noise'|'blur'|'broken'|'missing'|'occlusion'|'final'
MODE      = 'new'       # 'new' = train từ đầu | 'resume' = train tiếp

Hai chế độ:
MODE	Nguồn weight	Copy .state?	Train từ
'new'	Bước trước (hoặc 0_pretrained/)	❌ Không	iter 0
'resume'	Bước hiện tại	✅ Có	iter tiếp theo
📌 Cell 1B — Tạo dataset replay

Gộp dữ liệu degradation mới + mẫu replay từ các bước cũ theo tỷ lệ định sẵn.

Tự động theo STEP_NAME từ Cell 1 — không cần sửa gì.

    Tỷ lệ NEW_RATIO giảm dần: bước 1 = 1.0 → bước 6 = 0.2.

    Dùng hardlink thay vì copy → tiết kiệm dung lượng.

    File replay trỏ cùng inode với file gốc trong /content/data/.

📌 Cell 2 — Sinh YAML config

Tự động sinh file config với:

    pretrain_network_g = checkpoint từ Cell 1.

    resume_state = .state (chỉ khi MODE = 'resume').

    dataroot_gt/lq = đường dẫn dataset replay từ Cell 1B.

📌 Cell 3 — Train
bash

!cd /content/Restormer && python basicsr/train.py -opt my_config_{STEP_NAME}.yml

📌 Cell 4 — Backup lên Drive

Copy checkpoint mới nhất + log + config lên Google Drive:
text

Document_Restoration_Project/checkpoints/{STEP_NUMBER}_{STEP_NAME}/
├── models/net_g_XXXXX.pth
├── training_states/XXXXX.state
├── log/
└── my_config.yml

Chuyển bước tiếp theo

Sửa duy nhất Cell 1:
python

STEP_NAME = 'missing'   # bước kế tiếp
MODE      = 'new'

Rồi chạy lại: Cell 1 → 1B → 2 → 3 → 4.
Ví dụ chạy đủ 6 bước
Lần chạy	STEP_NAME	MODE	Weight lấy từ
1	noise	new	0_pretrained/
2	blur	new	noise
3	broken	new	blur
4	missing	new	broken
5	occlusion	new	missing
6	final	new	occlusion

Nếu Colab bị ngắt giữa chừng, mở lại và đổi:
python

STEP_NAME = 'broken'    # bước đang dở
MODE      = 'resume'    # ← đổi thành resume

→ Train tiếp từ iter cuối cùng đã lưu.
