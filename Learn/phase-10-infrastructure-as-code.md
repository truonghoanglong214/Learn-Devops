# Phase 10 — Infrastructure as Code (Terraform & Ansible)

## Mục tiêu
- Dựng lại toàn bộ hạ tầng cloud bằng **code** thay vì click chuột.
- Cấu hình server tự động, có thể tái lập và phiên bản hóa.

---

## Kiến thức sẽ học

- **Vì sao cần IaC** (tái lập, version control, không "click thủ công").
- **Terraform** (provider, resource, variable, output, state, module).
- **Quản lý Terraform state** (remote state trên S3, locking).
- **Ansible** (playbook, inventory, role, idempotency).
- **Phân biệt provisioning (Terraform) vs configuration (Ansible)** — đúng tool đúng việc.

---

## Kiến thức chi tiết

### 1. Tại sao cần Infrastructure as Code

#### Vấn đề với ClickOps

ClickOps là cách làm truyền thống: engineer đăng nhập AWS Console, click từng bước để tạo VPC, EC2, RDS, security group. Cách này có rất nhiều vấn đề nghiêm trọng trong môi trường production:

**Không reproducible (không tái lập được):**
- Khi cần dựng lại môi trường (disaster recovery, staging mới), không ai nhớ chính xác đã click những gì.
- Mỗi lần dựng lại là mỗi lần có sai khác nhỏ. Staging khác production. Production khác DR.
- "Works on my environment" cho hạ tầng — không thể prove hai môi trường giống nhau.

**Không có audit trail:**
- Ai đã tạo security group này? Khi nào? Tại sao lại mở port 22 cho 0.0.0.0/0?
- AWS CloudTrail ghi lại API calls nhưng không đủ để reconstruct toàn bộ ý định.
- Không có dòng commit nào giải thích "lý do tại sao".

**Human error rate cao:**
- Nhập sai CIDR block. Quên enable versioning cho S3. Tạo sai loại instance.
- Những lỗi này đôi khi không phát hiện ngay, accumulate thành technical debt.
- Dưới áp lực incident, tốc độ click nhanh hơn, error rate tăng.

**Không có review process:**
- Không có khái niệm "PR review" cho thay đổi hạ tầng qua console.
- Junior engineer có thể xóa production database mà không ai biết trước.

**Drift không kiểm soát được:**
- Sau 6 tháng, không ai biết chính xác hạ tầng hiện tại trông như thế nào.
- Configuration drift giữa các environment tích lũy dần.

#### IaC giải quyết tất cả vấn đề trên

Infrastructure as Code là nguyên lý: **mọi thành phần hạ tầng được mô tả bằng code, được lưu trong version control, được review trước khi apply, được kiểm thử tự động.**

Nguyên tắc cốt lõi: *"If it runs on a server, it should be in code."*

**Version control:**
```
git log terraform/
commit a3f1d2b — Add RDS Multi-AZ for production (PR #142, reviewed by @lead)
commit 8bc4e1f — Open port 443 for CloudFront IPs only (security review passed)
commit 2de9a0c — Upgrade EC2 instance type t3.medium -> t3.large (capacity planning)
```

Mỗi thay đổi hạ tầng đều có: ai làm, khi nào, tại sao, ai review.

**Code review cho hạ tầng:**
- Mở port cho public? Phải qua PR, security team review.
- Tăng instance size? Cost estimate có trong PR description.
- Xóa database? Pipeline yêu cầu ít nhất 2 approvals.

**Automated testing:**
- `terraform validate` kiểm tra syntax.
- `terraform plan` preview chính xác những gì sẽ thay đổi.
- Checkov, tfsec scan security misconfigurations.
- Unit test cho modules bằng Terratest.

**Disaster recovery:**
- Toàn bộ hạ tầng có thể dựng lại trong vòng 30 phút từ `git clone` + `terraform apply`.
- DR drill trở thành chuyện bình thường, chạy quarterly.

#### Declarative vs Imperative

Đây là phân biệt quan trọng về paradigm:

**Declarative (Terraform, CloudFormation, Kubernetes manifests):**
- Bạn mô tả **DESIRED STATE** — hạ tầng phải trông như thế nào.
- Tool tự tìm ra cách để đạt được state đó từ state hiện tại.
- Bạn không quan tâm đến thứ tự các bước, dependency được tool xử lý.
- Idempotent theo thiết kế: chạy lại sẽ không thay đổi gì nếu state đã đúng.

```hcl
# Mô tả: "Tôi muốn có 1 EC2 instance loại t3.medium chạy Ubuntu"
resource "aws_instance" "web" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t3.medium"
}
# Terraform tự lo: instance đã tồn tại chưa? Cần tạo mới hay update?
```

**Imperative (Ansible ad-hoc, shell scripts, AWS CLI scripts):**
- Bạn mô tả **CÁC BƯỚC** cần thực hiện theo thứ tự.
- Phù hợp cho configuration management (cài packages, copy files, restart services).
- Cần tự xử lý idempotency (check trước khi thực hiện).
- Dễ hiểu hơn cho người mới, nhưng phức tạp hơn khi scale.

```bash
# Script imperative: bước 1, 2, 3...
apt-get update
apt-get install -y nginx
cp nginx.conf /etc/nginx/
systemctl restart nginx
```

**Trong thực tế, dùng cả hai:**
- Terraform (declarative) để provision hạ tầng (VPC, EC2, RDS).
- Ansible (imperative với idempotent modules) để configure software trên server.

---

### 2. Terraform Fundamentals

#### Các khái niệm cốt lõi

**Provider:**
Provider là plugin kết nối Terraform với API của cloud/service provider. Terraform giao tiếp với AWS, GCP, Azure thông qua provider tương ứng.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }
}

provider "aws" {
  region = var.aws_region
  # Credentials từ env vars: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
  # Hoặc từ ~/.aws/credentials (profile)
  # Hoặc từ IAM Role attached to EC2/ECS (best practice cho CI/CD)
}
```

Provider version constraint:
- `= 5.1.0` — chính xác version này.
- `~> 5.0` — bất kỳ 5.x.x (>=5.0, <6.0). Phổ biến nhất.
- `>= 5.0, < 6.0` — tương đương ~> 5.0.

**Resource:**
Resource là thành phần hạ tầng được Terraform quản lý. Mỗi resource có type (aws_instance) và local name (web). Cú pháp tham chiếu: `aws_instance.web`.

```hcl
resource "aws_instance" "web" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = var.instance_type
  vpc_security_group_ids = [aws_security_group.web.id]
  subnet_id              = aws_subnet.public_a.id
  key_name               = aws_key_pair.deploy.key_name
  iam_instance_profile   = aws_iam_instance_profile.web.name

  root_block_device {
    volume_size = 20
    volume_type = "gp3"
    encrypted   = true
  }

  tags = merge(local.common_tags, {
    Name = "${var.environment}-web"
    Role = "web"
  })
}
```

**Data Source:**
Data source cho phép đọc thông tin từ hạ tầng đã tồn tại (không do Terraform tạo ra, hoặc được tạo ở nơi khác). Tiền tố `data.`.

```hcl
# Lấy AMI Ubuntu mới nhất
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical official account

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Lấy thông tin VPC đã tồn tại (được tạo bởi team khác)
data "aws_vpc" "existing" {
  tags = {
    Name = "production-vpc"
  }
}

# Lấy AWS account ID hiện tại
data "aws_caller_identity" "current" {}

# Sử dụng:
# data.aws_ami.ubuntu.id
# data.aws_vpc.existing.cidr_block
# data.aws_caller_identity.current.account_id
```

**Variable:**
Variable là input parameter. Cho phép tái sử dụng code với các giá trị khác nhau cho mỗi environment.

```hcl
# Khai báo trong variables.tf
variable "aws_region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "ap-southeast-1"
}

variable "environment" {
  description = "Environment name (staging/production)"
  type        = string

  validation {
    condition     = contains(["staging", "production"], var.environment)
    error_message = "Environment must be 'staging' or 'production'."
  }
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.medium"
}

variable "db_password" {
  description = "Password for RDS database"
  type        = string
  sensitive   = true  # Không hiển thị trong plan/apply output và logs
}

variable "allowed_cidr_blocks" {
  description = "CIDR blocks allowed to access web servers"
  type        = list(string)
  default     = ["0.0.0.0/0"]
}

variable "tags" {
  description = "Additional tags for all resources"
  type        = map(string)
  default     = {}
}
```

Cách truyền giá trị vào variable (ưu tiên từ cao đến thấp):
1. `-var "environment=production"` — command line flag.
2. `-var-file="prod.tfvars"` — file được chỉ định.
3. `terraform.tfvars` hoặc `*.auto.tfvars` — tự động load.
4. Environment variable `TF_VAR_environment`.
5. Default value trong khai báo.
6. Hỏi interactive (nếu không có default).

**Output:**
Output là return value, expose thông tin từ Terraform để dùng ở nơi khác (scripts, CI/CD, module khác).

```hcl
# Khai báo trong outputs.tf
output "web_public_ip" {
  description = "Public IP address của web server"
  value       = aws_eip.web.public_ip
}

output "rds_endpoint" {
  description = "RDS endpoint để application kết nối"
  value       = aws_db_instance.main.endpoint
  # Không set sensitive ở đây vì endpoint không phải secret
}

output "rds_password" {
  description = "RDS master password"
  value       = var.db_password
  sensitive   = true  # Không hiển thị khi terraform output
}

output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "private_subnet_ids" {
  description = "Danh sách private subnet IDs"
  value       = aws_subnet.private[*].id
}
```

Sử dụng output:
```bash
terraform output web_public_ip
terraform output -json  # output tất cả dưới dạng JSON
terraform output -raw web_public_ip  # chỉ giá trị, không có quotes
```

**Locals:**
Locals là intermediate computed values, tránh lặp lại logic phức tạp.

```hcl
locals {
  # Tags chung cho tất cả resources
  common_tags = merge(var.tags, {
    Project     = "ShopLite"
    Environment = var.environment
    ManagedBy   = "Terraform"
    Owner       = "devops-team"
  })

  # Name prefix nhất quán
  name_prefix = "${var.environment}-shoplite"

  # Tính toán: production dùng Multi-AZ, staging không
  rds_multi_az = var.environment == "production" ? true : false

  # Danh sách AZ available
  azs = slice(data.aws_availability_zones.available.names, 0, 3)
}

# Sử dụng
resource "aws_instance" "web" {
  tags = merge(local.common_tags, { Name = "${local.name_prefix}-web" })
}
```

**Module:**
Module là nhóm resources có thể tái sử dụng. Mọi thư mục chứa .tf files đều là module. Gọi module bằng `module` block.

```hcl
# Gọi module networking
module "networking" {
  source = "./modules/networking"

  environment    = var.environment
  vpc_cidr       = "10.0.0.0/16"
  public_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.11.0/24", "10.0.12.0/24"]
}

# Gọi module compute, dùng output của module networking
module "compute" {
  source = "./modules/compute"

  environment = var.environment
  vpc_id      = module.networking.vpc_id
  subnet_ids  = module.networking.public_subnet_ids
  key_name    = var.key_name
}

# Module từ Terraform Registry (community modules)
module "rds" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 6.0"

  identifier = "${var.environment}-shoplite-db"
  engine     = "mysql"
  # ...
}
```

**State:**
State là file `terraform.tfstate` — mapping giữa Terraform config và real infrastructure. Đây là nguồn sự thật duy nhất của Terraform.

```json
// terraform.tfstate (simplified)
{
  "version": 4,
  "resources": [
    {
      "type": "aws_instance",
      "name": "web",
      "instances": [
        {
          "attributes": {
            "id": "i-0abc123def456",
            "ami": "ami-0c02fb55956c7d316",
            "instance_type": "t3.medium",
            "public_ip": "54.123.456.789"
          }
        }
      ]
    }
  ]
}
```

Vấn đề với local state:
- Không share được với team.
- Không lock được (race condition khi 2 người apply cùng lúc).
- Bị mất nếu mất máy tính.

Giải pháp: Remote state (S3 + DynamoDB).

#### Terraform Workflow

```
terraform init
  -> Download providers (vào .terraform/providers/)
  -> Download modules (vào .terraform/modules/)
  -> Configure backend (kết nối S3 nếu dùng remote state)

terraform fmt
  -> Format code theo standard (tương đương prettier cho HCL)
  -> terraform fmt -recursive (format toàn bộ thư mục)

terraform validate
  -> Kiểm tra syntax và logic cơ bản
  -> Không kết nối cloud, chỉ kiểm tra config

terraform plan
  -> So sánh config với state hiện tại
  -> Show chính xác resource nào sẽ: create (+), destroy (-), update (~)
  -> KHÔNG thay đổi gì trên cloud
  -> terraform plan -out=tfplan (lưu plan để apply sau)

terraform apply
  -> Apply những thay đổi từ plan
  -> Yêu cầu confirm "yes" (hoặc -auto-approve cho CI/CD)
  -> terraform apply tfplan (apply plan đã lưu, không hỏi confirm)

terraform destroy
  -> Xóa TẤT CẢ resources được quản lý bởi state này
  -> Yêu cầu confirm "yes"
  -> Dùng cho lab/staging cleanup, KHÔNG dùng production trừ khi chắc chắn

terraform state list
  -> Liệt kê tất cả resources trong state

terraform state show aws_instance.web
  -> Xem chi tiết state của 1 resource

terraform import aws_instance.web i-0abc123def456
  -> Import resource đã tồn tại vào Terraform management

terraform taint aws_instance.web
  -> Đánh dấu resource cần được recreate trong lần apply tiếp theo
  -> (Từ Terraform 0.15.2 dùng: terraform apply -replace=aws_instance.web)
```

---

### 3. HCL Syntax Chi Tiết

#### Provider và Backend Configuration

```hcl
# main.tf

terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  # Remote state backend — PHẢI bootstrap S3 bucket và DynamoDB table trước
  backend "s3" {
    bucket         = "shoplite-tfstate"
    key            = "prod/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    dynamodb_table = "shoplite-tflock"
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      ManagedBy = "Terraform"
      Project   = "ShopLite"
    }
  }
}
```

#### Variables File

```hcl
# variables.tf

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-southeast-1"
}

variable "environment" {
  description = "Deployment environment"
  type        = string

  validation {
    condition     = contains(["staging", "production"], var.environment)
    error_message = "Must be staging or production."
  }
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks for private subnets"
  type        = list(string)
  default     = ["10.0.11.0/24", "10.0.12.0/24"]
}

variable "instance_type" {
  description = "EC2 instance type for web server"
  type        = string
  default     = "t3.medium"
}

variable "key_name" {
  description = "EC2 Key Pair name for SSH"
  type        = string
}

variable "db_instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.t3.micro"
}

variable "db_name" {
  description = "Database name"
  type        = string
  default     = "shoplite"
}

variable "db_username" {
  description = "Database master username"
  type        = string
  default     = "shopliteadmin"
}

variable "db_password" {
  description = "Database master password"
  type        = string
  sensitive   = true
}

variable "app_port" {
  description = "Port the application listens on"
  type        = number
  default     = 8080
}
```

#### terraform.tfvars.example

```hcl
# terraform.tfvars.example
# Sao chép thành terraform.tfvars và điền giá trị thực
# KHÔNG commit terraform.tfvars vào git (chứa secrets)

aws_region  = "ap-southeast-1"
environment = "staging"

vpc_cidr             = "10.0.0.0/16"
public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
private_subnet_cidrs = ["10.0.11.0/24", "10.0.12.0/24"]

instance_type = "t3.medium"
key_name      = "my-deploy-key"

db_instance_class = "db.t3.micro"
db_name           = "shoplite"
db_username       = "shopliteadmin"
db_password       = "CHANGE_ME_USE_STRONG_PASSWORD"

app_port = 8080
```

#### .gitignore cho Terraform

```gitignore
# .gitignore trong thư mục terraform/

# Terraform state files — chứa sensitive data, quản lý bởi remote backend
*.tfstate
*.tfstate.*
*.tfstate.backup

# Terraform plan output — chứa sensitive values
tfplan
*.tfplan

# Variables file với giá trị thực — chứa secrets
terraform.tfvars
*.auto.tfvars

# Terraform working directory
.terraform/
.terraform.lock.hcl  # Commit cái này để lock provider versions

# Crash log
crash.log
crash.*.log

# Override files (dùng cho local testing)
override.tf
override.tf.json
*_override.tf
*_override.tf.json
```

Lưu ý: `.terraform.lock.hcl` nên được commit để đảm bảo toàn team dùng cùng provider version.

---

### 4. Terraform Project Structure cho ShopLite

#### Cấu trúc thư mục hoàn chỉnh

```
terraform/
├── main.tf                      # Provider config, backend, gọi modules
├── variables.tf                 # Input variable declarations
├── outputs.tf                   # Root outputs
├── locals.tf                    # Local values
├── terraform.tfvars.example     # Example values (COMMIT này)
├── .terraform.lock.hcl          # Provider version lock (COMMIT này)
├── .gitignore                   # Ignore tfstate, tfvars, .terraform/
│
└── modules/
    ├── networking/
    │   ├── main.tf              # VPC, subnets, IGW, NAT Gateway, route tables
    │   ├── variables.tf
    │   └── outputs.tf
    ├── compute/
    │   ├── main.tf              # EC2, security groups, key pair, EIP, IAM role
    │   ├── variables.tf
    │   └── outputs.tf
    ├── database/
    │   ├── main.tf              # RDS instance, subnet group, parameter group
    │   ├── variables.tf
    │   └── outputs.tf
    └── storage/
        ├── main.tf              # S3 buckets, bucket policies
        ├── variables.tf
        └── outputs.tf
```

#### modules/networking/main.tf

```hcl
# modules/networking/main.tf

locals {
  name_prefix = "${var.environment}-shoplite"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${local.name_prefix}-vpc"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${local.name_prefix}-igw"
  }
}

# Public Subnets
resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${local.name_prefix}-public-${var.availability_zones[count.index]}"
    Type = "public"
  }
}

# Private Subnets
resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "${local.name_prefix}-private-${var.availability_zones[count.index]}"
    Type = "private"
  }
}

# Elastic IP cho NAT Gateway
resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? 1 : 0
  domain = "vpc"

  tags = {
    Name = "${local.name_prefix}-nat-eip"
  }
}

# NAT Gateway (trong public subnet để private subnet ra internet)
resource "aws_nat_gateway" "main" {
  count = var.enable_nat_gateway ? 1 : 0

  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id

  tags = {
    Name = "${local.name_prefix}-nat"
  }

  depends_on = [aws_internet_gateway.main]
}

# Route table cho public subnets
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${local.name_prefix}-public-rt"
  }
}

# Route table cho private subnets
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  dynamic "route" {
    for_each = var.enable_nat_gateway ? [1] : []
    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.main[0].id
    }
  }

  tags = {
    Name = "${local.name_prefix}-private-rt"
  }
}

# Associate public route table với public subnets
resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Associate private route table với private subnets
resource "aws_route_table_association" "private" {
  count = length(aws_subnet.private)

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}
```

#### modules/networking/outputs.tf

```hcl
# modules/networking/outputs.tf

output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "VPC CIDR block"
  value       = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "nat_gateway_ip" {
  description = "NAT Gateway public IP"
  value       = var.enable_nat_gateway ? aws_eip.nat[0].public_ip : null
}
```

#### modules/compute/main.tf

```hcl
# modules/compute/main.tf

locals {
  name_prefix = "${var.environment}-shoplite"
}

# Security Group cho Web Server
resource "aws_security_group" "web" {
  name        = "${local.name_prefix}-web-sg"
  description = "Security group for web servers"
  vpc_id      = var.vpc_id

  # HTTP
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP from anywhere"
  }

  # HTTPS
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS from anywhere"
  }

  # App port (từ load balancer hoặc direct)
  ingress {
    from_port   = var.app_port
    to_port     = var.app_port
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
    description = "App port from VPC"
  }

  # SSH - chỉ từ bastion hoặc management CIDR
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.ssh_allowed_cidrs
    description = "SSH from management IPs only"
  }

  # Outbound: cho phép tất cả
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound traffic"
  }

  tags = {
    Name = "${local.name_prefix}-web-sg"
  }
}

# IAM Role cho EC2 (least privilege)
resource "aws_iam_role" "web" {
  name = "${local.name_prefix}-web-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

# Policy cho phép EC2 đọc Parameter Store
resource "aws_iam_role_policy" "web_ssm" {
  name = "${local.name_prefix}-web-ssm-policy"
  role = aws_iam_role.web.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ssm:GetParameter",
          "ssm:GetParameters",
          "ssm:GetParametersByPath"
        ]
        Resource = "arn:aws:ssm:${var.aws_region}:${var.account_id}:parameter/${var.environment}/shoplite/*"
      },
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = "${var.app_bucket_arn}/*"
      }
    ]
  })
}

resource "aws_iam_instance_profile" "web" {
  name = "${local.name_prefix}-web-profile"
  role = aws_iam_role.web.name
}

# EC2 Instance
resource "aws_instance" "web" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_ids[0]
  vpc_security_group_ids = [aws_security_group.web.id]
  key_name               = var.key_name
  iam_instance_profile   = aws_iam_instance_profile.web.name

  root_block_device {
    volume_size           = 20
    volume_type           = "gp3"
    encrypted             = true
    delete_on_termination = true
  }

  user_data = base64encode(templatefile("${path.module}/user_data.sh", {
    environment = var.environment
    app_port    = var.app_port
  }))

  tags = {
    Name = "${local.name_prefix}-web"
    Role = "web"
  }

  lifecycle {
    # Không destroy rồi tạo lại khi AMI thay đổi — tạo mới trước, xóa cũ sau
    create_before_destroy = true
    # Ignore thay đổi AMI (update qua Ansible, không phải Terraform)
    ignore_changes = [ami, user_data]
  }
}

# Elastic IP cho web server
resource "aws_eip" "web" {
  instance = aws_instance.web.id
  domain   = "vpc"

  tags = {
    Name = "${local.name_prefix}-web-eip"
  }
}
```

#### modules/database/main.tf

```hcl
# modules/database/main.tf

locals {
  name_prefix = "${var.environment}-shoplite"
}

# Security Group cho RDS — chỉ cho phép từ web servers
resource "aws_security_group" "rds" {
  name        = "${local.name_prefix}-rds-sg"
  description = "Security group for RDS database"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [var.web_sg_id]
    description     = "MySQL from web servers only"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${local.name_prefix}-rds-sg"
  }
}

# DB Subnet Group — RDS phải nằm trong private subnets
resource "aws_db_subnet_group" "main" {
  name        = "${local.name_prefix}-db-subnet-group"
  subnet_ids  = var.private_subnet_ids
  description = "DB subnet group for ShopLite"

  tags = {
    Name = "${local.name_prefix}-db-subnet-group"
  }
}

# RDS Parameter Group
resource "aws_db_parameter_group" "mysql" {
  family = "mysql8.0"
  name   = "${local.name_prefix}-mysql-params"

  parameter {
    name  = "character_set_server"
    value = "utf8mb4"
  }

  parameter {
    name  = "collation_server"
    value = "utf8mb4_unicode_ci"
  }

  parameter {
    name  = "slow_query_log"
    value = "1"
  }

  parameter {
    name  = "long_query_time"
    value = "2"
  }
}

# RDS Instance
resource "aws_db_instance" "main" {
  identifier     = "${local.name_prefix}-db"
  engine         = "mysql"
  engine_version = "8.0"
  instance_class = var.instance_class

  allocated_storage     = 20
  max_allocated_storage = 100  # Auto scaling storage tới 100GB
  storage_type          = "gp3"
  storage_encrypted     = true

  db_name  = var.db_name
  username = var.db_username
  password = var.db_password

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  parameter_group_name   = aws_db_parameter_group.mysql.name

  # Production settings
  multi_az               = var.environment == "production"
  publicly_accessible    = false
  deletion_protection    = var.environment == "production"

  # Backup
  backup_retention_period = var.environment == "production" ? 7 : 1
  backup_window           = "03:00-04:00"
  maintenance_window      = "sun:04:00-sun:05:00"

  # Monitoring
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn

  # Prevent accidental destruction in production
  skip_final_snapshot       = var.environment != "production"
  final_snapshot_identifier = var.environment == "production" ? "${local.name_prefix}-final-snapshot" : null

  tags = {
    Name = "${local.name_prefix}-db"
  }
}

# IAM Role cho Enhanced Monitoring
resource "aws_iam_role" "rds_monitoring" {
  name = "${local.name_prefix}-rds-monitoring-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "monitoring.rds.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "rds_monitoring" {
  role       = aws_iam_role.rds_monitoring.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole"
}
```

---

### 5. Remote State với S3 + DynamoDB

#### Tại sao cần Remote State

Khi làm việc một mình, local state (`terraform.tfstate`) có thể chấp nhận được. Nhưng khi làm team hoặc CI/CD:

- **Race condition:** Hai người apply cùng lúc → state conflict, hạ tầng hỏng.
- **Lost state:** Mất máy tính = mất state = Terraform không biết gì đang quản lý.
- **No sharing:** Người khác không thể chạy Terraform vì không có state.

S3 + DynamoDB giải quyết:
- S3 lưu state file (versioned, encrypted).
- DynamoDB làm lock: chỉ 1 người apply tại một thời điểm.

#### Bootstrap Script

```bash
#!/bin/bash
# scripts/bootstrap-tfstate.sh
# Chạy 1 lần để tạo S3 bucket và DynamoDB table
# SAU ĐÓ mới có thể dùng remote backend

set -euo pipefail

AWS_REGION="ap-southeast-1"
BUCKET_NAME="shoplite-tfstate"
DYNAMO_TABLE="shoplite-tflock"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo "=== Bootstrap Terraform Remote State ==="
echo "Account: ${ACCOUNT_ID}"
echo "Region:  ${AWS_REGION}"
echo "Bucket:  ${BUCKET_NAME}"
echo "Table:   ${DYNAMO_TABLE}"
echo ""

# 1. Tạo S3 bucket
echo ">>> Creating S3 bucket..."
aws s3api create-bucket \
  --bucket "${BUCKET_NAME}" \
  --region "${AWS_REGION}" \
  --create-bucket-configuration LocationConstraint="${AWS_REGION}"

# 2. Enable versioning (quan trọng: cho phép rollback state)
echo ">>> Enabling versioning..."
aws s3api put-bucket-versioning \
  --bucket "${BUCKET_NAME}" \
  --versioning-configuration Status=Enabled

# 3. Enable server-side encryption
echo ">>> Enabling encryption..."
aws s3api put-bucket-encryption \
  --bucket "${BUCKET_NAME}" \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms"
      },
      "BucketKeyEnabled": true
    }]
  }'

# 4. Block public access
echo ">>> Blocking public access..."
aws s3api put-public-access-block \
  --bucket "${BUCKET_NAME}" \
  --public-access-block-configuration '{
    "BlockPublicAcls": true,
    "IgnorePublicAcls": true,
    "BlockPublicPolicy": true,
    "RestrictPublicBuckets": true
  }'

# 5. Tạo DynamoDB table cho locking
echo ">>> Creating DynamoDB lock table..."
aws dynamodb create-table \
  --table-name "${DYNAMO_TABLE}" \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region "${AWS_REGION}"

echo ""
echo "=== Bootstrap complete! ==="
echo "Now add this to terraform/main.tf:"
echo ""
echo 'backend "s3" {'
echo "  bucket         = \"${BUCKET_NAME}\""
echo '  key            = "prod/terraform.tfstate"'
echo "  region         = \"${AWS_REGION}\""
echo '  encrypt        = true'
echo "  dynamodb_table = \"${DYNAMO_TABLE}\""
echo '}'
```

#### Migration từ Local sang Remote State

```bash
# Sau khi thêm backend "s3" vào main.tf
# Chạy lệnh này để migrate state hiện có lên S3
terraform init -migrate-state

# Terraform sẽ hỏi:
# Do you want to copy existing state to the new backend? yes

# Verify
terraform state list  # Phải thấy tất cả resources như trước
```

#### State Operations hữu ích

```bash
# Xem state
terraform state list
terraform state show aws_instance.web

# Move resource (rename trong state mà không destroy/recreate)
terraform state mv aws_instance.web aws_instance.app_server

# Remove resource khỏi state (Terraform quản lý nhưng không xóa actual resource)
terraform state rm aws_instance.old_server

# Pull state về local để inspect
terraform state pull > current-state.json

# Import resource đã tồn tại vào Terraform management
terraform import aws_s3_bucket.existing my-existing-bucket-name

# Xem lịch sử state trên S3
aws s3api list-object-versions \
  --bucket shoplite-tfstate \
  --prefix prod/terraform.tfstate \
  --query 'Versions[*].{Version:VersionId,Date:LastModified}'
```

---

### 6. Terraform trong CI/CD (GitHub Actions)

#### Workflow File Hoàn Chỉnh

```yaml
# .github/workflows/terraform.yml

name: Terraform Infrastructure

on:
  pull_request:
    paths:
      - 'terraform/**'
    branches: [main]
  push:
    paths:
      - 'terraform/**'
    branches: [main]

env:
  TF_VERSION: "1.6.6"
  AWS_REGION: "ap-southeast-1"
  TF_WORKING_DIR: "./terraform"

permissions:
  contents: read
  pull-requests: write  # Cần để comment plan output lên PR
  id-token: write       # Cần cho OIDC auth với AWS

jobs:
  terraform-plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC — không cần static credentials)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/github-actions-terraform
          aws-region: ${{ env.AWS_REGION }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        working-directory: ${{ env.TF_WORKING_DIR }}
        run: terraform init

      - name: Terraform Format Check
        working-directory: ${{ env.TF_WORKING_DIR }}
        run: terraform fmt -check -recursive
        continue-on-error: true

      - name: Terraform Validate
        working-directory: ${{ env.TF_WORKING_DIR }}
        run: terraform validate

      - name: Terraform Plan
        id: plan
        working-directory: ${{ env.TF_WORKING_DIR }}
        run: |
          terraform plan \
            -var="environment=production" \
            -var="db_password=${{ secrets.DB_PASSWORD }}" \
            -out=tfplan \
            -no-color 2>&1 | tee plan_output.txt
        continue-on-error: true

      - name: Comment Plan on PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const planOutput = fs.readFileSync(
              '${{ env.TF_WORKING_DIR }}/plan_output.txt', 'utf8'
            );
            const maxLength = 60000;
            const truncated = planOutput.length > maxLength
              ? planOutput.substring(0, maxLength) + '\n... (truncated)'
              : planOutput;

            const body = `## Terraform Plan Output

            <details>
            <summary>Click to expand plan</summary>

            \`\`\`
            ${truncated}
            \`\`\`

            </details>

            **Reviewer:** Please review the plan carefully before approving.
            `;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: body
            });

      - name: Upload Plan Artifact
        uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: ${{ env.TF_WORKING_DIR }}/tfplan
          retention-days: 7

  terraform-apply:
    name: Terraform Apply
    runs-on: ubuntu-latest
    needs: []  # Không phụ thuộc vào plan job
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment: production  # Require manual approval trong GitHub Settings

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/github-actions-terraform
          aws-region: ${{ env.AWS_REGION }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        working-directory: ${{ env.TF_WORKING_DIR }}
        run: terraform init

      - name: Terraform Apply
        working-directory: ${{ env.TF_WORKING_DIR }}
        run: |
          terraform apply \
            -var="environment=production" \
            -var="db_password=${{ secrets.DB_PASSWORD }}" \
            -auto-approve
```

#### IAM Role cho GitHub Actions (OIDC — không cần static credentials)

```hcl
# Trong Terraform, tạo IAM role cho GitHub Actions
data "aws_iam_policy_document" "github_actions_trust" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }

    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:your-org/shoplite:ref:refs/heads/main"]
    }
  }
}

resource "aws_iam_role" "github_actions_terraform" {
  name               = "github-actions-terraform"
  assume_role_policy = data.aws_iam_policy_document.github_actions_trust.json
}

# Attach policy — quyền tối thiểu cho Terraform operations
resource "aws_iam_role_policy_attachment" "github_actions_terraform" {
  role       = aws_iam_role.github_actions_terraform.name
  policy_arn = aws_iam_policy.terraform_deploy.arn
}
```

---

### 7. Ansible Fundamentals

#### Tại sao Ansible

Sau khi Terraform tạo EC2 instance, server đó chỉ là một Ubuntu bare-metal trống. Cần cài Docker, copy config, deploy application. Đây là việc của Ansible.

**Ưu điểm Ansible:**
- **Agentless:** Không cài gì trên server, chỉ cần SSH. Đơn giản hơn Chef/Puppet.
- **Push model:** Control node (laptop/CI) push config đến managed nodes. Không cần daemon.
- **Idempotent:** Mỗi Ansible module tự kiểm tra xem action có cần thiết không. Chạy 10 lần = chạy 1 lần.
- **Human-readable:** YAML syntax, dễ đọc, dễ học hơn Chef DSL hay Puppet manifests.
- **Large ecosystem:** 3000+ modules, Galaxy community roles.

#### Cài đặt Ansible

```bash
# Ubuntu/Debian
sudo apt update && sudo apt install -y python3-pip
pip3 install ansible

# macOS
brew install ansible

# Kiểm tra
ansible --version

# Cài thêm community collections
ansible-galaxy collection install community.docker
ansible-galaxy collection install community.general
ansible-galaxy collection install amazon.aws
```

#### Inventory File

```ini
# ansible/inventory.ini

# Web servers group
[web]
web1 ansible_host=54.123.456.789 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/deploy.pem

# Database servers (thường không access trực tiếp từ Ansible)
[db]
db1 ansible_host=10.0.11.5 ansible_user=ubuntu

# Staging environment
[staging:children]
web
db

# Variables cho tất cả hosts
[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args=-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null
```

**Dynamic Inventory từ Terraform output:**

```python
#!/usr/bin/env python3
# scripts/gen_inventory.py
# Dùng: terraform output -json | python3 scripts/gen_inventory.py

import json
import sys

tf_output = json.load(sys.stdin)

web_ip = tf_output.get("web_public_ip", {}).get("value", "")
db_endpoint = tf_output.get("rds_endpoint", {}).get("value", "")

inventory = f"""[web]
web1 ansible_host={web_ip} ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/deploy.pem

[all:vars]
ansible_python_interpreter=/usr/bin/python3
db_host={db_endpoint.split(':')[0]}
db_port=3306
"""

print(inventory)
```

```bash
# Tích hợp Terraform + Ansible
cd terraform/
terraform output -json | python3 ../ansible/scripts/gen_inventory.py > ../ansible/inventory.ini

cd ../ansible/
ansible-playbook -i inventory.ini site.yml
```

#### Test Connectivity

```bash
# Ping tất cả hosts
ansible all -i inventory.ini -m ping

# Chạy ad-hoc command
ansible web -i inventory.ini -m shell -a "uname -a"
ansible web -i inventory.ini -m shell -a "df -h"
ansible web -i inventory.ini -m shell -a "docker ps"

# Gather facts về server
ansible web -i inventory.ini -m setup | grep -E "ansible_os|ansible_distribution|ansible_memory"

# Copy file
ansible web -i inventory.ini -m copy -a "src=./nginx.conf dest=/tmp/nginx.conf"
```

---

### 8. Ansible Playbook Chi Tiết

#### Cấu trúc thư mục Ansible

```
ansible/
├── site.yml                     # Master playbook
├── inventory.ini                # Inventory (generated từ Terraform output)
├── ansible.cfg                  # Ansible configuration
│
├── group_vars/
│   ├── all.yml                  # Variables cho tất cả hosts
│   └── web.yml                  # Variables chỉ cho web group
│
├── host_vars/
│   └── web1.yml                 # Variables cho specific host (nếu cần)
│
├── vars/
│   └── secrets.yml              # Encrypted secrets (Ansible Vault)
│
└── roles/
    ├── common/
    │   ├── tasks/main.yml       # Basic setup: timezone, packages, users
    │   ├── handlers/main.yml
    │   └── files/
    ├── docker/
    │   ├── tasks/main.yml       # Install Docker, docker-compose
    │   └── handlers/main.yml
    ├── nginx/
    │   ├── tasks/main.yml       # Install nginx, configure reverse proxy
    │   ├── handlers/main.yml
    │   └── templates/
    │       └── nginx.conf.j2
    ├── app/
    │   ├── tasks/main.yml       # Clone repo, configure .env, start containers
    │   ├── handlers/main.yml
    │   └── templates/
    │       └── env.j2
    └── monitoring/
        ├── tasks/main.yml       # Node exporter, log rotation
        └── templates/
```

#### ansible.cfg

```ini
# ansible/ansible.cfg
[defaults]
inventory = inventory.ini
remote_user = ubuntu
private_key_file = ~/.ssh/deploy.pem
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml

# Tốc độ: dùng pipelining, mitogen
[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

#### site.yml (Master Playbook)

```yaml
# ansible/site.yml
---
# Play 1: Cấu hình tất cả servers (baseline)
- name: Common configuration for all servers
  hosts: all
  become: yes
  gather_facts: yes

  vars_files:
    - vars/secrets.yml

  roles:
    - common

# Play 2: Cấu hình web servers
- name: Configure web servers
  hosts: web
  become: yes

  vars:
    docker_version: "24.0"
    app_version: "{{ lookup('env', 'APP_VERSION') | default('main') }}"

  pre_tasks:
    - name: Update apt cache nếu cũ hơn 1 giờ
      apt:
        update_cache: yes
        cache_valid_time: 3600
      tags: [packages]

  roles:
    - docker
    - nginx
    - app

  post_tasks:
    - name: Verify application đang chạy
      uri:
        url: "http://localhost:{{ app_port }}/health"
        status_code: 200
      register: health_check
      retries: 5
      delay: 10
      until: health_check.status == 200
      tags: [verify]
```

#### roles/common/tasks/main.yml

```yaml
# ansible/roles/common/tasks/main.yml
---
- name: Set timezone to Asia/Ho_Chi_Minh
  community.general.timezone:
    name: Asia/Ho_Chi_Minh
  tags: [timezone]

- name: Install essential packages
  apt:
    name:
      - curl
      - wget
      - git
      - htop
      - unzip
      - jq
      - python3-pip
      - fail2ban
    state: present
  tags: [packages]

- name: Configure fail2ban
  copy:
    content: |
      [DEFAULT]
      bantime = 3600
      findtime = 600
      maxretry = 5

      [sshd]
      enabled = true
    dest: /etc/fail2ban/jail.local
  notify: restart fail2ban
  tags: [security]

- name: Create deploy user
  user:
    name: deploy
    shell: /bin/bash
    groups: sudo
    append: yes
    create_home: yes
  tags: [users]

- name: Setup log rotation cho app logs
  copy:
    content: |
      /app/shoplite/logs/*.log {
        daily
        rotate 14
        compress
        delaycompress
        missingok
        notifempty
        sharedscripts
      }
    dest: /etc/logrotate.d/shoplite
  tags: [logging]
```

#### roles/docker/tasks/main.yml

```yaml
# ansible/roles/docker/tasks/main.yml
---
- name: Remove old Docker versions nếu có
  apt:
    name:
      - docker
      - docker-engine
      - docker.io
      - containerd
      - runc
    state: absent
  tags: [docker]

- name: Install Docker prerequisites
  apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg
      - lsb-release
    state: present
  tags: [docker]

- name: Add Docker GPG key
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present
  tags: [docker]

- name: Add Docker APT repository
  apt_repository:
    repo: >
      deb [arch=amd64]
      https://download.docker.com/linux/ubuntu
      {{ ansible_distribution_release }} stable
    state: present
    filename: docker
  tags: [docker]

- name: Install Docker CE và Docker Compose plugin
  apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin
    state: present
    update_cache: yes
  tags: [docker]

- name: Start và enable Docker service
  service:
    name: docker
    state: started
    enabled: yes
  tags: [docker]

- name: Add ubuntu user vào docker group
  user:
    name: ubuntu
    groups: docker
    append: yes
  tags: [docker]

- name: Verify Docker installation
  command: docker --version
  register: docker_version_result
  changed_when: false
  tags: [docker, verify]

- name: Print Docker version
  debug:
    msg: "Docker installed: {{ docker_version_result.stdout }}"
  tags: [docker, verify]
```

#### roles/nginx/tasks/main.yml và template

```yaml
# ansible/roles/nginx/tasks/main.yml
---
- name: Install nginx
  apt:
    name: nginx
    state: present

- name: Remove default nginx config
  file:
    path: /etc/nginx/sites-enabled/default
    state: absent
  notify: reload nginx

- name: Configure nginx reverse proxy cho ShopLite
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/sites-available/shoplite
    owner: root
    group: root
    mode: '0644'
  notify: reload nginx

- name: Enable ShopLite nginx config
  file:
    src: /etc/nginx/sites-available/shoplite
    dest: /etc/nginx/sites-enabled/shoplite
    state: link
  notify: reload nginx

- name: Start và enable nginx
  service:
    name: nginx
    state: started
    enabled: yes

- name: Test nginx configuration
  command: nginx -t
  changed_when: false
```

```nginx
# ansible/roles/nginx/templates/nginx.conf.j2
# Template Jinja2 — {{ variable }} và {% logic %}

upstream shoplite_backend {
    server 127.0.0.1:{{ app_port }};
    keepalive 32;
}

server {
    listen 80;
    server_name {{ ansible_host }} _;

    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";

    # Redirect HTTP -> HTTPS nếu production
    {% if environment == "production" %}
    return 301 https://$host$request_uri;
    {% else %}
    # Staging: serve trực tiếp qua HTTP

    location / {
        proxy_pass http://shoplite_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 30s;
        proxy_read_timeout 60s;
    }

    location /health {
        proxy_pass http://shoplite_backend/health;
        access_log off;
    }
    {% endif %}
}
```

#### roles/app/tasks/main.yml

```yaml
# ansible/roles/app/tasks/main.yml
---
- name: Create application directory
  file:
    path: /app/shoplite
    state: directory
    owner: ubuntu
    group: ubuntu
    mode: '0755'

- name: Create logs directory
  file:
    path: /app/shoplite/logs
    state: directory
    owner: ubuntu
    group: ubuntu
    mode: '0755'

- name: Clone hoặc update repository
  git:
    repo: "https://github.com/your-org/shoplite.git"
    dest: /app/shoplite
    version: "{{ app_version | default('main') }}"
    force: yes
  become: yes
  become_user: ubuntu
  register: git_result

- name: Copy .env file từ template
  template:
    src: env.j2
    dest: /app/shoplite/.env
    owner: ubuntu
    group: ubuntu
    mode: '0600'  # Chỉ owner đọc được — file chứa secrets
  notify: restart app

- name: Pull Docker images mới nhất
  community.docker.docker_compose_v2:
    project_src: /app/shoplite
    pull: always
    state: present
  become: yes
  become_user: ubuntu
  when: git_result.changed

- name: Start application với docker compose
  community.docker.docker_compose_v2:
    project_src: /app/shoplite
    state: present
  become: yes
  become_user: ubuntu
```

```ini
# ansible/roles/app/templates/env.j2
# Template cho .env file của application

# Application
APP_ENV={{ environment }}
APP_PORT={{ app_port }}
APP_SECRET_KEY={{ app_secret_key }}

# Database
DB_HOST={{ db_host }}
DB_PORT={{ db_port | default(3306) }}
DB_NAME={{ db_name }}
DB_USER={{ db_username }}
DB_PASSWORD={{ db_password }}

# AWS
AWS_REGION={{ aws_region }}
S3_BUCKET={{ s3_bucket_name }}

# Logging
LOG_LEVEL={{ 'INFO' if environment == 'production' else 'DEBUG' }}
```

#### roles/app/handlers/main.yml

```yaml
# ansible/roles/app/handlers/main.yml
---
- name: restart app
  community.docker.docker_compose_v2:
    project_src: /app/shoplite
    state: restarted
  become: yes
  become_user: ubuntu

- name: reload nginx
  service:
    name: nginx
    state: reloaded

- name: restart fail2ban
  service:
    name: fail2ban
    state: restarted
```

---

### 9. Ansible Vault cho Secrets

#### Tại sao cần Ansible Vault

File `vars/secrets.yml` chứa passwords, API keys, secret keys. Không thể commit plaintext vào git. Ansible Vault mã hóa file với AES-256.

```bash
# Tạo file secrets mới (sẽ mở editor để nhập nội dung)
ansible-vault create ansible/vars/secrets.yml

# Nội dung của secrets.yml (plaintext trước khi mã hóa):
# ---
# db_password: "StrongPasswordHere123!"
# app_secret_key: "random-64-char-secret-key"
# s3_bucket_name: "shoplite-assets-prod"

# Xem nội dung file đã mã hóa (sẽ hỏi password)
ansible-vault view ansible/vars/secrets.yml

# Chỉnh sửa (decrypt, mở editor, re-encrypt khi lưu)
ansible-vault edit ansible/vars/secrets.yml

# Mã hóa file đã tồn tại
ansible-vault encrypt ansible/vars/secrets.yml

# Giải mã file (để plaintext — cẩn thận không commit)
ansible-vault decrypt ansible/vars/secrets.yml

# Mã hóa một chuỗi cụ thể (nhúng inline vào playbook)
ansible-vault encrypt_string 'StrongPasswordHere123!' --name 'db_password'
# Output có thể paste trực tiếp vào YAML file

# Đổi vault password
ansible-vault rekey ansible/vars/secrets.yml
```

#### Chạy Playbook với Vault

```bash
# Option 1: Hỏi password khi chạy
ansible-playbook -i inventory.ini site.yml --ask-vault-pass

# Option 2: Password trong file (không commit file này!)
echo "vault_password" > ~/.vault_pass
chmod 600 ~/.vault_pass
ansible-playbook -i inventory.ini site.yml --vault-password-file ~/.vault_pass

# Option 3: Environment variable
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass
ansible-playbook -i inventory.ini site.yml

# Option 4: Trong CI/CD — lấy từ secret manager
# .github/workflows/deploy.yml
# - run: |
#     echo "${{ secrets.ANSIBLE_VAULT_PASSWORD }}" > /tmp/vault_pass
#     ansible-playbook -i inventory.ini site.yml --vault-password-file /tmp/vault_pass
#     rm /tmp/vault_pass
```

#### ansible.cfg với vault configuration

```ini
# ansible/ansible.cfg
[defaults]
vault_password_file = ~/.vault_pass
```

---

### 10. Terraform vs Ansible — Đúng Tool Đúng Việc

#### Nguyên tắc phân chia

| Tiêu chí | Terraform | Ansible |
|---|---|---|
| Mục đích chính | Provision infrastructure | Configure software |
| Thứ tự workflow | Chạy TRƯỚC | Chạy SAU |
| Ví dụ | Tạo VPC, EC2, RDS | Cài Docker, deploy app |
| Paradigm | Declarative | Imperative (idempotent) |
| State management | Có state file | Stateless |
| Agentless | Giao tiếp qua AWS API | Giao tiếp qua SSH |

**Terraform làm tốt:**
- Tạo, modify, xóa cloud resources (AWS, GCP, Azure, Kubernetes).
- Quản lý lifecycle của infrastructure.
- Dependency graph tự động (tạo VPC trước, rồi subnet, rồi EC2).
- Cross-provider resources (AWS + Cloudflare + GitHub trong 1 config).

**Ansible làm tốt:**
- Cài đặt packages, configure services.
- Deploy application, update config files.
- Ad-hoc tasks (restart service, run migration, check status).
- Configuration management khi không dùng containers.

**Không nên dùng Terraform cho:**
- Cài packages trên server (dùng Ansible hoặc user_data script).
- Deploy application code (dùng Ansible, CI/CD pipeline).
- Running commands inside server (dùng Ansible).

**Không nên dùng Ansible cho:**
- Tạo cloud resources (dùng Terraform).
- Managing state phức tạp (Ansible không track state).

#### Full Pipeline: Terraform + Ansible

```bash
#!/bin/bash
# scripts/full-deploy.sh
# Deploy toàn bộ hạ tầng và application từ đầu

set -euo pipefail

echo "=== STEP 1: Provision Infrastructure với Terraform ==="
cd terraform/

terraform init
terraform validate
terraform plan -var-file="prod.tfvars" -out=tfplan

echo ">>> Review plan above. Proceed? (yes/no)"
read -r confirm
if [ "$confirm" != "yes" ]; then
  echo "Aborted."
  exit 1
fi

terraform apply tfplan

echo ""
echo "=== STEP 2: Generate Ansible Inventory từ Terraform Output ==="
terraform output -json | python3 ../ansible/scripts/gen_inventory.py > ../ansible/inventory.ini
echo "Generated inventory:"
cat ../ansible/inventory.ini

echo ""
echo "=== STEP 3: Wait for EC2 instance to be ready ==="
WEB_IP=$(terraform output -raw web_public_ip)
echo "Waiting for SSH on ${WEB_IP}..."
until ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 \
    -i ~/.ssh/deploy.pem ubuntu@${WEB_IP} "echo ready" 2>/dev/null; do
  echo "  Not ready yet, retrying in 10s..."
  sleep 10
done
echo "SSH is up!"

echo ""
echo "=== STEP 4: Configure Server với Ansible ==="
cd ../ansible/

# Kiểm tra connectivity
ansible all -i inventory.ini -m ping

# Run full playbook
ansible-playbook -i inventory.ini site.yml \
  --vault-password-file ~/.vault_pass \
  --extra-vars "environment=production app_version=${APP_VERSION:-main}"

echo ""
echo "=== STEP 5: Verify Deployment ==="
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "http://${WEB_IP}/health")
if [ "$HTTP_STATUS" = "200" ]; then
  echo "Application is healthy! HTTP ${HTTP_STATUS}"
else
  echo "ERROR: Application returned HTTP ${HTTP_STATUS}"
  exit 1
fi

echo ""
echo "=== Deployment complete! ==="
echo "Application URL: http://${WEB_IP}"
```

#### Dynamic Inventory Script hoàn chỉnh

```python
#!/usr/bin/env python3
# ansible/scripts/gen_inventory.py
"""
Generate Ansible inventory từ Terraform output.
Usage: terraform output -json | python3 scripts/gen_inventory.py
"""

import json
import sys
import argparse


def parse_tf_output(tf_output: dict) -> dict:
    """Parse Terraform output JSON và extract relevant values."""
    return {
        "web_ip": tf_output.get("web_public_ip", {}).get("value", ""),
        "db_host": tf_output.get("rds_endpoint", {}).get("value", "").split(":")[0],
        "db_port": "3306",
        "s3_bucket": tf_output.get("s3_bucket_name", {}).get("value", ""),
        "environment": tf_output.get("environment", {}).get("value", "staging"),
    }


def generate_inventory(values: dict) -> str:
    """Generate INI-format Ansible inventory."""
    return f"""[web]
web1 ansible_host={values['web_ip']} ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/deploy.pem

[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args=-o StrictHostKeyChecking=no

db_host={values['db_host']}
db_port={values['db_port']}
s3_bucket_name={values['s3_bucket']}
environment={values['environment']}
aws_region=ap-southeast-1
"""


def main():
    try:
        tf_output = json.load(sys.stdin)
        values = parse_tf_output(tf_output)

        if not values["web_ip"]:
            print("ERROR: Could not find web_public_ip in Terraform output", file=sys.stderr)
            sys.exit(1)

        print(generate_inventory(values))

    except json.JSONDecodeError as e:
        print(f"ERROR: Invalid JSON from terraform output: {e}", file=sys.stderr)
        sys.exit(1)


if __name__ == "__main__":
    main()
```

#### Checklist trước khi apply Terraform

```bash
# 1. Format code
terraform fmt -recursive -check

# 2. Validate
terraform validate

# 3. Security scan
tfsec terraform/
# hoặc
checkov -d terraform/ --framework terraform

# 4. Cost estimate
infracost breakdown --path terraform/

# 5. Plan review
terraform plan -out=tfplan
# Đọc kỹ output:
# + create (màu xanh): resource mới
# ~ update (màu vàng): thay đổi in-place
# - destroy (màu đỏ): XÓA resource — CẦN XEM XÉT KỸ

# 6. Chú ý đặc biệt với:
# -/+ replace (xóa rồi tạo lại): downtime!
# Plan: 0 to add, 0 to change, 1 to destroy — nguy hiểm!

# 7. Apply
terraform apply tfplan
```

---

## Kỹ năng đạt được
- Viết code dựng hạ tầng cloud lặp lại được.
- Tự động cấu hình server bằng Ansible.
- Quản lý hạ tầng như quản lý phần mềm (qua Git).

---

## Thực hành

**Môi trường:** Terraform + Ansible + AWS.

**Lab:**
- Viết Terraform dựng lại VPC, EC2, security group, RDS từ Phase 9.
- Lưu state trên S3 với DynamoDB lock.
- Viết Ansible playbook cài Docker + cấu hình server tự động.
- Phá đi rồi dựng lại toàn bộ hạ tầng chỉ bằng lệnh.

**Công cụ:** Terraform, Ansible, AWS.

---

## Bài tập

- **Bắt buộc:** `terraform apply` dựng được toàn bộ hạ tầng ShopLite từ đầu.
- **Nâng cao:** Module hóa Terraform (network, compute, database tách riêng) + remote state.
- **Mô phỏng doanh nghiệp:** Tích hợp `terraform plan` vào CI để review hạ tầng trước khi apply.

---

## Deliverable
- [ ] Thư mục `terraform/` dựng toàn bộ hạ tầng.
- [ ] Thư mục `ansible/` cấu hình server.
- [ ] Tài liệu hướng dẫn dựng hạ tầng từ con số 0.

---

## Tiêu chí hoàn thành
- [ ] Xóa sạch hạ tầng rồi dựng lại hoàn chỉnh chỉ bằng code.
- [ ] Terraform state được quản lý từ xa, an toàn.
- [ ] Giải thích được khác biệt provisioning vs configuration.

---

## Ghi chú học tập

> _Ghi lại những điều học được, vướng mắc và insight trong quá trình học phase này._
