Here is a proper step-by-step deployment documentation for your Medical ERP project based on your uploaded file 

# Medical ERP Application Deployment on AWS EKS

# Project Overview

This project is a production-grade Medical ERP application deployed on AWS using:

* React Frontend
* Spring Boot Microservices
* MongoDB Atlas
* Docker
* Kubernetes (EKS)
* Terraform
* AWS Load Balancer Controller
* Route53
* ACM
* CloudFront
* S3

---

# Architecture

## Frontend

* React Application
* Hosted on S3
* Distributed via CloudFront
* Domain managed using Route53

## Backend

* Spring Boot Microservices
* Deployed on AWS EKS
* Exposed using AWS ALB Ingress

## Database

* MongoDB Atlas

---

# Step 1 — Launch EC2 Instance

Launch EC2 instance with:

| Parameter      | Value             |
| -------------- | ----------------- |
| Instance Type  | m7i-flex.large    |
| OS             | Ubuntu 22.04      |
| Storage        | 30 GB             |
| Security Group | Allow 22, 80, 443 |

Connect to instance:

```bash
ssh -i key.pem ubuntu@<EC2_PUBLIC_IP>
```

---

# Step 2 — Clone Project Repository

```bash
git clone https://github.com/faizanmansuri77/edublitz-b2b-medical-erp.git

cd edublitz-b2b-medical-erp

git checkout project-b22
```

---

# Step 3 — Install Terraform

```bash
sudo apt update -y

sudo apt install -y gnupg software-properties-common curl unzip
```

Download Terraform:

```bash
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraformon
```

---

# Step 4 — Deploy AWS Infrastructure Using Terraform

Initialize Terraform:

```bash
terraform init
```

Deploy Infrastructure:

```bash
terraform apply -auto-approve
```

This creates:

* VPC
* Subnets
* EKS Cluster
* Node Groups
* Security Groups
* IAM Roles

---

# Step 5 — Configure kubectl Access

```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name EKS_CLOUD
```

Verify:

```bash
kubectl get nodes
```

Expected:

* Worker nodes should appear

---

# Step 6 — Install AWS CLI

```bash
sudo apt install unzip -y

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

unzip awscliv2.zip

sudo ./aws/install
```

Verify:

```bash
aws --version
```

---

# Step 7 — Install kubectl and eksctl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client

```
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```
---

# Step 8 — Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify:

```bash
helm version
```

---

# Step 9 — Configure MongoDB Atlas

## Create MongoDB Atlas Cluster

1. Open MongoDB Atlas
2. Create Cluster
3. Create database user
4. Allow network access: `0.0.0.0/0`

## Create Databases

Inside Browse Collections create:

* users_db
* products_db
* orders_db

---

# Step 10 — Update MongoDB URLs

## Update Order Service

```bash
nano order-service/src/main/resources/application.yml
```

Update:

```yaml
${MONGODB_URI:mongodb+srv://admin:redhat123@medpharma-cluster.04epudg.mongodb.net/orders_db?appName=medpharma-cluster}
```

## Update Product Service

```bash
nano product-service/src/main/resources/application.yml
```

Update:

```yaml
${MONGODB_URI:mongodb+srv://admin:redhat123@medpharma-cluster.04epudg.mongodb.net/products_db?appName=medpharma-cluster}
```

## Update User Service

```bash
nano user-service/src/main/resources/application.yml
```

Update:

```yaml
${MONGODB_URI:mongodb+srv://admin:redhat123@medpharma-cluster.04epudg.mongodb.net/users_db?appName=medpharma-cluster}
```

---

# Step 11 — Install Docker and Build Docker Images

```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

## Build Order Service

```bash
cd order-service

docker build -t order-service:latest .
```

---

## Build Product Service

```bash
cd product-service

docker build -t product-service:latest .
```

---

## Build User Service

```bash
cd user-service

docker build -t user-service:latest .
```

---

# Step 12 — Push Images to ECR

Login to ECR:

```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com
```

Tag Images:

```bash
docker tag user-service:latest <ECR_REPO>/user-service:latest

docker tag product-service:latest <ECR_REPO>/product-service:latest

docker tag order-service:latest <ECR_REPO>/order-service:latest
```

Push Images:

```bash
docker push <ECR_REPO>/user-service:latest

docker push <ECR_REPO>/product-service:latest

docker push <ECR_REPO>/order-service:latest
```

---

# Step 13 — Create Kubernetes namespace and Secret

## Create Namespace

```bash
kubectl apply -f k8s/namespace/
```

---

```bash
kubectl create secret generic app-secrets \
  -n med-erp \
  --from-literal=MONGODB_URI_USER="mongodb+srv://admin:redhat123@medpharma-cluster.04epudg.mongodb.net/user_db?appName=medpharma-cluster" \
  --from-literal=MONGODB_URI_PRODUCT="mongodb+srv://admin:redhat123@medpharma-cluster.04epudg.mongodb.net/product_db?appName=medpharma-cluster" \
  --from-literal=MONGODB_URI_ORDER="mongodb+srv://admin:redhat123@medpharma-cluster.04epudg.mongodb.net/order_db?appName=medpharma-cluster" \
  --from-literal=JWT_SECRET="404E635266556A586E3272357538782F413F4428472B4B6250645367566B5970" \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

# Step 14 — Configure Route53

1. Create Hosted Zone
2. Add your domain
3. Copy Route53 Nameservers
4. Replace nameservers in Hostinger

---

# Step 15 — Create and configure ACM SSL Certificate in both regions

1. Open AWS Certificate Manager
2. Request Public Certificate
3. Add:
   * *.matrixofleadership.online
4. Use DNS validation
5. Click Create Records in Route53

---

# Step 16 — Install npm Deploy Frontend to S3

```bash
apt install npm -y
```

## Build Frontend

```bash
cd frontend

export VITE_USER_SERVICE_URL="https://api.matrixofleadership.online/api/v1"

export VITE_PRODUCT_SERVICE_URL="https://api.matrixofleadership.online/api/v1"

export VITE_ORDER_SERVICE_URL="https://api.matrixofleadership.online/api/v1"

npm install

npm run build
```

---

# Step 17 — Upload Frontend to S3

```bash
aws s3 sync dist/ s3://matrixofleadership-frontend --delete
```
1. Make all files public using ACLs
2. Enable static website hosting
---

# Step 18 — Configure CloudFront

1. Create Distribution
2. Origin Type → Amazon S3
3. Select S3 Bucket
4. Disable WAF
5. General → Alternate domain name → matrixofleadership.online → Select SSL Certificate

---

# Step 19 — Install AWS Load Balancer Controller

## Create OIDC Provider

Open:

* EKS Console
* Cluster Overview
* Copy OIDC URL

## Create Identity Provider in IAM.

Configuration:
* Provider Type →	OpenID Connect
* Provider URL →	https://oidc.eks.ap-south-1.amazonaws.com/id/AE50ABFA426F6BE267BC7073E88F78F9
* Audience →	sts.amazonaws.com

---

## Download IAM and Create IAM Policy

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```
Create IAM Role

## AWS Console Steps

1. Open IAM Console
2. Go to `Roles`
3. Click `Create Role`

### Trusted Entity

| Setting             | Value                                                                     |
| ------------------- | ------------------------------------------------------------------------- |
| Trusted Entity Type | Web Identity                                                              |
| Identity Provider   | oidc.eks.ap-south-1.amazonaws.com/id/238D7150432AC9BA9325B113C10F5FE2 |
| Audience            | sts.amazonaws.com                                                         |

### Attach Permission Policy

Select:

```text
AWSLoadBalancerControllerIAMPolicy
ElasticLoadBalancingFullAccess
AmazonEKSLoadBalancingPolicy
```

### Role Name

```text
AmazonEKSLoadBalancerControllerRole1
```

Click `Create Role`.

---

# Update Trust Relationship

Open role:

```text
AmazonEKSLoadBalancerControllerRole1
```

Go to:

* Trust Relationships
* Edit Trust Policy

Replace policy with:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::344807217216:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/238D7150432AC9BA9325B113C10F5FE2"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-south-1.amazonaws.com/id/238D7150432AC9BA9325B113C10F5FE2:aud": "sts.amazonaws.com",
          "oidc.eks.ap-south-1.amazonaws.com/id/238D7150432AC9BA9325B113C10F5FE2:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
        }
      }
    }
  ]
}
```

Click `Update Policy`.

---

## Create Kubernetes Service Account

Create file:

```bash
nano aws-load-balancer-controller-service-account.yaml
```

Content: update your role ARN

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-load-balancer-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::338394180817:role/AmazonEKSLoadBalancerControllerRole1
```

Apply:

```bash
kubectl apply -f aws-load-balancer-controller-service-account.yaml
```

Verify:

```bash
kubectl get sa -n kube-system
```

Expected:

* `aws-load-balancer-controller` service account should exist.

---

---
# Step 20 — Install AWS Load Balancer Controller via Helm

```bash
helm repo add eks https://aws.github.io/eks-charts

helm repo update
```

## Get VPC ID

Run:

```bash
aws eks describe-cluster \
  --name EKS_CLOUD \
  --region ap-south-1 \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text
```

Example Output:

```text
vpc-02032a03540bf603c
```

Copy the VPC ID.

---
Install Controller:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=EKS_CLOUD \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-south-1 \
  --set vpcId=vpc-02032a03540bf603c
```

## Verify Installation

Check deployment:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

Expected:

```text
2/2 READY
```

Check pods:

```bash
kubectl get pods -n kube-system
```

Expected:

* aws-load-balancer-controller pods should be running.

---


# Step 21 — Update Deployements files and Deploy Kubernetes Resources

## Update Deployment files with your registry URI
1. Order-service
```bash
nano k8s/deployments/order-service-deployment.yaml
```
```text
public.ecr.aws/s8f5a7n4/order-service
```
2. Product-service

```bash
nano k8s/deployments/product-service-deployment.yaml
```
```text
public.ecr.aws/s8f5a7n4/product-service
```
3. User-service

```bash
nano k8s/deployments/user-service-deployment.yaml
```
```text
public.ecr.aws/s8f5a7n4/user-service
```

---
## Apply ConfigMaps

```bash
kubectl apply -f k8s/configmaps/
```

---

## Apply Deployments

```bash
kubectl apply -f k8s/deployments/
```

---

## Apply Services

```bash
kubectl apply -f k8s/services/
```

---

## Apply HPA

```bash
kubectl apply -f k8s/hpa/
```

---

## Apply Ingress Update ingress manifest file

```bash
nano k8s/ingress/ingress.yaml
```
Update:

* ACM ARN (Make sure to use ap-south-1 arn of ACM)
* Security Group
* Domain name
* 
```text
alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-south-1:338394180817:certificate/882b32a9-01ea-4e6e-b9b8-f1f1fe12704d # use ap-south-1
alb.ingress.kubernetes.io/security-groups: sg-03c9795c5f71bc17f
- host: api.matrixofleadership.online
```

Then:

```bash
kubectl apply -f k8s/ingress/
```

---

# Step 22 — Verify Deployment

Check Pods:

```bash
kubectl get pods -n med-erp
```

Check Services:

```bash
kubectl get svc -n med-erp
```

Check Ingress:

```bash
kubectl get ingress -n med-erp
```

---

# Step 23 — Final Verification

Frontend:

```text
https://matrixofleadership.online
```

Backend:

```text
https://api.matrixofleadership.online
```

Verify:

* Login
* Product APIs
* Order APIs
* HTTPS
* ALB Health
* MongoDB Connectivity
