# Fully Sequential Deployment Guide: Frontend + Backend on AWS ECS (Fargate)

## ✅ Plan

1. Create VPC, subnets, route tables, and NAT
2. Create security groups
3. Create ECR repositories and push images
4. Create Cloud Map namespace
5. Create ECS task definitions
6. Create ALB for frontend
7. Create ECS cluster
8. Create backend ECS service (with Cloud Map)
9. Create frontend ECS service (with ALB)
10. Configure ECS Auto Scaling

---

## 🔧 1. VPC and Networking

### 1.1. Create VPC
- Name: `MyApp-VPC`
- IPv4 CIDR block: `10.0.0.0/16`
- Tenancy: Default

### 1.2. Subnets
Create 4 subnets:

| Name              | AZ         | CIDR        |
|-------------------|------------|-------------|
| PublicSubnet1     | us-east-1a | 10.0.1.0/24 |
| PublicSubnet2     | us-east-1b | 10.0.2.0/24 |
| PrivateSubnet1    | us-east-1a | 10.0.3.0/24 |
| PrivateSubnet2    | us-east-1b | 10.0.4.0/24 |

### 1.3. Internet Gateway
- Create and attach to VPC

### 1.4. Route Tables
#### Public Route Table:
- Associate with `PublicSubnet1`, `PublicSubnet2`
- Add route: `0.0.0.0/0` → Internet Gateway

#### Private Route Table:
- Associate with `PrivateSubnet1`, `PrivateSubnet2`
- Add NAT route later

### 1.5. NAT Gateway
- Create in `PublicSubnet1` with Elastic IP
- Add `0.0.0.0/0` route in Private Route Table → NAT Gateway

---

## 🔐 2. Security Groups

### 2.1. frontend-alb-sg
- Inbound: HTTP (port 80) from 0.0.0.0/0

### 2.2. frontend-tasks-sg
- Inbound: HTTP (port 80) from `frontend-alb-sg`

### 2.3. backend-tasks-sg
- Inbound: TCP (port 8080) from `frontend-tasks-sg`

---

## 📦 3. ECR Repositories and Image Push

Create:
- `frontend-repo`
- `backend-repo`

Push Docker images using `docker tag` and `docker push`.

---

## 🌐 4. Cloud Map Namespace

- Name: `backend.local`
- Type: Private DNS
- VPC: `MyApp-VPC`
- TTL: `60`
- Discovery type: **API calls and DNS queries in VPCs**

---

## ⚙️ 5. Task Definitions

### 5.1. backend-task (with environment variables)

- Launch type: Fargate
- CPU: 512, Memory: 1024
- Container:
  - Name: `backend`
  - Image: `.../backend-repo:latest`
  - Port mapping: 8080
  - Environment variables:
    - `DB_URL` = `...`
    - `JWT_SECRET` = `...`
- Network mode: awsvpc

### 5.2. frontend-task

- Image: `.../frontend-repo:latest`
- Port: 80
- nginx.conf contains:
```nginx
proxy_pass http://backend.local:8080/account/;
```

---

## 🖥 6. Application Load Balancer (ALB)

- Name: `frontend-alb`
- Scheme: Internet-facing
- Subnets: `PublicSubnet1`, `PublicSubnet2`
- Security Group: `frontend-alb-sg`

### Target Group:
- Name: `frontend-tg`
- Target type: IP
- Port: 80

---

## 🧱 7. ECS Cluster

- Name: `MyApp-Cluster`
- Launch type: Fargate

---

## 🧩 8. Backend ECS Service

- Task definition: `backend-task`
- Subnets: Private
- Security Group: `backend-tasks-sg`
- Public IP: DISABLED
- ✅ Enable service discovery:
  - Namespace: `backend.local`
  - Service name: `backend`

---

## 🧩 9. Frontend ECS Service

- Task definition: `frontend-task`
- Subnets: Public
- Security Group: `frontend-tasks-sg`
- Public IP: ENABLED
- Load Balancer:
  - ALB: `frontend-alb`
  - Target Group: `frontend-tg`

---

## 📈 10. Auto Scaling

- Desired tasks: 2
- Minimum: 2, Maximum: 6
- Policy: Target tracking
  - Metric: CPU utilization
  - Target: 70%

---

## 🔐 Secure Environment Variable Injection Options

### Option A — AWS Secrets Manager

1. Create a secret (e.g., `/myapp/backend/creds`)
2. Reference secret in ECS task definition (via `ValueFrom`)
3. Attach `SecretsManagerReadWrite` policy to ECS task role

### Option B — AWS Systems Manager Parameter Store

1. Create SecureString parameter (e.g., `/myapp/backend/JWT_SECRET`)
2. Reference in ECS task definition with `ValueFrom`
3. Attach `AmazonSSMReadOnlyAccess` policy to ECS task role

---
