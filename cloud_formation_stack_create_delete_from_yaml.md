# CloudFormation Stack — Create & Delete (from YAML)

This guide shows how to **create, verify, update, and delete** a CloudFormation stack from a local YAML template. It includes both **AWS Console** and **AWS CLI** paths.

---

## 0) Prerequisites
- **Template file**: `ecs-fargate-stack.yml` (your latest YAML with frontend + backend on ECS Fargate).
- **AWS account/region**: examples use `us-east-1`.
- **Permissions**: ability to create IAM roles/policies (you must acknowledge IAM capability).
- **Images & parameters ready**:
  - ECR images (e.g., `982081087075.dkr.ecr.us-east-1.amazonaws.com/accounting:latest`, `.../forum:latest`).
  - SSM Parameter Store keys: `/backend/forum/MONGODB_USER`, `/backend/forum/MONGODB_BASE`, `/backend/forum/MONGODB_PASSWORD`.

> Tip: If your parameters differ, pass them at creation time (Console or CLI `--parameters ParameterKey=...,ParameterValue=...`).

---

## 1) Create Stack — AWS Console
1. Open **CloudFormation** → **Create stack** → **With new resources (standard)**.
2. **Specify template** → **Upload a template file** → select `ecs-fargate-stack.yml` → **Next**.
3. **Stack name**: `ECS-Fargate-Forum` (or any unique name).
4. **Parameters** (examples):
   - `FrontendImage` = `982081087075.dkr.ecr.us-east-1.amazonaws.com/accounting:latest`
   - `BackendImage`  = `982081087075.dkr.ecr.us-east-1.amazonaws.com/forum:latest`
   - `MongodbUserParamName`   = `/backend/forum/MONGODB_USER`
   - `MongodbBaseParamName`   = `/backend/forum/MONGODB_BASE`
   - `MongodbPasswordParamName` = `/backend/forum/MONGODB_PASSWORD`
   - Leave VPC/Subnet CIDRs as defaults unless you need custom ranges.
5. **Configure stack options**: keep defaults (add tags if desired).
6. **Review**: at the bottom tick the checkbox **“I acknowledge that AWS CloudFormation might create IAM resources.”**
7. Click **Create stack**.
8. Watch **Events** until the status becomes **CREATE_COMPLETE**.

### 1.1 Verify after create
- **Outputs** tab → copy `AlbDNSName` and open `http://<AlbDNSName>`.
- **ECS → Clusters → MyApp-Cluster → Services**: both `frontend-service` and `backend-service` should be **ACTIVE** with desired tasks running.
- **EC2 → Target Groups → <frontend TG> → Targets**: health should be **healthy (200–399)** on `HealthCheckPath`.
- **Cloud Map**: namespace (e.g., `forum.local`) includes a service `backend` with A-records.
- **CloudWatch Logs**: `/ecs/myapp/<stackName>` streams for `frontend` and `backend` show normal startup.

---

## 2) Create Stack — AWS CLI
Set variables and run:

```bash
export AWS_REGION=us-east-1
export STACK_NAME=ECS-Fargate-Forum
export TEMPLATE_FILE=ecs-fargate-stack.yml

aws cloudformation create-stack \
  --region "$AWS_REGION" \
  --stack-name "$STACK_NAME" \
  --template-body file://$TEMPLATE_FILE \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=FrontendImage,ParameterValue=982081087075.dkr.ecr.us-east-1.amazonaws.com/accounting:latest \
    ParameterKey=BackendImage,ParameterValue=982081087075.dkr.ecr.us-east-1.amazonaws.com/forum:latest \
    ParameterKey=MongodbUserParamName,ParameterValue=/backend/forum/MONGODB_USER \
    ParameterKey=MongodbBaseParamName,ParameterValue=/backend/forum/MONGODB_BASE \
    ParameterKey=MongodbPasswordParamName,ParameterValue=/backend/forum/MONGODB_PASSWORD

aws cloudformation wait stack-create-complete \
  --region "$AWS_REGION" \
  --stack-name "$STACK_NAME"

aws cloudformation describe-stacks \
  --region "$AWS_REGION" \
  --stack-name "$STACK_NAME" \
  --query 'Stacks[0].Outputs'
```

> If you need to override network CIDRs or counts, add extra `ParameterKey=...,ParameterValue=...` pairs.

---

## 3) Update Stack
Use **Update** when you change the template or parameters (e.g., new image tags, desired counts, health-check path).

### 3.1 Console
- **Stacks → your stack → Update**
  - **Use current template** (edit inline) or **Replace current template** (upload a new file).
  - Adjust parameters (e.g., `FrontendImage`, `BackendImage`, or `BackendDesiredCount`).
  - **Next → Update stack** and watch **Events**.

### 3.2 CLI
```bash
aws cloudformation update-stack \
  --region "$AWS_REGION" \
  --stack-name "$STACK_NAME" \
  --template-body file://$TEMPLATE_FILE \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=FrontendImage,ParameterValue=982081087075.dkr.ecr.us-east-1.amazonaws.com/accounting:latest \
    ParameterKey=BackendImage,ParameterValue=982081087075.dkr.ecr.us-east-1.amazonaws.com/forum:latest

aws cloudformation wait stack-update-complete \
  --region "$AWS_REGION" \
  --stack-name "$STACK_NAME"
```

> Rolling updates will create new Task Definition revisions and replace tasks safely.

---

## 4) Delete Stack (Cleanup)
### 4.1 Console
1. **Stacks → select your stack → Delete**.
2. Confirm and wait for **DELETE_COMPLETE**.
3. If deletion fails due to remaining dependencies (e.g., EIP, log groups, or manually-created resources), remove those in their services and retry delete.

### 4.2 CLI
```bash
aws cloudformation delete-stack \
  --region "$AWS_REGION" \
  --stack-name "$STACK_NAME"

aws cloudformation wait stack-delete-complete \
  --region "$AWS_REGION" \
  --stack-name "$STACK_NAME"
```

---

## 5) Troubleshooting Quick Reference
- **AlreadyExists (names)**: remove hard-coded `Name` fields (ALB/TG/RoleName) or delete conflicting resources.
- **CREATE_IN_PROGRESS for a long time**: usually ECS services failing to stabilize.
  - **Container health check fails** → fix/remove health check or correct endpoint (temporarily set `BackendDesiredCount=0` to finish stack create, then debug tasks).
  - **CannotPullContainerError** → verify ECR tags exist and NAT GW/route is correct.
  - **SSM AccessDenied/KMS** → ensure ExecutionRole has `AmazonSSMReadOnlyAccess` and KMS `Decrypt` if SecureString uses a custom CMK.
- **Target group unhealthy**: make sure `HealthCheckPath` returns HTTP 200–399.

---

## 6) Useful One-liners
- **Fetch ALB DNS from Outputs (CLI):**
```bash
aws cloudformation describe-stacks \
  --region "$AWS_REGION" \
  --stack-name "$STACK_NAME" \
  --query "Stacks[0].Outputs[?OutputKey=='AlbDNSName'].OutputValue" \
  --output text
```
- **See backend-service events quickly:**
```bash
aws ecs describe-services \
  --region "$AWS_REGION" \
  --cluster MyApp-Cluster \
  --services backend-service \
  --query 'services[0].events[0:10].[createdAt,message]' \
  --output table
```

---

### Notes
- When experimenting, prefer **unique stack names** or remove explicit resource names to avoid collisions.
- After initial stabilization, consider re-adding container health checks to improve resilience.
- For HTTPS, add ACM cert + HTTPS listener and HTTP→HTTPS redirect in the template.

