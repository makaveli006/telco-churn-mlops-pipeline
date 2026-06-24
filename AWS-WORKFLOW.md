# AWS Production Deployment Workflow

End-to-end steps to deploy the Telco Churn FastAPI app on AWS ECS Fargate with an Application Load Balancer.

## Architecture

```
GitHub push to main
        ↓
GitHub Actions (ci.yml)
        ↓
Docker image built + pushed to makaveli006/telco-fastapi:latest on Docker Hub
        ↓
ECS pulls image from Docker Hub
        ↓
Fargate runs the container (port 8000)
        ↓
ALB routes internet traffic (port 80) → Fargate task (port 8000)
        ↓
telco-alb-172744476.us-east-1.elb.amazonaws.com
```

## AWS Resources Summary

| Resource | Name | ID / ARN |
|----------|------|----------|
| Region | US East (N. Virginia) | `us-east-1` |
| VPC | Default VPC | `vpc-e000b887` |
| ECS Cluster | `telco-churn` | `arn:aws:ecs:us-east-1:572132468634:cluster/telco-churn` |
| IAM Role | `ecsTaskExecutionRole` | `arn:aws:iam::572132468634:role/ecsTaskExecutionRole` |
| Task Definition | `telco-churn-task:1` | — |
| ALB SG | `telco-alb-sg` | `sg-0e2a2f2cb432ff5c5` |
| Task SG | `telco-task-sg` | `sg-0e642b85221e6e247` |
| Target Group | `telco-tg` | port 8000, health check `/` |
| Load Balancer | `telco-alb` | `telco-alb-172744476.us-east-1.elb.amazonaws.com` |
| ECS Service | `telco-churn-service` | desired count: 1 |

---

## Phase 1 — Prerequisites

All of these were already in place before starting AWS setup:

- Docker Hub repo: `makaveli006/telco-fastapi` exists
- GitHub secrets set: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`
- AWS CLI configured locally
- CI/CD pipeline (`.github/workflows/ci.yml`) builds and pushes image automatically on every push to `main` — no manual `docker push` needed

---

## Phase 2 — AWS Setup

### Step 1 — Create ECS Cluster

1. Go to **ECS → Clusters → Create Cluster**
2. **Cluster name**: `telco-churn`
3. **Infrastructure**: `Fargate only` (serverless)
4. Click **Create**

> **Gotcha:** If you see `CREATE_FAILED` in CloudFormation with error *"Unable to assume the service linked role"*, run:
> ```bash
> aws iam create-service-linked-role --aws-service-name ecs.amazonaws.com
> ```
> If it says the role already exists, that's fine — go to CloudFormation, delete the failed stack (`Infra-ECS-Cluster-telco-churn-*`), then try creating the cluster again.

---

### Step 2 — Create IAM Task Execution Role

This role allows Fargate to pull the Docker image and write logs to CloudWatch.

1. Go to **IAM → Roles → Create role**
2. **Trusted entity type**: `AWS service`
3. **Service**: `Elastic Container Service`
4. **Use case**: `Task Execution Role for Elastic Container Service`
   - ⚠️ Do NOT select "Elastic Container Service" or "Elastic Container Service Task" — those attach the wrong policy
5. Click **Next** — you will see `AmazonECSTaskExecutionRolePolicy` already attached
6. **Role name**: `ecsTaskExecutionRole`
7. Click **Create role**

---

### Step 3 — Create Task Definition

1. Go to **ECS → Task definitions → Create new task definition**
2. **Task definition family**: `telco-churn-task`
3. **Infrastructure**: `AWS Fargate`
4. **CPU**: `1 vCPU`
5. **Memory**: `3 GB`
6. **Task execution role**: `ecsTaskExecutionRole`
7. Under **Container**:
   - **Container name**: `telco-churn`
   - **Image URI**: `makaveli006/telco-fastapi:latest`
   - **Container port**: `8000`
   - **Protocol**: `TCP`
8. Leave everything else default → Click **Create**

---

### Step 4 — Create Security Groups

You need 2 security groups — one for the ALB and one for the Fargate task.

#### SG 1 — ALB Security Group

1. Go to **EC2 → Security Groups → Create security group**
2. **Name**: `telco-alb-sg`
3. **VPC**: `vpc-e000b887` (default VPC)
4. **Inbound rules**:
   - Type: `HTTP` | Port: `80` | Source: `Anywhere-IPv4`
5. **Outbound rules**: leave default (all traffic)
6. Click **Create security group**

#### SG 2 — Fargate Task Security Group

1. Go to **EC2 → Security Groups → Create security group**
2. **Name**: `telco-task-sg`
3. **VPC**: `vpc-e000b887` (default VPC)
4. **Inbound rules**:
   - Type: `Custom TCP` | Port: `8000` | Source: `Custom` → paste `sg-0e2a2f2cb432ff5c5` (telco-alb-sg ID)
   - This ensures only the ALB can reach the container
5. **Outbound rules**: leave default
6. Click **Create security group**

---

## Phase 3 — Load Balancer

### Step 1 — Create Target Group

The target group is what the ALB forwards traffic to (your Fargate tasks).

1. Go to **EC2 → Target Groups → Create target group**
2. **Target type**: `IP addresses`
3. **Target group name**: `telco-tg`
4. **Protocol**: `HTTP` | **Port**: `8000`
5. **VPC**: `vpc-e000b887` (default VPC)
6. **Health check path**: `/`
7. Click **Next** → do NOT register any targets manually → Click **Create target group**

---

### Step 2 — Create Application Load Balancer

1. Go to **EC2 → Load Balancers → Create load balancer**
2. Choose **Application Load Balancer**
3. **Name**: `telco-alb`
4. **Scheme**: `Internet-facing`
5. **IP address type**: `IPv4`
6. **VPC**: `vpc-e000b887` (default VPC)
7. **Subnets**: select at least 2 AZs — tick `us-east-1a` (`subnet-934f5ae5`) and `us-east-1b` (`subnet-fbfcdba3`)
8. **Security groups**: remove the default SG → add `telco-alb-sg`
9. **Listeners**: HTTP:80 → forward to `telco-tg`
10. Click **Create load balancer**

> ALB takes 2-3 minutes to become active.

---

## Phase 4 — ECS Service

The service keeps your container running and connects it to the ALB.

1. Go to **ECS → Clusters → telco-churn → Services → Create**
2. **Launch type**: `Fargate`
3. **Task definition**: `telco-churn-task` → revision `1`
4. **Service name**: `telco-churn-service`
5. **Desired tasks**: `1`

### Networking:
6. **VPC**: `vpc-e000b887`
7. **Subnets**: select `us-east-1a` and `us-east-1b`
8. **Security group**: remove default → add `telco-task-sg`
9. **Public IP**: `Enabled` (required so Fargate can pull image from Docker Hub)

### Load balancing:
10. **Load balancer type**: `Application Load Balancer`
11. **Load balancer**: `Use an existing load balancer` → select `telco-alb`
12. **Listener**: `Use an existing listener` → select `80:HTTP`
    - ⚠️ Do NOT use "Create new listener" — port 80 already exists on the ALB
13. **Target group**: `Use an existing target group` → select `telco-tg`
    - ⚠️ If `telco-tg` appears greyed out saying "in use by a different load balancer", select "Use an existing listener" first — that unlocks the existing target group
14. Click **Create**

> Fargate pulls the Docker image from Docker Hub — takes 2-3 minutes for Running count to go from 0 to 1.

---

## Phase 5 — Verify

### Health check
```bash
curl http://telco-alb-172744476.us-east-1.elb.amazonaws.com/
# Expected: {"status":"ok"}
```

### Prediction
```bash
curl -X POST http://telco-alb-172744476.us-east-1.elb.amazonaws.com/predict -H "Content-Type: application/json" -d "{\"gender\":\"Female\",\"Partner\":\"No\",\"Dependents\":\"No\",\"PhoneService\":\"Yes\",\"MultipleLines\":\"No\",\"InternetService\":\"Fiber optic\",\"OnlineSecurity\":\"No\",\"OnlineBackup\":\"No\",\"DeviceProtection\":\"No\",\"TechSupport\":\"No\",\"StreamingTV\":\"Yes\",\"StreamingMovies\":\"Yes\",\"Contract\":\"Month-to-month\",\"PaperlessBilling\":\"Yes\",\"PaymentMethod\":\"Electronic check\",\"tenure\":1,\"MonthlyCharges\":85.0,\"TotalCharges\":85.0}"
# Expected: {"prediction":"Likely to churn"}
```

### Gradio UI
```
http://telco-alb-172744476.us-east-1.elb.amazonaws.com/ui
```

---

## Networking Notes

- **Why default VPC**: Has 6 public subnets across 6 AZs — ALB requires at least 2 AZs. The custom VPC (`vpc-b8b8e4df`) only had 1 private subnet, not enough for ALB.
- **Why Public IP enabled on Fargate**: The task needs to pull the Docker image from Docker Hub on the public internet. Private subnets would need a NAT Gateway ($32+/month) to do this.
- **Security group flow**: Internet → ALB SG (port 80) → ALB → Task SG (port 8000 from ALB SG only) → Fargate container

---

## Cost Notes

| Resource | Cost |
|----------|------|
| ECS Cluster | Free |
| IAM Role | Free |
| Task Definition | Free |
| Security Groups | Free |
| ALB | ~$16/month minimum |
| Fargate (1 vCPU, 3GB, 1 task) | ~$0.04/hour ≈ ~$29/month |
| Target Group | Free |

**To stop costs**: delete the ECS service (sets running tasks to 0) and delete the ALB.
