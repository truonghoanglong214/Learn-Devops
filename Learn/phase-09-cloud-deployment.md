# Phase 9 — Cloud Deployment

## Mục tiêu
- Đưa ShopLite ra Internet thật trên hạ tầng cloud.
- Hiểu các dịch vụ cloud nền tảng và mô hình chi phí.

---

## Kiến thức sẽ học

- **Khái niệm Cloud & mô hình dịch vụ** (IaaS/PaaS/SaaS, regions, AZ).
- **AWS cơ bản:**
  - **EC2** (máy ảo cloud) — nơi chạy ứng dụng.
  - **VPC, Subnet, Security Group, Internet Gateway** — mạng trên cloud.
  - **RDS** (PostgreSQL managed) — database không cần tự quản.
  - **S3** (object storage) — lưu file tĩnh/ảnh sản phẩm/backup.
  - **IAM** (user, role, policy) — phân quyền, bảo mật.
  - **Route 53** (DNS) — gắn domain.
- **Quản lý chi phí & Free Tier** — học mà không tốn nhiều tiền.

---

## Kiến thức chi tiết

### 1. Cloud Fundamentals — Nền tảng điện toán đám mây

#### Các mô hình dịch vụ cloud

**IaaS — Infrastructure as a Service**

IaaS là mô hình mà nhà cung cấp cloud cung cấp hạ tầng ảo hóa (máy chủ, mạng, storage) và bạn chịu trách nhiệm quản lý mọi thứ từ hệ điều hành trở lên.

Ví dụ AWS IaaS: EC2, VPC, EBS (Elastic Block Store).

Bạn quản lý:
- Hệ điều hành (Ubuntu, Amazon Linux)
- Runtime (Node.js, Python, Java)
- Middleware (Nginx, Docker)
- Application code
- Data

AWS quản lý:
- Physical servers
- Networking hardware
- Hypervisor
- Data center security

Khi nào dùng IaaS:
- Cần toàn quyền kiểm soát hệ thống
- Ứng dụng có yêu cầu đặc biệt về OS
- Muốn chạy Docker/Kubernetes tự quản
- Migrate từ on-premise lên cloud mà không thay đổi nhiều

**PaaS — Platform as a Service**

PaaS là mô hình mà nhà cung cấp quản lý hạ tầng + OS + runtime, bạn chỉ cần deploy code.

Ví dụ AWS PaaS: Elastic Beanstalk, App Runner.
Ví dụ bên ngoài: Heroku, Railway, Render.

Bạn quản lý:
- Application code
- Data
- Cấu hình ứng dụng

AWS quản lý tất cả còn lại.

Khi nào dùng PaaS:
- Startup muốn deploy nhanh, không có DevOps chuyên sâu
- Ứng dụng nhỏ-vừa, không cần custom OS
- Giảm operational overhead

**SaaS — Software as a Service**

SaaS là phần mềm hoàn chỉnh chạy trên cloud, bạn chỉ sử dụng qua browser/API.

Ví dụ: Gmail, Slack, GitHub, Jira, Salesforce.

Bạn không quản lý gì cả, chỉ dùng.

#### Shared Responsibility Model — Mô hình trách nhiệm chia sẻ

Đây là khái niệm **cực kỳ quan trọng** trong bảo mật cloud.

AWS chịu trách nhiệm **"Security OF the Cloud"**:
- Physical data centers (khóa, camera, bảo vệ)
- Networking infrastructure
- Hypervisor và virtualization layer
- Managed service software (RDS engine, S3 storage layer)

Bạn chịu trách nhiệm **"Security IN the Cloud"**:
- Data encryption (at rest + in transit)
- Identity and Access Management (IAM)
- Operating system patches (EC2)
- Application security
- Network configuration (Security Groups, NACLs)
- Firewall rules

Ví dụ thực tế với ShopLite:
- Nếu AWS data center bị hack -> AWS chịu trách nhiệm
- Nếu bạn để EC2 mở port 22 cho 0.0.0.0/0 và bị brute force -> bạn chịu trách nhiệm
- Nếu RDS bị SQL injection vì code của bạn -> bạn chịu trách nhiệm

#### AWS Global Infrastructure

**Regions — Vùng địa lý**

AWS có hơn 30 regions trên toàn cầu. Mỗi region là một tập hợp data center độc lập ở một vị trí địa lý.

Các regions quan trọng:
- `us-east-1` (N. Virginia) — region lớn nhất, nhiều dịch vụ mới ra đây trước
- `us-west-2` (Oregon) — phổ biến cho startup
- `eu-west-1` (Ireland) — cho châu Âu
- `ap-southeast-1` (Singapore) — **gần Việt Nam nhất**, latency khoảng 20-30ms
- `ap-northeast-1` (Tokyo) — Nhật Bản, latency khoảng 50-70ms từ VN

Cho ShopLite target người dùng Việt Nam: chọn `ap-southeast-1`.

**Availability Zones (AZ)**

Mỗi region có từ 2-6 AZ. AZ là một hoặc nhiều data center vật lý độc lập nhau về điện, mạng, làm mát.

- ap-southeast-1 có 3 AZ: ap-southeast-1a, ap-southeast-1b, ap-southeast-1c
- Cùng region -> latency < 1ms giữa các AZ
- Nếu một AZ gặp sự cố (hỏa hoạn, mất điện), các AZ khác vẫn hoạt động

Multi-AZ deployment: deploy ứng dụng ra nhiều AZ để tăng high availability.

**AWS Free Tier**

Ba loại Free Tier:
1. **12 tháng miễn phí** (kể từ ngày tạo tài khoản):
   - EC2: t2.micro 750 giờ/tháng
   - RDS: db.t2.micro 750 giờ/tháng, 20GB storage
   - S3: 5GB storage, 20,000 GET requests, 2,000 PUT requests
   - Data transfer: 15GB/tháng egress

2. **Always Free** (không giới hạn thời gian):
   - Lambda: 1 triệu requests/tháng, 400,000 GB-seconds compute
   - DynamoDB: 25GB storage, 25 WCU, 25 RCU
   - CloudWatch: 10 custom metrics, 10 alarms
   - SNS: 1 triệu publishes

3. **Free Trial** (thời gian cố định khi bật dịch vụ):
   - Inspector: 90 ngày free

Lưu ý quan trọng: Free Tier áp dụng per account, không phải per region. Nếu bạn chạy t2.micro ở cả us-east-1 và ap-southeast-1, tổng cộng vẫn chỉ free 750 giờ/tháng.

---

### 2. IAM — Identity and Access Management

IAM là dịch vụ quản lý quyền truy cập cho AWS. **Đây là thứ đầu tiên phải học** vì ảnh hưởng đến toàn bộ bảo mật cloud của bạn.

#### Root Account — Tài khoản gốc

Root account là tài khoản tạo bằng email khi đăng ký AWS. Root account có toàn quyền với mọi thứ và không thể bị giới hạn bởi IAM policy.

**Nguyên tắc bắt buộc:**
1. Bật MFA (Multi-Factor Authentication) cho root account ngay lập tức
2. Tạo IAM User với quyền Admin để dùng hàng ngày
3. Không tạo access key cho root account
4. Chỉ dùng root account cho các tác vụ bắt buộc (ví dụ: close account, change support plan)

Bật MFA: AWS Console -> Security Credentials -> MFA -> Assign MFA device -> Virtual MFA (Google Authenticator)

#### IAM Users — Người dùng

IAM User đại diện cho một người hoặc application. Có thể có:
- **Console password**: đăng nhập AWS Console
- **Access Key ID + Secret Access Key**: dùng với AWS CLI/SDK

Nguyên tắc least privilege: chỉ cấp quyền tối thiểu cần thiết.

Không tạo access key cho EC2 instance — dùng IAM Role thay thế.

Tạo admin user bằng CLI:
```bash
# Tạo user
aws iam create-user --user-name admin-phung

# Gán policy AdministratorAccess
aws iam attach-user-policy \
  --user-name admin-phung \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Tạo access key
aws iam create-access-key --user-name admin-phung
# Output: AccessKeyId và SecretAccessKey — lưu lại ngay, không xem lại được
```

#### IAM Groups — Nhóm người dùng

Group giúp quản lý policy cho nhiều user cùng lúc.

```bash
# Tạo group
aws iam create-group --group-name developers

# Gán policy cho group
aws iam attach-group-policy \
  --group-name developers \
  --policy-arn arn:aws:iam::aws:policy/PowerUserAccess

# Thêm user vào group
aws iam add-user-to-group --group-name developers --user-name phung
```

#### IAM Roles — Vai trò

Role là identity mà AWS services hoặc users có thể assume tạm thời. Khác với User, Role không có long-term credentials — nó tự động rotate credentials ngắn hạn.

Use cases chính:
- **EC2 Instance Profile**: EC2 instance assume role để gọi S3, RDS, SSM
- **Lambda Execution Role**: Lambda function cần quyền gọi DynamoDB
- **Cross-account access**: cho phép account A access resource của account B
- **Federated access**: SSO với corporate identity provider

Tạo role cho EC2 để access S3:
```bash
# Tạo trust policy (ai được phép assume role này)
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Tạo role
aws iam create-role \
  --role-name shoplite-ec2-role \
  --assume-role-policy-document file://trust-policy.json

# Gán policy S3 access
aws iam attach-role-policy \
  --role-name shoplite-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# Tạo instance profile (wrapper để gán role cho EC2)
aws iam create-instance-profile --instance-profile-name shoplite-ec2-profile
aws iam add-role-to-instance-profile \
  --instance-profile-name shoplite-ec2-profile \
  --role-name shoplite-ec2-role
```

#### IAM Policies — Chính sách phân quyền

Policy là JSON document định nghĩa permissions. Có hai loại:
- **AWS Managed Policy**: AWS tạo sẵn, ví dụ `AmazonS3FullAccess`
- **Customer Managed Policy**: bạn tự tạo, linh hoạt hơn

Cấu trúc policy document:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3BucketAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::shoplite-assets/*"
    },
    {
      "Sid": "AllowS3ListBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::shoplite-assets"
    }
  ]
}
```

Giải thích các trường:
- `Version`: luôn để "2012-10-17"
- `Sid`: Statement ID, tùy chọn, mô tả statement
- `Effect`: "Allow" hoặc "Deny" — Deny luôn override Allow
- `Action`: danh sách actions, dùng wildcard `*` (ví dụ `s3:*`)
- `Resource`: ARN của resource, `*` cho tất cả
- `Condition`: điều kiện tùy chọn (ví dụ chỉ cho phép từ IP cụ thể)

ARN (Amazon Resource Name) format:
`arn:partition:service:region:account-id:resource`

Ví dụ:
- `arn:aws:s3:::shoplite-assets` — S3 bucket
- `arn:aws:ec2:ap-southeast-1:123456789:instance/i-xxxx` — EC2 instance
- `arn:aws:iam::123456789:user/phung` — IAM user

Tạo custom policy:
```bash
aws iam create-policy \
  --policy-name ShopLiteS3Access \
  --policy-document file://s3-policy.json

# Gán cho role
aws iam attach-role-policy \
  --role-name shoplite-ec2-role \
  --policy-arn arn:aws:iam::123456789012:policy/ShopLiteS3Access
```

#### Cấu hình AWS CLI

Cài đặt:
```bash
# Linux/Mac
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Verify
aws --version
```

Cấu hình credentials:
```bash
aws configure
# AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name [None]: ap-southeast-1
# Default output format [None]: json
```

Credentials được lưu tại `~/.aws/credentials` và `~/.aws/config`.

Multiple profiles:
```bash
aws configure --profile production
aws configure --profile staging

# Dùng profile cụ thể
aws s3 ls --profile production
export AWS_PROFILE=production  # Set default cho session
```

Verify credentials:
```bash
aws sts get-caller-identity
# Output:
# {
#   "UserId": "AIDAIOSFODNN7EXAMPLE",
#   "Account": "123456789012",
#   "Arn": "arn:aws:iam::123456789012:user/admin-phung"
# }
```

---

### 3. VPC — Virtual Private Cloud

VPC là mạng ảo riêng của bạn trên AWS. Mọi resource (EC2, RDS, Lambda) đều nằm trong một VPC.

#### Khái niệm cơ bản

**CIDR Block**: địa chỉ IP range cho VPC.
- `10.0.0.0/16`: từ 10.0.0.0 đến 10.0.255.255, tổng 65,536 IPs
- `/16`: phổ biến cho VPC
- `/24`: phổ biến cho subnet (256 IPs, thực tế dùng được 251 vì AWS reserve 5)

AWS reserve 5 IPs đầu và cuối của mỗi subnet:
- `.0`: Network address
- `.1`: VPC router
- `.2`: DNS server
- `.3`: Reserved cho tương lai
- `.255`: Broadcast address

**Default VPC**: AWS tự tạo một VPC mặc định ở mỗi region. Không nên dùng cho production vì:
- Tất cả subnets đều public
- Không kiểm soát được cấu hình mạng

#### Thiết kế mạng cho ShopLite (Multi-AZ)

```
VPC: 10.0.0.0/16 (shoplite-vpc)
├── Public Subnet AZ-a: 10.0.1.0/24 (shoplite-public-a)
│   └── EC2 Web Server (Nginx + App)
├── Public Subnet AZ-b: 10.0.2.0/24 (shoplite-public-b)
│   └── EC2 backup / Load Balancer
├── Private Subnet AZ-a: 10.0.11.0/24 (shoplite-private-a)
│   └── RDS PostgreSQL Primary
└── Private Subnet AZ-b: 10.0.12.0/24 (shoplite-private-b)
    └── RDS PostgreSQL Standby (Multi-AZ)
```

Tạo VPC bằng CLI:
```bash
# Tạo VPC
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --query "Vpc.VpcId" --output text)

# Đặt tên
aws ec2 create-tags \
  --resources $VPC_ID \
  --tags Key=Name,Value=shoplite-vpc

# Bật DNS hostname (cần thiết cho RDS)
aws ec2 modify-vpc-attribute \
  --vpc-id $VPC_ID \
  --enable-dns-hostnames

# Tạo public subnets
SUBNET_PUB_A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone ap-southeast-1a \
  --query "Subnet.SubnetId" --output text)

SUBNET_PUB_B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 \
  --availability-zone ap-southeast-1b \
  --query "Subnet.SubnetId" --output text)

# Tạo private subnets
SUBNET_PRIV_A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.11.0/24 \
  --availability-zone ap-southeast-1a \
  --query "Subnet.SubnetId" --output text)

SUBNET_PRIV_B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.12.0/24 \
  --availability-zone ap-southeast-1b \
  --query "Subnet.SubnetId" --output text)
```

#### Internet Gateway và Route Tables

Internet Gateway (IGW) là gateway cho phép traffic vào/ra Internet từ VPC.

```bash
# Tạo Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --query "InternetGateway.InternetGatewayId" --output text)

# Gán vào VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID

# Tạo Route Table cho public subnets
RTB_PUBLIC=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query "RouteTable.RouteTableId" --output text)

# Thêm route 0.0.0.0/0 -> IGW
aws ec2 create-route \
  --route-table-id $RTB_PUBLIC \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID

# Gán route table cho public subnets
aws ec2 associate-route-table \
  --route-table-id $RTB_PUBLIC \
  --subnet-id $SUBNET_PUB_A

aws ec2 associate-route-table \
  --route-table-id $RTB_PUBLIC \
  --subnet-id $SUBNET_PUB_B

# Bật auto-assign public IP cho public subnets
aws ec2 modify-subnet-attribute \
  --subnet-id $SUBNET_PUB_A \
  --map-public-ip-on-launch

aws ec2 modify-subnet-attribute \
  --subnet-id $SUBNET_PUB_B \
  --map-public-ip-on-launch
```

**NAT Gateway**: cho phép instance trong private subnet access Internet (outbound only). Cần thiết để EC2 trong private subnet download packages.

```bash
# Cấp Elastic IP cho NAT Gateway
EIP_ALLOC=$(aws ec2 allocate-address \
  --domain vpc \
  --query "AllocationId" --output text)

# Tạo NAT Gateway trong PUBLIC subnet
NAT_ID=$(aws ec2 create-nat-gateway \
  --subnet-id $SUBNET_PUB_A \
  --allocation-id $EIP_ALLOC \
  --query "NatGateway.NatGatewayId" --output text)

# Chờ NAT Gateway available (khoảng 1-2 phút)
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_ID

# Tạo Route Table cho private subnets
RTB_PRIVATE=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query "RouteTable.RouteTableId" --output text)

# Thêm route qua NAT Gateway
aws ec2 create-route \
  --route-table-id $RTB_PRIVATE \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_ID

# Gán cho private subnets
aws ec2 associate-route-table \
  --route-table-id $RTB_PRIVATE \
  --subnet-id $SUBNET_PRIV_A

aws ec2 associate-route-table \
  --route-table-id $RTB_PRIVATE \
  --subnet-id $SUBNET_PRIV_B
```

Lưu ý chi phí: NAT Gateway tốn tiền ngay cả khi idle (~$0.045/giờ + data processing). Cho môi trường học tập, không cần NAT Gateway — chỉ cần private subnet không có route ra Internet.

#### Security Groups

Security Group là virtual firewall ở cấp instance (stateful). Stateful có nghĩa là nếu cho phép inbound traffic, return traffic tự động được cho phép mà không cần thêm outbound rule.

Tạo Security Group cho EC2:
```bash
# EC2 Security Group
SG_EC2=$(aws ec2 create-security-group \
  --group-name shoplite-ec2-sg \
  --description "ShopLite EC2 Security Group" \
  --vpc-id $VPC_ID \
  --query "GroupId" --output text)

# SSH từ IP của bạn (thay YOUR_IP)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_EC2 \
  --protocol tcp --port 22 \
  --cidr $(curl -s ifconfig.me)/32

# HTTP từ Internet
aws ec2 authorize-security-group-ingress \
  --group-id $SG_EC2 \
  --protocol tcp --port 80 \
  --cidr 0.0.0.0/0

# HTTPS từ Internet
aws ec2 authorize-security-group-ingress \
  --group-id $SG_EC2 \
  --protocol tcp --port 443 \
  --cidr 0.0.0.0/0

# Port 8080 cho Node.js app (chỉ cần nếu không dùng Nginx)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_EC2 \
  --protocol tcp --port 8080 \
  --cidr 0.0.0.0/0
```

Tạo Security Group cho RDS:
```bash
SG_RDS=$(aws ec2 create-security-group \
  --group-name shoplite-rds-sg \
  --description "ShopLite RDS Security Group" \
  --vpc-id $VPC_ID \
  --query "GroupId" --output text)

# PostgreSQL CHỈ từ EC2 Security Group (không phải IP)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_RDS \
  --protocol tcp --port 5432 \
  --source-group $SG_EC2
```

Tạo Security Group cho Redis (ElastiCache):
```bash
SG_REDIS=$(aws ec2 create-security-group \
  --group-name shoplite-redis-sg \
  --description "ShopLite Redis Security Group" \
  --vpc-id $VPC_ID \
  --query "GroupId" --output text)

# Redis CHỈ từ EC2 Security Group
aws ec2 authorize-security-group-ingress \
  --group-id $SG_REDIS \
  --protocol tcp --port 6379 \
  --source-group $SG_EC2
```

Tư duy bảo mật: luôn dùng Security Group reference thay vì IP cho internal services. Lý do: IP có thể thay đổi khi restart instance, nhưng Security Group ID thì không đổi.

---

### 4. EC2 — Elastic Compute Cloud

EC2 là dịch vụ cung cấp máy ảo trên cloud. Đây là nơi bạn chạy ứng dụng ShopLite.

#### Instance Types — Loại máy ảo

Cách đọc instance type: `t3.micro`
- `t`: Family (t = burstable general purpose, m = general purpose, c = compute optimized, r = memory optimized)
- `3`: Generation (số càng cao càng mới, hiệu năng tốt hơn, giá tốt hơn)
- `micro`: Size (nano < micro < small < medium < large < xlarge < 2xlarge...)

Các loại phổ biến cho ShopLite:
- `t3.micro`: 2 vCPU, 1GB RAM — free tier, dùng để học
- `t3.small`: 2 vCPU, 2GB RAM — production nhỏ (~$0.02/giờ)
- `t3.medium`: 2 vCPU, 4GB RAM — production vừa
- `t3.large`: 2 vCPU, 8GB RAM — production lớn hơn

Burstable performance: t3 instances có CPU credits. Khi idle, tích lũy credits. Khi cần burst, dùng credits. Tốt cho workload không đều.

#### AMI — Amazon Machine Image

AMI là template cho EC2 instance, bao gồm OS và pre-installed software.

Tìm Ubuntu 22.04 LTS AMI ở ap-southeast-1:
```bash
aws ec2 describe-images \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
    "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text
# Output: ami-0df7a207adb9748c7 (có thể thay đổi theo thời gian)
```

#### Tạo Key Pair

Key pair dùng để SSH vào EC2. AWS lưu public key, bạn lưu private key.

```bash
# Tạo key pair và lưu private key
aws ec2 create-key-pair \
  --key-name shoplite-key \
  --query "KeyMaterial" \
  --output text > ~/.ssh/shoplite-key.pem

# Set đúng permissions (bắt buộc, SSH từ chối nếu permissions quá rộng)
chmod 400 ~/.ssh/shoplite-key.pem

# Verify
ls -la ~/.ssh/shoplite-key.pem
# -r-------- 1 user user 1674 Jun 21 10:00 /home/user/.ssh/shoplite-key.pem
```

#### User Data Script

User data chạy một lần khi instance khởi động lần đầu. Dùng để tự động cài đặt phần mềm.

```bash
cat > user-data.sh << 'EOF'
#!/bin/bash
# Log output
exec > /var/log/user-data.log 2>&1
set -x

# Update system
apt-get update -y
apt-get upgrade -y

# Cài Docker
apt-get install -y docker.io docker-compose-plugin git curl

# Khởi động Docker
systemctl enable docker
systemctl start docker

# Thêm ubuntu user vào docker group
usermod -aG docker ubuntu

# Cài AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "/tmp/awscliv2.zip"
apt-get install -y unzip
unzip /tmp/awscliv2.zip -d /tmp
/tmp/aws/install

# Cài certbot
snap install --classic certbot
ln -s /snap/bin/certbot /usr/bin/certbot

echo "User data complete at $(date)"
EOF
```

#### Launch EC2 Instance

```bash
# Lấy AMI ID mới nhất của Ubuntu 22.04
AMI_ID=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
    "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text)

# Launch instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --key-name shoplite-key \
  --security-group-ids $SG_EC2 \
  --subnet-id $SUBNET_PUB_A \
  --iam-instance-profile Name=shoplite-ec2-profile \
  --user-data file://user-data.sh \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":20,"VolumeType":"gp3","DeleteOnTermination":true}}]' \
  --tag-specifications \
    'ResourceType=instance,Tags=[{Key=Name,Value=shoplite-web},{Key=Environment,Value=production}]' \
  --query "Instances[0].InstanceId" \
  --output text)

echo "Instance ID: $INSTANCE_ID"

# Chờ instance running
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
echo "Instance is running"
```

#### Elastic IP

Elastic IP là địa chỉ IPv4 public tĩnh. Khi stop/start EC2, public IP thông thường sẽ thay đổi, nhưng Elastic IP thì không.

```bash
# Cấp phát Elastic IP
EIP_ALLOC_ID=$(aws ec2 allocate-address \
  --domain vpc \
  --query "AllocationId" \
  --output text)

EIP_ADDRESS=$(aws ec2 describe-addresses \
  --allocation-ids $EIP_ALLOC_ID \
  --query "Addresses[0].PublicIp" \
  --output text)

echo "Elastic IP: $EIP_ADDRESS"

# Gán vào EC2 instance
aws ec2 associate-address \
  --instance-id $INSTANCE_ID \
  --allocation-id $EIP_ALLOC_ID

echo "Elastic IP $EIP_ADDRESS gán vào $INSTANCE_ID thành công"
```

Lưu ý: Elastic IP miễn phí khi đang được gán cho running instance. Nếu bạn allocate nhưng không gán, hoặc instance đang stopped, bị tính phí $0.005/giờ.

#### SSH vào EC2

```bash
# SSH lần đầu
ssh -i ~/.ssh/shoplite-key.pem ubuntu@$EIP_ADDRESS

# Shortcut trong ~/.ssh/config
cat >> ~/.ssh/config << EOF

Host shoplite
  HostName $EIP_ADDRESS
  User ubuntu
  IdentityFile ~/.ssh/shoplite-key.pem
  StrictHostKeyChecking no
EOF

# Sau đó chỉ cần
ssh shoplite
```

Troubleshooting SSH:
- `Permission denied (publickey)`: sai key hoặc wrong user (Ubuntu AMI dùng `ubuntu`, Amazon Linux dùng `ec2-user`)
- `Connection timeout`: Security Group chưa mở port 22, hoặc Elastic IP chưa gán
- `WARNING: UNPROTECTED PRIVATE KEY FILE!`: chạy `chmod 400 ~/.ssh/shoplite-key.pem`

---

### 5. RDS — Relational Database Service

RDS là managed database service. AWS quản lý: hardware, OS, database engine installation, patching, backups, replication. Bạn chỉ quản lý: schema, data, queries.

#### Tại sao dùng RDS thay vì tự cài PostgreSQL trên EC2?

| Tính năng | RDS | Tự cài trên EC2 |
|-----------|-----|-----------------|
| Backup tự động | Có (0-35 ngày retention) | Tự làm |
| Multi-AZ failover | Có (tự động) | Tự cấu hình |
| Read replicas | Có | Tự làm |
| Patching | AWS quản lý | Tự làm |
| Monitoring | CloudWatch metrics tích hợp | Tự cài |
| Giá | Cao hơn | Thấp hơn |
| Control | Ít hơn | Toàn quyền |

#### Tạo DB Subnet Group

Subnet group là tập hợp subnets trong các AZ khác nhau cho RDS. Yêu cầu tối thiểu 2 AZ.

```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name shoplite-db-subnet-group \
  --db-subnet-group-description "ShopLite Database Subnet Group" \
  --subnet-ids $SUBNET_PRIV_A $SUBNET_PRIV_B \
  --tags Key=Name,Value=shoplite-db-subnet-group
```

#### Tạo RDS PostgreSQL Instance

```bash
aws rds create-db-instance \
  --db-instance-identifier shoplite-postgres \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 15.4 \
  --master-username shopliteadmin \
  --master-user-password "SuperStr0ng!Pass#2024" \
  --db-name shoplite \
  --vpc-security-group-ids $SG_RDS \
  --db-subnet-group-name shoplite-db-subnet-group \
  --storage-type gp2 \
  --allocated-storage 20 \
  --max-allocated-storage 100 \
  --backup-retention-period 7 \
  --preferred-backup-window "02:00-03:00" \
  --preferred-maintenance-window "sun:04:00-sun:05:00" \
  --no-publicly-accessible \
  --deletion-protection \
  --enable-cloudwatch-logs-exports postgresql \
  --tags Key=Name,Value=shoplite-postgres Key=Environment,Value=production

# Chờ available (mất khoảng 5-10 phút)
aws rds wait db-instance-available --db-instance-identifier shoplite-postgres

# Lấy endpoint
RDS_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier shoplite-postgres \
  --query "DBInstances[0].Endpoint.Address" \
  --output text)

echo "RDS Endpoint: $RDS_ENDPOINT"
# Output: shoplite-postgres.xxxx.ap-southeast-1.rds.amazonaws.com
```

#### Kết nối đến RDS từ local (SSH Tunnel)

Vì RDS nằm trong private subnet, không thể kết nối trực tiếp từ local. Dùng SSH tunnel qua EC2 (bastion host).

```bash
# Terminal 1: Tạo SSH tunnel
ssh -i ~/.ssh/shoplite-key.pem \
  -L 5432:$RDS_ENDPOINT:5432 \
  -N ubuntu@$EIP_ADDRESS &

TUNNEL_PID=$!
echo "SSH tunnel PID: $TUNNEL_PID"

# Terminal 2: Kết nối qua tunnel
psql -h localhost -p 5432 -U shopliteadmin -d shoplite

# Đóng tunnel
kill $TUNNEL_PID
```

#### Connection String trong ứng dụng

```bash
# .env file trên EC2
DATABASE_URL=postgresql://shopliteadmin:SuperStr0ng!Pass#2024@shoplite-postgres.xxxx.ap-southeast-1.rds.amazonaws.com:5432/shoplite?sslmode=require
```

Lưu ý `sslmode=require`: RDS PostgreSQL yêu cầu SSL mặc định. Không thêm sẽ kết nối thất bại.

#### Backup và Restore

```bash
# Tạo snapshot thủ công trước khi làm gì quan trọng
aws rds create-db-snapshot \
  --db-instance-identifier shoplite-postgres \
  --db-snapshot-identifier shoplite-snapshot-before-migration

# Restore từ snapshot (tạo instance mới)
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier shoplite-postgres-restored \
  --db-snapshot-identifier shoplite-snapshot-before-migration \
  --db-instance-class db.t3.micro \
  --vpc-security-group-ids $SG_RDS \
  --db-subnet-group-name shoplite-db-subnet-group \
  --no-publicly-accessible

# Xóa instance (tiết kiệm chi phí khi học xong, nhưng BACKUP TRƯỚC)
aws rds delete-db-instance \
  --db-instance-identifier shoplite-postgres \
  --final-db-snapshot-identifier shoplite-final-snapshot \
  --delete-automated-backups false
```

---

### 6. S3 — Simple Storage Service

S3 là object storage — lưu trữ file dưới dạng objects trong buckets. Không phải file system, không có folder thật (chỉ là prefix trong key name).

#### Khái niệm

- **Bucket**: container chứa objects, tên globally unique trên toàn AWS
- **Object**: file + metadata, có thể tới 5TB
- **Key**: đường dẫn của object trong bucket (ví dụ: `images/product-001.jpg`)
- **Storage Class**: Standard, IA (Infrequent Access), Glacier (archive)

#### Tạo và cấu hình Bucket

```bash
# Tạo bucket (tên phải globally unique)
BUCKET_NAME="shoplite-assets-$(date +%s)"
aws s3 mb s3://$BUCKET_NAME --region ap-southeast-1

# Bật versioning (recover từ accidental delete)
aws s3api put-bucket-versioning \
  --bucket $BUCKET_NAME \
  --versioning-configuration Status=Enabled

# Block tất cả public access (mặc định) — sau đó mở từng phần nếu cần
aws s3api put-public-access-block \
  --bucket $BUCKET_NAME \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# Enable server-side encryption
aws s3api put-bucket-encryption \
  --bucket $BUCKET_NAME \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'
```

#### Bucket Policy cho Public Product Images

```bash
cat > bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadProductImages",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::$BUCKET_NAME/images/products/*"
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket $BUCKET_NAME \
  --policy file://bucket-policy.json
```

#### Upload và Sync Files

```bash
# Upload single file
aws s3 cp ./product-001.jpg s3://$BUCKET_NAME/images/products/product-001.jpg

# Upload với metadata
aws s3 cp ./product-001.jpg s3://$BUCKET_NAME/images/products/ \
  --content-type "image/jpeg" \
  --cache-control "max-age=31536000"

# Sync local folder lên S3
aws s3 sync ./public/images/ s3://$BUCKET_NAME/images/ \
  --delete \
  --exclude "*.DS_Store" \
  --include "*.jpg" \
  --include "*.png" \
  --include "*.webp"

# Download từ S3
aws s3 cp s3://$BUCKET_NAME/images/products/product-001.jpg ./

# Liệt kê files
aws s3 ls s3://$BUCKET_NAME/ --recursive --human-readable

# Xóa file
aws s3 rm s3://$BUCKET_NAME/images/products/old-product.jpg

# Xóa toàn bộ prefix
aws s3 rm s3://$BUCKET_NAME/temp/ --recursive
```

#### CORS Configuration cho Frontend

Cần thiết khi frontend (chạy ở domain khác) cần gọi S3 trực tiếp để upload.

```bash
cat > cors-config.json << 'EOF'
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
    "AllowedOrigins": [
      "https://shoplite.com",
      "https://www.shoplite.com",
      "http://localhost:3000"
    ],
    "ExposeHeaders": ["ETag"],
    "MaxAgeSeconds": 3000
  }
]
EOF

aws s3api put-bucket-cors \
  --bucket $BUCKET_NAME \
  --cors-configuration file://cors-config.json
```

#### Pre-signed URLs (Secure Upload)

Thay vì public upload, tạo pre-signed URL có thời hạn để frontend upload trực tiếp lên S3 mà không cần expose credentials.

```javascript
// Backend Node.js tạo pre-signed URL
const { S3Client, PutObjectCommand } = require("@aws-sdk/client-s3");
const { getSignedUrl } = require("@aws-sdk/s3-request-presigner");

const s3 = new S3Client({ region: "ap-southeast-1" });

async function getUploadUrl(key, contentType) {
  const command = new PutObjectCommand({
    Bucket: process.env.S3_BUCKET,
    Key: key,
    ContentType: contentType,
  });
  // URL hết hạn sau 5 phút
  return await getSignedUrl(s3, command, { expiresIn: 300 });
}
```

#### Backup Database lên S3

```bash
cat > /home/ubuntu/backup-db.sh << 'EOF'
#!/bin/bash
set -e

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="shoplite-backup-$DATE.sql.gz"
S3_BUCKET="shoplite-backup-unique-name"

echo "Starting backup at $DATE"

# Dump và compress
pg_dump $DATABASE_URL | gzip > /tmp/$BACKUP_FILE

# Upload lên S3
aws s3 cp /tmp/$BACKUP_FILE s3://$S3_BUCKET/database/$BACKUP_FILE \
  --storage-class STANDARD_IA

# Xóa file local
rm /tmp/$BACKUP_FILE

echo "Backup complete: s3://$S3_BUCKET/database/$BACKUP_FILE"
EOF

chmod +x /home/ubuntu/backup-db.sh

# Chạy hàng ngày lúc 3:00 AM
(crontab -l 2>/dev/null; echo "0 3 * * * /home/ubuntu/backup-db.sh >> /var/log/backup.log 2>&1") | crontab -
```

#### Lifecycle Rules — Tự động quản lý vòng đời

```bash
cat > lifecycle.json << 'EOF'
{
  "Rules": [
    {
      "ID": "DeleteOldBackups",
      "Filter": {"Prefix": "database/"},
      "Status": "Enabled",
      "Expiration": {"Days": 30},
      "NoncurrentVersionExpiration": {"NoncurrentDays": 7}
    },
    {
      "ID": "ArchiveOldImages",
      "Filter": {"Prefix": "images/archive/"},
      "Status": "Enabled",
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER"}
      ]
    }
  ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket $BUCKET_NAME \
  --lifecycle-configuration file://lifecycle.json
```

---

### 7. Route 53 — DNS Service

Route 53 là DNS service của AWS, cho phép quản lý domain và routing traffic.

#### Khái niệm DNS

- **Hosted Zone**: container cho DNS records của một domain
- **Record Types**: A (IPv4), AAAA (IPv6), CNAME (alias), MX (mail), TXT (verification)
- **Alias Record**: AWS-specific, trỏ đến AWS resources (ELB, CloudFront) miễn phí, tự động follow IP changes
- **TTL (Time To Live)**: thời gian cache DNS record (giây)

#### Tạo Hosted Zone

```bash
# Tạo hosted zone cho domain của bạn
HOSTED_ZONE_ID=$(aws route53 create-hosted-zone \
  --name shoplite.com \
  --caller-reference $(date +%s) \
  --query "HostedZone.Id" \
  --output text | cut -d'/' -f3)

echo "Hosted Zone ID: $HOSTED_ZONE_ID"

# Lấy Name Servers để cấu hình tại domain registrar
aws route53 get-hosted-zone \
  --id $HOSTED_ZONE_ID \
  --query "DelegationSet.NameServers"
# Output: 4 name servers như ns-xxx.awsdns-xx.com
# Copy 4 NS này vào domain registrar (GoDaddy, Namecheap, etc.)
```

#### Tạo DNS Records

```bash
# Tạo A record cho root domain (trỏ đến Elastic IP)
cat > dns-records.json << EOF
{
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "shoplite.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "$EIP_ADDRESS"}]
      }
    },
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "www.shoplite.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "$EIP_ADDRESS"}]
      }
    }
  ]
}
EOF

aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch file://dns-records.json

# Kiểm tra propagation
aws route53 get-change --id /change/XXXXX
# Status: PENDING -> INSYNC (thường < 60 giây)

# Verify DNS
dig shoplite.com
nslookup shoplite.com 8.8.8.8
```

---

### 8. Deploy ShopLite lên AWS — Hướng dẫn từng bước

Đây là quy trình đầy đủ để deploy ShopLite production.

#### Bước 1: Chuẩn bị trước khi bắt đầu

```bash
# Checklist:
# [ ] Đã cài AWS CLI và cấu hình credentials
# [ ] Đã có domain (mua tại Namecheap/GoDaddy ~$10/năm)
# [ ] Đã tạo VPC, subnets, security groups (theo hướng dẫn phần 3)
# [ ] Repo ShopLite có Dockerfile và docker-compose.yml hoạt động local

# Set environment variables cho tiện
export AWS_DEFAULT_REGION=ap-southeast-1
export PROJECT_NAME=shoplite
```

#### Bước 2: Launch EC2 và cấu hình

```bash
# Launch instance (đã hướng dẫn ở phần 4)
# Sau khi SSH vào:

ssh shoplite

# Verify Docker đã cài từ user-data
docker --version
docker compose version

# Clone repo
cd /home/ubuntu
git clone https://github.com/your-username/shoplite.git
cd shoplite

# Tạo .env production
cat > .env.production << 'EOF'
NODE_ENV=production
PORT=8080

DATABASE_URL=postgresql://shopliteadmin:PASSWORD@RDS_ENDPOINT:5432/shoplite?sslmode=require
REDIS_URL=redis://localhost:6379

JWT_SECRET=your-super-secret-jwt-key-change-this
JWT_EXPIRES_IN=7d

S3_BUCKET=shoplite-assets-unique-name
S3_REGION=ap-southeast-1
AWS_ACCESS_KEY_ID=  # để trống nếu dùng IAM Role
AWS_SECRET_ACCESS_KEY=  # để trống nếu dùng IAM Role

FRONTEND_URL=https://shoplite.com
EOF
```

#### Bước 3: Cấu hình Nginx

```bash
# Cài Nginx (nếu chưa có trong user-data)
sudo apt-get install -y nginx

# Tạo cấu hình cho ShopLite
sudo tee /etc/nginx/sites-available/shoplite << 'EOF'
server {
    listen 80;
    server_name shoplite.com www.shoplite.com;

    # API
    location /api/ {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Frontend static files
    location / {
        root /home/ubuntu/shoplite/frontend/dist;
        try_files $uri $uri/ /index.html;
        
        # Cache static assets
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }

    # Health check
    location /health {
        proxy_pass http://localhost:8080/health;
    }
}
EOF

# Enable site
sudo ln -s /etc/nginx/sites-available/shoplite /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

#### Bước 4: Chạy ứng dụng với Docker Compose

```bash
# Build và start containers
cd /home/ubuntu/shoplite
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Verify containers running
docker compose ps

# Chạy migrations
docker compose exec backend npm run db:migrate

# Seed initial data (nếu cần)
docker compose exec backend npm run db:seed

# Xem logs
docker compose logs -f backend
```

#### Bước 5: Cài SSL bằng Certbot

```bash
# Đảm bảo DNS đã propagate (A record trỏ đến Elastic IP)
dig +short shoplite.com  # phải trả về Elastic IP của bạn

# Cài SSL certificate
sudo certbot --nginx \
  -d shoplite.com \
  -d www.shoplite.com \
  --non-interactive \
  --agree-tos \
  --email your-email@gmail.com \
  --redirect

# Certbot tự động:
# 1. Lấy certificate từ Let's Encrypt
# 2. Cấu hình Nginx với HTTPS
# 3. Setup auto-renewal

# Verify auto-renewal
sudo certbot renew --dry-run

# Certificate tự động renew qua cron/systemd timer
sudo systemctl status certbot.timer
```

#### Bước 6: Cấu hình systemd để auto-start

```bash
sudo tee /etc/systemd/system/shoplite.service << 'EOF'
[Unit]
Description=ShopLite Application
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/ubuntu/shoplite
ExecStart=/usr/bin/docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable shoplite
sudo systemctl start shoplite
```

#### Bước 7: Verify deployment

```bash
# Test từ local
curl -I https://shoplite.com
# HTTP/2 200

curl https://shoplite.com/api/health
# {"status": "ok", "database": "connected", "timestamp": "..."}

# Test performance
curl -w "@curl-format.txt" -o /dev/null -s https://shoplite.com/api/products
```

---

### 9. Quản lý chi phí AWS

#### Bật Budget Alerts

```bash
# Tạo budget $10/tháng với email alert khi vượt 80%
cat > budget.json << 'EOF'
{
  "BudgetName": "ShopLite-Monthly-Budget",
  "BudgetLimit": {
    "Amount": "10",
    "Unit": "USD"
  },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST",
  "NotificationsWithSubscribers": [
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [{
        "SubscriptionType": "EMAIL",
        "Address": "your-email@gmail.com"
      }]
    }
  ]
}
EOF

aws budgets create-budget \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget file://budget.json
```

#### Tiết kiệm chi phí khi học

```bash
# Stop EC2 khi không dùng (chỉ tốn chi phí storage EBS ~$0.10/GB/tháng)
aws ec2 stop-instances --instance-ids $INSTANCE_ID
aws ec2 start-instances --instance-ids $INSTANCE_ID

# Script tự động stop vào cuối ngày
cat > /home/ubuntu/auto-stop.sh << 'SCRIPT'
#!/bin/bash
# Chạy lúc 22:00 mỗi ngày
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
aws ec2 stop-instances --instance-ids $INSTANCE_ID
SCRIPT

chmod +x /home/ubuntu/auto-stop.sh
# Thêm vào cron: 0 22 * * * /home/ubuntu/auto-stop.sh

# Xóa RDS khi học xong (backup trước)
aws rds create-db-snapshot \
  --db-instance-identifier shoplite-postgres \
  --db-snapshot-identifier shoplite-final-$(date +%Y%m%d)

aws rds delete-db-instance \
  --db-instance-identifier shoplite-postgres \
  --final-db-snapshot-identifier shoplite-final-$(date +%Y%m%d)

# Monitor S3 usage
aws s3 ls s3://$BUCKET_NAME --recursive --human-readable --summarize

# Xem chi phí hiện tại
aws ce get-cost-and-usage \
  --time-period Start=$(date -d "1 month ago" +%Y-%m-01),End=$(date +%Y-%m-01) \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --query "ResultsByTime[0].Total.UnblendedCost"
```

#### Ước tính chi phí ShopLite hàng tháng

Môi trường học tập (Free Tier):
- EC2 t3.micro: $0 (750 giờ/tháng free tier)
- RDS db.t3.micro: $0 (750 giờ/tháng free tier)
- S3 5GB: $0 (free tier)
- Route 53: $0.50/hosted zone + $0.40/million queries
- Elastic IP: $0 (khi gán cho running instance)
- **Tổng: ~$1/tháng**

Môi trường production thực (sau Free Tier):
- EC2 t3.small: ~$15/tháng
- RDS db.t3.micro Multi-AZ: ~$30/tháng
- S3 10GB: ~$0.23/tháng
- NAT Gateway: ~$32/tháng (nếu dùng)
- Route 53: ~$0.50/tháng
- **Tổng: ~$50-80/tháng**

Mẹo giảm chi phí:
1. Dùng Reserved Instances (commit 1-3 năm) giảm 40-60% so với On-Demand
2. Spot Instances cho workload có thể interrupt giảm 70-90%
3. Tắt NAT Gateway khi không cần egress từ private subnet
4. Dùng S3 lifecycle để move cold data sang Glacier

---

### 10. AWS CLI Essentials — Các lệnh hay dùng

#### Quản lý EC2

```bash
# Liệt kê tất cả instances
aws ec2 describe-instances \
  --query "Reservations[].Instances[].[InstanceId,State.Name,InstanceType,PublicIpAddress,Tags[?Key=='Name'].Value|[0]]" \
  --output table

# Start/Stop/Terminate
aws ec2 start-instances --instance-ids i-xxxx
aws ec2 stop-instances --instance-ids i-xxxx
aws ec2 terminate-instances --instance-ids i-xxxx

# Lấy console output (debug khi không SSH được)
aws ec2 get-console-output --instance-id i-xxxx --output text

# Reboot
aws ec2 reboot-instances --instance-ids i-xxxx

# Xem security groups của instance
aws ec2 describe-instances \
  --instance-ids i-xxxx \
  --query "Reservations[0].Instances[0].SecurityGroups"
```

#### Quản lý S3

```bash
# Liệt kê tất cả buckets
aws s3 ls

# Liệt kê contents
aws s3 ls s3://bucket-name/ --recursive

# Copy
aws s3 cp source destination
aws s3 cp s3://bucket/file.txt . # Download
aws s3 cp ./file.txt s3://bucket/ # Upload

# Sync
aws s3 sync . s3://bucket/ --exclude ".git/*"

# Move (copy + delete)
aws s3 mv s3://bucket/old-key s3://bucket/new-key

# Presign URL (1 giờ)
aws s3 presign s3://bucket/file.txt --expires-in 3600

# Get bucket size
aws s3 ls --summarize --recursive s3://bucket | tail -2
```

#### Quản lý RDS

```bash
# Liệt kê instances
aws rds describe-db-instances \
  --query "DBInstances[].[DBInstanceIdentifier,DBInstanceStatus,DBInstanceClass,Endpoint.Address]" \
  --output table

# Tạo snapshot
aws rds create-db-snapshot \
  --db-instance-identifier shoplite-postgres \
  --db-snapshot-identifier snapshot-$(date +%Y%m%d)

# Liệt kê snapshots
aws rds describe-db-snapshots \
  --db-instance-identifier shoplite-postgres \
  --query "DBSnapshots[].[DBSnapshotIdentifier,Status,SnapshotCreateTime]" \
  --output table

# Reboot RDS (apply parameter changes)
aws rds reboot-db-instance --db-instance-identifier shoplite-postgres
```

#### Xem Logs với CloudWatch

```bash
# Liệt kê log groups
aws logs describe-log-groups \
  --query "logGroups[].[logGroupName,storedBytes]" \
  --output table

# Xem logs theo thời gian thực
aws logs tail /aws/rds/instance/shoplite-postgres/postgresql --follow

# Filter logs
aws logs filter-log-events \
  --log-group-name /aws/rds/instance/shoplite-postgres/postgresql \
  --filter-pattern "ERROR" \
  --start-time $(date -d "1 hour ago" +%s000)
```

#### IAM Utilities

```bash
# Xem current identity
aws sts get-caller-identity

# Liệt kê users
aws iam list-users --query "Users[].[UserName,CreateDate]" --output table

# Xem policies của user
aws iam list-attached-user-policies --user-name admin-phung

# Simulate policy (check quyền trước khi thực thi)
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789:user/phung \
  --action-names s3:PutObject \
  --resource-arns arn:aws:s3:::shoplite-assets/*
```

#### Networking

```bash
# Liệt kê VPCs
aws ec2 describe-vpcs \
  --query "Vpcs[].[VpcId,CidrBlock,Tags[?Key=='Name'].Value|[0]]" \
  --output table

# Liệt kê subnets
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[].[SubnetId,CidrBlock,AvailabilityZone,MapPublicIpOnLaunch]" \
  --output table

# Liệt kê security groups
aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "SecurityGroups[].[GroupId,GroupName,Description]" \
  --output table

# Xem inbound rules của security group
aws ec2 describe-security-groups \
  --group-ids $SG_EC2 \
  --query "SecurityGroups[0].IpPermissions"
```

---

### 11. Troubleshooting phổ biến

#### EC2 không start được application

```bash
# SSH vào và kiểm tra
ssh shoplite

# Xem user-data logs
sudo cat /var/log/cloud-init-output.log

# Kiểm tra Docker status
sudo systemctl status docker
docker ps -a  # xem tất cả containers kể cả đã stop

# Xem logs container
docker logs container-name --tail 100

# Kiểm tra port binding
ss -tlnp | grep -E '80|443|8080|5432'

# Kiểm tra Nginx
sudo nginx -t
sudo systemctl status nginx
sudo journalctl -u nginx --since "10 minutes ago"
```

#### Không connect được RDS

```bash
# Từ EC2, test connectivity
nc -zv $RDS_ENDPOINT 5432
# Expected: Connection to xxxx.rds.amazonaws.com 5432 port [tcp/postgresql] succeeded

# Nếu fail: kiểm tra security group của RDS
# - Inbound 5432 từ EC2 security group?
# - EC2 và RDS cùng VPC?
# - RDS subnet group có đúng subnets?

# Kiểm tra từ EC2
psql -h $RDS_ENDPOINT -U shopliteadmin -d shoplite -c "SELECT version();"
```

#### SSL Certificate lỗi

```bash
# Kiểm tra certbot logs
sudo journalctl -u certbot --since today

# Manual renew
sudo certbot renew --force-renewal

# Check certificate expiry
sudo certbot certificates

# Verify SSL
curl -I https://shoplite.com
openssl s_client -connect shoplite.com:443 -servername shoplite.com < /dev/null
```

---

### 12. Security Best Practices cho ShopLite

```bash
# Disable password authentication SSH (chỉ dùng key)
sudo sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart sshd

# Enable automatic security updates
sudo apt-get install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades

# Install fail2ban (chống brute force SSH)
sudo apt-get install -y fail2ban
sudo systemctl enable --now fail2ban

# Kiểm tra listening ports
ss -tlnp

# Xem auth logs
sudo journalctl -u sshd --since "1 hour ago"
sudo grep "Failed password" /var/log/auth.log | tail -20

# AWS Config (detect security misconfigurations)
# Bật trong AWS Console -> Config -> Get started
# Rule: restricted-ssh, rds-instance-public-access-check, s3-bucket-public-read-prohibited

# CloudTrail (audit log mọi API call)
aws cloudtrail create-trail \
  --name shoplite-audit-trail \
  --s3-bucket-name shoplite-cloudtrail-logs-unique \
  --is-multi-region-trail

aws cloudtrail start-logging --name shoplite-audit-trail
```

---

## Kỹ năng đạt được
- Khởi tạo và cấu hình hạ tầng cloud cơ bản (thủ công qua Console).
- Triển khai ứng dụng container lên cloud.
- Cấu hình mạng, bảo mật và DNS trên cloud.

---

## Thực hành

**Môi trường:** Tài khoản AWS (Free Tier).

**Lab:**
- Tạo VPC, subnet, security group.
- Khởi tạo EC2, cài Docker, chạy ShopLite (Compose) trên đó.
- Chuyển database sang RDS PostgreSQL.
- Dùng S3 lưu ảnh sản phẩm / file backup.
- Gắn domain qua Route 53, bật HTTPS bằng Let's Encrypt.

**Công cụ:** AWS Console, EC2, RDS, S3, IAM, Route 53.

---

## Bài tập

- **Bắt buộc:** ShopLite truy cập được công khai qua domain thật trên AWS.
- **Nâng cao:** Tách database ra RDS, ứng dụng chỉ kết nối qua mạng riêng (private subnet).
- **Mô phỏng doanh nghiệp:** Viết tài liệu kiến trúc cloud + ước tính chi phí hàng tháng.

---

## Deliverable
- [ ] ShopLite chạy public trên AWS với domain + HTTPS.
- [ ] Sơ đồ kiến trúc cloud.
- [ ] Tài liệu cấu hình hạ tầng (thủ công).

---

## Tiêu chí hoàn thành
- [ ] Người khác mở được ShopLite qua Internet.
- [ ] Database chạy trên RDS, app kết nối an toàn.
- [ ] Giải thích được toàn bộ kiến trúc cloud đã dựng.

---

## Ghi chú học tập

> _Ghi lại những điều học được, vướng mắc và insight trong quá trình học phase này._
