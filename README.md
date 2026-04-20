# Do-an-Xu-ly-anh

Danh sách thành viên:<br>
<br>
Chử Văn Dũng - 0205768 <br>
Ngô Thị Thùy Linh - 0210168 <br> 
Nguyễn Anh Tuấn - 0214768 <br>
<br>
Đề tài: Nâng cao độ phân giải <br>
Sử dụng thuật toán Edge-Directed Unsharp Masking Sharpening (EDUMS) để phóng to ảnh lên với độ phân giải tốt hơn.<br>
<br>
# Xử Lý Hình Ảnh - Image Super-Resolution Pipeline

Dự án này triển khai một pipeline xử lý ảnh để nâng cao chất lượng ảnh (super-resolution) thông qua nhiều bước, bao gồm phóng to ảnh, khử răng cưa, làm sắc nét và khử rung.

## Dataset: Unsplash Dataset
---

## Tổng Quan Pipeline

```
Ảnh gốc
   │
   ▼
[1] Bicubic Upscaling (x2)
   │
   ▼
[2] EIAAF – Khử răng cưa theo hướng cạnh
   │
   ▼
[3] EDUMS – Làm sắc nét trên kênh Y (YCbCr)
   │
   ▼
[4] EICL – Khử ringing artifacts
   │
   ▼
Ảnh đầu ra chất lượng cao
```

Ngoài ra còn có phương pháp thứ hai là **IESR** để so sánh.

---

## Yêu Cầu Cài Đặt

```bash
pip install opencv-python numpy scipy scikit-image
```

| Thư viện | Mục đích |
|---|---|
| `opencv-python` | Đọc/ghi ảnh, xử lý ảnh cơ bản |
| `numpy` | Tính toán ma trận |
| `scipy` | Convolution, Gaussian filter |
| `scikit-image` | Đánh giá chất lượng ảnh (SSIM) |

---

## Cấu Trúc Thư Mục

```
project/
├── Dataset/           # Thư mục chứa ảnh đầu vào (1.jpg → 1254.jpg)
├── Output/            # Ảnh đầu ra sau từng bước
│   ├── Use_edums/     # Ảnh sau EDUMS + EICL
│   └── Use_IESR/      # Ảnh sau IESR
├── Thong_so_co_ban.txt    # Kết quả đánh giá pipeline EDUMS
├── Thong_so_co_ban_2.txt  # Kết quả đánh giá pipeline IESR
└── Xu-ly-hinh-anh.ipynb  # Notebook chính
```

---

## Các Phương Pháp

### 1. Bicubic Upscaling
Phóng to ảnh lên 2x bằng nội suy bicubic (`cv2.INTER_CUBIC`). Đây là bước khởi đầu cho toàn bộ pipeline.

---

### 2. EIAAF – Edge-Informed Anti-Aliasing Filter
Khử răng cưa (jagged edges) dựa trên thông tin hướng cạnh:
- Tính gradient theo chiều X và Y bằng Sobel operator
- Tạo mặt nạ (mask) từ độ lớn gradient
- Trộn ảnh gốc với ảnh đã làm mờ Gaussian theo mask: vùng gần cạnh sẽ được làm mượt hơn

---

### 3. EDUMS – Edge-Directed Unsharp Masking Sharpening
Làm sắc nét ảnh thông minh chỉ trên kênh Y (độ sáng) trong không gian màu YCbCr, giữ nguyên Cb và Cr để tránh làm nhiễu màu sắc:

```python
img3 = edums_nhanh_ycbcr(img2, alpha=1.5, sigma=1.5, bgr=True)
```

| Tham số | Ý nghĩa | Giá trị mặc định |
|---|---|---|
| `alpha` | Cường độ làm sắc nét | `1.5` |
| `sigma` | Độ rộng Gaussian blur (unsharp mask) | `1.5` |
| `bgr` | `True` nếu ảnh input là BGR (OpenCV) | `True` |
| `verbose` | In thông tin debug | `False` |

**Hỗ trợ:** Ảnh xám, ảnh màu RGB/BGR, ảnh có kênh alpha, float 0–1 và uint8.

---

### 4. EICL – Edge-Informed Contrast Limiting
Khử ringing artifacts (hiện tượng rung) xuất hiện sau bước làm sắc nét:
- Phát hiện cạnh bằng Sobel, tạo bản đồ trọng số (weight map)
- Làm mượt có chọn lọc tại vùng gần cạnh
- Xử lý trên kênh Y (YCbCr) để giữ màu sắc

```python
img4 = eicl_sau_edums(img3, nguong_canh=0.08, sigma_smooth=1.2, alpha=0.5, bgr=True)
```

| Tham số | Ý nghĩa | Khuyến nghị |
|---|---|---|
| `nguong_canh` | Ngưỡng phát hiện cạnh | `0.05 – 0.15` |
| `sigma_smooth` | Độ mạnh làm mượt | `1.0 – 2.0` |
| `alpha` | Cường độ loại bỏ ringing | `0.3 – 0.7` |

---

### 5. IESR – Iterative Edge-Sharpening Residual
Phương pháp thay thế, thực hiện làm sắc nét lặp lại bằng Laplacian:

```python
img5 = IESR(img, iters=3)
```

- Upscale kênh Y bằng bicubic, Cr/Cb bằng nearest-neighbor
- Lặp lại `iters` lần: cộng thêm phần residual Laplacian vào kênh Y
- Trả về ảnh BGR

---

## Đánh Giá Chất Lượng

Hàm `compare_images_2()` so sánh ảnh gốc và ảnh đã xử lý (cho phép khác kích thước), ghi kết quả ra file `.txt`:

| Chỉ số | Ý nghĩa |
|---|---|
| **SSIM** | Độ tương đồng cấu trúc (1 = giống hệt) |
| **PSNR** | Tỷ lệ tín hiệu/nhiễu theo đỉnh (dB); >40 dB = rất tốt |
| **MSE** | Sai số bình phương trung bình (0 = giống hệt) |
| **Histogram Correlation** | Độ tương quan phân bố màu sắc (1 = giống nhất) |
| **Độ sáng & tương phản** | Giá trị trung bình và độ lệch chuẩn kênh xám |

---

## Hướng Dẫn Sử Dụng

### Xử lý toàn bộ dataset (1–1254 ảnh)

```python
for i in range(1, 1255):
    nameImg = f"{i}"
    img = cv2.imread(f"Dataset/{nameImg}.jpg")

    # Bước 1: Bicubic upscale x2
    img1 = cv2.resize(img, None, fx=2, fy=2, interpolation=cv2.INTER_CUBIC)

    # Bước 2: EIAAF – khử răng cưa
    gray = cv2.cvtColor(img1, cv2.COLOR_BGR2GRAY)
    gx = cv2.Sobel(gray, cv2.CV_32F, 1, 0)
    gy = cv2.Sobel(gray, cv2.CV_32F, 0, 1)
    mag = cv2.magnitude(gx, gy)
    blur1 = cv2.GaussianBlur(img1, (5, 5), 0.5)
    mask = cv2.normalize(mag, None, 0, 1, cv2.NORM_MINMAX)
    mask = np.stack([mask] * 3, axis=-1)
    img2 = (img1*(1-mask) + blur1*mask).astype(np.uint8)

    # Bước 3: EDUMS – làm sắc nét
    img3 = edums_nhanh_ycbcr(img2, alpha=1.5, sigma=1.5, bgr=True)

    # Bước 4: EICL – khử ringing
    img4 = eicl_sau_edums(img3, nguong_canh=0.08, sigma_smooth=1.2, alpha=0.5, bgr=True)

    # So sánh và ghi kết quả
    compare_images_2(img, img4, nameImg, output_file='Thong_so_co_ban.txt')
```

### Xử lý một ảnh đơn lẻ

```python
import cv2
import numpy as np

nameImg = "1"
img = cv2.imread(f"Dataset/{nameImg}.jpg")

img1 = cv2.resize(img, None, fx=2, fy=2, interpolation=cv2.INTER_CUBIC)
img3 = edums_nhanh_ycbcr(img1, alpha=1.5, sigma=1.5, bgr=True)
img4 = eicl_sau_edums(img3, nguong_canh=0.08, sigma_smooth=1.2, alpha=0.5, bgr=True)

cv2.imwrite(f"Output/{nameImg}_result.png", img4)
```

---

## Ghi Chú

- Pipeline xử lý toàn bộ trong không gian màu **YCbCr**, chỉ thao tác trên kênh **Y (luminance)** để bảo toàn màu sắc gốc.
- Cả hai phương pháp (EDUMS pipeline và IESR) đều được đánh giá song song để so sánh hiệu quả.
- Kết quả đánh giá được ghi tự động vào `Thong_so_co_ban.txt` (EDUMS) và `Thong_so_co_ban_2.txt` (IESR).
