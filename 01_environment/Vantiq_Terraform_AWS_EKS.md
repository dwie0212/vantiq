

# Terraform으로 VANTIQ 서버를 AWS EKS에 배포하기

## 개요

이 문서는 [fujitake/vantiq-related](https://github.com/fujitake/vantiq-related/blob/main/vantiq-cloud-infra-operations/terraform_aws/new/readme.md) 리포지토리의 Terraform 코드를 사용하여 AWS 위에 VANTIQ 인프라를 배포하는 과정을 단계별로 정리한 가이드입니다.

## 아키텍처 개요

### 배포되는 리소스

| 구분 | 리소스 | 설명 |
|------|--------|------|
| 네트워크 | VPC, Subnet(Public/Private x 3AZ), IGW, NAT GW | 10_network 단계에서 생성 |
| 컴퓨팅 | EKS 클러스터 + 6개 Managed Node Group | 20_main 단계에서 생성 |
| 데이터베이스 | RDS PostgreSQL | Keycloak(인증 서버) 전용 |
| 운영 | Bastion Host (EC2 + Elastic IP) | SSH 접속 및 클러스터 관리용 |
| 애드온 | EBS CSI Driver | EKS 스토리지 지원 |

### EKS Node Group 구성 (dev 환경)

| Node Group | 인스턴스 타입 | 레이블 | 용도 |
|------------|--------------|--------|------|
| VANTIQ | c5.xlarge | compute | VANTIQ 서버 |
| MongoDB | r5.xlarge | database | MongoDB (데이터 저장) |
| Keycloak | m5.large | shared | 인증 서버 |
| Grafana | r5.xlarge | influxdb | InfluxDB + Grafana 모니터링 |
| Metrics | m5.xlarge | compute | 메트릭 수집/처리 |
| Ai_assistant | c5.xlarge | orgCompute | AI 어시스턴트 |

> **참고**: VANTIQ 데이터는 RDS가 아닌 EKS 위의 MongoDB Pod에 저장됩니다. RDS PostgreSQL은 Keycloak 전용입니다.

### Terraform State 구조

```
10_network (VPC, Subnet 등)
    ↑
20_main (EKS, RDS, Bastion) — terraform_remote_state로 10_network의 output 참조
```

---

## 사전 준비

### 1. 필수 도구 설치 (macOS 기준)

```bash
# AWS CLI
brew install awscli

# Terraform (HashiCorp tap 사용)
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# kubectl (이미 설치되어 있다면 생략)
brew install kubectl
```

설치 확인:

```bash
aws --version
terraform --version
kubectl version --client
```

### 2. AWS Credential 설정

```bash
aws configure
# AWS Access Key ID: <YOUR_ACCESS_KEY>
# AWS Secret Access Key: <YOUR_SECRET_KEY>
# Default region name: ap-northeast-2
# Default output format: json
```

설정 확인:

```bash
aws sts get-caller-identity
```

### 3. S3 Bucket 생성 (tfstate 원격 저장용)

```bash
# S3 Bucket 생성
aws s3 mb s3://<YOUR-BUCKET-NAME> --region ap-northeast-2

# 버전 관리 활성화
aws s3api put-bucket-versioning \
    --bucket <YOUR-BUCKET-NAME> \
    --versioning-configuration Status=Enabled

# 확인
aws s3api get-bucket-versioning --bucket <YOUR-BUCKET-NAME>
```

### 4. SSH 키 페어 생성

Worker Node 접속용과 Bastion Host 접속용 두 세트가 필요합니다.

```bash
mkdir -p ~/vantiq-deploy && cd ~/vantiq-deploy

# Worker Node용
ssh-keygen -t rsa -b 4096 -f worker_key -N ""

# Bastion Host용
ssh-keygen -t rsa -b 4096 -f bastion_key -N ""
```

---

## 배포 절차

### 1. 리포지토리 클론

```bash
cd ~/vantiq-deploy

# 필요한 폴더만 sparse checkout
git clone --no-checkout --depth 1 https://github.com/fujitake/vantiq-related.git
cd vantiq-related
git sparse-checkout set vantiq-cloud-infra-operations/terraform_aws/new
git checkout
```

### 2. 환경 디렉토리 생성

```bash
cd vantiq-cloud-infra-operations/terraform_aws/new

# 템플릿 복사
cp -r env-template env-dev

# SSH 키 파일 복사
cp ~/vantiq-deploy/worker_key* env-dev/
cp ~/vantiq-deploy/bastion_key* env-dev/
```

### 3. constants.tf 수정

`env-dev/constants.tf`를 아래 내용으로 교체합니다.

```hcl
###
###  common config
###
locals {
  common_config = {
    cluster_name                   = "vantiq-dev"
    cluster_version                = "1.34"
    bastion_kubectl_version        = "1.34.0"
    env_name                       = "dev"
    region                         = "ap-northeast-2"
    worker_access_private_key      = "worker_key"
    worker_access_public_key_name  = "worker_key.pub"
    bastion_access_public_key_name = "bastion_key.pub"
    bastion_enabled                = true
    bastion_instance_type          = "t2.small"
    bastion_jdk_version            = "11"
  }
}

###
###  network config
###
locals {
  network_config = {
    vpc_cidr_block = "172.20.0.0/16"
    public_subnet_config = {
      "az-0" = {
        cidr_block        = "172.20.0.0/24"
        availability_zone = "ap-northeast-2a"
      }
      "az-1" = {
        cidr_block        = "172.20.1.0/24"
        availability_zone = "ap-northeast-2b"
      }
      "az-2" = {
        cidr_block        = "172.20.2.0/24"
        availability_zone = "ap-northeast-2c"
      }
    }
    private_subnet_config = {
      "az-0" = {
        cidr_block        = "172.20.10.0/24"
        availability_zone = "ap-northeast-2a"
      }
      "az-1" = {
        cidr_block        = "172.20.11.0/24"
        availability_zone = "ap-northeast-2b"
      }
      "az-2" = {
        cidr_block        = "172.20.12.0/24"
        availability_zone = "ap-northeast-2c"
      }
    }
  }
}

###
### rds config
###
locals {
  rds_config = {
    db_name                 = "keycloak"
    db_username             = "keycloak"
    db_password             = null
    db_instance_class       = "db.t3.micro"
    db_storage_size         = 20
    db_storage_type         = "gp2"
    postgres_engine_version = "17.5"
    keycloak_db_expose_port = 5432
  }
}

###
###  eks config (dev 환경)
###
locals {
  eks_config = {
    managed_node_group_config = {
      "VANTIQ" = {
        ami_type           = "AL2023_x86_64_STANDARD"
        kubernetes_version = "1.34"
        instance_types     = ["c5.xlarge"]
        disk_size          = 40
        scaling_config = {
          desired_size = 1
          max_size     = 3
          min_size     = 0
        }
        node_workload_label = "compute"
      },
      "MongoDB" = {
        ami_type           = "AL2023_x86_64_STANDARD"
        kubernetes_version = "1.34"
        instance_types     = ["r5.xlarge"]
        disk_size          = 40
        scaling_config = {
          desired_size = 1
          max_size     = 3
          min_size     = 0
        }
        node_workload_label = "database"
      },
      "Keycloak" = {
        ami_type           = "AL2023_x86_64_STANDARD"
        kubernetes_version = "1.34"
        instance_types     = ["m5.large"]
        disk_size          = 40
        scaling_config = {
          desired_size = 1
          max_size     = 3
          min_size     = 0
        }
        node_workload_label = "shared"
      },
      "Grafana" = {
        ami_type           = "AL2023_x86_64_STANDARD"
        kubernetes_version = "1.34"
        instance_types     = ["r5.xlarge"]
        disk_size          = 40
        scaling_config = {
          desired_size = 1
          max_size     = 3
          min_size     = 0
        }
        node_workload_label = "influxdb"
      },
      "Metrics" = {
        ami_type           = "AL2023_x86_64_STANDARD"
        kubernetes_version = "1.34"
        instance_types     = ["m5.xlarge"]
        disk_size          = 40
        scaling_config = {
          desired_size = 1
          max_size     = 3
          min_size     = 0
        }
        node_workload_label = "compute"
      },
      "Ai_assistant" = {
        ami_type           = "AL2023_x86_64_STANDARD"
        kubernetes_version = "1.34"
        instance_types     = ["c5.xlarge"]
        disk_size          = 40
        scaling_config = {
          desired_size = 1
          max_size     = 3
          min_size     = 0
        }
        node_workload_label = "orgCompute"
      },
    }
    single_az_node_list          = ["Grafana", "Metrics", "Ai_assistant"]
    sg_ids_allowed_ssh_to_worker = []
  }
  eks_addon_config = {
    ebs_csi_driver_enabled = true
  }
}

###
### tf remote backend
###
locals {
  tf_remote_backend = {
    bucket_name = "<YOUR-BUCKET-NAME>"
    key_prefix  = "tfstate/dev"
    region      = "ap-northeast-2"
  }
}
```

### 4. backend.tf 수정

**env-dev/10_network/backend.tf** — 아래 내용으로 교체:

```hcl
terraform {
  backend "s3" {
    bucket = "<YOUR-BUCKET-NAME>"
    key    = "tfstate/dev/network.tfstate"
    region = "ap-northeast-2"
  }
}
```

**env-dev/20_main/backend.tf** — 아래 내용으로 교체:

```hcl
terraform {
  backend "s3" {
    bucket = "<YOUR-BUCKET-NAME>"
    key    = "tfstate/dev/main.tfstate"
    region = "ap-northeast-2"
  }
}

data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "<YOUR-BUCKET-NAME>"
    key    = "tfstate/dev/network.tfstate"
    region = "ap-northeast-2"
  }
}
```

### 5. Apple Silicon(darwin_arm64) 호환 수정

`hashicorp/template` 프로바이더가 Apple Silicon을 지원하지 않으므로, `templatefile()` 함수로 대체해야 합니다.

**modules/opnode/bastion-instance.tf** 수정:

`data "template_file" "bastion"` 블록 전체를 삭제하고, `aws_instance`의 `user_data`를 아래로 변경:

```hcl
user_data = templatefile("${path.module}/bastion-userdata.sh.tpl", {
  jdk_version               = var.bastion_jdk_version
  bastion_kubectl_version   = var.bastion_kubectl_version
  worker_access_private_key = file(var.worker_access_private_key)
})
```

같은 파일에서 `aws_eip`의 deprecated 경고도 수정:

```hcl
# 기존
vpc = true

# 변경
domain = "vpc"
```

**modules/opnode/output.tf** 수정:

`bastion_userdata` output을 아래로 변경:

```hcl
output "bastion_userdata" {
  value = templatefile("${path.module}/bastion-userdata.sh.tpl", {
    jdk_version               = var.bastion_jdk_version
    bastion_kubectl_version   = var.bastion_kubectl_version
    worker_access_private_key = file(var.worker_access_private_key)
  })
  description = "bastion instance user data"
}
```

### 6. 네트워크 배포 (10_network)

```bash
cd env-dev/10_network

terraform init
terraform plan
# Plan: 32 to add 확인

terraform apply
# yes 입력 → 2~3분 소요
```

### 7. 메인 인프라 배포 (20_main)

```bash
cd ../20_main

terraform init
terraform plan
# Plan: 28 to add 확인

terraform apply
# yes 입력 → 15~20분 소요 (EKS 클러스터 생성)
```

## Apple Silicon(darwin_arm64) 및 EKS 1.33+ 호환 수정

### EKS 모듈 SSM 경로 수정

EKS 1.33 이상부터 Amazon Linux 2 AMI가 제공되지 않습니다. `ami_type`을 `AL2023`으로 사용하는 경우 SSM Parameter 경로도 변경해야 합니다.

**`modules/eks/eks.tf`** 수정:

```hcl
# 기존 (amazon-linux-2 — EKS 1.32 이하)
data "aws_ssm_parameter" "eks_ami_release_version" {
  for_each = var.managed_node_group_config
  name     = "/aws/service/eks/optimized-ami/${each.value.kubernetes_version}/amazon-linux-2/recommended/release_version"
}

# 변경 (amazon-linux-2023 — EKS 1.33 이상)
data "aws_ssm_parameter" "eks_ami_release_version" {
  for_each = var.managed_node_group_config
  name     = "/aws/service/eks/optimized-ami/${each.value.kubernetes_version}/amazon-linux-2023/x86_64/standard/recommended/release_version"
}
```

> **참고**: 이 수정 없이 EKS 1.33 이상 + `AL2023_x86_64_STANDARD` 조합을 사용하면 `couldn't find resource` 에러가 발생합니다.

### 8. EKS 접속 설정

```bash
# kubeconfig 설정
aws eks update-kubeconfig --region ap-northeast-2 --name vantiq-dev-dev

# 노드 확인 (6개 노드가 Ready 상태여야 함)
kubectl get nodes
```

---

## 배포 결과 확인

```bash
# 전체 output 확인
terraform output

# RDS 패스워드 확인 (sensitive)
terraform output postgres_admin_password
```

### 주요 Output 항목

| 항목 | 설명 |
|------|------|
| `bastion_public_ip` | Bastion Host 접속 IP |
| `cluster_eks_name` | EKS 클러스터 이름 |
| `kube_config_update_command` | kubeconfig 설정 명령어 |
| `postgres_endpoint` | RDS PostgreSQL 엔드포인트 |
| `postgres_admin_user` | RDS 관리자 사용자명 |
| `postgres_admin_password` | RDS 관리자 패스워드 (sensitive) |

---

## 삭제 절차

배포의 역순으로 진행합니다.

```bash
# 1. 메인 인프라 삭제
cd env-dev/20_main
terraform destroy

# 2. 네트워크 삭제
cd ../10_network
terraform destroy

# 3. S3 Bucket 삭제 (필요시)
aws s3 rb s3://<YOUR-BUCKET-NAME> --force
```

---

## 주의 사항

- **AZ 확인**: 리전에 따라 사용 가능한 Availability Zone이 다릅니다. `aws ec2 describe-availability-zones --region <region>`으로 확인하세요.
- **vCPU 한도**: c5, r5, m5 등 다수의 인스턴스를 사용하므로, AWS 콘솔 > Service Quotas에서 "Running On-Demand Standard instances"의 vCPU 한도를 확인하세요.
- **비용**: dev 환경(노드 6개, desired_size=1)에서도 월 수백 달러가 발생합니다. 사용하지 않을 때는 `terraform destroy`로 정리하세요.
- **다음 단계**: 이 문서는 AWS 인프라 배포까지만 다룹니다. VANTIQ 애플리케이션 배포(Helm)는 별도 문서를 참고하세요.

---

## 참고

- [원본 가이드 (fujitake/vantiq-related)](https://github.com/fujitake/vantiq-related/blob/main/vantiq-cloud-infra-operations/terraform_aws/new/readme.md)
- [VANTIQ 운영 가이드](https://github.com/fujitake/vantiq-related/tree/main/vantiq-cloud-infra-operations)
