# Phase 7 — CI/CD

## Mục tiêu
- Tự động hóa toàn bộ quy trình từ commit code → build → test → deploy.
- Loại bỏ deploy thủ công, giảm lỗi con người.

---

## Kiến thức sẽ học

- **Khái niệm CI/CD & pipeline** (Continuous Integration / Delivery / Deployment).
- **GitHub Actions (hoặc GitLab CI)** (workflow, job, step, runner, trigger).
- **Build & test tự động** — đảm bảo code luôn chạy được.
- **Build & push Docker image trong pipeline** — gắn với Phase 4.
- **Secrets trong CI/CD** — không để lộ thông tin nhạy cảm.
- **Tự động deploy** (qua SSH/registry) và **chiến lược deploy** (rolling, blue-green cơ bản).
- **Quản lý môi trường** (dev/staging/prod).

---

## Kiến thức chi tiết

### 1. Khái niệm CI/CD

#### Continuous Integration (CI)

Continuous Integration là thực hành phát triển phần mềm trong đó các developer merge code vào nhánh chính thường xuyên, ít nhất 1 lần mỗi ngày. Mỗi lần merge sẽ tự động kích hoạt quá trình build và chạy test để phát hiện lỗi tích hợp sớm nhất có thể.

**Nguyên tắc cốt lõi:**
- Code được merge thường xuyên, tránh "integration hell" khi merge sau nhiều ngày.
- Mỗi commit phải pass toàn bộ automated tests trước khi được coi là hợp lệ.
- Build phải nhanh (dưới 10 phút) để feedback loop ngắn.
- Fix broken build là ưu tiên cao nhất của team, không ai được merge khi build đang đỏ.

**Lợi ích thực tế:**
- Phát hiện lỗi sớm khi còn dễ fix (conflict nhỏ hơn, context còn tươi trong đầu).
- Giảm rủi ro merge lớn cuối sprint — không có "big bang integration".
- Code luôn ở trạng thái có thể build được.
- Tăng confidence khi refactor vì test chạy tự động ngay lập tức.

#### Continuous Delivery (CD)

Continuous Delivery đảm bảo code luôn ở trạng thái có thể deploy lên production bất cứ lúc nào. Quá trình deploy là quyết định của con người (bấm nút), không phải tự động hoàn toàn.

**Đặc điểm:**
- Pipeline chạy đến staging tự động, production cần approval.
- Mọi commit trên nhánh main phải pass đầy đủ acceptance tests.
- Deployment process được tự động hóa, chỉ thiếu bước "bấm nút xác nhận".
- Team có thể deploy bất cứ lúc nào trong giờ làm việc với rủi ro thấp.

#### Continuous Deployment

Continuous Deployment là bước tiếp theo của Continuous Delivery — 100% tự động deploy lên production sau khi pass toàn bộ tests, không cần bất kỳ human approval nào.

**Khi nào dùng:**
- Khi test coverage rất cao và đáng tin cậy.
- Khi team có culture "if it's not tested, it doesn't exist".
- Phù hợp với SaaS products, ít phù hợp với enterprise software có compliance requirements.

**Khác biệt quan trọng:**

| Thực hành | Tự động đến đâu | Ai quyết định deploy prod |
|---|---|---|
| Continuous Integration | Build + test | N/A |
| Continuous Delivery | Build + test + deploy staging | Con người (bấm nút) |
| Continuous Deployment | Build + test + deploy staging + deploy prod | Không ai (tự động 100%) |

#### DORA Metrics — Đo lường hiệu quả DevOps

DORA (DevOps Research and Assessment) là bộ 4 metrics được Google nghiên cứu và công nhận là tiêu chuẩn đo lường hiệu quả của một team DevOps. Đây là metrics thực tế được dùng để đánh giá mức độ trưởng thành của CI/CD process.

**1. Deployment Frequency (Tần suất deploy)**
- Đo: Team deploy lên production bao nhiêu lần trong một khoảng thời gian.
- Elite: Multiple deployments per day (nhiều lần mỗi ngày).
- High: Between once per day and once per week.
- Medium: Between once per week and once per month.
- Low: Less than once per month.
- Ý nghĩa: Frequency cao nghĩa là batch size nhỏ, rủi ro mỗi deploy thấp, feedback nhanh.

**2. Lead Time for Changes (Thời gian từ commit đến production)**
- Đo: Từ khi developer commit code cho đến khi code đó chạy trên production.
- Elite: Less than one hour (dưới 1 tiếng).
- High: Between one day and one week.
- Medium: Between one month and six months.
- Low: More than six months.
- Ý nghĩa: Lead time ngắn nghĩa là pipeline hiệu quả, không có bottleneck thủ công.

**3. Mean Time to Recovery — MTTR (Thời gian phục hồi trung bình)**
- Đo: Khi có incident trên production, mất bao lâu để phục hồi về trạng thái bình thường.
- Elite: Less than one hour.
- High: Less than one day.
- Medium: Between one day and one week.
- Low: More than one week.
- Ý nghĩa: MTTR thấp nghĩa là team có observability tốt, quy trình rollback rõ ràng.

**4. Change Failure Rate (Tỷ lệ deploy gây ra sự cố)**
- Đo: Phần trăm deployments dẫn đến degradation of service hoặc cần hotfix/rollback.
- Elite: 0%–5%.
- High: 0%–15%.
- Medium/Low: 16%–30%.
- Ý nghĩa: Rate thấp nghĩa là test coverage tốt, staging environment reflect production chính xác.

**Cách dùng DORA metrics trong thực tế:**
- Đo baseline hiện tại trước khi cải thiện.
- Chọn 1–2 metrics yếu nhất để focus cải thiện.
- Track theo quý, không theo tuần (quá nhiều noise).
- Không dùng để so sánh giữa các team, chỉ so với baseline của chính team đó.

---

### 2. GitHub Actions Architecture

GitHub Actions là nền tảng CI/CD được tích hợp trực tiếp vào GitHub. Hiểu rõ kiến trúc giúp viết workflow hiệu quả và debug khi có vấn đề.

#### Workflow

Workflow là file YAML nằm trong thư mục `.github/workflows/` của repository. Một repo có thể có nhiều workflow. Workflow được kích hoạt bởi events.

```
.github/
  workflows/
    ci.yml          # chạy khi push/PR
    cd.yml          # chạy khi merge vào main
    nightly.yml     # chạy theo schedule
    release.yml     # chạy khi tạo release
```

#### Events (Triggers)

Event là điều kiện kích hoạt workflow. Các event phổ biến nhất:

| Event | Mô tả | Ví dụ dùng khi |
|---|---|---|
| `push` | Khi có commit được push | Chạy CI trên mọi push |
| `pull_request` | Khi tạo/cập nhật PR | Validate trước khi merge |
| `schedule` | Theo lịch cron | Nightly builds, security scans |
| `workflow_dispatch` | Trigger thủ công qua UI | Deploy khẩn cấp, one-off tasks |
| `release` | Khi tạo GitHub Release | Build và publish artifacts |
| `workflow_call` | Gọi từ workflow khác | Reusable workflows |

Ví dụ filter event theo branch và path:
```yaml
on:
  push:
    branches: [main, develop]
    paths:
      - 'backend/**'    # chỉ trigger khi thay đổi trong backend/
      - '!**.md'        # bỏ qua file markdown
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]
```

#### Job

Job là một nhóm các steps chạy trên cùng một runner. Mặc định các jobs chạy song song. Dùng `needs` để tạo dependency giữa các jobs.

```yaml
jobs:
  job-a:          # chạy ngay
    runs-on: ubuntu-latest
    ...

  job-b:          # chạy ngay (song song với job-a)
    runs-on: ubuntu-latest
    ...

  job-c:          # chờ job-a và job-b xong mới chạy
    needs: [job-a, job-b]
    runs-on: ubuntu-latest
    ...
```

**Truyền output giữa jobs:**
```yaml
jobs:
  job-a:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.get-version.outputs.version }}
    steps:
      - id: get-version
        run: echo "version=1.2.3" >> $GITHUB_OUTPUT

  job-b:
    needs: job-a
    runs-on: ubuntu-latest
    steps:
      - run: echo "Version is ${{ needs.job-a.outputs.version }}"
```

#### Step

Step là đơn vị nhỏ nhất trong một job. Mỗi step có thể là:
- `uses`: Sử dụng một action có sẵn.
- `run`: Chạy shell command trực tiếp.

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v4      # dùng action

  - name: Run tests
    run: |                          # chạy shell commands
      npm ci
      npm test
    working-directory: ./backend
    env:
      NODE_ENV: test
```

#### Action

Action là đơn vị tái sử dụng trong GitHub Actions. Có 3 loại:
- **Marketplace actions**: `actions/checkout@v4`, `docker/build-push-action@v5`.
- **Repository actions**: action trong cùng repo (`.github/actions/my-action`).
- **Docker actions**: action chạy trong Docker container.

**Luôn pin action version bằng SHA hoặc tag cụ thể:**
```yaml
# Tốt — dùng tag cụ thể
uses: actions/checkout@v4

# Tốt hơn — pin bằng SHA (bảo mật cao nhất)
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683

# Xấu — dùng branch, dễ bị tấn công supply chain
uses: actions/checkout@main
```

#### Runner

Runner là máy chủ thực thi jobs. GitHub cung cấp hosted runners miễn phí:

| Runner | OS | CPU | RAM |
|---|---|---|---|
| `ubuntu-latest` | Ubuntu 24.04 | 4 vCPU | 16 GB |
| `windows-latest` | Windows Server 2022 | 4 vCPU | 16 GB |
| `macos-latest` | macOS 14 | 3 vCPU (M1) | 7 GB |

**Self-hosted runner** là runner do bạn quản lý, chạy trên máy chủ của bạn. Phù hợp khi cần:
- Phần cứng đặc biệt (GPU, ARM).
- Truy cập private network (database nội bộ).
- Tiết kiệm chi phí cho large repos với nhiều CI minutes.

#### Contexts và Expressions

Contexts cung cấp thông tin về workflow run, repository, event, runner, và nhiều thứ khác.

```yaml
# github context — thông tin về repository và event
${{ github.sha }}           # full commit SHA (ví dụ: abc1234...)
${{ github.ref }}           # ref đầy đủ (ví dụ: refs/heads/main)
${{ github.ref_name }}      # tên branch/tag ngắn (ví dụ: main)
${{ github.event_name }}    # tên event (push, pull_request, ...)
${{ github.repository }}    # owner/repo (ví dụ: myorg/myapp)
${{ github.actor }}         # username kích hoạt workflow
${{ github.run_number }}    # số thứ tự của run (1, 2, 3, ...)

# secrets context — giá trị bí mật (không bao giờ in ra log)
${{ secrets.MY_SECRET }}
${{ secrets.GITHUB_TOKEN }} # token tự động, không cần tạo

# env context — environment variables
${{ env.MY_VAR }}

# steps context — output từ step trước
${{ steps.my-step-id.outputs.my-output }}
${{ steps.my-step-id.outcome }}   # success, failure, cancelled, skipped

# needs context — output từ job khác
${{ needs.build.outputs.image-tag }}
${{ needs.build.result }}          # success, failure, cancelled, skipped

# runner context
${{ runner.os }}            # Linux, Windows, macOS
${{ runner.arch }}          # X64, ARM64
```

**Conditional expressions:**
```yaml
steps:
  - name: Deploy only on main
    if: github.ref == 'refs/heads/main'
    run: ./deploy.sh

  - name: Run even if previous step failed
    if: always()
    run: ./cleanup.sh

  - name: Run only if previous step succeeded
    if: success()
    run: ./notify.sh

  - name: Skip on draft PR
    if: github.event.pull_request.draft == false
    run: ./expensive-test.sh
```

---

### 3. CI Workflow cho ShopLite

File `.github/workflows/ci.yml` đầy đủ — chạy lint, test với database thật, và build Docker image:

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

# Cancel in-progress runs khi có push mới trên cùng branch
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Restore dependencies
        run: dotnet restore
        working-directory: ./backend

      - name: Check formatting
        run: dotnet format --verify-no-changes
        working-directory: ./backend

      - name: Build (check for compile errors)
        run: dotnet build --no-restore --configuration Release
        working-directory: ./backend

  test:
    name: Test
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: shoplite_test
          POSTGRES_USER: shoplite
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Restore dependencies
        run: dotnet restore
        working-directory: ./backend

      - name: Run database migrations (EF Core)
        run: dotnet ef database update
        working-directory: ./backend
        env:
          ConnectionStrings__DefaultConnection: "Host=localhost;Port=5432;Database=shoplite_test;Username=shoplite;Password=test"

      - name: Run tests
        run: dotnet test --no-restore --collect:"XPlat Code Coverage"
        working-directory: ./backend
        env:
          ConnectionStrings__DefaultConnection: "Host=localhost;Port=5432;Database=shoplite_test;Username=shoplite;Password=test"
          ConnectionStrings__Redis: "localhost:6379"
          JwtSettings__Secret: "test-jwt-secret-for-ci-only-must-be-long-enough"
          ASPNETCORE_ENVIRONMENT: Test

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-report
          path: backend/TestResults/
          retention-days: 7

  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [lint, test]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build image (no push on PR)
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          push: false
          tags: shoplite/backend:ci-test
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: shoplite/backend:ci-test
          format: table
          exit-code: 1
          severity: CRITICAL,HIGH
```

**Giải thích các phần quan trọng:**

`concurrency`: Tự động cancel run cũ khi có push mới trên cùng branch. Tiết kiệm CI minutes, tránh queue dài khi developer push nhiều lần liên tiếp.

`services`: Spin up containers phụ trợ (postgres, redis) cho job. Containers này tự động khởi động trước steps và bị xóa sau khi job xong. `options` với `--health-cmd` đảm bảo service sẵn sàng trước khi step đầu tiên chạy.

`setup-dotnet`: Cache NuGet packages tự động theo dotnet-version. `dotnet restore` lần đầu mất 1–2 phút tải packages, các lần sau dùng cache chỉ mất vài giây.

`dotnet ef database update`: Chạy EF Core migrations trước khi test. Cần package `Microsoft.EntityFrameworkCore.Design` và tool `dotnet-ef` được cài trong csproj hoặc global tool.

`cache-from/cache-to: type=gha`: Docker layer cache lưu trong GitHub Actions cache storage. Build lần đầu mất 5–8 phút, build sau khi chỉ thay đổi code (không thay đổi dependencies) chỉ mất 30–60 giây.

`retention-days: 7`: Artifacts (coverage report) chỉ giữ 7 ngày để tiết kiệm storage.

---

### 4. CD Workflow cho ShopLite

File `.github/workflows/cd.yml` đầy đủ — build và push image, deploy staging tự động, deploy production với approval:

```yaml
name: CD

on:
  push:
    branches: [main]

concurrency:
  group: cd-${{ github.ref }}
  cancel-in-progress: false   # KHÔNG cancel CD, tránh deploy dở dang

jobs:
  build-and-push:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Generate Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}/backend
          tags: |
            type=sha,prefix=main-,format=short
            type=raw,value=latest,enable={{is_default_branch}}
            type=raw,value=stable,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push image
        id: build
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            GIT_SHA=${{ github.sha }}

      - name: Print image info
        run: |
          echo "Image tags: ${{ steps.meta.outputs.tags }}"
          echo "Image digest: ${{ steps.build.outputs.digest }}"

  deploy-staging:
    name: Deploy to Staging
    needs: build-and-push
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.shoplite.example.com

    steps:
      - name: Deploy to staging server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ubuntu
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          script: |
            set -e

            echo "=== Pulling new image ==="
            cd /app/shoplite
            export IMAGE_TAG=main-$(echo ${{ github.sha }} | cut -c1-7)
            echo "IMAGE_TAG=${IMAGE_TAG}" > .env.deploy

            echo "=== Updating docker-compose override ==="
            cat > docker-compose.override.yml <<EOF
            services:
              backend:
                image: ghcr.io/${{ github.repository }}/backend:${IMAGE_TAG}
            EOF

            echo "=== Pulling image ==="
            docker compose pull backend

            echo "=== Starting new container ==="
            docker compose up -d backend

            echo "=== Running migrations ==="
            docker compose exec -T backend dotnet ef database update

            echo "=== Health check ==="
            sleep 10
            docker compose exec -T backend curl -f http://localhost:8080/health || exit 1

            echo "=== Deploy staging complete ==="

      - name: Notify staging deploy success
        if: success()
        run: echo "Staging deploy successful for commit ${{ github.sha }}"

  run-smoke-tests:
    name: Run Smoke Tests on Staging
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run smoke tests against staging
        run: |
          npm ci
          npm run test:smoke
        working-directory: ./tests
        env:
          BASE_URL: https://staging.shoplite.example.com
          TEST_TIMEOUT: 30000

  deploy-production:
    name: Deploy to Production
    needs: [build-and-push, run-smoke-tests]
    runs-on: ubuntu-latest
    environment:
      name: production   # environment này yêu cầu manual approval
      url: https://shoplite.example.com

    steps:
      - name: Deploy to production server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ubuntu
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          script: |
            set -e

            echo "=== Starting production deploy ==="
            cd /app/shoplite
            export IMAGE_TAG=main-$(echo ${{ github.sha }} | cut -c1-7)

            echo "=== Pulling all images ==="
            docker compose pull

            echo "=== Rolling update backend ==="
            docker compose up -d --no-deps backend

            echo "=== Running migrations ==="
            docker compose exec -T backend dotnet ef database update

            echo "=== Health check ==="
            sleep 15
            docker compose exec -T backend curl -f http://localhost:8080/health || exit 1

            echo "=== Cleaning up old images ==="
            docker image prune -f

            echo "=== Production deploy complete ==="

      - name: Tag release in GitHub
        if: success()
        run: |
          gh release create "deploy-$(date +%Y%m%d-%H%M%S)" \
            --title "Production Deploy $(date +%Y-%m-%d)" \
            --notes "Deployed commit ${{ github.sha }}" \
            --target ${{ github.sha }}
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Giải thích flow CD:**

1. **build-and-push**: Build image, gắn tag bao gồm short SHA (ví dụ `main-abc1234`) và `latest`. Push lên GitHub Container Registry (GHCR).

2. **deploy-staging**: SSH vào staging server, tạo file override với image tag mới, pull image, restart container, chạy migration, health check. Chạy tự động ngay sau build.

3. **run-smoke-tests**: Chạy test cơ bản trên staging URL để verify deploy thành công.

4. **deploy-production**: Chỉ chạy sau khi smoke test pass VÀ có manual approval (cấu hình trong GitHub Environments). Deploy tương tự staging, thêm bước tạo release tag.

**`concurrency: cancel-in-progress: false`**: Khác với CI, CD không nên cancel mid-deploy vì có thể để lại server ở trạng thái không nhất quán. Nếu có 2 CD chạy cùng lúc, run sau sẽ queue đợi run trước hoàn thành.

---

### 5. Secrets Management

#### Tại sao cần quản lý secrets cẩn thận

Secrets (SSH keys, passwords, API tokens) nếu lộ trong log hoặc source code sẽ dẫn đến breach. GitHub tự động mask secrets trong logs, nhưng cần tránh các cách bypass như echo, base64 encode, hay lưu vào file.

#### Các loại secrets trong GitHub Actions

**Repository secrets**: Áp dụng cho toàn bộ repository, mọi workflow đều có thể dùng.

**Environment secrets**: Chỉ available khi job chạy trong environment cụ thể (staging, production). Phù hợp với credentials riêng cho từng môi trường.

**Organization secrets**: Chia sẻ across nhiều repos trong organization, phù hợp với shared credentials.

#### Secrets cần thiết cho ShopLite

```
# Deploy secrets
DEPLOY_SSH_KEY         : Private SSH key để SSH vào server deploy
STAGING_HOST           : IP hoặc hostname của staging server
PROD_HOST              : IP hoặc hostname của production server

# Application secrets
POSTGRES_PASSWORD      : Password của database PostgreSQL
JWT_SECRET             : Secret key để ký JWT tokens
REDIS_PASSWORD         : Password của Redis (nếu có)

# Registry secrets (chỉ cần nếu không dùng GHCR)
DOCKER_USERNAME        : Docker Hub username
DOCKER_PASSWORD        : Docker Hub access token (không dùng password thật)

# Notification secrets (tuỳ chọn)
SLACK_WEBHOOK_URL      : Webhook để gửi thông báo deploy vào Slack
```

**Lưu ý quan trọng**: `GITHUB_TOKEN` là secret đặc biệt được GitHub tự động tạo cho mỗi workflow run. Không cần tạo thủ công, và được tự động revoke sau khi workflow kết thúc.

#### Tạo SSH key cho CI/CD deploy

```bash
# Tạo SSH key pair chuyên dụng cho CI/CD (KHÔNG dùng key cá nhân)
ssh-keygen -t ed25519 \
  -C "github-actions-deploy-$(date +%Y%m%d)" \
  -f ~/.ssh/deploy_key \
  -N ""    # passphrase rỗng vì CI không thể nhập password

# Xem public key (copy lên server)
cat ~/.ssh/deploy_key.pub

# Thêm public key vào authorized_keys trên server
ssh user@server "echo '$(cat ~/.ssh/deploy_key.pub)' >> ~/.ssh/authorized_keys"

# Xem private key (copy vào GitHub Secret DEPLOY_SSH_KEY)
cat ~/.ssh/deploy_key

# Xóa key khỏi máy local sau khi lưu vào GitHub Secrets
rm ~/.ssh/deploy_key ~/.ssh/deploy_key.pub
```

**Hạn chế quyền của deploy key trên server:**

```bash
# Sửa ~/.ssh/authorized_keys, thêm restrictions trước key
# Key chỉ được phép chạy các lệnh trong script deploy, không thể dùng cho mục đích khác
command="/app/shoplite/deploy.sh",no-port-forwarding,no-X11-forwarding,no-agent-forwarding ssh-ed25519 AAAA...
```

#### Cách lưu secrets vào GitHub

1. Mở repository trên GitHub.
2. Vào **Settings** → **Secrets and variables** → **Actions**.
3. Click **New repository secret**.
4. Điền **Name** (ví dụ `DEPLOY_SSH_KEY`) và **Secret** (nội dung private key).
5. Click **Add secret**.

Để thêm environment secrets:
1. Vào **Settings** → **Environments**.
2. Tạo environment (ví dụ `staging`, `production`).
3. Trong environment đó, thêm secrets riêng.

#### Anti-patterns cần tránh

```yaml
# XẤU — lộ secret trong log
- run: echo "${{ secrets.DB_PASSWORD }}"

# XẤU — lưu secret vào file không mã hoá
- run: echo "${{ secrets.SSH_KEY }}" > key.pem

# XẤU — truyền secret qua environment variable với tên rõ ràng trong log
- run: MY_DB_PASS=${{ secrets.DB_PASSWORD }} ./script.sh

# TỐT — GitHub tự mask khi dùng đúng cách
- run: ./deploy.sh
  env:
    DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
```

---

### 6. Docker Layer Caching trong CI

#### Vì sao caching quan trọng

Không có cache: mỗi CI run phải download base image, install dependencies từ đầu → mất 5–10 phút mỗi run.

Với cache: chỉ rebuild các layer thay đổi. Nếu chỉ thay đổi code (không đổi dependencies), build chỉ mất 30–60 giây.

#### GitHub Actions Cache cho Docker Buildx

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: myapp:latest
    cache-from: type=gha           # đọc cache từ GitHub Actions cache
    cache-to: type=gha,mode=max    # ghi cache vào GitHub Actions cache
                                   # mode=max: cache tất cả layers, kể cả intermediate
```

#### Tối ưu Dockerfile để maximize cache hit

Nguyên tắc: đặt các layer ít thay đổi lên trên, thay đổi nhiều xuống dưới.

```dockerfile
# Stage 1: Build (layer ít thay đổi → layer thay đổi nhiều)
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Layer 1: Copy csproj và restore (thay đổi khi dependency thay đổi)
COPY *.csproj ./
RUN dotnet restore

# Layer 2: Copy source code (thay đổi thường xuyên nhất)
COPY . .

# Layer 3: Build và publish
RUN dotnet publish -c Release -o /app/publish --no-restore

# Stage 2: Runtime image (nhỏ hơn SDK ~3x)
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app/publish .

# ASP.NET Core mặc định listen port 8080 từ .NET 8
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080

ENTRYPOINT ["dotnet", "ShopLite.Api.dll"]
```

**Sai lầm phổ biến**: COPY . . trước RUN dotnet restore. Điều này làm cache của restore bị invalidate mỗi khi thay đổi bất kỳ file nào trong source code, dù không thêm package mới.

#### Registry cache (cho production setup)

```yaml
# Cache lưu trên GHCR hoặc Docker Hub — tốt hơn khi có nhiều runners
cache-from: type=registry,ref=ghcr.io/myorg/myapp:buildcache
cache-to: type=registry,ref=ghcr.io/myorg/myapp:buildcache,mode=max
```

---

### 7. Deployment Strategies

#### Recreate (Đơn giản nhất, có downtime)

```bash
# Dừng container cũ, khởi động container mới
docker compose down
docker compose pull
docker compose up -d
```

**Khi dùng**: Development/staging, workload không cần 24/7, batch jobs.

**Nhược điểm**: Có downtime từ lúc down đến lúc service mới ready (thường 10–30 giây).

#### Rolling Update (Zero downtime với multiple replicas)

Dần dần thay thế instances cũ bằng instances mới. Luôn có ít nhất một instance healthy đang chạy.

```yaml
# docker-compose.yml với multiple replicas
services:
  backend:
    image: myapp:latest
    deploy:
      replicas: 3
      update_config:
        parallelism: 1          # update 1 instance tại một thời điểm
        delay: 10s              # chờ 10s giữa mỗi update
        failure_action: rollback
        order: start-first      # khởi động container mới trước khi dừng cái cũ
      rollback_config:
        parallelism: 0          # rollback tất cả cùng lúc
        order: stop-first
```

```bash
# Trigger rolling update
docker compose up -d --no-deps backend
```

**Kubernetes Rolling Update** (tự động):
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0         # không cho phép pod nào unavailable
      maxSurge: 1               # cho phép tạo thêm 1 pod so với desired count
```

#### Blue-Green Deployment

Luôn duy trì 2 môi trường giống hệt nhau: Blue (đang chạy production) và Green (deploy version mới).

```
Internet → Load Balancer → [Blue: v1.0] (đang nhận traffic)
                         → [Green: v1.1] (đang được deploy)
```

**Quy trình:**
1. Deploy version mới lên Green environment.
2. Chạy smoke tests trên Green.
3. Switch load balancer từ Blue sang Green (near-instant, zero downtime).
4. Monitor Green.
5. Nếu vấn đề: switch ngay về Blue (rollback tức thì).
6. Sau khi ổn: Blue trở thành môi trường để deploy version tiếp theo.

```nginx
# nginx upstream switch
upstream backend {
    server green-backend:3000;   # đổi từ blue-backend sang green-backend
}
```

**Ưu điểm**: Rollback tức thì, testing trên môi trường production-identical trước khi switch.

**Nhược điểm**: Chi phí hạ tầng gấp đôi, phức tạp hơn, cần xử lý database migration cẩn thận.

#### Canary Deployment

Route một phần nhỏ traffic (thường 5–10%) sang version mới, theo dõi metrics, sau đó dần tăng tỷ lệ.

```
100% users → Load Balancer → [v1.0] 90% traffic
                           → [v1.1] 10% traffic (canary)
```

**Quy trình:**
1. Deploy v1.1 và route 5% traffic vào.
2. Monitor error rate, latency, business metrics trong 30 phút.
3. Nếu ổn: tăng lên 25%, 50%, 100%.
4. Nếu có vấn đề: route 0% về v1.1, tất cả về v1.0 (rollback).

**Tại sao gọi là "canary"**: Thợ mỏ ngày xưa dùng chim canary để phát hiện khí độc — chim chết trước khi người bị ảnh hưởng. Tương tự, canary deployment phát hiện lỗi trước khi ảnh hưởng toàn bộ users.

```yaml
# Nginx canary routing (thủ công)
upstream backend {
    server v1-backend:3000 weight=90;
    server v2-backend:3000 weight=10;
}
```

**Feature flags** (cách tiếp cận hiện đại hơn): Thay vì route theo server, dùng code logic để enable feature cho % users.

---

### 8. Environment Protection (Production Gates)

#### Cấu hình GitHub Environment với Required Reviewers

1. Vào **Settings** → **Environments** → **New environment** → Đặt tên `production`.
2. Trong section **Environment protection rules**:
   - Tick **Required reviewers**.
   - Thêm GitHub username hoặc team của người approve.
   - Tick **Prevent self-review** để người tạo PR không tự approve.
3. Optionally thêm **Wait timer**: delay N phút sau khi staging deploy trước khi cho phép trigger production job.
4. **Deployment branches**: chỉ cho phép deploy từ nhánh `main`.

#### Flow approval khi có thay đổi lên main

```
1. Developer merge PR vào main
2. CI workflow chạy (lint + test + build)    ← tự động
3. CD workflow: build-and-push               ← tự động
4. CD workflow: deploy-staging               ← tự động
5. CD workflow: run-smoke-tests              ← tự động
6. CD workflow: deploy-production            ← DỪNG, chờ approval
7. Reviewer nhận email/Slack notification
8. Reviewer vào GitHub, review staging, click "Approve and deploy"
9. CD workflow: deploy-production            ← chạy tiếp
```

#### Deployment policies nâng cao

```yaml
# Trong workflow, environment với URL hiển thị trên GitHub
jobs:
  deploy-production:
    environment:
      name: production
      url: https://shoplite.example.com   # hiển thị link trong GitHub UI
```

**Deployment history**: GitHub lưu lại toàn bộ lịch sử deployment, ai approve, lúc mấy giờ — hữu ích cho audit trail.

---

### 9. Rollback Procedure

#### Tại sao cần rollback strategy rõ ràng

Dù CI/CD tốt đến đâu, vẫn có lúc code lên production gây ra vấn đề không phát hiện được qua automated tests (edge cases, load issues, data issues). Phải có quy trình rollback nhanh và được luyện tập thường xuyên.

#### Strategy 1: Rollback bằng cách redeploy image cũ

Đây là cách nhanh nhất. Image cũ vẫn còn trong registry với tag cụ thể.

```bash
# SSH vào production server
ssh ubuntu@prod-server

# Xem các version đang có
docker images | grep shoplite/backend

# Rollback về SHA cụ thể
cd /app/shoplite
cat > docker-compose.override.yml <<EOF
services:
  backend:
    image: ghcr.io/myorg/shoplite/backend:main-abc1234
EOF

docker compose up -d --no-deps backend

# Verify
docker compose ps
curl http://localhost:3000/health
```

#### Strategy 2: Git revert + redeploy

Cách này tạo commit mới reverting lại thay đổi, pipeline sẽ tự động deploy lại.

```bash
# Trên máy local
git revert abc1234..HEAD --no-commit
git commit -m "revert: roll back problematic deploy

Reverts commits from abc1234 to def5678 due to production issue.
See incident report: #123"

git push origin main
# → CI/CD tự động chạy và deploy version đã revert
```

**Ưu điểm của git revert**: Lịch sử git sạch, không mất commit, có thể review lại sau.

#### Strategy 3: Feature flag disable

Nếu dùng feature flags, đây là cách nhanh nhất:
```bash
# Tắt feature flag trong config/feature-flags.json hoặc via API
curl -X POST https://api.launchdarkly.com/api/v2/flags/my-project/new-feature \
  -H "Authorization: $LD_API_TOKEN" \
  -d '{"enabled": false}'
```

#### Checklist rollback

```
[ ] Xác định chính xác commit/version nào gây ra vấn đề
[ ] Thông báo team về incident và rollback plan
[ ] Tắt alerting tạm thời để tránh noise trong quá trình rollback
[ ] Thực hiện rollback
[ ] Verify service trở về bình thường (health check, smoke test)
[ ] Bật lại alerting
[ ] Update incident ticket
[ ] Schedule post-mortem
```

#### Giữ retention policy cho Docker images

```bash
# Giữ ít nhất 10 images gần nhất trong registry
# Xóa images cũ hơn 30 ngày nhưng giữ ít nhất 5 bản

# GitHub Actions workflow chạy weekly để clean up
- name: Delete old images
  uses: actions/delete-package-versions@v5
  with:
    package-name: shoplite/backend
    package-type: container
    min-versions-to-keep: 10
    delete-only-pre-release-versions: false
```

---

### 10. GitLab CI So Sánh

Nếu team dùng GitLab thay GitHub, file `.gitlab-ci.yml` tương tự:

```yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - build
  - deploy-staging
  - deploy-production

variables:
  POSTGRES_DB: shoplite_test
  POSTGRES_USER: shoplite
  POSTGRES_PASSWORD: test
  CONNECTION_STRING: "Host=postgres;Port=5432;Database=${POSTGRES_DB};Username=${POSTGRES_USER};Password=${POSTGRES_PASSWORD}"

# Template tái sử dụng
.dotnet-setup: &dotnet-setup
  image: mcr.microsoft.com/dotnet/sdk:8.0
  cache:
    key:
      files:
        - backend/*.csproj
    paths:
      - backend/.nuget/

lint:
  stage: lint
  <<: *dotnet-setup
  script:
    - cd backend
    - dotnet restore
    - dotnet format --verify-no-changes
    - dotnet build --no-restore -c Release

test:
  stage: test
  <<: *dotnet-setup
  services:
    - name: postgres:15
      alias: postgres
    - name: redis:7-alpine
      alias: redis
  script:
    - cd backend
    - dotnet restore
    - dotnet ef database update
    - dotnet test --no-restore --collect:"XPlat Code Coverage"
  coverage: '/Line coverage[^:]*:\s*([\d.]+)%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: backend/TestResults/**/coverage.cobertura.xml
    expire_in: 1 week
  variables:
    ConnectionStrings__DefaultConnection: $CONNECTION_STRING
    ConnectionStrings__Redis: "redis:6379"
    ASPNETCORE_ENVIRONMENT: Test

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  needs: [lint, test]
  variables:
    IMAGE_TAG: $CI_REGISTRY_IMAGE/backend:$CI_COMMIT_SHORT_SHA
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $IMAGE_TAG ./backend
    - docker push $IMAGE_TAG
    - docker tag $IMAGE_TAG $CI_REGISTRY_IMAGE/backend:latest
    - docker push $CI_REGISTRY_IMAGE/backend:latest

deploy-staging:
  stage: deploy-staging
  image: alpine:latest
  needs: [build]
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$DEPLOY_SSH_KEY" | ssh-add -
    - mkdir -p ~/.ssh
    - ssh-keyscan $STAGING_HOST >> ~/.ssh/known_hosts
  script:
    - ssh ubuntu@$STAGING_HOST "cd /app/shoplite && docker compose pull && docker compose up -d && docker compose exec -T backend dotnet ef database update"
  environment:
    name: staging
    url: https://staging.shoplite.example.com
  only:
    - main

deploy-production:
  stage: deploy-production
  image: alpine:latest
  needs: [deploy-staging]
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$DEPLOY_SSH_KEY" | ssh-add -
    - mkdir -p ~/.ssh
    - ssh-keyscan $PROD_HOST >> ~/.ssh/known_hosts
  script:
    - ssh ubuntu@$PROD_HOST "cd /app/shoplite && docker compose pull && docker compose up -d && docker compose exec -T backend dotnet ef database update"
  environment:
    name: production
    url: https://shoplite.example.com
  when: manual    # yêu cầu click thủ công trong GitLab UI
  only:
    - main
```

**Điểm khác biệt GitLab CI vs GitHub Actions:**

| Tính năng | GitHub Actions | GitLab CI |
|---|---|---|
| Config file | `.github/workflows/*.yml` | `.gitlab-ci.yml` (1 file) |
| Container registry | GHCR (ghcr.io) | GitLab Registry (tích hợp sẵn) |
| Built-in variables | `${{ github.sha }}` | `$CI_COMMIT_SHORT_SHA` |
| Manual gate | Environment with required reviewers | `when: manual` |
| Parallel jobs | `jobs:` cùng cấp | `stage:` cùng tên |
| Services | `services:` trong job | `services:` trong job (giống nhau) |
| Cache | `actions/cache@v4` | `cache:` native |
| Artifacts | `actions/upload-artifact@v4` | `artifacts:` native |
| Self-hosted runner | GitHub Actions Runner | GitLab Runner |
| Pricing | 2000 min/month free | 400 min/month free (SaaS) |

---

### 11. Debugging CI/CD Pipeline

#### Khi workflow không chạy

```bash
# Kiểm tra syntax YAML
# Dùng: https://rhysd.github.io/actionlint/
actionlint .github/workflows/ci.yml

# Hoặc
yamllint .github/workflows/ci.yml
```

Lý do phổ biến workflow không trigger:
- YAML indentation sai.
- Branch name không khớp với filter trong `on.push.branches`.
- Workflow file nằm sai thư mục.
- Secrets chưa được tạo (job fail ngay lập tức).

#### Khi Docker build fail

```yaml
# Thêm --progress=plain để xem đầy đủ output
- name: Build image
  uses: docker/build-push-action@v5
  with:
    context: .
    push: false
    build-args: BUILDKIT_INLINE_CACHE=1

# Hoặc build thủ công để debug
- name: Build image (debug mode)
  run: |
    DOCKER_BUILDKIT=1 docker build \
      --progress=plain \
      --no-cache \
      -t myapp:debug \
      ./backend
```

#### Enable debug logging

```bash
# Trong GitHub Secrets, thêm:
ACTIONS_STEP_DEBUG = true
ACTIONS_RUNNER_DEBUG = true

# Hoặc khi re-run workflow, chọn "Enable debug logging"
```

#### Tái hiện lỗi CI trên local

```bash
# Dùng act để chạy GitHub Actions trên local
brew install act   # macOS
# hoặc
curl https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash

# Chạy workflow ci.yml
act push -W .github/workflows/ci.yml

# Chạy job cụ thể
act push -j test -W .github/workflows/ci.yml

# Với secrets
act push -j test \
  --secret DATABASE_URL=postgresql://localhost/test \
  -W .github/workflows/ci.yml
```

---

### 12. Security Best Practices trong CI/CD

#### Principle of Least Privilege

```yaml
# Giới hạn permissions của GITHUB_TOKEN
permissions:
  contents: read       # đọc code
  packages: write      # push Docker image
  deployments: write   # update deployment status
  # KHÔNG cần: issues, pull-requests, admin
```

#### Không trust input từ event payload

```yaml
# NGUY HIỂM — injection attack nếu PR title chứa malicious code
- run: echo "${{ github.event.pull_request.title }}"

# AN TOÀN — dùng environment variable
- run: echo "$PR_TITLE"
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
```

#### Dependency review

```yaml
# Tự động scan dependencies mới trong PR
name: Dependency Review
on: pull_request

jobs:
  dependency-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: high
```

#### Pin third-party actions bằng SHA

```yaml
# XẤU — có thể bị thay đổi bởi maintainer
uses: some-org/some-action@v1

# TỐT — pin bằng commit SHA cụ thể
uses: some-org/some-action@a1b2c3d4e5f6789012345678901234567890abcd
```

---

## Kỹ năng đạt được
- Xây dựng pipeline CI/CD từ đầu.
- Tích hợp test và build image vào pipeline.
- Tự động deploy lên server.

---

## Thực hành

**Môi trường:** GitHub Actions (hoặc GitLab CI) + server đích.

**Lab:**
- Viết workflow CI: mỗi push chạy lint + test.
- Viết workflow CD: build image → push lên registry → deploy lên server bằng SSH/compose.
- Lưu secrets (SSH key, DB password) trong GitHub Secrets.
- Tách pipeline cho nhánh `develop` (staging) và `main` (prod).

**Công cụ:** GitHub Actions / GitLab CI, registry.

---

## Bài tập

- **Bắt buộc:** Push lên `main` → tự động build, push image, deploy lên server, không thao tác tay.
- **Nâng cao:** Thêm bước test bắt buộc; nếu test fail thì chặn deploy.
- **Mô phỏng doanh nghiệp:** Thiết lập approval thủ công trước khi deploy lên prod (môi trường production protection).

---

## Deliverable
- [ ] File pipeline `.github/workflows/*.yml` (hoặc `.gitlab-ci.yml`).
- [ ] Quy trình deploy tự động hoạt động.
- [ ] Tài liệu mô tả pipeline (sơ đồ các stage).

---

## Tiêu chí hoàn thành
- [ ] Một commit lên `main` tự động đưa thay đổi lên server đang chạy.
- [ ] Test fail thì pipeline dừng, không deploy.
- [ ] Secrets không bao giờ lộ trong log/code.

---

## Ghi chú học tập

> _Ghi lại những điều học được, vướng mắc và insight trong quá trình học phase này._
