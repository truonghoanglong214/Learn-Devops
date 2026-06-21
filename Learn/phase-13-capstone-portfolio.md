# Phase 13 — Capstone & Portfolio

## Mục tiêu
- Hoàn thiện, tài liệu hóa và "đóng gói" toàn bộ dự án để gây ấn tượng với nhà tuyển dụng.
- Chuẩn bị CV và kỹ năng phỏng vấn.

---

## Kiến thức sẽ học

- **Cách tài liệu hóa dự án DevOps** (kiến trúc, README, sơ đồ, runbook).
- **Cách trình bày dự án trong CV & phỏng vấn** — biến công sức thành cơ hội việc làm.
- **Câu hỏi phỏng vấn DevOps thường gặp** và cách trả lời dựa trên chính dự án.
- **Tự đánh giá & lấp lỗ hổng kiến thức.**

---

## Kiến thức chi tiết

### 1. Checklist hoàn thiện project trước khi cho vào CV

Đây là bộ tiêu chí tối thiểu để một project DevOps được coi là "production-ready" và xứng đáng xuất hiện trên CV. Đừng thêm project vào CV nếu chưa qua được checklist này — nhà tuyển dụng senior sẽ nhìn vào repo và biết ngay bạn có thực sự làm hay không.

#### Code Quality

- [ ] Không có hardcoded secrets trong code hoặc git history
- [ ] `.gitignore` đầy đủ, không có `.env`, `node_modules`, `*.pem` trong repo
- [ ] `README.md` rõ ràng, người lạ đọc 5 phút hiểu ngay
- [ ] Semantic commit messages (`feat:`, `fix:`, `docs:`, `ci:`)
- [ ] Code có lint rules và formatter (ESLint + Prettier cho JS, Black cho Python)
- [ ] Không có commented-out code thừa trong production branch
- [ ] Dependency versions được pin (package-lock.json, requirements.txt với phiên bản cụ thể)
- [ ] Không có TODO/FIXME comment tồn đọng trong code critical path

**Giải thích chi tiết:**

Hardcoded secrets là lỗi phổ biến nhất và nguy hiểm nhất. Ngay cả khi bạn đã xóa secrets khỏi code, nếu nó từng xuất hiện trong một commit nào đó, nó vẫn còn trong git history. Công cụ kiểm tra: `git log -p | grep -i "password\|secret\|key\|token"`. Nếu phát hiện, cần dùng `git filter-branch` hoặc `BFG Repo-Cleaner` để xóa sạch history.

`.gitignore` cần được setup ngay từ commit đầu tiên, không phải sau khi đã lỡ commit file nhạy cảm. Template cơ bản cho một project full-stack:

```
# Environment
.env
.env.local
.env.*.local

# Dependencies
node_modules/
vendor/
__pycache__/

# Build outputs
dist/
build/
*.pyc

# Secrets & Keys
*.pem
*.key
*.p12
*.pfx
id_rsa
id_ed25519

# IDE
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# Docker
.docker/
```

Semantic commit messages giúp tạo CHANGELOG tự động và dễ đọc git log. Format: `type(scope): description`. Ví dụ:
- `feat(auth): add JWT refresh token endpoint`
- `fix(db): resolve connection pool exhaustion under load`
- `ci(github-actions): add security scan job`
- `docs(readme): update quick start guide`
- `chore(deps): bump express from 4.18.1 to 4.18.2`

#### CI/CD

- [ ] CI chạy tự động khi push (lint + test)
- [ ] CD tự động deploy sau khi CI pass
- [ ] Pipeline fail khi test fail
- [ ] Không có secrets trong pipeline logs
- [ ] CI chạy trong vòng dưới 5 phút (nếu lâu hơn cần tối ưu)
- [ ] Build artifacts được cache (npm cache, Docker layer cache)
- [ ] Có branch protection rules: không merge trực tiếp vào main khi CI chưa pass
- [ ] Có manual approval gate trước khi deploy lên production

**Giải thích chi tiết:**

Pipeline chậm là dấu hiệu team không quan tâm đến developer experience. Nếu CI mất 20 phút, developer sẽ bắt đầu push ít hơn và batch nhiều thay đổi vào 1 commit — ngược lại với best practice. Cách tối ưu pipeline:

1. Cache node_modules giữa các run: `actions/cache` với key là hash của `package-lock.json`
2. Docker BuildKit với layer cache: `--cache-from` và `--cache-to`
3. Chạy test song song: Jest `--maxWorkers=4`
4. Chỉ build service bị thay đổi (monorepo): path filters

Về secrets trong logs: ngay cả khi bạn dùng GitHub Secrets, nếu lỡ `echo $SECRET_VAR` trong pipeline, nó sẽ bị mask nhưng có thể vẫn visible tùy cách log. Rule: không bao giờ print secrets, dù trong debug mode.

#### Infrastructure

- [ ] Toàn bộ infrastructure được code hóa (Terraform)
- [ ] Có thể destroy và rebuild từ 0 với `terraform apply` + `ansible-playbook`
- [ ] Monitoring dashboard hiển thị metrics
- [ ] Alerting được cấu hình
- [ ] Terraform state lưu remote (S3 + DynamoDB locking)
- [ ] Không có manual changes trên server (drift detection)
- [ ] Resource tagging đầy đủ (Project, Environment, Owner, CostCenter)
- [ ] Backup được cấu hình và test restore thực tế

**Giải thích chi tiết:**

"Có thể rebuild từ 0" là tiêu chí quan trọng nhất của IaC. Nếu bạn cần SSH vào server và chạy một vài lệnh thủ công mà không được document, infrastructure của bạn chưa thực sự được code hóa. Test bằng cách: destroy hoàn toàn, rồi rebuild chỉ bằng `terraform apply` và `ansible-playbook site.yml`. Nếu app chạy được — bạn pass.

Tagging resources trên AWS quan trọng vì: cost allocation (biết service nào tốn bao nhiêu tiền), compliance auditing, và tìm kiếm resources khi account có hàng trăm resources.

#### Security

- [ ] HTTPS hoạt động, HTTP redirect về HTTPS
- [ ] Container chạy non-root user
- [ ] No CRITICAL vulnerabilities (Trivy scan pass)
- [ ] Secrets trong Vault/AWS Secrets Manager, không trong code
- [ ] Security groups chỉ mở port cần thiết (least privilege)
- [ ] Database không expose ra internet (chỉ accessible từ VPC nội bộ)
- [ ] Dependency audit pass (`npm audit`, `pip-audit`)
- [ ] CORS được cấu hình chặt chẽ (không phải `*`)

**Giải thích chi tiết:**

Container chạy non-root là yêu cầu tối thiểu. Trong Dockerfile:

```dockerfile
# Tạo user riêng
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# Chuyển ownership
COPY --chown=nextjs:nodejs .next/standalone ./
COPY --chown=nextjs:nodejs .next/static ./.next/static

# Switch user
USER nextjs
```

Trivy scan tích hợp vào CI:

```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'ghcr.io/${{ github.repository }}/backend:${{ github.sha }}'
    format: 'sarif'
    exit-code: '1'
    severity: 'CRITICAL'
```

Pipeline sẽ fail nếu có CRITICAL vulnerability — đây là hard gate.

#### Documentation

- [ ] Architecture diagram có trong README
- [ ] Runbooks có trong `docs/`
- [ ] `.env.example` đầy đủ với comment giải thích từng biến
- [ ] API documentation (Swagger/OpenAPI hoặc Postman collection)
- [ ] Troubleshooting guide cho các lỗi thường gặp
- [ ] Changelog được duy trì
- [ ] Contributing guide nếu repo public

---

### 2. Repository Structure chuẩn production

Cấu trúc thư mục nói lên rất nhiều về kinh nghiệm của người làm. Dưới đây là cấu trúc chuẩn cho project ShopLite sau khi hoàn thành tất cả các phase:

```
shoplite/
├── README.md                          # Entry point, phải đọc được trong 5 phút
├── CHANGELOG.md                       # Lịch sử thay đổi theo semantic versioning
├── LICENSE                            # MIT hoặc Apache 2.0 cho project cá nhân
├── docker-compose.yml                 # Config cơ bản, dùng cho development
├── docker-compose.override.yml        # Override local (không commit, có trong .gitignore)
├── docker-compose.prod.yml            # Production overrides (replicas, resource limits)
├── .env.example                       # Template biến môi trường, có comment đầy đủ
├── .gitignore                         # Đầy đủ, không thiếu gì
├── .pre-commit-config.yaml            # Pre-commit hooks: lint, secret scan
│
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                     # Lint + Test, chạy khi push
│   │   ├── cd.yml                     # Build + Deploy, chạy khi merge vào main
│   │   └── security-scan.yml          # Trivy + gitleaks, chạy hàng tuần
│   ├── pull_request_template.md       # Template cho PR description
│   └── CODEOWNERS                     # Ai review PR của phần nào
│
├── frontend/
│   ├── Dockerfile                     # Multi-stage: build (node) + serve (nginx)
│   ├── nginx.conf                     # Nginx config cho SPA (try_files $uri /index.html)
│   ├── package.json
│   ├── package-lock.json
│   └── src/
│       ├── components/
│       ├── pages/
│       └── ...
│
├── backend/
│   ├── Dockerfile                     # Multi-stage: build + runtime image nhỏ
│   ├── package.json
│   ├── package-lock.json
│   ├── src/
│   │   ├── index.js
│   │   ├── routes/
│   │   ├── controllers/
│   │   ├── models/
│   │   └── middleware/
│   └── tests/
│       ├── unit/
│       └── integration/
│
├── nginx/
│   └── nginx.conf                     # Reverse proxy config, rate limiting, HTTPS
│
├── k8s/
│   ├── namespace.yaml                 # namespace: shoplite
│   └── shoplite-chart/                # Helm chart
│       ├── Chart.yaml
│       ├── values.yaml                # Default values
│       ├── values-prod.yaml           # Production overrides
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── ingress.yaml
│           ├── hpa.yaml
│           ├── configmap.yaml
│           └── _helpers.tpl
│
├── terraform/
│   ├── main.tf                        # Root module, provider config
│   ├── variables.tf                   # Input variables với description và defaults
│   ├── outputs.tf                     # Outputs: IP, DNS, ARNs
│   ├── versions.tf                    # Provider version constraints
│   ├── terraform.tfvars.example       # Template cho terraform.tfvars
│   └── modules/
│       ├── vpc/                       # VPC, subnets, route tables, IGW
│       ├── ec2/                       # EC2 instances, security groups, key pairs
│       ├── rds/                       # RDS PostgreSQL, parameter groups
│       ├── s3/                        # S3 buckets cho static assets và backups
│       └── iam/                       # IAM roles, policies cho EC2
│
├── ansible/
│   ├── inventory.ini                  # Server inventory (EC2 IPs từ Terraform output)
│   ├── inventory.yml                  # Dynamic inventory (nếu dùng)
│   ├── site.yml                       # Master playbook, import tất cả roles
│   ├── ansible.cfg                    # Ansible config (host_key_checking, etc.)
│   └── roles/
│       ├── common/                    # Cài packages cơ bản, timezone, SSH hardening
│       ├── docker/                    # Cài Docker CE, Docker Compose
│       ├── app/                       # Deploy application, env vars, start services
│       ├── nginx/                     # Cài Nginx, cấu hình HTTPS với Certbot
│       └── monitoring/                # Cài node_exporter, Promtail
│
├── monitoring/
│   ├── prometheus.yml                 # Scrape configs: node_exporter, cadvisor, app
│   ├── alerting_rules.yml             # Alert rules: high CPU, disk full, service down
│   ├── docker-compose.monitoring.yml  # Prometheus + Grafana + Loki + Promtail
│   └── grafana-dashboards/
│       ├── overview.json              # System overview: CPU, RAM, Network
│       ├── application.json           # App metrics: RPS, latency, error rate
│       └── database.json              # PostgreSQL metrics: connections, slow queries
│
├── scripts/
│   ├── backup-postgres.sh             # Dump DB, upload S3, cleanup old backups
│   ├── check-health.sh                # Quick health check cho tất cả services
│   └── rotate-secrets.sh             # Rotate DB password, update Secrets Manager
│
└── docs/
    ├── architecture.md                # Kiến trúc tổng thể, quyết định thiết kế
    ├── runbooks/
    │   ├── deploy.md                  # Hướng dẫn deploy step-by-step
    │   ├── rollback.md                # Rollback procedure
    │   ├── incident-response.md       # Quy trình xử lý sự cố
    │   └── database-restore.md        # Restore DB từ S3 backup
    └── dr-plan.md                     # Disaster Recovery plan
```

**Tại sao cấu trúc này quan trọng?**

Khi nhà tuyển dụng mở repo, điều đầu tiên họ nhìn vào là cấu trúc thư mục. Một cấu trúc rõ ràng, có tổ chức cho thấy bạn đã nghĩ về maintainability và team collaboration. Ngược lại, mọi thứ đặt ngang hàng trong root folder cho thấy thiếu kinh nghiệm.

**`docker-compose.override.yml`** không được commit vào repo. File này chứa cấu hình local của từng developer (port khác nhau, volume mounts, debug flags). Docker Compose tự động merge file này với `docker-compose.yml` khi chạy.

**`values-prod.yaml`** vs **`values.yaml`**: Không bao giờ hardcode production values vào chart. Mọi giá trị production-specific (số replicas, resource limits, image tags) đều override trong file riêng.

---

### 3. README.md Template chuyên nghiệp

README là "bộ mặt" của project. Viết như viết landing page — người đọc quyết định trong 30 giây có tiếp tục đọc không. Dưới đây là template hoàn chỉnh:

```markdown
# ShopLite — Production-Grade E-commerce System

[![CI](https://github.com/user/shoplite/actions/workflows/ci.yml/badge.svg)](https://github.com/user/shoplite/actions/workflows/ci.yml)
[![Docker](https://img.shields.io/docker/v/user/shoplite-backend?label=backend)](https://hub.docker.com/r/user/shoplite-backend)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Uptime](https://img.shields.io/uptimerobot/ratio/mXXXXXXXXX?label=uptime)](https://status.shoplite.example.com)

> Mini e-commerce system xây dựng với đầy đủ DevOps practices: Docker, K8s, CI/CD,
> Monitoring, và Infrastructure as Code. Mục tiêu: minh họa production workflow
> thực tế trong môi trường học tập.

## Kiến trúc hệ thống

```
┌─────────────────────────────────────────────────────────────┐
│                         Internet                            │
└─────────────────────────┬───────────────────────────────────┘
                          │ HTTPS :443
                    ┌─────▼──────┐
                    │   Nginx    │  ← Reverse Proxy, SSL Termination
                    │ (Ingress)  │    Rate Limiting
                    └─────┬──────┘
              ┌───────────┴───────────┐
              │                       │
        ┌─────▼──────┐         ┌──────▼─────┐
        │  Frontend   │         │  Backend   │
        │  React/Nginx│         │  Node.js   │
        │  :3000      │         │  :4000     │
        └─────────────┘         └──────┬─────┘
                                       │
                          ┌────────────┼────────────┐
                          │            │            │
                   ┌──────▼─────┐ ┌────▼────┐ ┌────▼────┐
                   │ PostgreSQL │ │  Redis  │ │   S3    │
                   │    :5432   │ │  :6379  │ │ (Files) │
                   └────────────┘ └─────────┘ └─────────┘

Monitoring Stack (separate):
Prometheus :9090 → Grafana :3001
Loki :3100 ← Promtail (log shipper)
```

## Tech Stack

| Layer | Technology | Lý do chọn |
|-------|------------|------------|
| Frontend | React 18, Nginx | SPA với static file serving |
| Backend | Node.js 20, Express | Quen thuộc, ecosystem lớn |
| Database | PostgreSQL 15 | ACID compliance, JSON support |
| Cache | Redis 7 | Session storage, rate limiting |
| Proxy | Nginx | SSL termination, static files |
| Container | Docker, Docker Compose | Reproducible environments |
| Orchestration | Kubernetes (EKS) | Auto-scaling, self-healing |
| CI/CD | GitHub Actions | Free, tích hợp tốt với GitHub |
| IaC | Terraform + Ansible | Provision + Configure separation |
| Monitoring | Prometheus, Grafana, Loki | Open source, industry standard |
| Cloud | AWS (EC2, RDS, S3, Route 53) | Phổ biến nhất, free tier |

## Quick Start (5 phút)

**Yêu cầu:** Docker Desktop 4.x+, Git

```bash
# Clone repo
git clone https://github.com/user/shoplite.git
cd shoplite

# Setup environment
cp .env.example .env
# Mở .env và điền các giá trị cần thiết (xem comment trong file)

# Khởi động tất cả services
docker compose up -d

# Kiểm tra status
docker compose ps

# Mở trình duyệt
# App:      http://localhost
# Grafana:  http://localhost:3001  (admin/admin)
# Prometheus: http://localhost:9090
```

## Deploy lên Production

Xem chi tiết tại [docs/runbooks/deploy.md](docs/runbooks/deploy.md)

Tóm tắt:
1. Merge PR vào main → CI chạy tự động
2. CI pass → Manual approval trong GitHub Actions
3. Approve → CD deploy lên EC2 qua SSH

## Monitoring

- Grafana dashboard: http://your-server:3001
- Prometheus metrics: http://your-server:9090
- Uptime status page: https://status.shoplite.example.com

## Rollback

```bash
# Kubernetes
kubectl rollout undo deployment/backend -n shoplite

# Docker Compose (về image trước)
IMAGE_TAG=abc1234 docker compose -f docker-compose.prod.yml up -d
```

## Cấu trúc repo

Xem [Repository Structure](#repository-structure) bên dưới.

## Tài liệu

- [Kiến trúc chi tiết](docs/architecture.md)
- [Deploy runbook](docs/runbooks/deploy.md)
- [Rollback procedure](docs/runbooks/rollback.md)
- [Incident response](docs/runbooks/incident-response.md)
- [Disaster recovery plan](docs/dr-plan.md)
```

**Lưu ý khi viết README:**

Badges tạo ấn tượng chuyên nghiệp ngay lập tức. CI badge đặc biệt quan trọng vì nó hiển thị real-time status. Nếu badge màu đỏ (failing), nhà tuyển dụng sẽ không impressed.

ASCII architecture diagram trong README cho thấy bạn hiểu toàn bộ hệ thống, không chỉ từng phần riêng lẻ. Không cần đẹp, cần rõ.

"Quick Start" phải thực sự chạy được trong 5 phút. Test bằng cách: clone repo vào máy mới (hoặc VM sạch), chạy theo đúng hướng dẫn. Nếu gặp lỗi, fix README cho đến khi không còn lỗi nữa.

---

### 4. Câu hỏi phỏng vấn DevOps + Câu trả lời dựa trên ShopLite

Nguyên tắc trả lời phỏng vấn kỹ thuật: luôn kết nối câu trả lời lý thuyết với kinh nghiệm thực tế từ dự án. Đừng chỉ nói "Blue-Green là..." — hãy nói "Trong ShopLite, chúng tôi dùng rolling update vì... nhưng nếu traffic tăng, tôi sẽ chuyển sang canary vì..."

---

**Q1: "Mô tả CI/CD pipeline của bạn"**

A: Pipeline tôi xây dựng cho ShopLite dùng GitHub Actions với 2 workflow riêng biệt: CI và CD.

CI workflow trigger khi có push lên bất kỳ branch nào hoặc khi mở PR. Jobs: checkout code, setup Node.js, install dependencies (với npm ci để đảm bảo reproducible install), chạy ESLint để lint, chạy Jest cho unit tests và integration tests. Integration tests kết nối với PostgreSQL và Redis service containers được spin up trong pipeline — không mock, test thật. Job này cần chạy dưới 5 phút.

CD workflow trigger khi CI pass trên branch main, và cần manual approval từ team lead. Steps: build Docker image với multi-stage build, tag với git SHA ngắn (7 ký tự), push lên GitHub Container Registry, SSH vào EC2 production server, pull image mới, chạy `docker compose up -d --no-deps backend`. Zero-downtime vì Docker Compose health check đảm bảo container mới healthy trước khi traffic chuyển sang.

Toàn bộ secrets (SSH key, GHCR token, DB password) lưu trong GitHub Secrets encrypted. Không bao giờ xuất hiện trong logs vì GitHub tự mask. Tôi xác nhận bằng cách check Actions logs sau mỗi deploy.

---

**Q2: "Bạn xử lý database migration như thế nào trong CI/CD?"**

A: Migration là phần phức tạp nhất của CD pipeline vì nó có thể gây downtime nếu làm sai.

Trong ShopLite, tôi dùng Flyway cho migration. Migration chạy trong một step riêng biệt trước khi restart application container. Mỗi migration file được đặt tên theo convention `V{version}__{description}.sql` và idempotent — chạy lại không gây lỗi vì Flyway track version đã apply.

Nguyên tắc backward-compatible migrations: không bao giờ drop column hay rename column trong cùng một deploy. Thay vào đó: deploy 1 thêm column mới + code support cả 2, deploy 2 migrate data sang column mới, deploy 3 drop column cũ. Cách này đảm bảo rollback luôn safe.

Trên Kubernetes, tôi dùng init container pattern: init container chạy migration với image riêng, chỉ khi exit 0 thì app container mới start. Nếu migration fail, pod ở trạng thái Init:Error và app không receive traffic.

---

**Q3: "Service bị chậm lúc 3 giờ sáng, bạn làm gì?"**

A: Đầu tiên không panic. Check Grafana dashboard ngay — nhìn vào Overview dashboard với 3 signal chính: request rate, error rate, p99 latency (RED method).

Nếu error rate tăng đột biến: check Loki logs với query `{service="backend"} |= "ERROR" | json`. Xem error message, stack trace. Nếu là DB connection error: check `pg_connections_active` metric. Nếu pool exhausted: tạm thời tăng `max_connections` trong configmap và restart.

Nếu latency cao nhưng error rate thấp: có thể slow query. Check Grafana dashboard "Database" panel: `pg_stat_statements` histogram. Nếu thấy query nào > 1s: EXPLAIN ANALYZE trong psql để xem query plan, thêm index nếu cần.

Nếu cần rollback ngay để giảm blast radius: `kubectl rollout undo deployment/backend` — K8s sẽ rollout về revision trước. Hoặc Docker Compose: `IMAGE_TAG=previous_sha docker compose up -d backend`.

Sau incident: viết post-mortem trong vòng 24 giờ, theo format: Timeline, Root Cause, Impact, Resolution, Action Items (blameless). Thêm alert rule mới để detect sớm hơn lần sau.

---

**Q4: "Kubernetes self-healing hoạt động như thế nào?"**

A: K8s self-healing dựa trên control loop pattern: mỗi controller liên tục reconcile desired state với actual state.

ReplicaSet controller: desired state là 3 replicas. Khi một pod crash hoặc node fail, actual state là 2 replicas. Controller phát hiện drift, tạo pod mới trên node healthy. Thường trong vòng 30 giây.

Liveness probe: kubelet định kỳ check endpoint `/health` hoặc exec command trong container. Nếu fail liên tiếp `failureThreshold` lần, kubelet restart container. Tôi config `initialDelaySeconds: 30` để tránh restart app khi đang khởi động.

Readiness probe: tương tự liveness nhưng không restart — chỉ báo Service không route traffic tới pod chưa ready. Quan trọng khi deploy: pod mới spin up nhưng đang warm up cache, readiness probe fail nên không nhận traffic cho đến khi thực sự sẵn sàng.

Tôi đã test live trong ShopLite: `kubectl delete pod backend-7d9f8b-xxx -n shoplite`. Pod mới được tạo trong ~10 giây. Traffic không bị gián đoạn vì service vẫn route qua pod còn lại (HPA đảm bảo có ít nhất 2 replicas).

---

**Q5: "Blue-Green vs Canary deployment khác nhau thế nào?"**

A: Cả hai đều giải quyết vấn đề deploy không downtime nhưng với trade-off khác nhau.

Blue-Green: duy trì 2 environments song song (blue = current production, green = new version). Khi deploy, spin up green hoàn toàn, chạy smoke tests, rồi switch toàn bộ traffic từ blue sang green bằng cách đổi DNS hoặc load balancer target group. Rollback là switch ngược lại — tức thì. Nhược điểm: tốn gấp đôi resources và cần đồng bộ database state giữa 2 environments.

Canary: route một phần nhỏ traffic (ví dụ 5%) sang version mới, monitor error rate và latency, rồi tăng dần: 5% → 25% → 50% → 100%. Nếu metrics xấu ở bất kỳ bước nào, rollback 100% về old version. Ưu điểm: blast radius nhỏ, test với real traffic. Nhược điểm: phức tạp hơn nhiều, cần traffic splitting ở load balancer level (Nginx, Istio, hoặc AWS ALB weighted target groups).

ShopLite hiện dùng rolling update vì team nhỏ và traffic thấp: K8s thay thế dần từng pod, đảm bảo có ít nhất `maxUnavailable: 0` pod healthy. Nếu scale lên và có user impact khi deploy, sẽ chuyển sang canary với Argo Rollouts.

---

**Q6: "Terraform state là gì và tại sao phải lưu remote?"**

A: Terraform state file (`terraform.tfstate`) là "source of truth" về những resources nào Terraform đang quản lý và metadata của chúng (IDs, attributes). Không có state file, Terraform không biết EC2 instance nào trong AWS tương ứng với resource nào trong code.

State local: chỉ tồn tại trên máy của người chạy. Vấn đề: nếu 2 người cùng chạy `terraform apply`, không có coordination — có thể gây conflict, duplicate resources, hoặc destroy resources người kia vừa tạo.

Remote state trên S3 + DynamoDB: S3 lưu state file (versioned, encrypted), DynamoDB lưu lock record. Khi ai đó chạy `terraform apply`, Terraform acquire lock trong DynamoDB. Nếu ai khác cũng chạy cùng lúc, họ thấy lock và phải đợi. Sau khi apply xong, lock được release và state mới được write lên S3.

Config trong ShopLite:

```hcl
terraform {
  backend "s3" {
    bucket         = "shoplite-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    dynamodb_table = "shoplite-terraform-locks"
  }
}
```

State versioning trên S3 quan trọng: nếu `terraform apply` bị interrupt giữa chừng, state có thể corrupt. Có thể restore về version trước và chạy lại.

---

**Q7: "Bạn đảm bảo không có secrets trong code/repo như thế nào?"**

A: Defense in depth với nhiều lớp:

Lớp 1 — Developer machine: pre-commit hook dùng `gitleaks`. Mỗi lần `git commit`, gitleaks scan staged files tìm patterns giống secret (AWS keys, tokens, passwords). Nếu tìm thấy, commit bị block.

Lớp 2 — CI pipeline: `trufflehog` scan toàn bộ git history của PR, không chỉ changed files. Tại sao? Vì developer có thể đã lỡ commit secret, nhận ra, xóa đi trong commit kế tiếp — nhưng secret vẫn còn trong history.

Lớp 3 — Secrets management: mọi secrets (DB password, API keys, JWT secret) lưu trong AWS Secrets Manager. Application fetch tại runtime qua IAM role của EC2 — không cần access key/secret key cứng trong code.

Lớp 4 — K8s Secrets: Kubernetes Secrets được encrypted at rest bằng AWS KMS (khi dùng EKS). Secrets không xuất hiện trong Helm values files — được inject từ AWS Secrets Manager qua External Secrets Operator.

Nếu secret bị leak: revoke ngay lập tức (không chờ kiểm tra xem có ai dùng chưa), rotate credential, dùng `BFG Repo-Cleaner` để xóa khỏi git history, force push, thông báo cho cả team.

---

**Q8: "Giải thích 12-factor app"**

A: 12-factor app là methodology cho phép application scale horizontally và deploy dễ dàng trên cloud. Tôi áp dụng cho ShopLite:

**I. Codebase**: 1 repo, nhiều deploys (dev/staging/prod). Branch strategy: main là production, feature branches cho development.

**II. Dependencies**: Khai báo explicit trong `package.json`, không dùng global packages. `npm ci` trong Docker build đảm bảo reproducible.

**III. Config**: Mọi thứ thay đổi giữa environments (DB URL, API keys) đều là environment variables. Không hardcode. `.env.example` document tất cả variables cần thiết.

**IV. Backing Services**: DB, Redis, S3 được treat như attached resources. Connection string trong env var — có thể swap từ local Redis sang ElastiCache chỉ bằng đổi env var.

**VII. Port Binding**: App expose service qua port, không cần Tomcat hay Apache làm webserver riêng. Node.js app bind port 4000.

**IX. Disposability**: Fast startup (< 10s), graceful shutdown khi nhận SIGTERM: đóng kết nối DB, xử lý nốt request đang chờ, rồi exit.

**XI. Logs**: Treat logs as event streams, write tới stdout/stderr. Container runtime thu thập và ship qua Promtail. Không bao giờ write logs ra file trong container.

---

**Q9: "Tại sao dùng K8s thay Docker Compose cho production?"**

A: Docker Compose đủ tốt cho development và thậm chí production nhỏ trên single host. Nhưng single host có những vấn đề cơ bản:

Single point of failure: nếu host die, toàn bộ service down. Không có cơ chế tự động recover trên host mới.

Scale ngang không được: muốn chạy 3 backend containers, phải tự config load balancer. Muốn auto-scale theo CPU, không có built-in mechanism.

K8s giải quyết những vấn đề này: multi-node cluster (node fail thì pods reschedule sang node khác), HPA auto-scale dựa trên CPU/custom metrics, rolling deployments không downtime, built-in service discovery và load balancing, secrets management tốt hơn.

Trade-off thực tế: K8s phức tạp hơn nhiều. Với ShopLite, tôi maintain cả 2 setup: Docker Compose cho development (simple, fast), EKS cho production (resilient, scalable). Điều này cũng giúp demo cả 2 workflows trong phỏng vấn.

Khi nào KHÔNG cần K8s: side project nhỏ, team 1-2 người, traffic thấp. Docker Compose trên VPS $10/tháng là đủ và ít operational overhead hơn nhiều.

---

**Q10: "SLI, SLO, SLA là gì? Bạn đã định nghĩa chúng như thế nào cho ShopLite?"**

A: SLI (Service Level Indicator) là metric đo lường thực tế: tỷ lệ request thành công, p99 latency, availability. SLO (Service Level Objective) là target cho SLI: "99.9% requests thành công trong 30 ngày". SLA (Service Level Agreement) là contract pháp lý với khách hàng, thường penalty bằng tiền nếu vi phạm.

ShopLite SLO:
- Availability: 99.5% requests trả về HTTP 2xx hoặc 3xx (không phải 5xx)
- Latency: p99 < 500ms cho API requests
- Error budget: 0.5% = khoảng 3.6 giờ downtime/tháng

Implement trong Prometheus:

```yaml
# SLI: tỷ lệ request thành công
- record: job:request_success_rate:5m
  expr: |
    sum(rate(http_requests_total{status=~"2..|3.."}[5m]))
    /
    sum(rate(http_requests_total[5m]))

# Alert khi burn rate cao (tiêu hết error budget nhanh)
- alert: HighErrorBudgetBurnRate
  expr: job:request_success_rate:5m < 0.99
  for: 10m
  annotations:
    summary: "Error budget burn rate cao, sẽ vi phạm SLO trong vài giờ"
```

Khi error budget burn rate cao: freeze non-critical deployments, focus 100% vào stability. Đây là cách Google SRE quản lý tradeoff giữa velocity và reliability.

---

**Q11: "Container không start, debug như thế nào?"**

A: Quy trình debug theo thứ tự từ bên ngoài vào trong:

Bước 1: `docker ps -a` — xem container có tồn tại không, status là gì (Exited, OOMKilled, Restarting).

Bước 2: `docker logs container-name --tail 100` — xem error message cuối cùng. 90% lỗi giải thích được ở đây.

Bước 3: `docker inspect container-name | jq '.[0].State'` — xem exit code và OOMKilled flag. Exit code 137 = OOMKilled (thiếu memory). Exit code 1 = application error. Exit code 125 = Docker daemon error.

Bước 4: Nếu container không thể start: `docker run -it --entrypoint sh image-name` — chạy container với shell thay vì entrypoint mặc định. Explore filesystem, chạy tay command trong entrypoint, xem error.

Bước 5: Check Kubernetes specific: `kubectl describe pod pod-name -n namespace` — xem Events section. `kubectl logs pod-name -n namespace --previous` — logs của container lần trước (khi pod restart).

Nguyên nhân thường gặp và fix:
- OOMKilled: tăng `resources.limits.memory` trong deployment yaml
- CrashLoopBackOff với permission denied: kiểm tra USER directive và volume mount permissions
- Image pull error: check imagePullPolicy, image tag tồn tại, imagePullSecret

---

**Q12: "Multi-stage Docker build giúp gì?"**

A: Multi-stage build tách biệt build environment (cần compiler, dev tools, test frameworks) khỏi runtime environment (chỉ cần binary/compiled output).

Ví dụ ShopLite backend:

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci                          # Install TẤT CẢ dependencies (kể cả devDeps)
COPY src/ ./src/
RUN npm run build                   # Compile TypeScript → JavaScript

# Stage 2: Runtime
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --only=production        # Chỉ production dependencies
COPY --from=builder /app/dist ./dist  # Chỉ copy compiled output
RUN addgroup -S nodejs && adduser -S nodejs -G nodejs
USER nodejs
EXPOSE 4000
CMD ["node", "dist/index.js"]
```

Kết quả: image từ ~1.2GB (full Node.js + devDependencies + source TypeScript) xuống ~150MB (Node.js alpine + production deps + compiled JS). Nhỏ hơn = pull nhanh hơn khi deploy = deploy nhanh hơn.

Security benefit: không có TypeScript compiler, jest, nodemon trong runtime image. Ít packages hơn = ít CVEs tiềm ẩn hơn. Trivy scan report sạch hơn nhiều.

Cache benefit: `COPY package*.json ./` trước, `RUN npm ci` sau. Nếu source code thay đổi nhưng `package.json` không đổi, Docker dùng lại cached layer của `npm install` — build nhanh hơn đáng kể.

---

**Q13: "Ansible idempotency nghĩa là gì?"**

A: Idempotency nghĩa là chạy cùng một operation nhiều lần cho kết quả như chạy 1 lần. Ansible modules được thiết kế idempotent theo mặc định.

Module `apt`: trước khi install, check package đã có chưa. Nếu có và đúng version → skip, không install lại.

Module `copy`: tính checksum của file hiện tại trên server và file muốn copy. Nếu giống nhau → skip. Chỉ copy khi content khác.

Module `service`: start service chỉ khi đang stopped. Restart chỉ khi handler được notify (ví dụ: config file thay đổi thì mới restart nginx, không restart vô lý).

Tại sao quan trọng: bạn có thể chạy `ansible-playbook site.yml` sau bất kỳ thay đổi nào mà không lo phá vỡ thứ gì đang chạy. "Converge to desired state" — Ansible luôn đưa server về đúng trạng thái được define, dù server đang ở trạng thái nào.

Ví dụ không idempotent (tránh dùng):

```yaml
# SAI — chạy lần 2 sẽ fail vì user đã tồn tại
- name: Create user
  command: useradd myapp

# ĐÚNG — idempotent
- name: Create user
  user:
    name: myapp
    state: present
```

---

**Q14: "Docker networking hoạt động như thế nào?"**

A: Docker tạo virtual network interfaces trên host. Mỗi container có network namespace riêng với virtual ethernet interface (veth pair).

Bridge network (default): Docker tạo `docker0` bridge interface trên host. Containers trong cùng network có thể communicate qua container name (DNS resolution tự động). Ví dụ: backend container có thể reach database qua hostname `postgres` (service name trong docker-compose).

Trong docker-compose, tôi define custom network để isolate:

```yaml
networks:
  frontend-net:
    # frontend và nginx trong network này
  backend-net:
    # backend, postgres, redis trong network này
  # nginx trong cả 2 networks → làm reverse proxy
```

Nginx container join cả 2 networks: có thể nhận traffic từ internet và forward tới backend. Backend không expose port ra ngoài — chỉ accessible qua Nginx. Đây là network segmentation cơ bản.

Trong Kubernetes: mỗi Pod có IP riêng. Service tạo stable IP và DNS entry, route traffic tới healthy Pods. NetworkPolicy để control Pod-to-Pod communication — ví dụ: chỉ backend Pod mới được kết nối tới postgres Pod.

---

**Q15: "HTTPS và TLS hoạt động như thế nào trong setup của bạn?"**

A: TLS handshake xảy ra giữa client (browser) và Nginx (SSL termination point). Backend servers chỉ nhận plain HTTP trong internal network — không cần TLS vì traffic nằm trong VPC.

Setup với Let's Encrypt và Certbot:

```bash
# Lần đầu: obtain certificate
certbot --nginx -d shoplite.example.com -d www.shoplite.example.com

# Certbot tự động modify nginx.conf và reload
# Certificate tự động renew qua systemd timer mỗi 60 ngày
```

Nginx config sau khi Certbot setup:

```nginx
server {
    listen 80;
    server_name shoplite.example.com;
    return 301 https://$server_name$request_uri;  # HTTP → HTTPS redirect
}

server {
    listen 443 ssl http2;
    server_name shoplite.example.com;

    ssl_certificate /etc/letsencrypt/live/shoplite.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/shoplite.example.com/privkey.pem;

    # Mozilla recommended config
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # HSTS: browser chỉ dùng HTTPS trong 1 năm
    add_header Strict-Transport-Security "max-age=31536000" always;
}
```

Trong K8s: cert-manager tự động issue và renew Let's Encrypt certificates, lưu vào Kubernetes Secret, Ingress controller dùng Secret để terminate TLS.

---

**Q16: "Database indexing ảnh hưởng thế nào đến performance?"**

A: Index là data structure (thường B-tree) cho phép PostgreSQL tìm rows mà không cần scan toàn bộ table.

Ví dụ thực tế trong ShopLite: table `orders` có 1 triệu rows. Query `SELECT * FROM orders WHERE user_id = 123` không có index: PostgreSQL sequential scan toàn bộ 1 triệu rows — O(n). Với index trên `user_id`: B-tree lookup O(log n), rồi đọc chỉ các rows match.

Tôi phát hiện slow query qua Grafana: `pg_stat_statements` cho thấy query `GET /api/orders?userId=xxx` mất 800ms. `EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123` confirm Seq Scan. Fix:

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
-- CONCURRENTLY: không lock table, an toàn khi chạy trên production
```

Sau đó: query mất 2ms. Index Scan thay vì Seq Scan.

Trade-off: index tốn disk space và làm chậm INSERT/UPDATE/DELETE (phải update cả index). Không đặt index trên mọi column. Đặt index dựa trên slow query logs thực tế.

Composite index: `CREATE INDEX idx_orders_user_status ON orders(user_id, status)` optimize query `WHERE user_id = 123 AND status = 'pending'`. Column order quan trọng: leading column phải có selectivity cao.

---

**Q17: "Metrics nào có ý nghĩa thực sự? Làm thế nào để tránh vanity metrics?"**

A: Vanity metrics là những con số trông đẹp nhưng không giúp đưa ra quyết định. Ví dụ: tổng số requests — tăng có thể là tốt (nhiều user hơn) hoặc xấu (retry storm).

Meaningful metrics theo USE Method (cho infrastructure) và RED Method (cho services):

USE Method: Utilization (% time resource busy), Saturation (queue length), Errors.
- CPU utilization vs CPU throttling (throttled = saturation)
- Memory usage vs OOMKilled events (OOMKilled = errors)

RED Method: Rate (requests/sec), Errors (% failed requests), Duration (latency distribution).
- Không chỉ average latency — dùng percentile (p50, p95, p99). Average bị distort bởi outliers.
- p99 latency 500ms nghĩa là 1% users chờ > 500ms — có thể là 1000 người/ngày

ShopLite Grafana dashboard metrics:
- `http_request_duration_seconds` histogram → p95/p99 latency
- `http_requests_total{status=~"5.."}` → error rate
- `process_resident_memory_bytes` → memory leak detection
- `pg_stat_activity_count` → connection pool health
- `redis_connected_clients` → Redis usage

Alert dựa trên symptom (p99 latency > 1s) tốt hơn alert dựa trên cause (CPU > 80%). CPU cao không nhất thiết là vấn đề — chỉ là vấn đề khi ảnh hưởng đến user experience.

---

**Q18: "Rollback procedure của bạn như thế nào?"**

A: Rollback nhanh nhất luôn thắng. Mục tiêu: từ lúc phát hiện vấn đề đến lúc service restore phải dưới 5 phút.

Kubernetes rollback:

```bash
# Xem history
kubectl rollout history deployment/backend -n shoplite

# Rollback về revision trước
kubectl rollout undo deployment/backend -n shoplite

# Rollback về revision cụ thể
kubectl rollout undo deployment/backend --to-revision=3 -n shoplite

# Monitor rollback progress
kubectl rollout status deployment/backend -n shoplite
```

Docker Compose rollback:

```bash
# Set IMAGE_TAG về sha cũ và restart
export IMAGE_TAG=abc1234  # sha của version trước
docker compose -f docker-compose.prod.yml pull backend
docker compose -f docker-compose.prod.yml up -d --no-deps backend
```

Database rollback: phức tạp hơn. Nếu migration đã chạy, cần chạy down migration (Flyway `undo`). Hoặc restore từ pre-deploy snapshot (RDS automated backup). Đây là lý do pre-deploy backup quan trọng.

Điều quan trọng nhất: quyết định rollback ngay khi có dấu hiệu. Đừng chờ "điều tra thêm" khi user đang bị ảnh hưởng. Investigate AFTER rollback.

---

**Q19: "On-call best practices bạn biết?"**

A: On-call không chỉ là "nhận điện thoại lúc 3 giờ sáng". Đây là một role cần chuẩn bị kỹ.

Trước khi on-call: hiểu rõ system bằng cách đọc runbooks. Test runbooks: thử follow từng bước xem có đúng không. Đảm bảo có access vào tất cả tools cần thiết (Grafana, Kubectl, SSH vào servers). Setup escalation path: nếu bạn stuck sau 15 phút, ai là người tiếp theo để escalate.

Trong incident: ưu tiên restore service trước khi điều tra root cause. Communicate status ra ngoài (status page, Slack channel) mỗi 15-30 phút. Ghi lại timeline trong document real-time — memory không đáng tin khi stressed.

Sau incident: post-mortem blameless trong 24-48 giờ. 5 Whys để tìm root cause thực sự. Action items phải có owner và deadline. Share post-mortem với toàn team để học.

Alert hygiene: alert phải actionable — nếu không biết phải làm gì khi nhận alert, alert đó không có giá trị. Tuning alert thresholds để giảm false positives (alert fatigue là nguy hiểm).

---

**Q20: "DevSecOps là gì? Bạn implement như thế nào?"**

A: DevSecOps là shift security left — đưa security checks vào sớm nhất trong development lifecycle thay vì kiểm tra ở cuối.

Trong ShopLite pipeline:

**Pre-commit (developer machine):**
- `gitleaks` scan secrets
- `npm audit` check vulnerable dependencies

**CI Pipeline:**
- `trufflehog` scan git history
- `npm audit --audit-level=high` fail build nếu có high vulnerability
- `trivy image` scan Docker image sau khi build
- SAST với CodeQL (GitHub native, free cho public repo)

**Container security:**
- Non-root user trong Dockerfile
- Read-only root filesystem: `securityContext.readOnlyRootFilesystem: true`
- Drop all capabilities: `securityContext.capabilities.drop: [ALL]`
- No privilege escalation: `securityContext.allowPrivilegeEscalation: false`

**Runtime:**
- Secrets từ AWS Secrets Manager, không bao giờ trong environment variables của container (nếu ai exec vào container có thể `env` để xem)
- Network Policies trong K8s để limit pod-to-pod communication
- AWS Security Groups: port 22 chỉ mở cho bastion host IP, không mở cho 0.0.0.0/0

Security không phải optional — đây là yêu cầu cơ bản mà mọi DevOps engineer cần biết.

---

### 5. Demo Script cho phỏng vấn (10 phút)

Bài demo 10 phút là cơ hội tốt nhất để phân biệt bạn với những ứng viên chỉ biết lý thuyết. Chuẩn bị kỹ: chạy thử ít nhất 3 lần trước khi phỏng vấn thật.

**Setup trước khi vào phỏng vấn:**
- Mở sẵn Grafana dashboard
- Mở sẵn GitHub Actions tab của repo
- Mở sẵn terminal với kubectl configured và namespace set
- Có sẵn một commit nhỏ (sửa string trong frontend) để demo CI/CD
- Kiểm tra tất cả services đang chạy: `docker compose ps` hoặc `kubectl get pods -n shoplite`

**Phút 0-2: Grafana Dashboard**

Mở Grafana → Overview dashboard. Nói: "Đây là monitoring dashboard của ShopLite. Tôi có 3 panels quan trọng nhất: Request Rate (hiện tại 45 req/min), Error Rate (0.2% — trong SLO 0.5%), và p99 Latency (180ms — trong SLO 500ms)."

Chuyển sang Application dashboard: "Panel này show HTTP status codes breakdown theo thời gian. Panel database show connection pool utilization — hiện tại 3/10 connections đang được dùng."

Point vào một spike nhỏ trong history: "Đây là lúc tôi chạy load test hôm qua. CPU tăng nhưng latency vẫn trong SLO — horizontal scaling hoạt động đúng."

**Phút 2-4: Trigger CI/CD**

Chuyển sang VS Code. "Tôi sẽ thực hiện một thay đổi nhỏ và push."

```bash
# Sửa file frontend/src/components/Header.jsx
# Đổi version string hoặc copyright year
git add frontend/src/components/Header.jsx
git commit -m "feat(ui): update footer copyright year"
git push origin main
```

Chuyển sang GitHub Actions tab: "CI đang chạy. Bạn có thể thấy 3 jobs song song: lint, unit-tests, và integration-tests. Integration tests spin up PostgreSQL service container và test actual DB queries."

"Tôi config `fail-fast: false` để tất cả jobs chạy và báo cáo đầy đủ, không dừng lại khi có 1 job fail."

**Phút 4-6: CD Deploy**

CI pass → CD workflow trigger. "CD cần manual approval — đây là gate trước production. Trong real workflow, team lead approve."

Click Approve. "Docker image đang được build với multi-stage build. Tag là git SHA ngắn — `abc1234`. Push lên GHCR. Sau đó SSH vào EC2 và chạy `docker compose pull && docker compose up -d`."

Show deployment succeeds: "Deploy xong. Zero-downtime vì Docker Compose health check — container mới phải pass health check trước khi cũ bị stop."

**Phút 6-8: Kubernetes Self-healing**

Chuyển sang terminal.

```bash
# Show current pods
kubectl get pods -n shoplite -w &

# Delete một pod trong khi watch
kubectl delete pod backend-7d9f8b-xxx -n shoplite
```

"Tôi vừa xóa một pod — simulate node failure. Nhìn vào watch output: pod đang Terminating và một pod mới đang ContainerCreating. Trong vòng... 12 giây, pod mới đã Running. Service không bị gián đoạt vì HPA ensure minimum 2 replicas — pod còn lại handle traffic trong thời gian này."

```bash
# Show ReplicaSet ensuring desired state
kubectl describe replicaset -n shoplite | grep -A 5 "Conditions"
```

**Phút 8-10: Loki Log Query**

Chuyển lại Grafana → Explore → Loki datasource.

"Đây là log aggregation với Loki. Query: `{service="backend", environment="production"} | json | level="error"`."

Show kết quả. "Mỗi log entry có structured fields: timestamp, level, service, traceId, userId. TraceId cho phép correlate logs qua nhiều services."

Demo live tail: "Tôi có thể live tail logs trong khi deploy để detect lỗi ngay lập tức. Đây là cách tôi debug lúc 3 giờ sáng — không cần SSH vào server, tất cả logs tập trung ở đây."

**Cách kết thúc demo:**
"Đây là full cycle: code change → automated CI → manual-approved CD → self-healing infrastructure với centralized observability. Tôi có thể đi sâu vào bất kỳ phần nào bạn muốn tìm hiểu thêm."

---

### 6. Lộ trình tiếp theo sau Junior DevOps

Đây là roadmap thực tế dựa trên market demand ở Việt Nam và khu vực Đông Nam Á.

**6 tháng tiếp — Chứng chỉ AWS SAA:**

AWS Solutions Architect Associate (SAA-C03) là chứng chỉ có ROI cao nhất cho DevOps engineers. Chi phí: ~$150 thi, ~$30-50 học qua Udemy (Adrian Cantrill hoặc Stephane Maarek). Thời gian học: 60-80 giờ nghiêm túc.

Tại sao SAA trước CKA: AWS SAA mở ra nhiều cơ hội hơn vì hầu hết công ty Việt Nam đang dùng AWS. SAA cũng reinforce rất nhiều concepts bạn đã học trong 13 phases này: VPC, IAM, RDS, S3, EKS, CloudWatch.

Kỹ năng cần build song song với SAA:
- Cost optimization: Reserved Instances, Spot Instances, S3 storage classes
- Multi-AZ architecture: tại sao và khi nào dùng
- IAM advanced: cross-account roles, service control policies

**9 tháng — CKA:**

Certified Kubernetes Administrator ($395) là chứng chỉ hands-on, thi trong terminal. Không múc lý thuyết — phải thực sự làm được trong 2 giờ thi.

Chuẩn bị bằng cách: killer.sh practice exams (2 sessions included với exam registration), Udemy course của Mumshad Mannambeth (tác giả KodeKloud). Practice trên local cluster với kubeadm hoặc KodeKloud playground.

Topics thường gặp trong CKA: cluster upgrade với kubeadm, ETCD backup/restore, networking (CNI, NetworkPolicies), RBAC, persistent volumes, troubleshooting (broken nodes, failed deployments).

**12 tháng — GitOps:**

ArgoCD là industry standard cho GitOps: Git repository là single source of truth, ArgoCD sync cluster state với Git. Setup ArgoCD trên EKS cluster, convert Helm deployments sang ArgoCD Applications. Learn ApplicationSets cho multi-environment deployment.

Flux là alternative với different philosophy — biết cả 2 sẽ impress interviewer.

Service mesh basics với Istio: traffic management (canary deployments native), mTLS between services, distributed tracing với Jaeger. Không cần CertifiedIstio chứng chỉ, chỉ cần hiểu concepts và basic setup.

**Skills để lên Mid-level:**

On-call experience là thứ không thể học từ sách. Tình nguyện on-call ngay khi có cơ hội. Mỗi incident handled là kinh nghiệm vô giá.

Multi-region deployment: DR setup với Route53 health checks và failover routing. RDS Multi-AZ vs Read Replicas. S3 Cross-Region Replication.

Cost optimization thực tế: Right-sizing EC2 instances bằng CloudWatch metrics. Identifying và xóa unused resources. Savings Plans analysis. Đây là kỹ năng nhiều junior thiếu và rất được đánh giá cao.

Microservices patterns: Circuit breaker (Resilience4j, Polly), Saga pattern cho distributed transactions, Event sourcing basics, Message queues (SQS, RabbitMQ, Kafka basics).

Event-driven architecture: SQS/SNS cho async processing, Lambda cho event handlers, EventBridge cho event routing. Biến batch jobs thành event-driven pipelines.

---

### 7. Nhật ký học tập — Learning Journal

Ghi lại hành trình học là khoản đầu tư tốt nhất bạn có thể làm. Mỗi lần debug thành công, mỗi concept "aha" moment, mỗi lần system fail và bạn tìm ra nguyên nhân — đó là material cho phỏng vấn.

**Template cho `learning-journal.md`:**

```markdown
# Learning Journal — DevOps Journey

## Phase 1 — Linux & Shell Scripting
### 2024-01-15 — 2024-01-22

**Học được:**
- File permissions: chmod 755 vs 644, setuid/setgid
- Process management: systemctl, journalctl, ps aux
- Shell scripting: loops, conditionals, functions, error handling với set -e

**Khó khăn gặp phải:**
- Không hiểu tại sao script chạy ok trên máy mình nhưng fail trên server
- Bash vs sh syntax differences gây confuse

**Cách giải quyết:**
- Đọc `man bash` cho shebangs: #!/bin/bash vs #!/bin/sh
- Dùng shellcheck để lint scripts — phát hiện bashisms trong sh scripts
- Test scripts trên Docker container alpine/ubuntu để simulate clean environment

**Insight quan trọng:**
- `set -euo pipefail` ở đầu mỗi script: fail fast, không tiếp tục khi có lỗi
- Luôn quote variables: "$VAR" thay vì $VAR để handle spaces

**Sẽ dùng khi phỏng vấn khi được hỏi về:**
- "Bạn handle errors trong shell scripts như thế nào?"
- "Tại sao scripts của bạn đáng tin cậy?"

---

## Phase 3 — Docker
### 2024-02-05 — 2024-02-19

**Học được:**
- Dockerfile best practices: layer caching, multi-stage builds, non-root user
- Docker networking: bridge, host, overlay networks
- Volume management: bind mounts vs named volumes

**Khó khăn gặp phải:**
- Node.js app in container không connect được tới PostgreSQL
- "Connection refused to localhost:5432" — mất 1 tiếng debug

**Cách giải quyết:**
- "localhost" trong container là container's localhost, không phải host's localhost
- Phải dùng service name "postgres" trong docker-compose (DNS resolution)
- Lesson learned: network isolation là feature, không phải bug

**Insight quan trọng:**
- Docker layer caching: instructions ít thay đổi phải đặt trước. COPY package.json trước npm install, sau đó mới COPY source code.
- Image size ảnh hưởng trực tiếp đến deploy time trên production

**Sẽ dùng khi phỏng vấn khi được hỏi về:**
- "Container của bạn không connect được DB, debug như thế nào?"
- "Tối ưu Docker image như thế nào?"
```

**Cách dùng journal hiệu quả:**

Viết ngay sau khi giải quyết được vấn đề — không chờ đến cuối phase. Memory fade rất nhanh: vấn đề bạn debug 3 tiếng hôm nay, 2 tuần sau chỉ nhớ mơ hồ.

Format "Sẽ dùng khi phỏng vấn khi được hỏi về" rất quan trọng: nó buộc bạn translate kỹ thuật học được thành narrative cho phỏng vấn. Phỏng vấn là storytelling về kỹ thuật.

Tối thiểu 3 entries/phase. Mỗi phase nên có ít nhất 1 story về "lúc đầu không hiểu, sau debug ra thấy nguyên nhân là..." — những story này authentic và dễ nhớ.

Sau 13 phases, bạn sẽ có ~40 stories kỹ thuật. Đủ để trả lời mọi câu hỏi phỏng vấn "Tell me about a time when..." với ví dụ cụ thể từ kinh nghiệm thực tế của chính bạn.

---

## Kỹ năng đạt được
- Trình bày một dự án kỹ thuật mạch lạc, có chiều sâu.
- Viết tài liệu kỹ thuật chuẩn doanh nghiệp.
- Tự tin trả lời phỏng vấn dựa trên kinh nghiệm thực tế.

---

## Thực hành

**Lab:**
- Viết tài liệu kiến trúc tổng thể + sơ đồ hệ thống cuối cùng.
- Viết README chuyên nghiệp cho repo (có badge CI, hướng dẫn chạy, kiến trúc).
- Quay một video demo ngắn (hoặc viết case study) toàn bộ luồng: commit → CI/CD → deploy → monitoring.
- Viết các runbook vận hành.
- Tổng duyệt: giả lập sự cố và xử lý từ đầu đến cuối.

**Công cụ:** draw.io, Markdown, GitHub.

---

## Bài tập

- **Bắt buộc:** README + tài liệu kiến trúc hoàn chỉnh trên repo.
- **Nâng cao:** Viết một blog/case study mô tả dự án và những bài học rút ra.
- **Mô phỏng doanh nghiệp:** Tự phỏng vấn với 20 câu hỏi DevOps phổ biến, ghi lại câu trả lời dựa trên dự án.

---

## Deliverable
- [ ] Repo ShopLite hoàn chỉnh, tài liệu đầy đủ, public trên GitHub.
- [ ] Sơ đồ kiến trúc cuối + bộ runbook.
- [ ] CV mục dự án + danh sách kỹ năng + bộ câu hỏi phỏng vấn đã chuẩn bị.

---

## Tiêu chí hoàn thành
- [ ] Người lạ đọc repo trong 5 phút hiểu được dự án làm gì và kiến trúc ra sao.
- [ ] Trình bày được toàn bộ dự án trong 10 phút.
- [ ] Trả lời tự tin các câu hỏi phỏng vấn dựa trên chính dự án.

---

## Câu hỏi phỏng vấn DevOps phổ biến

### Câu hỏi về DevOps & Mindset
1. DevOps là gì và giải quyết vấn đề gì?
2. Sự khác biệt giữa CI và CD là gì?
3. Bạn xử lý sự cố production như thế nào?

### Câu hỏi về Linux & Networking
4. Giải thích quy trình khi bạn gõ URL vào trình duyệt.
5. Làm thế nào để debug một service không khởi động được?
6. Sự khác biệt giữa TCP và UDP?

### Câu hỏi về Docker & Container
7. Container khác VM ở điểm nào?
8. Giải thích Docker layer và tại sao nó quan trọng.
9. Làm thế nào để tối ưu Dockerfile?

### Câu hỏi về Kubernetes
10. Pod là gì? Deployment là gì?
11. Service và Ingress khác nhau thế nào?
12. K8s tự phục hồi như thế nào?

### Câu hỏi về CI/CD
13. Mô tả pipeline CI/CD của bạn.
14. Bạn xử lý secrets trong CI/CD như thế nào?
15. Rolling deploy và Blue-Green deploy khác nhau thế nào?

### Câu hỏi về Monitoring
16. Metrics, Logs, Traces khác nhau thế nào?
17. SLO là gì? Bạn định nghĩa SLO thế nào cho ShopLite?
18. Khi nhận alert lúc 3 giờ sáng, bạn làm gì đầu tiên?

### Câu hỏi về Cloud & IaC
19. Giải thích kiến trúc VPC bạn đã dựng.
20. Tại sao dùng Terraform thay vì click trên Console?

---

## Ghi chú học tập

> _Ghi lại những điều học được, vướng mắc và insight trong quá trình học phase này._
