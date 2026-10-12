# Báo cáo: Cấu hình Xác thực GitHub Container Registry (GHCR) qua Repository Secret (Bài 4)

---

## 1. Mục tiêu bài thực hành
- **Bảo mật thông tin xác thực**: Áp dụng quy chuẩn bảo mật DevSecOps bằng cách lưu trữ Token truy cập cá nhân vào GitHub Repository Secrets, loại bỏ hoàn toàn nguy cơ để lộ khóa bí mật trong mã nguồn và kịch bản CI/CD.
- **Tự động hóa đăng nhập Container Registry**: Cấu hình bước xác thực tự động vào **GitHub Container Registry (`ghcr.io`)** trong luồng GitHub Actions CI bằng action chính thức `docker/login-action@v3`.
- **Chuẩn bị nền tảng Continuous Delivery (CD)**: Thiết lập phân quyền và xác thực sẵn sàng cho bước đóng gói và phát hành Docker Image tự động lên GHCR trong các giai đoạn tiếp theo.

---

## 2. Các yêu cầu & Giải pháp kỹ thuật

| Yêu cầu | Giải pháp thực hiện | Ghi chú kỹ thuật |
| :--- | :--- | :--- |
| **Personal Access Token (PAT)** | Tạo Token classic với quyền `write:packages` | Scope `write:packages` tự động bao gồm quyền đọc (`read:packages`) |
| **Bảo mật Secret** | Đặt tên bí mật là `REGISTRY_TOKEN` trong Repository Secrets | Truy cập trong workflow qua cú pháp `${{ secrets.REGISTRY_TOKEN }}` |
| **Kịch bản CI (`ci.yml`)** | Sử dụng action `docker/login-action@v3` với `registry: ghcr.io` | Tận dụng `${{ github.actor }}` làm username để tự động hóa hoàn toàn |
| **Kiểm tra & Xác minh** | Quan sát tab Actions trên GitHub, kiểm tra log chi tiết | Xác nhận kết quả hiển thị dòng chữ `Login Succeeded` |

---

## 3. Cấu hình kịch bản CI (`.github/workflows/ci.yml`)

Nội dung kịch bản CI hoàn chỉnh với bước đăng nhập vào GHCR:

```yaml
name: Java Spring Boot CI (Gradle)

on:
  push:
    branches: [ "main" ]

jobs:
  test:
    name: Run Unit Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Cấp quyền cho Gradle Wrapper
        run: chmod +x gradlew

      - name: Thực thi JUnit Test với Gradle
        run: ./gradlew test

      - name: Đóng gói tệp JAR với Gradle
        run: ./gradlew bootJar

      - name: Lưu trữ thành phẩm JAR Artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: build/libs/*.jar

      - name: Đăng nhập vào GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.REGISTRY_TOKEN }}
```

---

## 4. Hướng dẫn chi tiết các bước triển khai

### Bước 1: Khởi tạo Personal Access Token (Classic)
1. Đăng nhập vào tài khoản GitHub cá nhân, nhấn vào ảnh đại diện góc trên cùng bên phải -> chọn **Settings**.
2. Cuộn menu bên trái xuống cuối cùng, chọn **Developer settings**.
3. Chọn mục **Personal access tokens** -> **Tokens (classic)**.
4. Nhấn **Generate new token** -> chọn **Generate new token (classic)**.
5. Cấu hình thông tin token:
   - **Note**: `IT209_SS11_EX4_GHCR_TOKEN` (hoặc tên gợi nhớ mục đích).
   - **Expiration**: Chọn thời hạn phù hợp (ví dụ: `30 days` hoặc `90 days`).
   - **Select scopes**: Tích chọn vào mục **`write:packages`** (Hệ thống sẽ tự động tích kèm quyền đọc `read:packages: upload and download packages`).
6. Nhấn nút xanh **Generate token** ở cuối trang.
7. **Sao chép và lưu trữ mã token** (chuỗi ký tự bắt đầu bằng `ghp_...`) vào nơi an toàn. *Lưu ý: GitHub chỉ hiển thị token này một lần duy nhất.*

---

### Bước 2: Thiết lập Repository Secret `REGISTRY_TOKEN`
1. Truy cập vào kho lưu trữ bài thực hành trên GitHub (ví dụ: repository `IT209_SS11_Ex4`).
2. Nhấn vào tab **Settings** của repository.
3. Ở menu bên trái, tìm mục **Security** -> chọn **Secrets and variables** -> bấm vào **Actions**.
4. Tại phần **Repository secrets**, nhấn nút **New repository secret**.
5. Nhập thông tin:
   - **Name**: `REGISTRY_TOKEN` *(Chính xác từng ký tự in hoa theo đúng yêu cầu bài học)*.
   - **Secret**: Dán chuỗi Personal Access Token (`ghp_...`) vừa tạo ở Bước 1.
6. Nhấn **Add secret** để lưu lại.

---

### Bước 3: Đẩy mã nguồn lên GitHub Repository
Khởi tạo và đẩy mã nguồn lên GitHub:

```bash
# 1. Kiểm tra trạng thái Git
git status

# 2. Thêm toàn bộ tệp vào staging
git add .

# 3. Tạo commit
git commit -m "feat: configure ghcr authentication with docker login action"

# 4. Thêm remote origin (thay URL bằng link repository của bạn)
git remote add origin https://github.com/vtrung77/IT209_SS11_Ex4.git

# 5. Đẩy nhánh main lên GitHub
git branch -M main
git push -u origin main
```

---

### Bước 4: Kiểm tra và xác nhận kết quả (Verification)
1. Mở trang repository trên GitHub, chuyển sang tab **Actions**.
2. Nhấn vào luồng workflow vừa kích hoạt (commit `feat: configure ghcr authentication...`).
3. Bấm vào chi tiết job **Run Unit Tests**.
4. Mở rộng bước **Đăng nhập vào GitHub Container Registry**:
   - Kiểm tra log hiển thị:
     ```text
     Run docker/login-action@v3
       with:
         registry: ghcr.io
         username: vtrung77
         password: ***
     Logging into ghcr.io...
     Login Succeeded
     ```
   - Nhận diện các điểm quan trọng:
     + Token bí mật được tự động che giấu bằng `***` (masking) để chống lộ mật khẩu trong log.
     + Registry hiển thị chính xác `ghcr.io`.
     + Dòng xác nhận thành công hiển thị: **`Login Succeeded`**.

---

## 5. Cơ chế bảo mật và Nguyên lý hoạt động

### 5.1. Tại sao cần sử dụng GitHub Repository Secrets?
- **Nguyên tắc "Zero Secret in Code"**: Kịch bản CI/CD được lưu trực tiếp dưới dạng mã nguồn (`.github/workflows/ci.yml`). Nếu hardcode trực tiếp token/mật khẩu, bất kỳ ai có quyền đọc kho lưu trữ (hoặc khi repo được public) đều có thể đánh cắp và chiếm quyền truy cập registry.
- **Mã hóa một chiều an toàn (Asymmetric Encryption)**: GitHub sử dụng thư viện [Libsodium (NaCl sealed box)](https://libsodium.gitbook.io/doc/public-key_cryptography/sealed_boxes) để mã hóa secret trước khi lưu vào cơ sở dữ liệu. Sau khi lưu, ngay cả chủ sở hữu repository cũng không thể xem lại giá trị thô (Plaintext).
- **Log Masking tự động**: Trong quá trình Runner thực thi, môi trường GitHub Actions tự động tìm kiếm và che giấu (mask) giá trị của mọi secret thành `***`, bảo vệ dữ liệu bí mật khỏi việc vô tình in ra console/log.

### 5.2. Sự khác biệt giữa `REGISTRY_TOKEN` (PAT) và `${{ secrets.GITHUB_TOKEN }}`
- `${{ secrets.GITHUB_TOKEN }}`: Là token tạm thời do GitHub Actions tự động cấp phát cho mỗi lần chạy workflow, vòng đời gắn liền với thời gian chạy job.
- `REGISTRY_TOKEN` (PAT classic): Là token cấp quyền độc lập của tài khoản lập trình viên, có thể duy trì quyền `write:packages` linh hoạt, kiểm soát phạm vi truy cập (scope) rõ ràng, và phù hợp cho các quy trình CI/CD tích hợp đa dự án hoặc tải lên/quản lý package giữa các tổ chức/kho chứa.

---

## 6. Hình ảnh minh chứng kết quả (Screenshots)

### 6.1. Ảnh chụp Repository Secret `REGISTRY_TOKEN` đã khởi tạo
> Minh chứng biến `REGISTRY_TOKEN` đã được lưu thành công trong mục **Settings -> Secrets and variables -> Actions** của Repository.

![Repository Secret REGISTRY_TOKEN](image.png)

---

### 6.2. Ảnh chụp log bước Đăng nhập Registry hiển thị "Login Succeeded"
> Minh chứng bước `Đăng nhập vào GitHub Container Registry` hoàn tất với thông báo `Login Succeeded`.

![Log Actions Login Succeeded](image%20copy.png)
