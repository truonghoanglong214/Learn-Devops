# Tổng quan Roadmap

> **Vai trò người thiết kế:** Senior DevOps Engineer / Cloud Architect / Technical Lead & Mentor.
> **Người học:** Bắt đầu từ con số 0 → mục tiêu ứng tuyển **DevOps Intern / Fresher / Junior**.
> **Triết lý:** Học theo kiểu **đào tạo thực chiến trong doanh nghiệp**, không học rời rạc. Mỗi phase xây thêm một phần của **một sản phẩm duy nhất chạy production thật**.

---

## Bức tranh tổng thể

Toàn bộ lộ trình được chia thành **14 Phase** (Phase 0 → Phase 13). Mỗi phase là một "tầng" của một hệ thống thực tế. Khi học xong, bạn không chỉ có kiến thức mà có **một dự án production hoàn chỉnh** để đưa vào CV.

| Phase | Tên | Khối kiến thức | Kết quả thêm vào dự án |
|-------|-----|----------------|------------------------|
| 0 | Chuẩn bị & Tư duy DevOps | Mindset, môi trường làm việc | Workstation + source code ứng dụng mẫu |
| 1 | Linux Foundation | Hệ điều hành, CLI, Bash | Server Linux vận hành được + script tự động |
| 2 | Networking Foundation | TCP/IP, DNS, HTTP, SSH, Firewall | Server được bảo mật, truy cập qua SSH key |
| 3 | Git & Version Control | Git, GitHub/GitLab, branching | Repo chuẩn, có branching strategy |
| 4 | Docker | Container, image, registry | Mỗi service có Dockerfile, image đẩy lên registry |
| 5 | Docker Compose | Multi-container orchestration | `docker-compose.yml` chạy toàn hệ thống local |
| 6 | Nginx & Reverse Proxy | Reverse proxy, SSL/TLS, load balancing | Một cổng vào duy nhất, HTTPS |
| 7 | CI/CD | Tự động build/test/deploy | Pipeline tự động (GitHub Actions/GitLab CI) |
| 8 | Monitoring & Logging | Prometheus, Grafana, Loki | Stack giám sát + dashboard + cảnh báo |
| 9 | Cloud Deployment | AWS (EC2, VPC, RDS, S3) | Hệ thống chạy trên cloud thật |
| 10 | Infrastructure as Code | Terraform, Ansible | Hạ tầng được dựng bằng code |
| 11 | Kubernetes | K8s, Helm, Ingress | Hệ thống chạy trên cluster K8s |
| 12 | Production Hardening & Security | Backup, secrets, HA, scanning | Hệ thống production an toàn, có backup |
| 13 | Capstone & Portfolio | Tổng hợp, tài liệu hóa, CV | Dự án hoàn chỉnh + CV + chuẩn bị phỏng vấn |

**Thời lượng tham khảo:** 5–7 tháng nếu học bán thời gian (10–15 giờ/tuần). Có thể rút xuống ~3.5 tháng nếu học toàn thời gian.

**Sản phẩm xuyên suốt:** Một ứng dụng web 3 tầng tên là **"ShopLite"** — một mini e-commerce (cửa hàng online thu nhỏ). Lý do chọn: nó có đủ độ phức tạp thật (người dùng, sản phẩm, đơn hàng), nhiều service, cần database + cache, dễ kể chuyện khi phỏng vấn. Chi tiết ở mục **Dự án cuối cùng**.

---

# Phase 0 — Chuẩn bị & Tư duy DevOps

## Mục tiêu
- Hiểu DevOps thực chất là gì, giải quyết vấn đề gì trong doanh nghiệp (không chỉ là "biết dùng tool").
- Hiểu vòng đời phần mềm (SDLC) và DevOps nằm ở đâu trong đó.
- Có một môi trường làm việc đủ mạnh để học suốt lộ trình.
- Nhận source code ứng dụng mẫu (ShopLite) và chạy được nó ở mức cơ bản.

## Kiến thức sẽ học
- **DevOps là gì & văn hóa DevOps** — vì sao Dev và Ops phải hợp nhất; khái niệm CALMS (Culture, Automation, Lean, Measurement, Sharing). Cần học để hiểu *tại sao* làm chứ không học vẹt.
- **Vòng đời phát triển phần mềm (SDLC)** và **DevOps lifecycle** (Plan → Code → Build → Test → Release → Deploy → Operate → Monitor) — để biết mỗi phase sau này lắp vào đâu.
- **Mô hình kiến trúc ứng dụng** (monolith vs 3-tier vs microservices) — để hiểu ShopLite được thiết kế thế nào.
- **Lựa chọn môi trường làm việc:** WSL2 (Windows), máy ảo (VirtualBox/VMware), hoặc dual-boot/Linux native — cần để có chỗ thực hành an toàn.

## Kỹ năng đạt được
- Diễn đạt được "DevOps là gì" bằng ngôn ngữ của doanh nghiệp.
- Thiết lập được môi trường lab cá nhân.
- Đọc hiểu kiến trúc của một ứng dụng web nhiều tầng.

## Thực hành
- **Môi trường:** Máy cá nhân + WSL2 hoặc VirtualBox.
- **Lab:**
  - Cài đặt môi trường ảo hóa / WSL2.
  - Tải source ShopLite về, đọc cấu trúc thư mục, chạy thử frontend + backend ở chế độ dev (chỉ để hiểu ứng dụng làm gì, chưa cần tối ưu).
- **Công cụ:** VirtualBox/WSL2, VS Code, trình duyệt.

## Bài tập
- **Bắt buộc:** Vẽ sơ đồ kiến trúc ShopLite (frontend ↔ backend ↔ database ↔ cache) bằng tay hoặc draw.io.
- **Nâng cao:** Viết một trang ghi chú so sánh monolith vs microservices và nêu ShopLite thuộc loại nào, vì sao.
- **Mô phỏng doanh nghiệp:** Viết một "định nghĩa dự án" (project charter) 1 trang: mục tiêu hệ thống, người dùng, các thành phần — giống tài liệu khởi động dự án thật.

## Deliverable
- Môi trường lab hoạt động.
- Sơ đồ kiến trúc v0 của ShopLite (file ảnh/draw.io).
- File `README.md` mô tả dự án mục tiêu.

## Tiêu chí hoàn thành
- Giải thích được DevOps cho người không chuyên trong 2 phút.
- Lab khởi động được, chạy được ứng dụng mẫu ở mức cơ bản.
- Có sơ đồ kiến trúc và project charter.

---

# Phase 1 — Linux Foundation

## Mục tiêu
- Tự tin thao tác trên Linux server không cần giao diện đồ họa.
- Quản trị được một server: user, quyền, tiến trình, dịch vụ, gói phần mềm.
- Viết được Bash script để tự động hóa công việc lặp lại.

## Kiến thức sẽ học
- **Cấu trúc hệ thống file Linux** (FHS), điều hướng, thao tác file/thư mục — nền tảng của mọi việc.
- **Quyền & sở hữu (permissions, ownership, chmod, chown, sudo)** — bảo mật và vận hành đều dựa vào đây.
- **Quản lý user & group** — vì hệ thống thật luôn nhiều người dùng/dịch vụ.
- **Quản lý tiến trình & dịch vụ (ps, top, htop, systemd, journalctl)** — để vận hành và troubleshoot.
- **Quản lý gói (apt/yum/dnf)** — cài đặt phần mềm.
- **Text processing (grep, sed, awk, pipe, redirection)** — kỹ năng "cơm bữa" khi đọc log.
- **Bash scripting** (biến, vòng lặp, điều kiện, hàm, cron job) — tự động hóa.

## Kỹ năng đạt được
- Quản trị Linux server cơ bản qua dòng lệnh.
- Đọc và lọc log bằng grep/awk.
- Viết script tự động hóa tác vụ và lập lịch bằng cron.

## Thực hành
- **Môi trường:** Ubuntu Server (LTS) trên VM hoặc WSL2.
- **Lab:**
  - Cài đặt Ubuntu Server, cấu hình hostname, timezone.
  - Cài các runtime mà ShopLite cần (ví dụ Node.js / Python, PostgreSQL) **bằng tay** để hiểu phụ thuộc — sau này Docker sẽ thay thế việc này.
  - Tạo user riêng cho ứng dụng, phân quyền thư mục.
  - Viết script backup file log và lập lịch chạy bằng cron.
- **Công cụ:** Ubuntu Server, systemd, cron, bash.

## Bài tập
- **Bắt buộc:** Viết script kiểm tra dung lượng đĩa, RAM, và in cảnh báo nếu vượt ngưỡng.
- **Nâng cao:** Script tự động tạo user mới với quyền chuẩn và thư mục home cấu hình sẵn.
- **Mô phỏng doanh nghiệp:** Viết "runbook" 1 trang hướng dẫn khởi động lại dịch vụ ShopLite chạy bằng systemd khi nó sập.

## Deliverable
- Một Ubuntu Server vận hành được, có user ứng dụng riêng.
- ShopLite chạy được thủ công trên server này (chưa container hóa).
- Thư mục `scripts/` chứa các bash script tiện ích.

## Tiêu chí hoàn thành
- Làm chủ ≥ 40 lệnh Linux thông dụng mà không cần tra cứu.
- Tự cài và chạy được ShopLite thủ công trên Linux.
- Có ít nhất 2 bash script hoạt động + 1 cron job.

---

# Phase 2 — Networking Foundation

## Mục tiêu
- Hiểu cách dữ liệu di chuyển giữa các thành phần trong hệ thống.
- Bảo mật và truy cập server từ xa một cách an toàn.
- Chẩn đoán được lỗi mạng cơ bản — kỹ năng troubleshoot cốt lõi của DevOps.

## Kiến thức sẽ học
- **Mô hình TCP/IP & OSI (mức thực dụng)**, IP, subnet, port — nền tảng hiểu mọi giao tiếp.
- **DNS** (A, CNAME, record, phân giải tên miền) — để gắn domain cho ShopLite sau này.
- **HTTP/HTTPS, status code, header, request/response** — vì ứng dụng web chạy trên đó.
- **SSH** (key-based auth, config, port forwarding) — cách chuẩn để vào server.
- **Firewall** (ufw / iptables, security group) — kiểm soát lưu lượng.
- **Công cụ chẩn đoán mạng** (ping, curl, dig, netstat/ss, traceroute, tcpdump cơ bản).
- **Khái niệm load balancing & reverse proxy** (lý thuyết, để chuẩn bị Phase 6).

## Kỹ năng đạt được
- Truy cập server an toàn bằng SSH key.
- Cấu hình firewall mở/đóng đúng port.
- Dùng curl/dig/ss để debug kết nối và dịch vụ.

## Thực hành
- **Môi trường:** Ubuntu Server (Phase 1).
- **Lab:**
  - Tắt đăng nhập password, chỉ cho phép SSH key.
  - Cấu hình firewall: chỉ mở port cần thiết (SSH, HTTP, HTTPS, port ứng dụng).
  - Dùng curl test API backend, dùng dig kiểm tra phân giải tên miền.
  - Vẽ lại sơ đồ luồng mạng của ShopLite (port nào, service nào nói chuyện với ai).
- **Công cụ:** OpenSSH, ufw, curl, dig, ss.

## Bài tập
- **Bắt buộc:** Thiết lập SSH key login và vô hiệu hóa password login.
- **Nâng cao:** Cấu hình SSH config (`~/.ssh/config`) với alias + jump host.
- **Mô phỏng doanh nghiệp:** Viết tài liệu "network security baseline" liệt kê các port được mở và lý do — giống tài liệu audit bảo mật.

## Deliverable
- Server chỉ truy cập được bằng SSH key, firewall đã cấu hình.
- Sơ đồ luồng mạng (network diagram) của ShopLite.
- File tài liệu network baseline.

## Tiêu chí hoàn thành
- SSH vào server bằng key, password login bị tắt.
- Giải thích được toàn bộ port đang mở và vì sao.
- Debug được một lỗi "không kết nối được service" bằng curl/ss.

---

# Phase 3 — Git & Version Control

## Mục tiêu
- Quản lý mã nguồn chuyên nghiệp, làm việc nhóm được.
- Áp dụng quy trình branching đúng chuẩn doanh nghiệp.
- Chuẩn bị nền tảng cho CI/CD (Phase 7).

## Kiến thức sẽ học
- **Git cơ bản** (init, add, commit, log, diff, status) — nền tảng quản lý phiên bản.
- **Branching & merging, conflict resolution** — làm việc song song không đè code nhau.
- **Remote (GitHub/GitLab), push/pull/fetch, Pull/Merge Request** — cộng tác.
- **Branching strategy** (Git Flow, GitHub Flow, Trunk-based) — chuẩn quy trình thực tế.
- **`.gitignore`, semantic commit, tag & release** — sạch sẽ và chuyên nghiệp.
- **Bảo vệ branch, code review** — văn hóa kỹ thuật doanh nghiệp.

## Kỹ năng đạt được
- Quản lý source code theo nhánh, xử lý conflict.
- Tạo và review Pull Request.
- Tổ chức repo theo chuẩn dự án thật.

## Thực hành
- **Môi trường:** GitHub (hoặc GitLab) + máy local.
- **Lab:**
  - Đưa toàn bộ source ShopLite lên Git, viết `.gitignore` chuẩn.
  - Thiết lập cấu trúc nhánh: `main`, `develop`, `feature/*`.
  - Thực hành tạo feature branch → PR → review → merge.
  - Tạo cố tình một conflict và giải quyết.
- **Công cụ:** Git, GitHub/GitLab.

## Bài tập
- **Bắt buộc:** Tổ chức repo ShopLite với README, .gitignore, cấu trúc thư mục rõ ràng.
- **Nâng cao:** Cấu hình branch protection cho `main` (bắt buộc PR + review).
- **Mô phỏng doanh nghiệp:** Viết file `CONTRIBUTING.md` mô tả branching strategy và quy ước commit của team.

## Deliverable
- Repo ShopLite hoàn chỉnh trên GitHub/GitLab.
- Branching strategy được áp dụng + branch protection.
- `README.md`, `.gitignore`, `CONTRIBUTING.md`.

## Tiêu chí hoàn thành
- Toàn bộ source nằm trên Git với lịch sử commit sạch.
- Thực hiện trọn vẹn 1 vòng feature branch → PR → merge.
- Giải thích được branching strategy đang dùng và vì sao.

---

# Phase 4 — Docker (Containerization)

## Mục tiêu
- Đóng gói từng service của ShopLite thành container chạy được ở bất kỳ đâu.
- Hiểu khác biệt giữa container và máy ảo, vì sao container thay đổi cách deploy.

## Kiến thức sẽ học
- **Khái niệm container & vì sao dùng** (giải quyết "máy tôi chạy được mà") — lý do tồn tại của Docker.
- **Image vs Container vs Registry** — mô hình cốt lõi.
- **Dockerfile** (FROM, RUN, COPY, CMD, ENTRYPOINT, layer caching) — cách build image.
- **Quản lý image & container** (build, run, exec, logs, ps, prune).
- **Volume & bind mount** — lưu dữ liệu bền vững (database không được mất khi container chết).
- **Docker network cơ bản** — container nói chuyện với nhau.
- **Multi-stage build & tối ưu image** — giảm dung lượng, tăng bảo mật.
- **Image registry** (Docker Hub / GitHub Container Registry) — lưu trữ và phân phối image.

## Kỹ năng đạt được
- Viết Dockerfile tối ưu cho ứng dụng thật.
- Build, chạy, debug container.
- Đẩy image lên registry để dùng lại.

## Thực hành
- **Môi trường:** Docker Engine trên Ubuntu Server.
- **Lab:**
  - Viết Dockerfile cho backend ShopLite (multi-stage).
  - Viết Dockerfile cho frontend (build static + serve).
  - Chạy PostgreSQL và Redis bằng image official.
  - Đẩy image backend/frontend lên registry.
- **Công cụ:** Docker, Docker Hub / GHCR.

## Bài tập
- **Bắt buộc:** Dockerfile chạy được backend, image < kích thước hợp lý (dùng multi-stage).
- **Nâng cao:** Tối ưu để image cuối dùng base image nhẹ (alpine/distroless) và chạy bằng non-root user.
- **Mô phỏng doanh nghiệp:** Tạo tag image theo chuẩn version (`v1.0.0`, `latest`) và viết quy ước đặt tag.

## Deliverable
- `Dockerfile` cho backend và frontend.
- Image của 2 service đã đẩy lên registry.
- Tài liệu ngắn về quy ước đặt tên/tag image.

## Tiêu chí hoàn thành
- Chạy được backend + frontend dưới dạng container độc lập.
- Image được tối ưu (multi-stage, non-root).
- Image có mặt trên registry và pull về chạy được trên máy khác.

---

# Phase 5 — Docker Compose (Local Orchestration)

## Mục tiêu
- Khởi động toàn bộ hệ thống ShopLite bằng **một lệnh duy nhất**.
- Quản lý phụ thuộc giữa các service (backend cần database sẵn sàng trước).

## Kiến thức sẽ học
- **Cú pháp `docker-compose.yml`** (services, networks, volumes, depends_on, healthcheck) — định nghĩa cả hệ thống bằng file.
- **Biến môi trường & file `.env`** — tách cấu hình khỏi code (12-factor app).
- **Networking giữa các service trong Compose** — gọi nhau bằng tên service.
- **Healthcheck & thứ tự khởi động** — đảm bảo backend chỉ chạy khi DB sẵn sàng.
- **Quản lý dữ liệu bền vững bằng named volume** — DB không mất dữ liệu.
- **Profiles & override files (dev vs prod)** — nhiều cấu hình cho nhiều môi trường.

## Kỹ năng đạt được
- Mô tả toàn bộ stack bằng một file khai báo.
- Quản lý biến môi trường và bí mật cơ bản.
- Vận hành multi-container như một hệ thống.

## Thực hành
- **Môi trường:** Docker + Docker Compose trên Ubuntu.
- **Lab:**
  - Viết `docker-compose.yml` gồm: frontend, backend, PostgreSQL, Redis.
  - Cấu hình volume cho PostgreSQL, biến môi trường qua `.env`.
  - Thêm healthcheck và `depends_on` để khởi động đúng thứ tự.
  - Khởi động toàn hệ thống bằng `docker compose up`.
- **Công cụ:** Docker Compose.

## Bài tập
- **Bắt buộc:** Toàn bộ ShopLite (4 service) chạy bằng một lệnh, dữ liệu DB bền vững sau khi restart.
- **Nâng cao:** Tạo `docker-compose.override.yml` cho môi trường dev (hot reload) tách khỏi prod.
- **Mô phỏng doanh nghiệp:** Viết tài liệu "cách dựng môi trường local trong 5 phút" cho thành viên mới.

## Deliverable
- `docker-compose.yml` chạy toàn bộ hệ thống.
- File `.env.example` mẫu.
- Tài liệu onboarding môi trường local.

## Tiêu chí hoàn thành
- `docker compose up` dựng được toàn bộ ShopLite, truy cập được từ trình duyệt.
- Dữ liệu DB còn nguyên sau khi `down` rồi `up` lại.
- Giải thích được luồng giao tiếp giữa các service trong Compose.

---

# Phase 6 — Nginx & Reverse Proxy

## Mục tiêu
- Đặt một "cổng vào" duy nhất cho hệ thống, ẩn các service phía sau.
- Phục vụ HTTPS, định tuyến request đến đúng service.

## Kiến thức sẽ học
- **Reverse proxy là gì & vì sao cần** — bảo mật, định tuyến, một entrypoint.
- **Cấu hình Nginx** (server block, location, upstream, proxy_pass) — định tuyến request.
- **Phục vụ static frontend + proxy API tới backend** — kiến trúc web điển hình.
- **SSL/TLS & HTTPS** (chứng chỉ, Let's Encrypt/Certbot) — bắt buộc cho production.
- **Load balancing cơ bản** (round robin, upstream nhiều backend) — chuẩn bị scale.
- **Rate limiting, gzip, caching, security header** — tối ưu & bảo mật.

## Kỹ năng đạt được
- Cấu hình Nginx làm reverse proxy cho ứng dụng thật.
- Thiết lập HTTPS với chứng chỉ.
- Áp dụng kỹ thuật bảo mật và tối ưu ở tầng proxy.

## Thực hành
- **Môi trường:** Nginx chạy như một service trong Docker Compose.
- **Lab:**
  - Thêm service Nginx vào Compose làm cổng vào duy nhất.
  - Định tuyến `/` → frontend, `/api` → backend.
  - Cấu hình HTTPS (self-signed cho local, Let's Encrypt khi lên cloud).
  - Thêm gzip, security header, rate limiting cơ bản.
- **Công cụ:** Nginx, Certbot/Let's Encrypt.

## Bài tập
- **Bắt buộc:** Truy cập toàn bộ ShopLite qua một domain/port duy nhất qua Nginx.
- **Nâng cao:** Bật HTTPS và tự động chuyển hướng HTTP → HTTPS.
- **Mô phỏng doanh nghiệp:** Cấu hình upstream với 2 instance backend để minh họa load balancing.

## Deliverable
- File cấu hình `nginx.conf` cho ShopLite.
- Service Nginx trong Compose, hệ thống truy cập qua một entrypoint.
- HTTPS hoạt động.

## Tiêu chí hoàn thành
- Người dùng truy cập frontend và API qua cùng một domain.
- HTTPS hoạt động, HTTP tự chuyển sang HTTPS.
- Giải thích được vai trò reverse proxy trong kiến trúc.

---

# Phase 7 — CI/CD

## Mục tiêu
- Tự động hóa toàn bộ quy trình từ commit code → build → test → deploy.
- Loại bỏ deploy thủ công, giảm lỗi con người.

## Kiến thức sẽ học
- **Khái niệm CI/CD & pipeline** (Continuous Integration / Delivery / Deployment) — trung tâm của DevOps.
- **GitHub Actions (hoặc GitLab CI)** (workflow, job, step, runner, trigger) — công cụ pipeline phổ biến nhất hiện nay.
- **Build & test tự động** — đảm bảo code luôn chạy được.
- **Build & push Docker image trong pipeline** — gắn với Phase 4.
- **Secrets trong CI/CD** — không để lộ thông tin nhạy cảm.
- **Tự động deploy** (qua SSH/registry) và **chiến lược deploy** (rolling, blue-green cơ bản).
- **Quản lý môi trường** (dev/staging/prod).

## Kỹ năng đạt được
- Xây dựng pipeline CI/CD từ đầu.
- Tích hợp test và build image vào pipeline.
- Tự động deploy lên server.

## Thực hành
- **Môi trường:** GitHub Actions (hoặc GitLab CI) + server đích.
- **Lab:**
  - Viết workflow CI: mỗi push chạy lint + test.
  - Viết workflow CD: build image → push lên registry → deploy lên server bằng SSH/compose.
  - Lưu secrets (SSH key, DB password) trong GitHub Secrets.
  - Tách pipeline cho nhánh `develop` (staging) và `main` (prod).
- **Công cụ:** GitHub Actions / GitLab CI, registry.

## Bài tập
- **Bắt buộc:** Push lên `main` → tự động build, push image, deploy lên server, không thao tác tay.
- **Nâng cao:** Thêm bước test bắt buộc; nếu test fail thì chặn deploy.
- **Mô phỏng doanh nghiệp:** Thiết lập approval thủ công trước khi deploy lên prod (môi trường production protection).

## Deliverable
- File pipeline `.github/workflows/*.yml` (hoặc `.gitlab-ci.yml`).
- Quy trình deploy tự động hoạt động.
- Tài liệu mô tả pipeline (sơ đồ các stage).

## Tiêu chí hoàn thành
- Một commit lên `main` tự động đưa thay đổi lên server đang chạy.
- Test fail thì pipeline dừng, không deploy.
- Secrets không bao giờ lộ trong log/code.

---

# Phase 8 — Monitoring & Logging

## Mục tiêu
- Biết hệ thống "khỏe hay ốm" theo thời gian thực.
- Thu thập, xem và truy vết log tập trung.
- Nhận cảnh báo khi có sự cố — chuyển từ bị động sang chủ động.

## Kiến thức sẽ học
- **Vì sao cần observability** (3 trụ cột: Metrics, Logs, Traces) — không đo thì không vận hành được.
- **Prometheus** (thu thập metrics, exporter, PromQL cơ bản) — chuẩn metrics phổ biến nhất.
- **Grafana** (dashboard, trực quan hóa) — nhìn thấy hệ thống.
- **Logging tập trung** (Loki + Promtail, hoặc ELK/EFK) — gom log mọi service về một nơi.
- **Alerting** (Alertmanager, ngưỡng cảnh báo, gửi thông báo) — biết sự cố trước người dùng.
- **Health check & uptime monitoring** — kiểm tra sống/chết của service.
- **Khái niệm SLI/SLO/SLA** — ngôn ngữ vận hành doanh nghiệp.

## Kỹ năng đạt được
- Dựng stack giám sát hoàn chỉnh.
- Tạo dashboard và cảnh báo có ý nghĩa.
- Truy vết sự cố qua log tập trung.

## Thực hành
- **Môi trường:** Prometheus + Grafana + Loki chạy bằng Docker Compose.
- **Lab:**
  - Thêm Prometheus + Grafana vào hệ thống, expose metrics của backend.
  - Tạo dashboard: CPU, RAM, request rate, error rate, latency.
  - Gom log các container về Loki, xem trong Grafana.
  - Cấu hình cảnh báo (vd: error rate cao → gửi thông báo).
- **Công cụ:** Prometheus, Grafana, Loki/Promtail, Alertmanager, node_exporter, cAdvisor.

## Bài tập
- **Bắt buộc:** Dashboard Grafana hiển thị sức khỏe hệ thống ShopLite.
- **Nâng cao:** Cấu hình alert khi service sập hoặc error rate vượt ngưỡng, gửi về email/Discord/Telegram.
- **Mô phỏng doanh nghiệp:** Định nghĩa SLO cho ShopLite (vd: 99% request < 300ms) và tạo dashboard theo dõi nó.

## Deliverable
- Stack monitoring + logging trong Compose.
- Dashboard Grafana (xuất file JSON).
- Cấu hình alert hoạt động.

## Tiêu chí hoàn thành
- Nhìn dashboard biết ngay hệ thống có khỏe không.
- Tự gây sự cố (tắt service) và nhận được cảnh báo.
- Truy được log của một request lỗi qua hệ thống log tập trung.

---

# Phase 9 — Cloud Deployment

## Mục tiêu
- Đưa ShopLite ra Internet thật trên hạ tầng cloud.
- Hiểu các dịch vụ cloud nền tảng và mô hình chi phí.

## Kiến thức sẽ học
- **Khái niệm Cloud & mô hình dịch vụ** (IaaS/PaaS/SaaS, regions, AZ) — nền tảng tư duy cloud.
- **AWS cơ bản** (chọn AWS vì phổ biến nhất thị trường tuyển dụng):
  - **EC2** (máy ảo cloud) — nơi chạy ứng dụng.
  - **VPC, Subnet, Security Group, Internet Gateway** — mạng trên cloud.
  - **RDS** (PostgreSQL managed) — database không cần tự quản.
  - **S3** (object storage) — lưu file tĩnh/ảnh sản phẩm/backup.
  - **IAM** (user, role, policy) — phân quyền, bảo mật.
  - **Route 53** (DNS) — gắn domain.
- **Quản lý chi phí & Free Tier** — học mà không tốn nhiều tiền.

## Kỹ năng đạt được
- Khởi tạo và cấu hình hạ tầng cloud cơ bản (thủ công qua Console).
- Triển khai ứng dụng container lên cloud.
- Cấu hình mạng, bảo mật và DNS trên cloud.

## Thực hành
- **Môi trường:** Tài khoản AWS (Free Tier).
- **Lab:**
  - Tạo VPC, subnet, security group.
  - Khởi tạo EC2, cài Docker, chạy ShopLite (Compose) trên đó.
  - Chuyển database sang RDS PostgreSQL.
  - Dùng S3 lưu ảnh sản phẩm / file backup.
  - Gắn domain qua Route 53, bật HTTPS bằng Let's Encrypt.
- **Công cụ:** AWS Console, EC2, RDS, S3, IAM, Route 53.

## Bài tập
- **Bắt buộc:** ShopLite truy cập được công khai qua domain thật trên AWS.
- **Nâng cao:** Tách database ra RDS, ứng dụng chỉ kết nối qua mạng riêng (private subnet).
- **Mô phỏng doanh nghiệp:** Viết tài liệu kiến trúc cloud + ước tính chi phí hàng tháng.

## Deliverable
- ShopLite chạy public trên AWS với domain + HTTPS.
- Sơ đồ kiến trúc cloud.
- Tài liệu cấu hình hạ tầng (thủ công).

## Tiêu chí hoàn thành
- Người khác mở được ShopLite qua Internet.
- Database chạy trên RDS, app kết nối an toàn.
- Giải thích được toàn bộ kiến trúc cloud đã dựng.

---

# Phase 10 — Infrastructure as Code (Terraform & Ansible)

## Mục tiêu
- Dựng lại toàn bộ hạ tầng cloud bằng **code** thay vì click chuột.
- Cấu hình server tự động, có thể tái lập và phiên bản hóa.

## Kiến thức sẽ học
- **Vì sao cần IaC** (tái lập, version control, không "click thủ công") — cốt lõi DevOps hiện đại.
- **Terraform** (provider, resource, variable, output, state, module) — chuẩn IaC phổ biến nhất.
- **Quản lý Terraform state** (remote state trên S3, locking) — làm việc nhóm an toàn.
- **Ansible** (playbook, inventory, role, idempotency) — cấu hình server tự động.
- **Phân biệt provisioning (Terraform) vs configuration (Ansible)** — đúng tool đúng việc.

## Kỹ năng đạt được
- Viết code dựng hạ tầng cloud lặp lại được.
- Tự động cấu hình server bằng Ansible.
- Quản lý hạ tầng như quản lý phần mềm (qua Git).

## Thực hành
- **Môi trường:** Terraform + Ansible + AWS.
- **Lab:**
  - Viết Terraform dựng lại VPC, EC2, security group, RDS từ Phase 9.
  - Lưu state trên S3 với DynamoDB lock.
  - Viết Ansible playbook cài Docker + cấu hình server tự động.
  - Phá đi rồi dựng lại toàn bộ hạ tầng chỉ bằng lệnh.
- **Công cụ:** Terraform, Ansible, AWS.

## Bài tập
- **Bắt buộc:** `terraform apply` dựng được toàn bộ hạ tầng ShopLite từ đầu.
- **Nâng cao:** Module hóa Terraform (network, compute, database tách riêng) + remote state.
- **Mô phỏng doanh nghiệp:** Tích hợp `terraform plan` vào CI để review hạ tầng trước khi apply.

## Deliverable
- Thư mục `terraform/` dựng toàn bộ hạ tầng.
- Thư mục `ansible/` cấu hình server.
- Tài liệu hướng dẫn dựng hạ tầng từ con số 0.

## Tiêu chí hoàn thành
- Xóa sạch hạ tầng rồi dựng lại hoàn chỉnh chỉ bằng code.
- Terraform state được quản lý từ xa, an toàn.
- Giải thích được khác biệt provisioning vs configuration.

---

# Phase 11 — Kubernetes

## Mục tiêu
- Chạy ShopLite trên cluster Kubernetes — kỹ năng được tuyển dụng săn đón nhất.
- Hiểu cách K8s tự phục hồi, scale và quản lý ứng dụng container ở quy mô lớn.

## Kiến thức sẽ học
- **Vì sao cần orchestration & kiến trúc K8s** (control plane, node, kubelet, etcd) — bức tranh tổng thể.
- **Đối tượng cốt lõi:** Pod, ReplicaSet, **Deployment**, Service, Namespace — đơn vị vận hành.
- **ConfigMap & Secret** — cấu hình và bí mật.
- **Ingress & Ingress Controller** — thay vai trò reverse proxy của Nginx ở quy mô cluster.
- **Volume & PersistentVolume/PVC** — lưu dữ liệu bền vững.
- **Health probe (liveness/readiness), resource limit** — vận hành ổn định.
- **Scaling (HPA) & self-healing** — tự động co giãn, tự phục hồi.
- **Helm** (chart, values, template) — đóng gói và triển khai ứng dụng K8s.
- **Managed K8s (EKS/GKE)** vs local (kind/minikube/k3s).

## Kỹ năng đạt được
- Triển khai ứng dụng đa service lên Kubernetes.
- Cấu hình networking, config, secret, storage trong K8s.
- Đóng gói ứng dụng bằng Helm chart.

## Thực hành
- **Môi trường:** Local (kind/minikube) trước, sau đó EKS trên AWS.
- **Lab:**
  - Viết manifest cho frontend, backend, Redis (Deployment + Service).
  - Dùng ConfigMap/Secret cho cấu hình & mật khẩu.
  - Cài Ingress Controller, định tuyến traffic vào cluster.
  - Cấu hình HPA, test tự scale khi tải cao.
  - Đóng gói toàn bộ thành một Helm chart.
  - Triển khai lên EKS.
- **Công cụ:** kubectl, kind/minikube, EKS, Helm, Ingress-NGINX.

## Bài tập
- **Bắt buộc:** ShopLite chạy được trên cluster K8s, truy cập qua Ingress.
- **Nâng cao:** Cấu hình HPA + minh họa self-healing (xóa pod, K8s tự tạo lại).
- **Mô phỏng doanh nghiệp:** Đóng gói ShopLite thành Helm chart có `values.yaml` cho dev/prod, deploy bằng một lệnh.

## Deliverable
- Thư mục `k8s/` chứa manifest hoặc Helm chart.
- ShopLite chạy trên EKS, truy cập qua Ingress + domain.
- Tài liệu kiến trúc K8s.

## Tiêu chí hoàn thành
- Toàn bộ hệ thống chạy trên K8s, truy cập công khai.
- Xóa một pod → hệ thống tự phục hồi, không downtime.
- Triển khai lại bằng Helm chỉ với một lệnh.

---

# Phase 12 — Production Hardening & Security

## Mục tiêu
- Biến hệ thống "chạy được" thành hệ thống "chạy production an toàn, bền vững".
- Áp dụng tư duy bảo mật, sao lưu và độ sẵn sàng cao.

## Kiến thức sẽ học
- **Quản lý secrets nâng cao** (Vault / AWS Secrets Manager / Sealed Secrets) — không để lộ bí mật.
- **Backup & Restore & Disaster Recovery** (sao lưu DB tự động, kiểm thử phục hồi) — dữ liệu là tài sản.
- **High Availability** (multi-AZ, replica) — chịu lỗi.
- **Security scanning** (image scanning với Trivy, dependency scanning, SAST cơ bản) — phát hiện lỗ hổng.
- **Least privilege & IAM hardening** — chỉ cấp quyền tối thiểu.
- **Network policy & bảo mật cluster** — cô lập service.
- **DevSecOps cơ bản** — tích hợp bảo mật vào pipeline.
- **Cost optimization** — tối ưu chi phí cloud.

## Kỹ năng đạt được
- Thiết lập backup tự động và quy trình phục hồi.
- Quét và xử lý lỗ hổng bảo mật.
- Hardening hạ tầng theo best practice production.

## Thực hành
- **Môi trường:** AWS + K8s + CI/CD (toàn bộ stack đã có).
- **Lab:**
  - Thiết lập backup tự động cho database (snapshot/cron) + test restore.
  - Tích hợp Trivy quét image trong CI, chặn image có lỗ hổng nghiêm trọng.
  - Chuyển secrets sang công cụ quản lý bí mật.
  - Cấu hình HA cho database (multi-AZ) và nhiều replica cho service.
  - Rà soát và siết quyền IAM theo least privilege.
- **Công cụ:** Trivy, AWS Secrets Manager/Vault, RDS backup, network policy.

## Bài tập
- **Bắt buộc:** Backup database tự động + chứng minh restore thành công.
- **Nâng cao:** Pipeline tự động fail khi image có lỗ hổng nghiêm trọng.
- **Mô phỏng doanh nghiệp:** Viết "Disaster Recovery Plan" — kịch bản khi database mất, các bước phục hồi và thời gian mục tiêu (RTO/RPO).

## Deliverable
- Cơ chế backup/restore hoạt động + tài liệu DR.
- Image scanning trong CI/CD.
- Secrets được quản lý an toàn, IAM theo least privilege.

## Tiêu chí hoàn thành
- Mất dữ liệu giả lập → phục hồi được từ backup.
- Pipeline chặn được image có lỗ hổng.
- Không còn secret nào hardcode trong code/repo.

---

# Phase 13 — Capstone & Portfolio

## Mục tiêu
- Hoàn thiện, tài liệu hóa và "đóng gói" toàn bộ dự án để gây ấn tượng với nhà tuyển dụng.
- Chuẩn bị CV và kỹ năng phỏng vấn.

## Kiến thức sẽ học
- **Cách tài liệu hóa dự án DevOps** (kiến trúc, README, sơ đồ, runbook) — thể hiện sự chuyên nghiệp.
- **Cách trình bày dự án trong CV & phỏng vấn** — biến công sức thành cơ hội việc làm.
- **Câu hỏi phỏng vấn DevOps thường gặp** và cách trả lời dựa trên chính dự án.
- **Tự đánh giá & lấp lỗ hổng kiến thức.**

## Kỹ năng đạt được
- Trình bày một dự án kỹ thuật mạch lạc, có chiều sâu.
- Viết tài liệu kỹ thuật chuẩn doanh nghiệp.
- Tự tin trả lời phỏng vấn dựa trên kinh nghiệm thực tế.

## Thực hành
- **Lab:**
  - Viết tài liệu kiến trúc tổng thể + sơ đồ hệ thống cuối cùng.
  - Viết README chuyên nghiệp cho repo (có badge CI, hướng dẫn chạy, kiến trúc).
  - Quay một video demo ngắn (hoặc viết case study) toàn bộ luồng: commit → CI/CD → deploy → monitoring.
  - Viết các runbook vận hành.
  - Tổng duyệt: giả lập sự cố và xử lý từ đầu đến cuối.
- **Công cụ:** draw.io, Markdown, GitHub.

## Bài tập
- **Bắt buộc:** README + tài liệu kiến trúc hoàn chỉnh trên repo.
- **Nâng cao:** Viết một blog/case study mô tả dự án và những bài học rút ra.
- **Mô phỏng doanh nghiệp:** Tự phỏng vấn với 20 câu hỏi DevOps phổ biến, ghi lại câu trả lời dựa trên dự án.

## Deliverable
- Repo ShopLite hoàn chỉnh, tài liệu đầy đủ, public trên GitHub.
- Sơ đồ kiến trúc cuối + bộ runbook.
- CV mục dự án + danh sách kỹ năng + bộ câu hỏi phỏng vấn đã chuẩn bị.

## Tiêu chí hoàn thành
- Người lạ đọc repo trong 5 phút hiểu được dự án làm gì và kiến trúc ra sao.
- Trình bày được toàn bộ dự án trong 10 phút.
- Trả lời tự tin các câu hỏi phỏng vấn dựa trên chính dự án.

---

# Dự án cuối cùng — Hệ thống "ShopLite"

Một **mini e-commerce production** được xây dựng dần qua từng phase. Đây là "linh hồn" của portfolio.

## Kiến trúc hệ thống

```
                          Internet
                             │
                      [ DNS / Route 53 ]
                             │
                   [ Ingress / Nginx + HTTPS ]
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                     │
   [ Frontend ]         [ Backend API ]        (static assets)
   (React/Vue SPA)     (Node.js/FastAPI)            │
                             │                       │
                ┌────────────┼────────────┐     [ S3 Bucket ]
                │                          │
         [ PostgreSQL / RDS ]        [ Redis cache ]
                │
         [ Backup tự động ]

   Bao quanh toàn hệ thống:
   - CI/CD pipeline (GitHub Actions/GitLab CI)
   - Monitoring: Prometheus + Grafana
   - Logging tập trung: Loki
   - Hạ tầng dựng bằng Terraform + Ansible
   - Chạy trên Kubernetes (EKS)
```

## Các thành phần

- **Frontend:** Ứng dụng SPA (React hoặc Vue) — giao diện cửa hàng: xem sản phẩm, giỏ hàng, đặt hàng, đăng nhập. Build thành static, phục vụ qua Nginx/Ingress.
- **Backend:** REST API (Node.js Express hoặc Python FastAPI) — xử lý người dùng, sản phẩm, đơn hàng, xác thực (JWT). Stateless để dễ scale.
- **Database:** PostgreSQL (local qua Docker → RDS trên cloud) — lưu user, sản phẩm, đơn hàng.
- **Cache:** Redis — cache truy vấn và lưu session, tăng tốc và minh họa kiến trúc nhiều tầng.
- **Reverse Proxy:** Nginx (Compose) → Ingress Controller (K8s) — một cổng vào, HTTPS, định tuyến, rate limiting.
- **Containerization:** Mỗi service một Dockerfile tối ưu, image lưu trên registry.
- **Orchestration:** Docker Compose (local) → Kubernetes/Helm (cloud).
- **CI/CD:** Pipeline tự động build → test → scan → push image → deploy, có môi trường staging và prod.
- **Monitoring:** Prometheus thu metrics, Grafana dashboard, Alertmanager cảnh báo, định nghĩa SLO.
- **Logging:** Loki + Promtail gom log toàn hệ thống, xem trong Grafana.
- **Cloud:** AWS (EC2/EKS, VPC, RDS, S3, IAM, Route 53).
- **Kubernetes:** Deployment, Service, Ingress, ConfigMap/Secret, HPA, PVC, đóng gói bằng Helm.
- **Backup:** Backup tự động database + test restore + Disaster Recovery Plan.
- **Security:** HTTPS, secrets management, image scanning (Trivy), IAM least privilege, network policy, non-root container.

## Điểm mạnh khi kể chuyện phỏng vấn
Dự án chứng minh bạn đã đi qua **toàn bộ vòng đời DevOps**: từ một dòng code đến hệ thống chạy thật trên cloud, tự động deploy, tự giám sát, tự phục hồi, có backup và bảo mật. Đây chính xác là điều nhà tuyển dụng muốn thấy ở Junior DevOps.

---

# Giá trị CV

## Có thể ghi gì vào CV

**Mục kỹ năng (Skills):**
- Linux (Ubuntu Server) administration, Bash scripting
- Networking: TCP/IP, DNS, HTTP/HTTPS, SSH, Firewall
- Git, GitHub/GitLab, branching strategy
- Docker, Docker Compose, container optimization
- Nginx (reverse proxy, load balancing, SSL/TLS)
- CI/CD: GitHub Actions / GitLab CI
- Monitoring & Logging: Prometheus, Grafana, Loki
- Cloud: AWS (EC2, VPC, RDS, S3, IAM, Route 53)
- Infrastructure as Code: Terraform, Ansible
- Kubernetes, Helm
- Security: secrets management, Trivy scanning, backup/DR

**Mục dự án (Project) — ví dụ mô tả:**
> *Thiết kế và triển khai hệ thống e-commerce 3 tầng (ShopLite) chạy production trên AWS EKS. Container hóa toàn bộ service bằng Docker, dựng hạ tầng bằng Terraform, tự động hóa deploy bằng CI/CD pipeline (GitHub Actions), giám sát bằng Prometheus/Grafana và logging tập trung với Loki. Triển khai backup tự động, image scanning và quản lý secrets theo chuẩn production.*

## Kỹ năng nhà tuyển dụng đánh giá cao nhất
1. **Kubernetes + Helm** — nhu cầu cao, khó tự học, tạo khác biệt rõ rệt.
2. **CI/CD pipeline thực tế** — chứng minh tư duy tự động hóa.
3. **Terraform (IaC)** — kỹ năng "must-have" của DevOps hiện đại.
4. **Cloud (AWS)** — gần như bắt buộc trên thị trường.
5. **Monitoring/Logging** — cho thấy tư duy vận hành, không chỉ deploy.
6. **Tư duy bảo mật & backup** — thứ phân biệt người "biết tool" với người "vận hành được".

## Mức độ tương đương

| Cấp độ | Sau khi hoàn thành |
|--------|---------------------|
| **Intern** | Đạt thừa — bạn vượt yêu cầu intern thông thường. |
| **Fresher** | Đạt tốt — có dự án end-to-end là điểm cộng lớn. |
| **Junior** | Đạt — đặc biệt mạnh nếu nắm chắc K8s, CI/CD, Terraform và giải thích được dự án. |

## Còn thiếu gì để lên Mid-level DevOps
- **Quy mô & kinh nghiệm production thật** (xử lý sự cố thật, tải lớn, on-call).
- **Microservices & service mesh** (Istio/Linkerd), message queue (Kafka/RabbitMQ).
- **GitOps** (ArgoCD/Flux) và chiến lược deploy nâng cao (canary, progressive delivery).
- **Đa cloud / nâng cao về networking, cost & performance tuning ở quy mô lớn.**
- **Chứng chỉ** (AWS Certified, CKA — Certified Kubernetes Administrator) để củng cố hồ sơ.
- **Kinh nghiệm vận hành trong môi trường team thật** (làm việc với SRE, on-call rotation, postmortem).

---

# Timeline dự kiến

Giả định **học bán thời gian, ~10–15 giờ/tuần**. (Học toàn thời gian rút khoảng 40–50%.)

| Phase | Nội dung | Thời lượng |
|-------|----------|------------|
| 0 | Chuẩn bị & Tư duy | 3–5 ngày |
| 1 | Linux Foundation | 2 tuần |
| 2 | Networking | 1–1.5 tuần |
| 3 | Git | 1 tuần |
| 4 | Docker | 2 tuần |
| 5 | Docker Compose | 1 tuần |
| 6 | Nginx & Reverse Proxy | 1 tuần |
| 7 | CI/CD | 2 tuần |
| 8 | Monitoring & Logging | 2 tuần |
| 9 | Cloud (AWS) | 2–3 tuần |
| 10 | Infrastructure as Code | 2–3 tuần |
| 11 | Kubernetes | 3–4 tuần |
| 12 | Production Hardening & Security | 2 tuần |
| 13 | Capstone & Portfolio | 1–2 tuần |
| **Tổng** | | **~5–7 tháng** |

**Mốc kiểm tra quan trọng:**
- Hết Phase 5: bạn đã có hệ thống chạy local hoàn chỉnh (có thể demo).
- Hết Phase 9: hệ thống đã online thật trên cloud (có link để khoe).
- Hết Phase 11: đã có kỹ năng "ăn tiền" nhất (K8s) → bắt đầu rải CV được.
- Hết Phase 13: portfolio hoàn chỉnh, sẵn sàng phỏng vấn nghiêm túc.

---

# Kế hoạch học tập theo tuần

Mẫu lịch ~12 giờ/tuần. Điều chỉnh theo tốc độ thực tế.

| Tuần | Trọng tâm | Mục tiêu cuối tuần |
|------|-----------|---------------------|
| 1 | Phase 0 + bắt đầu Linux | Lab sẵn sàng, sơ đồ kiến trúc, làm chủ lệnh Linux cơ bản |
| 2 | Phase 1 (Linux nâng cao + Bash) | ShopLite chạy thủ công trên Linux, có bash script + cron |
| 3 | Phase 2 (Networking) | SSH key, firewall, sơ đồ mạng |
| 4 | Phase 3 (Git) | Repo chuẩn, branching strategy, 1 vòng PR |
| 5 | Phase 4 (Docker) — phần 1 | Dockerfile backend chạy được |
| 6 | Phase 4 (Docker) — phần 2 | Dockerfile frontend, image lên registry |
| 7 | Phase 5 (Compose) | Toàn hệ thống chạy bằng `docker compose up` |
| 8 | Phase 6 (Nginx) | Reverse proxy + HTTPS, một entrypoint |
| 9 | Phase 7 (CI/CD) — phần 1 | Pipeline CI (build + test) |
| 10 | Phase 7 (CI/CD) — phần 2 | Pipeline CD tự động deploy |
| 11 | Phase 8 (Monitoring) — phần 1 | Prometheus + Grafana + dashboard |
| 12 | Phase 8 (Monitoring) — phần 2 | Logging tập trung + alerting |
| 13–14 | Phase 9 (Cloud) | ShopLite online thật trên AWS + domain |
| 15–16 | Phase 10 (IaC) | Hạ tầng dựng lại hoàn toàn bằng Terraform + Ansible |
| 17–18 | Phase 11 (K8s) — phần 1 | ShopLite chạy trên K8s local (kind/minikube) |
| 19–20 | Phase 11 (K8s) — phần 2 | Helm chart + deploy lên EKS + Ingress + HPA |
| 21–22 | Phase 12 (Hardening) | Backup/DR, image scanning, secrets, IAM |
| 23–24 | Phase 13 (Capstone) | Tài liệu, README, CV, luyện phỏng vấn |

**Nguyên tắc học xuyên suốt:**
- Mỗi phase đều phải tạo ra **artifact đưa lên Git** (file cấu hình, script, manifest) — không học chay.
- Cuối mỗi phase: tự kiểm tra bằng **Tiêu chí hoàn thành** trước khi sang phase sau.
- Ưu tiên **làm tay hiểu bản chất trước, tự động hóa sau** (đó là lý do thứ tự: thủ công → Compose → IaC → K8s).
- Ghi **nhật ký học tập** (mỗi phase học được gì, vướng gì) — sau này thành chất liệu trả lời phỏng vấn.

---

*Roadmap này được thiết kế ở mức tổng quan để bạn thấy bức tranh toàn cảnh. Chưa đi vào dạy chi tiết, code hay cài đặt — đúng như yêu cầu. Khi bạn sẵn sàng bắt đầu, ta sẽ đi sâu vào từng phase theo thứ tự.*