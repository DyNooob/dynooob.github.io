---
layout: post
title: "Terraform 基础设施即代码实战指南"
date: 2026-08-14 21:00:00 +0800
categories: [开发]
tags: [Terraform, IaC, DevOps, 基础设施, 云计算, 自动化, 最佳实践]
---

## 为什么需要 Terraform

现代应用架构越来越复杂：一台虚拟机不够用，你需要负载均衡、数据库、对象存储、CDN、DNS 解析、容器编排……手动在云控制台上点来点去，不仅效率低、容易出错，而且无法复现。更糟糕的是，不同环境（开发、测试、生产）之间的配置差异，往往导致"在我机器上好好的"这类经典问题。

Terraform 是 HashiCorp 推出的基础设施即代码（Infrastructure as Code, IaC）工具。它用声明式的 HCL 语言描述云资源，一条 `terraform apply` 就能创建或更新整个基础设施。相比 Ansible、Puppet 这类配置管理工具，Terraform 专注于**资源编排**——它关心的是"有什么资源"而不是"服务器里装了什么软件"。换句话说，Terraform 管的是基础设施的"生老病死"，Ansible 管的是装好之后的"衣食住行"。

Terraform 的核心优势可以概括为四点：

- **声明式配置**：你描述想要的最终状态，Terraform 计算如何达到它
- **执行计划**：`terraform plan` 展示所有变更，不会有意外操作
- **资源图**：自动计算依赖关系，并行创建无关资源
- **状态管理**：跟踪每个资源的元数据，知道哪些已经存在

本文从零开始，带你搭建一个完整的 Terraform 项目，覆盖核心概念、状态管理、模块化设计、CI/CD 集成，以及生产环境中常见的坑。

## 安装与初始化

### 安装 Terraform

Linux 下安装很简单：

```bash
# 下载最新版本
wget https://releases.hashicorp.com/terraform/1.9.5/terraform_1.9.5_linux_amd64.zip
unzip terraform_1.9.5_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# 验证
terraform version
```

或者用包管理器（版本可能不是最新）：

```bash
# Ubuntu/Debian
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install terraform

# macOS
brew install terraform
```

### 第一个项目

创建项目目录，写一个最简单的配置文件：

```hcl
# main.tf
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
  }
}

resource "local_file" "hello" {
  content  = "Hello, Terraform!"
  filename = "${path.module}/hello.txt"
}
```

初始化并执行：

```bash
terraform init
terraform plan
terraform apply -auto-approve
cat hello.txt  # 输出: Hello, Terraform!
terraform destroy  # 清理
```

`terraform init` 下载 provider 插件，`plan` 预览变更，`apply` 执行，`destroy` 销毁。这个工作流就是 Terraform 的核心循环。

## 核心概念

### Provider

Provider 是 Terraform 与云平台/服务交互的插件。每个 provider 负责一组资源和数据源，可以理解为"Terraform 的翻译官"——它知道如何通过云平台的 API 创建、读取、更新、删除资源。

配置 provider 时需要指定 `source` 和 `version`。`source` 的格式是 `[registry_host/]namespace/type`，默认从 Terraform Registry 下载。常用 provider 包括：

- `hashicorp/aws` — Amazon Web Services（EC2、S3、RDS 等）
- `hashicorp/azurerm` — Microsoft Azure
- `hashicorp/google` — Google Cloud Platform
- `aliyun/alicloud` — 阿里云
- `hashicorp/kubernetes` — Kubernetes API
- `hashicorp/helm` — Helm Chart 部署
- `cloudflare/cloudflare` — DNS、CDN、WAF
- `kreuzwerker/docker` — Docker 容器管理

Provider 可以指定 `alias` 来管理多个同一类型的云账号：

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

resource "aws_instance" "web" {
  # 使用默认 provider
}

resource "aws_instance" "backup" {
  provider = aws.west
  # 使用 alias 为 west 的 provider
}
```

### Resource

Resource 是 Terraform 管理的核心对象，代表一个真实的基础设施组件。语法：

```hcl
resource "provider_type" "local_name" {
  argument1 = "value1"
  argument2 = "value2"
}
```

`provider_type` 是 provider 提供的资源类型（如 `aws_instance`、`alicloud_security_group`），`local_name` 是你在代码中引用它的逻辑名称。同一个资源类型可以有多个实例，`local_name` 用来区分它们。

资源引用使用 `resource_type.local_name.attribute` 语法。例如：

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

output "web_ip" {
  value = aws_instance.web.public_ip
}
```

Terraform 会自动分析这些引用，构建依赖图，确保资源按正确的顺序创建销毁。

### Variable 与 Output

变量让配置可复用，输出暴露有用信息：

```hcl
# variables.tf
variable "region" {
  description = "云区域"
  type        = string
  default     = "cn-hangzhou"
}

variable "instance_type" {
  description = "实例规格"
  type        = string
}

# outputs.tf
output "instance_ip" {
  description = "实例公网 IP"
  value       = alicloud_instance.web.public_ip
}
```

变量赋值方式（优先级从低到高）：

1. 默认值 `default`
2. 环境变量 `TF_VAR_region=cn-shanghai`
3. `terraform.tfvars` 文件
4. `*.auto.tfvars` 文件
5. 命令行 `-var="region=cn-shanghai"`

### Data Source

数据源用于读取已有资源的信息，而不是创建新资源：

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-22.04-amd64-server-*"]
  }
  owners = ["099720109477"]
}
```

## 实战：部署一个 Web 应用

我们以阿里云为例，部署一个完整的 Web 应用：VPC + 交换机 + 安全组 + ECS 实例。

### 目录结构

```
terraform-webapp/
├── main.tf           # 主配置文件
├── variables.tf      # 变量定义
├── outputs.tf        # 输出定义
├── terraform.tfvars  # 变量值
└── modules/
    └── vpc/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

### 主配置

```hcl
# main.tf
terraform {
  required_version = ">= 1.6"
  required_providers {
    alicloud = {
      source  = "aliyun/alicloud"
      version = "~> 1.230"
    }
  }
}

provider "alicloud" {
  region = var.region
}

# 使用模块创建 VPC
module "vpc" {
  source = "./modules/vpc"

  vpc_name   = "${var.project_name}-vpc"
  vpc_cidr   = "10.0.0.0/16"
  zone_id    = var.zone_id
}

# 安全组
resource "alicloud_security_group" "web" {
  name        = "${var.project_name}-sg"
  description = "Web server security group"
  vpc_id      = module.vpc.vpc_id
}

resource "alicloud_security_group_rule" "allow_http" {
  type              = "ingress"
  ip_protocol       = "tcp"
  policy            = "accept"
  port_range        = "80/80"
  priority          = 1
  security_group_id = alicloud_security_group.web.id
  cidr_ip           = "0.0.0.0/0"
}

resource "alicloud_security_group_rule" "allow_ssh" {
  type              = "ingress"
  ip_protocol       = "tcp"
  policy            = "accept"
  port_range        = "22/22"
  priority          = 1
  security_group_id = alicloud_security_group.web.id
  cidr_ip           = var.admin_ip  # 只允许管理员 IP
}

# ECS 实例
resource "alicloud_instance" "web" {
  instance_name        = "${var.project_name}-ecs"
  instance_type        = var.instance_type
  image_id             = "ubuntu_22_04_x64_20G_alibase_20240628.vhd"
  vswitch_id           = module.vpc.vswitch_id
  security_groups      = [alicloud_security_group.web.id]
  internet_max_bandwidth_out = 100
  system_disk_category = "cloud_essd"
  system_disk_size     = 40

  user_data = <<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl enable nginx
    systemctl start nginx
    echo "<h1>Deployed by Terraform</h1>" > /var/www/html/index.html
  EOF
}
```

### VPC 模块

```hcl
# modules/vpc/main.tf
variable "vpc_name" {
  type = string
}
variable "vpc_cidr" {
  type = string
}
variable "zone_id" {
  type = string
}

resource "alicloud_vpc" "main" {
  vpc_name   = var.vpc_name
  cidr_block = var.vpc_cidr
}

resource "alicloud_vswitch" "main" {
  vswitch_name = "${var.vpc_name}-vswitch"
  cidr_block   = cidrsubnet(var.vpc_cidr, 8, 1)
  vpc_id       = alicloud_vpc.main.id
  zone_id      = var.zone_id
}

output "vpc_id" {
  value = alicloud_vpc.main.id
}

output "vswitch_id" {
  value = alicloud_vswitch.main.id
}
```

### 变量文件

```hcl
# terraform.tfvars
region         = "cn-hangzhou"
zone_id        = "cn-hangzhou-h"
project_name   = "terraform-webapp"
instance_type  = "ecs.g7.large"
admin_ip       = "114.114.114.114/32"  # 替换为你的公网 IP
```

### 执行

```bash
terraform init
terraform fmt          # 格式化代码
terraform validate     # 验证语法
terraform plan -out=tfplan
terraform apply tfplan

# 看到输出中的公网 IP 后，浏览器访问 http://<IP>
```

## 状态管理

### 为什么状态重要

Terraform 用状态文件（`terraform.tfstate`）记录已创建的资源及其属性。状态是 Terraform 的"数据库"——没有它，Terraform 不知道哪些资源已经存在，也无法在后续 apply 中判断哪些需要更新、哪些需要删除。

状态文件是 JSON 格式，包含每个资源的 `id`、`attributes`、`dependencies` 等元数据。每次 `apply` 后，Terraform 都会更新状态文件，确保它反映真实世界的基础设施。

### 本地状态的问题

本地状态文件看似方便，但生产环境中有几个致命问题：

- **团队协作**：A 先 apply，B 后 apply —— B 的本地状态文件看不到 A 的变更，导致冲突或资源重复创建
- **数据丢失**：本地文件损坏、误删或笔记本丢失，Terraform 和基础设施之间就"失联"了
- **无锁机制**：多人同时 `terraform apply`，会破坏状态文件，造成不可恢复的损坏
- **CI/CD 不可用**：流水线每次运行时，需要拿到最新的状态文件，本地文件显然做不到

### 远程状态存储

生产环境必须用远程后端存储状态。推荐用阿里云 OSS（或 AWS S3）：

```hcl
# backend.tf
terraform {
  backend "oss" {
    bucket  = "my-terraform-state"
    prefix  = "prod/terraform-webapp"
    key     = "terraform.tfstate"
    region  = "cn-hangzhou"
  }
}
```

首次配置 backend 需要先手动创建 OSS Bucket，然后运行 `terraform init -migrate-state` 迁移本地状态。

### 状态锁

远程后端自动支持状态锁。OSS 后端使用 DynamoDB（阿里云 OTS）防止并发 apply。配置：

```hcl
terraform {
  backend "oss" {
    bucket         = "my-terraform-state"
    prefix         = "prod/terraform-webapp"
    key            = "terraform.tfstate"
    region         = "cn-hangzhou"
    tablestore_endpoint = "https://terraform-lock.cn-hangzhou.ots.aliyuncs.com"
    tablestore_table     = "terraform_lock"
  }
}
```

## 模块化设计

### 什么时候用模块

- 资源被多个项目复用（VPC 模块、RDS 模块）
- 逻辑边界清晰（网络层、计算层、存储层）
- 团队需要统一规范

### 从 Terraform Registry 引用模块

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.13.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
  enable_vpn_gateway = false
  tags = {
    Environment = "production"
  }
}
```

### 模块版本管理

模块应该有语义化版本。在 `source` 中指定版本约束：

```hcl
module "database" {
  source  = "git::https://github.com/team/infra-modules.git//rds?ref=v1.2.0"
  # 或从本地路径
  # source = "./modules/rds"
}
```

## CI/CD 集成

将 Terraform 集成到 CI/CD 流水线中，实现基础设施的自动化变更。

### GitHub Actions 示例

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches: [main]
    paths:
      - 'terraform/**'

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: terraform

    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.5

      - name: Terraform Init
        run: terraform init

      - name: Terraform Format
        run: terraform fmt -check

      - name: Terraform Plan
        run: terraform plan -out=tfplan

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve tfplan
```

### 审批流程（Pull Request 模式）

推荐做法：PR 中自动执行 `terraform plan`，合并后自动执行 `apply`。

```yaml
# PR 中只做 plan
name: Terraform Plan
on: [pull_request]
jobs:
  plan:
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - run: terraform plan
```

## 生产环境最佳实践

### 1. 目录结构规范

```
environments/
├── production/
│   ├── main.tf
│   ├── variables.tf
│   └── terraform.tfvars
└── staging/
    ├── main.tf
    ├── variables.tf
    └── terraform.tfvars
modules/
├── vpc/
├── ecs/
└── rds/
```

每个环境独立目录，避免误操作影响生产环境。

### 2. 敏感信息管理

绝不能把密钥明文写在代码里。用环境变量或 secret store：

```bash
export TF_VAR_db_password="your-secure-password"
```

或者使用 `sops` 加密变量文件：

```bash
sops encrypt terraform.tfvars > terraform.tfvars.enc
```

更推荐的做法：使用 Vault 或 AWS Secrets Manager 动态获取密钥。

### 3. 锁定 Terraform 版本

```hcl
terraform {
  required_version = "~> 1.9.0"
  required_providers {
    alicloud = {
      source  = "aliyun/alicloud"
      version = "~> 1.230.0"
    }
  }
}
```

版本约束的 `~> X.Y.Z` 表示允许补丁版本升级，但不允许跨 minor 版本升级。

### 4. 使用 `terraform plan` 做变更评审

任何基础设施变更，先 `plan` 再 review。关注以下信号：

- 资源被标记为 `destroy`（是不是不小心删了？）
- 强制替换 `force new`（重建会丢数据吗？）
- 预期外的变更（谁改了共享的状态？）

### 5. 避免手动修改

```bash
# 永远不要这样做
terraform state rm alicloud_instance.web
terraform import alicloud_instance.web i-xxxxx
```

手动操作状态是灾难的开始。如果资源需要修改，改代码然后 `apply`。

### 6. 资源命名规范

一致的命名规则让基础设施可读可查：

```
{project}-{environment}-{resource_type}-{purpose}
# 示例: blog-prod-ecs-web, blog-staging-rds-main
```

### 7. 合理使用 `lifecycle` 元参数

```hcl
resource "aws_db_instance" "main" {
  # ...

  lifecycle {
    prevent_destroy = true          # 防止误删
    ignore_changes  = [tags]        # 忽略外部对 tags 的修改
    create_before_destroy = true    # 先创建再销毁（零停机）
  }
}
```

## 常见陷阱

### 陷阱 1：状态文件泄露

`terraform.tfstate` 包含明文密码和密钥。必须：
- 加入 `.gitignore`
- 始终使用远程后端
- 对 state 文件设置访问控制

### 陷阱 2：忘记 `terraform plan` 直接 apply

尤其在多人协作时，`plan` 是发现问题的最后机会。养成习惯：先 `plan`，确认无误后 `apply`。

### 陷阱 3：依赖循环

模块 A 引用模块 B 的输出，模块 B 又引用模块 A 的输出，导致循环依赖。解决方案：重新设计模块边界，或者将共享资源提取到父模块。

### 陷阱 4：硬编码

```hcl
# 错误的做法
resource "alicloud_instance" "web" {
  instance_type = "ecs.g7.large"  # 硬编码
}

# 正确的做法
variable "instance_type" {
  type    = string
  default = "ecs.g7.large"
}
resource "alicloud_instance" "web" {
  instance_type = var.instance_type
}
```

### 陷阱 5：忽略 `terraform validate`

语法错误会在 `apply` 时才发现，但 `validate` 可以提前捕获。顺手加进 pre-commit hook：

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.96.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
```

## 总结

Terraform 将基础设施管理从手动点击变成代码化的流程，其核心价值在于：

- **可复现**：同样的代码，在任何环境创建同样的基础设施
- **可审计**：每一次变更通过 Git 记录，谁改了什么一目了然
- **可自动化**：与 CI/CD 集成，实现基础设施的持续交付
- **可组合**：模块化设计，团队共享最佳实践

从一个小项目开始，用远程状态管理，逐步建立模块库，集成 CI/CD——这就是一条平滑的 Terraform 进阶之路。基础设施即代码不是终点，而是 DevOps 文化的起点。