## 🔁 Overview

This workflow:

1. Triggers on code push to the `master` branch.
2. Builds a Docker image.
3. Pushes it to Amazon ECR.
4. Forces a new deployment on ECS.

---

## 🗂️ Workflow Name and Trigger

```yaml
name: Deploy DoorFeed Web App.

on:
  push:
    branches:
      - master
```

* **`name:`** Display name of the workflow.
* **`on: push:`** This workflow will run automatically when a push happens on the `master` branch.
* 🔁 You can change this to `staging` or use `workflow_dispatch` for manual triggers.

---

## 🚀 Job: `deploy`

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
```

* **Job Name:** `deploy`
* **Runner:** GitHub-hosted Linux virtual machine.

### 🌍 Environment Variables

```yaml
    env:
      AWS_REGION: ${{ secrets.AWS_REGION }}
      ECR_REPO: ${{ secrets.ECR_REPO_URL }}
      IMAGE_TAG: latest
```

* Values pulled from GitHub Secrets:

  * `AWS_REGION` — AWS region like `us-east-1`.
  * `ECR_REPO_URL` — Your full ECR URI (e.g., `123456789012.dkr.ecr.us-east-1.amazonaws.com/doorfeed-app`)
  * `IMAGE_TAG` — Using `latest`, but could be `${{ github.sha }}` for immutable tagging.

---

## 🧩 Step-by-Step Breakdown

---

### 📥 Step 1: Checkout Code

```yaml
- name: 📥 Checkout Code
  uses: actions/checkout@v4
```

* Checks out the repo's code into the runner for building and deployment.

---

### ⚙️ Step 2: Setup Node.js

```yaml
- name: ⚙️ Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: 20
```

* Installs Node.js v20.
* Required to run tests and build your Node.js app.

---

### 🧪 Step 3: Install Dependencies & Run Tests

```yaml
- name: 🧪 Install Dependencies & Run Tests
  run: |
    npm install
    npm test || echo "Skipping tests for speed"
```

* Installs all NPM packages.
* Runs tests (`npm test`). If tests fail, it prints a message and continues — this avoids workflow failure.

> ✅ Best Practice: Fail the build if tests fail. This setup skips them for fast deploys.

---

### 🔐 Step 4: Configure AWS Credentials

```yaml
- name: 🔐 Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ env.AWS_REGION }}
```

* Sets up AWS CLI auth on the GitHub runner using secrets.
* Required to access ECR and ECS.

---

### 🐳 Step 5: Login to Amazon ECR

```yaml
- name: 🐳 Login to Amazon ECR
  run: |
    aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REPO
```

* Retrieves ECR login password via AWS CLI.
* Logs Docker into your private ECR registry.

---

### 📦 Step 6: Build and Push Docker Image

```yaml
- name: 📦 Build and Push Docker Image
  run: |
    docker build -t $ECR_REPO:$IMAGE_TAG .
    docker push $ECR_REPO:$IMAGE_TAG
```

* Builds the Docker image using your local `Dockerfile`.
* Tags it with the ECR URI + `:latest`.
* Pushes it to ECR.

✅ Example image:
`123456789012.dkr.ecr.us-east-1.amazonaws.com/doorfeed-app:latest`

---

### 🚀 Step 7: Deploy to ECS (Force New Deployment)

```yaml
- name: 🚀 Deploy to ECS (Force New Deployment)
  run: |
    aws ecs update-service \
      --cluster doorfeed-cluster \
      --service doorfeed-staging \
      --force-new-deployment \
      --region $AWS_REGION
```

* Triggers ECS to fetch the latest image from ECR and redeploy.
* `--force-new-deployment`: Even if task definition is unchanged, ECS will redeploy.

✅ Make sure your ECS Service (e.g., `doorfeed-staging`) is already pointing to `:latest`.

---

## ✅ Prerequisites You Must Set Up

| Component             | Status    | Details                                                                    |
| --------------------- | --------- | -------------------------------------------------------------------------- |
| **ECS Cluster**       | ✅ Ready   | e.g., `doorfeed-cluster` (Fargate or EC2)                                  |
| **ECS Service**       | ✅ Running | e.g., `doorfeed-staging` with image set to `:latest`                       |
| **ECR Repository**    | ✅ Created | Example: `doorfeed-app`                                                    |
| **Secrets in GitHub** | ✅ Added   | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `ECR_REPO_URL` |

---

## 🧪 Optional Enhancements

1. **Immutable Tags**
   Replace `latest` with `${{ github.sha }}` and update ECS task definition to use specific tags for rollback capabilities.

2. **Auto Task Definition Registration**
   Instead of relying on `:latest`, you can script task definition updates:

   ```bash
   aws ecs register-task-definition ... # and update service
   ```

3. **Notifications**
   Integrate Slack or Email notifications for deployment success/failure.

---

## 📘 Summary

| Step | Description                   |
| ---- | ----------------------------- |
| 1️⃣  | Checkout source code          |
| 2️⃣  | Set up Node.js environment    |
| 3️⃣  | Install packages and test app |
| 4️⃣  | Authenticate with AWS         |
| 5️⃣  | Log in to ECR                 |
| 6️⃣  | Build & push Docker image     |
| 7️⃣  | Redeploy updated image on ECS |

---
