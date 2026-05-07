# Terraform IaC 基础

> **学习日期**: 2026-04-22  
> **实验环境**: 阿里云（新账号免费额度）  
> **实验目标**: 用 Terraform 定义 AI 训练集群基础设施

---

## 一、核心概念

### IaC (Infrastructure as Code)

用代码定义基础设施，版本控制、可重复、可审计。

### Terraform 工作流

```
Write → Plan → Apply
写代码 → 预览变更 → 执行
```

### 声明式 vs 命令式

- **声明式**（Terraform）：描述期望状态，Terraform 计算如何达到
- **命令式**（Shell 脚本）：描述操作步骤

---

## 二、HCL 语法要点

### 资源引用语法

```hcl
# 格式: 资源类型.资源名称.属性
alicloud_vpc.ai_vpc.id
#        ↑      ↑    ↑
#      类型   名称  属性

# _ 是名字的一部分，. 是访问符
alicloud_security_group.ai_sg.id  # ✓ 正确
alicloud_security_group_ai_sg.id  # ✗ 错误
```

### 版本约束语法

```hcl
version = "~> 1.210.0"   # 悲观约束：>=1.210.0, <1.211.0
version = "<= 2.5.1"     # 小于等于
version = ">= 1.0, < 2.0" # 范围
```

### count 循环 + splat 表达式

```hcl
# 创建多个实例
resource "alicloud_instance" "ai_node" {
  count = var.instance_count
  instance_name = "ai-node-${count.index}"  # 索引从 0 开始
}

# 批量取值
output "node_ips" {
  value = alicloud_instance.ai_node[*].private_ip  # splat: 取所有实例的 private_ip
}
```

### 文件命名规则

**唯一硬性要求：以 `.tf` 结尾**

Terraform 加载目录下所有 `.tf` 文件并合并。文件名是约定，非强制：


| 文件名                            | 内容                 |
| ------------------------------ | ------------------ |
| `main.tf` / `*.tf`             | 资源定义（按职责或类型分文件均可）  |
| `variables.tf`                 | 输入变量               |
| `outputs.tf`                   | 输出值                |
| `providers.tf` / `versions.tf` | Provider 配置 + 版本约束 |
| `terraform.tfvars`             | 变量赋值（**自动加载**）     |


---

## 三、CLI 命令


| 命令                                | 作用              |
| --------------------------------- | --------------- |
| `terraform init`                  | 初始化，下载 Provider |
| `terraform plan`                  | 预览变更（不执行）       |
| `terraform plan -var="key=value"` | 带变量预览           |
| `terraform apply`                 | 执行变更            |
| `terraform destroy`               | 销毁资源            |
| `terraform fmt`                   | 格式化代码           |
| `terraform validate`              | 验证语法            |


---

## 四、实验：阿里云 AI 训练集群

### 4.1 目录结构

```
terraform-ai-cluster/
├── versions.tf      # Provider 配置
├── variables.tf     # 变量定义
├── network.tf       # VPC + VSwitch + 安全组
└── compute.tf       # ECS 实例 + 输出
```

### 4.2 配置文件

#### versions.tf - Provider 配置

```hcl
terraform {
  required_providers {
    alicloud = {
      source = "aliyun/alicloud"
      version = "~> 1.210.0"
    }
  }
}

provider "alicloud" {}
```

#### variables.tf - 输入变量

```hcl
variable "instance_count"{
  description = "number of nodes in the cluster"
  type = number
  default = 2
}

variable "instance_type"{
  description = "ECS instance type"
  type = string
  default = "ecs.c6.large"
}
```

#### network.tf - 网络配置

```hcl
resource "alicloud_vpc""ai_vpc"{
  vpc_name = "ai-training-vpc"
  cidr_block = "172.16.0.0/12"
}

resource "alicloud_vswitch""ai_vswitch"{
  vpc_id = alicloud_vpc.ai_vpc.id
  cidr_block = "172.16.0.0/24"
  zone_id = "cn-hangzhou-b"
}

resource "alicloud_security_group""ai_sg"{
  name = "ai-cluster-sg"
  vpc_id = alicloud_vpc.ai_vpc.id
}

# 节点间互通（NCCL需要）
resource "alicloud_security_group_rule""internal_all"{
  type = "ingress"
  ip_protocol = "all"
  port_range = "-1/-1"
  security_group_id = alicloud_security_group.ai_sg.id
  cidr_ip = "172.16.0.0/12"
}

# SSH访问
resource "alicloud_security_group_rule""ssh"{
  type = "ingress"
  ip_protocol = "tcp"
  port_range = "22/22"
  security_group_id = alicloud_security_group.ai_sg.id
  cidr_ip = "0.0.0.0/0"
}
```

#### compute.tf - 计算资源

```hcl
resource "alicloud_instance" "ai_node" {
  count = var.instance_count
  instance_name = "ai-node-${count.index}"
  image_id = "ubuntu_22_04_x64_20G_alibase_20231221.vhd"
  instance_type = var.instance_type
  vswitch_id = alicloud_vswitch.ai_vswitch.id
  security_groups = [alicloud_security_group.ai_sg.id]

  # 按量付费
  instance_charge_type = "PostPaid"
  # 系统盘
  system_disk_category = "cloud_essd"
  system_disk_size = 40
}

# 输出节点IP
output "node_ips" {
  value = alicloud_instance.ai_node[*].private_ip
}
```

### 4.3 实验输出

#### terraform init

```
Initializing the backend...
Initializing provider plugins...
- Finding aliyun/alicloud versions matching "~> 1.210.0"...
- Installing aliyun/alicloud v1.210.0...
- Installed aliyun/alicloud v1.210.0 (signed by a HashiCorp partner, key ID Chinese)

Terraform has been successfully initialized!
```

#### terraform plan（默认 ecs.c6.large）

```
Plan: 7 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + node_ips = [
      + (known after apply),
      + (known after apply),
    ]
```

#### terraform plan -var="instance_type=ecs.gn7i-c8g1.2xlarge"（GPU 实例）

```
Terraform will perform the following actions:

  # alicloud_instance.ai_node[0] will be created
  + resource "alicloud_instance" "ai_node" {
      + instance_name        = "ai-node-0"
      + instance_type        = "ecs.gn7i-c8g1.2xlarge"  # NVIDIA A10 GPU
      + instance_charge_type = "PostPaid"
      + system_disk_category = "cloud_essd"
      + system_disk_size     = 40
      ...
    }

  # alicloud_instance.ai_node[1] will be created
  + resource "alicloud_instance" "ai_node" {
      + instance_name        = "ai-node-1"
      + instance_type        = "ecs.gn7i-c8g1.2xlarge"
      ...
    }

  # alicloud_security_group.ai_sg will be created
  # alicloud_security_group_rule.internal_all will be created
  # alicloud_security_group_rule.ssh will be created
  # alicloud_vpc.ai_vpc will be created
  # alicloud_vswitch.ai_vswitch will be created

Plan: 7 to add, 0 to change, 0 to destroy.
```

**资源清单**:


| 资源                                          | 说明                |
| ------------------------------------------- | ----------------- |
| `alicloud_vpc.ai_vpc`                       | VPC 172.16.0.0/12 |
| `alicloud_vswitch.ai_vswitch`               | 子网 172.16.0.0/24  |
| `alicloud_security_group.ai_sg`             | 安全组               |
| `alicloud_security_group_rule.internal_all` | 内网全通（NCCL）        |
| `alicloud_security_group_rule.ssh`          | SSH 22 端口         |
| `alicloud_instance.ai_node[0]`              | GPU 节点 0          |
| `alicloud_instance.ai_node[1]`              | GPU 节点 1          |


---

## 五、踩坑记录

### 拼写错误（terraform validate 可检测）

```hcl
# ✗ 错误
source = "aliyun/aicloud"        # 少了 l → alicloud
instace_type = "ecs.c6.large"    # 少了 n → instance_type
system_disk_catagory = "..."     # 拼错 → system_disk_category

# ✓ 正确
source = "aliyun/alicloud"
instance_type = "ecs.c6.large"
system_disk_category = "cloud_essd"
```

### 阿里云 GPU 配额

- GPU 实例（gn 系列）默认配额为 0
- `terraform plan` 不受配额限制，可正常预览
- `terraform apply` 会因配额不足失败
- 需要在阿里云控制台申请 GPU 配额

### 环境变量认证

```bash
export ALICLOUD_ACCESS_KEY="your-access-key"
export ALICLOUD_SECRET_KEY="your-secret-key"
export ALICLOUD_REGION="cn-hangzhou"
```

---

## 六、AI Infra 视角

这套配置是 AI 训练集群的基础架构：

```
┌─────────────────────────────────────────┐
│              VPC (172.16.0.0/12)        │
│  ┌───────────────────────────────────┐  │
│  │      VSwitch (172.16.0.0/24)      │  │
│  │  ┌─────────┐    ┌─────────┐       │  │
│  │  │ ai-node-0│    │ ai-node-1│      │  │
│  │  │  A10 GPU │◄──►│  A10 GPU │      │  │
│  │  └─────────┘    └─────────┘       │  │
│  │         NCCL 全通信               │  │
│  └───────────────────────────────────┘  │
│              Security Group             │
│         (内网全通 + SSH 22)            │
└─────────────────────────────────────────┘
```

**扩展方向**：

- 添加 NAS/OSS 存储挂载
- 配置 RDMA 网络（高性能训练）
- 使用 Spot 实例降低成本
- 添加 Kubernetes 集群（容器化训练）

---

## 七、参考资源


| 资源              | 链接                                                                                                                                         |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Terraform 官方教程  | [https://developer.hashicorp.com/terraform/tutorials](https://developer.hashicorp.com/terraform/tutorials)                                 |
| 阿里云 Provider 文档 | [https://registry.terraform.io/providers/aliyun/alicloud/latest/docs](https://registry.terraform.io/providers/aliyun/alicloud/latest/docs) |
| Terraform 最佳实践  | [https://www.terraform-best-practices.com/](https://www.terraform-best-practices.com/)                                                     |


---

**学习完成**: 2026-04-22