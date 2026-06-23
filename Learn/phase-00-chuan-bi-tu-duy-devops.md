# Phase 0 — Chuẩn bị & Tư duy DevOps

## Mục tiêu
- Hiểu DevOps thực chất là gì, giải quyết vấn đề gì trong doanh nghiệp (không chỉ là "biết dùng tool").
- Hiểu vòng đời phần mềm (SDLC) và DevOps nằm ở đâu trong đó.
- Có một môi trường làm việc đủ mạnh để học suốt lộ trình.
- Nhận source code ứng dụng mẫu (ShopLite) và chạy được nó ở mức cơ bản.

---

## Kiến thức sẽ học

- **DevOps là gì & văn hóa DevOps** — vì sao Dev và Ops phải hợp nhất; khái niệm CALMS (Culture, Automation, Lean, Measurement, Sharing).
- **Vòng đời phát triển phần mềm (SDLC)** và **DevOps lifecycle** (Plan → Code → Build → Test → Release → Deploy → Operate → Monitor).
- **Mô hình kiến trúc ứng dụng** (monolith vs 3-tier vs microservices) — để hiểu ShopLite được thiết kế thế nào.
- **Lựa chọn môi trường làm việc:** WSL2 (Windows), máy ảo (VirtualBox/VMware), hoặc dual-boot/Linux native.

---

## Kiến thức chi tiết

### 1. DevOps là gì — Câu chuyện thực tế

#### 1.1 Trước DevOps: Bức tường vô hình giữa Dev và Ops

Để hiểu tại sao DevOps ra đời, ta cần nhìn lại cách các công ty phần mềm vận hành trước năm 2010. Hãy tưởng tượng một công ty thương mại điện tử trung bình với 50 kỹ sư.

**Cơ cấu tổ chức kiểu cũ:**

```
Phòng Phát triển (Development)       Phòng Vận hành (Operations)
─────────────────────────────         ─────────────────────────────
- Lập trình viên frontend             - Quản trị hệ thống (sysadmin)
- Lập trình viên backend              - Kỹ sư mạng (network engineer)
- QA testers                          - DBA (database admin)
- Tech lead                           - Security team
```

Hai phòng này không chỉ ngồi khác tầng — họ có KPI hoàn toàn đối lập nhau:

- **Dev được thưởng** khi ship nhiều tính năng mới, nhanh chóng.
- **Ops được thưởng** khi hệ thống ổn định, không có sự cố.

Kết quả? Mỗi lần Dev muốn deploy là một cuộc chiến ngầm.

#### 1.2 Quy trình deploy kiểu cũ — "Deployment Friday"

Dưới đây là một quy trình deploy điển hình trước thời DevOps:

```
Tuần 1-12: Dev code tính năng mới
     ↓
Tuần 13: Dev hoàn thành, viết ticket gửi Ops
     ↓
Ops review: "Tài liệu đâu? Specs đâu? Sao không báo sớm hơn?"
     ↓
Tuần 14: Ops chuẩn bị môi trường production (thủ công)
     ↓
Thứ Sáu 23:00: Deploy (vì sợ ảnh hưởng giờ hành chính)
     ↓
00:30 Thứ Bảy: Production sập. Lỗi cấu hình environment
     ↓
Cuộc gọi khẩn: Ops gọi Dev. Dev gọi Ops. Ai chịu trách nhiệm?
     ↓
03:00 Thứ Bảy: Rollback về version cũ
     ↓
Post-mortem: "Lỗi của Dev vì code sai / Lỗi của Ops vì setup sai"
```

Đây không phải câu chuyện tưởng tượng. Năm 2009, John Allspaw và Paul Hammond từ Flickr trình bày tại O'Reilly Velocity Conference về việc họ đã làm **10 deploys mỗi ngày** — điều mà cả ngành coi là không thể. Bài trình bày đó đã châm ngòi cho phong trào DevOps.

#### 1.3 Bốn vấn đề cốt lõi của mô hình Dev-Ops tách biệt

**Vấn đề 1: Deployment sợ hãi (Fear-based Deployment)**

Khi deploy chỉ xảy ra 1-2 lần mỗi quý, mỗi lần deploy là một sự kiện rủi ro cao. Giống như không leo núi suốt 6 tháng rồi đột ngột leo Everest. Ironically, càng ít deploy thì mỗi lần deploy càng nguy hiểm vì:
- Thay đổi tích lũy quá nhiều
- Không có feedback loop nhanh
- Team mất kỹ năng thao tác

**Vấn đề 2: Blame Game (Đổ lỗi)**

Khi sự cố xảy ra, câu hỏi đầu tiên không phải "Sao xảy ra?" mà là "Ai làm điều này?". Văn hóa đổ lỗi dẫn đến:
- Kỹ sư che giấu sai lầm thay vì học từ chúng
- Không ai dám thử nghiệm cái mới
- Mọi người làm CYA (Cover Your Ass) — ghi chép để tự bảo vệ, không để chia sẻ

**Vấn đề 3: Silo Effect (Hiệu ứng Silo)**

Kiến thức bị giữ trong từng bộ phận:
- Ops biết production environment nhưng không biết code logic
- Dev biết code nhưng không biết production chạy thế nào
- Khi Dev viết code, họ test trên máy cá nhân (localhost) — hoàn toàn khác production

Câu cửa miệng nổi tiếng nhất trong lịch sử phần mềm: **"It works on my machine!"**

**Vấn đề 4: Handoff Waste (Lãng phí do bàn giao)**

Mỗi lần code chuyển từ Dev sang QA sang Ops là một điểm chờ đợi (wait time). Trong Lean Manufacturing, đây gọi là "waste" — không tạo ra giá trị. Một tính năng có thể mất 3 tháng để đến tay người dùng, trong khi thời gian code thực tế chỉ là 2 tuần.

#### 1.4 Sau DevOps: Thay đổi gì?

DevOps không phải là một công cụ hay một chức danh. Đó là một **văn hóa và tập hợp thực hành** nhằm phá vỡ bức tường giữa Dev và Ops.

Sau khi áp dụng DevOps:

```
TRƯỚC DevOps                          SAU DevOps
─────────────────────────             ─────────────────────────
Deploy 1 lần/quý                  →   Deploy hàng ngày/hàng giờ
Dev và Ops không giao tiếp        →   Cùng một team, cùng on-call
Môi trường khác nhau              →   Infrastructure as Code
Test thủ công                     →   Automated testing pipeline
Rollback = thủ công, mất giờ      →   Rollback tự động trong phút
Post-mortem = blame                →   Blameless post-mortem
Monitoring = Ops job               →   "You build it, you run it"
```

Amazon — một trong những công ty áp dụng DevOps sớm nhất — đạt được tỷ lệ **deploy mỗi 11.6 giây** vào năm 2011. Đây là kết quả của văn hóa "you build it, you run it" do Werner Vogels đề xuất.

---

### 2. CALMS Framework — Nền tảng triết học của DevOps

CALMS là framework được Nicole Forsgren, Jez Humble và Gene Kim (tác giả cuốn The DevOps Handbook) phát triển để mô tả các trụ cột của một tổ chức DevOps trưởng thành.

#### 2.1 C — Culture (Văn hóa)

Văn hóa là trụ cột khó thay đổi nhất nhưng quan trọng nhất. Không có văn hóa đúng, công cụ tốt nhất cũng vô dụng.

**Blameless Postmortem (Phân tích sự cố không đổ lỗi)**

Đây là một trong những thực hành văn hóa quan trọng nhất. Khi xảy ra sự cố, thay vì tìm người có lỗi, team thực hiện:

```markdown
## Blameless Postmortem Template

### Tóm tắt sự cố
- Thời gian xảy ra: [timestamp]
- Thời gian phát hiện: [timestamp]
- Thời gian khắc phục: [timestamp]
- Mức độ ảnh hưởng: [số người dùng bị ảnh hưởng]

### Timeline sự kiện
- 14:32 - Deploy version 2.3.1 lên production
- 14:45 - Monitoring alert: Error rate tăng từ 0.1% lên 15%
- 14:47 - On-call engineer nhận PagerDuty alert
- 14:55 - Xác định nguyên nhân: database connection pool exhausted
- 15:10 - Rollback về version 2.3.0
- 15:15 - Error rate trở về bình thường

### Nguyên nhân gốc rễ (Root Cause)
[Mô tả kỹ thuật, không đề cập tên người]

### 5 Whys Analysis
Why 1: Tại sao service bị lỗi?
→ Connection pool to database bị exhausted

Why 2: Tại sao connection pool bị exhausted?
→ Query mới trong version 2.3.1 không release connection đúng cách

Why 3: Tại sao query không release connection?
→ Missing finally block trong connection handling code

Why 4: Tại sao lỗi này không được phát hiện trong testing?
→ Integration test chưa cover scenario này

Why 5: Tại sao integration test chưa cover?
→ Chưa có test coverage requirement cho database connection lifecycle

### Action Items
- [ ] Fix connection handling code
- [ ] Add integration test cho connection lifecycle
- [ ] Review toàn bộ database queries tương tự
- [ ] Thêm connection pool metrics vào dashboard
```

Triết lý cốt lõi: **Con người không sai — hệ thống mới sai**. Kỹ sư làm điều tốt nhất họ có thể với thông tin họ có lúc đó. Nếu họ ra quyết định sai, đó là vì hệ thống không cung cấp đủ thông tin hoặc không có safeguard.

**Psychological Safety (An toàn tâm lý)**

Google's Project Aristotle (2012-2015) nghiên cứu hàng trăm team và phát hiện: yếu tố quan trọng nhất của một team hiệu quả không phải là tài năng cá nhân, mà là **psychological safety** — cảm giác an toàn khi nói ra sai lầm, ý kiến trái chiều, hoặc câu hỏi ngây thơ mà không sợ bị phán xét.

#### 2.2 A — Automation (Tự động hóa)

Automation trong DevOps có nghĩa là: **bất cứ điều gì làm thủ công hơn một lần, hãy tự động hóa nó**.

**CI/CD Pipeline — Backbone của Automation**

```
Developer push code
        ↓
   Git Repository (GitHub/GitLab)
        ↓
   CI Server trigger (GitHub Actions/Jenkins/GitLab CI)
        ↓
   ┌─────────────────────────────────────┐
   │         CI Pipeline                 │
   │  1. Checkout code                   │
   │  2. Install dependencies            │
   │  3. Run linting (ESLint/Pylint)     │
   │  4. Run unit tests                  │
   │  5. Run integration tests           │
   │  6. Build Docker image              │
   │  7. Push to Container Registry      │
   │  8. Security scan (Trivy/Snyk)      │
   └─────────────────────────────────────┘
        ↓
   CD Pipeline (deploy to staging)
        ↓
   Automated smoke tests
        ↓
   Manual approval gate (nếu cần)
        ↓
   Deploy to production
        ↓
   Post-deploy health checks
```

**Infrastructure as Code (IaC)**

IaC có nghĩa là mô tả infrastructure bằng code, không phải bằng click chuột trên UI.

```hcl
# Ví dụ Terraform tạo một EC2 instance
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"  # Ubuntu 22.04
  instance_type = "t3.medium"

  tags = {
    Name        = "shoplite-web"
    Environment = "production"
    Team        = "platform"
  }

  vpc_security_group_ids = [aws_security_group.web.id]
  subnet_id              = aws_subnet.public.id
}
```

Lợi ích của IaC:
- **Reproducible**: Tạo lại environment y hệt trong vài phút
- **Version controlled**: Xem ai thay đổi gì, khi nào
- **Self-documenting**: Code là documentation
- **Drift detection**: Phát hiện khi thực tế khác với code

#### 2.3 L — Lean (Tinh gọn)

Lean trong DevOps được lấy cảm hứng từ Toyota Production System. Mục tiêu là **loại bỏ lãng phí** và **tối ưu hóa flow** của công việc từ ý tưởng đến tay người dùng.

**8 loại lãng phí trong phát triển phần mềm (Lean Software):**

| Loại lãng phí | Ví dụ trong phần mềm |
|---------------|---------------------|
| Overproduction | Code tính năng chưa cần |
| Waiting | Chờ approval, chờ test, chờ deploy |
| Transportation | Handoff giữa teams |
| Over-processing | Review meetings không cần thiết |
| Inventory | Backlog quá lớn, WIP quá nhiều |
| Motion | Context switching giữa nhiều tasks |
| Defects | Bugs phát hiện muộn |
| Underutilized talent | Senior developer làm việc junior |

**Value Stream Mapping**

Value Stream Map là công cụ visualize toàn bộ quá trình từ khi có ý tưởng đến khi feature lên production:

```
Ý tưởng → Backlog → In Progress → Code Review → QA → Staging → Production
  [1d]      [5d]       [3d]          [2d]        [3d]   [1d]       [0.5d]

Total Lead Time: ~15.5 ngày
Value-added time: 3 ngày (code)
Waste: 12.5 ngày (chờ đợi)
```

Mục tiêu của Lean là giảm lead time từ 15.5 ngày xuống còn 1-2 ngày bằng cách loại bỏ các điểm chờ đợi.

**WIP Limits (Work In Progress)**

Nguyên tắc: **Làm ít thứ hơn, hoàn thành nhanh hơn**. Giới hạn số lượng việc đang làm đồng thời giúp phát hiện bottleneck và tăng throughput.

#### 2.4 M — Measurement (Đo lường)

"Bạn không thể cải thiện điều bạn không đo lường." — Peter Drucker

**DORA Metrics — Bốn chỉ số vàng của DevOps**

DORA (DevOps Research and Assessment) là chương trình nghiên cứu kéo dài nhiều năm, khảo sát hơn 32,000 kỹ sư toàn cầu. Họ phát hiện 4 chỉ số có tương quan mạnh nhất với hiệu suất tổ chức:

**Chỉ số 1: Deployment Frequency (Tần suất deploy)**

```
Đo lường: Bao nhiêu lần mỗi ngày/tuần/tháng bạn deploy lên production?

Elite performers:    Multiple times per day (nhiều lần mỗi ngày)
High performers:     Once per day to once per week
Medium performers:   Once per week to once per month
Low performers:      Once per month to once per six months

Ý nghĩa: Deployment frequency cao = batch size nhỏ = rủi ro thấp hơn
```

**Chỉ số 2: Lead Time for Changes (Thời gian từ commit đến production)**

```
Đo lường: Từ khi developer commit code đến khi code đó chạy trên production mất bao lâu?

Elite performers:    < 1 giờ
High performers:     1 ngày đến 1 tuần
Medium performers:   1 tuần đến 1 tháng
Low performers:      1 tháng đến 6 tháng

Ý nghĩa: Lead time ngắn = feedback nhanh = phát hiện lỗi sớm
```

**Chỉ số 3: Mean Time to Recovery / MTTR (Thời gian khôi phục trung bình)**

```
Đo lường: Khi production bị sự cố, mất bao lâu để khôi phục về trạng thái bình thường?

Elite performers:    < 1 giờ
High performers:     < 1 ngày
Medium performers:   1 ngày đến 1 tuần
Low performers:      > 1 tuần

Ý nghĩa: MTTR thấp = team có khả năng detect và fix nhanh
```

**Chỉ số 4: Change Failure Rate (Tỷ lệ thay đổi gây lỗi)**

```
Đo lường: Bao nhiêu phần trăm deploy lên production gây ra sự cố cần rollback/hotfix?

Elite performers:    0% - 15%
High performers:     16% - 30%
Medium performers:   16% - 30%
Low performers:      16% - 30%

Ý nghĩa: Change failure rate thấp = quy trình testing và review tốt
```

**Cách theo dõi DORA metrics trong thực tế:**

```yaml
# Ví dụ GitHub Actions workflow tracking deployment frequency
name: Track Deployment

on:
  deployment_status:
    types: [success]

jobs:
  track-metrics:
    runs-on: ubuntu-latest
    steps:
      - name: Record deployment
        run: |
          # Ghi vào database hoặc monitoring system
          curl -X POST https://metrics.company.com/deployments \
            -H "Content-Type: application/json" \
            -d '{
              "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
              "environment": "production",
              "service": "${{ github.repository }}",
              "commit": "${{ github.sha }}"
            }'
```

#### 2.5 S — Sharing (Chia sẻ)

**Knowledge Sharing (Chia sẻ kiến thức)**

Một team DevOps khỏe mạnh không có "bus factor" cao — tức là không phụ thuộc vào một người duy nhất biết cách vận hành hệ thống.

Các hình thức sharing phổ biến:
- **Runbooks**: Tài liệu step-by-step để xử lý tình huống cụ thể
- **Architecture Decision Records (ADR)**: Ghi lại quyết định kỹ thuật và lý do
- **Internal tech talks**: Chia sẻ học hỏi trong team
- **Pair programming / mob programming**: Code cùng nhau
- **On-call rotation**: Mọi kỹ sư đều on-call, mọi người đều hiểu production

**Runbook mẫu:**

```markdown
## Runbook: Xử lý khi Database Connection Pool Exhausted

### Triệu chứng
- Alert: "DB connection pool utilization > 90%"
- Logs: "Error: too many connections" trong application logs
- User-facing: Trang web chậm hoặc lỗi 500

### Bước 1: Xác nhận sự cố
```bash
# Kiểm tra số connection hiện tại
psql -h prod-db.internal -U postgres -c "
  SELECT count(*), state
  FROM pg_stat_activity
  GROUP BY state;
"
```

### Bước 2: Giảm tải tạm thời
```bash
# Scale up số pod để phân tán connection
kubectl scale deployment shoplite-backend --replicas=5
```

### Bước 3: Tìm nguyên nhân
```bash
# Xem query nào đang chạy lâu
psql -c "SELECT pid, query, state, wait_event_type, wait_event
         FROM pg_stat_activity
         WHERE state != 'idle'
         ORDER BY query_start;"
```

### Escalation
Nếu không giải quyết được trong 15 phút → Gọi cho Database Admin on-call
```
```

---

### 3. DevOps Lifecycle — Từng bước với công cụ thực tế

DevOps lifecycle là một vòng tròn liên tục, không bao giờ kết thúc. Mỗi iteration là một cơ hội cải thiện.

#### 3.1 Plan (Lập kế hoạch)

**Mục đích:** Xác định việc cần làm, ưu tiên, và phân công.

**Công cụ phổ biến:**

| Công cụ | Phù hợp với |
|---------|-------------|
| Jira | Enterprise, Scrum/Kanban |
| Linear | Startup, modern UX |
| GitHub Issues | Small team, open source |
| Trello | Kanban đơn giản |
| Notion | Kết hợp docs + tasks |

**Quy trình Agile Sprint trong DevOps:**

```
Sprint Planning (Thứ Hai)
    └── Chọn stories từ backlog
    └── Estimate story points
    └── Commit sprint goal

Daily Standup (Mỗi sáng, 15 phút)
    └── Yesterday: Đã làm gì?
    └── Today: Sẽ làm gì?
    └── Blockers: Có bị chặn không?

Sprint Review (Thứ Sáu, 2 tuần)
    └── Demo tính năng mới
    └── Stakeholder feedback

Sprint Retrospective (Sau Review)
    └── What went well?
    └── What could improve?
    └── Action items
```

**Định nghĩa "Definition of Done" trong DevOps:**

Không chỉ "code xong là done". Một story thực sự "done" khi:
- Code được review và merge
- Unit tests viết và pass
- Integration tests pass
- Đã deploy lên staging
- QA verify trên staging
- Documentation cập nhật
- Monitoring/alerts được cấu hình
- Đã deploy lên production
- Post-deploy health check pass

#### 3.2 Code (Viết code)

**Công cụ:** Git, GitHub/GitLab, VS Code, pre-commit hooks

**Git Flow vs Trunk-based Development:**

```
Git Flow (phức tạp, phù hợp release cycle dài):
main
  └── develop
        ├── feature/user-auth
        ├── feature/payment
        └── release/v2.0
              └── hotfix/critical-bug

Trunk-based Development (đơn giản, phù hợp CI/CD):
main (trunk)
  ├── feature/user-auth  [tồn tại < 2 ngày]
  └── feature/payment    [tồn tại < 2 ngày]
```

High-performing DevOps teams hầu hết dùng **trunk-based development** vì:
- Giảm merge conflict
- Feedback nhanh hơn
- Buộc phải deploy nhỏ, thường xuyên

**Conventional Commits — Quy ước viết commit message:**

```bash
# Format: <type>(<scope>): <description>
feat(auth): add JWT refresh token mechanism
fix(payment): handle timeout when Stripe API is slow
docs(readme): update installation instructions
test(cart): add unit tests for discount calculation
refactor(api): extract common validation middleware
perf(db): add index on orders.created_at column
ci(pipeline): add security scanning step
chore(deps): upgrade express from 4.18 to 4.19
```

**Pre-commit hooks (kiểm tra trước khi commit):**

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json

  - repo: local
    hooks:
      - id: dotnet-format
        name: dotnet format
        entry: dotnet format --verify-no-changes
        language: system
        pass_filenames: false

      - id: dotnet-build
        name: dotnet build
        entry: dotnet build --no-restore -warnaserror
        language: system
        pass_filenames: false
```

#### 3.3 Build (Xây dựng)

**Công cụ:** Docker, dotnet CLI, MSBuild, GitHub Actions

**Docker build process:**

```dockerfile
# Dockerfile cho ShopLite Backend (.NET 8)
# Multi-stage build để giảm image size

# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS builder
WORKDIR /app
COPY *.csproj ./
RUN dotnet restore

COPY . .
RUN dotnet publish -c Release -o /app/publish

# Stage 2: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=builder /app/publish .

# Chạy với non-root user (security best practice)
RUN addgroup -S appgroup && adduser --no-create-home -S appuser -G appgroup
USER appuser

EXPOSE 8080
CMD ["dotnet", "ShopLite.dll"]
```

**Build artifacts và versioning:**

```bash
# Semantic versioning: MAJOR.MINOR.PATCH
# MAJOR: breaking changes
# MINOR: new features, backward compatible
# PATCH: bug fixes

# Ví dụ tag Docker image
docker build -t shoplite-backend:1.2.3 .
docker build -t shoplite-backend:1.2.3-$(git rev-parse --short HEAD) .
# → shoplite-backend:1.2.3-a3f9b2c
```

#### 3.4 Test (Kiểm thử)

**Công cụ:** xUnit, NUnit, Playwright, k6, OWASP ZAP

**Testing Pyramid — Chiến lược test:**

```
         /\
        /  \
       / E2E \     ← Ít nhất (chậm, đắt, brittle)
      /  Tests \   Cypress, Playwright
     /──────────\
    /Integration \  ← Vừa phải
   /    Tests    \  Supertest, pytest
  /────────────────\
 /    Unit Tests    \ ← Nhiều nhất (nhanh, rẻ, reliable)
/____________________\ Jest, pytest, go test
```

**Ví dụ Unit Test cho API endpoint:**

```csharp
// tests/unit/CartTests.cs
using Xunit;
using ShopLite.Utils;

public class CartTests
{
    [Fact]
    public void CalculateDiscount_OrderOver500k_Returns10Percent()
    {
        var result = CartUtils.CalculateDiscount(600000);
        Assert.Equal(60000, result);
    }

    [Fact]
    public void CalculateDiscount_OrderUnder500k_ReturnsZero()
    {
        var result = CartUtils.CalculateDiscount(400000);
        Assert.Equal(0, result);
    }

    [Fact]
    public void CalculateDiscount_NegativeAmount_ThrowsArgumentException()
    {
        Assert.Throws<ArgumentException>(
            () => CartUtils.CalculateDiscount(-100));
    }
}
```

**Ví dụ Integration Test:**

```csharp
// tests/integration/OrderApiTests.cs
using System.Net;
using System.Net.Http.Json;
using Microsoft.AspNetCore.Mvc.Testing;
using Xunit;

public class OrderApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public OrderApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task CreateOrder_ValidRequest_Returns201()
    {
        var response = await _client.PostAsJsonAsync("/api/orders", new
        {
            UserId = 1,
            Items = new[] { new { ProductId = 5, Quantity = 2 } },
            PaymentMethod = "credit_card"
        });

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        var data = await response.Content.ReadFromJsonAsync<OrderResponse>();
        Assert.NotNull(data?.OrderId);
        Assert.Equal("pending", data?.Status);
    }
}
```

**Load Testing với k6:**

```javascript
// k6/load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // Ramp up to 100 users
    { duration: '5m', target: 100 },   // Stay at 100 users
    { duration: '2m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% requests < 500ms
    http_req_failed: ['rate<0.01'],    // Error rate < 1%
  },
};

export default function () {
  const response = http.get('https://shoplite.staging.com/api/products');
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 200ms': (r) => r.timings.duration < 200,
  });
  sleep(1);
}
```

#### 3.5 Release (Phát hành)

**Công cụ:** GitHub Releases, JFrog Artifactory, AWS ECR, Docker Hub, Harbor

**Artifact Registry — Nơi lưu trữ build artifacts:**

```bash
# Push Docker image lên AWS ECR
aws ecr get-login-password --region ap-southeast-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.ap-southeast-1.amazonaws.com

docker tag shoplite-backend:1.2.3 \
  123456789.dkr.ecr.ap-southeast-1.amazonaws.com/shoplite-backend:1.2.3

docker push \
  123456789.dkr.ecr.ap-southeast-1.amazonaws.com/shoplite-backend:1.2.3
```

**Release strategies:**

```
Blue-Green Deployment:
┌──────────┐     ┌──────────┐
│  Blue    │     │  Green   │
│  v1.2.2  │     │  v1.2.3  │
│ (active) │     │ (new)    │
└──────────┘     └──────────┘
       ↑                ↑
   100% traffic    0% traffic
   
Sau khi test Green OK → chuyển 100% traffic sang Green

Canary Deployment:
v1.2.3 ──── 5% traffic  ─────────────┐
v1.2.2 ──── 95% traffic ──────────────┤── Load Balancer ──── Users
Theo dõi metrics, nếu OK → tăng dần lên 100%

Feature Flags:
if (featureFlag.isEnabled('new-checkout-flow', userId)) {
    return newCheckoutFlow();
} else {
    return oldCheckoutFlow();
}
```

#### 3.6 Deploy (Triển khai)

**Công cụ:** Kubernetes, Helm, ArgoCD, AWS CodeDeploy, Ansible

**GitOps với ArgoCD:**

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shoplite-backend
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/shoplite-infra
    targetRevision: HEAD
    path: k8s/backend
  destination:
    server: https://kubernetes.default.svc
    namespace: shoplite-prod
  syncPolicy:
    automated:
      prune: true      # Xóa resources không còn trong Git
      selfHeal: true   # Tự sửa nếu ai đó manual change trên cluster
```

#### 3.7 Operate (Vận hành)

**Công cụ:** Kubernetes, systemd, supervisor, PM2

**Health checks và readiness probes:**

```yaml
# kubernetes deployment với health checks
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shoplite-backend
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: backend
          image: shoplite-backend:1.2.3
          ports:
            - containerPort: 8080
          # Kiểm tra app có ready nhận traffic chưa
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          # Kiểm tra app có còn sống không
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

#### 3.8 Monitor (Giám sát)

**Công cụ:** Prometheus, Grafana, ELK Stack, Datadog, New Relic, PagerDuty

**Ba trụ cột của Observability:**

```
Metrics (Số liệu)
├── CPU usage, memory usage
├── Request rate, error rate, latency
├── Business metrics: orders/minute, revenue/hour
└── Tool: Prometheus + Grafana

Logs (Nhật ký)
├── Application logs (structured JSON)
├── Access logs
├── Error logs với stack trace
└── Tool: ELK Stack (Elasticsearch + Logstash + Kibana)
       hoặc Loki + Grafana

Traces (Vết thực thi)
├── Distributed tracing: request đi qua bao nhiêu services?
├── Latency breakdown: service nào chậm nhất?
└── Tool: Jaeger, Zipkin, Tempo
```

**Structured Logging (Logging có cấu trúc):**

```json
// Thay vì: console.log("User 123 logged in at 14:32")
// Dùng structured JSON:
{
  "timestamp": "2024-01-15T14:32:00.123Z",
  "level": "info",
  "service": "shoplite-backend",
  "version": "1.2.3",
  "event": "user.login",
  "user_id": 123,
  "ip": "192.168.1.100",
  "duration_ms": 45,
  "trace_id": "abc123def456"
}
```

**Alerting — Cảnh báo thông minh:**

```yaml
# prometheus-alerts.yaml
groups:
  - name: shoplite.rules
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Error rate cao bất thường"
          description: "Error rate: {{ $value | humanizePercentage }}"
          runbook_url: "https://wiki.company.com/runbooks/high-error-rate"
```

---

### 4. Architecture Patterns — So sánh mô hình kiến trúc

#### 4.1 Monolith Architecture

**Định nghĩa:** Toàn bộ ứng dụng được đóng gói và deploy như một đơn vị duy nhất.

```
┌─────────────────────────────────────────┐
│           Monolith Application          │
│  ┌──────────┐  ┌──────────┐  ┌───────┐ │
│  │   UI     │  │ Business │  │  DB   │ │
│  │  Layer   │  │  Logic   │  │ Layer │ │
│  └──────────┘  └──────────┘  └───────┘ │
│   /products    │ validate()  │ SQL     │ │
│   /orders      │ calculate() │ queries │ │
│   /users       │ notify()    │         │ │
└─────────────────────────────────────────┘
          Deploy as one unit
                  ↓
           single process
```

**Ưu điểm:**
- Đơn giản để phát triển ban đầu
- Dễ debug: toàn bộ code trong một process
- Không có network latency giữa components
- Transaction đơn giản (ACID trong một database)
- Ít operational overhead

**Nhược điểm:**
- Khó scale: phải scale toàn bộ app kể cả khi chỉ một phần bận
- Deploy rủi ro cao: một thay đổi nhỏ đòi hỏi deploy toàn bộ
- Codebase phình to theo thời gian, khó maintain
- Công nghệ bị lock: không thể dùng Python cho một module và Go cho module khác
- Một module lỗi có thể kéo sập toàn bộ app

**Khi nào nên dùng Monolith:**
- Startup giai đoạn đầu (MVP)
- Team nhỏ (< 5 developers)
- Domain chưa được hiểu rõ
- Không chắc về scale requirements

#### 4.2 3-Tier Architecture (Kiến trúc 3 tầng)

**Định nghĩa:** Phân tách ứng dụng thành 3 tầng độc lập về mặt logic (và thường cả vật lý).

```
┌─────────────────────────────────────────────┐
│  Tier 1: Presentation Layer (Frontend)      │
│  React SPA, Angular, Vue                    │
│  Chạy trên browser hoặc CDN                 │
└───────────────────┬─────────────────────────┘
                    │ HTTP/REST API
┌───────────────────▼─────────────────────────┐
│  Tier 2: Application/Logic Layer (Backend)  │
│  ASP.NET Core (.NET 8), C#                  │
│  Business rules, validation, orchestration  │
└───────────────────┬─────────────────────────┘
                    │ SQL / ORM
┌───────────────────▼─────────────────────────┐
│  Tier 3: Data Layer (Database)              │
│  PostgreSQL, MySQL, MongoDB                 │
│  Persistent storage                         │
└─────────────────────────────────────────────┘
```

**Ưu điểm:**
- Separation of concerns rõ ràng
- Từng tier có thể scale độc lập (scale backend mà không cần scale database)
- Dễ dàng thay thế từng tier (đổi frontend framework không ảnh hưởng backend)
- Security: database không exposed trực tiếp ra internet
- Phù hợp cho hầu hết ứng dụng web truyền thống

**Nhược điểm:**
- Phức tạp hơn monolith
- Network round-trip giữa tiers
- Cần quản lý nhiều services hơn

#### 4.3 Microservices Architecture

**Định nghĩa:** Ứng dụng được chia thành nhiều services nhỏ, mỗi service làm một việc, deploy độc lập, giao tiếp qua network.

```
           API Gateway
               │
    ┌──────────┼──────────┐
    │          │          │
┌───▼──┐  ┌───▼──┐  ┌───▼──┐
│User  │  │Order │  │Prod- │
│Svc   │  │Svc   │  │uct   │
│:3001 │  │:3002 │  │Svc   │
└──┬───┘  └──┬───┘  │:3003 │
   │ DB1      │ DB2  └──┬───┘
   │          │         │ DB3
┌──▼──┐  ┌───▼─┐   ┌───▼──┐
│user │  │order│   │prod  │
│ DB  │  │ DB  │   │ DB   │
└─────┘  └─────┘   └──────┘
```

**Ưu điểm:**
- Scale từng service độc lập (scale Order service trong Black Friday)
- Fault isolation: một service chết không kéo sập service khác (nếu thiết kế tốt)
- Technology flexibility: mỗi service dùng tech stack phù hợp
- Team autonomy: mỗi team own một service
- Easier to understand từng service nhỏ

**Nhược điểm:**
- Phức tạp về operations: cần quản lý N services thay vì 1
- Network overhead và latency
- Distributed transactions khó (không có ACID đơn giản)
- Debugging khó hơn: request có thể đi qua 5-10 services
- Cần infrastructure mạnh (Kubernetes, service mesh)
- "Microservices tax": overhead ban đầu rất cao

**Khi nào nên dùng Microservices:**
- Team lớn (50+ developers)
- Domain boundaries rõ ràng
- Scale requirements không đồng đều giữa các phần
- Tổ chức đã mature về DevOps (không có CI/CD tốt, microservices là thảm họa)

#### 4.4 ShopLite thuộc loại nào và tại sao?

**ShopLite sử dụng mô hình 3-Tier với xu hướng Modular Monolith.**

```
ShopLite Architecture:

Browser
  │
  │ HTTPS
  ▼
┌──────────────────────────┐
│  Frontend (React SPA)    │  ← Tier 1: Presentation
│  - Product listing       │    Chạy trên Nginx/CDN
│  - Shopping cart (Redux) │    Static files
│  - Checkout flow         │
└─────────────┬────────────┘
              │ REST API (JSON)
              │ Port 8080
              ▼
┌──────────────────────────┐
│  Backend (ASP.NET Core)  │  ← Tier 2: Business Logic
│  .NET 8                  │    Chạy trên App Server
│  - /api/products         │    Business rules
│  - /api/orders           │    Authentication
│  - /api/auth             │    Validation
│  - /api/users            │
└──────┬──────────┬────────┘
       │ SQL      │ Redis Protocol
       ▼          ▼
┌──────────┐ ┌──────────┐     ← Tier 3: Data
│PostgreSQL│ │  Redis   │       Persistent data
│Port 5432 │ │Port 6379 │       Cache layer
│Orders,   │ │Session,  │
│Products, │ │Cart,     │
│Users     │ │Rate limit│
└──────────┘ └──────────┘
```

**Lý do ShopLite chọn 3-Tier:**

1. **Phù hợp với mục tiêu học tập**: Đủ phức tạp để học real-world practices, không quá phức tạp để overwhelm người học
2. **Team size**: ShopLite là dự án nhỏ, microservices sẽ tạo quá nhiều overhead
3. **Domain clarity**: Với e-commerce đơn giản, không cần chia nhỏ hơn
4. **Operations simplicity**: Dễ dàng chạy trên một VPS hoặc vài containers
5. **Học DevOps core skills**: CI/CD, Docker, Kubernetes basics đều áp dụng được với 3-tier

---

### 5. Cài đặt môi trường từng bước

#### 5.1 Phương án A: WSL2 trên Windows

WSL2 (Windows Subsystem for Linux 2) cho phép chạy Linux kernel thực sự bên trong Windows, hiệu suất gần như native Linux.

**Bước 1: Bật tính năng WSL và Virtual Machine Platform**

Mở PowerShell với quyền Administrator:

```powershell
# Bật WSL
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

# Bật Virtual Machine Platform (cần cho WSL2)
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

# Khởi động lại máy
Restart-Computer
```

**Bước 2: Cập nhật WSL2 Linux kernel**

```powershell
# Sau khi restart, mở PowerShell as Admin
# Tải và cài đặt WSL2 kernel update
# Truy cập: https://aka.ms/wsl2kernel
# Hoặc dùng lệnh:
wsl --update

# Đặt WSL2 làm mặc định
wsl --set-default-version 2
```

**Bước 3: Cài Ubuntu 22.04 từ Microsoft Store**

```powershell
# Cách 1: Qua Microsoft Store
# Tìm "Ubuntu 22.04.3 LTS" → Install

# Cách 2: Qua command line
wsl --install -d Ubuntu-22.04

# Kiểm tra các distro đã cài
wsl --list --verbose
```

**Bước 4: Cấu hình WSL2**

Tạo file cấu hình WSL:

```bash
# Trong Windows, tạo file C:\Users\<username>\.wslconfig
[wsl2]
memory=4GB          # Giới hạn RAM cho WSL2
processors=2        # Số CPU cores
swap=2GB            # Swap space
localhostForwarding=true

[experimental]
autoMemoryReclaim=gradual  # Tự động giải phóng memory
```

Cấu hình bên trong Ubuntu:

```bash
# Tạo /etc/wsl.conf trong Ubuntu
sudo tee /etc/wsl.conf << 'EOF'
[boot]
systemd=true  # Bật systemd (cần cho nhiều services)

[automount]
options = "metadata,umask=22,fmask=11"

[network]
generateHosts = true
generateResolvConf = true

[interop]
enabled = true
appendWindowsPath = true
EOF
```

**Bước 5: Cài đặt các tool cần thiết trong WSL2**

```bash
# Cập nhật package list
sudo apt update && sudo apt upgrade -y

# Cài build tools
sudo apt install -y \
  build-essential \
  curl \
  wget \
  git \
  vim \
  htop \
  jq \
  unzip \
  ca-certificates \
  gnupg \
  lsb-release

# Cài Docker trong WSL2
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Cài .NET 8 SDK
wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb \
  -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb
sudo apt update && sudo apt install -y dotnet-sdk-8.0

# Verify
uname -a           # Linux kernel version
lsb_release -a     # Ubuntu version
docker --version   # Docker version
dotnet --version   # .NET SDK version
```

#### 5.2 Phương án B: VirtualBox Ubuntu Server

**Bước 1: Download và cài đặt VirtualBox**

1. Truy cập https://www.virtualbox.org/wiki/Downloads
2. Tải VirtualBox 7.x cho Windows
3. Tải VirtualBox Extension Pack (cùng version)
4. Cài đặt VirtualBox trước, sau đó install Extension Pack

**Bước 2: Download Ubuntu Server 22.04 LTS ISO**

```
URL: https://ubuntu.com/download/server
File: ubuntu-22.04.3-live-server-amd64.iso
Size: ~1.4 GB
SHA256: Verify sau khi download
```

**Bước 3: Tạo VM mới trong VirtualBox**

```
New VM:
├── Name: DevOps-Lab
├── Type: Linux
├── Version: Ubuntu (64-bit)
├── Memory: 4096 MB (4GB) — tối thiểu 2GB
├── Processors: 2 CPUs
└── Create virtual hard disk:
    ├── Type: VDI (VirtualBox Disk Image)
    ├── Storage: Dynamically allocated
    └── Size: 30 GB (tối thiểu 20GB)
```

**Bước 4: Cấu hình Network**

```
VM Settings → Network:
Adapter 1:
  ├── Enable Network Adapter: ✓
  ├── Attached to: Bridged Adapter
  └── Name: [chọn card mạng WiFi/Ethernet của máy host]

→ Bridged mode: VM sẽ có IP riêng trong cùng network với host
→ Giúp dễ dàng SSH từ Windows vào VM
```

**Bước 5: Cài Ubuntu Server**

```
Boot từ ISO → Chọn language → Keyboard → Network (DHCP)
Storage: Use entire disk
Profile setup:
  ├── Your name: devops
  ├── Server name: devops-lab
  ├── Username: devops
  └── Password: [đặt password mạnh]
SSH: Install OpenSSH server ✓
Featured Snaps: Bỏ qua, nhấn Done
```

**Bước 6: SSH vào VM từ Windows**

```powershell
# Lấy IP của VM (trong VM, chạy)
ip addr show | grep inet

# Từ Windows Terminal/PowerShell
ssh devops@192.168.1.xxx

# Tạo SSH key để không phải nhập password
ssh-keygen -t ed25519 -C "devops-lab"
ssh-copy-id devops@192.168.1.xxx
```

#### 5.3 Cài đặt VS Code với Extensions

**Cài VS Code:**

1. Download từ https://code.visualstudio.com/
2. Chọn "System Installer" cho Windows
3. Bật option "Add to PATH" trong quá trình cài

**Extensions cần thiết:**

```
Remote Development (bắt buộc):
├── Remote - WSL (ms-vscode-remote.remote-wsl)
│   → Cho phép VS Code kết nối vào WSL2
├── Remote - SSH (ms-vscode-remote.remote-ssh)
│   → Kết nối vào VM qua SSH
└── Remote - Containers (ms-vscode-remote.remote-containers)
    → Develop bên trong Docker container

Infrastructure as Code:
├── HashiCorp Terraform (hashicorp.terraform)
│   → Syntax highlighting, autocomplete cho .tf files
├── Ansible (redhat.ansible)
│   → Lint và autocomplete cho Ansible playbooks
└── Kubernetes (ms-kubernetes-tools.vscode-kubernetes-tools)
    → Manage K8s cluster từ VS Code

Docker:
└── Docker (ms-azuretools.vscode-docker)
    → Manage containers, images, registries

Git & Code Quality:
├── GitLens (eamodio.gitlens)
│   → Xem git blame, history inline
├── Git Graph (mhutchie.git-graph)
│   → Visualize git branches
└── Error Lens (usernamehako.error-lens)
    → Hiển thị lỗi inline

Languages:
├── YAML (redhat.vscode-yaml)
│   → Validate YAML, schema support
├── C# Dev Kit (ms-dotnettools.csdevkit)
│   → Full C# development support (IntelliSense, debug, test)
└── .NET Install Tool (ms-dotnettools.vscode-dotnet-runtime)
    → Manage .NET runtime versions

Productivity:
├── Thunder Client (rangav.vscode-thunder-client)
│   → REST API client như Postman nhưng trong VS Code
└── Draw.io Integration (hediet.vscode-drawio)
    → Vẽ diagram trong VS Code
```

**Cài Extension qua command line:**

```bash
# Cài tất cả extensions cần thiết một lần
code --install-extension ms-vscode-remote.remote-wsl
code --install-extension ms-vscode-remote.remote-ssh
code --install-extension ms-vscode-remote.remote-containers
code --install-extension hashicorp.terraform
code --install-extension ms-kubernetes-tools.vscode-kubernetes-tools
code --install-extension ms-azuretools.vscode-docker
code --install-extension eamodio.gitlens
code --install-extension redhat.vscode-yaml
code --install-extension ms-dotnettools.csdevkit
code --install-extension ms-dotnettools.vscode-dotnet-runtime
code --install-extension rangav.vscode-thunder-client
```

**Kết nối VS Code với WSL2:**

```bash
# Trong WSL2 terminal
cd ~/projects
code .  # Mở VS Code với WSL2 remote mode

# VS Code sẽ tự động cài VS Code Server trong WSL2
# và kết nối. Bottom-left sẽ hiển thị "><WSL: Ubuntu-22.04"
```

**Bước xác minh môi trường:**

```bash
# Chạy các lệnh này trong WSL2 / VM để verify
uname -a
# → Linux hostname 5.15.153.1-microsoft-standard-WSL2 #1 SMP ...

lsb_release -a
# → Ubuntu 22.04.3 LTS

docker --version
# → Docker version 24.0.x, build xxxxx

docker run hello-world
# → Hello from Docker!

dotnet --version
# → 8.x.x

git --version
# → git version 2.43.x
```

---

### 6. ShopLite Architecture Overview

#### 6.1 Tổng quan hệ thống

ShopLite là một ứng dụng thương mại điện tử quy mô nhỏ được thiết kế đặc biệt để học DevOps. Nó đủ thực tế để áp dụng các practices của industry, nhưng đủ đơn giản để không bị overwhelm bởi business logic.

```
ShopLite System Overview
═══════════════════════════════════════════════════════════════

                    Internet
                       │
              ┌────────▼────────┐
              │   Load Balancer  │
              │  (Nginx/Traefik) │
              └────────┬────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
   ┌──────▼──┐  ┌──────▼──┐  ┌─────▼───┐
   │Frontend  │  │Frontend  │  │Frontend │   CDN/Static
   │(React)   │  │(React)   │  │(React)  │   Replicas
   │:80       │  │:80       │  │:80      │
   └──────────┘  └──────────┘  └─────────┘
                       │ API Calls
              ┌────────▼────────┐
              │   API Gateway   │
              │  (optional)     │
              └────────┬────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
   ┌──────▼──┐  ┌──────▼──┐  ┌─────▼───┐
   │Backend   │  │Backend   │  │Backend  │   App Replicas
   │(ASP.NET) │  │(ASP.NET) │  │(ASP.NET)│   Horizontal
   │:8080     │  │:8080     │  │:8080    │   Scaling
   └──────────┘  └──────────┘  └─────────┘
                       │
          ┌────────────┼────────────┐
          │                         │
   ┌──────▼──────┐         ┌────────▼───┐
   │ PostgreSQL   │         │   Redis    │
   │ (Primary)    │         │  Cluster   │
   │ :5432        │         │  :6379     │
   └──────┬───────┘         └────────────┘
          │ Replication
   ┌──────▼───────┐
   │ PostgreSQL   │
   │ (Replica)    │
   │ Read-only    │
   └──────────────┘
```

#### 6.2 Frontend — React SPA

```
Frontend Stack:
├── React 18 (UI framework)
├── Redux Toolkit (state management)
├── React Router (client-side routing)
├── Axios (HTTP client)
├── Tailwind CSS (styling)
└── Vite (build tool)

Pages:
├── / (Home — product listing)
├── /products/:id (Product detail)
├── /cart (Shopping cart)
├── /checkout (Checkout flow)
├── /orders (Order history)
└── /login, /register (Auth)
```

**Luồng request từ Frontend:**

```javascript
// Khi user click "Add to Cart"
const addToCart = async (productId, quantity) => {
  // 1. Lấy auth token từ Redux store
  const token = store.getState().auth.token;
  
  // 2. Gọi backend API
  const response = await axios.post('/api/cart/items', 
    { productId, quantity },
    { headers: { Authorization: `Bearer ${token}` } }
  );
  
  // 3. Cập nhật Redux state
  dispatch(cartSlice.actions.itemAdded(response.data));
  
  // 4. Cập nhật UI tự động qua React re-render
};
```

#### 6.3 Backend — Node.js REST API

```
Backend Stack:
├── .NET 8 (ASP.NET Core)
├── ASP.NET Core Web API (HTTP framework)
├── Entity Framework Core (PostgreSQL ORM)
├── StackExchange.Redis (Redis client)
├── Microsoft.AspNetCore.Authentication.JwtBearer (JWT auth)
├── BCrypt.Net-Next (password hashing)
├── FluentValidation (input validation)
└── Serilog (structured logging)

API Endpoints:
├── POST   /api/auth/login
├── POST   /api/auth/register
├── POST   /api/auth/refresh
│
├── GET    /api/products
├── GET    /api/products/:id
├── POST   /api/products (admin)
├── PUT    /api/products/:id (admin)
│
├── GET    /api/cart
├── POST   /api/cart/items
├── PUT    /api/cart/items/:id
├── DELETE /api/cart/items/:id
│
├── GET    /api/orders
├── POST   /api/orders
├── GET    /api/orders/:id
│
└── GET    /health/live
    GET    /health/ready
```

#### 6.4 Database — PostgreSQL Schema

```sql
-- Core tables trong ShopLite

CREATE TABLE users (
    id          SERIAL PRIMARY KEY,
    email       VARCHAR(255) UNIQUE NOT NULL,
    password    VARCHAR(255) NOT NULL,  -- bcrypt hash
    name        VARCHAR(255) NOT NULL,
    role        VARCHAR(50) DEFAULT 'customer',
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE products (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    description TEXT,
    price       DECIMAL(10, 2) NOT NULL,
    stock       INTEGER DEFAULT 0,
    image_url   VARCHAR(500),
    category    VARCHAR(100),
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    user_id     INTEGER REFERENCES users(id),
    status      VARCHAR(50) DEFAULT 'pending',
    -- pending → paid → processing → shipped → delivered
    total       DECIMAL(10, 2) NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE order_items (
    id          SERIAL PRIMARY KEY,
    order_id    INTEGER REFERENCES orders(id),
    product_id  INTEGER REFERENCES products(id),
    quantity    INTEGER NOT NULL,
    price       DECIMAL(10, 2) NOT NULL  -- giá tại thời điểm mua
);

-- Index để tăng performance
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
CREATE INDEX idx_products_category ON products(category);
```

#### 6.5 Redis — Cache Layer

```
Redis được dùng cho:

1. Session Storage
   Key: session:{token}
   Value: { user_id, role, email }
   TTL: 24 giờ

2. Product Cache
   Key: product:{id}
   Value: JSON product object
   TTL: 5 phút (invalidate khi product thay đổi)

3. Cart Storage
   Key: cart:{user_id}
   Value: Hash { product_id: quantity }
   TTL: 7 ngày

4. Rate Limiting
   Key: rate_limit:{ip}:{endpoint}
   Value: counter
   TTL: 1 phút
   Logic: Nếu counter > 100, return 429 Too Many Requests
```

#### 6.6 Luồng request hoàn chỉnh: từ browser đến database

Hãy trace một request "GET /api/products?page=1" từ đầu đến cuối:

```
1. Browser gửi request
   GET https://shoplite.com/api/products?page=1
   Headers: Authorization: Bearer eyJhbGc...

2. DNS Resolution
   shoplite.com → 203.0.113.50 (Load Balancer IP)

3. Load Balancer (Nginx)
   Nhận request, health check backends
   → Chọn backend instance ít tải nhất (round-robin/least-conn)
   → Forward đến 10.0.1.5:8080

4. Backend (Node.js Express)
   a. Middleware stack:
      - Morgan: log request
      - Helmet: set security headers
      - CORS: check origin
      - JWT verify: decode token, attach user to req
      - Rate limiter: check Redis counter
   
   b. Route handler: GET /api/products
   
   c. Check Redis cache:
      GET products:page:1:limit:20
      → Cache HIT? Return JSON, skip DB
      → Cache MISS? Continue to DB

   d. Query PostgreSQL:
      SELECT p.*, COUNT(oi.id) as order_count
      FROM products p
      LEFT JOIN order_items oi ON p.id = oi.product_id
      WHERE p.stock > 0
      ORDER BY p.created_at DESC
      LIMIT 20 OFFSET 0;

   e. Store result in Redis:
      SET products:page:1:limit:20 <json> EX 300

   f. Return JSON response:
      {
        "products": [...],
        "pagination": {
          "page": 1,
          "total": 150,
          "hasNext": true
        }
      }

5. Response đi ngược từ backend → LB → browser

6. Browser render:
   React nhận data → Update Redux store
   → Components re-render → Hiển thị sản phẩm
```

**Thời gian thực tế:**
- DNS: ~1ms (cached)
- Network + LB: ~5ms
- Backend processing: ~2ms
- Redis lookup: ~1ms (hit) hoặc ~50ms (miss + DB query)
- **Total: ~10-60ms**

---

### 7. Tài nguyên học tập

#### 7.1 Lộ trình học DevOps

**roadmap.sh/devops** — Website tốt nhất để có bức tranh tổng quan

```
URL: https://roadmap.sh/devops

Cấu trúc lộ trình:
├── Learn a Programming Language (Python/Go/Bash)
├── OS & Linux Basics
│   ├── Process management
│   ├── Networking basics
│   └── File system
├── Networking (HTTP, SSH, DNS, Firewall)
├── What is and how to setup X
│   ├── Reverse proxy (Nginx)
│   ├── Caching server (Redis)
│   └── Load balancer
├── Containers (Docker)
├── Container Orchestration (Kubernetes)
├── Infrastructure as Code (Terraform, Ansible)
├── CI/CD (GitHub Actions, Jenkins)
├── Monitoring & Observability
│   ├── Prometheus + Grafana
│   └── ELK Stack
└── Cloud Providers (AWS/GCP/Azure)
```

#### 7.2 Sách nên đọc

**Cuốn 1: The DevOps Handbook (2016)**

```
Tác giả: Gene Kim, Jez Humble, Patrick Debois, John Willis
Nhà xuất bản: IT Revolution Press

Nội dung chính:
├── Phần 1: The Three Ways
│   ├── First Way: Flow (từ Dev đến Ops đến Customer)
│   ├── Second Way: Feedback (từ right-to-left, phát hiện vấn đề sớm)
│   └── Third Way: Continual Learning (thử nghiệm và học)
├── Phần 2: Where to Start (chọn value stream để bắt đầu)
├── Phần 3: Technical Practices of Flow (CI/CD, deployment)
├── Phần 4: Technical Practices of Feedback (monitoring, testing in production)
└── Phần 5: Technical Practices of Learning (blameless postmortems, chaos engineering)

Tại sao đọc: Đây là "bible" của DevOps. Mọi concept bạn học đều có
nguồn gốc từ cuốn này.
```

**Cuốn 2: The Phoenix Project (2013)**

```
Tác giả: Gene Kim, Kevin Behr, George Spafford
Nhà xuất bản: IT Revolution Press
Thể loại: Tiểu thuyết kỹ thuật (Technical Novel)

Cốt truyện: Bill Palmer được bổ nhiệm đột xuất làm VP of IT Operations
tại Parts Unlimited — một công ty ô tô đang hấp hối vì hệ thống IT
hỗn loạn. Cuốn sách follow hành trình của Bill trong 90 ngày để
"cứu" công ty.

Tại sao đặc biệt: Thay vì giải thích khái niệm khô khan, cuốn sách
cho bạn *cảm nhận* vấn đề của Dev-Ops tách biệt. Bạn sẽ đồng cảm
với từng nhân vật và hiểu tại sao DevOps là câu trả lời.

Các nhân vật đáng nhớ:
├── Bill Palmer: Protagonist, VP IT Operations bất đắc dĩ
├── Erik Reid: Mentor huyền bí, dạy Bill về "The Three Ways"
├── Brent: "The hero" — người duy nhất biết fix mọi thứ
│          (anti-pattern: Single point of failure)
└── Steve Masters: CEO muốn đóng cửa IT department
```

**Cuốn 3: Site Reliability Engineering (2016)**

```
Tác giả: Niall Richard Murphy et al. (Google SRE team)
Nhà xuất bản: O'Reilly
Đọc online miễn phí: https://sre.google/sre-book/table-of-contents/

Relevance với DevOps:
SRE là cách Google implement DevOps. Nhiều concepts trong SRE
(error budgets, SLI/SLO/SLA, toil elimination) đã trở thành
industry standard.
```

#### 7.3 Khóa học online

| Khóa học | Platform | Link |
|----------|----------|------|
| DevOps Beginners to Advanced | Udemy | Tìm "Mumshad Mannambeth" |
| Docker and Kubernetes: The Complete Guide | Udemy | Stephen Grider |
| Linux for DevOps Engineers | YouTube | TechWorld with Nana (free) |
| Kubernetes Tutorial for Beginners | YouTube | TechWorld with Nana (free) |
| AWS Certified Solutions Architect | A Cloud Guru | Có free tier |

#### 7.4 Communities và Blogs

```
Communities:
├── DevOps subreddit: reddit.com/r/devops
├── CNCF Slack: Slack của Cloud Native Computing Foundation
│              slack.cncf.io (free)
└── DevOps Vietnam Facebook Group

Blogs/News:
├── The New Stack: thenewstack.io
├── InfoQ DevOps: infoq.com/devops
├── Increment Magazine: increment.com
└── Martin Fowler's blog: martinfowler.com

Practice Platforms:
├── KillerCoda: killercoda.com (Kubernetes labs)
├── Play with Docker: labs.play-with-docker.com
├── Katacoda: katacoda.com (DevOps scenarios)
└── AWS Free Tier: aws.amazon.com/free
```

---

## Kỹ năng đạt được
- Diễn đạt được "DevOps là gì" bằng ngôn ngữ của doanh nghiệp.
- Thiết lập được môi trường lab cá nhân.
- Đọc hiểu kiến trúc của một ứng dụng web nhiều tầng.

---

## Thực hành

**Môi trường:** Máy cá nhân + WSL2 hoặc VirtualBox.

**Lab:**
- Cài đặt môi trường ảo hóa / WSL2.
- Tải source ShopLite về, đọc cấu trúc thư mục, chạy thử frontend + backend ở chế độ dev.

**Công cụ:** VirtualBox/WSL2, VS Code, trình duyệt.

---

## Bài tập

- **Bắt buộc:** Vẽ sơ đồ kiến trúc ShopLite (frontend ↔ backend ↔ database ↔ cache) bằng tay hoặc draw.io.
- **Nâng cao:** Viết một trang ghi chú so sánh monolith vs microservices và nêu ShopLite thuộc loại nào, vì sao.
- **Mô phỏng doanh nghiệp:** Viết một "định nghĩa dự án" (project charter) 1 trang: mục tiêu hệ thống, người dùng, các thành phần.

---

## Deliverable
- [ ] Môi trường lab hoạt động.
- [ ] Sơ đồ kiến trúc v0 của ShopLite (file ảnh/draw.io).
- [ ] File `README.md` mô tả dự án mục tiêu.

---

## Tiêu chí hoàn thành
- [ ] Giải thích được DevOps cho người không chuyên trong 2 phút.
- [ ] Lab khởi động được, chạy được ứng dụng mẫu ở mức cơ bản.
- [ ] Có sơ đồ kiến trúc và project charter.

---

## Ghi chú học tập

> _Ghi lại những điều học được, vướng mắc và insight trong quá trình học phase này._
