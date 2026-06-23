# Phase 4 — Docker (Containerization)

## Mục tiêu
- Đóng gói từng service của ShopLite thành container chạy được ở bất kỳ đâu.
- Hiểu khác biệt giữa container và máy ảo, vì sao container thay đổi cách deploy.

---

## Kiến thức sẽ học

- **Khái niệm container & vì sao dùng** (giải quyết "máy tôi chạy được mà").
- **Image vs Container vs Registry** — mô hình cốt lõi.
- **Dockerfile** (FROM, RUN, COPY, CMD, ENTRYPOINT, layer caching).
- **Quản lý image & container** (build, run, exec, logs, ps, prune).
- **Volume & bind mount** — lưu dữ liệu bền vững.
- **Docker network cơ bản** — container nói chuyện với nhau.
- **Multi-stage build & tối ưu image** — giảm dung lượng, tăng bảo mật.
- **Image registry** (Docker Hub / GitHub Container Registry).

---

## Kiến thức chi tiết

### 1. Container vs VM — Hiểu từ gốc rễ

#### 1.1 Virtual Machine (Máy ảo)

Máy ảo ra đời để giải quyết bài toán: chạy nhiều hệ điều hành độc lập trên cùng một phần cứng vật lý. Cơ chế hoạt động dựa trên **hypervisor** — một lớp phần mềm nằm giữa phần cứng và các hệ điều hành ảo.

**Hai loại hypervisor:**

**Type 1 — Bare-metal hypervisor** (chạy thẳng trên phần cứng, không cần host OS):
- **VMware ESXi**: chuẩn mực trong enterprise, dùng rộng rãi ở data center, hỗ trợ vMotion (di chuyển VM đang chạy giữa các host).
- **Microsoft Hyper-V**: tích hợp sẵn trong Windows Server, dùng nhiều trong môi trường Microsoft.
- **KVM (Kernel-based Virtual Machine)**: built-in trong Linux kernel từ 2007, nền tảng của OpenStack và nhiều cloud provider.
- **Xen**: dùng trong AWS EC2 thế hệ đầu, nay AWS chuyển sang Nitro (KVM-based).

**Type 2 — Hosted hypervisor** (chạy như application trên host OS):
- **Oracle VirtualBox**: miễn phí, cross-platform, phổ biến cho dev local.
- **VMware Workstation/Fusion**: có phí, hiệu năng tốt hơn VirtualBox.
- **Parallels Desktop**: phổ biến trên macOS (đặc biệt khi chạy Windows trên Mac).

**Đặc điểm của VM:**
- Mỗi VM có **guest OS đầy đủ**: kernel riêng, filesystem riêng, system processes riêng.
- **Dung lượng lớn**: một VM Ubuntu minimal cũng chiếm 1-2 GB disk, Windows VM có thể 20-40 GB.
- **Khởi động chậm**: phải boot đầy đủ OS, BIOS/UEFI, init system — mất 1-3 phút.
- **Tiêu tốn RAM**: mỗi VM cần RAM cho guest OS (512 MB - 2 GB chỉ để chạy OS).
- **Cách ly mạnh**: mỗi VM có kernel riêng biệt hoàn toàn, lỗi trong VM không ảnh hưởng host.
- **Phù hợp khi**: cần chạy OS khác nhau (Windows trên Linux host), cần cách ly hoàn toàn ở cấp kernel, môi trường legacy.

#### 1.2 Container

Container ra đời từ nhu cầu khác: đóng gói và chạy application một cách nhất quán mà không cần overhead của VM. Container **chia sẻ kernel của host OS** nhưng được cách ly với nhau.

**Hai cơ chế Linux kernel làm nền tảng cho container:**

**Linux Namespaces — Cách ly tài nguyên:**

Namespace tạo ra "cái nhìn riêng biệt" cho mỗi process group. Docker sử dụng 6 loại namespace:

- **PID namespace**: Container thấy process tree riêng. Process đầu tiên trong container có PID 1 (thực ra trên host nó có thể là PID 4521). Khi PID 1 trong container chết, toàn bộ container dừng.
- **NET namespace**: Mỗi container có network stack riêng: network interfaces riêng (eth0, lo), routing table riêng, iptables rules riêng, port space riêng (container A và B đều có thể listen port 3000 mà không xung đột).
- **MNT namespace (Mount)**: Filesystem view riêng. Container thấy filesystem của mình, không thấy filesystem của host hay container khác.
- **UTS namespace (Unix Time-sharing System)**: Mỗi container có hostname và domain name riêng. Đó là lý do container có thể đặt hostname khác host.
- **IPC namespace**: Cách ly inter-process communication: System V IPC, POSIX message queues. Container A không thể gửi signal hay shared memory tới container B.
- **USER namespace**: Map user IDs giữa container và host. Root trong container (UID 0) có thể map tới non-root user trên host (UID 1000) — đây là nền tảng của rootless containers.

**Linux cgroups (Control Groups) — Giới hạn tài nguyên:**

cgroups không chỉ cách ly mà còn **giới hạn và đo lường** tài nguyên:

- **CPU**: giới hạn phần trăm CPU một container được dùng. `--cpus="0.5"` tức container dùng tối đa 50% của 1 CPU core.
- **Memory**: giới hạn RAM. `--memory="512m"` tức container bị kill (OOMKill) nếu dùng quá 512 MB.
- **Disk I/O**: giới hạn tốc độ đọc/ghi disk để tránh một container làm saturate storage.
- **Network bandwidth**: (qua tc/netem kết hợp cgroups) giới hạn băng thông mạng.
- **PIDs**: giới hạn số process tối đa trong container, phòng fork bomb.

**Đặc điểm của Container:**
- **Shared kernel**: tất cả container trên một host dùng chung kernel của host OS.
- **Nhẹ**: image Alpine Linux chỉ ~5 MB, Node.js Alpine ~50 MB, ứng dụng thực tế ~100-200 MB.
- **Khởi động nhanh**: không cần boot OS, chỉ cần start process — thường dưới 1 giây.
- **Tiêu tốn ít RAM**: chỉ cần RAM cho application, không overhead cho guest OS kernel.
- **Mật độ cao**: có thể chạy hàng trăm container trên một máy chủ bình thường.

#### 1.3 Vấn đề "Works on My Machine"

Đây là vấn đề cổ điển trong phần mềm:
- Developer A code trên macOS, dùng .NET SDK 8.0.100, Npgsql 8.0.
- CI server chạy Ubuntu 22.04, .NET SDK 7.0, Npgsql 7.0.
- Production server chạy CentOS Stream 9, .NET Runtime 6.0.
- Kết quả: app chạy tốt trên máy dev, lỗi trên CI, crash trên production.

Container giải quyết bằng cách **đóng gói toàn bộ môi trường vào image**:
```
Image ShopLite Backend chứa:
  - Alpine Linux base filesystem
  - ASP.NET Core 8.0 runtime (exact version)
  - NuGet packages (packages.lock.json locked)
  - Environment variables
  - Config files
  - Compiled .NET publish artifacts (.dll, appsettings.json)
```

Image này chạy **giống hệt nhau** trên laptop dev, CI server, staging, production — miễn là host có Docker Engine.

---

### 2. Docker Architecture — Kiến trúc từng thành phần

#### 2.1 Docker Daemon (dockerd)

`dockerd` là tiến trình chạy nền trên host, lắng nghe các API request. Nó chịu trách nhiệm:
- Quản lý lifecycle của containers (create, start, stop, kill, remove).
- Build images từ Dockerfile.
- Pull/push images từ/lên registry.
- Quản lý volumes và networks.
- Giao tiếp với containerd (runtime cấp thấp hơn).

Dockerd lắng nghe trên Unix socket `/var/run/docker.sock` (mặc định) hoặc TCP socket (nếu cấu hình remote access).

```bash
# Xem trạng thái dockerd
sudo systemctl status docker

# Xem logs của dockerd
sudo journalctl -u docker -f

# Cấu hình dockerd
sudo cat /etc/docker/daemon.json
```

**Ví dụ daemon.json cho production:**
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "5"
  },
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 64000,
      "Soft": 64000
    }
  },
  "live-restore": true,
  "storage-driver": "overlay2"
}
```

#### 2.2 Docker CLI (docker)

`docker` là công cụ dòng lệnh mà người dùng tương tác. Bản thân CLI không làm gì nặng nề — nó chỉ:
1. Parse lệnh người dùng nhập.
2. Chuyển thành HTTP request.
3. Gửi tới Docker daemon qua Unix socket hoặc TCP.
4. Nhận response và hiển thị cho người dùng.

```bash
# Khi bạn chạy:
docker run nginx

# CLI gửi HTTP POST request tới daemon:
# POST /v1.41/containers/create
# POST /v1.41/containers/{id}/start
```

#### 2.3 containerd

containerd là container runtime được tách ra khỏi Docker từ 2017, trở thành CNCF project. Nó nằm giữa dockerd và runc:

- Quản lý container lifecycle ở cấp thấp hơn dockerd.
- Quản lý image storage (pull, push, unpack layers).
- Quản lý snapshots (filesystem layers).
- Kubernetes dùng containerd trực tiếp (không cần Docker).

```bash
# Xem containerd trên hệ thống
sudo systemctl status containerd

# containerd CLI (dùng trong debugging/advanced)
sudo ctr containers list
sudo ctr images list
```

#### 2.4 runc

runc là OCI (Open Container Initiative) runtime chuẩn, thực sự tạo và chạy containers:

- Đọc OCI bundle (config.json + rootfs).
- Gọi Linux kernel APIs: clone() với namespace flags, cgroups, chroot.
- Start process trong môi trường đã cách ly.
- runc là reference implementation của OCI Runtime Spec.

```bash
# runc được gọi bởi containerd, bình thường không dùng trực tiếp
which runc
runc --version
```

**Luồng đầy đủ khi `docker run nginx`:**
```
docker CLI
  -> HTTP POST /containers/create -> dockerd
     -> containerd (tạo container spec)
        -> runc (clone() Linux namespaces, cgroups)
           -> nginx process chạy trong container
```

#### 2.5 Docker Hub

Docker Hub là public registry mặc định tại `hub.docker.com`:
- **Official images**: `nginx`, `postgres`, `redis`, `node` — được Docker Inc. maintain.
- **Verified Publisher**: `bitnami/postgresql`, `elastic/elasticsearch` — từ các công ty uy tín.
- **Community images**: `username/image-name` — từ cộng đồng.
- Free tier: unlimited public repos, 1 private repo, rate limit 100 pulls/6h (anonymous), 200 pulls/6h (authenticated).

---

### 3. Image & Layer System — Hệ thống layers

#### 3.1 Image là gì

Image là **read-only template** dùng để tạo containers. Image không phải là một file đơn lẻ mà là một tập hợp các **layers** xếp chồng lên nhau.

Một image có:
- Nhiều filesystem layers (read-only).
- Metadata: environment variables, exposed ports, default command, author, etc.
- Image ID (SHA256 hash của toàn bộ nội dung).
- Một hoặc nhiều tags (human-readable names).

#### 3.2 Layer và Union Filesystem

Mỗi instruction trong Dockerfile tạo ra một **layer mới**:

```
Layer 0 (base): mcr.microsoft.com/dotnet/aspnet:8.0-alpine (~103 MB)
Layer 1: COPY ShopLite.Api/*.csproj ./ShopLite.Api/ (~5 KB)
Layer 2: RUN dotnet restore (~50 MB NuGet packages)
Layer 3: COPY ShopLite.Api/ . (~2 MB source code)
Layer 4: RUN dotnet publish -c Release -o /app/publish (~5 MB)
```

Docker dùng **overlay2** filesystem driver (mặc định trên Linux hiện đại) để stack các layers:

```
upperdir (writable layer - container specific)
      |
layer 4: /app/publish (compiled .dll artifacts)
      |
layer 3: /src/ShopLite.Api (source code .cs files)
      |
layer 2: /root/.nuget/packages (NuGet packages)
      |
layer 1: /src/ShopLite.Api/*.csproj
      |
layer 0: /bin, /lib, /usr, /etc (aspnet alpine base)
```

Khi container đọc file, overlay2 tìm từ upperdir xuống: tìm thấy ở layer nào thì dùng layer đó.
Khi container ghi file, overlay2 dùng **copy-on-write**: copy file từ lower layer lên upperdir rồi mới ghi.

#### 3.3 Layer Caching — Tại sao build nhanh lần 2

Docker cache từng layer. Nếu:
- Instruction không thay đổi **và**
- Tất cả layers trước nó không thay đổi

thì Docker dùng lại cached layer thay vì build lại.

**Thứ tự instruction ảnh hưởng lớn đến cache efficiency:**

```dockerfile
# XẤU: thay đổi source code sẽ invalidate dotnet restore cache
COPY . .
RUN dotnet restore

# TỐT: chỉ khi .csproj thay đổi mới re-run dotnet restore
COPY *.csproj ./
RUN dotnet restore
COPY . .
```

Với cách tốt, trong development, chỉ thay đổi source code sẽ chỉ re-run `COPY . .` (nhanh), không phải `dotnet restore` (chậm).

#### 3.4 Container Layer

Khi tạo container từ image, Docker thêm một **writable layer** lên trên các read-only image layers:

```
writable layer (container) <- mọi thay đổi runtime ghi vào đây
      |
image layers (read-only)
```

Khi container bị xóa, writable layer bị xóa, các image layers vẫn còn.
Đây là lý do dữ liệu trong container bị mất khi container bị xóa — phải dùng volume để persist.

**Nhiều container từ cùng một image** dùng chung image layers, mỗi container chỉ có writable layer riêng:
```
Container A writable layer
Container B writable layer    <- mỗi container 1 writable layer riêng
Container C writable layer
         |
    Image layers              <- chia sẻ chung, tiết kiệm disk
```

#### 3.5 Commands quản lý images

```bash
# Liệt kê images
docker image ls
docker image ls --filter dangling=true  # images không có tag

# Xem thông tin chi tiết một image
docker image inspect nginx
docker image inspect nginx | jq '.[0].Config.Env'       # environment vars
docker image inspect nginx | jq '.[0].Config.ExposedPorts'  # exposed ports
docker image inspect nginx | jq '.[0].RootFS.Layers | length'  # số layers

# Xem lịch sử layers của image
docker image history nginx
docker image history --no-trunc nginx  # không truncate command

# Xóa images
docker image rm nginx:alpine
docker image prune         # xóa dangling images (không có tag)
docker image prune -a      # xóa tất cả unused images (nguy hiểm trên prod)
docker image prune -a --filter "until=24h"  # xóa images cũ hơn 24 giờ

# Build image
docker build -t shoplite-backend:v1.0.0 .
docker build -t shoplite-backend:v1.0.0 -f Dockerfile.prod .  # chỉ định Dockerfile
docker build --no-cache -t shoplite-backend:v1.0.0 .  # build không dùng cache

# Save/load image (offline transfer)
docker save shoplite-backend:v1.0.0 | gzip > shoplite-backend-v1.0.0.tar.gz
docker load < shoplite-backend-v1.0.0.tar.gz
```

---

### 4. Dockerfile — Giải thích từng instruction

Dockerfile là file text chứa các instructions để build Docker image. Mỗi instruction thường tạo một layer mới.

#### 4.1 FROM — Base image

```dockerfile
FROM ubuntu:22.04
```

- Instruction bắt buộc, phải là instruction đầu tiên (trừ ARG).
- Chỉ định base image để xây dựng tiếp.
- **Luôn pin version cụ thể** trong production, không dùng `:latest` vì:
  - `:latest` thay đổi khi có release mới.
  - Build hôm nay và build tháng sau có thể ra image khác nhau.
  - Khó debug khi base image thay đổi ngầm.

```dockerfile
# Dùng scratch cho static binary (Go, Rust)
FROM scratch
COPY myapp /myapp
CMD ["/myapp"]

# Multi-stage: dùng nhiều FROM
FROM node:18-alpine AS builder
FROM nginx:alpine AS runner
```

#### 4.2 ARG — Build-time variable

```dockerfile
ARG DOTNET_VERSION=8.0
ARG BUILD_DATE
ARG GIT_COMMIT
```

- Biến chỉ có tác dụng **trong quá trình build**.
- Không tồn tại trong image cuối và không có trong container runtime.
- Có thể override khi build: `docker build --build-arg DOTNET_VERSION=9.0 .`
- Dùng kết hợp với FROM: `FROM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION}-alpine`
- **Không dùng ARG cho secrets** — ARG value hiện trong `docker history`.

```dockerfile
ARG DOTNET_VERSION=8.0
FROM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION}-alpine AS builder

# ARG phía trên FROM chỉ dùng được trong FROM
# Để dùng sau FROM, cần khai báo lại:
ARG DOTNET_VERSION=8.0
RUN echo "Building with .NET SDK ${DOTNET_VERSION}"
```

#### 4.3 ENV — Runtime variable

```dockerfile
ENV ASPNETCORE_ENVIRONMENT=Production
ENV ASPNETCORE_URLS=http://+:8080
ENV ConnectionStrings__Redis=redis:6379
```

- Tồn tại trong image và available khi container chạy.
- Có thể override khi run: `docker run -e ASPNETCORE_ENVIRONMENT=Staging image`.
- Khác ARG: ENV persist trong image, visible trong `docker inspect`.
- **Không dùng ENV cho secrets** — hiện trong `docker inspect` và image metadata.
- Dùng Docker secrets hoặc volume mount cho passwords/tokens.

```dockerfile
# Đặt nhiều biến trên một dòng
ENV ASPNETCORE_ENVIRONMENT=Production \
    ASPNETCORE_URLS=http://+:8080 \
    Logging__LogLevel__Default=Warning
```

#### 4.4 WORKDIR — Working directory

```dockerfile
WORKDIR /app
```

- Set working directory cho các instructions phía sau (RUN, COPY, ADD, CMD, ENTRYPOINT).
- Tạo directory nếu chưa tồn tại.
- Luôn dùng **absolute path**.
- Có thể stack: `WORKDIR /app` rồi `WORKDIR src` -> hiện tại là `/app/src`.
- Tốt hơn `RUN cd /app` vì `RUN cd` không persist giữa các RUN layers.

```dockerfile
# Không nên
RUN mkdir -p /app && cd /app

# Nên dùng
WORKDIR /app
```

#### 4.5 COPY — Copy files

```dockerfile
COPY package*.json ./
COPY src/ ./src/
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

- Copy files/directories từ build context (local filesystem) vào image.
- `COPY <src> <dest>`: src là relative to build context, dest là absolute hoặc relative to WORKDIR.
- `COPY --chown=user:group`: set ownership khi copy.
- `COPY --from=stage`: copy từ stage khác trong multi-stage build.

```dockerfile
# Copy và đặt ownership ngay
COPY --chown=appuser:appgroup ShopLite.Api/*.csproj ./ShopLite.Api/

# Copy từ stage khác
COPY --from=builder --chown=appuser:appgroup /app/publish .
```

**ADD vs COPY:**
- `ADD` làm được tất cả những gì `COPY` làm, cộng thêm:
  - Tự động extract tar archives.
  - Fetch từ URL (không recommend — tốt hơn dùng RUN curl).
- **Prefer COPY** vì tường minh hơn, ADD có behavior ngầm có thể gây nhầm lẫn.

#### 4.6 RUN — Execute command khi build

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*
```

- Execute command trong shell và commit kết quả thành layer mới.
- **Mỗi RUN tạo một layer** — gộp các lệnh liên quan vào một RUN để giảm layers.
- **Clean up trong cùng RUN** vì xóa file ở RUN tiếp theo không giảm được layer size.

```dockerfile
# SAI: rm không giảm size layer trước
RUN apt-get update && apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*   # layer trước vẫn chứa lists

# ĐÚNG: gộp lại, dọn dẹp trong cùng RUN
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
```

**Exec form vs Shell form:**
```dockerfile
# Shell form: chạy qua /bin/sh -c "..."
RUN apt-get update

# Exec form: chạy trực tiếp, không qua shell
RUN ["apt-get", "update"]
```

#### 4.7 EXPOSE — Document port

```dockerfile
EXPOSE 3000
EXPOSE 80/tcp
EXPOSE 53/udp
```

- **Chỉ là documentation** — không thực sự mở port hay cho phép access từ bên ngoài.
- Báo cho người dùng image biết container dùng port nào.
- Port mapping thực sự qua `docker run -p <host-port>:<container-port>`.
- Dùng kết hợp với `docker run -P` (uppercase P) để Docker tự mapping tất cả exposed ports sang random host ports.

#### 4.8 USER — Run as non-root

```dockerfile
RUN addgroup -g 1001 -S appgroup && adduser -S -u 1001 -G appgroup appuser
USER appuser
```

- **Security best practice**: chạy application dưới non-root user.
- Root trong container = root trên host (nếu không có USER namespace mapping).
- Nếu container bị compromise, attacker không có root access trên host.
- Tạo user trước khi dùng USER instruction.

```dockerfile
# Tạo system user/group (không có shell, không có home directory login)
RUN addgroup -g 1001 -S appgroup && \
    adduser -S -u 1001 -G appgroup appuser

USER appuser
```

#### 4.9 HEALTHCHECK — Container health monitoring

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
    CMD wget -qO- http://localhost:8080/health || exit 1
```

- Định nghĩa cách Docker kiểm tra container có healthy không.
- `--interval`: kiểm tra mỗi bao lâu (default 30s).
- `--timeout`: sau bao lâu coi là failed (default 30s).
- `--start-period`: chờ bao lâu trước lần check đầu (cho app khởi động).
- `--retries`: bao nhiêu lần fail liên tiếp mới coi là unhealthy (default 3).
- Container status: `starting` -> `healthy` hoặc `unhealthy`.
- Docker Swarm và Kubernetes dùng healthcheck để quyết định restart container.

```dockerfile
# Dùng wget (Alpine không có curl mặc định)
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s \
    CMD wget -qO- http://localhost:8080/health || exit 1
```

#### 4.10 CMD và ENTRYPOINT — Default command

```dockerfile
# CMD: default command, có thể override hoàn toàn khi docker run
CMD ["node", "src/index.js"]

# ENTRYPOINT: fixed executable, CMD là default args
ENTRYPOINT ["node"]
CMD ["src/index.js"]
```

**Exec form (array) vs Shell form (string):**

```dockerfile
# Shell form: /bin/sh -c "node src/index.js"
CMD node src/index.js
# Vấn đề: PID 1 là /bin/sh, không phải node
# Signal SIGTERM không được forward tới node process
# Graceful shutdown không hoạt động đúng

# Exec form: chạy trực tiếp
CMD ["node", "src/index.js"]
# node là PID 1, nhận SIGTERM trực tiếp
# Graceful shutdown hoạt động
```

**Prefer exec form** cho CMD và ENTRYPOINT trong production vì signal handling đúng.

**Sự khác biệt CMD và ENTRYPOINT:**

```bash
# Chỉ có CMD
# docker run image              -> node src/index.js
# docker run image custom.js   -> custom.js (override hoàn toàn)

# ENTRYPOINT + CMD
# docker run image              -> node src/index.js
# docker run image custom.js   -> node custom.js (chỉ override args)
# docker run --entrypoint /bin/sh image  -> /bin/sh (override entrypoint)
```

**ENTRYPOINT dùng cho:**
- Images được dùng như executable: `docker run mytool --flag value`.
- Wrapper scripts cần chạy trước app (init, wait-for-it, env injection).

```dockerfile
# Wrapper script pattern
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["node", "src/index.js"]
```

---

### 5. Multi-stage Build cho .NET Backend ShopLite

Multi-stage build giải quyết vấn đề: image production chỉ cần ASP.NET Core **runtime** (nhỏ hơn nhiều so với SDK), không cần build tools hay source code.

```dockerfile
# ============================================================
# Stage 1: Builder — restore NuGet packages, compile, publish
# ============================================================
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS builder

WORKDIR /src

# Copy project file trước để tận dụng layer cache
# Chỉ re-run dotnet restore khi .csproj thay đổi
COPY ShopLite.Api/*.csproj ./ShopLite.Api/

# dotnet restore: download NuGet packages
# --locked-mode: dùng packages.lock.json để đảm bảo reproducible build
RUN dotnet restore ./ShopLite.Api/ShopLite.Api.csproj --locked-mode

# Copy toàn bộ source code (sau restore để cache tầng restore)
COPY ShopLite.Api/ ./ShopLite.Api/

# Publish với cấu hình Release
# -c Release: tối ưu hóa, bỏ debug symbols
# -o /app/publish: output compiled artifacts vào đây
# --no-restore: đã restore ở bước trên rồi
# --self-contained false: dùng runtime từ base image (image nhỏ hơn)
RUN dotnet publish ./ShopLite.Api/ShopLite.Api.csproj \
    -c Release \
    -o /app/publish \
    --no-restore \
    --self-contained false

# ============================================================
# Stage 2: Runner — chỉ chứa ASP.NET Core runtime + published app
# ============================================================
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS runner

# Tạo non-root user/group
RUN addgroup -g 1001 -S appgroup && \
    adduser -S -u 1001 -G appgroup appuser

WORKDIR /app

# ASPNETCORE_ENVIRONMENT: Production tắt detailed error pages, bật caching
ENV ASPNETCORE_ENVIRONMENT=Production
# ASPNETCORE_URLS: lắng nghe trên port 8080 (không dùng 80 vì cần root)
ENV ASPNETCORE_URLS=http://+:8080

# Copy published output từ builder stage
# --chown: set ownership khi copy, không cần chmod riêng
COPY --from=builder --chown=appuser:appgroup /app/publish .

# Chạy với non-root user
USER appuser

# Document port (không mở thực sự)
EXPOSE 8080

# Health check — Docker tự động restart nếu unhealthy
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
    CMD wget -qO- http://localhost:8080/health || exit 1

# Exec form: dotnet là PID 1, nhận SIGTERM trực tiếp
# ASP.NET Core tự xử lý graceful shutdown khi nhận SIGTERM
ENTRYPOINT ["dotnet", "ShopLite.Api.dll"]
```

**Giải thích tại sao image giảm từ ~400 MB xuống ~110 MB:**

| Thành phần | Builder stage | Runner stage |
|---|---|---|
| dotnet/sdk:8.0-alpine base | ~300 MB | Không dùng |
| NuGet packages (restore) | ~80 MB | Không có |
| Source code (.cs files) | ~2 MB | Không có |
| dotnet/aspnet:8.0-alpine base | — | ~103 MB |
| Published artifacts (.dll) | ~5 MB | ~5 MB |
| **Total** | **~387 MB** | **~108 MB** |

SDK (Roslyn compiler, MSBuild, build tools) chỉ cần trong builder stage. Runner stage chỉ cần `aspnet` runtime — nhỏ hơn `sdk` khoảng 3 lần.

**Build và chạy:**

```bash
# Build image
docker build -t shoplite-backend:v1.0.0 .

# Xem kết quả
docker image ls shoplite-backend

# Chạy container
docker run -d \
  --name shoplite-backend \
  -p 8080:8080 \
  -e ConnectionStrings__DefaultConnection="Host=postgres;Database=shoplite;Username=shoplite;Password=secret" \
  -e ConnectionStrings__Redis="redis:6379" \
  -e ASPNETCORE_ENVIRONMENT=Production \
  --network shoplite-net \
  shoplite-backend:v1.0.0

# Verify health
docker ps  # xem STATUS: Up X minutes (healthy)
```

**Với project có nhiều layers (.sln + nhiều .csproj):**

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS builder

WORKDIR /src

# Copy solution và tất cả .csproj trước để restore hiệu quả
COPY ShopLite.sln .
COPY ShopLite.Api/*.csproj ./ShopLite.Api/
COPY ShopLite.Domain/*.csproj ./ShopLite.Domain/
COPY ShopLite.Infrastructure/*.csproj ./ShopLite.Infrastructure/

RUN dotnet restore ShopLite.sln --locked-mode

COPY . .

# Publish chỉ project API (dependencies tự động được include)
RUN dotnet publish ./ShopLite.Api/ShopLite.Api.csproj \
    -c Release -o /app/publish --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS runner
# ... (phần còn lại giống ở trên)
```

---

### 6. Multi-stage Build cho React Frontend

```dockerfile
# ============================================================
# Stage 1: Build React app
# ============================================================
FROM node:18-alpine AS builder

WORKDIR /app

# Copy dependency files
COPY package*.json ./

# Install all dependencies (including devDeps for build)
RUN npm ci

# Copy source code
COPY . .

# Build production bundle
# Kết quả: /app/dist hoặc /app/build
RUN npm run build

# ============================================================
# Stage 2: Serve với Nginx
# ============================================================
FROM nginx:alpine AS runner

# Copy built static files từ builder stage
COPY --from=builder /app/dist /usr/share/nginx/html

# Copy custom Nginx config
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

# nginx -g "daemon off;": chạy nginx ở foreground (không background)
# Cần thiết để container không thoát ngay sau khi start
CMD ["nginx", "-g", "daemon off;"]
```

**File nginx.conf đi kèm:**

```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # SPA routing: redirect tất cả về index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API calls tới backend
    location /api/ {
        proxy_pass http://shoplite-backend:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

**Build và chạy:**

```bash
docker build -t shoplite-frontend:v1.0.0 .
docker run -d \
  --name shoplite-frontend \
  -p 80:80 \
  --network shoplite-net \
  shoplite-frontend:v1.0.0
```

---

### 7. .dockerignore — Quan trọng, thường bị bỏ quên

`.dockerignore` hoạt động giống `.gitignore` nhưng cho Docker build context.

**Tại sao quan trọng:**

Khi chạy `docker build .`, Docker gửi toàn bộ directory (build context) tới daemon. Nếu không có `.dockerignore`:
- `bin/` và `obj/` (~100-500 MB) bị copy vào build context dù Docker sẽ build lại từ source.
- `.git` directory (~50-200 MB) chứa toàn bộ lịch sử commit bị copy dù không cần.
- `.env` files chứa secrets bị đưa vào build context (và có thể vào image).
- Tốc độ build chậm vì phải transfer lượng data lớn.

**File .dockerignore đầy đủ cho .NET project:**

```
# Build artifacts — sẽ được build lại bên trong Docker
bin/
obj/
publish/

# Visual Studio / JetBrains Rider artifacts
.vs/
.idea/
*.user
*.suo
*.DotSettings.user

# Git
.git/
.gitignore
.gitattributes

# Docker files themselves
Dockerfile
Dockerfile.*
.dockerignore

# Environment files (NEVER copy secrets into image)
.env
.env.local
.env.*.local
appsettings.Development.json
appsettings.Local.json

# Editor/IDE files
.vscode/
*.swp
.DS_Store
Thumbs.db

# Test projects và artifacts
**Tests/
**Test/
TestResults/
coverage/
*.trx
*.coverage
*.coveragexml

# Logs
logs/
*.log

# Documentation
*.md
docs/

# CI/CD configs
.github/
.gitlab-ci.yml
.travis.yml
Jenkinsfile

# Misc
*.nupkg
.packages/
```

**Kiểm tra build context size:**

```bash
# Xem build context size trước khi build
docker build . 2>&1 | head -5
# Output: Sending build context to Docker daemon  5.632kB  <- tốt
# So sánh: 523.1MB                                         <- quá lớn
```

---

### 8. Container Management Commands

#### 8.1 docker run — Tạo và chạy container

```bash
docker run -d \
  --name shoplite-backend \
  -p 8080:8080 \
  -e ConnectionStrings__DefaultConnection="Host=postgres;Database=shoplite;Username=user;Password=pass" \
  -e ConnectionStrings__Redis="redis:6379" \
  -e ASPNETCORE_ENVIRONMENT=Production \
  -v /data/uploads:/app/uploads \
  --network shoplite-net \
  --restart unless-stopped \
  --memory="512m" \
  --cpus="0.5" \
  --log-driver json-file \
  --log-opt max-size=100m \
  --log-opt max-file=5 \
  shoplite-backend:v1.0.0
```

**Giải thích từng flag:**
- `-d`: detached mode, chạy nền.
- `--name`: đặt tên container (không bắt buộc nhưng nên đặt).
- `-p 3000:3000`: mapping `<host-port>:<container-port>`.
- `-e`: set environment variable.
- `-v /host/path:/container/path`: bind mount volume.
- `--network`: kết nối vào Docker network.
- `--restart unless-stopped`: tự restart trừ khi bị dừng tay.
- `--memory="512m"`: giới hạn RAM 512 MB.
- `--cpus="0.5"`: giới hạn 50% của 1 CPU core.

**Các giá trị --restart:**
```bash
--restart no             # không restart (default)
--restart always         # luôn restart (kể cả sau docker restart)
--restart unless-stopped # restart trừ khi dừng bằng docker stop
--restart on-failure     # chỉ restart khi exit code != 0
--restart on-failure:3   # restart tối đa 3 lần
```

#### 8.2 docker exec — Thực thi lệnh trong container đang chạy

```bash
# Mở interactive shell trong container
docker exec -it shoplite-backend /bin/sh

# Chạy command một lần không interactive
docker exec shoplite-backend node --version
docker exec shoplite-backend cat /app/package.json
docker exec shoplite-backend ls -la /app/dist

# Chạy với user khác
docker exec -u root shoplite-backend chown -R nodejs:nodejs /app/uploads

# Set environment variable
docker exec -e DEBUG=true shoplite-backend node debug-script.js
```

#### 8.3 docker logs — Xem logs

```bash
# Xem tất cả logs
docker logs shoplite-backend

# Follow (streaming, giống tail -f)
docker logs -f shoplite-backend

# Xem 100 dòng cuối
docker logs --tail 100 shoplite-backend

# Follow và xem 50 dòng cuối
docker logs -f --tail 50 shoplite-backend

# Filter theo thời gian
docker logs --since 2h shoplite-backend          # 2 giờ trước đến nay
docker logs --since "2024-01-15T10:00:00" shoplite-backend
docker logs --until "2024-01-15T12:00:00" shoplite-backend
docker logs --since 1h --until 30m shoplite-backend  # từ 1h trước đến 30 phút trước

# Xem cả stdout và stderr
docker logs shoplite-backend 2>&1 | grep ERROR
```

#### 8.4 docker ps — Liệt kê containers

```bash
# Containers đang chạy
docker ps

# Tất cả containers (bao gồm đã dừng)
docker ps -a

# Custom format
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.CreatedAt}}\t{{.Status}}"

# Chỉ lấy Container IDs
docker ps -q        # đang chạy
docker ps -aq       # tất cả

# Filter
docker ps --filter "status=exited"
docker ps --filter "name=shoplite"
docker ps --filter "ancestor=nginx:alpine"  # container từ image này
```

#### 8.5 Stop, Kill, Remove containers

```bash
# Graceful stop: gửi SIGTERM, chờ 10s, nếu không dừng thì SIGKILL
docker stop shoplite-backend
docker stop --time 30 shoplite-backend  # chờ 30 giây

# Immediate kill: gửi SIGKILL ngay
docker kill shoplite-backend
docker kill --signal SIGUSR1 shoplite-backend  # gửi custom signal

# Remove container (phải stop trước)
docker rm shoplite-backend

# Force remove (stop + remove)
docker rm -f shoplite-backend

# Remove tất cả stopped containers
docker container prune
docker container prune --filter "until=24h"  # chỉ xóa cái cũ hơn 24h

# Remove tất cả: containers, images, volumes, networks không dùng
docker system prune
docker system prune -a --volumes  # bao gồm cả volumes (nguy hiểm!)
```

#### 8.6 Monitoring và Inspection

```bash
# Realtime resource usage (giống top cho containers)
docker stats

# Snapshot một lần (không streaming)
docker stats --no-stream

# Custom format
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"

# Inspect: xem toàn bộ metadata của container
docker inspect shoplite-backend

# Trích xuất thông tin cụ thể với jq
docker inspect shoplite-backend | jq '.[0].State'                    # trạng thái
docker inspect shoplite-backend | jq '.[0].NetworkSettings.Networks' # network config
docker inspect shoplite-backend | jq '.[0].HostConfig.RestartPolicy' # restart policy
docker inspect shoplite-backend | jq '.[0].Mounts'                   # volumes
docker inspect shoplite-backend | jq '.[0].State.ExitCode'           # exit code
docker inspect shoplite-backend | jq '.[0].State.OOMKilled'          # bị OOM kill?

# Xem processes trong container
docker top shoplite-backend

# Xem port mappings
docker port shoplite-backend
```

#### 8.7 Copying files

```bash
# Copy từ local vào container
docker cp ./config.json shoplite-backend:/app/config.json
docker cp ./migrations/ shoplite-backend:/app/migrations/

# Copy từ container ra local
docker cp shoplite-backend:/app/logs/ ./local-logs/
docker cp shoplite-backend:/app/.env.example ./.env.example

# Dùng để lấy file từ container đã stopped
docker create --name temp-container shoplite-backend:v1.0.0
docker cp temp-container:/app/dist ./dist-backup
docker rm temp-container
```

---

### 9. Volumes — Lưu trữ dữ liệu bền vững

Container có writable layer nhưng nó bị xóa khi container bị remove. Volumes giải quyết vấn đề persist data.

#### 9.1 Named Volumes (Khuyên dùng cho data persistence)

```bash
# Tạo volume
docker volume create postgres-data
docker volume create redis-data
docker volume create uploads-data

# Liệt kê volumes
docker volume ls

# Xem thông tin volume
docker volume inspect postgres-data
# Output bao gồm Mountpoint trên host filesystem

# Dùng volume khi run container
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=shoplite \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:15-alpine

docker run -d \
  --name redis \
  -v redis-data:/data \
  redis:7-alpine

# Xóa volume (chỉ khi không có container nào dùng)
docker volume rm postgres-data

# Xóa tất cả volumes không dùng (NGUY HIỂM: xóa data!)
docker volume prune
```

**Ưu điểm named volume:**
- Docker quản lý location trên host.
- Dễ backup: `docker run --rm -v postgres-data:/data -v $(pwd):/backup alpine tar czf /backup/backup.tar.gz /data`.
- Platform independent: chạy được trên Linux, Mac, Windows.
- Tốt hơn bind mount về performance khi dùng Docker Desktop.

#### 9.2 Bind Mounts (Khuyên dùng cho development hot-reload)

```bash
# Mount current directory vào /src (để dotnet watch detect thay đổi)
docker run -d \
  --name shoplite-dev \
  -p 8080:8080 \
  -v $(pwd):/src \
  -e ASPNETCORE_ENVIRONMENT=Development \
  -e DOTNET_USE_POLLING_FILE_WATCHER=true \
  mcr.microsoft.com/dotnet/sdk:8.0-alpine \
  dotnet watch run --project ShopLite.Api --no-launch-profile

# Read-only bind mount (ví dụ: config file)
docker run -d \
  --name nginx \
  -p 80:80 \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  nginx:alpine

# Bind mount với user mapping
docker run -d \
  --name shoplite-dev \
  --user "$(id -u):$(id -g)" \
  -v $(pwd):/src \
  mcr.microsoft.com/dotnet/sdk:8.0-alpine
```

#### 9.3 tmpfs Mounts (In-memory, ephemeral)

```bash
# Mount tmpfs vào /tmp (in-memory, mất khi container stop)
docker run -d \
  --name app \
  --tmpfs /tmp \
  --tmpfs /run:size=128m \
  myapp

# Dùng khi: cần high-performance temp storage, không cần persist
# Ví dụ: session files, temp uploads trước khi process
```

#### 9.4 Backup và Restore Volumes

```bash
# Backup volume
docker run --rm \
  -v postgres-data:/source:ro \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/postgres-backup-$(date +%Y%m%d).tar.gz -C /source .

# Restore volume
docker run --rm \
  -v postgres-data:/target \
  -v $(pwd):/backup \
  alpine \
  sh -c "cd /target && tar xzf /backup/postgres-backup-20240115.tar.gz"
```

---

### 10. Docker Networking — Container giao tiếp với nhau

#### 10.1 Network Drivers

**bridge (default):**
- Mạng ảo private trên host.
- Containers kết nối vào bridge network.
- Containers giao tiếp qua IP addresses.
- Host giao tiếp với containers qua port mapping.

**Default bridge (docker0):**
```bash
# Container trong default bridge network
docker run -d --name c1 nginx
docker run -d --name c2 nginx

# c1 phải dùng IP của c2 (không resolve được tên)
docker inspect c1 | jq '.[0].NetworkSettings.IPAddress'  # ví dụ 172.17.0.2
docker exec c2 curl http://172.17.0.2  # chạy được
docker exec c2 curl http://c1          # không chạy được (no DNS)
```

**User-defined bridge (recommended):**
```bash
# Tạo network
docker network create shoplite-net

# Containers trong user-defined bridge có automatic DNS resolution
docker run -d --name backend --network shoplite-net shoplite-backend:v1.0.0
docker run -d --name postgres --network shoplite-net postgres:15-alpine

# backend có thể connect tới postgres bằng tên
docker exec backend ping postgres           # có thể ping
docker exec backend curl http://postgres:5432  # resolve tên đúng
```

#### 10.2 Tạo và Quản lý Networks

```bash
# Tạo network với custom subnet
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  shoplite-net

# Liệt kê networks
docker network ls

# Xem chi tiết network
docker network inspect shoplite-net

# Connect container vào network sau khi đã chạy
docker network connect shoplite-net shoplite-backend

# Disconnect container khỏi network
docker network disconnect shoplite-net shoplite-backend

# Xóa network (phải không có container nào dùng)
docker network rm shoplite-net

# Xóa networks không dùng
docker network prune
```

#### 10.3 Multi-network setup cho ShopLite

```bash
# Tạo 2 networks để phân tách frontend và backend
docker network create frontend-net   # frontend <-> nginx
docker network create backend-net    # backend <-> database

# Database: chỉ kết nối backend-net (không expose ra ngoài)
docker run -d \
  --name postgres \
  --network backend-net \
  -e POSTGRES_PASSWORD=secret \
  postgres:15-alpine

# Backend: kết nối cả 2 networks
docker run -d \
  --name shoplite-backend \
  --network backend-net \
  shoplite-backend:v1.0.0

docker network connect frontend-net shoplite-backend

# Frontend/Nginx: chỉ kết nối frontend-net
docker run -d \
  --name shoplite-frontend \
  --network frontend-net \
  -p 80:80 \
  shoplite-frontend:v1.0.0

# Kết quả:
# - postgres chỉ accessible từ backend (không expose port ra host)
# - backend accessible từ frontend qua frontend-net
# - frontend accessible từ internet qua port 80
```

#### 10.4 Host và None Networks

```bash
# Host network: container dùng network stack của host
# Linux only, không hoạt động trên Docker Desktop Mac/Windows
docker run -d \
  --network host \
  --name nginx-host \
  nginx

# None network: hoàn toàn isolated
docker run -d \
  --network none \
  --name isolated-job \
  batch-processor
```

---

### 11. Image Registry — Lưu trữ và chia sẻ images

#### 11.1 Docker Hub

```bash
# Login
docker login
# Nhập Docker Hub username và password (hoặc access token)

# Tag image theo chuẩn Docker Hub
# Format: <username>/<repository>:<tag>
docker tag shoplite-backend:v1.0.0 johndoe/shoplite-backend:v1.0.0
docker tag shoplite-backend:v1.0.0 johndoe/shoplite-backend:latest

# Push lên Docker Hub
docker push johndoe/shoplite-backend:v1.0.0
docker push johndoe/shoplite-backend:latest

# Pull về
docker pull johndoe/shoplite-backend:v1.0.0

# Logout
docker logout
```

#### 11.2 GitHub Container Registry (GHCR)

GHCR tốt hơn Docker Hub cho team vì:
- Gắn liền với GitHub repository và team permissions.
- Unlimited private packages cho GitHub free tier.
- Không có rate limit như Docker Hub.
- Tích hợp tốt với GitHub Actions.

```bash
# Login vào GHCR
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Tag image cho GHCR
# Format: ghcr.io/<github-owner>/<repository>:<tag>
docker tag shoplite-backend:v1.0.0 ghcr.io/johndoe/shoplite-backend:v1.0.0

# Push
docker push ghcr.io/johndoe/shoplite-backend:v1.0.0

# Pull (nếu public package)
docker pull ghcr.io/johndoe/shoplite-backend:v1.0.0

# Pull (nếu private, cần login trước)
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin
docker pull ghcr.io/johndoe/shoplite-backend:v1.0.0
```

#### 11.3 Tagging Strategy — Quy ước đặt tag

Quy ước tag nhất quán giúp dễ trace, rollback và deploy:

```bash
# latest: luôn trỏ vào latest main branch build
# KHÔNG dùng trong production deployment (không biết version cụ thể)
docker tag shoplite-backend:v1.0.0 ghcr.io/company/shoplite-backend:latest

# Semantic versioning: dùng cho release chính thức
# Format: vMAJOR.MINOR.PATCH
docker tag build ghcr.io/company/shoplite-backend:v1.0.0
docker tag build ghcr.io/company/shoplite-backend:v1.0    # alias cho minor
docker tag build ghcr.io/company/shoplite-backend:v1      # alias cho major

# Branch + Git SHA: dùng cho CI builds
# Format: <branch>-<short-git-sha>
GIT_SHA=$(git rev-parse --short HEAD)  # ví dụ: abc1234
docker tag build ghcr.io/company/shoplite-backend:main-${GIT_SHA}

# Pull Request preview
# Format: pr-<pr-number>
docker tag build ghcr.io/company/shoplite-backend:pr-45

# Timestamp-based (ít dùng, khó remember)
docker tag build ghcr.io/company/shoplite-backend:20240115-abc1234
```

**Quy tắc golden trong production deployment:**
- Luôn deploy bằng specific tag (`:v1.0.0` hoặc `:main-abc1234`).
- Không bao giờ deploy bằng `:latest` lên production.
- Giữ ít nhất 3 phiên bản gần nhất để rollback nhanh.

---

### 12. Debugging Containers — Xử lý sự cố thường gặp

#### 12.1 Container không start

```bash
# Bước 1: Xem logs để biết error message
docker logs shoplite-backend
docker logs shoplite-backend 2>&1  # bao gồm stderr

# Bước 2: Xem exit code và state
docker inspect shoplite-backend | jq '.[0].State'
# ExitCode: 1 thường là lỗi application
# ExitCode: 137 = SIGKILL (OOMKill hoặc docker kill)
# ExitCode: 143 = SIGTERM (docker stop)

# Bước 3: Override entrypoint để explore image
docker run -it --entrypoint /bin/sh shoplite-backend:v1.0.0
# Trong shell, chạy tay command để xem lỗi:
dotnet ShopLite.Api.dll
# -> Unhandled exception: System.IO.FileNotFoundException: ...

# Nếu image là Alpine (không có bash, chỉ có sh)
docker run -it --entrypoint /bin/sh shoplite-backend:v1.0.0
# hoặc dùng debug container share namespace
docker run -it --pid=container:shoplite-backend --network=container:shoplite-backend alpine sh
```

#### 12.2 OOMKilled — Out of Memory

```bash
# Kiểm tra container có bị OOMKill không
docker inspect shoplite-backend | jq '.[0].State.OOMKilled'
# true = bị kill vì hết memory

# Xem memory usage
docker stats --no-stream shoplite-backend

# Giải pháp:
# 1. Tăng memory limit
docker update --memory="1g" shoplite-backend

# 2. Kiểm tra GC pressure trong .NET (xem dotnet-counters)
docker exec shoplite-backend sh -c "dotnet-counters monitor --process-id 1"

# 3. Xem top memory consumers
docker exec shoplite-backend cat /proc/meminfo
docker exec shoplite-backend ps aux --sort=-vsz | head -20
```

#### 12.3 Permission Denied

```bash
# Triệu chứng:
# Error: EACCES: permission denied, open '/app/uploads/file.jpg'

# Kiểm tra user đang chạy trong container
docker exec shoplite-backend whoami
docker exec shoplite-backend id

# Kiểm tra quyền thư mục
docker exec shoplite-backend ls -la /app/uploads

# Giải pháp 1: Fix quyền trong container (tạm thời)
docker exec -u root shoplite-backend chown -R appuser:appgroup /app/uploads

# Giải pháp 2: Fix trong Dockerfile (permanent)
# Thêm vào Dockerfile:
# RUN mkdir -p /app/uploads && chown -R appuser:appgroup /app/uploads

# Giải pháp 3: Fix quyền trên host khi dùng bind mount
sudo chown -R 1001:1001 /data/uploads  # 1001 là UID của appuser trong container
```

#### 12.4 Network Connectivity Issues

```bash
# Container không reach được service khác

# Bước 1: Kiểm tra containers có cùng network không
docker network inspect shoplite-net | jq '.[0].Containers'

# Bước 2: Ping từ container này sang container kia
docker exec shoplite-backend ping postgres
docker exec shoplite-backend ping -c 3 redis

# Bước 3: Test port connectivity
docker exec shoplite-backend nc -zv postgres 5432
docker exec shoplite-backend curl -v http://redis:6379

# Bước 4: Xem DNS resolution
docker exec shoplite-backend nslookup postgres
docker exec shoplite-backend cat /etc/resolv.conf

# Bước 5: Kiểm tra service có thực sự lắng nghe port không
docker exec postgres netstat -tlnp
docker exec postgres ss -tlnp

# Giải pháp phổ biến:
# - Containers chưa join cùng network: docker network connect
# - Service chưa ready khi backend start: dùng wait-for-it.sh hoặc healthcheck
# - Sai port: kiểm tra EXPOSE và connection string
```

#### 12.5 Debugging Image Build

```bash
# Build với verbose output
docker build --progress=plain -t shoplite-backend:debug .

# Dừng build tại một stage cụ thể để debug
docker build --target builder -t debug-builder .
docker run -it --entrypoint /bin/sh debug-builder

# Xem cache hit/miss
docker build --no-cache -t shoplite-backend:nocache .

# Build với BuildKit (faster, better caching)
DOCKER_BUILDKIT=1 docker build -t shoplite-backend:v1.0.0 .

# Kiểm tra image sau build
docker run --rm shoplite-backend:v1.0.0 dotnet --info
docker run --rm shoplite-backend:v1.0.0 ls -la /app
docker run --rm shoplite-backend:v1.0.0 dotnet ShopLite.Api.dll --help
```

---

### 13. Best Practices tổng hợp

#### 13.1 Security

```dockerfile
# 1. Không chạy root
USER appuser

# 2. Scan image vulnerabilities
# docker scout cves shoplite-backend:v1.0.0
# trivy image shoplite-backend:v1.0.0

# 3. Dùng specific base image versions
FROM mcr.microsoft.com/dotnet/aspnet:8.0.8-alpine3.20  # cụ thể hơn aspnet:8.0-alpine

# 4. Không copy .env hay appsettings.Development.json vào image
# .dockerignore phải có những file này

# 5. Readonly filesystem nếu không cần ghi
docker run --read-only --tmpfs /tmp shoplite-backend:v1.0.0
```

#### 13.2 Performance

```bash
# 1. Sắp xếp layers để tận dụng cache
# Layers ít thay đổi -> Layers hay thay đổi

# 2. Dùng .dockerignore để giảm build context

# 3. Multi-stage build để giảm image size

# 4. Dùng Alpine base image khi có thể

# 5. Combine RUN commands để giảm số layers

# 6. Cleanup trong cùng RUN layer
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
```

#### 13.3 Reliability

```dockerfile
# 1. Luôn có HEALTHCHECK
HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:3000/health || exit 1

# 2. Graceful shutdown (exec form ENTRYPOINT)
ENTRYPOINT ["dotnet", "ShopLite.Api.dll"]

# 3. ASP.NET Core tự xử lý graceful shutdown khi nhận SIGTERM
# Cấu hình thời gian chờ drain requests trong Program.cs:
builder.Services.Configure<HostOptions>(options =>
{
    options.ShutdownTimeout = TimeSpan.FromSeconds(30);
});

# 4. Restart policy
docker run --restart unless-stopped app
```

---

## Kỹ năng đạt được
- Viết Dockerfile tối ưu cho ứng dụng thật.
- Build, chạy, debug container.
- Đẩy image lên registry để dùng lại.

---

## Thực hành

**Môi trường:** Docker Engine trên Ubuntu Server.

**Lab:**
- Viết Dockerfile cho backend ShopLite (multi-stage).
- Viết Dockerfile cho frontend (build static + serve).
- Chạy PostgreSQL và Redis bằng image official.
- Đẩy image backend/frontend lên registry.

**Công cụ:** Docker, Docker Hub / GHCR.

---

## Bài tập

- **Bắt buộc:** Dockerfile chạy được backend, image < kích thước hợp lý (dùng multi-stage).
- **Nâng cao:** Tối ưu để image cuối dùng base image nhẹ (alpine/distroless) và chạy bằng non-root user.
- **Mô phỏng doanh nghiệp:** Tạo tag image theo chuẩn version (`v1.0.0`, `latest`) và viết quy ước đặt tag.

---

## Deliverable
- [ ] `Dockerfile` cho backend và frontend.
- [ ] Image của 2 service đã đẩy lên registry.
- [ ] Tài liệu ngắn về quy ước đặt tên/tag image.

---

## Tiêu chí hoàn thành
- [ ] Chạy được backend + frontend dưới dạng container độc lập.
- [ ] Image được tối ưu (multi-stage, non-root).
- [ ] Image có mặt trên registry và pull về chạy được trên máy khác.

---

## Ghi chú học tập

> _Ghi lại những điều học được, vướng mắc và insight trong quá trình học phase này._
