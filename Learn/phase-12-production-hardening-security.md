# Phase 12 — Production Hardening & Security

## Mục tiêu
- Biến hệ thống "chạy được" thành hệ thống "chạy production an toàn, bền vững".
- Áp dụng tư duy bảo mật, sao lưu và độ sẵn sàng cao.

---

## Kiến thức sẽ học

- **Quản lý secrets nâng cao** (Vault / AWS Secrets Manager / Sealed Secrets).
- **Backup & Restore & Disaster Recovery** (sao lưu DB tự động, kiểm thử phục hồi).
- **High Availability** (multi-AZ, replica) — chịu lỗi.
- **Security scanning** (image scanning với Trivy, dependency scanning, SAST cơ bản).
- **Least privilege & IAM hardening** — chỉ cấp quyền tối thiểu.
- **Network policy & bảo mật cluster** — cô lập service.
- **DevSecOps cơ bản** — tích hợp bảo mật vào pipeline.
- **Cost optimization** — tối ưu chi phí cloud.

---

## Kiến thức chi tiết

### 1. Security Mindset — Tư duy bảo mật

Bảo mật không phải là một tính năng thêm vào sau khi hệ thống đã xây xong. Bảo mật phải được thiết kế từ đầu, baked-in vào mọi quyết định kiến trúc và vận hành. Dưới đây là bốn nguyên tắc nền tảng mà mọi Senior DevOps Engineer cần nắm vững.

#### 1.1 Defense in Depth — Nhiều lớp bảo vệ

Defense in Depth (bảo vệ theo chiều sâu) là nguyên tắc thiết kế nhiều lớp bảo mật độc lập nhau. Nếu một lớp bị vượt qua, còn các lớp tiếp theo ngăn chặn attacker. Không bao giờ tin vào một single point of security.

Ví dụ áp dụng cho ShopLite:
- **Lớp 1 — Network perimeter:** AWS Security Group chỉ mở port 443/80 ra ngoài, port 22 chỉ mở cho VPN/bastion host.
- **Lớp 2 — Application gateway:** NGINX Ingress với rate limiting, WAF (Web Application Firewall) block SQL injection, XSS.
- **Lớp 3 — Kubernetes network policy:** Pods không thể giao tiếp tự do, chỉ theo rule được định nghĩa rõ ràng.
- **Lớp 4 — Pod security context:** Container chạy với non-root user, read-only filesystem, drop capabilities.
- **Lớp 5 — Application layer:** Input validation, parameterized queries, output encoding.
- **Lớp 6 — Data layer:** Encryption at rest (RDS encrypted), encryption in transit (TLS), database user chỉ có quyền tối thiểu.
- **Lớp 7 — Audit & Monitoring:** CloudTrail log mọi AWS API call, Falco detect runtime anomalies, Grafana alert bất thường.

Khi một lớp bị compromise, attacker vẫn phải vượt qua 6 lớp còn lại. Đây là lý do tại sao defense in depth mạnh hơn việc chỉ có một tường lửa tốt.

#### 1.2 Principle of Least Privilege — Quyền tối thiểu

Mọi người dùng, service account, IAM role, database user đều chỉ được cấp đúng quyền cần thiết để thực hiện nhiệm vụ, không hơn một bit.

Sai lầm phổ biến:
- EC2 instance có policy `AdministratorAccess` — nếu instance bị compromise, attacker có toàn quyền AWS account.
- Application database user có quyền `CREATE TABLE`, `DROP TABLE` — nếu app bị SQL injection, attacker có thể xóa toàn bộ database.
- Tất cả developer có quyền push trực tiếp vào branch main — một developer bất cẩn có thể deploy code nguy hiểm.

Thực hành đúng:
- EC2 instance chỉ có quyền đọc secrets từ Secrets Manager và ghi logs lên CloudWatch.
- Application database user chỉ có `SELECT`, `INSERT`, `UPDATE`, `DELETE` trên các tables cần thiết. Migration scripts chạy bằng user riêng với quyền cao hơn, nhưng chỉ trong thời gian migration.
- Branch protection rules: require review, require passing CI, disallow force push.

#### 1.3 Zero Trust — Không tin tưởng mặc định

Mô hình bảo mật truyền thống: "những gì ở trong mạng nội bộ thì tin tưởng." Zero Trust lật ngược điều này: **không tin tưởng bất kỳ request nào, dù từ internal network hay external network, cho đến khi xác thực và ủy quyền thành công.**

Trong Kubernetes, điều này có nghĩa:
- Mọi service-to-service communication phải được authenticate (service mesh như Istio với mTLS).
- Không có "trusted namespace" — mọi namespace đều phải có NetworkPolicy.
- Service account token không nên được mount tự động vào pods không cần thiết.

```yaml
# Tắt auto-mount service account token cho pods không cần gọi K8s API
apiVersion: v1
kind: ServiceAccount
metadata:
  name: shoplite-backend
  namespace: shoplite
automountServiceAccountToken: false
```

Trong AWS:
- Dùng VPC endpoints thay vì traffic đi ra internet để gọi AWS services.
- Security Group không có rule "0.0.0.0/0 inbound" trừ port 80/443 trên Load Balancer.
- Dùng IAM Conditions để giới hạn API calls theo IP, theo thời gian, theo MFA.

#### 1.4 Assume Breach — Giả định đã bị xâm nhập

Tư duy "assume breach" không phải là thái độ tiêu cực mà là phương pháp thiết kế thực tế. Hãy tự hỏi: **"Nếu attacker đã vào được hệ thống, chúng ta có thể giới hạn thiệt hại như thế nào?"**

Minimize blast radius (giảm bán kính nổ):
- Mỗi microservice có database riêng — nếu backend bị compromise, attacker không tự động có quyền truy cập database của payment service.
- Secrets được phân chia theo môi trường — credentials của production không có trong staging.
- Network segmentation — nếu một pod trong namespace `dev` bị compromise, nó không thể kết nối sang namespace `production`.

Điều kiện phát hiện sớm:
- Honeypot tokens: tạo fake AWS credentials, nếu có request dùng credentials này thì chắc chắn là breach.
- Unusual API call patterns: CloudTrail + AWS GuardDuty tự động phát hiện.
- File integrity monitoring: Falco alert khi binary hệ thống bị sửa trong container.

---

### 2. Secrets Management — Quản lý bí mật

#### 2.1 Vấn đề phổ biến và nguy hiểm

**Hardcode password trong code:**

```python
# SAI - đừng bao giờ làm thế này
DATABASE_URL = "postgresql://admin:SuperSecret123@prod-db.example.com:5432/shoplite"
```

Vấn đề: git history không thể xóa hoàn toàn. Dù bạn commit xóa file đó, attacker vẫn có thể chạy `git log --all --full-history` để tìm lại. Ngay cả khi repo là private hôm nay, nếu nó trở thành public trong tương lai (hoặc bị leak), password đó coi như bị lộ vĩnh viễn.

**.env file bị commit vào git:**

```bash
# .gitignore PHẢI có dòng này
.env
.env.local
.env.production
*.env
```

Rất nhiều developer quên thêm `.env` vào `.gitignore`. Sau khi file đã được commit một lần, chỉ xóa file đó khỏi `.gitignore` là không đủ — secret đã nằm trong git history.

**Secret trong Docker image layers:**

```dockerfile
# SAI - password bị ghi vào image layer
FROM node:18
RUN npm config set //registry.npmjs.org/:_authToken=npm_secrettoken123
```

Dù bạn có xóa token ở bước sau, nó vẫn tồn tại trong layer trung gian. Ai kéo image về đều có thể chạy `docker history --no-trunc` để thấy.

```dockerfile
# ĐÚNG - dùng BuildKit secrets
# syntax=docker/dockerfile:1
FROM node:18
RUN --mount=type=secret,id=npm_token \
    npm config set //registry.npmjs.org/:_authToken=$(cat /run/secrets/npm_token)
```

**Secret trong CI/CD logs:**

```yaml
# SAI - in password ra logs
- name: Deploy
  run: |
    echo "Connecting to DB: $DATABASE_PASSWORD"  # logs này public nếu repo public
```

GitHub Actions, GitLab CI tự động mask các biến được đánh dấu là secret, nhưng bạn phải đánh dấu đúng. Nếu bạn dùng `echo` in ra một phần của secret (ví dụ: first 4 chars để debug), masking có thể không hoạt động.

#### 2.2 Git History Scanning

Trước khi push repo lên, nên scan toàn bộ git history để chắc chắn không có secret nào bị commit:

**Cài đặt và dùng git-secrets:**

```bash
# Cài trên Mac
brew install git-secrets

# Cài global hooks
git secrets --install ~/.git-templates/git-secrets
git config --global init.templateDir ~/.git-templates/git-secrets

# Thêm pattern AWS
git secrets --register-aws --global

# Scan repo hiện tại
git secrets --scan
git secrets --scan-history  # scan toàn bộ history
```

**Dùng trufflehog (mạnh hơn, detect entropy-based):**

```bash
# Cài
pip install trufflehog3
# hoặc dùng Docker
docker run --rm -v "$(pwd):/pwd" trufflesecurity/trufflehog:latest \
  git file:///pwd --since-commit HEAD~50 --only-verified

# Scan toàn bộ git history
trufflehog git file://. --json | jq '.SourceMetadata.Data.Git.commit'
```

**Gitleaks — tool phổ biến nhất:**

```bash
# Cài
brew install gitleaks  # Mac
# hoặc download binary từ GitHub releases

# Scan working directory
gitleaks detect --source . --verbose

# Scan git history
gitleaks detect --source . --log-opts="--all"

# Tạo report
gitleaks detect --source . --report-format json --report-path gitleaks-report.json
```

File cấu hình `.gitleaks.toml` cho phép whitelist false positives:

```toml
[extend]
useDefault = true

[[rules]]
id = "custom-api-key"
description = "Custom API Key"
regex = '''shoplite_[a-zA-Z0-9]{32}'''
tags = ["api-key", "shoplite"]

[allowlist]
description = "Global Allowlist"
commits = ["abc123def456"]  # commit hash cụ thể nếu cần ignore
regexes = [
  '''EXAMPLE_API_KEY''',  # test/example values
  '''shoplite_test_[a-zA-Z0-9]+''',  # test tokens
]
paths = [
  '''tests/fixtures/''',
  '''docs/examples/''',
]
```

#### 2.3 Pre-commit Hook Ngăn Commit Secrets

Pre-commit hooks chạy trước khi commit được tạo, có thể abort commit nếu phát hiện vấn đề.

**Cài pre-commit framework:**

```bash
pip install pre-commit
# hoặc
brew install pre-commit
```

**File `.pre-commit-config.yaml`:**

```yaml
repos:
  # Gitleaks - detect secrets
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

  # Detect private keys
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: detect-private-key
      - id: check-added-large-files
        args: ['--maxkb=1000']
      - id: check-merge-conflict
      - id: no-commit-to-branch
        args: [--branch, main, --branch, master, --branch, production]

  # Trufflehog
  - repo: https://github.com/trufflesecurity/trufflehog
    rev: v3.63.7
    hooks:
      - id: trufflehog
        name: TruffleHog
        entry: bash -c 'trufflehog git file://. --since-commit HEAD --only-verified --fail'
        language: system
        stages: ["commit"]
```

**Cài hooks:**

```bash
# Cài hooks vào .git/hooks/
pre-commit install

# Chạy thủ công trên toàn bộ files
pre-commit run --all-files

# Update hooks lên phiên bản mới nhất
pre-commit autoupdate
```

#### 2.4 AWS Secrets Manager

AWS Secrets Manager là dịch vụ managed để lưu, rotate và truy cập secrets.

**Tạo secret:**

```bash
# Tạo secret JSON chứa database credentials
aws secretsmanager create-secret \
  --name "shoplite/prod/database" \
  --description "ShopLite production database credentials" \
  --secret-string '{"username":"shoplite_app","password":"Sup3rS3cur3P@ss","host":"shoplite-prod.cluster-xyz.ap-southeast-1.rds.amazonaws.com","port":5432,"dbname":"shoplite"}'

# Tạo secret cho API keys
aws secretsmanager create-secret \
  --name "shoplite/prod/stripe-api" \
  --secret-string '{"secret_key":"sk_live_xxxxx","webhook_secret":"whsec_xxxxx"}'
```

**Đọc secret:**

```bash
# Đọc toàn bộ secret value
aws secretsmanager get-secret-value \
  --secret-id "shoplite/prod/database" \
  --query 'SecretString' \
  --output text

# Parse JSON để lấy field cụ thể
aws secretsmanager get-secret-value \
  --secret-id "shoplite/prod/database" \
  --query 'SecretString' \
  --output text | jq -r '.password'
```

**Trong ứng dụng .NET (C#) — fetch tại runtime:**

Cài package:

```bash
dotnet add package AWSSDK.SecretsManager
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
```

```csharp
// SecretsService.cs
using Amazon.SecretsManager;
using Amazon.SecretsManager.Model;
using System.Text.Json;

public record DatabaseCredentials(
    string username, string password,
    string host, int port, string dbname);

public class SecretsService
{
    private readonly IAmazonSecretsManager _client;

    public SecretsService()
    {
        _client = new AmazonSecretsManagerClient(Amazon.RegionEndpoint.APSoutheast1);
    }

    public async Task<DatabaseCredentials> GetDatabaseCredentialsAsync()
    {
        var request = new GetSecretValueRequest
        {
            SecretId = "shoplite/prod/database"
        };
        var response = await _client.GetSecretValueAsync(request);
        return JsonSerializer.Deserialize<DatabaseCredentials>(response.SecretString)!;
    }
}
```

```csharp
// Program.cs — đăng ký và sử dụng
builder.Services.AddSingleton<SecretsService>();

// Fetch credentials khi startup
var tempProvider = builder.Services.BuildServiceProvider();
var secretsService = tempProvider.GetRequiredService<SecretsService>();
var creds = await secretsService.GetDatabaseCredentialsAsync();

builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseNpgsql(
        $"Host={creds.host};Port={creds.port};Database={creds.dbname};" +
        $"Username={creds.username};Password={creds.password};SSL Mode=Require;Trust Server Certificate=false;"));
```

**Auto-rotation — RDS password tự động rotate:**

```bash
# Bật auto-rotation cho RDS credentials, rotate mỗi 30 ngày
aws secretsmanager rotate-secret \
  --secret-id "shoplite/prod/database" \
  --rotation-rules AutomaticallyAfterDays=30 \
  --rotation-lambda-arn arn:aws:lambda:ap-southeast-1:123456789:function:SecretsManagerRDSPostgreSQLRotationSingleUser
```

AWS cung cấp sẵn Lambda function để rotate RDS passwords. Lambda function sẽ:
1. Tạo password mới
2. Update password trong RDS
3. Update secret trong Secrets Manager
4. Test kết nối với password mới
5. Xóa password cũ

#### 2.5 Kubernetes Sealed Secrets

Vấn đề với Kubernetes Secrets: chúng chỉ được base64 encode, không phải encrypt. Nếu commit Kubernetes Secret vào git, bất kỳ ai có quyền đọc repo đều có thể decode và lấy secret.

Sealed Secrets giải quyết vấn đề này bằng cách encrypt Secret bằng public key của cluster. Chỉ cluster đó mới có thể decrypt với private key.

**Cài Sealed Secrets controller vào cluster:**

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system \
  --set fullnameOverride=sealed-secrets-controller
```

**Cài kubeseal CLI:**

```bash
brew install kubeseal  # Mac
# hoặc download từ GitHub releases
```

**Tạo SealedSecret:**

```bash
# Tạo Secret thường (KHÔNG commit cái này vào git)
kubectl create secret generic shoplite-db-secret \
  --namespace shoplite \
  --from-literal=password=SuperSecret123 \
  --from-literal=username=shoplite_app \
  --dry-run=client -o yaml > secret.yaml

# Seal secret (file này CÓ THỂ commit vào git)
kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --format yaml < secret.yaml > sealed-secret.yaml

# Xóa file secret thường (không commit)
rm secret.yaml

# Apply SealedSecret vào cluster
kubectl apply -f sealed-secret.yaml
```

File `sealed-secret.yaml` trông như sau:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: shoplite-db-secret
  namespace: shoplite
spec:
  encryptedData:
    password: AgBy8hX9jKLm...  # encrypted, safe to commit
    username: AgCd4eF7gHiJ...  # encrypted, safe to commit
  template:
    metadata:
      name: shoplite-db-secret
      namespace: shoplite
    type: Opaque
```

Sealed Secrets controller trong cluster sẽ watch `SealedSecret` resources, decrypt chúng và tạo ra Kubernetes `Secret` thực sự. Pods có thể dùng Secret đó như bình thường.

**Lưu ý quan trọng:** Backup private key của Sealed Secrets controller! Nếu mất private key, bạn không thể decrypt các SealedSecrets cũ và phải re-seal tất cả secrets.

```bash
# Backup private key
kubectl get secret -n kube-system sealed-secrets-key -o yaml > sealed-secrets-master-key.yaml
# Lưu file này ở nơi an toàn (không phải git repo)
```

---

### 3. Container Security — Bảo mật Container

#### 3.1 Non-root User

Chạy container với root user là rủi ro nghiêm trọng. Nếu container bị compromise và process escape khỏi container, attacker có thể có quyền root trên host node.

**Trong Dockerfile (.NET multi-stage build):**

```dockerfile
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /app

# Restore riêng để tận dụng Docker layer cache
COPY *.csproj ./
RUN dotnet restore --runtime linux-musl-x64

# Build và publish
COPY . .
RUN dotnet publish -c Release -o /app/publish \
    --runtime linux-musl-x64 --self-contained false --no-restore

# Stage 2: Runtime image nhỏ, không có SDK
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS runtime

# Tạo group và user riêng (non-root)
RUN addgroup -g 1001 -S dotnet && \
    adduser -S -u 1001 -G dotnet dotnet

WORKDIR /app

# Copy chỉ published output với đúng ownership
COPY --from=build --chown=dotnet:dotnet /app/publish ./

# Switch sang non-root user TRƯỚC CMD
USER dotnet

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
CMD ["dotnet", "ShopLite.Backend.dll"]
```

**Trong Kubernetes Pod SecurityContext:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shoplite-backend
  namespace: shoplite
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shoplite
      tier: backend
  template:
    metadata:
      labels:
        app: shoplite
        tier: backend
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: backend
          image: ghcr.io/user/shoplite-backend:v1.0.0
          securityContext:
            runAsNonRoot: true
            runAsUser: 1001
            allowPrivilegeEscalation: false
```

#### 3.2 Read-only Root Filesystem

Với read-only root filesystem, attacker không thể ghi file mới vào container (ví dụ: không thể download và chạy malware).

```yaml
spec:
  containers:
    - name: backend
      image: ghcr.io/user/shoplite-backend:v1.0.0
      securityContext:
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
        runAsNonRoot: true
        runAsUser: 1001
      # Mount các directory cần write
      volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
        - name: app-cache
          mountPath: /app/.cache
        - name: logs-dir
          mountPath: /app/logs
  volumes:
    - name: tmp-dir
      emptyDir: {}
    - name: app-cache
      emptyDir:
        sizeLimit: 100Mi
    - name: logs-dir
      emptyDir:
        sizeLimit: 500Mi
```

#### 3.3 Drop Linux Capabilities

Linux capabilities là các quyền fine-grained thay vì binary root/non-root. Mặc định Docker container có một số capabilities không cần thiết.

```yaml
securityContext:
  capabilities:
    drop:
      - ALL  # drop tất cả capabilities
    add:
      - NET_BIND_SERVICE  # chỉ add lại nếu cần bind port < 1024
```

Các capabilities phổ biến cần biết:
- `NET_BIND_SERVICE`: bind port dưới 1024 (HTTP port 80, HTTPS port 443)
- `NET_ADMIN`: configure network interfaces — hầu như không bao giờ cần
- `SYS_ADMIN`: nhiều quyền admin — rất nguy hiểm, không bao giờ dùng trừ có lý do đặc biệt
- `CHOWN`: thay đổi file ownership

#### 3.4 Pod Security Standards (Kubernetes 1.25+)

Pod Security Standards thay thế PodSecurityPolicy (deprecated). Có ba profile:

| Profile | Mô tả | Use case |
|---------|-------|----------|
| `privileged` | Không có restrictions | Infrastructure components (CNI, CSI) |
| `baseline` | Ngăn known privilege escalations | General workloads |
| `restricted` | Hardened theo best practices | Production applications |

**Annotate namespace để enforce policy:**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: shoplite
  labels:
    # enforce: fail pod nếu vi phạm
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28
    # audit: log vi phạm nhưng không block
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.28
    # warn: hiện warning khi apply manifest vi phạm
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.28
```

Với profile `restricted`, pod phải:
- Không dùng `hostNetwork`, `hostPID`, `hostIPC`
- Không mount hostPath volumes
- Chạy với non-root user
- Drop ALL capabilities
- Set `allowPrivilegeEscalation: false`
- Set `seccompProfile: RuntimeDefault` hoặc `Localhost`

**Kiểm tra namespace hiện tại có vi phạm không:**

```bash
# Dry-run enforce restricted policy
kubectl label --dry-run=server --overwrite ns shoplite \
  pod-security.kubernetes.io/enforce=restricted

# Xem warnings sẽ xuất hiện
kubectl get pods -n shoplite -o yaml | \
  kubectl-convert -f - --output-version v1 2>&1 | grep -i warning
```

---

### 4. Image Security với Trivy

#### 4.1 Cài đặt Trivy

```bash
# macOS
brew install trivy

# Ubuntu/Debian
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install trivy

# Dùng Docker (không cần cài)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest image nginx:latest
```

#### 4.2 Scan Docker Image

```bash
# Scan image cơ bản
trivy image nginx:latest

# Chỉ hiện HIGH và CRITICAL vulnerabilities
trivy image --severity HIGH,CRITICAL nginx:latest

# Scan image của ShopLite
trivy image --severity HIGH,CRITICAL ghcr.io/user/shoplite-backend:v1.0.0

# Exit code 1 nếu có CRITICAL — dùng trong CI/CD để fail pipeline
trivy image --exit-code 1 --severity CRITICAL ghcr.io/user/shoplite-backend:v1.0.0

# Ignore unfixed vulnerabilities (chưa có fix thì không làm được gì)
trivy image --exit-code 1 --severity CRITICAL --ignore-unfixed ghcr.io/user/shoplite-backend:v1.0.0

# Output dạng JSON để parse
trivy image --format json --output trivy-report.json ghcr.io/user/shoplite-backend:v1.0.0

# Output dạng SARIF để upload lên GitHub Security tab
trivy image --format sarif --output trivy.sarif ghcr.io/user/shoplite-backend:v1.0.0
```

#### 4.3 Scan Source Code và Dependencies

```bash
# Scan filesystem — detect vulnerabilities trong dependencies
trivy fs .

# Scan với nhiều check types
trivy fs --security-checks vuln,config,secret .

# Scan chỉ package lock files
trivy fs --security-checks vuln package-lock.json
trivy fs --security-checks vuln requirements.txt
trivy fs --security-checks vuln go.sum

# Scan secrets trong source code
trivy fs --security-checks secret .
```

#### 4.4 Scan Kubernetes Manifests và Terraform

```bash
# Scan K8s manifests để detect misconfigurations
trivy config ./k8s/

# Scan Terraform files
trivy config ./terraform/

# Scan Helm charts
trivy config ./helm/shoplite/

# Verbose output với các best practices bị vi phạm
trivy config --severity MEDIUM,HIGH,CRITICAL ./k8s/
```

Ví dụ lỗi Trivy sẽ phát hiện trong K8s manifests:
- Container chạy với root user
- `privileged: true`
- Thiếu resource limits
- `hostNetwork: true`
- Không có readiness/liveness probes

#### 4.5 Tích hợp Trivy vào GitHub Actions

```yaml
name: Security Scan

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  trivy-scan:
    name: Trivy Security Scan
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write  # cần để upload SARIF

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build Docker image (.NET multi-stage)
        run: |
          docker build -t shoplite-backend:${{ github.sha }} ./backend

      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: shoplite-backend:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          ignore-unfixed: true
          exit-code: 1

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v3
        if: always()  # upload kể cả khi scan fail
        with:
          sarif_file: trivy-results.sarif

      - name: Scan K8s manifests
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: config
          scan-ref: ./k8s
          exit-code: 1
          severity: CRITICAL,HIGH
```

#### 4.6 File .trivyignore — Whitelist False Positives

```
# .trivyignore
# Format: CVE-ID hoặc GHSA-ID, một dòng một

# False positive - thư viện này chỉ dùng ở test environment, không expose ra production
CVE-2023-45853

# Đã review, không exploitable trong context của chúng ta vì không dùng feature này
CVE-2023-12345
# Reason: We don't use the XML parser component that contains this vulnerability
# Reviewed by: security-team@company.com
# Review date: 2024-01-15

# Chưa có fix, sẽ update khi có patch
GHSA-xxxx-yyyy-zzzz
```

---

### 5. Backup & Disaster Recovery

#### 5.1 RPO và RTO

**RPO (Recovery Point Objective)** — Mất tối đa bao nhiêu data?

Ví dụ RPO = 1 giờ: nếu disaster xảy ra lúc 14:30, bạn có thể restore về trạng thái 13:30. Tối đa mất 1 giờ dữ liệu.

**RTO (Recovery Time Objective)** — Downtime tối đa bao lâu?

Ví dụ RTO = 4 giờ: kể từ khi disaster xảy ra, hệ thống phải online trở lại trong vòng 4 giờ.

ShopLite targets:
- RPO: 1 giờ — acceptable với e-commerce nhỏ, không cần real-time replication
- RTO: 4 giờ — acceptable với business hours operation

Để đạt RPO 1 giờ: backup mỗi 1 giờ (cron job).
Để đạt RPO 5 phút: dùng RDS Multi-AZ với standby replica.
Để đạt RPO ~0: dùng synchronous replication (đắt hơn, latency cao hơn).

#### 5.2 RDS Automated Backup

AWS RDS có built-in backup tự động. Kích hoạt và cấu hình:

```bash
# Bật automated backup với retention 7 ngày
aws rds modify-db-instance \
  --db-instance-identifier shoplite-postgres \
  --backup-retention-period 7 \
  --preferred-backup-window "02:00-03:00" \
  --apply-immediately

# Tạo manual snapshot trước major change (migration, upgrade)
aws rds create-db-snapshot \
  --db-instance-identifier shoplite-postgres \
  --db-snapshot-identifier shoplite-pre-v2-migration-$(date +%Y%m%d)

# List snapshots
aws rds describe-db-snapshots \
  --db-instance-identifier shoplite-postgres \
  --query 'DBSnapshots[*].[DBSnapshotIdentifier,SnapshotCreateTime,Status]' \
  --output table

# Point-in-time restore — restore về bất kỳ giây nào trong retention period
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier shoplite-postgres \
  --target-db-instance-identifier shoplite-postgres-restored \
  --restore-time 2024-01-15T14:30:00Z
```

#### 5.3 PostgreSQL Backup Script

Script backup tự động, upload lên S3, tự cleanup backup cũ:

```bash
#!/bin/bash
# /opt/scripts/backup-postgres.sh
# Chạy bởi cron: 0 * * * * /opt/scripts/backup-postgres.sh >> /var/log/backup.log 2>&1

set -euo pipefail

# Configuration — được inject từ environment hoặc AWS Secrets Manager
DB_HOST="${DB_HOST:-localhost}"
DB_PORT="${DB_PORT:-5432}"
DB_NAME="${DB_NAME:-shoplite}"
DB_USER="${DB_USER:-shoplite}"
BACKUP_BUCKET="${BACKUP_BUCKET:-shoplite-backup}"
BACKUP_PREFIX="postgres"
RETENTION_DAYS=30

# Fetch password từ AWS Secrets Manager nếu chưa có trong env
if [ -z "${DB_PASSWORD:-}" ]; then
  DB_PASSWORD=$(aws secretsmanager get-secret-value \
    --secret-id "shoplite/prod/database" \
    --query 'SecretString' \
    --output text | jq -r '.password')
fi

# Tạo tên file backup với timestamp
BACKUP_DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="shoplite_${BACKUP_DATE}.sql.gz"
TEMP_DIR=$(mktemp -d)
TEMP_FILE="${TEMP_DIR}/${BACKUP_FILE}"

echo "[$(date)] Starting backup of database ${DB_NAME}"

# Cleanup temp directory khi script exit (dù thành công hay thất bại)
trap "rm -rf ${TEMP_DIR}" EXIT

# Dump database và compress
PGPASSWORD="${DB_PASSWORD}" pg_dump \
  --host="${DB_HOST}" \
  --port="${DB_PORT}" \
  --username="${DB_USER}" \
  --dbname="${DB_NAME}" \
  --format=custom \
  --no-password \
  --verbose 2>&1 | gzip > "${TEMP_FILE}"

# Kiểm tra file có được tạo ra không
if [ ! -f "${TEMP_FILE}" ]; then
  echo "[$(date)] ERROR: Backup file was not created"
  exit 1
fi

BACKUP_SIZE=$(du -sh "${TEMP_FILE}" | cut -f1)
echo "[$(date)] Backup created: ${BACKUP_FILE} (${BACKUP_SIZE})"

# Upload lên S3
aws s3 cp "${TEMP_FILE}" "s3://${BACKUP_BUCKET}/${BACKUP_PREFIX}/${BACKUP_FILE}" \
  --storage-class STANDARD_IA

echo "[$(date)] Uploaded to S3: s3://${BACKUP_BUCKET}/${BACKUP_PREFIX}/${BACKUP_FILE}"

# Gửi metric về CloudWatch
aws cloudwatch put-metric-data \
  --namespace "ShopLite/Backup" \
  --metric-name "BackupSuccess" \
  --value 1 \
  --unit Count \
  --dimensions Service=PostgreSQL,Environment=production

echo "[$(date)] Backup completed successfully"

# Cleanup backups cũ hơn RETENTION_DAYS ngày
echo "[$(date)] Cleaning up backups older than ${RETENTION_DAYS} days"
CUTOFF_DATE=$(date -d "${RETENTION_DAYS} days ago" +%Y%m%d)

aws s3 ls "s3://${BACKUP_BUCKET}/${BACKUP_PREFIX}/" | \
  awk '{print $4}' | \
  while read -r filename; do
    # Extract date từ filename format: shoplite_YYYYMMDD_HHMMSS.sql.gz
    FILE_DATE=$(echo "${filename}" | grep -oP '\d{8}' | head -1)
    if [ -n "${FILE_DATE}" ] && [ "${FILE_DATE}" -lt "${CUTOFF_DATE}" ]; then
      aws s3 rm "s3://${BACKUP_BUCKET}/${BACKUP_PREFIX}/${filename}"
      echo "[$(date)] Deleted old backup: ${filename}"
    fi
  done

echo "[$(date)] Cleanup completed"
```

**Cài cron job:**

```bash
# Edit crontab
crontab -e

# Backup mỗi giờ (đáp ứng RPO 1 giờ)
0 * * * * /opt/scripts/backup-postgres.sh >> /var/log/backup.log 2>&1

# Backup daily vào 2 giờ sáng
0 2 * * * /opt/scripts/backup-postgres.sh >> /var/log/backup.log 2>&1

# Kiểm tra cron đang chạy
systemctl status cron  # Ubuntu
systemctl status crond  # CentOS/RHEL
```

**Monitoring backup cron:**

```bash
# Kiểm tra backup có chạy thành công không (dùng trong alerting)
#!/bin/bash
# check-backup-health.sh
BACKUP_BUCKET="shoplite-backup"
MAX_AGE_HOURS=2  # alert nếu backup cũ hơn 2 giờ

LATEST_BACKUP=$(aws s3 ls "s3://${BACKUP_BUCKET}/postgres/" | \
  sort | tail -1 | awk '{print $1, $2}')

if [ -z "${LATEST_BACKUP}" ]; then
  echo "CRITICAL: No backups found!"
  exit 2
fi

BACKUP_DATE=$(echo "${LATEST_BACKUP}" | awk '{print $1 "T" $2}')
BACKUP_EPOCH=$(date -d "${BACKUP_DATE}" +%s)
NOW_EPOCH=$(date +%s)
AGE_HOURS=$(( (NOW_EPOCH - BACKUP_EPOCH) / 3600 ))

if [ "${AGE_HOURS}" -gt "${MAX_AGE_HOURS}" ]; then
  echo "WARNING: Latest backup is ${AGE_HOURS} hours old (threshold: ${MAX_AGE_HOURS}h)"
  exit 1
fi

echo "OK: Latest backup is ${AGE_HOURS} hours old (${LATEST_BACKUP})"
exit 0
```

#### 5.4 Test Restore — Bắt buộc phải test định kỳ

**Backup không được test = backup không tồn tại.**

Rất nhiều team có backup nhưng đến khi cần restore thì không được vì:
- File backup bị corrupt
- Quá trình restore có lỗi chưa từng test
- Permissions không đủ để restore
- Database schema thay đổi, backup cũ không compatible

**Script test restore:**

```bash
#!/bin/bash
# test-restore.sh — chạy hàng tuần để kiểm tra backup có restore được không

set -euo pipefail

BACKUP_BUCKET="shoplite-backup"
RESTORE_DB_NAME="shoplite_restore_test"
DB_HOST="localhost"
DB_USER="postgres"

echo "[$(date)] Starting restore test"

# Lấy backup mới nhất
LATEST_BACKUP=$(aws s3 ls "s3://${BACKUP_BUCKET}/postgres/" | \
  sort | tail -1 | awk '{print $4}')

echo "[$(date)] Downloading backup: ${LATEST_BACKUP}"
aws s3 cp "s3://${BACKUP_BUCKET}/postgres/${LATEST_BACKUP}" /tmp/

# Tạo database mới để restore vào
psql -h "${DB_HOST}" -U "${DB_USER}" -c "DROP DATABASE IF EXISTS ${RESTORE_DB_NAME};"
psql -h "${DB_HOST}" -U "${DB_USER}" -c "CREATE DATABASE ${RESTORE_DB_NAME};"

echo "[$(date)] Restoring to ${RESTORE_DB_NAME}"
gunzip -c "/tmp/${LATEST_BACKUP}" | \
  pg_restore --host="${DB_HOST}" --username="${DB_USER}" \
  --dbname="${RESTORE_DB_NAME}" --no-password --verbose

# Verify data integrity — check số lượng records trong các tables quan trọng
echo "[$(date)] Verifying data integrity"
TABLE_COUNTS=$(psql -h "${DB_HOST}" -U "${DB_USER}" -d "${RESTORE_DB_NAME}" \
  -c "SELECT 'users: ' || count(*) FROM users UNION ALL
      SELECT 'orders: ' || count(*) FROM orders UNION ALL
      SELECT 'products: ' || count(*) FROM products;" \
  -t)

echo "[$(date)] Table counts after restore:"
echo "${TABLE_COUNTS}"

# Cleanup
psql -h "${DB_HOST}" -U "${DB_USER}" -c "DROP DATABASE IF EXISTS ${RESTORE_DB_NAME};"
rm -f "/tmp/${LATEST_BACKUP}"

echo "[$(date)] Restore test completed successfully"
```

#### 5.5 Disaster Recovery Plan Template

```
==============================================================
DISASTER RECOVERY PLAN — ShopLite Production
Version: 1.0
Last Updated: 2024-01-15
Owner: DevOps Team
==============================================================

SCENARIO 1: Database Server Failure / Data Corruption
------------------------------------------------------
Trigger: Application cannot connect to database, or data is corrupted.
RPO: 1 hour
RTO: 4 hours

Prerequisites:
- AWS console access
- Database backup bucket: s3://shoplite-backup/postgres/
- AWS Secrets Manager access for new credentials

Steps:
1. [ ] Alert team trong Slack channel #incidents (15 min)
2. [ ] Assess scope: là partial failure hay total loss? (15 min)
3. [ ] Create new RDS instance từ latest automated snapshot:
       aws rds restore-db-instance-from-db-snapshot \
         --db-instance-identifier shoplite-postgres-recovery \
         --db-snapshot-identifier <snapshot-id>
4. [ ] Chờ RDS instance available (15-30 min)
5. [ ] Update DATABASE_URL trong Secrets Manager:
       aws secretsmanager update-secret \
         --secret-id shoplite/prod/database \
         --secret-string '{"host":"new-endpoint",...}'
6. [ ] Restart application pods để pick up new credentials:
       kubectl rollout restart deployment/shoplite-backend -n shoplite
7. [ ] Verify application health: curl health check endpoints
8. [ ] Monitor error rates trong Grafana 30 phút
9. [ ] Communicate resolution tới stakeholders
10. [ ] Schedule post-mortem trong 24 giờ

SCENARIO 2: Kubernetes Cluster Unresponsive
-------------------------------------------
Trigger: kubectl commands timeout, nodes NotReady.
RPO: N/A (stateless application layer)
RTO: 2 hours

Steps:
1. [ ] Check AWS EC2 console: nodes còn running không?
2. [ ] Check EKS cluster status trong AWS console
3. [ ] If node failure: AWS Auto Scaling Group sẽ replace nodes tự động
4. [ ] If cluster control plane issue: contact AWS support
5. [ ] Emergency: deploy trực tiếp lên EC2 bằng Docker Compose

SCENARIO 3: Security Breach Detected
-------------------------------------
Trigger: Unusual API calls trong CloudTrail, GuardDuty alerts.
RPO: N/A
RTO: Immediate containment, recovery 4-8 hours

Steps:
1. [ ] Isolate affected resources: deny all traffic bằng Security Group
2. [ ] Rotate tất cả credentials: database passwords, API keys, JWT secrets
3. [ ] Forensic snapshot: snapshot EBS volumes trước khi terminate instances
4. [ ] Replace compromised instances với fresh AMI
5. [ ] Review CloudTrail logs để xác định scope của breach
6. [ ] Notify legal/compliance nếu có customer data exposed
7. [ ] Post-mortem và security review toàn diện

Contacts:
- DevOps On-call: PagerDuty escalation policy "shoplite-production"
- AWS Support: Business/Enterprise plan, phone support 24/7
- Security Incident: security@company.com
==============================================================
```

---

### 6. High Availability — Độ Sẵn Sàng Cao

#### 6.1 RDS Multi-AZ

Multi-AZ RDS duy trì một standby replica trong Availability Zone khác. Khi primary AZ fails, AWS tự động failover sang standby (khoảng 60 giây).

```bash
# Bật Multi-AZ cho RDS instance hiện có
aws rds modify-db-instance \
  --db-instance-identifier shoplite-postgres \
  --multi-az \
  --apply-immediately

# Tạo RDS mới với Multi-AZ từ đầu (trong Terraform)
```

```hcl
# terraform/rds.tf
resource "aws_db_instance" "postgres" {
  identifier        = "shoplite-postgres"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.t3.medium"
  allocated_storage = 20
  storage_encrypted = true

  db_name  = "shoplite"
  username = "shoplite_admin"
  password = random_password.db_password.result

  multi_az               = true  # High Availability
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name

  backup_retention_period   = 7
  backup_window             = "02:00-03:00"
  maintenance_window        = "Mon:03:00-Mon:04:00"
  deletion_protection       = true  # prevent accidental deletion
  skip_final_snapshot       = false
  final_snapshot_identifier = "shoplite-final-snapshot"

  performance_insights_enabled = true
  monitoring_interval          = 60

  tags = {
    Name        = "shoplite-postgres"
    Environment = "production"
    ManagedBy   = "terraform"
  }
}
```

#### 6.2 Multiple Replicas với Pod Disruption Budget

**Minimum 3 replicas cho production services:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shoplite-backend
  namespace: shoplite
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # tối đa 1 pod unavailable trong rolling update
      maxSurge: 1          # tối đa 1 pod extra trong rolling update
```

**Pod Disruption Budget — đảm bảo minimum availability trong cluster maintenance:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: shoplite-backend-pdb
  namespace: shoplite
spec:
  # Tối đa 1 pod bị disrupt cùng lúc (ví dụ: node drain)
  maxUnavailable: 1
  selector:
    matchLabels:
      app: shoplite
      tier: backend
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: shoplite-frontend-pdb
  namespace: shoplite
spec:
  # Tối thiểu 2 pods phải luôn available
  minAvailable: 2
  selector:
    matchLabels:
      app: shoplite
      tier: frontend
```

#### 6.3 Pod Anti-Affinity — Spread Pods Across Nodes

Nếu 3 pods của bạn đều chạy trên cùng 1 node, node đó down là toàn bộ service down.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shoplite-backend
  namespace: shoplite
spec:
  replicas: 3
  template:
    spec:
      affinity:
        podAntiAffinity:
          # Hard rule: KHÔNG schedule 2 pods cùng app/tier lên cùng 1 node
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: shoplite
                  tier: backend
              topologyKey: kubernetes.io/hostname
          # Soft rule: ưu tiên spread across AZs
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: shoplite
                    tier: backend
                topologyKey: topology.kubernetes.io/zone
```

**Kiểm tra pods đang chạy trên nodes nào:**

```bash
kubectl get pods -n shoplite -o wide \
  -l app=shoplite,tier=backend
```

#### 6.4 Topology Spread Constraints (K8s 1.19+)

Cách hiện đại hơn để spread pods:

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1  # chênh lệch tối đa giữa các zones
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule  # hard constraint
      labelSelector:
        matchLabels:
          app: shoplite
          tier: backend
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway  # soft constraint
      labelSelector:
        matchLabels:
          app: shoplite
          tier: backend
```

---

### 7. IAM Hardening

#### 7.1 EC2 Instance Role Thay Vì Access Keys

**Sai lầm phổ biến:**

```bash
# KHÔNG làm thế này trên EC2
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

Access keys lưu trong EC2 có thể bị lộ qua metadata service (CVE dạng SSRF), config files, logs.

**Dùng IAM Instance Role:**

```bash
# Tạo trust policy cho EC2
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Tạo IAM role
aws iam create-role \
  --role-name shoplite-ec2-role \
  --assume-role-policy-document file://trust-policy.json \
  --description "Role for ShopLite EC2 instances"

# Tạo inline policy với quyền tối thiểu
cat > shoplite-ec2-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:ap-southeast-1:*:secret:shoplite/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData",
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::shoplite-backup",
        "arn:aws:s3:::shoplite-backup/*"
      ]
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name shoplite-ec2-role \
  --policy-name shoplite-ec2-policy \
  --policy-document file://shoplite-ec2-policy.json

# Tạo instance profile và gắn role
aws iam create-instance-profile \
  --instance-profile-name shoplite-ec2-profile

aws iam add-role-to-instance-profile \
  --instance-profile-name shoplite-ec2-profile \
  --role-name shoplite-ec2-role

# Gắn vào EC2 instance
aws ec2 associate-iam-instance-profile \
  --instance-id i-1234567890abcdef0 \
  --iam-instance-profile Name=shoplite-ec2-profile
```

#### 7.2 IRSA — IAM Roles for Service Accounts trong EKS

IRSA cho phép Kubernetes pods assume IAM role mà không cần access keys. Đây là cách recommended cho EKS.

```bash
# Bật OIDC provider cho EKS cluster
eksctl utils associate-iam-oidc-provider \
  --region ap-southeast-1 \
  --cluster shoplite-cluster \
  --approve

# Tạo IAM role cho service account
eksctl create iamserviceaccount \
  --name shoplite-backend-sa \
  --namespace shoplite \
  --cluster shoplite-cluster \
  --role-name shoplite-backend-irsa \
  --attach-policy-arn arn:aws:iam::123456789:policy/ShopliteBackendPolicy \
  --approve

# Trong Kubernetes Deployment — annotate service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: shoplite-backend-sa
  namespace: shoplite
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/shoplite-backend-irsa
```

Pods dùng service account này sẽ tự động có credentials để assume IAM role, không cần set environment variables.

#### 7.3 AWS Config Rules

AWS Config tự động kiểm tra và alert khi resources vi phạm security best practices:

```bash
# Kích hoạt AWS Config
aws configservice start-configuration-recorder \
  --configuration-recorder-name default

# Enable conformance pack với security rules
aws configservice put-conformance-pack \
  --conformance-pack-name SecurityBestPractices \
  --template-s3-uri s3://shoplite-config/security-conformance-pack.yaml

# Một số managed rules hữu ích
# s3-bucket-public-read-prohibited: S3 bucket không được public read
# rds-instance-public-access-check: RDS không được public accessible
# restricted-ssh: SSH chỉ từ approved IP ranges
# iam-password-policy: Password policy đủ mạnh
# mfa-enabled-for-iam-console-access: MFA required cho console access
# root-account-mfa-enabled: Root account phải bật MFA
```

#### 7.4 CloudTrail Audit Logging

```bash
# Tạo CloudTrail trail
aws cloudtrail create-trail \
  --name shoplite-audit-trail \
  --s3-bucket-name shoplite-cloudtrail \
  --is-multi-region-trail \
  --include-global-service-events \
  --enable-log-file-validation

# Bật trail
aws cloudtrail start-logging \
  --name shoplite-audit-trail

# Query CloudTrail với Athena để tìm suspicious activity
# Ví dụ: ai đã DeleteSecurityGroup trong 24h qua?
SELECT userIdentity.userName,
       eventTime,
       eventName,
       requestParameters
FROM cloudtrail_logs
WHERE eventName = 'DeleteSecurityGroup'
  AND eventTime > date_sub(now(), interval 1 day)
ORDER BY eventTime DESC;
```

---

### 8. Network Security

#### 8.1 Kubernetes NetworkPolicy

Mặc định trong Kubernetes, tất cả pods có thể giao tiếp với nhau (flat network). NetworkPolicy cho phép định nghĩa rules để giới hạn traffic.

**Default deny-all policy — ngăn tất cả traffic, sau đó mở dần:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: shoplite
spec:
  podSelector: {}  # áp dụng cho tất cả pods trong namespace
  policyTypes:
    - Ingress
    - Egress
  # Không có ingress hay egress rules = deny all
```

**Allow backend pods nhận traffic từ frontend:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: shoplite
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - protocol: TCP
          port: 3000
    - from:
        - podSelector:
            matchLabels:
              app: prometheus  # allow Prometheus scraping
      ports:
        - protocol: TCP
          port: 9090
```

**Full NetworkPolicy cho ShopLite backend:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: shoplite-backend-netpol
  namespace: shoplite
spec:
  podSelector:
    matchLabels:
      app: shoplite
      tier: backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Nhận từ frontend
    - from:
        - podSelector:
            matchLabels:
              app: shoplite
              tier: frontend
      ports:
        - protocol: TCP
          port: 3000
    # Nhận từ Ingress controller
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 3000
  egress:
    # Kết nối database trong cùng namespace
    - to:
        - podSelector:
            matchLabels:
              app: shoplite
              tier: database
      ports:
        - protocol: TCP
          port: 5432
    # Kết nối Redis cache
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    # Kết nối DNS (cần thiết cho name resolution)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # HTTPS ra ngoài (gọi third-party APIs: Stripe, email...)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8     # private networks
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443
```

**Kiểm tra NetworkPolicy có hoạt động không:**

```bash
# Thử kết nối từ pod không được phép
kubectl exec -it test-pod -n shoplite -- \
  curl -m 3 http://shoplite-backend:3000/health

# Nếu NetworkPolicy đúng, kết nối sẽ timeout
# Dùng netpol-tester để visualize
kubectl apply -f https://raw.githubusercontent.com/ahmetb/kubernetes-network-policy-recipes/master/netpol-tester.yaml
```

#### 8.2 Ingress với Rate Limiting và WAF

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shoplite-ingress
  namespace: shoplite
  annotations:
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"          # 10 requests/sec per IP
    nginx.ingress.kubernetes.io/limit-connections: "20"  # 20 connections per IP

    # Security headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
      more_set_headers "Strict-Transport-Security: max-age=31536000; includeSubDomains";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";
      more_set_headers "Content-Security-Policy: default-src 'self'";

    # SSL
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - shoplite.example.com
      secretName: shoplite-tls
  rules:
    - host: shoplite.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shoplite-frontend
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: shoplite-backend
                port:
                  number: 3000
```

---

### 9. DevSecOps — Bảo Mật Trong Pipeline

#### 9.1 SAST với Semgrep

SAST (Static Application Security Testing) phân tích code để tìm vulnerabilities mà không cần chạy ứng dụng.

```bash
# Cài Semgrep
pip install semgrep
# hoặc
brew install semgrep

# Scan với ruleset tự động (community rules)
semgrep --config auto .

# Scan với rules cụ thể cho .NET
semgrep --config "p/owasp-top-ten" .
semgrep --config "p/csharp" .
semgrep --config "p/docker" .

# Fail nếu tìm thấy issues
semgrep --config auto --error .

# Output JSON để parse trong CI
semgrep --config auto --json > semgrep-results.json
```

**Custom Semgrep rules cho ShopLite (.NET):**

```yaml
# .semgrep/shoplite-rules.yaml
rules:
  - id: no-hardcoded-connection-string
    patterns:
      - pattern: |
          new SqlConnection("...password=...")
      - pattern: |
          "Server=...;Password=..."
    message: "Potential hardcoded connection string detected"
    severity: ERROR
    languages: [csharp]

  - id: sql-injection-risk-csharp
    patterns:
      - pattern: |
          new SqlCommand("..." + $USER_INPUT, ...)
      - pattern: |
          $CMD.CommandText = "..." + $USER_INPUT
    message: "Potential SQL injection - use parameterized queries or EF Core"
    severity: ERROR
    languages: [csharp]

  - id: weak-password-hashing
    patterns:
      - pattern: |
          MD5.HashData(...)
      - pattern: |
          SHA1.HashData(...)
    message: "Weak hashing for passwords - use BCrypt or ASP.NET Identity"
    severity: WARNING
    languages: [csharp]
```

#### 9.2 Dependency Scanning

```bash
# .NET — kiểm tra NuGet packages có CVE
dotnet list package --vulnerable                  # hiện packages có lỗ hổng đã biết
dotnet list package --vulnerable --include-transitive  # bao gồm cả transitive deps
dotnet list package --outdated                    # kiểm tra packages lỗi thời

# Exit code khác 0 nếu có vulnerabilities — dùng trong CI
dotnet list package --vulnerable 2>&1 | grep -i "has the following vulnerable packages"
if [ $? -eq 0 ]; then echo "Vulnerable packages found!"; exit 1; fi

# OWASP Dependency-Check (mạnh hơn, check NVD database)
docker run --rm \
  -v "$(pwd):/src" \
  -v "$(pwd)/reports:/report" \
  owasp/dependency-check:latest \
  --scan /src \
  --format HTML --format JSON \
  --out /report \
  --failOnCVSS 7

# Snyk (requires account — hỗ trợ .NET tốt)
dotnet tool install -g snyk
snyk test --severity-threshold=high   # scan
snyk monitor                          # monitor ongoing
```

**Trong GitHub Actions:**

```yaml
- name: Setup .NET
  uses: actions/setup-dotnet@v4
  with:
    dotnet-version: '8.0.x'

- name: Restore dependencies
  run: dotnet restore
  working-directory: ./backend

- name: Check for vulnerable packages
  run: |
    dotnet list package --vulnerable --include-transitive 2>&1 | tee vuln-report.txt
    if grep -q "has the following vulnerable packages" vuln-report.txt; then
      echo "FAIL: Vulnerable NuGet packages found"
      cat vuln-report.txt
      exit 1
    fi
  working-directory: ./backend

- name: Snyk security scan
  uses: snyk/actions/dotnet@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high
```

#### 9.3 IaC Security Scanning với Checkov

```bash
# Cài checkov
pip install checkov

# Scan Terraform
checkov -d ./terraform --framework terraform

# Scan Kubernetes manifests
checkov -d ./k8s --framework kubernetes

# Scan Dockerfile
checkov -f ./Dockerfile --framework dockerfile

# Fail nếu có issues
checkov -d . --hard-fail-on HIGH,CRITICAL

# Output dạng SARIF
checkov -d . --output sarif --output-file-path ./checkov.sarif
```

#### 9.4 Full Security Pipeline trong GitHub Actions

```yaml
name: Security Pipeline

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  secret-detection:
    name: Secret Detection
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # full history để scan
      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  sast:
    name: SAST Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/owasp-top-ten
            p/javascript
            p/docker

  dependency-scan:
    name: Dependency Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      - run: dotnet restore
        working-directory: ./backend
      - name: Check for vulnerable packages
        run: |
          dotnet list package --vulnerable --include-transitive 2>&1 | tee vuln-report.txt
          if grep -q "has the following vulnerable packages" vuln-report.txt; then
            echo "FAIL: Vulnerable NuGet packages found"; exit 1
          fi
        working-directory: ./backend

  iac-scan:
    name: IaC Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: kubernetes,dockerfile,terraform
          soft_fail: false

  container-scan:
    name: Container Scan
    runs-on: ubuntu-latest
    needs: []
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t shoplite-backend:${{ github.sha }} ./backend
      - name: Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: shoplite-backend:${{ github.sha }}
          exit-code: 1
          severity: CRITICAL,HIGH
          ignore-unfixed: true
```

---

### 10. Incident Response — Xử Lý Sự Cố

#### 10.1 Quy Trình Khi Nhận Alert 2 Giờ Sáng

**Bước 1: Acknowledge và triage (5 phút đầu)**

```bash
# Acknowledge alert trong PagerDuty/Alertmanager để stop escalation

# Kiểm tra tổng quan
kubectl get pods -n shoplite --all
kubectl get nodes
kubectl top nodes
kubectl top pods -n shoplite
```

**Bước 2: Identify scope (5-10 phút)**

```bash
# Pods có healthy không?
kubectl get pods -n shoplite
# CrashLoopBackOff, ImagePullBackOff, OOMKilled, Pending?

# Services có endpoint không?
kubectl get endpoints -n shoplite

# Ingress có đúng không?
kubectl describe ingress shoplite-ingress -n shoplite

# Xem logs của pod đang lỗi
kubectl logs deployment/shoplite-backend -n shoplite --tail=100
kubectl logs deployment/shoplite-backend -n shoplite --previous  # logs của container trước khi restart

# Describe pod để xem events
kubectl describe pod <pod-name> -n shoplite
```

**Bước 3: Check recent changes (5 phút)**

```bash
# Có deployment nào gần đây không?
kubectl rollout history deployment/shoplite-backend -n shoplite

# Xem diff của revision mới nhất vs revision trước
kubectl rollout history deployment/shoplite-backend -n shoplite --revision=5
kubectl rollout history deployment/shoplite-backend -n shoplite --revision=4

# Rollback nếu deployment gây ra issue
kubectl rollout undo deployment/shoplite-backend -n shoplite
# hoặc rollback về revision cụ thể
kubectl rollout undo deployment/shoplite-backend -n shoplite --to-revision=3

# Theo dõi rollback
kubectl rollout status deployment/shoplite-backend -n shoplite
```

**Bước 4: Database issues**

```bash
# Kiểm tra kết nối database
kubectl exec -it deployment/shoplite-backend -n shoplite -- \
  sh -c "pg_isready -h $DB_HOST -p $DB_PORT"

# Số lượng connections hiện tại
kubectl exec -it deployment/shoplite-backend -n shoplite -- \
  psql $DATABASE_URL -c "SELECT count(*) FROM pg_stat_activity;"

# Connections đang chờ lock
kubectl exec -it deployment/shoplite-backend -n shoplite -- \
  psql $DATABASE_URL -c "SELECT pid, state, wait_event_type, query FROM pg_stat_activity WHERE wait_event IS NOT NULL;"
```

**Bước 5: Resource issues**

```bash
# OOMKilled — memory limit quá thấp
kubectl describe pod <pod-name> -n shoplite | grep -A5 "OOM"

# Tăng memory limit tạm thời
kubectl set resources deployment/shoplite-backend -n shoplite \
  --limits=memory=512Mi --requests=memory=256Mi

# Node disk pressure
kubectl describe node <node-name> | grep -A5 "Conditions"

# Disk usage trên node
kubectl debug node/<node-name> -it --image=ubuntu -- \
  df -h
```

#### 10.2 Communication Template

```
INCIDENT REPORT — [Date Time]

Status: INVESTIGATING | IDENTIFIED | MONITORING | RESOLVED

Impact:
- Service: ShopLite Production
- Affected users: [estimate]
- Affected functionality: [what's broken]

Timeline:
- HH:MM - Alert triggered
- HH:MM - On-call engineer acknowledged
- HH:MM - Root cause identified: [brief description]
- HH:MM - Fix applied: [what was done]
- HH:MM - Service restored

Root Cause: [one sentence]

Resolution: [one sentence]

Next Steps:
- [ ] Post-mortem scheduled for [date]
- [ ] Preventive measures: [list]
```

#### 10.3 Post-mortem Blameless

Post-mortem (hay còn gọi là incident review) phải blameless — không tìm người để đổ lỗi mà tìm hệ thống để cải thiện.

**Template:**

```markdown
# Post-mortem: [Incident Title]
Date: [date]
Duration: [start time] - [end time] ([total duration])
Severity: P1/P2/P3/P4
On-call: [name]

## Summary
[2-3 câu mô tả incident và impact]

## Impact
- Users affected: ~X users
- Revenue impact: ~$X lost orders
- Duration of degraded service: X minutes

## Timeline
| Time | Event |
|------|-------|
| 02:13 | Alert fired: backend-high-error-rate |
| 02:17 | On-call acknowledged |
| 02:25 | Identified cause: OOM kills |
| 02:31 | Applied fix: increased memory limits |
| 02:35 | Service restored to normal |

## Root Cause Analysis (5 Whys)
- Why did service go down? Backend pods were OOMKilled.
- Why were pods OOMKilled? Memory usage exceeded 256Mi limit.
- Why did memory usage spike? New feature loaded entire product catalog into memory.
- Why was catalog loaded into memory? No pagination implemented in product query.
- Why was this not caught? Performance testing did not include large catalog scenarios.

## What Went Well
- Alert fired within 1 minute of issue start
- On-call response time was 4 minutes
- Rollback option was available and clear

## What Could Be Improved
- Memory profiling should be part of code review for new features
- Load testing should include production-scale data volumes
- Memory limit should have more headroom (256Mi -> 512Mi)

## Action Items
| Action | Owner | Due Date |
|--------|-------|----------|
| Add pagination to product list API | Backend Team | 2024-01-22 |
| Increase memory limits to 512Mi | DevOps | 2024-01-16 |
| Add load tests with 10k products | QA Team | 2024-01-25 |
| Add memory trend alert (>80% for 5min) | DevOps | 2024-01-18 |
```

---

## Kỹ năng đạt được
- Thiết lập backup tự động và quy trình phục hồi.
- Quét và xử lý lỗ hổng bảo mật.
- Hardening hạ tầng theo best practice production.

---

## Thực hành

**Môi trường:** AWS + K8s + CI/CD (toàn bộ stack đã có).

**Lab:**
- Thiết lập backup tự động cho database (snapshot/cron) + test restore.
- Tích hợp Trivy quét image trong CI, chặn image có lỗ hổng nghiêm trọng.
- Chuyển secrets sang công cụ quản lý bí mật.
- Cấu hình HA cho database (multi-AZ) và nhiều replica cho service.
- Rà soát và siết quyền IAM theo least privilege.

**Công cụ:** Trivy, AWS Secrets Manager/Vault, RDS backup, network policy.

---

## Bài tập

- **Bắt buộc:** Backup database tự động + chứng minh restore thành công.
- **Nâng cao:** Pipeline tự động fail khi image có lỗ hổng nghiêm trọng.
- **Mô phỏng doanh nghiệp:** Viết "Disaster Recovery Plan" — kịch bản khi database mất, các bước phục hồi và thời gian mục tiêu (RTO/RPO).

---

## Deliverable
- [ ] Cơ chế backup/restore hoạt động + tài liệu DR.
- [ ] Image scanning trong CI/CD.
- [ ] Secrets được quản lý an toàn, IAM theo least privilege.

---

## Tiêu chí hoàn thành
- [ ] Mất dữ liệu giả lập → phục hồi được từ backup.
- [ ] Pipeline chặn được image có lỗ hổng.
- [ ] Không còn secret nào hardcode trong code/repo.

---

## Ghi chú học tập

> _Ghi lại những điều học được, vướng mắc và insight trong quá trình học phase này._
