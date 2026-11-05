# Bài 6: Ngăn chặn rò rỉ thông tin bí mật trên GitHub

## 1. Bối cảnh

Nhóm em đang làm một dự án thương mại điện tử và chuẩn bị đưa mã nguồn lên GitHub Public để đối tác kiểm tra. Vì repository để công khai nên ai cũng có thể xem code. Nếu vô tình đẩy API Key, mật khẩu cơ sở dữ liệu hoặc token lên đó thì người khác có thể lấy được và sử dụng trái phép.

Theo em, trước khi push, cả nhóm cần kiểm tra mã nguồn, tách thông tin bí mật ra khỏi code và bật xác thực hai lớp cho tài khoản GitHub.

## 2. Cấu trúc repository mẫu

Em xây dựng một repository mẫu có tên `ecommerce-safe-demo` với cấu trúc như sau:

```text
ecommerce-safe-demo/
├── .gitignore
├── .env.example
├── .githooks/
│   └── pre-push
├── README.md
├── config/
│   ├── dev.env.example
│   ├── test.env.example
│   └── production.env.example
├── scripts/
│   └── check_secrets.sh
└── src/
    └── app.py
```

Trong đó:

- `.gitignore`: tránh đưa các file chứa thông tin bí mật và file tạm lên GitHub.
- `.env.example`: hướng dẫn những biến môi trường cần có nhưng **không chứa mật khẩu hoặc API Key thật**.
- `config/`: ví dụ cấu hình riêng cho Dev, Test và Production.
- `scripts/check_secrets.sh`: quét các thông tin nhạy cảm bằng Gitleaks.
- `.githooks/pre-push`: chặn push nếu chưa kiểm tra được hoặc phát hiện dấu hiệu rò rỉ.
- `src/`: chứa mã nguồn mẫu, chỉ đọc cấu hình từ biến môi trường.

## 3. Pre-push Checklist

Trước khi đẩy code lên GitHub, em sẽ thực hiện các bước sau:

- [ ] Kiểm tra code, không viết trực tiếp API Key, password, access token vào file Python, JavaScript hoặc file cấu hình.
- [ ] Đưa dữ liệu bí mật vào biến môi trường hoặc dịch vụ quản lý secrets; chỉ commit file `.env.example` có giá trị mẫu.
- [ ] Kiểm tra `.gitignore` đã bỏ qua `.env`, `.env.production`, file khóa riêng và file thông tin tài khoản dịch vụ.
- [ ] Chạy `git status` và `git diff --cached` để biết chính xác những gì sắp commit.
- [ ] Dùng Gitleaks quét thư mục hiện tại và lịch sử commit bằng `bash scripts/check_secrets.sh`.
- [ ] Kiểm tra quyền truy cập repository, bật 2FA và bật Push Protection nếu có thể.
- [ ] Chỉ push khi không phát hiện dữ liệu nhạy cảm.

Ví dụ lệnh kiểm tra trên Terminal:

```bash
brew install gitleaks  # Chỉ cài một lần trên macOS
bash scripts/check_secrets.sh
git status
git diff --cached
```

Có thể bật hook pre-push khi đang đứng trong repository:

```bash
git config core.hooksPath .githooks
chmod +x .githooks/pre-push
```

**Lưu ý:** Gitleaks hỗ trợ phát hiện nhiều dạng secrets nhưng không bảo đảm tìm được 100%. Vì vậy vẫn cần kiểm tra thủ công. Nếu thiếu Gitleaks thì hook mẫu sẽ không cho push để tránh bỏ qua bước quét.

## 4. Thiết lập `.env.example` theo môi trường

Em không để chung mật khẩu thật trong mã nguồn mà chỉ tạo file mẫu.

**File `.env.example` (mẫu chung):**

```dotenv
APP_ENV=dev
APP_PORT=8000
DATABASE_URL=replace_with_database_url
PAYMENT_API_KEY=replace_with_payment_api_key
JWT_SECRET=replace_with_jwt_secret
```

Bên cạnh đó có các file mẫu riêng:

| Môi trường | File ví dụ | Cách sử dụng |
|---|---|---|
| Dev | `config/dev.env.example` | Dùng khi lập trình trên máy cá nhân, kết nối cơ sở dữ liệu phát triển. |
| Test | `config/test.env.example` | Dùng dữ liệu thử nghiệm, không dùng tài khoản thanh toán thật. |
| Production | `config/production.env.example` | Dùng cấu hình triển khai chính thức, secrets thật phải được cấp qua hệ thống bảo mật. |

Khi chạy trên máy cá nhân, mỗi người có thể sao chép file mẫu thành `.env` rồi điền giá trị của riêng mình. File `.env` phải được bỏ qua bởi `.gitignore` và không được đưa lên GitHub.

## 5. Hướng dẫn bật 2FA cho cả nhóm

Mỗi thành viên vào **GitHub → Settings → Password and authentication → Two-factor authentication**, sau đó làm theo hướng dẫn để bật xác thực bằng ứng dụng Authenticator, passkey hoặc security key. Mã khôi phục nên lưu riêng ở nơi an toàn, không lưu công khai trong repository.

Nếu dự án thuộc một **GitHub Organization**, người quản lý có thể vào **Organization → Settings → Authentication security** và bật **Require two-factor authentication for everyone in your organization**. Trước khi bật, cần báo cho mọi người chuẩn bị vì thành viên hoặc cộng tác viên chưa đáp ứng yêu cầu có thể bị hạn chế hoặc mất quyền truy cập.

Nếu repository chỉ thuộc tài khoản cá nhân, nhóm nên yêu cầu từng cộng tác viên tự bật 2FA và kiểm tra trước khi cấp quyền, thay vì nghĩ rằng có một nút bắt buộc 2FA ngay trong mọi repository cá nhân.

## 6. Đánh giá rủi ro nếu thiếu `.gitignore`

| Rủi ro | Mức độ | Hậu quả có thể xảy ra |
|---|---|---|
| File `.env` bị push lên GitHub Public | Rất cao | Lộ mật khẩu database, API Key, JWT Secret. |
| Khóa truy cập dịch vụ đám mây bị lộ | Rất cao | Kẻ xấu có thể dùng tài nguyên và gây thiệt hại tiền bạc. |
| Thông tin thanh toán bị lộ | Rất cao | Có nguy cơ bị gọi API trái phép và ảnh hưởng đối tác. |
| File log hoặc dữ liệu thử nghiệm bị đưa lên | Trung bình–cao | Có thể lộ thông tin người dùng và cấu trúc hệ thống. |
| File cache, file tạm bị đẩy lên | Thấp–trung bình | Repository lộn xộn, dễ vô tình lộ thêm dữ liệu. |

Theo em, `.gitignore` là lớp bảo vệ cơ bản nhưng chưa đủ. Git vẫn có thể theo dõi file đã commit từ trước, kể cả sau khi thêm vào `.gitignore`.

**Nếu lỡ push API Key thật:** phải thu hồi hoặc thay khóa ngay, kiểm tra lịch sử truy cập và thông báo cho người quản lý. Sau đó mới xử lý file và lịch sử Git nếu cần; chỉ xóa file khỏi commit mới nhất thì không làm khóa cũ hết nguy hiểm.

## 7. Cách đưa repository mẫu lên GitHub

Đầu tiên tạo một repository trống trên GitHub với tên `ecommerce-safe-demo`. Sau đó mở Terminal trong thư mục mẫu và chạy:

```bash
git init
git branch -M main
git config core.hooksPath .githooks
chmod +x .githooks/pre-push
bash scripts/check_secrets.sh
git add .gitignore .env.example .githooks README.md config scripts src
git diff --cached
git commit -m "Create safe ecommerce demo repository"
git remote add origin https://github.com/USERNAME/ecommerce-safe-demo.git
git push -u origin main
```

Thay `USERNAME` bằng tên tài khoản GitHub của mình. Chỉ chạy lệnh push sau khi đã cài Gitleaks và quét thành công. Vì chưa có thông tin đăng nhập GitHub của nhóm nên đây là **repository mẫu có thể tải lên**, không phải đường dẫn tới repository đã được đăng công khai.

