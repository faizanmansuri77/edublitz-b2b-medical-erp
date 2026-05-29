# EduBlitz Medical B2B ERP — Complete Deployment Guide
### From EC2 Launch → Jenkins CI/CD → Kubernetes on EKS → Live Application

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites & Accounts](#2-prerequisites--accounts)
3. [Phase 1 — Launch & Configure the Jenkins EC2 Server](#3-phase-1--launch--configure-the-jenkins-ec2-server)
4. [Phase 2 — Install Jenkins & Required Tools](#4-phase-2--install-jenkins--required-tools)
5. [Phase 3 — AWS Infrastructure Setup (Manual)](#5-phase-3--aws-infrastructure-setup-manual)
6. [Phase 4 — MongoDB Atlas Setup](#6-phase-4--mongodb-atlas-setup)
7. [Phase 5 — ECR — Container Registry](#7-phase-5--ecr--container-registry)
8. [Phase 6 — EKS Cluster Setup](#8-phase-6--eks-cluster-setup)
9. [Phase 7 — Configure Kubernetes Cluster](#9-phase-7--configure-kubernetes-cluster)
10. [Phase 8 — Deploy Application Manifests to EKS](#10-phase-8--deploy-application-manifests-to-eks)
11. [Phase 9 — Jenkins Pipeline Setup](#11-phase-9--jenkins-pipeline-setup)
12. [Phase 10 — Frontend Deployment (S3 + CloudFront)](#12-phase-10--frontend-deployment-s3--cloudfront)
13. [Phase 11 — DNS & Domain Configuration](#13-phase-11--dns--domain-configuration)
14. [Phase 12 — Verify Live Application](#14-phase-12--verify-live-application)
15. [Troubleshooting Reference](#15-troubleshooting-reference)
16. [Security Hardening Checklist](#16-security-hardening-checklist)

---

## 1. Architecture Overview

```
Internet
   │
   ├── CloudFront CDN ─── S3 (React SPA)
   │
   └── Route53 (api.med-erp.yourdomain.com)
          │
          └── AWS ALB (created by EKS Ingress Controller)
                 │
                 └── EKS Cluster (namespace: med-erp)
                        ├── user-service    (port 8081) — Auth, JWT, RBAC
                        ├── product-service (port 8082) — Products, Inventory
                        └── order-service   (port 8083) — Orders, Billing
                               │
                               └── MongoDB Atlas (3 separate databases)
```

**Tech Stack Summary**

| Component | Technology |
|---|---|
| Frontend | React 18 + Vite + TailwindCSS, hosted on S3+CloudFront |
| Backend | 3 Spring Boot 3.x microservices (Java 17) |
| Database | MongoDB Atlas (cloud-managed) |
| Auth | JWT (RS256), role-based (ADMIN / DISTRIBUTOR / HOSPITAL) |
| Container Registry | AWS ECR |
| Orchestration | AWS EKS (Kubernetes) |
| CI/CD | Jenkins (running on EC2) |
| IaC | Terraform (optional, modular) |
| Security Scanning | Trivy (container images) |
| Code Quality | SonarCloud |

---

## 2. Prerequisites & Accounts

Before starting, ensure you have the following ready:

| Requirement | Details |
|---|---|
| AWS Account | With IAM admin access |
| MongoDB Atlas Account | Free tier works for dev; M10+ for prod |
| GitHub/GitLab Account | Repository hosting for source code |
| SonarCloud Account | For quality gate (free for open-source) |
| Domain Name | For Route53/SSL (e.g., yourdomain.com) |
| Local Machine | AWS CLI, kubectl, eksctl installed |

**Required AWS services that will be used:**
- EC2 (Jenkins server)
- EKS (Kubernetes cluster)
- ECR (Docker image registry)
- S3 (Frontend hosting)
- CloudFront (CDN)
- ACM (SSL/TLS certificates)
- Route53 (DNS)
- IAM (Roles and policies)
- ALB (Application Load Balancer — auto-created by EKS)

---

## 3. Phase 1 — Launch & Configure the Jenkins EC2 Server

### Step 1.1 — Create IAM Role for Jenkins EC2

Before launching EC2, create an IAM role so Jenkins can access AWS services without hard-coding credentials.

1. Open **AWS Console → IAM → Roles → Create Role**
2. Select **EC2** as the trusted entity
3. Attach these policies:
   - `AmazonEC2ContainerRegistryFullAccess`
   - `AmazonEKSClusterPolicy`
   - `AmazonEKSWorkerNodePolicy`
   - `AmazonS3FullAccess`
   - `CloudFrontFullAccess`
4. Name it: `Jenkins-EC2-Role`
5. Click **Create Role**

### Step 1.2 — Launch the EC2 Instance

1. Open **AWS Console → EC2 → Launch Instance**
2. Configure the instance:

| Setting | Value |
|---|---|
| Name | `jenkins-server` |
| AMI | Ubuntu Server 22.04 LTS (HVM, SSD) |
| Instance Type | `t3.medium` (minimum) or `t3.large` (recommended) |
| Key Pair | Create new or select existing `.pem` key |
| VPC | Default VPC or your custom VPC |
| Subnet | Public subnet |
| Auto-assign Public IP | Enabled |
| Storage | 30 GB gp3 (increase to 50 GB for Maven cache) |

3. Under **Security Group**, create a new one named `jenkins-sg` with these inbound rules:

| Type | Protocol | Port | Source | Purpose |
|---|---|---|---|---|
| SSH | TCP | 22 | Your IP only | Server access |
| Custom TCP | TCP | 8080 | 0.0.0.0/0 | Jenkins Web UI |
| Custom TCP | TCP | 50000 | 0.0.0.0/0 | Jenkins agent JNLP |

4. Under **Advanced Details → IAM instance profile**, select `Jenkins-EC2-Role`
5. Click **Launch Instance**

### Step 1.3 — Connect to the Instance

```bash
# Replace with your actual key file and EC2 public IP
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>
```

### Step 1.4 — Update System Packages

```bash
sudo apt-get update -y
sudo apt-get upgrade -y
```

---

## 4. Phase 2 — Install Jenkins & Required Tools

Run all commands via SSH on the EC2 instance.

### Step 2.1 — Install Java 21 (required for Jenkins)

```bash
sudo apt-get install -y fontconfig openjdk-21-jre
java -version
# Expected output: openjdk version "17.x.x"
```

### Step 2.2 — Install Jenkins

```bash
# Add Jenkins GPG key and repository
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update -y
sudo apt-get install -y jenkins

# Start and enable Jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

### Step 2.3 — Install Docker

Jenkins pipelines use Docker-in-Docker (DinD) via Kubernetes pods. Docker must also be on the host for local builds if needed.

```bash
# Install Docker
sudo apt-get install -y ca-certificates curl gnupg lsb-release

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# Add jenkins and ubuntu users to docker group
sudo usermod -aG docker jenkins
sudo usermod -aG docker ubuntu

# Start Docker
sudo systemctl enable docker
sudo systemctl start docker
docker --version
```

### Step 2.4 — Install AWS CLI v2

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt-get install -y unzip
unzip awscliv2.zip
sudo ./install
aws --version
# Expected: aws-cli/2.x.x
```

### Step 2.5 — Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

### Step 2.6 — Install eksctl

```bash
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

### Step 2.7 — Install Helm

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | \
  sudo tee /usr/share/keyrings/helm.gpg > /dev/null

sudo apt-get install -y apt-transport-https

echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/usr/share/keyrings/helm.gpg] \
  https://baltocdn.com/helm/stable/debian/ all main" | \
  sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

sudo apt-get update -y
sudo apt-get install -y helm
helm version
```

### Step 2.8 — Configure Jenkins Initial Setup

1. Get the initial admin password:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
2. Open browser: `http://<EC2-PUBLIC-IP>:8080`
3. Enter the password from above
4. Click **Install suggested plugins** and wait for installation
5. Create your admin user (save credentials safely)
6. Set Jenkins URL to `http://<EC2-PUBLIC-IP>:8080`

### Step 2.9 — Install Required Jenkins Plugins

Go to **Manage Jenkins → Plugins → Available plugins** and install:

- Kubernetes (for Kubernetes agent pods)
- Docker Pipeline
- Amazon ECR
- AWS Credentials
- Pipeline: AWS Steps
- SonarQube Scanner
- JUnit
- Git
- Blue Ocean (optional, for better UI)

Restart Jenkins after installation.

### Step 2.10 — Install Kubernetes Plugin for Jenkins (Agent Pods)

This project uses **Kubernetes-based Jenkins agents** (pods spin up per build). Configure the Kubernetes cloud:

1. Go to **Manage Jenkins → Clouds → New Cloud → Kubernetes**
2. Set **Kubernetes URL**: Run on EC2 and get value with:
   ```bash
   kubectl cluster-info | grep "Kubernetes control plane"
   ```
3. Set **Jenkins URL**: `http://<EC2-PRIVATE-IP>:8080`
4. Set **Jenkins tunnel**: `<EC2-PRIVATE-IP>:50000`
5. Click **Test Connection** — it should succeed
6. Save

---

## 5. Phase 3 — AWS Infrastructure Setup (Manual)

> **Note:** This project includes Terraform modules in `terraform/` for automated infrastructure. The manual steps below are the equivalent — follow one approach only.

### Step 3.1 — Create VPC (if not using default)

For production, create a dedicated VPC:

```bash
# Using AWS CLI
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=med-erp-vpc}]'
```

For the full Terraform approach, see `terraform/modules/vpc/main.tf` in the repository.

### Step 3.2 — Create an ACM SSL Certificate

Required for HTTPS on ALB and CloudFront:

1. **AWS Console → Certificate Manager → Request Certificate**
2. Select **Request a public certificate**
3. Add domain names:
   - `yourdomain.com`
   - `*.yourdomain.com`
   - `api.med-erp.yourdomain.com`
4. Select **DNS validation**
5. Click **Request** — copy the CNAME record shown
6. Add that CNAME to your DNS provider (or Route53)
7. Wait for status to show **Issued** (up to 30 minutes)
8. **Save the Certificate ARN** — you'll need it later

---

## 6. Phase 4 — MongoDB Atlas Setup

### Step 4.1 — Create Atlas Cluster

1. Log in at [cloud.mongodb.com](https://cloud.mongodb.com)
2. Create a new project: `med-erp-prod`
3. **Build a Database → Shared (M0 free) or M10+ for production**
4. Select **AWS** as provider, region `us-east-1`
5. Name the cluster: `med-erp-cluster`
6. Click **Create**

### Step 4.2 — Create Databases and Users

In MongoDB Atlas → **Database Access → Add New Database User**:

Create three separate users (least-privilege):

| Username | Password | Database | Role |
|---|---|---|---|
| `user-svc` | strong password | `users_db` | `readWrite` |
| `product-svc` | strong password | `products_db` | `readWrite` |
| `order-svc` | strong password | `orders_db` | `readWrite` |

### Step 4.3 — Allow Network Access

In Atlas → **Network Access → Add IP Address**:

- Add the **NAT Gateway IP** of your EKS cluster's VPC (for production)
- During testing you can temporarily add `0.0.0.0/0` (not recommended for prod)

### Step 4.4 — Get Connection Strings

In Atlas → **Database → Connect → Drivers → Java**:

Your connection strings will look like:

```
mongodb+srv://user-svc:<PASSWORD>@med-erp-cluster.xxxxx.mongodb.net/users_db?retryWrites=true&w=majority
mongodb+srv://product-svc:<PASSWORD>@med-erp-cluster.xxxxx.mongodb.net/products_db?retryWrites=true&w=majority
mongodb+srv://order-svc:<PASSWORD>@med-erp-cluster.xxxxx.mongodb.net/orders_db?retryWrites=true&w=majority
```

**Save these three strings — they become Kubernetes Secrets.**

---

## 7. Phase 5 — ECR — Container Registry

### Step 5.1 — Create ECR Repositories

Create one repository per microservice:

```bash
aws ecr create-repository \
  --repository-name med-erp/user-service \
  --region us-east-1 \
  --image-scanning-configuration scanOnPush=true

aws ecr create-repository \
  --repository-name med-erp/product-service \
  --region us-east-1 \
  --image-scanning-configuration scanOnPush=true

aws ecr create-repository \
  --repository-name med-erp/order-service \
  --region us-east-1 \
  --image-scanning-configuration scanOnPush=true
```

### Step 5.2 — Note Your ECR Registry URL

```bash
aws sts get-caller-identity --query Account --output text
# Output: 123456789012 (your AWS account ID)

# Your ECR base URL will be:
# 123456789012.dkr.ecr.us-east-1.amazonaws.com
```

Save this as `ECR_REGISTRY` — it's referenced throughout the Jenkinsfiles.

---

## 8. Phase 6 — EKS Cluster Setup

### Step 6.1 — Create EKS Cluster

This takes 15–20 minutes. Run from the Jenkins EC2 or your local machine with AWS CLI configured:

```bash
eksctl create cluster \
  --name med-erp-prod-eks \
  --region us-east-1 \
  --version 1.29 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 5 \
  --managed \
  --with-oidc \
  --alb-ingress-access \
  --full-ecr-access
```

> The `--with-oidc` flag enables IAM Roles for Service Accounts (IRSA), needed for the ALB controller.

### Step 6.2 — Verify Cluster

```bash
# Update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name med-erp-prod-eks

# Check nodes
kubectl get nodes
# You should see 2 Ready nodes

# Check cluster info
kubectl cluster-info
```

### Step 6.3 — Install AWS Load Balancer Controller

The Ingress in `k8s/ingress/ingress.yaml` uses annotations that require the AWS ALB Ingress Controller.

**Step A — Create IAM policy for ALB Controller:**

```bash
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

**Step B — Create IAM service account:**

```bash
eksctl create iamserviceaccount \
  --cluster=med-erp-prod-eks \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<YOUR_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

**Step C — Install via Helm:**

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=med-erp-prod-eks \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Verify it's running
kubectl get deployment -n kube-system aws-load-balancer-controller
```

### Step 6.4 — Install Metrics Server (required for HPA)

The project uses HorizontalPodAutoscalers. Metrics server must be running:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify
kubectl get deployment metrics-server -n kube-system
```

---

## 9. Phase 7 — Configure Kubernetes Cluster

### Step 7.1 — Create Namespace

```bash
kubectl apply -f k8s/namespace/namespace.yaml
# OR directly:
kubectl create namespace med-erp
kubectl get namespace med-erp
```

### Step 7.2 — Create Kubernetes Secrets

The secrets file `k8s/secrets/app-secrets.yaml` contains placeholder base64 values. Replace them with your real MongoDB URIs and JWT secret before applying.

**Generate your JWT secret (256-bit hex):**

```bash
openssl rand -hex 32
# Example output: 7f3a9b2c1d4e5f6a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a
```

**Base64 encode your values:**

```bash
# MongoDB URIs (replace with your actual Atlas connection strings)
echo -n "mongodb+srv://user-svc:PASSWORD@cluster.mongodb.net/users_db?retryWrites=true&w=majority" | base64

echo -n "mongodb+srv://product-svc:PASSWORD@cluster.mongodb.net/products_db?retryWrites=true&w=majority" | base64

echo -n "mongodb+srv://order-svc:PASSWORD@cluster.mongodb.net/orders_db?retryWrites=true&w=majority" | base64

# JWT Secret
echo -n "your-openssl-generated-hex-secret" | base64
```

**Edit `k8s/secrets/app-secrets.yaml`** and replace the base64 values:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: med-erp
type: Opaque
data:
  MONGODB_URI_USER: <base64-encoded-users-db-uri>
  MONGODB_URI_PRODUCT: <base64-encoded-products-db-uri>
  MONGODB_URI_ORDER: <base64-encoded-orders-db-uri>
  JWT_SECRET: <base64-encoded-jwt-secret>
```

**Apply secrets:**

```bash
kubectl apply -f k8s/secrets/app-secrets.yaml
kubectl get secrets -n med-erp
```

### Step 7.3 — Update ConfigMap

Edit `k8s/configmaps/app-config.yaml` — update the CORS origin to your actual frontend domain:

```yaml
data:
  JWT_EXPIRATION: "86400000"
  JWT_REFRESH_EXPIRATION: "604800000"
  CORS_ALLOWED_ORIGINS: "https://med-erp.yourdomain.com"   # ← Update this
  LOW_STOCK_THRESHOLD: "10"
  PRODUCT_SERVICE_URL: "http://product-service:8082/api/v1"
  USER_SERVICE_URL: "http://user-service:8081/api/v1"
```

```bash
kubectl apply -f k8s/configmaps/app-config.yaml
kubectl get configmap -n med-erp
```

### Step 7.4 — Update Deployment Manifests

Before first manual deploy, update each deployment YAML to point to your ECR registry.

In `k8s/deployments/user-service-deployment.yaml`, replace:
```yaml
image: YOUR_ECR_REGISTRY/user-service:latest
```
With:
```yaml
image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/med-erp/user-service:latest
```

Do the same for `product-service-deployment.yaml` and `order-service-deployment.yaml`.

### Step 7.5 — Update Ingress

Edit `k8s/ingress/ingress.yaml`:

```yaml
# Replace these two values:
alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:YOUR_ACCOUNT_ID:certificate/YOUR_CERT_ID
alb.ingress.kubernetes.io/security-groups: sg-XXXXXXXXXX

# Replace the host:
- host: api.med-erp.yourdomain.com    # ← your actual domain
```

---

## 10. Phase 8 — Deploy Application Manifests to EKS

This is the **first manual deployment** — subsequent deployments will be automated by Jenkins.

### Step 8.1 — Build Docker Images Locally (First Time)

On the Jenkins EC2 server (which has Docker and AWS CLI):

```bash
# Authenticate to ECR
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com"

aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin $ECR_REGISTRY

# Build and push each service
cd /path/to/edublitz-b2b-medical-erp-main

# user-service
docker build -t $ECR_REGISTRY/med-erp/user-service:latest ./user-service/
docker push $ECR_REGISTRY/med-erp/user-service:latest

# product-service
docker build -t $ECR_REGISTRY/med-erp/product-service:latest ./product-service/
docker push $ECR_REGISTRY/med-erp/product-service:latest

# order-service
docker build -t $ECR_REGISTRY/med-erp/order-service:latest ./order-service/
docker push $ECR_REGISTRY/med-erp/order-service:latest
```

> Each Dockerfile is a multi-stage build: Stage 1 compiles with Maven (JDK 17), Stage 2 creates a minimal runtime image (JRE 17). The app runs as a non-root user (`appuser`).

### Step 8.2 — Apply All Kubernetes Manifests

```bash
cd /path/to/edublitz-b2b-medical-erp-main

# 1. Namespace (already created above, idempotent)
kubectl apply -f k8s/namespace/namespace.yaml

# 2. Secrets
kubectl apply -f k8s/secrets/app-secrets.yaml

# 3. ConfigMap
kubectl apply -f k8s/configmaps/app-config.yaml

# 4. Services (ClusterIP — internal routing)
kubectl apply -f k8s/services/services.yaml

# 5. Deployments
kubectl apply -f k8s/deployments/user-service-deployment.yaml
kubectl apply -f k8s/deployments/product-service-deployment.yaml
kubectl apply -f k8s/deployments/order-service-deployment.yaml

# 6. HPA (auto-scaling)
kubectl apply -f k8s/hpa/hpa.yaml

# 7. Ingress (creates ALB)
kubectl apply -f k8s/ingress/ingress.yaml
```

### Step 8.3 — Verify All Pods Are Running

```bash
# Watch pods come up (takes 60-90 seconds for Spring Boot to start)
kubectl get pods -n med-erp -w

# Expected output:
# NAME                               READY   STATUS    RESTARTS
# user-service-xxxxx-xxxxx           1/1     Running   0
# user-service-xxxxx-yyyyy           1/1     Running   0
# product-service-xxxxx-xxxxx        1/1     Running   0
# product-service-xxxxx-yyyyy        1/1     Running   0
# order-service-xxxxx-xxxxx          1/1     Running   0
# order-service-xxxxx-yyyyy          1/1     Running   0
```

```bash
# Check services
kubectl get svc -n med-erp

# Check ingress (ALB DNS will appear after ~3 minutes)
kubectl get ingress -n med-erp

# Check HPA
kubectl get hpa -n med-erp

# Check pod logs if issues
kubectl logs -n med-erp deployment/user-service --tail=50
kubectl logs -n med-erp deployment/product-service --tail=50
kubectl logs -n med-erp deployment/order-service --tail=50
```

### Step 8.4 — Verify Health Endpoints

Once the ingress has an ALB DNS name, test the health endpoints:

```bash
# Get ALB DNS
ALB_DNS=$(kubectl get ingress med-erp-ingress -n med-erp -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo $ALB_DNS

# Test health endpoints
curl http://$ALB_DNS/api/v1/actuator/health     # user-service
# Expected: {"status":"UP"}
```

---

## 11. Phase 9 — Jenkins Pipeline Setup

### Step 9.1 — Configure Jenkins Credentials

Go to **Jenkins → Manage Jenkins → Credentials → System → Global credentials → Add Credentials**:

**Credential 1 — AWS Credentials:**
- Kind: `AWS Credentials`
- ID: `AWS_CREDENTIALS`
- Access Key ID: your AWS IAM user access key
- Secret Access Key: your AWS IAM user secret key

> **Tip:** Create a dedicated IAM user (`jenkins-ci-user`) with these policies: `AmazonEC2ContainerRegistryFullAccess`, `AmazonEKSClusterPolicy`, `AmazonS3FullAccess`, `CloudFrontFullAccess`

**Credential 2 — SonarCloud Token:**
- Kind: `Secret text`
- ID: `SONARCLOUD_TOKEN`
- Secret: your SonarCloud API token (get from sonarcloud.io → My Account → Security)

**Credential 3 — Kubeconfig File:**
- Kind: `Secret file`
- ID: `KUBECONFIG_CREDENTIAL`
- File: Upload your `~/.kube/config` file (contains EKS cluster access)

  Generate it first:
  ```bash
  aws eks update-kubeconfig --region us-east-1 --name med-erp-prod-eks
  cat ~/.kube/config
  ```

### Step 9.2 — Push Code to GitHub

Push the project to your GitHub repository. The pipelines are triggered by branch pushes.

```bash
cd edublitz-b2b-medical-erp-main
git init
git remote add origin https://github.com/YOUR_ORG/edublitz-med-erp.git
git add .
git commit -m "Initial commit"
git branch -M main
git push -u origin main
```

### Step 9.3 — Create Backend Jenkins Pipeline

1. **Jenkins Dashboard → New Item**
2. Name: `med-erp-backend`
3. Type: `Pipeline`
4. Click **OK**
5. Under **Pipeline**:
   - Definition: `Pipeline script from SCM`
   - SCM: `Git`
   - Repository URL: your GitHub repo URL
   - Credentials: add GitHub credentials if private
   - Branch: `*/main`
   - Script Path: `jenkins/Jenkinsfile.backend`
6. Under **Build Triggers**:
   - Check `GitHub hook trigger for GITScm polling` (for webhooks)
7. **Save**

**Add GitHub Webhook:**
- GitHub Repo → Settings → Webhooks → Add webhook
- Payload URL: `http://<EC2-PUBLIC-IP>:8080/github-webhook/`
- Content type: `application/json`
- Events: `Push events`

### Step 9.4 — Set Pipeline Environment Variables

In the pipeline job → **Configure → Pipeline → Environment variables**, add:

| Variable | Value |
|---|---|
| `AWS_ACCOUNT_ID` | `123456789012` (your actual account ID) |
| `DOMAIN_NAME` | `yourdomain.com` |
| `CLOUDFRONT_DISTRIBUTION_ID` | (fill after CloudFront setup in Phase 10) |

### Step 9.5 — Create Frontend Jenkins Pipeline

1. **Jenkins Dashboard → New Item**
2. Name: `med-erp-frontend`
3. Type: `Pipeline`
4. Script Path: `jenkins/Jenkinsfile.frontend`
5. Same SCM settings as backend
6. **Save**

### Step 9.6 — Pipeline Stages Explained

**Backend Pipeline (`Jenkinsfile.backend`)** runs these stages:

| Stage | What It Does | Branch |
|---|---|---|
| Checkout | Pulls latest code, captures git commit SHA | All |
| Build & Test | Runs `mvn clean verify` for all 3 services in parallel | All |
| SonarCloud Quality Gate | Scans code quality, blocks on failures | `main` only |
| Docker Build & Push | Builds images tagged with git commit SHA, pushes to ECR | `main` only |
| Trivy Security Scan | Scans images for HIGH/CRITICAL CVEs — fails build if found | `main` only |
| Deploy to EKS | `kubectl set image` rolling update, waits for rollout | `main` only |

**Frontend Pipeline (`Jenkinsfile.frontend`)** stages:

| Stage | What It Does |
|---|---|
| Checkout | Pull code |
| Install Dependencies | `npm ci` |
| Lint | `npm run lint` |
| Build React App | `npm run build` with env vars injected |
| Upload to S3 | Sync `dist/` to S3 with cache headers |
| Invalidate CloudFront | Purge CDN cache |

### Step 9.7 — Run First Build

1. Open `med-erp-backend` pipeline
2. Click **Build Now**
3. Open **Console Output** to watch live logs
4. First build on feature branches will run Build & Test only
5. After merging to `main`, full pipeline (including Docker push and deploy) runs

---

## 12. Phase 10 — Frontend Deployment (S3 + CloudFront)

### Step 10.1 — Create S3 Bucket for Frontend

```bash
aws s3 mb s3://med-erp-frontend-prod --region us-east-1

# Disable block public access (CloudFront will handle access control)
aws s3api put-public-access-block \
  --bucket med-erp-frontend-prod \
  --public-access-block-configuration \
  "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# Enable static website hosting
aws s3 website s3://med-erp-frontend-prod \
  --index-document index.html \
  --error-document index.html
```

### Step 10.2 — Set Bucket Policy

Create a file `bucket-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::med-erp-frontend-prod/*"
    }
  ]
}
```

```bash
aws s3api put-bucket-policy \
  --bucket med-erp-frontend-prod \
  --policy file://bucket-policy.json
```

### Step 10.3 — Create CloudFront Distribution

```bash
aws cloudfront create-distribution --distribution-config '{
  "CallerReference": "med-erp-frontend-'$(date +%s)'",
  "Origins": {
    "Quantity": 1,
    "Items": [{
      "Id": "S3-med-erp-frontend-prod",
      "DomainName": "med-erp-frontend-prod.s3-website-us-east-1.amazonaws.com",
      "CustomOriginConfig": {
        "HTTPPort": 80,
        "HTTPSPort": 443,
        "OriginProtocolPolicy": "http-only"
      }
    }]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3-med-erp-frontend-prod",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "Compress": true
  },
  "CustomErrorResponses": {
    "Quantity": 1,
    "Items": [{
      "ErrorCode": 404,
      "ResponsePagePath": "/index.html",
      "ResponseCode": "200",
      "ErrorCachingMinTTL": 0
    }]
  },
  "Enabled": true,
  "HttpVersion": "http2",
  "Comment": "Med ERP Frontend"
}'
```

**Save the Distribution ID** (e.g., `E1ABCDEF1234`) — add it to Jenkins environment variables as `CLOUDFRONT_DISTRIBUTION_ID`.

### Step 10.4 — Build and Deploy Frontend Manually (First Time)

```bash
cd frontend

# Copy example env and fill in values
cp .env.example .env

# Edit .env:
VITE_USER_SERVICE_URL=https://api.med-erp.yourdomain.com/api/v1
VITE_PRODUCT_SERVICE_URL=https://api.med-erp.yourdomain.com/api/v1
VITE_ORDER_SERVICE_URL=https://api.med-erp.yourdomain.com/api/v1

npm install
npm run build

# Deploy to S3
aws s3 sync dist/ s3://med-erp-frontend-prod/ \
  --delete \
  --cache-control "public,max-age=31536000,immutable" \
  --exclude "*.html"

aws s3 cp dist/index.html s3://med-erp-frontend-prod/index.html \
  --cache-control "no-cache,no-store,must-revalidate" \
  --content-type "text/html"
```

---

## 13. Phase 11 — DNS & Domain Configuration

### Step 11.1 — Get ALB DNS Name

```bash
kubectl get ingress -n med-erp
# HOSTS                           ADDRESS
# api.med-erp.yourdomain.com      k8s-mederpxxxxxxxx.us-east-1.elb.amazonaws.com
```

### Step 11.2 — Create Route53 Hosted Zone (if not exists)

```bash
aws route53 create-hosted-zone \
  --name yourdomain.com \
  --caller-reference $(date +%s)
```

Note the nameservers and update them in your domain registrar.

### Step 11.3 — Create DNS Records

**API subdomain (CNAME to ALB):**

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id YOUR_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.med-erp.yourdomain.com",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [{"Value": "k8s-mederpxxxxxxxx.us-east-1.elb.amazonaws.com"}]
      }
    }]
  }'
```

**Frontend subdomain (Alias to CloudFront):**

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id YOUR_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "med-erp.yourdomain.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z2FDTNDATAQYW2",
          "DNSName": "XXXXXXXXXXXX.cloudfront.net",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'
```

> CloudFront's hosted zone ID is always `Z2FDTNDATAQYW2` (AWS global constant).

---

## 14. Phase 12 — Verify Live Application

### Step 14.1 — Backend Health Checks

```bash
# Test all service health endpoints via ALB
curl https://api.med-erp.yourdomain.com/api/v1/actuator/health
# Expected: {"status":"UP","components":{"mongo":{"status":"UP"},...}}

# Test auth endpoint (should return 401 without token)
curl -X POST https://api.med-erp.yourdomain.com/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"wrong"}'
# Expected: 401 Unauthorized
```

### Step 14.2 — Swagger API Documentation

Each service exposes Swagger UI:

- User Service: `https://api.med-erp.yourdomain.com/api/v1/swagger-ui.html`
- Product Service: `https://api.med-erp.yourdomain.com/api/v1/swagger-ui.html` (via `/api/v1/products/...`)
- Order Service: `https://api.med-erp.yourdomain.com/api/v1/swagger-ui.html`

### Step 14.3 — Create First Admin User

```bash
curl -X POST https://api.med-erp.yourdomain.com/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@yourdomain.com",
    "password": "SecurePassword123!",
    "firstName": "Admin",
    "lastName": "User",
    "role": "ADMIN",
    "organizationName": "EduBlitz Admin"
  }'
```

### Step 14.4 — Access the Frontend

Navigate to `https://med-erp.yourdomain.com` in your browser. You should see the React login page.

### Step 14.5 — Verify Kubernetes Cluster Health

```bash
# All pods running
kubectl get pods -n med-erp
kubectl get pods -n kube-system

# HPA active
kubectl get hpa -n med-erp

# Ingress has an address
kubectl get ingress -n med-erp

# Services accessible
kubectl get svc -n med-erp

# Check resource usage
kubectl top pods -n med-erp
kubectl top nodes
```

### Step 14.6 — Verify CI/CD Pipeline

1. Make a small code change and push to `main`
2. Jenkins should auto-trigger the pipeline
3. Watch pipeline stages execute in Blue Ocean UI
4. After ~10 minutes, new image should be deployed to EKS
5. Verify with `kubectl rollout history deployment/user-service -n med-erp`

---

## 15. Troubleshooting Reference

### Pods not starting (CrashLoopBackOff)

```bash
kubectl describe pod <POD_NAME> -n med-erp
kubectl logs <POD_NAME> -n med-erp --previous
```

Common causes:
- MongoDB connection string wrong in Secret — re-encode and reapply
- JWT secret mismatch between services
- ECR image not pushed / wrong image name in deployment

### ALB not created after Ingress apply

```bash
kubectl describe ingress med-erp-ingress -n med-erp
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

Common causes:
- ALB Controller not installed or not running
- Security group ID in ingress annotations is wrong
- ACM certificate ARN incorrect

### Jenkins pipeline fails at Deploy to EKS

```bash
# Check if kubeconfig is valid
kubectl get nodes  # run on EC2

# Check AWS credentials have EKS access
aws eks describe-cluster --name med-erp-prod-eks --region us-east-1
```

### HPA not scaling

```bash
kubectl describe hpa -n med-erp
# If "unknown" metrics, metrics-server is not running
kubectl get deployment metrics-server -n kube-system
```

### Frontend shows blank page

- Check browser console for network errors
- Verify `VITE_*` environment variables are set correctly
- Check CloudFront has the right origin and index.html exists in S3:
  ```bash
  aws s3 ls s3://med-erp-frontend-prod/
  ```

---

## 16. Security Hardening Checklist

Before going fully live, verify these security items:

- [ ] Kubernetes Secrets contain real values (not placeholder base64 from the repo)
- [ ] JWT secret is a strong random 256-bit hex value
- [ ] MongoDB Atlas users use least-privilege (not `atlasAdmin`)
- [ ] MongoDB Atlas network access restricts to EKS NAT Gateway IPs only
- [ ] AWS Security Groups allow only required ports
- [ ] Jenkins EC2 Security Group restricts SSH (port 22) to your IP only
- [ ] ACM certificate is valid and ALB is redirecting HTTP → HTTPS
- [ ] Trivy scans are not suppressed (pipeline fails on HIGH/CRITICAL)
- [ ] SonarCloud quality gate is enforced (pipeline fails on quality gate failure)
- [ ] `k8s/secrets/app-secrets.yaml` with real values is **never committed to Git**
- [ ] ECR image scanning on push is enabled
- [ ] All pods run as non-root user (`appuser`) — verified in Dockerfiles
- [ ] Pod Anti-Affinity is configured (ensures replicas land on different nodes)
- [ ] HPA min replicas = 2 for all services (no single point of failure)

---

## Quick Reference — Useful Commands

```bash
# --- Kubernetes ---
kubectl get all -n med-erp                          # Full overview
kubectl rollout restart deployment/user-service -n med-erp    # Force restart
kubectl rollout undo deployment/user-service -n med-erp       # Roll back
kubectl exec -it <POD> -n med-erp -- /bin/sh        # Shell into pod
kubectl port-forward svc/user-service 8081:8081 -n med-erp    # Local test

# --- Jenkins ---
sudo systemctl restart jenkins
sudo cat /var/lib/jenkins/logs/tasks/*.log           # Jenkins logs

# --- ECR ---
aws ecr list-images --repository-name med-erp/user-service --region us-east-1

# --- EKS ---
eksctl get cluster --region us-east-1
aws eks update-kubeconfig --region us-east-1 --name med-erp-prod-eks

# --- Docker ---
docker system prune -f   # Clean up disk on Jenkins server
```

---

*Documentation version: 1.0 | Project: EduBlitz Medical B2B ERP | Stack: Spring Boot + React + MongoDB + EKS + Jenkins*
