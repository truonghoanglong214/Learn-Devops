# Phase 6 — Nginx & Reverse Proxy

## Mục tiêu
- Đặt một "cổng vào" duy nhất cho hệ thống, ẩn các service phía sau.
- Phục vụ HTTPS, định tuyến request đến đúng service.

---

## Kiến thức sẽ học

- **Reverse proxy là gì & vì sao cần** — bảo mật, định tuyến, một entrypoint.
- **Cấu hình Nginx** (server block, location, upstream, proxy_pass).
- **Phục vụ static frontend + proxy API tới backend** — kiến trúc web điển hình.
- **SSL/TLS & HTTPS** (chứng chỉ, Let's Encrypt/Certbot) — bắt buộc cho production.
- **Load balancing cơ bản** (round robin, upstream nhiều backend).
- **Rate limiting, gzip, caching, security header** — tối ưu & bảo mật.

---

## Kiến thức chi tiết

### 1. Reverse Proxy Là Gì

#### Forward Proxy vs Reverse Proxy

**Forward Proxy** là proxy đứng giữa client và internet, ẩn danh tính của client:

```
Client --> Forward Proxy --> Internet
```

- Client biết mình đang dùng proxy.
- Internet (server đích) chỉ thấy địa chỉ IP của proxy, không thấy IP client thật.
- Dùng để: vượt tường lửa, ẩn danh, kiểm soát truy cập nội bộ (corporate proxy), cache nội dung.
- Ví dụ thực tế: Squid Proxy, VPN công ty, Tor.

**Reverse Proxy** là proxy đứng giữa internet và các server nội bộ, ẩn danh tính của servers:

```
Internet --> Reverse Proxy --> Backend Servers
```

- Client không biết có bao nhiêu server phía sau.
- Client chỉ giao tiếp với địa chỉ của reverse proxy.
- Dùng để: load balancing, SSL termination, caching, bảo vệ server nội bộ.
- Ví dụ thực tế: Nginx, HAProxy, Traefik, AWS ALB.

#### Lợi ích của Reverse Proxy

**Single Entry Point (Cổng vào duy nhất):**
- Toàn bộ traffic từ internet đi qua một điểm duy nhất.
- Dễ kiểm soát, monitor, log tập trung.
- Không cần expose nhiều port ra internet.
- Client chỉ cần biết một địa chỉ (domain), không cần biết cấu trúc nội bộ.

**SSL Termination (Kết thúc SSL tại proxy):**
- Reverse proxy xử lý toàn bộ việc mã hóa/giải mã SSL/TLS.
- Các server nội bộ giao tiếp qua HTTP thuần (không cần SSL), giảm tải CPU.
- Quản lý chứng chỉ SSL tập trung tại một chỗ thay vì mỗi server một chứng chỉ.
- Dễ renew, rotate chứng chỉ mà không ảnh hưởng backend.

**Load Balancing (Cân bằng tải):**
- Phân phối request đến nhiều server backend.
- Tự động phát hiện server bị lỗi và ngừng gửi request đến đó.
- Hỗ trợ nhiều thuật toán: round robin, least connections, ip_hash...
- Scale horizontal dễ dàng bằng cách thêm server vào upstream pool.

**Caching (Bộ nhớ đệm):**
- Cache response từ backend, giảm số lần phải gọi đến backend.
- Giảm latency cho client.
- Giảm tải cho backend server.
- Nginx có thể cache static files, API responses theo cấu hình.

**DDoS Protection (Bảo vệ DDoS):**
- Rate limiting: giới hạn số request từ một IP trong một khoảng thời gian.
- Connection limiting: giới hạn số kết nối đồng thời từ một IP.
- Whitelist/blacklist IP.
- Kết hợp với các dịch vụ như Cloudflare để chặn DDoS ở tầng mạng.

**Ẩn Kiến Trúc Nội Bộ (Hide Internal Architecture):**
- Client không biết backend dùng ngôn ngữ gì (Node.js, Python, Java...).
- Không biết có bao nhiêu server, địa chỉ IP nội bộ, port nào đang dùng.
- Nếu bị tấn công, attacker chỉ thấy reverse proxy, không thấy backend thật.
- Dễ thay đổi kiến trúc nội bộ mà không ảnh hưởng đến client.

#### So Sánh Nginx vs Apache

| Tiêu chí | Nginx | Apache |
|---|---|---|
| Kiến trúc | Event-driven (async, non-blocking) | Thread-based (mỗi request một thread/process) |
| Concurrency | Xử lý hàng nghìn connections với ít RAM | Mỗi connection tốn một thread (RAM tăng tuyến tính) |
| Static files | Rất nhanh, phục vụ trực tiếp từ disk | Chậm hơn do overhead của module |
| Config syntax | Đơn giản, dạng block | Phức tạp hơn, dùng .htaccess |
| Dynamic content | Cần pass qua FastCGI/upstream | Có module tích hợp (mod_php) |
| Memory usage | Thấp hơn đáng kể | Cao hơn với nhiều connections |
| Market share | Phổ biến hơn cho high-traffic sites | Phổ biến hơn cho shared hosting |
| Use case | Reverse proxy, static files, high concurrency | Shared hosting, .htaccess dynamic config |

**Tại sao Nginx nhanh hơn với static files và concurrent connections:**

Nginx dùng kiến trúc event loop — một process duy nhất có thể xử lý hàng nghìn connections đồng thời nhờ multiplexing I/O (epoll trên Linux). Apache truyền thống (prefork/worker MPM) tạo một thread mới cho mỗi request, dẫn đến "C10K problem" khi có hàng nghìn connections đồng thời.

#### So Sánh Nginx vs Traefik vs Caddy

**Nginx:**
- Phổ biến nhất, có từ năm 2004, cộng đồng lớn nhất.
- Documentation phong phú, nhiều tutorial, StackOverflow answers.
- Config tĩnh (phải reload khi thay đổi).
- Phù hợp khi muốn kiểm soát chi tiết, môi trường on-premise.
- Hạn chế: không tự động discover services trong container environment.

**Traefik:**
- Được thiết kế cho container/microservices (Docker, Kubernetes native).
- Tự động discover services qua Docker labels, Kubernetes annotations.
- Không cần reload khi thêm/xóa service — dynamic configuration.
- Có dashboard UI tích hợp để xem routing rules.
- Tự động HTTPS với Let's Encrypt.
- Phù hợp cho môi trường Kubernetes, nhiều microservices thay đổi thường xuyên.

**Caddy:**
- Tự động HTTPS mặc định — tự lấy và renew chứng chỉ Let's Encrypt.
- Cấu hình đơn giản nhất (Caddyfile dễ đọc hơn nginx.conf).
- Viết bằng Go, binary đơn giản, dễ deploy.
- Phù hợp cho developer cần setup nhanh, dự án nhỏ/vừa.
- Hạn chế: ít tài liệu hơn Nginx, cộng đồng nhỏ hơn.

**Kết luận lựa chọn:**
- Dự án cần kiểm soát chi tiết, team đã quen: **Nginx**
- Microservices trên Kubernetes, nhiều service dynamic: **Traefik**
- Setup nhanh, tự động HTTPS, dự án đơn giản: **Caddy**

---

### 2. Cấu Trúc Nginx Config

#### Tổ Chức File

```
/etc/nginx/
├── nginx.conf              # File config chính
├── mime.types              # Mapping file extension -> MIME type
├── conf.d/                 # Thư mục include thêm config
│   ├── default.conf
│   └── shoplite.conf
├── sites-available/        # Pattern của Debian/Ubuntu
│   ├── shoplite.example.com
│   └── default
├── sites-enabled/          # Symlink đến sites-available (đang active)
│   └── shoplite.example.com -> ../sites-available/shoplite.example.com
└── ssl/                    # Thư mục chứa chứng chỉ SSL
    ├── fullchain.pem
    └── privkey.pem
```

**nginx.conf (main config)** — include các file khác:
```nginx
include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*;
```

**Pattern Debian/Ubuntu (sites-available + sites-enabled):**
- `sites-available/`: chứa tất cả config của các site (kể cả site chưa bật).
- `sites-enabled/`: chứa symlink đến các site đang hoạt động.
- Bật site: `ln -s /etc/nginx/sites-available/site /etc/nginx/sites-enabled/site`
- Tắt site: `rm /etc/nginx/sites-enabled/site`

#### Cấu Trúc Context (Block Hierarchy)

```
main context
├── worker_processes
├── error_log
├── pid
│
├── events context
│   └── worker_connections
│
└── http context
    ├── include mime.types
    ├── log_format
    ├── access_log
    ├── gzip settings
    │
    ├── upstream block (định nghĩa backend pool)
    │   └── server entries
    │
    └── server block (virtual host)
        ├── listen
        ├── server_name
        │
        └── location block
            ├── proxy_pass
            ├── root
            └── try_files
```

**Main Context** — các directive ở cấp cao nhất:
- `worker_processes auto;` — số worker process, `auto` = số CPU cores.
- `error_log /var/log/nginx/error.log warn;` — ghi error log mức warn trở lên.
- `pid /var/run/nginx.pid;` — file lưu PID của master process.

**Events Context** — cấu hình xử lý network events:
- `worker_connections 1024;` — mỗi worker xử lý tối đa 1024 connections đồng thời.
- `use epoll;` — dùng epoll (Linux) thay vì select/poll, hiệu năng cao hơn.
- `multi_accept on;` — mỗi worker chấp nhận nhiều connections một lúc.

**HTTP Context** — cấu hình HTTP protocol:
- Chứa các server blocks (virtual hosts).
- Chứa upstream blocks (định nghĩa backend pools).
- Cấu hình logging, gzip, timeouts áp dụng cho toàn bộ HTTP.

**Server Context** — một virtual host:
- `listen 80;` hoặc `listen 443 ssl;`
- `server_name example.com www.example.com;`
- Chứa các location blocks để route request.

**Location Context** — xử lý từng loại request:
- Match URL pattern với các modifier khác nhau.
- Có thể nest (location trong location).
- `proxy_pass` — forward request đến upstream.
- `root` — serve static files từ thư mục này.
- `try_files` — thử các file/path theo thứ tự.

---

### 3. nginx.conf Đầy Đủ cho ShopLite

File cấu hình hoàn chỉnh cho dự án ShopLite chạy trong Docker Compose:

```nginx
# =============================================================================
# nginx.conf - Cấu hình Nginx cho ShopLite
# Architecture: Nginx (reverse proxy) -> Frontend (React) + Backend (ASP.NET Core)
# =============================================================================

# ---- Main Context ----
worker_processes auto;                          # Tự động = số CPU cores
error_log /var/log/nginx/error.log warn;        # Log lỗi từ mức warn trở lên
pid /var/run/nginx.pid;                         # File lưu PID của master process

# ---- Events Context ----
events {
    worker_connections 1024;    # Tối đa 1024 connections mỗi worker
    use epoll;                  # Dùng epoll cho Linux (hiệu năng cao)
    multi_accept on;            # Chấp nhận nhiều connections cùng lúc
}

# ---- HTTP Context ----
http {
    # MIME types - để browser biết cách xử lý file
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # ---- Logging ----
    # Format log bao gồm: IP, user, thời gian, request, status, bytes, referer, UA, thời gian xử lý
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" $request_time';

    access_log /var/log/nginx/access.log main;

    # ---- Performance Settings ----
    sendfile on;            # Dùng sendfile() syscall để gửi file hiệu quả hơn
    tcp_nopush on;          # Gửi header và file trong cùng một TCP packet
    tcp_nodelay on;         # Tắt Nagle algorithm cho latency thấp hơn
    keepalive_timeout 65;   # Giữ kết nối TCP trong 65 giây
    types_hash_max_size 2048;

    # ---- Gzip Compression ----
    gzip on;
    gzip_vary on;           # Thêm header Vary: Accept-Encoding
    gzip_proxied any;       # Nén cả response từ upstream
    gzip_comp_level 6;      # Mức nén 1-9 (6 là cân bằng giữa tốc độ và tỉ lệ nén)
    gzip_min_length 1024;   # Chỉ nén file lớn hơn 1KB
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;

    # ---- Rate Limiting Zones ----
    # Định nghĩa zone rate limiting (phải đặt trong http context, ngoài server block)
    # Zone "api": theo dõi theo IP nhị phân, lưu 10MB state, giới hạn 10 req/giây
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    # Zone "login": nghiêm ngặt hơn cho endpoint đăng nhập
    limit_req_zone $binary_remote_addr zone=login:10m rate=3r/m;

    # ---- Upstream Definitions ----
    # Pool backend Node.js
    upstream backend {
        # Thuật toán: least_conn (ít connections nhất) - tốt hơn round_robin cho API
        least_conn;

        # Server entries
        server backend:8080;            # Service name trong Docker Compose

        # Với nhiều instances (load balancing):
        # server backend1:8080 weight=3;  # Nhận 3x traffic hơn backend2
        # server backend2:8080 weight=1;
        # server backend3:8080 backup;    # Chỉ dùng khi backend1 và backend2 down

        # Passive health check
        # max_fails=3: sau 3 lần lỗi trong fail_timeout giây thì đánh dấu server down
        # fail_timeout=30s: thời gian đếm lỗi và thời gian server bị đánh dấu down
        # server backend:8080 max_fails=3 fail_timeout=30s;

        # Connection reuse đến backend
        keepalive 32;                   # Giữ 32 persistent connections đến backend
    }

    # ---- Server Block 1: HTTP -> HTTPS Redirect ----
    server {
        listen 80;
        listen [::]:80;     # IPv6
        server_name _;      # Bắt tất cả domain (wildcard)

        # Chuyển hướng toàn bộ HTTP sang HTTPS (301 Permanent)
        return 301 https://$host$request_uri;
    }

    # ---- Server Block 2: HTTPS Main Server ----
    server {
        listen 443 ssl http2;       # SSL + HTTP/2
        listen [::]:443 ssl http2;  # IPv6
        server_name shoplite.example.com;

        # ---- SSL/TLS Configuration ----
        ssl_certificate /etc/nginx/ssl/fullchain.pem;
        ssl_certificate_key /etc/nginx/ssl/privkey.pem;

        # Chỉ cho phép TLS 1.2 và 1.3 (bỏ TLS 1.0/1.1 đã lỗi thời)
        ssl_protocols TLSv1.2 TLSv1.3;

        # Cipher suites mạnh (ECDHE = forward secrecy, GCM = authenticated encryption)
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
        ssl_prefer_server_ciphers off;  # Để client chọn cipher (TLS 1.3 best practice)

        # Session cache để tái sử dụng SSL handshake (giảm latency cho connections sau)
        ssl_session_cache shared:SSL:10m;   # Cache 10MB, shared giữa workers
        ssl_session_timeout 1d;             # Session valid trong 1 ngày
        ssl_session_tickets off;            # Tắt session tickets (bảo mật hơn)

        # OCSP Stapling (giảm thời gian verify chứng chỉ)
        ssl_stapling on;
        ssl_stapling_verify on;
        resolver 8.8.8.8 8.8.4.4 valid=300s;
        resolver_timeout 5s;

        # ---- Security Headers ----
        # HSTS: bắt buộc HTTPS trong 1 năm, bao gồm subdomain
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
        # Không cho phép site được nhúng vào iframe (chống clickjacking)
        add_header X-Frame-Options DENY always;
        # Tắt MIME type sniffing (chống MIME confusion attack)
        add_header X-Content-Type-Options nosniff always;
        # XSS protection (cũ nhưng vẫn hữu ích cho IE/Edge cũ)
        add_header X-XSS-Protection "1; mode=block" always;
        # Kiểm soát referrer information
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;
        # Permissions Policy (tắt các browser feature không dùng)
        add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;

        # Ẩn version Nginx (tránh lộ thông tin)
        server_tokens off;

        # ---- Giới hạn kích thước request ----
        client_max_body_size 10m;       # Upload tối đa 10MB
        client_body_timeout 12s;
        client_header_timeout 12s;

        # ---- Proxy Timeout Settings ----
        proxy_read_timeout 30s;         # Timeout chờ response từ backend
        proxy_connect_timeout 5s;       # Timeout kết nối đến backend
        proxy_send_timeout 10s;         # Timeout gửi request đến backend

        # ---- Proxy Buffers ----
        proxy_buffer_size 4k;           # Buffer cho response header
        proxy_buffers 8 4k;             # 8 buffers x 4KB cho response body
        proxy_busy_buffers_size 8k;

        # ---- Location: Frontend Static Files ----
        location / {
            root /usr/share/nginx/html;

            # SPA fallback: thử file thật, thử directory, fallback về index.html
            # Cần thiết cho React Router, Vue Router (client-side routing)
            try_files $uri $uri/ /index.html;

            # Cache HTML: không cache (luôn kiểm tra phiên bản mới)
            location ~* \.html$ {
                expires -1;
                add_header Cache-Control "no-store, no-cache, must-revalidate";
            }

            # Cache static assets: cache 1 năm (fingerprint trong tên file)
            location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
                expires 1y;
                add_header Cache-Control "public, immutable";
                access_log off;         # Không log request cho static files
            }
        }

        # ---- Location: API Proxy ----
        location /api/ {
            # Rate limiting: tối đa 10 req/giây, burst 20 req, nodelay = không queue
            limit_req zone=api burst=20 nodelay;
            limit_req_status 429;       # Trả về 429 Too Many Requests khi vượt limit

            # Forward request đến upstream backend
            # Lưu ý trailing slash: /api/ -> http://backend/ (strip /api/ prefix)
            proxy_pass http://backend/;

            # Dùng HTTP/1.1 để hỗ trợ keepalive và WebSocket upgrade
            proxy_http_version 1.1;

            # Headers cho WebSocket upgrade
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";

            # Truyền thông tin thật về client đến backend
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Host $server_name;

            # Bỏ header Connection để keepalive hoạt động đúng
            proxy_set_header Connection "";

            # Không cache API responses (trừ khi cấu hình rõ ràng)
            add_header Cache-Control "no-store";
        }

        # ---- Location: Login Endpoint (Rate Limit Nghiêm Ngặt Hơn) ----
        location /api/auth/login {
            limit_req zone=login burst=5 nodelay;
            limit_req_status 429;

            proxy_pass http://backend/auth/login;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # ---- Location: Health Check ----
        location /health {
            access_log off;             # Không log health check requests
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }

        # ---- Location: Nginx Status (chỉ cho phép nội bộ) ----
        location /nginx_status {
            stub_status on;
            allow 127.0.0.1;
            allow 172.16.0.0/12;        # Docker internal network
            deny all;
        }

        # ---- Error Pages ----
        error_page 404 /404.html;
        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
            root /usr/share/nginx/html;
        }
    }
}
```

**Giải thích các phần quan trọng trong config trên:**

Config được chia làm 5 phần chính:
1. **Main + Events**: Cấu hình worker processes và event loop.
2. **HTTP**: Cấu hình chung cho toàn bộ HTTP traffic (logging, gzip, rate limit zones).
3. **Upstream**: Định nghĩa backend pool với health check và keepalive.
4. **Server HTTP**: Redirect tất cả HTTP sang HTTPS.
5. **Server HTTPS**: Virtual host chính với SSL, security headers, và routing.

---

### 4. Location Matching Precedence (Thứ Tự Ưu Tiên)

Nginx xử lý location blocks theo thứ tự ưu tiên, không phải theo thứ tự viết trong file.

#### Các Modifier và Thứ Tự Ưu Tiên

```nginx
# 1. EXACT MATCH (=) - ưu tiên cao nhất
# Chỉ match khi URL khớp chính xác 100%
# Nginx dừng tìm kiếm ngay khi tìm thấy
location = /favicon.ico {
    access_log off;
    return 204;
}

# 2. PREFERENTIAL PREFIX (^~) - ưu tiên thứ hai
# Match theo prefix, nhưng DỪNG tìm kiếm regex nếu match
# Không cho phép regex override sau đó
location ^~ /static/ {
    root /var/www;
    expires 1y;
}

# 3. REGEX CASE-SENSITIVE (~) - ưu tiên thứ ba (chia sẻ với ~*)
# Match theo regular expression, phân biệt hoa thường
location ~ \.php$ {
    fastcgi_pass php-fpm:9000;
}

# 3. REGEX CASE-INSENSITIVE (~*) - ưu tiên thứ ba (chia sẻ với ~)
# Match theo regular expression, KHÔNG phân biệt hoa thường
location ~* \.(jpg|jpeg|png|gif|css|js)$ {
    expires 30d;
    add_header Cache-Control "public";
}

# 4. PREFIX MATCH (không có modifier) - ưu tiên thứ tư
# Nginx tìm prefix dài nhất match, nhưng vẫn tiếp tục kiểm tra regex
location /api/ {
    proxy_pass http://backend;
}

# 5. DEFAULT (/) - ưu tiên thấp nhất
# Bắt tất cả request không match bất kỳ location nào ở trên
location / {
    try_files $uri $uri/ /index.html;
}
```

#### Thuật Toán Tìm Kiếm của Nginx

Nginx tìm location theo 2 bước:

**Bước 1 — Tìm prefix match:**
- Duyệt tất cả location blocks không có modifier và `^~`.
- Ghi nhớ prefix dài nhất match với URL.
- Nếu prefix dài nhất có `^~`, dùng luôn và bỏ qua regex (kết thúc).
- Nếu có `=` match chính xác, dùng luôn và kết thúc.

**Bước 2 — Kiểm tra regex (theo thứ tự viết trong file):**
- Duyệt tất cả location `~` và `~*` theo thứ tự viết trong config.
- Dùng regex đầu tiên match.
- Nếu không có regex nào match, dùng prefix tốt nhất từ bước 1.

#### Ví Dụ Thực Tế

```nginx
# URL: /api/users/123

location = /api/users/123 { }   # Bước 1: exact match? Không (nếu URL khác)
location ^~ /api/             { }   # Bước 1: prefix match ^~ /api/ -> DỪNG regex
location ~ \.json$            { }   # Bị bỏ qua vì ^~ đã match
location /api/                { }   # Bước 1: prefix match /api/
location /                    { }   # Bước 1: prefix match /

# Kết quả: ^~ /api/ được dùng (nếu tồn tại), nếu không thì regex ~ \.json$
```

#### Lỗi Thường Gặp

```nginx
# SAI: Thứ tự này có thể gây bất ngờ
location / {
    proxy_pass http://frontend;
}
location /api/ {
    proxy_pass http://backend;  # Sẽ không hoạt động đúng
}

# ĐÚNG: Specific trước general
location /api/ {
    proxy_pass http://backend;
}
location / {
    proxy_pass http://frontend;
}
```

---

### 5. SSL/TLS với Let's Encrypt

#### Certbot — Công Cụ Lấy Chứng Chỉ Let's Encrypt

**Cài đặt Certbot:**
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install certbot python3-certbot-nginx

# CentOS/RHEL
sudo dnf install certbot python3-certbot-nginx
```

**Lấy chứng chỉ cho domain (với Nginx plugin):**
```bash
# Tự động cấu hình Nginx và lấy chứng chỉ
sudo certbot --nginx -d shoplite.com -d www.shoplite.com

# Chỉ lấy chứng chỉ, không tự động sửa Nginx config
sudo certbot certonly --nginx -d shoplite.com -d www.shoplite.com

# Dùng webroot method (nếu không dùng Nginx plugin)
sudo certbot certonly --webroot -w /var/www/html -d shoplite.com
```

**Renew chứng chỉ:**
```bash
# Test xem renew có hoạt động không (dry run)
sudo certbot renew --dry-run

# Renew tất cả chứng chỉ sắp hết hạn (tự động)
sudo certbot renew

# Renew và reload Nginx sau khi renew thành công
sudo certbot renew --post-hook "systemctl reload nginx"
```

**Tự động renew với Cron:**
```bash
# Thêm vào crontab (chạy 2 lần/ngày, Let's Encrypt khuyến nghị)
# Crontab: minute hour day month weekday command
0 12 * * * /usr/bin/certbot renew --quiet --post-hook "systemctl reload nginx"
0 0 * * * /usr/bin/certbot renew --quiet --post-hook "systemctl reload nginx"

# Hoặc dùng systemd timer (Ubuntu modern)
sudo systemctl status certbot.timer     # Kiểm tra timer có active không
sudo systemctl enable certbot.timer     # Bật auto-renew
```

**Thông tin chứng chỉ:**
```bash
sudo certbot certificates            # Xem tất cả chứng chỉ
sudo certbot delete --cert-name shoplite.com  # Xóa chứng chỉ
```

**Vị trí file chứng chỉ sau khi lấy:**
```
/etc/letsencrypt/live/shoplite.com/
├── fullchain.pem   # Chứng chỉ + intermediate CA chain
├── privkey.pem     # Private key
├── cert.pem        # Chứng chỉ chính (không bao gồm chain)
└── chain.pem       # Intermediate CA certificates
```

#### Self-Signed Certificate cho Local Development

```bash
# Tạo self-signed certificate (hợp lệ 365 ngày)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/privkey.pem \
    -out /etc/nginx/ssl/fullchain.pem \
    -subj "/C=VN/ST=HCM/L=HoChiMinh/O=ShopLite/OU=Dev/CN=localhost"

# Tạo với Subject Alternative Names (SANs) - cần thiết cho Chrome/modern browsers
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/privkey.pem \
    -out /etc/nginx/ssl/fullchain.pem \
    -subj "/C=VN/O=ShopLite Dev/CN=localhost" \
    -extensions v3_req \
    -config <(cat <<EOF
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no
[req_distinguished_name]
C = VN
O = ShopLite Dev
CN = localhost
[v3_req]
subjectAltName = @alt_names
[alt_names]
DNS.1 = localhost
DNS.2 = shoplite.local
IP.1 = 127.0.0.1
EOF
)

# Thêm vào trusted certificates (macOS)
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain \
    /etc/nginx/ssl/fullchain.pem

# Thêm vào trusted certificates (Ubuntu)
sudo cp /etc/nginx/ssl/fullchain.pem /usr/local/share/ca-certificates/shoplite-dev.crt
sudo update-ca-certificates
```

#### Cấu Hình SSL Tối Ưu trong Nginx

```nginx
# SSL Certificate files
ssl_certificate /etc/nginx/ssl/fullchain.pem;
ssl_certificate_key /etc/nginx/ssl/privkey.pem;

# Protocols: Chỉ TLS 1.2 và 1.3 (TLS 1.0 và 1.1 đã deprecated)
ssl_protocols TLSv1.2 TLSv1.3;

# Cipher suites (theo Mozilla Intermediate compatibility)
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;

# Session cache (giảm overhead của SSL handshake)
ssl_session_cache shared:SSL:10m;   # 10MB shared cache
ssl_session_timeout 1d;
ssl_session_tickets off;            # Tắt để tăng security (no forward secrecy for tickets)

# DH parameters cho DHE (tùy chọn, tăng security)
# Tạo: openssl dhparam -out /etc/nginx/ssl/dhparam.pem 4096
ssl_dhparam /etc/nginx/ssl/dhparam.pem;
```

---

### 6. Performance Tuning

#### Gzip Compression

```nginx
http {
    # Bật gzip
    gzip on;

    # Thêm header Vary: Accept-Encoding để CDN cache đúng
    gzip_vary on;

    # Nén cả khi response đến từ proxy upstream
    gzip_proxied any;

    # Mức nén 1-9 (1=nhanh/ít nén, 9=chậm/nén nhiều, 6=cân bằng)
    gzip_comp_level 6;

    # Chỉ nén file > 1KB (file nhỏ không đáng nén)
    gzip_min_length 1024;

    # Buffers
    gzip_buffers 16 8k;

    # HTTP version tối thiểu để nén
    gzip_http_version 1.1;

    # Các MIME type được nén
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        text/x-component
        application/json
        application/javascript
        application/x-javascript
        application/xml
        application/xml+rss
        application/atom+xml
        application/xhtml+xml
        application/x-font-ttf
        application/x-font-opentype
        application/vnd.ms-fontobject
        image/svg+xml
        image/x-icon
        font/opentype;
    # Lưu ý: image/jpeg, image/png đã được nén, không cần thêm
}
```

#### Browser Caching (Cache-Control Headers)

```nginx
server {
    # Static assets có fingerprint trong tên file (webpack, vite)
    # Ví dụ: main.a1b2c3d4.js, styles.e5f6a7b8.css
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;                                     # Cache 1 năm
        add_header Cache-Control "public, immutable";   # immutable = không revalidate
        access_log off;                                 # Giảm log noise
    }

    # HTML files: không cache (luôn lấy phiên bản mới)
    location ~* \.html$ {
        expires -1;
        add_header Cache-Control "no-store, no-cache, must-revalidate";
        add_header Pragma "no-cache";
    }

    # API responses: không cache theo mặc định
    location /api/ {
        add_header Cache-Control "no-store";
        proxy_pass http://backend;
    }

    # Media files có thể cache ngắn hơn (nếu không có fingerprint)
    location ~* \.(mp4|webm|mp3|ogg)$ {
        expires 7d;
        add_header Cache-Control "public";
    }
}
```

#### Connection Reuse (Keepalive to Backend)

```nginx
# Trong upstream block
upstream backend {
    server backend:8080;
    keepalive 32;           # Giữ 32 connections sẵn sàng trong pool
}

# Trong location block proxy đến upstream
location /api/ {
    proxy_pass http://backend;

    # Bắt buộc phải có hai dòng này để keepalive hoạt động
    proxy_http_version 1.1;         # HTTP/1.1 hỗ trợ keepalive
    proxy_set_header Connection ""; # Xóa header Connection từ client (thường là "close")
}
```

Lợi ích của keepalive upstream:
- Giảm overhead TCP 3-way handshake cho mỗi request.
- Giảm overhead SSL handshake nếu backend cũng dùng HTTPS.
- Cải thiện latency đáng kể với high-traffic API.

#### Proxy Buffers

```nginx
server {
    # Buffer cho response header từ backend
    proxy_buffer_size 4k;

    # Number x size của buffers cho response body
    # Tổng: 8 x 4k = 32k buffer
    proxy_buffers 8 4k;

    # Bao nhiêu buffer có thể dùng khi client chậm
    proxy_busy_buffers_size 8k;

    # Giới hạn kích thước request body (upload)
    client_max_body_size 10m;

    # Buffer request body trên disk nếu > threshold
    client_body_buffer_size 128k;
}
```

---

### 7. Load Balancing với Nginx

#### Các Thuật Toán Load Balancing

```nginx
# Round Robin (mặc định) - phân phối đều theo vòng
upstream backend_rr {
    server backend1:8080;
    server backend2:8080;
    server backend3:8080;
}

# Weighted Round Robin - backend1 nhận 3x traffic hơn backend2
upstream backend_weighted {
    server backend1:8080 weight=3;
    server backend2:8080 weight=1;
}

# Least Connections - gửi request đến server ít connections nhất
# Tốt hơn round-robin khi request có thời gian xử lý khác nhau
upstream backend_lc {
    least_conn;
    server backend1:8080;
    server backend2:8080;
    server backend3:8080;
}

# IP Hash - cùng IP luôn được gửi đến cùng server (session persistence)
# Cần khi application không dùng shared session store
upstream backend_iphash {
    ip_hash;
    server backend1:8080;
    server backend2:8080;
}

# Hash by arbitrary key (Nginx Plus hoặc module)
upstream backend_hash {
    hash $request_uri consistent;  # Consistent hashing theo URL
    server backend1:8080;
    server backend2:8080;
}
```

#### Server Parameters

```nginx
upstream backend {
    # weight: trọng số, server nhận nhiều request hơn nếu weight cao hơn
    server backend1:8080 weight=5;

    # max_fails + fail_timeout: passive health check
    # Nếu backend2 fail 3 lần trong 30s -> đánh dấu down trong 30s tiếp
    server backend2:8080 max_fails=3 fail_timeout=30s;

    # backup: chỉ dùng khi tất cả server khác down
    server backend3:8080 backup;

    # down: đánh dấu server tạm thời down (maintenance)
    server backend4:8080 down;

    # max_conns: giới hạn connections đến server này
    server backend5:8080 max_conns=100;

    keepalive 32;
}
```

#### Passive Health Check

Nginx Open Source chỉ hỗ trợ passive health check (kiểm tra khi có request thật):

```nginx
upstream backend {
    server backend1:8080 max_fails=3 fail_timeout=30s;
    server backend2:8080 max_fails=3 fail_timeout=30s;
}
```

- Nếu server trả về lỗi (5xx, timeout) 3 lần trong 30 giây → đánh dấu "unhealthy".
- Sau 30 giây → Nginx thử gửi request đến lại.
- Nếu thành công → đánh dấu "healthy" lại.

Nginx Plus (commercial) hỗ trợ active health check với `health_check` directive.

---

### 8. Nginx trong Docker Compose

#### Dockerfile cho Nginx Service

```dockerfile
# Dùng nginx:alpine để image nhỏ gọn (~25MB thay vì ~140MB)
FROM nginx:alpine

# Copy config file
COPY nginx.conf /etc/nginx/nginx.conf

# Copy SSL certificates (cho production, nên dùng secrets hoặc volume)
COPY ssl/ /etc/nginx/ssl/

# Copy static frontend files (nếu build trong cùng Dockerfile)
# COPY --from=build /app/dist /usr/share/nginx/html

# Validate config khi build (fail fast nếu config sai)
RUN nginx -t

# Expose ports
EXPOSE 80 443

# Chạy nginx ở foreground (cần cho Docker)
CMD ["nginx", "-g", "daemon off;"]
```

#### docker-compose.yml với Nginx

```yaml
version: "3.9"

services:
  # Nginx: entry point duy nhất, expose port ra ngoài
  nginx:
    build:
      context: ./nginx
      dockerfile: Dockerfile
    ports:
      - "80:80"     # HTTP (redirect sang HTTPS)
      - "443:443"   # HTTPS
    volumes:
      # Mount config cho development (hot reload config)
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      # Mount SSL certs
      - ./nginx/ssl:/etc/nginx/ssl:ro
      # Mount frontend build output
      - frontend_build:/usr/share/nginx/html:ro
    depends_on:
      - backend
      - frontend
    networks:
      - app_network
    restart: unless-stopped

  # Frontend: React app build
  frontend:
    build:
      context: ./frontend
    volumes:
      - frontend_build:/app/dist  # Share build output với nginx
    networks:
      - app_network
    # KHÔNG expose port ra ngoài - chỉ nginx mới cần

  # Backend: ASP.NET Core API
  backend:
    build:
      context: ./backend
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ASPNETCORE_URLS=http://+:8080
      - ConnectionStrings__DefaultConnection=Host=db;Port=5432;Database=shoplite;Username=user;Password=pass
    # KHÔNG expose port ra ngoài
    networks:
      - app_network
    restart: unless-stopped

  # Database: chỉ accessible từ internal network
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: shoplite
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app_network
    # KHÔNG expose port ra ngoài

volumes:
  frontend_build:
  postgres_data:

networks:
  app_network:
    driver: bridge
```

**Nguyên tắc thiết kế:**
- Chỉ nginx expose port ra ngoài (80, 443).
- Backend, frontend, database chỉ giao tiếp qua internal Docker network.
- Attacker không thể trực tiếp kết nối đến backend hay database từ internet.

#### Mount Config cho Development

```yaml
# Development: mount config để thay đổi không cần rebuild image
volumes:
  - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro   # :ro = read-only

# Reload config sau khi thay đổi (không restart container)
# docker compose exec nginx nginx -s reload
```

---

### 9. Nginx Management Commands

#### Kiểm Tra và Validate Config

```bash
# Test cú pháp config (không reload)
nginx -t

# Test config và in ra toàn bộ config đã được parse (bao gồm includes)
nginx -T

# Test config của file cụ thể
nginx -t -c /path/to/nginx.conf

# Output khi config hợp lệ:
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful

# Output khi có lỗi:
# nginx: [emerg] unexpected "}" in /etc/nginx/nginx.conf:45
# nginx: configuration file /etc/nginx/nginx.conf test failed
```

#### Reload và Restart

```bash
# Graceful reload: áp dụng config mới, KHÔNG drop existing connections
# Master process load config mới, spawn worker mới, worker cũ tiếp tục xử lý requests đang có
nginx -s reload

# Graceful shutdown: đợi requests hiện tại xong rồi tắt
nginx -s quit

# Tắt ngay lập tức (không graceful)
nginx -s stop

# Reopen log files (sau khi rotate log)
nginx -s reopen

# Trên host với systemd
sudo systemctl reload nginx      # Graceful reload
sudo systemctl restart nginx     # Full restart (có downtime ngắn)
sudo systemctl status nginx      # Xem trạng thái
sudo systemctl start nginx       # Khởi động
sudo systemctl stop nginx        # Dừng
sudo systemctl enable nginx      # Auto-start khi boot
```

#### Quản Lý Trong Docker Compose

```bash
# Test config trong container
docker compose exec nginx nginx -t

# Graceful reload config trong container (không restart container)
docker compose exec nginx nginx -s reload

# Xem logs Nginx
docker compose logs nginx
docker compose logs -f nginx     # Follow logs

# Xem logs access trong container
docker compose exec nginx tail -f /var/log/nginx/access.log
docker compose exec nginx tail -f /var/log/nginx/error.log

# Restart container (nếu cần full restart)
docker compose restart nginx
```

#### Debugging

```bash
# Xem processes Nginx
ps aux | grep nginx

# Xem connections
ss -tlnp | grep nginx

# Xem open files
lsof -p $(cat /var/run/nginx.pid)

# Test kết nối từ trong container đến backend
docker compose exec nginx curl -v http://backend:8080/health

# Xem Nginx status (cần bật stub_status trong config)
curl http://localhost/nginx_status
```

---

### 10. Log Analysis

#### Format Log Mặc Định

```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" $request_time';
```

Ví dụ một dòng log:
```
192.168.1.100 - - [21/Jun/2026:10:30:45 +0700] "GET /api/products HTTP/1.1" 200 1234 "https://shoplite.example.com/" "Mozilla/5.0..." 0.023
```

Các trường:
- `$remote_addr`: IP của client.
- `$time_local`: Thời gian request.
- `$request`: Method + URL + protocol.
- `$status`: HTTP status code.
- `$body_bytes_sent`: Số bytes gửi về client.
- `$http_referer`: URL trước đó.
- `$http_user_agent`: Browser/client info.
- `$request_time`: Thời gian xử lý request (giây).

#### Phân Tích Log với Command Line

```bash
# Top 10 URLs nhiều request nhất
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Top 10 IPs gửi nhiều request nhất
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Tìm tất cả requests chậm hơn 1 giây (field cuối = $request_time)
awk '$NF > 1' /var/log/nginx/access.log

# Đếm số lỗi 5xx (server error)
awk '$9 >= 500 && $9 < 600' /var/log/nginx/access.log | wc -l

# Đếm số lỗi 4xx (client error)
awk '$9 >= 400 && $9 < 500' /var/log/nginx/access.log | wc -l

# Tỉ lệ lỗi theo giờ
awk '{print substr($4,2,14), $9}' /var/log/nginx/access.log | \
    awk '{hour=$1; if($2>=500) err[hour]++; total[hour]++} END {for(h in total) print h, err[h]/total[h]*100"%"}' | \
    sort

# Top User Agents (phát hiện bot)
awk -F'"' '{print $6}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20

# Request theo method (GET/POST/PUT/DELETE)
awk '{print $6}' /var/log/nginx/access.log | tr -d '"' | sort | uniq -c | sort -rn

# Bandwidth tiêu thụ theo URL (bytes)
awk '{bytes[$7]+=$10} END {for(url in bytes) print bytes[url], url}' \
    /var/log/nginx/access.log | sort -rn | head -10
```

#### Real-time Monitoring

```bash
# Xem status code phân phối theo thời gian thực
tail -f /var/log/nginx/access.log | awk '{print $9}' | sort | uniq -c

# Xem requests per second
tail -f /var/log/nginx/access.log | pv -l -i 1 > /dev/null

# Xem slow requests theo thời gian thực (> 0.5s)
tail -f /var/log/nginx/access.log | awk '$NF > 0.5 {print $NF, $7}'

# Xem error log theo thời gian thực
tail -f /var/log/nginx/error.log

# Monitor với watch (cập nhật mỗi 2 giây)
watch -n 2 'tail -100 /var/log/nginx/access.log | awk "{print \$9}" | sort | uniq -c'
```

#### Log Rotation với logrotate

```bash
# /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily                   # Rotate mỗi ngày
    missingok               # Không báo lỗi nếu file không tồn tại
    rotate 14               # Giữ 14 bản backup
    compress                # Nén file cũ với gzip
    delaycompress           # Nén từ bản thứ 2 trở đi (bản hôm nay giữ nguyên)
    notifempty              # Không rotate file rỗng
    create 0640 nginx adm   # Tạo file mới với permission 0640
    sharedscripts           # Chạy script một lần cho tất cả logs
    postrotate
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 `cat /var/run/nginx.pid`  # Signal nginx reopen logs
        fi
    endscript
}
```

---

## Kỹ năng đạt được
- Cấu hình Nginx làm reverse proxy cho ứng dụng thật.
- Thiết lập HTTPS với chứng chỉ.
- Áp dụng kỹ thuật bảo mật và tối ưu ở tầng proxy.

---

## Thực hành

**Môi trường:** Nginx chạy như một service trong Docker Compose.

**Lab:**
- Thêm service Nginx vào Compose làm cổng vào duy nhất.
- Định tuyến `/` → frontend, `/api` → backend.
- Cấu hình HTTPS (self-signed cho local, Let's Encrypt khi lên cloud).
- Thêm gzip, security header, rate limiting cơ bản.

**Công cụ:** Nginx, Certbot/Let's Encrypt.

---

## Bài tập

- **Bắt buộc:** Truy cập toàn bộ ShopLite qua một domain/port duy nhất qua Nginx.
- **Nâng cao:** Bật HTTPS và tự động chuyển hướng HTTP → HTTPS.
- **Mô phỏng doanh nghiệp:** Cấu hình upstream với 2 instance backend để minh họa load balancing.

---

## Deliverable
- [ ] File cấu hình `nginx.conf` cho ShopLite.
- [ ] Service Nginx trong Compose, hệ thống truy cập qua một entrypoint.
- [ ] HTTPS hoạt động.

---

## Tiêu chí hoàn thành
- [ ] Người dùng truy cập frontend và API qua cùng một domain.
- [ ] HTTPS hoạt động, HTTP tự chuyển sang HTTPS.
- [ ] Giải thích được vai trò reverse proxy trong kiến trúc.

---

## Ghi chú học tập

> _Ghi lại những điều học được, vướng mắc và insight trong quá trình học phase này._
