# 🚀 Từ Zero đến Deploy: Khởi tạo Ubuntu Server, cài đặt Nginx và Cấu hình Firewall

Bài viết này là cẩm nang bỏ túi dành cho việc khởi tạo một máy chủ Linux (Ubuntu) mới tinh từ nhà cung cấp (VPS/Cloud) và biến nó thành một Web Server an toàn, sẵn sàng chạy các dự án thực tế. 

Quy trình này áp dụng các Best Practices về bảo mật hệ thống, giúp bạn tránh được những rủi ro cơ bản khi đưa server ra môi trường Internet.

---

## Bước 1: Kết nối vào Server lần đầu (SSH)

Khi mới thuê một VPS (từ DigitalOcean, AWS, Linode...), bạn sẽ nhận được địa chỉ IP và mật khẩu của tài khoản `root`. Hãy mở Terminal và kết nối vào server:

```bash
ssh root@<địa_chỉ_IP_của_server>
```
*Hệ thống sẽ hỏi mật khẩu, hãy nhập mật khẩu bạn được cấp (khi gõ mật khẩu trên Linux sẽ không hiện ra ký tự nào, cứ gõ xong rồi bấm Enter).*

---

## Bước 2: Cập nhật hệ thống và Bảo mật cơ bản

**1. Cập nhật các gói phần mềm (Packages)**
Ngay khi đăng nhập, việc đầu tiên là phải đảm bảo hệ điều hành của bạn có các bản vá bảo mật mới nhất:
```bash
apt update && apt upgrade -y
```

**2. Tạo User mới (Best Practice)**
Tuyệt đối **không nên** dùng tài khoản `root` để thao tác hàng ngày hay chạy ứng dụng, vì `root` có quyền sinh sát tối cao, nếu gõ nhầm lệnh hoặc bị hack thì hậu quả rất nghiêm trọng. Chúng ta sẽ tạo một user mới (ví dụ: `tienadmin`) và cấp quyền `sudo` cho nó:

```bash
# Tạo user mới (hệ thống sẽ yêu cầu bạn nhập mật khẩu cho user này)
adduser tienadmin

# Cấp quyền sudo (quyền quản trị viên) cho user vừa tạo
usermod -aG sudo tienadmin
```

Từ bây giờ, hãy ngắt kết nối (`exit`) và đăng nhập lại bằng user mới:
```bash
ssh tienadmin@<địa_chỉ_IP_của_server>
```

---

## Bước 3: Cài đặt Web Server (Nginx)

Nginx (đọc là *Engine-X*) là một trong những web server và reverse proxy phổ biến, mạnh mẽ và nhẹ nhất hiện nay.

```bash
# Cài đặt Nginx (nhớ dùng sudo vì chúng ta không còn là root nữa)
sudo apt install nginx -y
```

Sau khi cài xong, Nginx sẽ tự động chạy. Bạn có thể kiểm tra trạng thái của nó:
```bash
sudo systemctl status nginx
```
*(Nếu thấy chữ `active (running)` màu xanh lá là thành công. Bấm `q` để thoát màn hình status).*

---

## Bước 4: Thiết lập Tường lửa (UFW - Uncomplicated Firewall)

Server mới ra mạng Internet sẽ bị quét (scan) liên tục bởi các bot độc hại. Tường lửa giúp chặn mọi cổng (port) không cần thiết và chỉ mở những cổng chúng ta cho phép. Ubuntu có sẵn công cụ `ufw` cực kỳ dễ dùng.

**1. Cho phép các cổng cần thiết:**
Chúng ta cần mở port cho SSH (để còn remote vào), HTTP (web bình thường) và HTTPS (web có SSL).

```bash
# Cho phép kết nối SSH (Port 22) - RẤT QUAN TRỌNG, quên cái này là bạn tự khóa mình ở ngoài!
sudo ufw allow OpenSSH

# Cho phép Nginx (mở cả port 80 HTTP và 443 HTTPS)
sudo ufw allow 'Nginx Full'
```

**2. Kích hoạt Firewall:**
```bash
sudo ufw enable
```
*Nhấn `y` khi được hỏi. Để kiểm tra lại các rule đã cài đặt, gõ: `sudo ufw status`.*

Lúc này, nếu bạn copy địa chỉ IP của server và dán vào trình duyệt, bạn sẽ thấy trang **"Welcome to nginx!"** huyền thoại.

---

## Bước 5: Trỏ Tên miền về Server (Cấu hình DNS)

Để người dùng truy cập bằng tên miền (ví dụ: `yourdomain.com`) thay vì những con số IP khó nhớ, bạn cần cấu hình DNS.

1. Đăng nhập vào trang quản lý tên miền của bạn (Nơi bạn mua domain).
2. Tìm đến phần **Quản lý DNS** (DNS Management / DNS Records).
3. Thêm một bản ghi mới với thông tin sau:
   * **Type (Loại):** `A`
   * **Name/Host:** `@` (hoặc để trống tùy giao diện, đại diện cho domain gốc).
   * **Value/Points to (Trỏ về):** Nhập `địa chỉ IP` server của bạn.
   * **TTL:** Để mặc định (Auto / 300 / 3600).
4. Thêm một bản ghi nữa cho sub-domain `www`:
   * **Type:** `A`
   * **Name/Host:** `www`
   * **Value:** Nhập `địa chỉ IP` server của bạn.

*Lưu ý: Bạn có thể sẽ phải đợi từ 5 phút đến vài tiếng để hệ thống DNS cập nhật trên toàn cầu.*

---

## Tổng kết

Chúc mừng! Bạn đã có một chiếc Server vững chãi, một Web Server đang chạy và một Tên miền đã được liên kết. Server của bạn hiện tại đã sẵn sàng để:
1. Đẩy mã nguồn website tĩnh (HTML/CSS/JS) lên để chạy trực tiếp.
2. Cấu hình Reverse Proxy cho các ứng dụng Backend (Node.js, Python...).
3. **Nâng cấp bảo mật bằng chứng chỉ SSL/TLS.**

👉 **Bước tiếp theo:** Hãy đọc tiếp bài **[Toàn tập về SSL/TLS: Nguyên lý hoạt động và Hướng dẫn thiết lập](https://github.com/lvtchuai/til-guides/blob/main/networking/ssl-tls-guide.md)** để trang bị ổ khóa xanh (HTTPS) cho website của bạn!