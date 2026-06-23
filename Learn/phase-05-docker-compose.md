# Phase 5 — Docker Compose (Điều phối Container Trên Máy Local)

## Mục tiêu
- Khởi động toàn bộ hệ thống ShopLite bằng **một lệnh duy nhất**.
- Quản lý phụ thuộc giữa các service (backend cần database sẵn sàng trước).

---

## Kiến thức sẽ học

- **Cú pháp `docker-compose.yml`** (services, networks, volumes, depends_on, healthcheck).
- **Biến môi trường & file `.env`** — tách cấu hình khỏi code (12-factor app).
- **Networking giữa các service trong Compose** — gọi nhau bằng tên service.
- **Healthcheck & thứ tự khởi động** — đảm bảo backend chỉ chạy khi DB sẵn sàng.
- **Quản lý dữ liệu bền vững bằng named volume** — DB không mất dữ liệu.
- **Profiles & override files (dev vs prod)** — nhiều cấu hình cho nhiều môi trường.

---

## Kiến thức chi tiết

### 1. Tại sao cần Docker Compose

#### Vấn đề khi chạy ShopLite thủ công

Trước khi có Docker Compose, để khởi động ShopLite bạn phải chạy từng lệnh một theo đúng thứ tự:

```bash
# Bước 1: Tạo network
docker network create shoplite-backend-net
docker network create shoplite-frontend-net

# Bước 2: Tạo volumes
docker volume create postgres-data
docker volume create redis-data

# Bước 3: Khởi động PostgreSQL trước
docker run -d \
  --name shoplite-postgres \
  --network shoplite-backend-net \
  -e POSTGRES_DB=shoplite \
  -e POSTGRES_USER=shoplite \
  -e POSTGRES_PASSWORD=change-me-in-production \
  -v postgres-data:/var/lib/postgresql/data \
  --restart unless-stopped \
  postgres:15-alpine

# Bước 4: Đợi PostgreSQL healthy rồi mới chạy Redis
docker run -d \
  --name shoplite-redis \
  --network shoplite-backend-net \
  -v redis-data:/data \
  --restart unless-stopped \
  redis:7-alpine redis-server --appendonly yes

# Bước 5: Đợi đủ điều kiện rồi mới chạy backend
docker run -d \
  --name shoplite-backend \
  --network shoplite-backend-net \
  --network shoplite-frontend-net \
  -p 8080:8080 \
  -e ConnectionStrings__DefaultConnection="Host=shoplite-postgres;Database=shoplite;Username=shoplite;Password=change-me-in-production" \
  -e ConnectionStrings__Redis="shoplite-redis:6379" \
  -e Jwt__Secret=your-secret \
  -e ASPNETCORE_ENVIRONMENT=Production \
  --restart unless-stopped \
  shoplite-backend:latest

# Bước 6: Cuối cùng mới chạy frontend
docker run -d \
  --name shoplite-frontend \
  --network shoplite-frontend-net \
  -p 80:80 \
  -e BACKEND_URL=http://shoplite-backend:3000 \
  --restart unless-stopped \
  shoplite-frontend:latest
```

Đây là những vấn đề thực tế:
- **Khó nhớ flags**: Mỗi lần chạy phải nhớ hàng chục tham số khác nhau cho từng container.
- **Phải start đúng thứ tự**: PostgreSQL phải healthy trước khi backend start. Nếu không, backend crash ngay.
- **Không reproducible**: Hai người có thể chạy lệnh hơi khác nhau dẫn đến môi trường khác nhau.
- **Không có single source of truth**: Cấu hình nằm rải rác trong các lệnh shell, không được version control.
- **Khó rollback và debug**: Khi có lỗi, khó biết container nào đang dùng cấu hình gì.
- **Onboarding mới**: Thành viên mới phải đọc hướng dẫn dài và thực hiện từng bước một cách thủ công.

#### Solution: Docker Compose

Docker Compose giải quyết tất cả vấn đề trên bằng cách:

- **Một file YAML mô tả toàn bộ stack**: Tất cả services, networks, volumes, environment variables đều nằm trong `docker-compose.yml`.
- **Một lệnh để start**: `docker compose up -d` khởi động toàn bộ hệ thống.
- **Một lệnh để stop**: `docker compose down` dừng và dọn dẹp.
- **Tự động xử lý thứ tự**: `depends_on` với `condition: service_healthy` đảm bảo đúng thứ tự.
- **Version controlled**: File YAML được commit vào git, mọi người dùng cùng một cấu hình.

```bash
# Sau khi có docker-compose.yml, chỉ cần:
git clone https://github.com/company/shoplite.git
cd shoplite
cp .env.example .env
# Sửa .env với giá trị thật
docker compose up -d
# Xong. Toàn bộ hệ thống đang chạy.
```

#### docker compose (v2) vs docker-compose (v1)

Đây là điểm quan trọng cần phân biệt rõ:

**Docker Compose V1** (`docker-compose`):
- Viết bằng Python, cài riêng (`pip install docker-compose`).
- Lệnh: `docker-compose up` (có dấu gạch ngang).
- Đã deprecated từ tháng 7/2023 và không còn được maintained.
- Nếu gõ `docker-compose --version` và thấy version 1.x thì đang dùng bản cũ.

**Docker Compose V2** (`docker compose`):
- Viết lại bằng Go, tích hợp sẵn vào Docker CLI như một plugin.
- Lệnh: `docker compose up` (không có dấu gạch ngang).
- Built-in trong Docker Desktop và Docker Engine >= 20.10.
- Hiệu suất tốt hơn, nhiều tính năng hơn.

```bash
# Kiểm tra version đang dùng
docker compose version
# Output: Docker Compose version v2.x.x

# Nếu vẫn cần backward compatibility, có thể tạo alias:
# alias docker-compose='docker compose'
```

**Kết luận**: Luôn dùng `docker compose` (v2). Không dùng `docker-compose` (v1) cho dự án mới.

---

### 2. docker-compose.yml hoàn chỉnh cho ShopLite

Dưới đây là file `docker-compose.yml` đầy đủ cho ShopLite với giải thích chi tiết từng phần:

```yaml
# docker-compose.yml
# Docker Compose V2 schema
# Docs: https://docs.docker.com/compose/compose-file/

services:

  # ─────────────────────────────────────────
  # FRONTEND SERVICE
  # ─────────────────────────────────────────
  frontend:
    build:
      context: ./frontend          # Đường dẫn tới Dockerfile của frontend
      dockerfile: Dockerfile       # Tên Dockerfile (mặc định là "Dockerfile")
      target: production           # Multi-stage build: dùng stage "production"
      args:                        # Build arguments (available tại build time)
        - NODE_VERSION=20
    image: shoplite-frontend:latest  # Tag cho image sau khi build
    container_name: shoplite-frontend
    ports:
      - "80:80"                    # host:container — expose port 80 ra ngoài
    depends_on:
      backend:
        condition: service_healthy # Chỉ start sau khi backend đã healthy
    environment:
      - BACKEND_URL=http://backend:3000  # Dùng service name "backend"
      - NODE_ENV=${NODE_ENV:-production}
    networks:
      - frontend-net               # Chỉ cần giao tiếp với backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:80/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # ─────────────────────────────────────────
  # BACKEND SERVICE (ASP.NET Core / .NET 8)
  # ─────────────────────────────────────────
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
      target: runner
    image: shoplite-backend:latest
    container_name: shoplite-backend
    ports:
      - "8080:8080"                # Expose API port (ASP.NET Core default)
    depends_on:
      postgres:
        condition: service_healthy # QUAN TRỌNG: phải đợi DB healthy
      redis:
        condition: service_healthy # QUAN TRỌNG: phải đợi Redis healthy
    environment:
      # .NET dùng double underscore __ để phân cấp config
      # ConnectionStrings__DefaultConnection = ConnectionStrings:DefaultConnection trong appsettings.json
      - ConnectionStrings__DefaultConnection=${DATABASE_CONNECTION_STRING}
      - ConnectionStrings__Redis=${REDIS_CONNECTION_STRING:-redis:6379}
      - Jwt__Secret=${JWT_SECRET}
      - ASPNETCORE_ENVIRONMENT=${ASPNETCORE_ENVIRONMENT:-Production}
    # Thay thế: dùng env_file để load toàn bộ file .env
    # env_file:
    #   - .env
    networks:
      - frontend-net               # Để frontend có thể gọi backend
      - backend-net                # Để backend gọi postgres và redis
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/health"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 30s            # Cho ASP.NET Core 30s để khởi động trước khi check
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "0.5"
        reservations:
          memory: 256M
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"

  # ─────────────────────────────────────────
  # POSTGRESQL SERVICE
  # ─────────────────────────────────────────
  postgres:
    image: postgres:15-alpine      # Alpine variant: nhỏ hơn, bảo mật hơn
    container_name: shoplite-postgres
    environment:
      - POSTGRES_DB=${POSTGRES_DB:-shoplite}
      - POSTGRES_USER=${POSTGRES_USER:-shoplite}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      # Nếu muốn init script chạy khi lần đầu tạo DB:
      # POSTGRES_INITDB_ARGS: "--encoding=UTF-8 --lc-collate=C"
    volumes:
      - postgres-data:/var/lib/postgresql/data   # Named volume để persist data
      # Optional: mount init scripts
      # - ./db/init:/docker-entrypoint-initdb.d
    networks:
      - backend-net                # Chỉ nằm trong backend network, không expose ra frontend
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-shoplite} -d ${POSTGRES_DB:-shoplite}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    # KHÔNG expose port ra host trong production
    # Nếu cần debug local:
    # ports:
    #   - "5432:5432"
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
    logging:
      driver: "json-file"
      options:
        max-size: "5m"
        max-file: "3"

  # ─────────────────────────────────────────
  # REDIS SERVICE
  # ─────────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: shoplite-redis
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    # Giải thích flags:
    # --appendonly yes: bật AOF persistence (data bền vững sau restart)
    # --maxmemory 256mb: giới hạn memory Redis dùng
    # --maxmemory-policy allkeys-lru: khi đầy, xóa key ít dùng nhất
    volumes:
      - redis-data:/data           # Persist Redis data
    networks:
      - backend-net                # Chỉ cần backend network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 10s
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: "0.25"
    logging:
      driver: "json-file"
      options:
        max-size: "5m"
        max-file: "3"

# ─────────────────────────────────────────
# NETWORKS
# ─────────────────────────────────────────
networks:
  frontend-net:
    driver: bridge
    # frontend và backend nằm trong network này
    # frontend gọi backend qua http://backend:3000

  backend-net:
    driver: bridge
    internal: true
    # internal: true = các container trong network này không có internet access
    # backend, postgres, redis nằm trong network này
    # postgres và redis KHÔNG accessible từ frontend

# ─────────────────────────────────────────
# VOLUMES (Named Volumes)
# ─────────────────────────────────────────
volumes:
  postgres-data:
    driver: local
    # Data của PostgreSQL được lưu trong Docker-managed volume
    # Không bị xóa khi "docker compose down" (chỉ xóa khi "docker compose down -v")

  redis-data:
    driver: local
    # Data của Redis (AOF files) được lưu persistent
```

#### Giải thích thiết kế chi tiết

**Tại sao dùng `postgres:15-alpine` thay vì `postgres:15`?**

Alpine Linux là bản phân phối Linux cực nhỏ (~5MB). Image Alpine variant nhỏ hơn nhiều so với Debian-based:
- `postgres:15` (Debian): ~412MB
- `postgres:15-alpine`: ~238MB

Nhỏ hơn nghĩa là download nhanh hơn, attack surface nhỏ hơn (ít packages = ít CVE tiềm năng), và deploy nhanh hơn.

**Tại sao backend thuộc cả hai network?**

Backend cần giao tiếp theo cả hai chiều:
- Nhận request từ `frontend` (frontend-net)
- Gọi `postgres` và `redis` để xử lý data (backend-net)

Frontend chỉ cần giao tiếp với backend, không cần biết postgres hay redis tồn tại. Đây là nguyên tắc least privilege trong networking.

**Tại sao `internal: true` cho backend-net?**

Để postgres và redis không thể tự kết nối ra internet. Điều này tăng bảo mật:
- Nếu backend bị compromise, attacker không thể dùng postgres/redis container để exfiltrate data ra ngoài qua internet.
- Postgres chỉ có thể nói chuyện với các container khác trong cùng network.

**Tại sao không expose port của postgres và redis ra host?**

Trong production, không bao giờ expose database ports ra ngoài trực tiếp. Nếu cần debug:
- Dùng `docker compose exec postgres psql -U shoplite` để vào trong container
- Hoặc mở port tạm thời trong `docker-compose.override.yml` (chỉ trong dev)

---

### 3. Environment Variables và File .env

#### 12-Factor App - Nguyên tắc cơ bản

12-Factor App là phương pháp xây dựng software-as-a-service được Heroku đề ra. Factor III nói về config:

> "An app's config is everything that is likely to vary between deploys (staging, production, developer environments). Apps sometimes store config as constants in the code. This is a violation of twelve-factor, which requires strict separation of config from code."

**Config phải đến từ environment variables, không được hardcode trong code hoặc config files được commit vào git.**

Lý do:
- Cùng một codebase có thể chạy ở nhiều môi trường (dev, staging, prod) với config khác nhau.
- Secrets (passwords, API keys) không được lưu trong git history.
- Dễ thay đổi config mà không cần rebuild image.

#### File .env (KHÔNG commit vào git)

```bash
# .env
# File này chứa giá trị thật - KHÔNG BAO GIỜ commit vào git
# Mỗi developer có file .env riêng của mình

# Database (Connection string theo định dạng Npgsql)
DATABASE_CONNECTION_STRING=Host=postgres;Database=shoplite;Username=shoplite;Password=change-me-in-production
POSTGRES_DB=shoplite
POSTGRES_USER=shoplite
POSTGRES_PASSWORD=change-me-in-production

# Redis
REDIS_CONNECTION_STRING=redis:6379

# Authentication
JWT_SECRET=your-super-secret-jwt-key-here-minimum-32-chars

# Application
ASPNETCORE_ENVIRONMENT=Development

# Logging có thể cấu hình qua env var (double underscore cho nested config)
# Logging__LogLevel__Default=Debug
```

#### File .env.example (commit vào git)

```bash
# .env.example
# Template cho file .env - commit file này vào git
# Hướng dẫn: copy file này thành .env và điền giá trị thật vào
# cp .env.example .env

# Database (Connection string theo định dạng Npgsql)
DATABASE_CONNECTION_STRING=Host=postgres;Database=shoplite;Username=shoplite;Password=change-me-in-production
POSTGRES_DB=shoplite
POSTGRES_USER=shoplite
POSTGRES_PASSWORD=change-me-in-production

# Redis
REDIS_CONNECTION_STRING=redis:6379

# Authentication
# Tạo secret ngẫu nhiên: openssl rand -base64 32
JWT_SECRET=generate-random-secret-here

# Application
ASPNETCORE_ENVIRONMENT=Development

# Logging (override appsettings.json)
# Logging__LogLevel__Default=Information
```

#### Cách sử dụng biến môi trường trong Compose

**Cách 1: Reference từng biến**

```yaml
environment:
  - ConnectionStrings__DefaultConnection=${DATABASE_CONNECTION_STRING}
  - ConnectionStrings__Redis=${REDIS_CONNECTION_STRING}
  - Jwt__Secret=${JWT_SECRET}
```

Docker Compose tự động đọc file `.env` trong cùng thư mục với `docker-compose.yml`.

**Cách 2: Biến có default value**

```yaml
environment:
  - ASPNETCORE_ENVIRONMENT=${ASPNETCORE_ENVIRONMENT:-Production}
  - ConnectionStrings__Redis=${REDIS_CONNECTION_STRING:-redis:6379}
  - Logging__LogLevel__Default=${LOG_LEVEL:-Warning}
```

Cú pháp `${VAR:-default}` rất hữu ích để có giá trị fallback.

**Cách 3: Load toàn bộ file .env**

```yaml
services:
  backend:
    env_file:
      - .env
      - .env.local    # Override thêm (nếu có)
```

Khi dùng `env_file`, tất cả biến trong file sẽ được inject vào container.

**Cách 4: Hard-code giá trị không nhạy cảm**

```yaml
environment:
  - ASPNETCORE_ENVIRONMENT=Production   # Giá trị này không nhạy cảm, có thể hard-code
  - TZ=Asia/Ho_Chi_Minh                 # Timezone
```

#### Bảo mật .env

```bash
# Đảm bảo .env nằm trong .gitignore
echo ".env" >> .gitignore
echo ".env.local" >> .gitignore
echo ".env.*.local" >> .gitignore

# Kiểm tra .gitignore đã có chưa
cat .gitignore | grep .env

# Nếu lỡ commit .env vào git, phải xóa khỏi git history:
git rm --cached .env
git commit -m "Remove .env from tracking"

# Nhưng lưu ý: nếu đã push lên remote, secret đã bị expose
# Cần rotate (thay đổi) tất cả passwords và secrets ngay lập tức
# Sau đó dùng git-filter-repo để xóa khỏi history (không dùng filter-branch)
pip install git-filter-repo
git filter-repo --path .env --invert-paths
```

**Best practices cho secrets**:
1. Dùng `.env` chỉ cho local development.
2. Trong production: dùng secrets manager (AWS Secrets Manager, HashiCorp Vault, Kubernetes Secrets).
3. Rotate secrets định kỳ (3-6 tháng một lần).
4. Audit ai có access vào secrets.
5. Không chia sẻ `.env` qua Slack, email, hay chat.

---

### 4. Healthcheck và depends_on

#### Vấn đề với depends_on đơn giản

```yaml
# KHÔNG đủ - chỉ đảm bảo container đã START, không đảm bảo đã READY
depends_on:
  - postgres
  - redis
```

Khi backend start, postgres container đã được tạo và đang chạy, nhưng PostgreSQL server bên trong có thể chưa sẵn sàng chấp nhận connection. Kết quả: backend kết nối tới postgres thất bại và crash.

#### depends_on với condition (cách đúng)

```yaml
depends_on:
  postgres:
    condition: service_healthy    # Chờ cho đến khi healthcheck pass
  redis:
    condition: service_healthy    # Chờ cho đến khi healthcheck pass
```

Với `condition: service_healthy`, Docker Compose sẽ:
1. Start postgres container
2. Chạy healthcheck command định kỳ
3. Đợi cho đến khi healthcheck trả về exit code 0 (healthy)
4. SAU ĐÓ mới start backend container

Các giá trị condition:
- `service_started`: Container đã start (không chờ healthy) - default
- `service_healthy`: Container phải healthy (cần có healthcheck được định nghĩa)
- `service_completed_successfully`: Container đã chạy xong và exit với code 0 (dành cho init containers)

#### Healthcheck cho PostgreSQL

```yaml
postgres:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-shoplite} -d ${POSTGRES_DB:-shoplite}"]
    interval: 10s      # Chạy healthcheck mỗi 10 giây
    timeout: 5s        # Timeout sau 5 giây nếu không có response
    retries: 5         # Fail 5 lần liên tiếp mới mark là unhealthy
    start_period: 30s  # Không count failures trong 30 giây đầu sau khi start
```

`pg_isready` là command có sẵn trong PostgreSQL image, trả về:
- Exit code 0: Server đang chạy và chấp nhận connections
- Exit code 1: Server reject connections
- Exit code 2: Không có response

**Giải thích `start_period`**: PostgreSQL mất thời gian để khởi tạo data directory, chạy recovery nếu cần, rồi mới sẵn sàng. `start_period: 30s` cho PostgreSQL 30 giây "grace period" - trong 30 giây đó, healthcheck failures không được tính. Sau 30 giây, nếu vẫn fail thì mới mark là unhealthy.

#### Healthcheck cho Redis

```yaml
redis:
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 10s
    timeout: 3s
    retries: 5
    start_period: 10s
```

`redis-cli ping` gửi PING command tới Redis server. Nếu Redis healthy, server trả về `PONG` và exit code 0.

#### Healthcheck cho backend Node.js

```yaml
backend:
  healthcheck:
    test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
    interval: 15s
    timeout: 5s
    retries: 5
    start_period: 30s
```

Hoặc dùng curl nếu có trong image:

```yaml
test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
```

Hoặc dùng Node.js script:

```yaml
test: ["CMD", "node", "-e", "require('http').get('http://localhost:3000/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1))"]
```

Backend cần có endpoint `/health` trả về HTTP 200 khi healthy:

```csharp
// Cách 1: Minimal API trong Program.cs
app.MapGet("/health", async (AppDbContext db, IConnectionMultiplexer redis) =>
{
    try
    {
        await db.Database.ExecuteSqlRawAsync("SELECT 1");
        await redis.GetDatabase().PingAsync();
        return Results.Ok(new { status = "healthy", timestamp = DateTime.UtcNow });
    }
    catch (Exception ex)
    {
        return Results.Json(new { status = "unhealthy", error = ex.Message },
            statusCode: StatusCodes.Status503ServiceUnavailable);
    }
});

// Cách 2: Built-in Health Checks middleware (khuyên dùng cho production)
// Trong Program.cs:
builder.Services.AddHealthChecks()
    .AddNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")!)
    .AddRedis(builder.Configuration.GetConnectionString("Redis")!);

app.MapHealthChecks("/health");
```

#### Trạng thái healthcheck

Một container có thể ở các trạng thái sau:
- `starting`: Container vừa start, đang trong `start_period`
- `healthy`: Healthcheck pass gần nhất
- `unhealthy`: Healthcheck fail nhiều hơn `retries` lần
- `none`: Không có healthcheck được định nghĩa

```bash
# Xem trạng thái health
docker compose ps
# NAME                STATUS
# shoplite-postgres   Up 2 minutes (healthy)
# shoplite-redis      Up 2 minutes (healthy)
# shoplite-backend    Up 1 minute (healthy)

# Xem chi tiết healthcheck history
docker inspect shoplite-postgres --format='{{json .State.Health}}' | jq
```

---

### 5. Networking trong Compose

#### Service Discovery - Cách containers giao tiếp

Khi các containers nằm trong cùng một Docker network, chúng có thể gọi nhau bằng **service name** (tên được định nghĩa trong `docker-compose.yml`), không cần biết IP address.

```yaml
services:
  backend:
    environment:
      # Dùng "postgres" (service name), không phải "localhost" hay "172.17.0.x"
      - ConnectionStrings__DefaultConnection=Host=postgres;Database=shoplite;Username=shoplite;Password=secret
      #                                              ^^^^^^^^
      #                                              Service name trong compose
      - ConnectionStrings__Redis=redis:6379
      #                          ^^^^^
      #                          Service name trong compose
```

Docker Compose tự động tạo DNS entries cho mỗi service trong network. Khi backend gọi `postgres:5432`, Docker DNS resolver sẽ resolve `postgres` thành IP của container đang chạy service đó.

#### Tại sao không dùng "localhost"?

Mỗi container là một process riêng biệt với network namespace riêng. `localhost` trong container backend trỏ tới chính container backend đó, không phải container postgres. Muốn kết nối tới postgres, phải dùng hostname/service name.

#### Custom Networks

```yaml
networks:
  # Network cho giao tiếp frontend - backend
  frontend-net:
    driver: bridge
    # bridge: mạng nội bộ trên host, containers có thể giao tiếp với nhau và với internet
    ipam:
      config:
        - subnet: 172.20.0.0/24   # Optional: custom subnet

  # Network cho giao tiếp backend - database
  backend-net:
    driver: bridge
    internal: true
    # internal: true = containers trong network này KHÔNG có internet access
    # Bảo mật tốt hơn cho database layer
```

#### Gán service vào multiple networks

```yaml
services:
  backend:
    networks:
      - frontend-net    # Để frontend gọi được backend
      - backend-net     # Để backend gọi được postgres và redis

  frontend:
    networks:
      - frontend-net    # Chỉ giao tiếp với backend, không biết postgres/redis

  postgres:
    networks:
      - backend-net     # Chỉ accessible từ backend (và services khác trong backend-net)

  redis:
    networks:
      - backend-net     # Chỉ accessible từ backend
```

#### Network Isolation - Tại sao quan trọng

Với cấu hình trên:
- Frontend CÓ THỂ gọi backend (cùng frontend-net)
- Frontend KHÔNG THỂ gọi trực tiếp postgres hay redis (khác network)
- Backend CÓ THỂ gọi postgres và redis (cùng backend-net)
- Postgres và redis KHÔNG THỂ gọi nhau nếu không cùng network

Đây là nguyên tắc **defense in depth** - nhiều lớp bảo vệ. Ngay cả khi frontend bị tấn công, attacker vẫn không thể trực tiếp query database.

#### Port Mapping

```yaml
services:
  frontend:
    ports:
      - "80:80"       # host_port:container_port
      # Expose port 80 của container ra port 80 trên host machine

  backend:
    ports:
      - "3000:3000"   # Expose API port ra host (cần cho external clients)
      # Trong production thường chỉ expose qua nginx reverse proxy

  postgres:
    # Không expose port ra host trong production
    # Chỉ accessible từ bên trong Docker network
    # Nếu cần debug tạm thời:
    # ports:
    #   - "127.0.0.1:5432:5432"   # Chỉ bind vào localhost, không bind 0.0.0.0
```

**Best practice**: Chỉ expose port ra host khi thực sự cần. Database ports không bao giờ nên expose ra `0.0.0.0` trong production.

---

### 6. Volumes trong Compose

#### Named Volumes vs Bind Mounts

**Named Volumes** (cho production data persistence):

```yaml
volumes:
  postgres-data:    # Docker quản lý volume này
    driver: local
  redis-data:
    driver: local

services:
  postgres:
    volumes:
      - postgres-data:/var/lib/postgresql/data
      # Named volume được mount vào path trong container
```

Đặc điểm:
- Docker quản lý vị trí lưu trữ trên host (thường trong `/var/lib/docker/volumes/`)
- Persist qua container restarts và recreations
- Có thể backup, migrate dễ dàng
- Không bị ảnh hưởng bởi host file system structure

**Bind Mounts** (cho development hot reload):

```yaml
services:
  backend:
    volumes:
      - ./backend:/src          # Mount source code vào /src để dotnet watch detect thay đổi
    environment:
      - DOTNET_USE_POLLING_FILE_WATCHER=true  # Cần thiết vì inotify không hoạt động qua Docker volume
```

Với .NET không cần anonymous volume trick như Node.js. SDK stage tự quản lý build artifacts trong container, không bị ảnh hưởng bởi `bin/` và `obj/` trên host (đã ignore trong .dockerignore).

#### Vòng đời của Volumes

```bash
# docker compose down: Dừng và xóa containers, networks
# NHƯNG volumes vẫn còn nguyên
docker compose down

# Kiểm tra volumes vẫn còn
docker volume ls | grep shoplite
# shoplite_postgres-data    (data vẫn intact)
# shoplite_redis-data       (data vẫn intact)

# docker compose up lại: containers mới nhưng data cũ vẫn còn
docker compose up -d

# ─────────────────────────────────────────
# Để XÓA volumes (cẩn thận - mất toàn bộ data)
docker compose down -v
# -v flag: xóa cả volumes

# Dùng khi cần: fresh start, testing, cleanup
# KHÔNG BAO GIỜ dùng -v trong production với data thật
```

#### Volume cho dev với hot reload

```yaml
# docker-compose.override.yml (chỉ dùng cho dev)
services:
  backend:
    volumes:
      - ./backend:/src     # Mount source code để dotnet watch detect thay đổi
    command: dotnet watch run --project ShopLite.Api --no-launch-profile
    environment:
      - DOTNET_USE_POLLING_FILE_WATCHER=true
```

---

### 7. Compose Commands Chi Tiết

#### Khởi động và dừng

```bash
# ─── KHỞI ĐỘNG ───────────────────────────────────────

# Khởi động tất cả services (foreground - xem logs trực tiếp)
docker compose up

# Khởi động tất cả services (detached - chạy nền)
docker compose up -d

# Rebuild images trước khi khởi động (khi có thay đổi code)
docker compose up --build

# Chỉ rebuild và restart một service cụ thể
docker compose up -d --build backend

# Khởi động một service cụ thể và dependencies của nó
docker compose up -d backend   # Tự động start postgres và redis trước

# ─── DỪNG ────────────────────────────────────────────

# Dừng nhưng không xóa containers (có thể resume)
docker compose stop

# Dừng và xóa containers, networks (giữ volumes)
docker compose down

# Dừng, xóa containers, networks VÀ volumes (mất data)
docker compose down -v

# Dừng, xóa containers, networks, volumes VÀ images
docker compose down -v --rmi all

# Chỉ dừng và xóa một service
docker compose rm -sf backend
```

#### Xem trạng thái và logs

```bash
# ─── STATUS ──────────────────────────────────────────

# Xem danh sách services và trạng thái
docker compose ps

# Output:
# NAME                  IMAGE                    COMMAND    STATUS              PORTS
# shoplite-backend      shoplite-backend:latest  ...        Up 5 minutes (healthy)   0.0.0.0:3000->3000/tcp
# shoplite-frontend     shoplite-frontend:latest ...        Up 5 minutes (healthy)   0.0.0.0:80->80/tcp
# shoplite-postgres     postgres:15-alpine        ...        Up 6 minutes (healthy)
# shoplite-redis        redis:7-alpine            ...        Up 6 minutes (healthy)

# Xem processes đang chạy trong containers
docker compose top

# Validate và xem config đã được resolve (biến .env đã được thay thế)
docker compose config

# ─── LOGS ────────────────────────────────────────────

# Xem logs của tất cả services
docker compose logs

# Follow (stream) logs của tất cả services
docker compose logs -f

# Chỉ xem logs của backend và follow
docker compose logs -f backend

# Xem 50 dòng cuối của postgres logs
docker compose logs --tail=50 postgres

# Xem logs từ 1 giờ trước đến giờ
docker compose logs --since 1h

# Xem logs kèm timestamp
docker compose logs -t backend

# Kết hợp: follow logs của backend và frontend
docker compose logs -f backend frontend
```

#### Truy cập container

```bash
# ─── EXEC ────────────────────────────────────────────

# Mở interactive shell trong container backend (Alpine image dùng sh, không có bash)
docker compose exec backend sh

# Xem thông tin .NET runtime trong container
docker compose exec backend dotnet --info

# Chạy command cụ thể trong container
docker compose exec backend dotnet --list-runtimes

# Vào psql trong postgres container
docker compose exec postgres psql -U shoplite -d shoplite

# Chạy Redis CLI
docker compose exec redis redis-cli

# Kiểm tra Redis
docker compose exec redis redis-cli INFO server
docker compose exec redis redis-cli KEYS "*"

# ─── RUN ─────────────────────────────────────────────
# run: tạo container MỚI (không phải container đang chạy) để chạy một lần

# Chạy EF Core migrations (tạo container mới, xóa sau khi xong)
docker compose run --rm backend dotnet ef database update --project ShopLite.Api

# Debug: mở shell trong container với entrypoint tùy chỉnh
docker compose run --rm --entrypoint sh backend

# Chạy tests
docker compose run --rm backend dotnet test

# Tạo container mới với environment variable tùy chỉnh
docker compose run --rm -e ASPNETCORE_ENVIRONMENT=Development backend dotnet ShopLite.Api.dll
```

#### Quản lý images và builds

```bash
# ─── BUILD ───────────────────────────────────────────

# Build tất cả images
docker compose build

# Build không dùng cache (full rebuild)
docker compose build --no-cache

# Chỉ build service backend
docker compose build backend

# Build với build argument
docker compose build --build-arg NODE_VERSION=20 backend

# Pull images mới nhất từ registry
docker compose pull

# Push images lên registry
docker compose push

# ─── SCALE ───────────────────────────────────────────

# Scale backend lên 3 instances (không khuyến khích nếu có port binding cố định)
docker compose up -d --scale backend=3

# Scale về 1 instance
docker compose up -d --scale backend=1
```

#### Các lệnh khác hữu ích

```bash
# Tạm dừng một service (freeze, không xóa)
docker compose pause backend

# Tiếp tục service đã pause
docker compose unpause backend

# Restart một service (stop rồi start lại)
docker compose restart backend

# Kiểm tra port binding
docker compose port backend 3000
# Output: 0.0.0.0:3000
```

---

### 8. Override Files Pattern (Dev vs Prod)

#### Cấu trúc files

Docker Compose có cơ chế merge files cho phép tổ chức config theo môi trường:

```
shoplite/
├── docker-compose.yml          # Base config (shared giữa tất cả môi trường)
├── docker-compose.override.yml # Dev overrides (auto-loaded, không commit hoặc commit tùy team)
├── docker-compose.prod.yml     # Production config
├── docker-compose.test.yml     # Test environment config
├── .env                        # Biến môi trường local (KHÔNG commit)
└── .env.example                # Template (commit vào git)
```

#### docker-compose.yml (Base)

```yaml
# docker-compose.yml
# Config chung, không chứa môi trường-specific settings
services:
  backend:
    build:
      context: ./backend
    environment:
      - ASPNETCORE_ENVIRONMENT=${ASPNETCORE_ENVIRONMENT:-Production}
      - ConnectionStrings__DefaultConnection=${DATABASE_CONNECTION_STRING}
      - Jwt__Secret=${JWT_SECRET}
    networks:
      - frontend-net
      - backend-net
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=${POSTGRES_DB:-shoplite}
      - POSTGRES_USER=${POSTGRES_USER:-shoplite}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - backend-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-shoplite}"]
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  frontend-net:
  backend-net:

volumes:
  postgres-data:
```

#### docker-compose.override.yml (Development - auto-loaded)

```yaml
# docker-compose.override.yml
# Được auto-load khi chạy "docker compose up" (không cần chỉ định -f)
# Dành cho development: hot reload, debug ports, bind mounts
services:
  backend:
    build:
      target: builder        # Dùng SDK stage để có dotnet watch
    volumes:
      - ./backend:/src       # Bind mount source code để hot reload
    command: dotnet watch run --project ShopLite.Api --no-launch-profile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - DOTNET_USE_POLLING_FILE_WATCHER=true  # Cần thiết cho file change detection qua Docker volume
      - ASPNETCORE_URLS=http://+:8080
      - Logging__LogLevel__Default=Debug
    ports:
      - "8080:8080"

  frontend:
    build:
      target: development
    volumes:
      - ./frontend:/app
      - /app/node_modules
    command: npm run dev     # Vite/webpack dev server với HMR
    ports:
      - "5173:5173"          # Vite dev server port

  postgres:
    ports:
      - "127.0.0.1:5432:5432"  # Expose để dùng với DBeaver, TablePlus, etc.
    environment:
      - POSTGRES_PASSWORD=dev-password-only   # Đơn giản hơn cho dev

  redis:
    ports:
      - "127.0.0.1:6379:6379"  # Expose để dùng với RedisInsight
```

#### docker-compose.prod.yml (Production)

```yaml
# docker-compose.prod.yml
# Dùng: docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
services:
  backend:
    restart: unless-stopped
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - Logging__LogLevel__Default=Warning
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "0.5"
        reservations:
          memory: 256M
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"

  frontend:
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 128M
          cpus: "0.25"

  postgres:
    restart: unless-stopped
    # Không expose port ra ngoài trong production
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: "1.0"
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "5"

  redis:
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "0.5"
```

#### Cách sử dụng

```bash
# Development (auto-loads override.yml)
docker compose up -d

# Production (override file chỉ định rõ)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Xem config đã được merge như thế nào
docker compose config                                              # dev
docker compose -f docker-compose.yml -f docker-compose.prod.yml config  # prod

# Testing
docker compose -f docker-compose.yml -f docker-compose.test.yml up --abort-on-container-exit
```

#### Merge behavior

Khi merge files, Docker Compose sử dụng rules sau:
- **Scalars** (strings, numbers): File sau override file trước
- **Lists** (ports, volumes, environment): Được nối lại (append), không override
- **Maps** (services, networks): Deep merge

```yaml
# Base file
services:
  backend:
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production

# Override file
services:
  backend:
    ports:
      - "5005:5005"     # Được THÊM VÀO, không replace (ví dụ: debugger port)
    environment:
      - ASPNETCORE_ENVIRONMENT=Development   # Override giá trị

# Result sau merge:
services:
  backend:
    ports:
      - "8080:8080"
      - "5005:5005"     # Cả hai ports
    environment:
      - ASPNETCORE_ENVIRONMENT=Development   # Giá trị mới nhất
```

---

### 9. Production Considerations

#### Restart Policies

```yaml
services:
  backend:
    restart: unless-stopped
    # Các giá trị khác:
    # restart: "no"           - Không tự restart (default)
    # restart: always         - Luôn restart, kể cả khi manual stop
    # restart: on-failure     - Chỉ restart khi exit với non-zero code
    # restart: on-failure:3   - Chỉ restart tối đa 3 lần
    # restart: unless-stopped - Restart trừ khi bị stop thủ công
```

**Khi nào dùng gì?**
- `unless-stopped`: Recommended cho hầu hết services. Container restart sau crash, sau reboot, nhưng không restart khi bạn manual `docker compose stop`.
- `always`: Dùng cho critical services cần luôn chạy. Restart ngay cả khi manual stop.
- `on-failure`: Dùng cho batch jobs, init containers. Chỉ restart khi có lỗi.
- `"no"`: Dùng cho containers chỉ chạy một lần (migrations, tests).

#### Logging Configuration

```yaml
services:
  backend:
    logging:
      driver: "json-file"          # Default driver, lưu vào JSON files trên host
      options:
        max-size: "10m"            # Rotate khi file đạt 10MB
        max-file: "5"              # Giữ tối đa 5 files (tổng cộng 50MB)

  # Các logging drivers khác:
  # driver: "syslog"    - Gửi tới syslog daemon
  # driver: "gelf"      - Graylog Extended Log Format (GELF)
  # driver: "fluentd"   - Fluentd
  # driver: "awslogs"   - Amazon CloudWatch Logs
  # driver: "none"      - Không log
```

**Tại sao quan trọng?** Nếu không cấu hình log rotation, log files có thể fill đầy disk của server theo thời gian. Đặc biệt nguy hiểm với verbose applications.

#### Resource Limits

```yaml
services:
  backend:
    deploy:
      resources:
        limits:
          memory: 512M    # Container không được dùng quá 512MB RAM
          cpus: "0.5"     # Container không dùng quá 50% của một CPU core
        reservations:
          memory: 256M    # Docker đảm bảo container luôn có 256MB RAM
          cpus: "0.25"    # Docker đảm bảo container có 25% CPU
```

**Tại sao cần limits?**
- Ngăn một container "ăn" hết resource của host, ảnh hưởng tới các services khác.
- Trong môi trường nhiều services trên cùng một máy, đây là biện pháp bảo vệ quan trọng.

**Lưu ý quan trọng về `deploy` key**:

```yaml
# CẢNH BÁO: Trong Docker Compose standalone mode, key "deploy" chỉ có tác dụng
# với "memory" limits. CPU limits cần version Compose khác hoặc dùng:
services:
  backend:
    mem_limit: 512m         # Tương đương với deploy.resources.limits.memory
    cpus: 0.5               # Tương đương với deploy.resources.limits.cpus
    memswap_limit: 512m     # Không cho phép swap (tốt hơn cho production)
```

#### Docker Compose vs Kubernetes

Điều quan trọng cần hiểu về giới hạn của Docker Compose trong production:

| Tính năng | Docker Compose | Kubernetes |
|-----------|----------------|------------|
| Single host | Tốt | Được |
| Multi-host clustering | Không hỗ trợ | Native |
| Auto scaling | Thủ công | Auto (HPA) |
| Rolling updates | Thủ công | Native |
| Self-healing | Chỉ restart | Native |
| Load balancing | Cơ bản (Round robin) | Advanced |
| Secret management | Hạn chế | K8s Secrets |

**Kết luận**: Docker Compose phù hợp cho:
- Local development
- Small team projects
- Single-server deployments với traffic thấp
- Staging/QA environments

Không phù hợp cho:
- High-availability production systems
- Multi-host clusters
- Applications cần auto-scaling

---

### 10. Database Migrations

#### Vấn đề với Database Migrations

Migrations là scripts thay đổi database schema (tạo bảng, thêm cột, etc.). Phải chạy migrations trước khi application start, vì application cần schema đúng để hoạt động.

#### Pattern 1: Command trong Compose

```yaml
services:
  backend:
    # Chạy EF Core migrations rồi mới start application
    # Vấn đề: nếu migrate fail, container cũng fail và restart
    command: sh -c "dotnet ef database update --project ShopLite.Api && dotnet ShopLite.Api.dll"
```

Đơn giản nhưng có nhược điểm: mỗi lần restart container, migrations chạy lại. EF Core xử lý tốt vì chỉ apply migrations chưa được apply (idempotent).

#### Pattern 2: Init Container (khuyên dùng)

```yaml
services:
  # Container chỉ chạy EF Core migrations rồi exit
  migrate:
    image: shoplite-backend:latest    # Dùng cùng image với backend
    command: dotnet ef database update --project ShopLite.Api
    depends_on:
      postgres:
        condition: service_healthy
    restart: "no"                     # Không restart sau khi xong
    environment:
      - ConnectionStrings__DefaultConnection=${DATABASE_CONNECTION_STRING}
      - ASPNETCORE_ENVIRONMENT=Production

  # Backend chỉ start sau khi migrate hoàn thành thành công
  backend:
    image: shoplite-backend:latest
    depends_on:
      migrate:
        condition: service_completed_successfully
      redis:
        condition: service_healthy
    environment:
      - ConnectionStrings__DefaultConnection=${DATABASE_CONNECTION_STRING}
      - ConnectionStrings__Redis=${REDIS_CONNECTION_STRING:-redis:6379}
```

Pattern này tốt hơn vì:
- Migrations chỉ chạy một lần tại startup
- Backend không start nếu migrations fail
- Separation of concerns rõ ràng

#### Pattern 3: Chạy migrations thủ công

```bash
# Chạy EF Core migrations khi cần (trước deploy)
docker compose run --rm backend dotnet ef database update --project ShopLite.Api

# Rollback về migration cụ thể
docker compose run --rm backend dotnet ef database update PreviousMigrationName --project ShopLite.Api

# Xem danh sách migrations và trạng thái applied/pending
docker compose run --rm backend dotnet ef migrations list --project ShopLite.Api
```

#### Seeding Database cho Development

```yaml
services:
  seed:
    image: shoplite-backend:latest
    command: dotnet run --project ShopLite.Api -- --seed
    depends_on:
      migrate:
        condition: service_completed_successfully
    restart: "no"
    profiles:
      - seed    # Chỉ chạy khi được chỉ định
```

```bash
# Chạy seed trong dev
docker compose --profile seed up
```

---

### 11. Tài liệu Onboarding "Chạy ShopLite Local trong 5 Phút"

Template này dành cho file `docs/local-setup.md` hoặc phần đầu của `README.md`:

```markdown
# Hướng dẫn Chạy ShopLite trên Máy Local

## Yêu cầu

| Tool | Version tối thiểu | Kiểm tra |
|------|-------------------|----------|
| Docker Desktop / Docker Engine | 24.0+ | `docker --version` |
| Docker Compose | V2 (plugin) | `docker compose version` |
| Git | 2.x | `git --version` |

> **Lưu ý cho Windows**: Sử dụng Docker Desktop với WSL2 backend.
> **Lưu ý cho Mac**: Docker Desktop cho Mac (Intel hoặc Apple Silicon đều được).

## Các bước cài đặt

### Bước 1: Clone repository

    git clone https://github.com/your-company/shoplite.git
    cd shoplite

### Bước 2: Tạo file cấu hình

    cp .env.example .env

Mở file `.env` và thay đổi ít nhất các giá trị sau:

    POSTGRES_PASSWORD=your-secure-password-here
    JWT_SECRET=your-random-secret-here

> Để tạo random secret: `openssl rand -base64 32`

### Bước 3: Khởi động toàn bộ hệ thống

    docker compose up -d

Lần đầu tiên có thể mất 3-5 phút để pull images.

### Bước 4: Kiểm tra hệ thống đã sẵn sàng

    # Xem trạng thái các services
    docker compose ps

    # Đợi tất cả services healthy, sau đó:
    curl http://localhost:8080/health

    # Mở trình duyệt
    open http://localhost       # Mac
    start http://localhost      # Windows

## Xác nhận cài đặt thành công

Bạn sẽ thấy:
- `docker compose ps` hiện tất cả services với status `(healthy)`
- `curl http://localhost:8080/health` trả về `{"status":"healthy"}`
- Trình duyệt mở `http://localhost` hiển thị trang chủ ShopLite
```

#### Reference Card - Lệnh hay dùng hàng ngày

In ra và dán cạnh màn hình:

```bash
# ─── HÀNG NGÀY ─────────────────────────────────────────
docker compose up -d              # Start tất cả (background)
docker compose down               # Stop + xóa containers
docker compose ps                 # Xem trạng thái
docker compose logs -f backend    # Theo dõi logs backend

# ─── DEBUG ─────────────────────────────────────────────
docker compose logs -f            # Xem tất cả logs
docker compose exec backend sh    # Vào shell backend (Alpine dùng sh)
docker compose exec postgres psql -U shoplite  # Vào psql
docker compose exec redis redis-cli            # Vào Redis CLI

# ─── BUILD ─────────────────────────────────────────────
docker compose up -d --build      # Rebuild + restart
docker compose build --no-cache backend  # Full rebuild backend

# ─── MAINTENANCE ───────────────────────────────────────
docker compose config             # Xem config đã resolve
docker compose run --rm backend dotnet ef database update  # EF Core migrations
docker compose down -v            # Xóa cả data (fresh start)

# ─── TROUBLESHOOTING ───────────────────────────────────
docker compose logs --tail=100 postgres  # 100 dòng log cuối
docker inspect shoplite-postgres | jq '.[] | .State.Health'
docker compose top                # Processes đang chạy
```

#### Troubleshooting thường gặp

**Lỗi: "port is already allocated"**

```bash
# Tìm process đang dùng port 5432
lsof -i :5432       # Mac/Linux
netstat -ano | findstr :5432  # Windows

# Kill process hoặc thay đổi port trong docker-compose.override.yml
```

**Lỗi: "no space left on device"**

```bash
# Dọn dẹp Docker resources không dùng
docker system prune -f
docker volume prune -f     # Cẩn thận: xóa volumes không dùng
```

**Backend không kết nối được database**

```bash
# Kiểm tra postgres có healthy không
docker compose ps postgres

# Kiểm tra logs
docker compose logs postgres

# Thử kết nối thủ công
docker compose exec backend nc -zv postgres 5432
```

**Lỗi "permission denied" trên Linux**

```bash
# Thêm user hiện tại vào docker group
sudo usermod -aG docker $USER

# Đăng xuất và đăng nhập lại để áp dụng
```

---

## Kỹ năng đạt được
- Mô tả toàn bộ stack bằng một file khai báo.
- Quản lý biến môi trường và bí mật cơ bản.
- Vận hành multi-container như một hệ thống.

---

## Thực hành

**Môi trường:** Docker + Docker Compose trên Ubuntu.

**Lab:**
- Viết `docker-compose.yml` gồm: frontend, backend, PostgreSQL, Redis.
- Cấu hình volume cho PostgreSQL, biến môi trường qua `.env`.
- Thêm healthcheck và `depends_on` để khởi động đúng thứ tự.
- Khởi động toàn hệ thống bằng `docker compose up`.

**Công cụ:** Docker Compose.

---

## Bài tập

- **Bắt buộc:** Toàn bộ ShopLite (4 service) chạy bằng một lệnh, dữ liệu DB bền vững sau khi restart.
- **Nâng cao:** Tạo `docker-compose.override.yml` cho môi trường dev (hot reload) tách khỏi prod.
- **Mô phỏng doanh nghiệp:** Viết tài liệu "cách dựng môi trường local trong 5 phút" cho thành viên mới.

---

## Deliverable
- [ ] `docker-compose.yml` chạy toàn bộ hệ thống.
- [ ] File `.env.example` mẫu.
- [ ] Tài liệu onboarding môi trường local.

---

## Tiêu chí hoàn thành
- [ ] `docker compose up` dựng được toàn bộ ShopLite, truy cập được từ trình duyệt.
- [ ] Dữ liệu DB còn nguyên sau khi `down` rồi `up` lại.
- [ ] Giải thích được luồng giao tiếp giữa các service trong Compose.

---

## Ghi chú học tập

> _Ghi lại những điều học được, vướng mắc và insight trong quá trình học phase này._
