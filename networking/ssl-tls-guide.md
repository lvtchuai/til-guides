# 🔐 Toàn tập về SSL/TLS: Nguyên lý hoạt động và Hướng dẫn thiết lập

Bài viết này sẽ giải thích cặn kẽ bản chất của SSL/TLS, cách hai máy tính "bắt tay" nhau để thiết lập kênh truyền an toàn, và hướng dẫn từng bước cấu hình SSL trên thực tế cho web server cũng như môi trường phát triển ứng dụng.

---

## 1. SSL/TLS là gì và tại sao lại quan trọng?

* **SSL (Secure Sockets Layer)** và phiên bản nâng cấp của nó là **TLS (Transport Layer Security)** là các giao thức bảo mật tiêu chuẩn giúp mã hóa liên kết giữa web server và trình duyệt.
* **Mục đích:** * **Mã hóa (Encryption):** Đảm bảo dữ liệu truyền đi (mật khẩu, thẻ tín dụng, API payload) không bị đọc trộm bởi bên thứ ba (Man-in-the-Middle attack).
    * **Xác thực (Authentication):** Đảm bảo client đang giao tiếp đúng với server thật chứ không phải một server giả mạo.
    * **Toàn vẹn dữ liệu (Data Integrity):** Đảm bảo dữ liệu không bị thay đổi trên đường truyền.

---

## 2. SSL/TLS hoạt động như thế nào? (Bản chất kỹ thuật)

Để hiểu SSL, chúng ta cần nắm vững hai khái niệm mã hóa cốt lõi mà SSL sử dụng kết hợp:

1.  **Mã hóa bất đối xứng (Asymmetric Encryption):** Sử dụng một cặp khóa (Public Key và Private Key). Dữ liệu mã hóa bằng Public Key chỉ có thể giải mã bằng Private Key. Cách này rất an toàn để xác thực nhưng tốc độ mã hóa chậm.
2.  **Mã hóa đối xứng (Symmetric Encryption):** Cả hai bên dùng chung một khóa bí mật (Session Key) để mã hóa và giải mã. Cách này tốc độ rất nhanh, phù hợp để truyền lượng dữ liệu lớn.

### Quá trình SSL Handshake (Bắt tay SSL)
Đây là cách client và server thống nhất phương thức bảo mật trước khi truyền dữ liệu thực tế. Quá trình này diễn ra trong vài mili-giây:

1.  **Client Hello:** Trình duyệt (Client) gửi thông điệp chào hỏi đến Server, kèm theo danh sách các bộ mã hóa (Cipher Suites) và phiên bản TLS mà nó hỗ trợ.
2.  **Server Hello:** Server phản hồi lại, chọn một bộ mã hóa mạnh nhất mà cả hai cùng hỗ trợ.
3.  **Trao đổi Chứng chỉ (Certificate Exchange):** Server gửi cho Client **Chứng chỉ số (Digital Certificate)** của nó. Trong chứng chỉ này có chứa **Public Key** của server và chữ ký của tổ chức cấp phát (CA - Certificate Authority).
4.  **Xác thực CA:** Client kiểm tra chứng chỉ này xem có hợp lệ, còn hạn và được ký bởi một CA đáng tin cậy không.
5.  **Tạo khóa chung (Key Generation):** * Nếu chứng chỉ hợp lệ, Client tạo ra một chuỗi ngẫu nhiên gọi là *Pre-master secret*.
    * Client dùng **Public Key** của server để mã hóa chuỗi này và gửi lại cho server.
    * Server dùng **Private Key** của mình để giải mã và lấy được *Pre-master secret*.
6.  **Hoàn tất & Truyền dữ liệu:** Cả Client và Server dùng *Pre-master secret* này để tự tính toán ra một **Session Key (Khóa đối xứng)**. Từ lúc này trở đi, mọi dữ liệu HTTP truyền qua lại sẽ được mã hóa bằng Session Key này với tốc độ rất nhanh.

---

## 3. Thực hành 1: Thiết lập SSL miễn phí cho Web Server (Nginx + Let's Encrypt)

Đây là kịch bản phổ biến nhất khi triển khai (deploy) ứng dụng lên môi trường Production. Chúng ta sẽ dùng Certbot để lấy chứng chỉ từ Let's Encrypt hoàn toàn miễn phí.

**Yêu cầu:** Máy chủ Ubuntu, đã cài Nginx và domain `yourdomain.com` đã trỏ về IP của máy chủ.

**Bước 1: Cài đặt Certbot và Nginx plugin**
```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx
```


**Bước 2: Yêu cấu cấp phát chứng chỉ SSL**

Certbot sẽ tự động tìm cấu hình server block trong Nginx của bạn để xác thực.
```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Hệ thống sẽ hỏi bạn một số thông tin (Email, đồng ý điều khoản) và hỏi bạn có muốn tự động redirect HTTP (port 80) sang HTTPS (port 443) hay không. Nên chọn có (Redirect).

**Bước 3: Kiểm tra tự động gia hạn**
```bash
sudo certbot renew --dry-run
```

---

## 4. Thực hành 2: Tạo Local CA và cấp phát SSL "chuẩn xịn" cho Localhost

Khi code ở local, nếu chỉ tạo chứng chỉ tự ký (self-signed) thông thường, trình duyệt hiện đại sẽ luôn cảnh báo không an toàn do thiếu thông tin SAN (Subject Alternative Name). 

Dưới đây là phương pháp chuyên nghiệp hơn: Chúng ta sẽ tự đóng vai trò là một Tổ chức cấp phát (Local CA), sau đó dùng CA này để ký chứng chỉ cho `localhost`.

### Bước 1: Tạo Root CA của riêng bạn
Đầu tiên, tạo một Private Key và chứng chỉ Root CA. (File `CA.pem` này sau này có thể thêm vào máy tính để trình duyệt tin tưởng mọi chứng chỉ do bạn tạo ra).
```bash
# Tạo Private Key cho CA
openssl genrsa -out CA.key 2048

# Tạo Root Certificate cho CA (Có giá trị 10 năm)
openssl req -x509 -sha256 -new -nodes -days 3650 -key CA.key -out CA.pem
```

### Bước 2: Tạo Yêu cầu cấp chứng chỉ (CSR) cho Localhost
Tiếp theo, tạo Private Key cho server localhost và một file yêu cầu cấp chứng chỉ (CSR - Certificate Signing Request).
```bash
# Tạo Private Key cho localhost
openssl genrsa -out localhost.key 2048

# Tạo file CSR
openssl req -new -key localhost.key -out localhost.csr
```

### Bước 3: Cấu hình SAN (Subject Alternative Name)
Tạo một file tên là localhost.ext và dán nội dung sau vào. Việc cấu hình DNS và IP trong này là bắt buộc để các trình duyệt hiện đại chấp nhận chứng chỉ.
```.cnf
authorityKeyIdentifier = keyid,issuer
basicConstraints = CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
IP.1 = 127.0.0.1
```

### Bước 4: Ký chứng chỉ SSL bằng Local CA
Sử dụng CA đã tạo ở Bước 1 để duyệt file yêu cầu (.csr) ở Bước 2, kết hợp với cấu hình mở rộng (.ext) ở Bước 3:
```bash
openssl x509 -req -in localhost.csr -CA CA.pem -CAkey CA.key -CAcreateserial -days 365 -sha256 -extfile localhost.ext -out localhost.crt
```

### Bước 5: Áp dụng vào Web Server / Ứng dụng
#### Tùy chọn A: Dùng cho Reverse Proxy (Nginx)
Cấu hình dưới đây sẽ ép mọi luồng traffic từ HTTP (port 80) sang HTTPS (port 443) và trỏ đến các file chứng chỉ vừa tạo.
```nginx
server {
  listen 80;
  server_name localhost;
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl;
  server_name localhost;
  
  # Trỏ đường dẫn tới file CRT và KEY vừa tạo
  ssl_certificate     /etc/nginx/ssl-certificate/localhost.crt;
  ssl_certificate_key /etc/nginx/ssl-certificate/localhost.key;
  
  location / {
    root /var/www/html;
    index index.html index.htm;
  }
}
```

#### Tùy chọn B: Dùng cho môi trường Code Backend (Ví dụ: NestJS)
Nếu bạn code server Node.js/NestJS và muốn chạy trực tiếp bằng file này, chỉ cần khai báo trong file bootstrap:
```typescript
import * as fs from 'fs';

const httpsOptions = {
  key: fs.readFileSync('./path/to/localhost.key'),
  cert: fs.readFileSync('./path/to/localhost.crt'),
};
const app = await NestFactory.create(AppModule, { httpsOptions });
```

---

## 5. Tổng kết
SSL/TLS là lớp khiên bảo vệ bắt buộc cho mọi ứng dụng web hiện đại.

Nắm vững cơ chế Handshake giúp bạn dễ dàng debug các lỗi liên quan đến kết nối mạng hoặc cấu hình load balancer/reverse proxy.

Bảo mật không chỉ nằm ở code, mà còn nằm ở tầng hạ tầng (Infrastructure). Việc tự tay cấu hình được chứng chỉ số là một kỹ năng quan trọng của bất kỳ kỹ sư hệ thống hay Full-stack Developer nào.


---
---

# Updates

## 6. TLS 1.3 — Phiên bản hiện đại và cách nó cải thiện mọi thứ

TLS 1.3 (RFC 8446, 2018) là bản nâng cấp lớn nhất của TLS kể từ khi ra đời. Phần Handshake ở Mục 2 mô tả TLS 1.2 truyền thống với RSA key exchange. Thực tế hiện đại đã chuyển sang TLS 1.3, và đây là những gì thay đổi.

### 6.1. Handshake TLS 1.3 — Chỉ 1 RTT thay vì 2

TLS 1.2 cũ cần **2 round-trip** trước khi truyền dữ liệu thực:

```
TLS 1.2 (2 RTT):
Client Hello        ──────────────────────────────>  RTT 1
                  <────────────────────────────── Server Hello
                  <────────────────────────────── Certificate
                  <────────────────────────────── ServerKeyExchange
                  <────────────────────────────── ServerHelloDone
ClientKeyExchange    ──────────────────────────────>  RTT 2
ChangeCipherSpec     ──────────────────────────────>
Finished             ──────────────────────────────>
                  <────────────────────────────── ChangeCipherSpec
                  <────────────────────────────── Finished
[Encrypted data flows]
```

TLS 1.3 tối ưu bằng cách gửi **key ngay trong Client Hello**:

```
TLS 1.3 (1 RTT):
Client Hello + key_share  ───────────────────────────>  RTT 1
                        <─────────────────────────── Server Hello
                        <─────────────────────────── {EncryptedExtensions}
                        <─────────────────────────── {Certificate}
                        <─────────────────────────── {CertificateVerify}
                        <─────────────────────────── {Finished}
{Finished}                ───────────────────────────>
[Encrypted data flows]
```

→ Giảm **50% latency** ngay từ handshake. Với website, điều này có thể giúp load time nhanh hơn đáng kể.

### 6.2. Chỉ còn (EC)DHE — Không còn RSA key exchange

Phần 2 bước 5 mô tả cách Client tạo Pre-master secret rồi mã hóa bằng Public Key của server để gửi sang. Cách này có một vấn đề nghiêm trọng:

- Nếu Private Key của server **bị leak sau này**, kẻ tấn công có thể giải mã **tất cả các phiên đã record lại** trước đó (vì dữ liệu được bảo vệ bằng key đó)
- → **Không có Forward Secrecy**

TLS 1.3 **loại bỏ hoàn toàn RSA key exchange**, chỉ chấp nhận **(EC)DHE** (Ephemeral Diffie-Hellman):

- Mỗi phiên, cả Client và Server đều tự sinh một **cặp khóa tạm** (ephemeral key pair) dùng một lần
- Session key được xây dựng từ hai ephemeral key pairs này — **không có đường truyền nào được bảo vệ bằng private key dài hạn**
- Khi phiên kết thúc, ephemeral key bị destroy
- → **Có Forward Secrecy mặc định** — dù key server có bị leak, các phiên cũ vẫn an toàn

```
TLS 1.3 — (EC)DHE key exchange:

Client sinh ephemeral key pair (client_keyshare)
Server sinh ephemeral key pair (server_keyshare)
Client gửi client_keyshare ngay trong Client Hello
Server phản hồi kèm server_keyshare
Cả hai dùng DH math để tính ra cùng một Session Key
```

### 6.3. Chỉ 5 cipher suite cố định, toàn bộ là AEAD

TLS 1.2 có **hàng chục cipher suites**, bao gồm cả những cipher yếu như 3DES, RC4. Server có thể bị buộc dùng cipher yếu nếu client cũ.

TLS 1.3 chỉ định nghĩa **5 bộ mã hóa duy nhất**, tất cả đều dùng **AEAD** (Authenticated Encryption with Associated Data):

| Cipher Suite | Mã hóa | KEX |
|---|---|---|
| TLS_AES_128_GCM_SHA256 | AES-128-GCM | ECDHE |
| TLS_AES_256_GCM_SHA384 | AES-256-GCM | ECDHE |
| TLS_CHACHA20_POLY1305_SHA256 | ChaCha20-Poly1305 | ECDHE |
| TLS_AES_128_CCM_SHA256 | AES-128-CCM | ECDHE |
| TLS_AES_128_CCM_8_SHA256 | AES-128-CCM-8 | ECDHE |

→ Không còn đàm phán cipher yếu. Server và client hiện đại luôn dùng bộ mã hóa mạnh nhất mà cả hai cùng hỗ trợ.

### 6.4. So sánh TLS 1.2 vs TLS 1.3

| Khía cạnh | TLS 1.2 | TLS 1.3 |
|---|---|---|
| Số RTT | 2 RTT (hoặc 1 với session resumption) | **1 RTT** |
| Key exchange | RSA / (EC)DHE | **Chỉ (EC)DHE** |
| Forward Secrecy | Tùy chọn (phải cấu hình đúng cipher) | **Mặc định bật** |
| Cipher suites | Hàng chục, có cipher yếu (RC4, 3DES) | **5 cố định, toàn AEAD** |
| 0-RTT | Có (nhưng dễ replay attack) | Có (cải thiện hơn) |
| Hỗ trợ trình duyệt | Rất rộng | Tất cả browser hiện đại |
| HTTP/3 support | Không | **Có (QUIC protocol)** |

### 6.5. Kích hoạt TLS 1.3 trên Nginx

```nginx
server {
    listen 443 ssl;
    server_name yourdomain.com;

    ssl_certificate     /etc/ssl/certs/yourdomain.crt;
    ssl_certificate_key /etc/ssl/private/yourdomain.key;

    # Chỉ cho phép TLS 1.2 và 1.3, disable TLS cũ
    ssl_protocols TLSv1.2 TLSv1.3;

    # Ưu tiên cipher suite mạnh
    ssl_ciphers 'TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256';
    ssl_prefer_server_ciphers on;

    # Bật OCSP Stapling (server tự gửi trạng thái certificate)
    ssl_stapling on;
    ssl_stapling_verify on;
}
```

Kiểm tra xem server có hỗ trợ TLS 1.3 không:
```bash
openssl s_client -connect yourdomain.com:443 -tls1_3 < /dev/null 2>&1 | grep "Protocol"
```

---

**Tóm lại:** TLS 1.3 không chỉ là phiên bản "mới hơn" — nó là bản thiết kế lại từ gốc để khắc phục các yếu điểm bảo mật của TLS 1.2, đồng thời giảm đáng kể latency. Handshake 1 RTT, Forward Secrecy mặc định, và bộ cipher cố định là ba thay đổi quan trọng nhất mà kỹ sư hiện đại cần nắm.