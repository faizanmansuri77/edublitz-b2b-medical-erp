# EduBlitz Medical B2B ERP — Complete Deployment Guide
### EC2 Launch → Terraform Infrastructure → Jenkins CI/CD → Kubernetes on EKS → Live Application

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites & Accounts](#2-prerequisites--accounts)
3. [Phase 1 — Launch the Jenkins EC2 Server](#3-phase-1--launch-the-jenkins-ec2-server)
4. [Phase 2 — Install Jenkins & All Required Tools](#4-phase-2--install-jenkins--all-required-tools)
5. [Phase 3 — Terraform Infrastructure Provisioning](#5-phase-3--terraform-infrastructure-provisioning)
6. [Phase 4 — MongoDB Atlas Setup](#6-phase-4--mongodb-atlas-setup)
7. [Phase 5 — Docker Hub Registry Setup](#7-phase-5--docker-hub-registry-setup)
8. [Phase 6 — Post-Terraform: EKS Add-ons](#8-phase-6--post-terraform-eks-add-ons)
9. [Phase 7 — Configure Kubernetes Cluster](#9-phase-7--configure-kubernetes-cluster)
10. [Phase 8 — First Manual Docker Build & EKS Deploy](#10-phase-8--first-manual-docker-build--eks-deploy)
11. [Phase 9 — Jenkins Pipeline Setup](#11-phase-9--jenkins-pipeline-setup)
12. [Phase 10 — Frontend Deployment (S3 + CloudFront)](#12-phase-10--frontend-deployment-s3--cloudfront)
13. [Phase 11 — DNS Final Wiring](#13-phase-11--dns-final-wiring)
14. [Phase 12 — Verify Live Application](#14-phase-12--verify-live-application)
15. [Troubleshooting Reference](#15-troubleshooting-reference)
16. [Security Hardening Checklist](#16-security-hardening-checklist)

---

## 1. Architecture Overview

```
Internet
   │
   ├── CloudFront CDN ─── S3 Bucket (React SPA)
   │       (med-erp.yourdomain.com)
   │
   └── Route53 (api.med-erp.yourdomain.com)
          │
          └── AWS ALB  ← created automatically by EKS ALB Ingress Controller
                 │
                 └── EKS Cluster  (namespace: med-erp)
                        ├── user-service    :8081  (Auth, JWT, RBAC, Orgs)
                        ├── product-service :8082  (Products, Inventory)
                        └── order-service   :8083  (Orders, Billing)
                               │
                               └── MongoDB Atlas
                                      ├── users_db
                                      ├── products_db
                                      └── orders_db

CI/CD: Jenkins (on EC2)
       └── GitHub push → Pipeline triggers
              ├── mvn test → SonarCloud → Docker build
              ├── Push image → Docker Hub (orionpax77/med-erp-*)
              ├── Trivy security scan
              └── kubectl rollout → EKS
```

**Infrastructure provisioned by Terraform (modules in `terraform/`):**

| Module | What It Creates |
|---|---|
| `modules/vpc` | VPC, 3 public + 3 private subnets across AZs, IGW, NAT Gateways, route tables |
| `modules/eks` | EKS cluster, IAM roles, node group, OIDC provider |
| `modules/s3-cloudfront` | S3 bucket (private), CloudFront OAI distribution |
| `modules/route53` | A record (frontend → CloudFront), CNAME (api → ALB) |

> **Note:** The project repo includes an ECR module. Since we're using Docker Hub, that module is skipped. All other modules are used unchanged.

---

## 2. Prerequisites & Accounts

| Requirement | Details |
|---|---|
| AWS Account | IAM admin access |
| MongoDB Atlas Account | Free M0 for dev, M10+ for production |
| Docker Hub Account | Username: `orionpax77` |
| GitHub Account | For source code hosting + webhooks |
| SonarCloud Account | Free at sonarcloud.io (connect to GitHub) |
| Domain Name | Registered domain for Route53 (e.g., `yourdomain.com`) |
| Local Machine | AWS CLI, Terraform, kubectl installed |

---

## 3. Phase 1 — Launch the Jenkins EC2 Server

### Step 1.1 — Create IAM Role for Jenkins EC2

Jenkins needs to talk to EKS, S3, and CloudFront without hard-coded keys.

1. Open **AWS Console → IAM → Roles → Create Role**
2. Trusted entity: **EC2**
3. Attach these managed policies:
   - `AmazonEKSClusterPolicy`
   - `AmazonEKSWorkerNodePolicy`
   - `AmazonS3FullAccess`
   - `CloudFrontFullAccess`
4. Name it: `Jenkins-EC2-Role`
5. Click **Create Role**

### Step 1.2 — Launch EC2 Instance

1. Open **AWS Console → EC2 → Launch Instance**

| Setting | Value |
|---|---|
| Name | `jenkins-server` |
| AMI | Ubuntu Server 22.04 LTS (HVM, SSD) |
| Instance Type | `t3.large` (recommended — Maven builds are CPU heavy) |
| Key Pair | Create new `.pem` — save it safely |
| VPC | Default VPC (Terraform will create the production VPC separately) |
| Subnet | Any public subnet |
| Auto-assign Public IP | Enabled |
| Storage | 50 GB gp3 (Maven cache needs space) |

2. Create a Security Group named `jenkins-sg`:

| Type | Port | Source | Purpose |
|---|---|---|---|
| SSH | 22 | Your IP/32 | Server access |
| Custom TCP | 8080 | 0.0.0.0/0 | Jenkins Web UI |
| Custom TCP | 50000 | 0.0.0.0/0 | Jenkins agent JNLP |

3. Under **Advanced Details → IAM instance profile** → select `Jenkins-EC2-Role`
4. Click **Launch Instance**

### Step 1.3 — Connect to the Instance

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>
```

### Step 1.4 — Update System

```bash
sudo apt-get update -y && sudo apt-get upgrade -y
```

---

## 4. Phase 2 — Install Jenkins & All Required Tools

Run all commands on the EC2 instance via SSH.

### Step 2.1 — Install Java 17

```bash
sudo apt-get install -y fontconfig openjdk-17-jre
java -version
# Expected: openjdk version "17.x.x"
```

### Step 2.2 — Install Jenkins

```bash
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update -y
sudo apt-get install -y jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

### Step 2.3 — Install Docker

```bash
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

sudo usermod -aG docker jenkins
sudo usermod -aG docker ubuntu

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

### Step 2.5 — Install Terraform

```bash
sudo apt-get install -y gnupg software-properties-common

wget -O- https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor | \
  sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt-get update -y
sudo apt-get install -y terraform
terraform version
# Expected: Terraform v1.x.x
```

### Step 2.6 — Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

### Step 2.7 — Install eksctl

```bash
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

### Step 2.8 — Install Helm

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

### Step 2.9 — Configure AWS CLI on EC2

The EC2 has the IAM role attached, but you still need a region default:

```bash
aws configure set region us-east-1
aws configure set output json

# Verify identity (uses instance role, no keys needed)
aws sts get-caller-identity
# Should return your account ID and the Jenkins-EC2-Role ARN
```

### Step 2.10 — Jenkins Initial Setup

1. Get the initial admin password:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
2. Open `http://<EC2-PUBLIC-IP>:8080` in your browser
3. Enter the password
4. Click **Install suggested plugins** — wait for completion
5. Create your admin user and save the credentials
6. Set Jenkins URL to `http://<EC2-PUBLIC-IP>:8080`

### Step 2.11 — Install Additional Jenkins Plugins

Go to **Manage Jenkins → Plugins → Available plugins**, search and install:

- `Kubernetes`
- `Docker Pipeline`
- `CloudBees Docker Build and Publish`
- `AWS Credentials`
- `Pipeline: AWS Steps`
- `SonarQube Scanner`
- `JUnit`
- `Blue Ocean` (optional — cleaner pipeline UI)

Restart Jenkins after installation: `sudo systemctl restart jenkins`

---

## 5. Phase 3 — Terraform Infrastructure Provisioning

All Terraform code lives in the `terraform/` directory of the project. We will provision the **dev** environment first, then the same steps apply for prod.

### Step 3.1 — Create Terraform State Backend (S3 + DynamoDB)

Terraform uses remote state. Create these AWS resources manually first (only done once):

```bash
# Create S3 bucket for state storage
aws s3 mb s3://med-erp-terraform-state-dev --region us-east-1

# Enable versioning on the bucket
aws s3api put-bucket-versioning \
  --bucket med-erp-terraform-state-dev \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket med-erp-terraform-state-dev \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

# Block public access on state bucket
aws s3api put-public-access-block \
  --bucket med-erp-terraform-state-dev \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Create DynamoDB table for state locking
aws dynamodb create-table \
  --table-name med-erp-terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1

echo "Terraform backend resources created."
```

### Step 3.2 — Request ACM SSL Certificate (Required Before Terraform)

CloudFront requires an ACM certificate in `us-east-1` regardless of your region. Request it before running Terraform:

```bash
aws acm request-certificate \
  --domain-name "yourdomain.com" \
  --subject-alternative-names "*.yourdomain.com" \
  --validation-method DNS \
  --region us-east-1
```

**Validate the certificate:**
1. Go to **AWS Console → Certificate Manager → Certificates**
2. Find your pending certificate
3. Click **Create records in Route53** (if domain is in Route53) — OR add the CNAME manually to your DNS provider
4. Wait for status to change to **Issued** (~5–30 minutes)
5. **Copy the Certificate ARN** — you'll need it in `terraform.tfvars`

### Step 3.3 — Clone the Project to the Jenkins Server

```bash
cd /home/ubuntu
git clone https://github.com/YOUR_ORG/edublitz-b2b-medical-erp.git
cd edublitz-b2b-medical-erp
```

### Step 3.4 — Prepare Terraform Variables File

Navigate to the dev environment:

```bash
cd terraform/env/dev
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars` with your real values:

```hcl
# terraform/env/dev/terraform.tfvars

aws_region          = "us-east-1"
domain_name         = "yourdomain.com"
acm_certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
developer_ip_cidr   = "YOUR_MACHINE_PUBLIC_IP/32"
alb_dns_name        = ""   # Leave blank — fill AFTER first kubectl ingress apply
```

> Get your public IP: `curl ifconfig.me`

### Step 3.5 — Understand What Terraform Will Create

Running `terraform apply` for the **dev** environment provisions:

**VPC Module** (`modules/vpc`):
- 1 VPC (`10.10.0.0/16`)
- 3 public subnets (one per AZ) — tagged for ALB
- 3 private subnets (one per AZ) — EKS worker nodes live here
- 1 Internet Gateway
- 3 NAT Gateways (one per AZ for HA) + Elastic IPs
- Public and private route tables

**EKS Module** (`modules/eks`):
- EKS cluster `med-erp-dev-eks` running Kubernetes 1.30
- IAM role for the control plane + policies
- IAM role for worker nodes + policies (Worker, CNI, ECR read, SSM)
- Managed node group: `t3.medium`, SPOT capacity, 1–4 nodes
- OIDC provider for IRSA (IAM Roles for Service Accounts)
- Security group for the cluster API endpoint

**S3 + CloudFront Module** (`modules/s3-cloudfront`):
- Private S3 bucket `med-erp-frontend-dev` (versioned, encrypted)
- CloudFront Origin Access Identity (OAI)
- CloudFront distribution with HTTPS redirect, SPA 404→index.html fallback, 1-year asset caching

**Route53 Module** (`modules/route53`):
- A record alias: `dev.med-erp.yourdomain.com` → CloudFront
- CNAME: `dev.api.med-erp.yourdomain.com` → ALB DNS (filled in later)

### Step 3.6 — Initialize Terraform

```bash
cd /home/ubuntu/edublitz-b2b-medical-erp/terraform/env/dev

terraform init
```

Expected output:
```
Initializing the backend...
Successfully configured the backend "s3"!
Initializing modules...
- dns in ../../modules/route53
- eks in ../../modules/eks
- frontend in ../../modules/s3-cloudfront
- vpc in ../../modules/vpc
Terraform has been successfully initialized!
```

### Step 3.7 — Review the Execution Plan

```bash
terraform plan -out=tfplan
```

Review the output carefully. You should see approximately **50–60 resources** to be created including the VPC, subnets, NAT gateways, EKS cluster, node group, S3 bucket, and CloudFront distribution.

Key things to verify in the plan:
- VPC CIDR matches `10.10.0.0/16`
- EKS cluster name is `med-erp-dev-eks`
- CloudFront uses your ACM certificate ARN
- Route53 zone matches your domain

### Step 3.8 — Apply Terraform

```bash
terraform apply tfplan
```

> **Important:** This takes 15–25 minutes. EKS cluster creation alone takes 10–15 minutes. Do not interrupt it.

Expected final output:
```
Apply complete! Resources: 54 added, 0 changed, 0 destroyed.

Outputs:
cloudfront_distribution_id = "E1ABCDEF1234567"
cloudfront_domain_name     = "dXXXXXXXXXXXX.cloudfront.net"
eks_cluster_endpoint       = "https://XXXXXXXXXXXX.gr7.us-east-1.eks.amazonaws.com"
eks_cluster_name           = "med-erp-dev-eks"
s3_frontend_bucket         = "med-erp-frontend-dev"
vpc_id                     = "vpc-XXXXXXXXXXXXXXXXX"
```

**Save all these output values** — you will need them in later phases.

### Step 3.9 — Update kubeconfig to Access New EKS Cluster

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name med-erp-dev-eks

# Verify cluster access
kubectl get nodes
# Expected: 2 nodes in Ready state (may take 2-3 minutes after apply)
```

### Step 3.10 — Production Environment (When Ready)

When you're ready to provision production, follow the same steps using `terraform/env/prod/`:

```bash
# Create prod state bucket (same DynamoDB table can be shared)
aws s3 mb s3://med-erp-terraform-state-prod --region us-east-1

cd /home/ubuntu/edublitz-b2b-medical-erp/terraform/env/prod
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with prod values

terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

Prod differences (already configured in `terraform/env/prod/main.tf`):
- CIDR: `10.20.0.0/16` (separate from dev)
- Node type: `m5.xlarge` (instead of `t3.medium`)
- Capacity: `ON_DEMAND` (no spot interruptions)
- Node count: 2–10 (instead of 1–4)
- EKS API endpoint: private-only (`endpoint_public_access = false`)

---

## 6. Phase 4 — MongoDB Atlas Setup

### Step 4.1 — Create Atlas Project and Cluster

1. Log in at [cloud.mongodb.com](https://cloud.mongodb.com)
2. Create project: `med-erp-dev`
3. **Build a Database** → M0 (free) for dev, M10 for prod
4. Provider: **AWS**, Region: `us-east-1`
5. Cluster name: `med-erp-cluster`
6. Click **Create**

### Step 4.2 — Create Database Users (Least-Privilege)

Go to **Database Access → Add New Database User** for each:

| Username | Password | Database Access |
|---|---|---|
| `user-svc` | strong_password_1 | `readWrite` on `users_db` only |
| `product-svc` | strong_password_2 | `readWrite` on `products_db` only |
| `order-svc` | strong_password_3 | `readWrite` on `orders_db` only |

### Step 4.3 — Allow Network Access from EKS

Get the NAT Gateway IPs from Terraform output:

```bash
# These are the IPs EKS worker nodes use to reach the internet
aws ec2 describe-nat-gateways \
  --filter "Name=tag:Project,Values=med-erp" \
  --query 'NatGateways[*].NatGatewayAddresses[*].PublicIp' \
  --output text
```

In Atlas → **Network Access → Add IP Address**, add all 3 NAT Gateway IPs with `/32`.

### Step 4.4 — Get Connection Strings

Atlas → **Database → Connect → Drivers → Java**:

```
mongodb+srv://user-svc:PASSWORD@med-erp-cluster.xxxxx.mongodb.net/users_db?retryWrites=true&w=majority
mongodb+srv://product-svc:PASSWORD@med-erp-cluster.xxxxx.mongodb.net/products_db?retryWrites=true&w=majority
mongodb+srv://order-svc:PASSWORD@med-erp-cluster.xxxxx.mongodb.net/orders_db?retryWrites=true&w=majority
```

**Save all three** — they become Kubernetes Secrets in Phase 7.

---

## 7. Phase 5 — Docker Hub Registry Setup

We use Docker Hub (`orionpax77`) instead of ECR.

### Step 5.1 — Create Docker Hub Repositories

Log in at [hub.docker.com](https://hub.docker.com) and create these **public or private** repositories:

| Repository Name | Full Image Path |
|---|---|
| `med-erp-user-service` | `orionpax77/med-erp-user-service` |
| `med-erp-product-service` | `orionpax77/med-erp-product-service` |
| `med-erp-order-service` | `orionpax77/med-erp-order-service` |

### Step 5.2 — Create a Docker Hub Access Token

Docker Hub → **Account Settings → Security → New Access Token**:

- Token description: `jenkins-ci`
- Access permissions: `Read, Write, Delete`
- Copy the generated token — you will not see it again

### Step 5.3 — Update Kubernetes Deployments for Docker Hub

The deployment YAMLs in `k8s/deployments/` reference `YOUR_ECR_REGISTRY`. Update each file:

`k8s/deployments/user-service-deployment.yaml`:
```yaml
# Change this line:
image: YOUR_ECR_REGISTRY/user-service:latest
# To:
image: orionpax77/med-erp-user-service:latest
```

`k8s/deployments/product-service-deployment.yaml`:
```yaml
image: orionpax77/med-erp-product-service:latest
```

`k8s/deployments/order-service-deployment.yaml`:
```yaml
image: orionpax77/med-erp-order-service:latest
```

### Step 5.4 — Update Jenkins Backend Jenkinsfile for Docker Hub

The Jenkinsfile at `jenkins/Jenkinsfile.backend` was written for ECR. Update the Docker stages:

**Replace the `Docker Build & Push` stage with:**

```groovy
stage('Docker Build & Push') {
    when { branch 'main' }
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'DOCKERHUB_CREDENTIALS',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
        )]) {
            container('docker') {
                script {
                    sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"

                    ['user-service', 'product-service', 'order-service'].each { svc ->
                        def imageName  = "orionpax77/med-erp-${svc}:${IMAGE_TAG}"
                        def imageLatest = "orionpax77/med-erp-${svc}:latest"

                        sh """
                            docker build -t ${imageName} -t ${imageLatest} ${svc}/
                            docker push ${imageName}
                            docker push ${imageLatest}
                        """
                    }
                }
            }
        }
    }
}
```

**Replace the `Trivy Security Scan` stage with:**

```groovy
stage('Trivy Security Scan') {
    when { branch 'main' }
    steps {
        container('trivy') {
            script {
                ['user-service', 'product-service', 'order-service'].each { svc ->
                    sh """
                        trivy image \
                          --exit-code 1 \
                          --severity HIGH,CRITICAL \
                          --no-progress \
                          --format table \
                          orionpax77/med-erp-${svc}:${IMAGE_TAG} \
                        || (echo "Trivy found vulnerabilities in ${svc}" && exit 1)
                    """
                }
            }
        }
    }
}
```

**Replace the `Deploy to EKS` stage with:**

```groovy
stage('Deploy to EKS') {
    when { branch 'main' }
    steps {
        withCredentials([
            file(credentialsId: 'KUBECONFIG_CREDENTIAL', variable: 'KUBECONFIG'),
            [
                $class: 'AmazonWebServicesCredentialsBinding',
                credentialsId: 'AWS_CREDENTIALS',
                accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
            ]
        ]) {
            container('kubectl') {
                script {
                    sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}"

                    ['user-service', 'product-service', 'order-service'].each { svc ->
                        sh """
                            kubectl set image deployment/${svc} \
                              ${svc}=orionpax77/med-erp-${svc}:${IMAGE_TAG} \
                              -n ${K8S_NAMESPACE}

                            kubectl rollout status deployment/${svc} \
                              -n ${K8S_NAMESPACE} --timeout=5m
                        """
                    }
                }
            }
        }
    }
}
```

**Also update the `environment` block** at the top of `Jenkinsfile.backend` — remove ECR references:

```groovy
environment {
    AWS_REGION    = 'us-east-1'
    DOCKER_REGISTRY = 'orionpax77'
    EKS_CLUSTER   = 'med-erp-dev-eks'
    K8S_NAMESPACE = 'med-erp'
    IMAGE_TAG     = "${env.GIT_COMMIT.take(8)}"
}
```

If Docker Hub repositories are **private**, EKS needs a pull secret:

```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=orionpax77 \
  --docker-password=YOUR_ACCESS_TOKEN \
  --docker-email=your-email@example.com \
  -n med-erp
```

Then add to each deployment YAML under `spec.template.spec`:

```yaml
imagePullSecrets:
  - name: dockerhub-secret
```

---

## 8. Phase 6 — Post-Terraform: EKS Add-ons

### Step 6.1 — Install AWS Load Balancer Controller

The Ingress resource uses ALB annotations. This controller must be installed after EKS is up.

**Create the IAM policy:**

```bash
curl -o iam_policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

**Create IAM service account (uses OIDC provider Terraform created):**

```bash
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

eksctl create iamserviceaccount \
  --cluster=med-erp-dev-eks \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

**Install via Helm:**

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=med-erp-dev-vpc" \
  --query 'Vpcs[0].VpcId' --output text)

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=med-erp-dev-eks \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set vpcId=$VPC_ID

# Verify it's running
kubectl get deployment aws-load-balancer-controller -n kube-system
```

### Step 6.2 — Install Metrics Server (required for HPA)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Wait and verify
kubectl get deployment metrics-server -n kube-system
kubectl top nodes   # Should return CPU/memory data after ~2 minutes
```

---

## 9. Phase 7 — Configure Kubernetes Cluster

### Step 7.1 — Create Namespace

```bash
kubectl apply -f k8s/namespace/namespace.yaml
kubectl get namespace med-erp
```

### Step 7.2 — Create Kubernetes Secrets

The file `k8s/secrets/app-secrets.yaml` has placeholder base64 values. Replace them with your real secrets.

**Generate a strong JWT secret:**

```bash
openssl rand -hex 32
# Example: a3f9c2d1b4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1
```

**Base64 encode your values:**

```bash
# MongoDB URIs
echo -n "mongodb+srv://user-svc:PASSWORD@med-erp-cluster.xxxxx.mongodb.net/users_db?retryWrites=true&w=majority" | base64

echo -n "mongodb+srv://product-svc:PASSWORD@med-erp-cluster.xxxxx.mongodb.net/products_db?retryWrites=true&w=majority" | base64

echo -n "mongodb+srv://order-svc:PASSWORD@med-erp-cluster.xxxxx.mongodb.net/orders_db?retryWrites=true&w=majority" | base64

# JWT secret
echo -n "YOUR_OPENSSL_HEX_OUTPUT" | base64
```

**Edit `k8s/secrets/app-secrets.yaml`** with your encoded values:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: med-erp
type: Opaque
data:
  MONGODB_URI_USER:    <paste-base64-users-uri-here>
  MONGODB_URI_PRODUCT: <paste-base64-products-uri-here>
  MONGODB_URI_ORDER:   <paste-base64-orders-uri-here>
  JWT_SECRET:          <paste-base64-jwt-secret-here>
```

```bash
kubectl apply -f k8s/secrets/app-secrets.yaml
kubectl get secrets -n med-erp
```

### Step 7.3 — Apply ConfigMap

Edit `k8s/configmaps/app-config.yaml` — update the CORS origin to your domain:

```yaml
data:
  JWT_EXPIRATION:          "86400000"
  JWT_REFRESH_EXPIRATION:  "604800000"
  CORS_ALLOWED_ORIGINS:    "https://dev.med-erp.yourdomain.com"   # ← your frontend URL
  LOW_STOCK_THRESHOLD:     "10"
  PRODUCT_SERVICE_URL:     "http://product-service:8082/api/v1"
  USER_SERVICE_URL:        "http://user-service:8081/api/v1"
```

```bash
kubectl apply -f k8s/configmaps/app-config.yaml
```

### Step 7.4 — Update Ingress Manifest

Edit `k8s/ingress/ingress.yaml` — replace placeholder values:

```yaml
annotations:
  alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:YOUR_ACCOUNT_ID:certificate/YOUR_CERT_ID
  alb.ingress.kubernetes.io/security-groups: sg-XXXXXXXXXX   # leave blank to auto-create

spec:
  rules:
    - host: dev.api.med-erp.yourdomain.com    # ← your actual API subdomain
```

> To find available security groups:
> ```bash
> aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" --query 'SecurityGroups[*].[GroupId,GroupName]' --output table
> ```
> You can remove the `security-groups` annotation entirely to let ALB create its own.

---

## 10. Phase 8 — First Manual Docker Build & EKS Deploy

### Step 8.1 — Build and Push Images to Docker Hub

On the Jenkins EC2 server:

```bash
cd /home/ubuntu/edublitz-b2b-medical-erp

# Login to Docker Hub
docker login -u orionpax77
# Enter your Docker Hub access token when prompted

# Build and push user-service
docker build -t orionpax77/med-erp-user-service:latest ./user-service/
docker push orionpax77/med-erp-user-service:latest

# Build and push product-service
docker build -t orionpax77/med-erp-product-service:latest ./product-service/
docker push orionpax77/med-erp-product-service:latest

# Build and push order-service
docker build -t orionpax77/med-erp-order-service:latest ./order-service/
docker push orionpax77/med-erp-order-service:latest
```

Each Dockerfile is a 2-stage build:
- **Stage 1 (builder):** `eclipse-temurin:17-jdk-alpine` — compiles with Maven, skipping tests
- **Stage 2 (runtime):** `eclipse-temurin:17-jre-alpine` — minimal image, non-root user `appuser`

### Step 8.2 — Apply All Kubernetes Manifests

```bash
cd /home/ubuntu/edublitz-b2b-medical-erp

# Apply in correct dependency order:
kubectl apply -f k8s/namespace/namespace.yaml
kubectl apply -f k8s/secrets/app-secrets.yaml
kubectl apply -f k8s/configmaps/app-config.yaml
kubectl apply -f k8s/services/services.yaml
kubectl apply -f k8s/deployments/user-service-deployment.yaml
kubectl apply -f k8s/deployments/product-service-deployment.yaml
kubectl apply -f k8s/deployments/order-service-deployment.yaml
kubectl apply -f k8s/hpa/hpa.yaml
kubectl apply -f k8s/ingress/ingress.yaml
```

### Step 8.3 — Watch Pods Start Up

```bash
kubectl get pods -n med-erp -w
```

Spring Boot takes 60–90 seconds to start. Expected final state:

```
NAME                               READY   STATUS    RESTARTS
user-service-7d9f4c-xxxxx          1/1     Running   0
user-service-7d9f4c-yyyyy          1/1     Running   0
product-service-6b8d3f-xxxxx       1/1     Running   0
product-service-6b8d3f-yyyyy       1/1     Running   0
order-service-5c7e2a-xxxxx         1/1     Running   0
order-service-5c7e2a-yyyyy         1/1     Running   0
```

### Step 8.4 — Get the ALB DNS Name

After applying the ingress, the ALB is created (takes ~3 minutes):

```bash
kubectl get ingress -n med-erp
# NAME               CLASS   HOSTS                          ADDRESS
# med-erp-ingress    alb     dev.api.med-erp.yourdomain.com k8s-mederp-XXXXXX.us-east-1.elb.amazonaws.com
```

**Copy that ALB DNS name** — you need it for:
1. Route53 DNS record (Terraform `alb_dns_name` variable)
2. Verifying health endpoints

### Step 8.5 — Update Route53 ALB Record via Terraform

Now that you have the ALB DNS, update `terraform.tfvars`:

```hcl
alb_dns_name = "k8s-mederp-XXXXXX.us-east-1.elb.amazonaws.com"
```

Re-run Terraform to create the Route53 CNAME:

```bash
cd terraform/env/dev
terraform apply -var="alb_dns_name=k8s-mederp-XXXXXX.us-east-1.elb.amazonaws.com"
```

---

## 11. Phase 9 — Jenkins Pipeline Setup

### Step 9.1 — Configure Jenkins Credentials

Go to **Jenkins → Manage Jenkins → Credentials → System → Global credentials → Add Credentials**:

**Credential 1 — Docker Hub:**
- Kind: `Username with password`
- ID: `DOCKERHUB_CREDENTIALS`
- Username: `orionpax77`
- Password: your Docker Hub access token (created in Phase 5)

**Credential 2 — AWS (for EKS deploy):**
- Kind: `AWS Credentials`
- ID: `AWS_CREDENTIALS`
- Access Key ID and Secret Key from a dedicated IAM user `jenkins-ci` with policies:
  - `AmazonEKSClusterPolicy`
  - `AmazonS3FullAccess`
  - `CloudFrontFullAccess`

**Credential 3 — SonarCloud Token:**
- Kind: `Secret text`
- ID: `SONARCLOUD_TOKEN`
- Get from sonarcloud.io → **My Account → Security → Generate Token**

**Credential 4 — Kubeconfig:**
- Kind: `Secret file`
- ID: `KUBECONFIG_CREDENTIAL`
- Upload your `~/.kube/config` from the EC2 server:
  ```bash
  aws eks update-kubeconfig --region us-east-1 --name med-erp-dev-eks
  cat ~/.kube/config
  ```

### Step 9.2 — Configure Kubernetes Plugin for Jenkins Agents

The pipelines use Kubernetes pods as build agents (defined in the Jenkinsfile `agent { kubernetes {...} }` blocks).

1. **Manage Jenkins → Clouds → New Cloud → Kubernetes**
2. **Kubernetes URL:** `https://XXXXXXXXXXXX.gr7.us-east-1.eks.amazonaws.com` (from Terraform output)
3. **Jenkins URL:** `http://<EC2-PRIVATE-IP>:8080`
4. **Jenkins tunnel:** `<EC2-PRIVATE-IP>:50000`
5. **Test Connection** — it should say "Connected"
6. **Save**

### Step 9.3 — Push Code to GitHub

```bash
cd /home/ubuntu/edublitz-b2b-medical-erp
git init
git remote add origin https://github.com/YOUR_ORG/edublitz-med-erp.git
git add .
git commit -m "Initial commit — update for Docker Hub"
git branch -M main
git push -u origin main
```

### Step 9.4 — Create Backend Pipeline Job

1. **Jenkins Dashboard → New Item**
2. Name: `med-erp-backend`
3. Type: `Pipeline` → OK
4. **Build Triggers** → check `GitHub hook trigger for GITScm polling`
5. **Pipeline**:
   - Definition: `Pipeline script from SCM`
   - SCM: Git
   - Repo URL: `https://github.com/YOUR_ORG/edublitz-med-erp.git`
   - Branch: `*/main`
   - Script Path: `jenkins/Jenkinsfile.backend`
6. Save

**Add GitHub Webhook:**
- GitHub → Repo → Settings → Webhooks → Add webhook
- Payload URL: `http://<EC2-PUBLIC-IP>:8080/github-webhook/`
- Content type: `application/json`
- Events: Push events only

### Step 9.5 — Create Frontend Pipeline Job

1. **New Item** → `med-erp-frontend` → Pipeline
2. Same settings, Script Path: `jenkins/Jenkinsfile.frontend`
3. Add pipeline environment variables:

| Variable | Value |
|---|---|
| `S3_BUCKET_NAME` | `med-erp-frontend-dev` (from Terraform output) |
| `CLOUDFRONT_DISTRIBUTION_ID` | `E1ABCDEF1234567` (from Terraform output) |
| `DOMAIN_NAME` | `yourdomain.com` |

### Step 9.6 — Full Backend Pipeline Stages Reference

When you push to `main`, the backend pipeline executes these stages:

| # | Stage | Branch | Description |
|---|---|---|---|
| 1 | Checkout | All | Clones repo, reads git commit SHA (first 8 chars = image tag) |
| 2 | Build & Test | All | `mvn clean verify` for all 3 services **in parallel** — runs JUnit tests |
| 3 | SonarCloud Quality Gate | `main` only | Scans code coverage, bugs, vulnerabilities — pipeline fails if gate fails |
| 4 | Docker Build & Push | `main` only | Builds multi-stage Docker images, tags with commit SHA + `latest`, pushes to `orionpax77/med-erp-*` |
| 5 | Trivy Security Scan | `main` only | Scans all 3 images for HIGH/CRITICAL CVEs — pipeline fails if found |
| 6 | Deploy to EKS | `main` only | `kubectl set image` rolling update for each deployment, waits for rollout |

### Step 9.7 — Trigger and Watch First Pipeline

1. Open `med-erp-backend` → **Build Now**
2. Click the build number → **Console Output**
3. Watch the full pipeline execute

Alternatively install Blue Ocean plugin for a visual pipeline graph.

---

## 12. Phase 10 — Frontend Deployment (S3 + CloudFront)

> Terraform already created the S3 bucket and CloudFront distribution. This phase handles the first deploy and Jenkins automation.

### Step 10.1 — Build the React Frontend Locally (First Deploy)

```bash
cd /home/ubuntu/edublitz-b2b-medical-erp/frontend
cp .env.example .env

# Edit .env:
VITE_USER_SERVICE_URL=https://dev.api.med-erp.yourdomain.com/api/v1
VITE_PRODUCT_SERVICE_URL=https://dev.api.med-erp.yourdomain.com/api/v1
VITE_ORDER_SERVICE_URL=https://dev.api.med-erp.yourdomain.com/api/v1
```

```bash
# Install Node.js 20 if not present
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

npm install
npm run build
# Creates a dist/ folder with the compiled React app
```

### Step 10.2 — Upload to S3

```bash
S3_BUCKET="med-erp-frontend-dev"   # From Terraform output

# Upload assets with long cache (JS, CSS, images)
aws s3 sync dist/ s3://$S3_BUCKET/ \
  --region us-east-1 \
  --delete \
  --cache-control "public,max-age=31536000,immutable" \
  --exclude "*.html"

# Upload index.html with no cache (always fresh for SPA)
aws s3 cp dist/index.html s3://$S3_BUCKET/index.html \
  --region us-east-1 \
  --cache-control "no-cache,no-store,must-revalidate" \
  --content-type "text/html"
```

### Step 10.3 — Invalidate CloudFront Cache

```bash
DIST_ID="E1ABCDEF1234567"   # From Terraform output

INVALIDATION_ID=$(aws cloudfront create-invalidation \
  --distribution-id $DIST_ID \
  --paths "/*" \
  --query 'Invalidation.Id' \
  --output text)

echo "Invalidation created: $INVALIDATION_ID"

# Wait for it to complete
aws cloudfront wait invalidation-completed \
  --distribution-id $DIST_ID \
  --id $INVALIDATION_ID

echo "CloudFront cache cleared."
```

After this, the frontend pipeline (triggered by Git push) handles all future deploys automatically via `jenkins/Jenkinsfile.frontend`.

---

## 13. Phase 11 — DNS Final Wiring

Terraform's Route53 module already created the DNS records when you ran `terraform apply` with the `alb_dns_name` variable. Verify they are working:

```bash
# Check A record for frontend
dig dev.med-erp.yourdomain.com
# Should return CloudFront IP addresses

# Check CNAME for API
dig dev.api.med-erp.yourdomain.com
# Should resolve to your ALB DNS name
```

If you registered the domain outside Route53, update your domain registrar's nameservers to the Route53 NS records:

```bash
aws route53 list-hosted-zones --query 'HostedZones[?Name==`yourdomain.com.`].Id' --output text
# Get zone ID, then:
aws route53 get-hosted-zone --id /hostedzone/XXXXXXXXXXXXX \
  --query 'DelegationSet.NameServers'
```

---

## 14. Phase 12 — Verify Live Application

### Step 14.1 — Test Backend Health Endpoints

```bash
# Should return {"status":"UP"} with MongoDB connected
curl https://dev.api.med-erp.yourdomain.com/api/v1/actuator/health

# Should return 401 (proves auth is working)
curl -X POST https://dev.api.med-erp.yourdomain.com/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"x@x.com","password":"wrong"}'
```

### Step 14.2 — Register First Admin User

```bash
curl -X POST https://dev.api.med-erp.yourdomain.com/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@yourdomain.com",
    "password": "SecurePassword123!",
    "firstName": "Admin",
    "lastName": "User",
    "role": "ADMIN",
    "organizationName": "EduBlitz Admin Org"
  }'
```

### Step 14.3 — Access the Frontend

Open `https://dev.med-erp.yourdomain.com` in your browser. You should see the React login page.

Log in with the admin credentials you just created.

### Step 14.4 — Verify All K8s Resources

```bash
# Pods — all should be 1/1 Running
kubectl get pods -n med-erp

# Services — 3 ClusterIP services
kubectl get svc -n med-erp

# Ingress — should have ALB ADDRESS
kubectl get ingress -n med-erp

# HPA — should show current vs target replicas
kubectl get hpa -n med-erp

# Resource usage
kubectl top pods -n med-erp
kubectl top nodes
```

### Step 14.5 — Verify Full CI/CD Loop

1. Make a trivial change (e.g., update a comment in a Java file)
2. Commit and push to `main`
3. GitHub webhook triggers Jenkins
4. Watch pipeline in Jenkins UI:
   - Build & Test (parallel Maven)
   - SonarCloud Quality Gate
   - Docker build + push to `orionpax77/med-erp-*`
   - Trivy scan
   - `kubectl set image` rolling deploy
5. After ~10 minutes: `kubectl rollout history deployment/user-service -n med-erp` shows a new revision

---

## 15. Troubleshooting Reference

### Pods in CrashLoopBackOff

```bash
kubectl describe pod <POD_NAME> -n med-erp
kubectl logs <POD_NAME> -n med-erp --previous
```

Common causes:
- MongoDB URI is wrong — check base64 encoding, re-apply secret
- JWT secret has whitespace — regenerate with `openssl rand -hex 32`
- Docker Hub image not pushed / wrong tag — verify on hub.docker.com

### ALB Not Created After Ingress Apply

```bash
kubectl describe ingress med-erp-ingress -n med-erp
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

Common causes:
- ALB Controller not installed — re-run Phase 6.1
- Subnet tags missing — check Terraform VPC module applied `kubernetes.io/role/elb=1` tags
- ACM certificate ARN wrong or not in `us-east-1`

### Terraform Apply Fails

```bash
# State lock stuck (previous run interrupted)
aws dynamodb delete-item \
  --table-name med-erp-terraform-locks \
  --key '{"LockID": {"S": "med-erp-terraform-state-dev/dev/terraform.tfstate"}}'

# Re-run
terraform apply
```

### Jenkins Pipeline Cannot Push to Docker Hub

- Verify credential ID is exactly `DOCKERHUB_CREDENTIALS`
- Test login from EC2: `docker login -u orionpax77`
- Ensure the access token has write permissions

### SonarCloud Quality Gate Fails

- Check the SonarCloud dashboard at sonarcloud.io
- Common fix: ensure `sonar.organization` in Jenkinsfile matches your SonarCloud org slug
- To bypass during initial setup only: remove `sonar.qualitygate.wait=true`

### HPA Shows `<unknown>` Metrics

```bash
kubectl describe hpa -n med-erp
# If metrics unavailable, metrics-server is not working
kubectl rollout restart deployment/metrics-server -n kube-system
```

---

## 16. Security Hardening Checklist

Before promoting to production, verify every item:

- [ ] `k8s/secrets/app-secrets.yaml` with real values is **never committed to Git** — add it to `.gitignore`
- [ ] MongoDB Atlas users are least-privilege (not atlasAdmin)
- [ ] Atlas network access is restricted to NAT Gateway IPs only (not 0.0.0.0/0)
- [ ] JWT secret is a fresh `openssl rand -hex 32` — not the default from the repo
- [ ] Docker Hub repositories are set to **private** if the code is proprietary
- [ ] EKS API endpoint is private-only in prod (`endpoint_public_access = false` in `terraform/env/prod/main.tf`)
- [ ] Prod EKS uses ON_DEMAND nodes (no Spot interruptions)
- [ ] ACM certificate is valid and ALB enforces HTTPS redirect
- [ ] Trivy pipeline stage is NOT suppressed — builds must fail on HIGH/CRITICAL CVEs
- [ ] SonarCloud quality gate is enforced on every merge to main
- [ ] All pods run as non-root `appuser` — confirmed in Dockerfiles
- [ ] Pod Anti-Affinity is set (pods spread across different nodes — already in deployment YAMLs)
- [ ] HPA min replicas = 2 (no single point of failure)
- [ ] Jenkins EC2 Security Group: SSH (port 22) restricted to your IP only
- [ ] Terraform state S3 bucket has public access blocked and versioning enabled

---

## Quick Reference Commands

```bash
# ── Kubernetes ──────────────────────────────────────────────────────
kubectl get all -n med-erp                                    # Full overview
kubectl rollout restart deployment/user-service -n med-erp    # Force pod restart
kubectl rollout undo deployment/user-service -n med-erp       # Roll back one version
kubectl rollout history deployment/user-service -n med-erp    # See deploy history
kubectl exec -it <POD> -n med-erp -- /bin/sh                  # Shell into pod
kubectl port-forward svc/user-service 8081:8081 -n med-erp    # Local port-forward

# ── Terraform ──────────────────────────────────────────────────────
cd terraform/env/dev
terraform plan                         # Preview changes
terraform apply                        # Apply changes
terraform output                       # Show all outputs
terraform destroy                      # Tear down everything (careful!)

# ── Docker Hub ─────────────────────────────────────────────────────
docker pull orionpax77/med-erp-user-service:latest
docker images | grep orionpax77
docker system prune -f                 # Clean up disk on Jenkins server

# ── EKS ────────────────────────────────────────────────────────────
aws eks update-kubeconfig --region us-east-1 --name med-erp-dev-eks
eksctl get cluster --region us-east-1

# ── Jenkins ────────────────────────────────────────────────────────
sudo systemctl restart jenkins
sudo journalctl -u jenkins -f          # Live logs
```

---

*Documentation version: 2.0 | Project: EduBlitz Medical B2B ERP | Registry: Docker Hub (orionpax77) | Stack: Spring Boot 3 + React 18 + MongoDB Atlas + EKS + Terraform + Jenkins*
