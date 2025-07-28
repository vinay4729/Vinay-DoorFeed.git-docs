# 🚀 CI/CD Pipeline Assessment Project – DoorFeed (Student Guide)

---

## ✅ PHASE 0: PROJECT OVERVIEW

---

### 🟩 **Problem Statement**

> DoorFeed is building a web application hosted on GitHub and wants to implement a fully automated CI/CD pipeline that deploys the app across **Development**, **Staging**, and **Production** environments using **Infrastructure as Code** on AWS.

---

### 🧩 **Real-World Use Case**

A DevOps Engineer at DoorFeed must ensure that every commit is automatically tested, packaged, deployed, and monitored without manual steps. The solution must ensure fast feedback, secure deployment, and environment segregation.

---

### 🎯 **Objectives**

* Design a complete CI/CD pipeline using GitHub → AWS
* Support three environments: dev, staging, prod
* Use Terraform for infrastructure provisioning
* Build and push Docker images to Amazon ECR
* Deploy containers to ECS Fargate
* Implement secrets management, logging, and monitoring
* Add security controls (IAM roles, secrets, approvals)

---

### 🧰 **Tech Stack Used**

| Purpose                | Tool/Service                            |
| ---------------------- | --------------------------------------- |
| Version Control        | GitHub                                  |
| CI/CD                  | GitHub Actions                          |
| Infrastructure as Code | Terraform                               |
| Containerization       | Docker                                  |
| Image Registry         | Amazon Elastic Container Registry (ECR) |
| Deployment Service     | Amazon ECS (Fargate)                    |
| Secrets Management     | AWS Secrets Manager                     |
| Monitoring & Alerting  | Amazon CloudWatch + SNS                 |
| Testing Frameworks     | Jest, Postman (optional)                |

---

### 📁 **Project Folder Structure**

```
doorfeed-cicd-project/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── terraform/
│   ├── main.tf
│   ├── dev.tfvars
│   ├── staging.tfvars
│   └── prod.tfvars
├── Dockerfile
├── README.md
└── test/
    ├── unit/
    └── integration/
```

---

# ✅ PHASE 1: ENVIRONMENT SETUP (AWS + LOCAL + GITHUB)

This phase will prepare all the necessary infrastructure, access, and tools to execute the CI/CD pipeline securely and effectively.

---

## 🎯 Goal

> Set up all the base services, permissions, and configurations needed before implementing the CI/CD pipeline.

---

## 🧱 Step-by-Step Implementation

---

### 1️⃣ 🧰 Install Required Tools Locally

Install the following on your **local machine** or **Cloud9 EC2 instance**:

| Tool      | Version | Install Guide                                                                                                                                 |
| --------- | ------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| Terraform | ≥ 1.5   | 🔗 [https://developer.hashicorp.com/terraform/downloads](https://developer.hashicorp.com/terraform/downloads)                                 |
| AWS CLI   | ≥ v2.0  | 🔗 [https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) |
| Docker    | Latest  | 🔗 [https://docs.docker.com/get-docker/](https://docs.docker.com/get-docker/)                                                                 |
| Node.js   | 20.x    | 🔗 [https://nodejs.org/en/download](https://nodejs.org/en/download)                                                                           |

---

### 2️⃣ 🔐 Create AWS IAM User for GitHub Actions

This user will allow GitHub Actions to push Docker images and run Terraform.

🛠️ **Steps:**

1. Go to AWS Console → IAM → Users → `Add user`
2. Name: `doorfeed-cicd-user`
3. Access type: ✅ Programmatic access
4. Permissions:

   * Attach existing policies:

     * `AmazonEC2ContainerRegistryFullAccess`
     * `AmazonECS_FullAccess`
     * `AmazonS3FullAccess`
     * `CloudWatchFullAccess`
     * `SecretsManagerReadWrite`
     * `IAMFullAccess`
5. Create user → Download the **Access Key** and **Secret Key**

---

### 3️⃣ 🔐 Add GitHub Secrets for Pipeline Authentication

Navigate to your GitHub repository → Settings → **Secrets and variables** → **Actions** → Add the following secrets:

| Secret Name             | Description               |
| ----------------------- | ------------------------- |
| `AWS_ACCESS_KEY_ID`     | From IAM User             |
| `AWS_SECRET_ACCESS_KEY` | From IAM User             |
| `AWS_REGION`            | e.g., `us-east-1`         |
| `ECR_REPO_URL`          | Your created ECR repo URI |

---

### 4️⃣ 🐳 Create ECR Repository for Docker Images

🛠️ **Command:**

```bash
aws ecr create-repository \
  --repository-name doorfeed-app \
  --region us-east-1
```

✅ **Output:**
You’ll get the `repositoryUri`, e.g.,

```
123456789012.dkr.ecr.us-east-1.amazonaws.com/doorfeed-app
```

Use this as `ECR_REPO_URL` in GitHub secrets.

---

### 5️⃣ 🪣 Setup Terraform Backend (S3 Bucket + DynamoDB)

This allows **remote state management** for Terraform.

🛠️ **Commands:**

```bash
aws s3api create-bucket --bucket doorfeed-terraform-backend --region us-east-1

aws dynamodb create-table \
  --table-name doorfeed-terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1
```

Then use this in your Terraform backend:

📁 `terraform/backend.tf`

```hcl
terraform {
  backend "s3" {
    bucket         = "doorfeed-terraform-backend"
    key            = "doorfeed/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "doorfeed-terraform-locks"
  }
}
```

---

### 6️⃣ 🧪 Test Terraform Setup

📁 Inside `terraform/` folder:

```bash
terraform init
```

✅ **Expected Output:**

```
Successfully configured the backend "s3"!
Terraform has been successfully initialized!
```

✅ Environment setup complete!

---

# ✅ PHASE 2: INFRASTRUCTURE AS CODE (TERRAFORM + ECS + ECR + NETWORKING)

---

## 🎯 Goal

> Provision all required AWS infrastructure using **Terraform** for three environments: `dev`, `staging`, and `prod`.

---

## 🧱 Infrastructure Components

We'll define and deploy the following AWS resources using **Terraform**:

| Component             | Purpose                              |
| --------------------- | ------------------------------------ |
| VPC, Subnets, IGW     | Network isolation                    |
| ECS Cluster (Fargate) | Container orchestration (serverless) |
| ECR                   | Docker image registry                |
| Load Balancer (ALB)   | External access to app               |
| ECS Task Definition   | Define container + CPU/memory        |
| ECS Service           | Deploy container using Fargate       |
| Security Groups       | Allow HTTP (port 80) traffic         |

---

## 📁 Recommended Terraform Folder Structure

```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── dev.tfvars
├── staging.tfvars
├── prod.tfvars
├── backend.tf
```

---

## 📜 Step-by-Step Terraform Implementation

---

### 1️⃣ Define Provider and Backend – `main.tf`

```hcl
provider "aws" {
  region = var.aws_region
}

module "network" {
  source = "terraform-aws-modules/vpc/aws"
  name = "${var.env}-vpc"
  cidr = "10.0.0.0/16"
  azs = ["us-east-1a", "us-east-1b"]
  public_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  enable_dns_hostnames = true
}

resource "aws_ecs_cluster" "main" {
  name = "${var.env}-cluster"
}

resource "aws_ecs_task_definition" "app" {
  family                   = "${var.env}-task"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"
  network_mode            = "awsvpc"
  execution_role_arn      = var.execution_role_arn
  container_definitions   = jsonencode([
    {
      name      = "doorfeed-app"
      image     = "${var.ecr_url}:${var.image_tag}"
      essential = true
      portMappings = [
        {
          containerPort = 80
          hostPort      = 80
        }
      ]
    }
  ])
}

resource "aws_ecs_service" "app" {
  name            = "${var.env}-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 1
  launch_type     = "FARGATE"
  network_configuration {
    subnets         = module.network.public_subnets
    assign_public_ip = true
    security_groups = [aws_security_group.ecs_sg.id]
  }
  load_balancer {
    target_group_arn = aws_lb_target_group.app_tg.arn
    container_name   = "doorfeed-app"
    container_port   = 80
  }
  depends_on = [aws_lb_listener.http]
}
```

---

### 2️⃣ Define Variables – `variables.tf`

```hcl
variable "aws_region" {}
variable "env" {}
variable "ecr_url" {}
variable "image_tag" {}
variable "execution_role_arn" {}
```

---

### 3️⃣ Define Environments – `dev.tfvars`, `staging.tfvars`, `prod.tfvars`

```hcl
aws_region        = "us-east-1"
env               = "dev"
ecr_url           = "123456789012.dkr.ecr.us-east-1.amazonaws.com/doorfeed-app"
image_tag         = "latest"
execution_role_arn = "arn:aws:iam::123456789012:role/ecsTaskExecutionRole"
```

Update values for each environment accordingly.

---

### 4️⃣ Initialize Terraform

```bash
cd terraform
terraform init
```

---

### 5️⃣ Create Workspace for Each Environment

```bash
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod
```

---

### 6️⃣ Apply for Dev Environment

```bash
terraform workspace select dev
terraform apply -var-file="dev.tfvars" -auto-approve
```

✅ **Output:**

* ECS service created
* Load balancer provisioned
* Your app is ready to deploy!

---

🧠 **Notes:**

* ECS will pull Docker image from ECR using the image tag passed (`image_tag`).
* The task definition and service are created per environment.

---

# ✅ PHASE 3: GITHUB ACTIONS CI/CD WORKFLOW (BUILD → PUSH → DEPLOY)

---

## 🎯 Goal

> Automate the entire CI/CD pipeline using GitHub Actions:

* Build the Docker image
* Push it to ECR
* Deploy using Terraform to ECS

---

## 🧱 Workflow Design Summary

| Stage   | Description                                                      |
| ------- | ---------------------------------------------------------------- |
| Trigger | On push to `dev`, `staging`, or `main` branches                  |
| Build   | Install dependencies & run tests                                 |
| Package | Dockerize the app                                                |
| Push    | Push Docker image to AWS ECR                                     |
| Deploy  | Use Terraform to deploy latest image to ECS via environment vars |

---

## 📁 Directory Structure

```
.github/
└── workflows/
    └── deploy.yml
```

---

## 📜 `deploy.yml` — GitHub Actions CI/CD Workflow

```yaml
name: Deploy DoorFeed Web App

on:
  push:
    branches:
      - dev
      - staging
      - main
  workflow_dispatch:

env:
  AWS_REGION: us-east-1
  ECR_REPO: ${{ secrets.ECR_REPO_URL }}
  IMAGE_TAG: ${{ github.sha }}

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: ⬇️ Checkout Code
        uses: actions/checkout@v4

      - name: 🧪 Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: 📦 Install Dependencies & Test
        run: |
          npm install
          npm test

      - name: 🐳 Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: 🐳 Login to Amazon ECR
        run: |
          aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REPO

      - name: 🏗️ Build Docker Image
        run: docker build -t $ECR_REPO:$IMAGE_TAG .

      - name: 🚀 Push Docker Image to ECR
        run: docker push $ECR_REPO:$IMAGE_TAG

      - name: 🔧 Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: 📂 Terraform Init & Apply
        run: |
          cd terraform
          terraform init
          if [[ "${{ github.ref }}" == "refs/heads/dev" ]]; then
            terraform workspace select dev || terraform workspace new dev
            terraform apply -var-file="dev.tfvars" -auto-approve -var="image_tag=$IMAGE_TAG"
          elif [[ "${{ github.ref }}" == "refs/heads/staging" ]]; then
            terraform workspace select staging || terraform workspace new staging
            terraform apply -var-file="staging.tfvars" -auto-approve -var="image_tag=$IMAGE_TAG"
          elif [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            terraform workspace select prod || terraform workspace new prod
            terraform apply -var-file="prod.tfvars" -auto-approve -var="image_tag=$IMAGE_TAG"
          fi
```

---

## 🔐 Required GitHub Secrets

Ensure these are configured:

| Name                    | Description                                              |
| ----------------------- | -------------------------------------------------------- |
| `AWS_ACCESS_KEY_ID`     | IAM User key                                             |
| `AWS_SECRET_ACCESS_KEY` | IAM User secret                                          |
| `ECR_REPO_URL`          | Full ECR URI (e.g., `1234...amazonaws.com/doorfeed-app`) |

---

## ✅ Output Verification

| Check                  | Command/Link                              |
| ---------------------- | ----------------------------------------- |
| ECR pushed image       | AWS Console → ECR → Images                |
| ECS Service is updated | AWS Console → ECS → Service Events        |
| App URL via ALB        | Terraform output or AWS Console → ALB DNS |

---

# ✅ PHASE 4: LOGGING, MONITORING & ALERTING (CLOUDWATCH + SNS)

---

## 🎯 Goal

> Enable monitoring of the application, infrastructure, and pipeline. Receive alerts for ECS errors, CPU/memory spikes, or deployment failures.

---

## 🧱 What We’ll Monitor

| Component     | Metric                             | Tool/Service      |
| ------------- | ---------------------------------- | ----------------- |
| ECS Service   | CPU, Memory, Deployment Failures   | CloudWatch        |
| ALB           | 5xx Errors, Latency, Health Checks | CloudWatch        |
| Logs          | App Console Output                 | CloudWatch Logs   |
| Notifications | Alerts on email/Slack              | SNS → Email/Slack |

---

## 🪵 Step 1: Enable CloudWatch Logs for ECS

### ✅ ECS Task Definition (`main.tf`):

Add log configuration:

```hcl
logConfiguration {
  logDriver = "awslogs"
  options = {
    awslogs-group         = "/ecs/${var.env}-doorfeed-app"
    awslogs-region        = var.aws_region
    awslogs-stream-prefix = "ecs"
  }
}
```

---

## 📈 Step 2: Create CloudWatch Alarms

### ✅ Example – ECS High CPU

```hcl
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "${var.env}-ecs-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = 60
  statistic           = "Average"
  threshold           = 70
  alarm_description   = "ECS CPU usage above 70%"
  dimensions = {
    ClusterName = aws_ecs_cluster.main.name
    ServiceName = aws_ecs_service.app.name
  }
  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

---

## 📣 Step 3: Create SNS Topic for Alerts

```hcl
resource "aws_sns_topic" "alerts" {
  name = "${var.env}-doorfeed-alerts"
}

resource "aws_sns_topic_subscription" "email_alerts" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = "devops.student@example.com"  # change to your email
}
```

📧 Confirm the email in your inbox to start receiving alerts.

---

## 🚦 Step 4: Monitor ALB Errors or Health Check Failures

Create alarms for:

* `HTTPCode_ELB_5XX_Count`
* `UnhealthyHostCount`

These will help track application crashes or misconfigurations.

---

## ✅ Output Verification

| Check                       | Tool              | Verification               |
| --------------------------- | ----------------- | -------------------------- |
| ECS Logs in CloudWatch      | CloudWatch Logs   | `/ecs/dev-doorfeed-app`    |
| Alarms Active               | CloudWatch Alarms | Trigger based on threshold |
| Email Notifications Working | SNS + Email       | Email received on alert    |

---

# ✅ PHASE 5: SECURITY, TESTING & PIPELINE FAILURE HANDLING

---

## 🎯 Goal

> Strengthen the CI/CD pipeline with proper security, structured testing stages, and automatic failure response to ensure production readiness.

---

## 🔐 SECURITY BEST PRACTICES

---

### 1️⃣ IAM Permissions (Least Privilege)

| Resource            | IAM Policy Attached                      |
| ------------------- | ---------------------------------------- |
| GitHub Actions User | ECR, ECS, S3, SecretsManager, CloudWatch |
| ECS Task Execution  | `AmazonECSTaskExecutionRolePolicy`       |

🛠️ Use managed policy for ECS task execution role:

```bash
arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

---

### 2️⃣ GitHub Secrets & OIDC Integration (Optional Advanced)

Use GitHub’s OIDC integration to **avoid static AWS keys**:

🔗 [GitHub OIDC with AWS IAM](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)

---

### 3️⃣ Secrets Handling for App Config

* Store `DATABASE_URL`, `API_KEYS`, etc. in **AWS Secrets Manager**
* Inject into ECS Task Definition via environment variables

---

## 🧪 TESTING PRACTICES IN THE PIPELINE

---

| Stage       | Test Type         | Tool Used           | Location in Workflow |
| ----------- | ----------------- | ------------------- | -------------------- |
| Build       | Unit Tests        | Jest / Mocha        | After `npm install`  |
| Pre-Deploy  | Integration Tests | Postman / Newman    | Optional stage       |
| Post-Deploy | Smoke Test        | curl + health check | ECS health endpoint  |

---

### 🔍 Sample Command: Unit Testing

```bash
npm run test
```

### 🔍 Sample Command: Postman API Testing

```bash
newman run collection.json --env-var baseUrl=http://alb-url
```

---

## 💥 PIPELINE FAILURE HANDLING

---

### 🚨 When Failure Occurs:

| Failure Stage     | Action Taken                       |
| ----------------- | ---------------------------------- |
| Docker Build Fail | Pipeline stops, email sent via SNS |
| Terraform Apply   | Rollback skipped, log error        |
| ECS Health Check  | Service rollback handled by ECS    |

---

### 🔁 Proactive Failure Minimization

| Strategy            | Implementation                              |
| ------------------- | ------------------------------------------- |
| Terraform Plan      | Add `terraform plan` before `apply`         |
| Manual Approval     | Add `workflow_dispatch` to restrict prod    |
| Version Locking     | Lock Docker + Terraform versions            |
| Canary / Blue-Green | Use ECS deployment config (advanced option) |

---

## ✅ Output Verification

* GitHub Actions dashboard shows failed job → ❌
* Email/Slack notification received → ✅
* ECS status rolls back failed deployment automatically

---

# ✅ PHASE 6: CLEANUP, `README.md`, AND SUBMISSION PREP

---

## 🎯 Goal

> Prepare the student for final project submission with cleanup instructions, a production-grade `README.md`, and a polished PDF file.

---

## 🧹 CLEANUP – AWS & LOCAL RESOURCES

---

### 1️⃣ Destroy Terraform Resources per Environment

📦 From within your `terraform/` folder:

```bash
terraform workspace select dev
terraform destroy -var-file="dev.tfvars" -auto-approve

terraform workspace select staging
terraform destroy -var-file="staging.tfvars" -auto-approve

terraform workspace select prod
terraform destroy -var-file="prod.tfvars" -auto-approve
```

---

### 2️⃣ Delete ECR Images (Optional)

```bash
aws ecr batch-delete-image \
  --repository-name doorfeed-app \
  --image-ids imageTag=latest
```

---

### 3️⃣ Delete GitHub Secrets (if rotating credentials)

* Go to GitHub repo → Settings → Secrets and variables → Remove/rotate secrets if needed

---

## 📦 FINAL `README.md`

```md
# 🚀 DoorFeed CI/CD Pipeline – AWS DevOps Assessment

## 🧩 Overview

Automated CI/CD pipeline deploying a GitHub-hosted web app to AWS ECS using GitHub Actions, Docker, and Terraform. Supports development, staging, and production environments.

---

## 📁 Project Structure

```

doorfeed-cicd-project/
├── .github/workflows/deploy.yml
├── terraform/
│   ├── main.tf, dev.tfvars, prod.tfvars...
├── Dockerfile
├── README.md
└── test/

````

---

## 🛠️ Tech Stack

- GitHub Actions
- Terraform
- Docker + Amazon ECR
- ECS Fargate
- AWS Secrets Manager
- CloudWatch + SNS
- Jest + Newman (Testing)

---

## 🔄 CI/CD Flow

1. **Code Push** → GitHub Actions triggers build
2. **Test & Build** → Docker image created
3. **Push** → Image pushed to ECR
4. **Deploy** → Terraform applies ECS update
5. **Monitor** → Logs/alerts in CloudWatch

---

## 🔐 Secrets

| Name                     | Description                 |
|--------------------------|-----------------------------|
| `AWS_ACCESS_KEY_ID`      | GitHub → AWS auth           |
| `AWS_SECRET_ACCESS_KEY`  | IAM user                    |
| `ECR_REPO_URL`           | ECR repo URL                |

---

## 📜 Terraform Usage

```bash
terraform init
terraform workspace new dev
terraform apply -var-file="dev.tfvars"
````

---

## 🧪 Testing

* Unit Tests: `npm test`
* API Tests: `newman run collection.json`
* Health: ECS Load Balancer check

---

## 🧾 Cleanup

```bash
terraform destroy -var-file="dev.tfvars"
```

---

## 📬 Author

Submitted for DevOps Technical Assessment – \[DoorFeed]

```

---

## 📝 Final Submission PDF

I'll now generate a **clean final PDF** including:

- Problem statement
- Architecture & diagrams
- All phases
- GitHub Actions code
- Terraform structure
- Testing + monitoring
- Cleanup

🛠️ Generating PDF...
```


✅ Your final CI/CD assessment submission PDF is ready!

📄 [Click here to download DoorFeed\_CICD\_Submission\_Final.pdf](sandbox:/mnt/data/DoorFeed_CICD_Submission_Final.pdf)

It includes:

* Problem statement
* Tech stack and CI/CD architecture
* Phase-wise implementation (Terraform, GitHub Actions, Monitoring)
* Sample YAML workflow
* Cleanup and testing notes

Let me know if you'd like me to:

* Include architecture diagrams
* Convert this into a GitHub project
* Generate a `.zip` with all YAML, Dockerfile, and Terraform templates

All set for submission ✅
