# 🌍 Wanderlust - Complete DevOps CI/CD Pipeline

> A production-grade 3-tier web application deployed on AWS EKS with comprehensive DevOps automation

![Architecture](https://img.shields.io/badge/Architecture-3--Tier-blue)
![CI/CD](https://img.shields.io/badge/CI%2FCD-Jenkins%20%2B%20ArgoCD-green)
![Cloud](https://img.shields.io/badge/Cloud-AWS%20EKS-orange)
![Monitoring](https://img.shields.io/badge/Monitoring-Prometheus%20%2B%20Grafana-red)

## 📋 Table of Contents

- [Overview](#-overview)
- [Architecture](#-architecture)
- [Tech Stack](#-tech-stack)
- [Prerequisites](#-prerequisites)
- [Infrastructure Setup](#-infrastructure-setup)
- [Installation Guide](#-installation-guide)
- [CI/CD Pipeline](#-cicd-pipeline)
- [Monitoring](#-monitoring)
- [Cleanup](#-cleanup)

## 🎯 Overview

Wanderlust is a full-stack travel blog application demonstrating enterprise-level DevOps practices. This project showcases:

- **3-Tier Architecture**: React frontend, Node.js backend, MongoDB database
- **Containerization**: Docker-based microservices
- **Orchestration**: Kubernetes (AWS EKS)
- **CI/CD**: Automated pipelines with Jenkins and ArgoCD
- **Security**: OWASP dependency checks, Trivy scanning, SonarQube analysis
- **Monitoring**: Prometheus and Grafana stack
- **Caching**: Redis implementation for performance optimization

## 🏗 Architecture

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   Frontend  │─────▶│   Backend   │─────▶│   MongoDB   │
│   (React)   │      │  (Node.js)  │      │  (Database) │
└─────────────┘      └─────────────┘      └─────────────┘
                            │
                            ▼
                     ┌─────────────┐
                     │    Redis    │
                     │  (Caching)  │
                     └─────────────┘
```

**Deployment Flow:**
```
GitHub → Jenkins (CI) → Docker Hub → ArgoCD (CD) → AWS EKS
```

## 🛠 Tech Stack

### Application Layer
- **Frontend**: React.js
- **Backend**: Node.js
- **Database**: MongoDB
- **Caching**: Redis

### DevOps Tools
- **Version Control**: GitHub
- **Containerization**: Docker
- **CI**: Jenkins
- **Security Scanning**: OWASP Dependency Check, Trivy, SonarQube
- **CD**: ArgoCD
- **Container Orchestration**: Kubernetes (AWS EKS)
- **Monitoring**: Prometheus, Grafana (via Helm)
- **Infrastructure**: AWS (EC2, EKS, IAM)

## ✅ Prerequisites

- AWS Account with appropriate permissions
- GitHub Account
- Docker Hub Account
- Basic knowledge of Linux, Docker, Kubernetes, and CI/CD concepts

## 🚀 Infrastructure Setup

### 1. AWS IAM Configuration

Create an IAM user with administrator access:

```bash
# User: terraform-user (Administrator Access)
# Generate access keys for programmatic access
```

Create an IAM role for EC2:

```bash
# Role Name: ec2-wanderlust-role
# Trusted Entity: EC2
# Permissions: AdministratorAccess
```

### 2. EC2 Master Instance Setup

**Instance Specifications:**
- **Type**: t2.large (2 vCPU, 8GB RAM)
- **Storage**: 29GB
- **Region**: us-east-2
- **Role**: Attach `ec2-wanderlust-role`

**Security Group Inbound Rules:**

| Port Range | Protocol | Description |
|------------|----------|-------------|
| 22 | TCP | SSH |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 3000-10000 | TCP | Application Ports |
| 6379 | TCP | Redis |
| 6443 | TCP | Kubernetes API |
| 8080 | TCP | Jenkins |
| 9000 | TCP | SonarQube |
| 30000-32767 | TCP | Kubernetes NodePorts |

## 📦 Installation Guide

### Step 1: Initial Setup

SSH into your EC2 instance and update packages:

```bash
sudo apt-get update -y
```

### Step 2: Install Docker

```bash
sudo apt-get install docker.io docker-compose-v2 -y
sudo chmod 777 /var/run/docker.sock

# Alternative: Add user to docker group
sudo usermod -aG docker $USER && newgrp docker
```

### Step 3: Install Jenkins

```bash
sudo apt install fontconfig openjdk-21-jre -y

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update -y
sudo apt install jenkins -y
```

Access Jenkins at: `http://<EC2-PUBLIC-IP>:8080`

Get initial admin password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Step 4: Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install

# Configure AWS CLI
aws configure
# Enter: Access Key ID, Secret Access Key, Region (us-east-2), Output format (json)
```

### Step 5: Install kubectl

```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```

### Step 6: Install eksctl

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

### Step 7: Create EKS Cluster

```bash
eksctl create cluster --name=wanderlust \
  --region=us-east-2 \
  --version=1.30 \
  --without-nodegroup
```

⏱️ *This process takes approximately 15-20 minutes*

### Step 8: Install Trivy (Security Scanner)

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update -y
sudo apt-get install trivy -y
```

### Step 9: Install SonarQube

```bash
docker run -itd --name SonarQube-Server -p 9000:9000 sonarqube:lts-community
```

Access SonarQube at: `http://<EC2-PUBLIC-IP>:9000`
- Default credentials: `admin/admin` (change on first login)

### Step 10: Associate IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --region us-east-2 \
  --cluster wanderlust \
  --approve
```

### Step 11: Create EKS Node Group

First, create an EC2 key pair named `eks-nodegroup-key` in us-east-2 region via AWS Console.

```bash
eksctl create nodegroup --cluster=wanderlust \
  --region=us-east-2 \
  --name=wanderlust \
  --node-type=t2.large \
  --nodes=2 \
  --nodes-min=2 \
  --nodes-max=2 \
  --node-volume-size=29 \
  --ssh-access \
  --ssh-public-key=eks-nodegroup-key
```

Verify cluster:
```bash
kubectl get nodes
```

### Step 12: Install and Configure ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl get pods -n argocd -w

# Install ArgoCD CLI
sudo curl --silent --location -o /usr/local/bin/argocd \
  https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64

sudo chmod +x /usr/local/bin/argocd

# Change service type to NodePort
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

# Get service details
kubectl get svc -n argocd
```

Access ArgoCD UI at: `https://<WORKER-NODE-IP>:<ARGOCD-NODEPORT>`

Get initial admin password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

**Note**: Update security groups for worker nodes to allow ArgoCD NodePort traffic.

### Step 13: Connect EKS Cluster to ArgoCD

```bash
# Login to ArgoCD
argocd login <ARGOCD-URL> --username admin

# Get cluster context
kubectl config get-contexts

# Add cluster to ArgoCD
argocd cluster add <CLUSTER-CONTEXT-NAME> --name wanderlust-eks-cluster
```

## 🔧 Jenkins Configuration

### Install Required Plugins

Navigate to: `Manage Jenkins → Plugins → Available Plugins`

Install:
- OWASP Dependency-Check
- SonarQube Scanner
- Docker
- Pipeline: Stage View
- Blue Ocean

**Restart Jenkins after installation**

### Configure OWASP

1. Go to `Manage Jenkins → Tools`
2. Add Dependency-Check installation
   - Name: `OWASP`
   - Install automatically from GitHub

### Integrate SonarQube

**Create SonarQube Token:**
1. SonarQube → Administration → Security → Users
2. Generate token with 30-day expiry

**Add to Jenkins:**
1. `Manage Jenkins → Credentials → Add Credentials`
   - Kind: Secret text
   - Secret: [SonarQube token]
   - ID: `Sonar`
   - Description: `Sonar Token`

2. `Manage Jenkins → Tools → SonarQube Scanner`
   - Name: `Sonar`
   - Install automatically

3. `Manage Jenkins → System → SonarQube Servers`
   - Name: `Sonar`
   - Server URL: `http://<EC2-IP>:9000`
   - Server authentication token: Select `Sonar`

**Configure SonarQube Webhook:**
1. SonarQube → Administration → Configuration → Webhooks
2. URL: `http://<JENKINS-URL>:8080/sonarqube-webhook/`

### Configure GitHub Integration

1. Generate GitHub Personal Access Token with repo permissions
2. `Jenkins → Credentials → Add Credentials`
   - Kind: Username with password
   - Username: [Your GitHub username]
   - Password: [GitHub token]
   - ID: `Github-cred`
   - Description: `GitHub credentials`

### Configure Docker Hub

1. Generate Docker Hub access token
2. `Jenkins → Credentials → Add Credentials`
   - Kind: Username with password
   - Username: [Docker Hub username]
   - Password: [Docker Hub token]
   - ID: `docker`
   - Description: `DockerHub`

### Configure Email Notifications

1. Generate Gmail App Password
2. `Manage Jenkins → Credentials → Add Credentials`
   - Kind: Username with password
   - Username: [Your Gmail]
   - Password: [App password]
   - ID: `Gmail`

3. `Manage Jenkins → System`
   - **Extended E-mail Notification:**
     - SMTP server: `smtp.gmail.com`
     - SMTP Port: `465`
     - Credentials: Select Gmail
     - Use SSL: ✓
   
   - **E-mail Notification:**
     - SMTP server: `smtp.gmail.com`
     - Use SMTP Authentication: ✓
     - Username: [Your Gmail]
     - Password: [App password]
     - Use SSL: ✓
     - SMTP Port: `465`

### Jenkins Shared Library

1. `Manage Jenkins → System → Global Trusted Pipeline Libraries`
2. Add library:
   - Name: `Shared`
   - Default version: `main`
   - Modern SCM: Git
   - Project repository: [Your shared library repo]

## 🔄 CI/CD Pipeline

### CI Pipeline Configuration

1. Create new Jenkins pipeline:
   - Name: `Wanderlust-CI`
   - Type: Pipeline
   - Description: `CI pipeline for Wanderlust application`

2. Configure:
   - Days to keep builds: `1`
   - Max # of builds: `2`
   - GitHub project URL: [Your repo URL]

3. Pipeline script: Use Jenkinsfile from repository root

**Pipeline Stages:**
- Checkout code from GitHub
- OWASP dependency scan
- Trivy filesystem scan
- SonarQube analysis
- Build Docker images (Frontend & Backend)
- Trivy image scan
- Push to Docker Hub
- Update deployment manifests

### CD Pipeline Configuration

1. Create new Jenkins pipeline:
   - Name: `Wanderlust-CD`
   - Type: Pipeline
   - Description: `CD pipeline for Wanderlust application`

2. Pipeline script: Use Jenkinsfile from GitOps folder

**Pipeline Stages:**
- Pull latest deployment configurations
- Trigger ArgoCD sync
- Deploy to Kubernetes cluster

### ArgoCD Application Setup

1. Create application in ArgoCD UI:
   - **Application Name**: `wanderlust`
   - **Project**: `default`
   - **Sync Policy**: Automatic
     - ✓ Prune Resources
     - ✓ Self Heal
     - ✓ Auto-create Namespace
   - **Source**:
     - Repository URL: [Your GitOps repo]
     - Revision: `main`
     - Path: `kubernetes`
   - **Destination**:
     - Cluster URL: [Select your cluster]
     - Namespace: `wanderlust`

2. Application endpoints:
   - **Frontend**: `http://<WORKER-NODE-IP>:31000`
   - **Backend**: `http://<WORKER-NODE-IP>:31100`

**Note**: Enable these NodePorts in worker node security groups.

## 📊 Monitoring

### Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

# Add Helm stable charts
helm repo add stable https://charts.helm.sh/stable
```

### Deploy Prometheus & Grafana

```bash
# Add Prometheus community Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Create namespace
kubectl create namespace prometheus

# Install Prometheus stack
helm install stable prometheus-community/kube-prometheus-stack -n prometheus

# Verify installation
kubectl get pods -n prometheus
kubectl get svc -n prometheus
```

### Expose Services via NodePort

**Prometheus:**
```bash
kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
# Change type from ClusterIP to NodePort
```

**Grafana:**
```bash
kubectl edit svc stable-grafana -n prometheus
# Change type from ClusterIP to NodePort
```

Verify services:
```bash
kubectl get svc -n prometheus
```

### Access Grafana

URL: `http://<WORKER-NODE-IP>:<GRAFANA-NODEPORT>`

**Credentials:**
- Username: `admin`
- Password: Get from command below

```bash
kubectl get secret --namespace prometheus stable-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

**Pre-configured Dashboards:**
- Kubernetes cluster monitoring
- Node exporter metrics
- Pod resource usage
- Application performance metrics

## 🧹 Cleanup

To delete all resources and avoid AWS charges:

```bash
# Delete EKS cluster and associated resources
eksctl delete cluster --name=wanderlust --region=us-east-2
```

This will remove:
- EKS cluster
- Node groups
- Associated VPC, subnets, and security groups
- Load balancers
- EC2 instances (worker nodes)

**Manual Cleanup:**
- Terminate the master EC2 instance
- Delete IAM roles and users created for the project
- Remove Docker images from Docker Hub (if desired)
- Delete GitHub repositories (if desired)

## 🎓 Learning Outcomes

This project demonstrates:

✅ Infrastructure as Code principles  
✅ Containerization best practices  
✅ Kubernetes orchestration  
✅ CI/CD pipeline implementation  
✅ Security scanning and quality gates  
✅ GitOps deployment methodology  
✅ Cloud-native monitoring solutions  
✅ Production-grade AWS architecture  

## 📝 Notes

- All sensitive credentials (passwords, tokens, API keys) should be stored securely using secrets management solutions
- Regularly update security scanning tools and dependencies
- Monitor AWS costs and set up billing alerts
- Implement backup strategies for production data
- Follow security best practices for production deployments

## 🤝 Contributing

Contributions are welcome! Please feel free to submit pull requests or open issues for improvements.

## 📄 License

This project is open source and available under the MIT License.

---

**The Wanderlust Website was created by:**: [https://github.com/krishnaacharyaa/wanderlust](https://github.com/krishnaacharyaa/wanderlust)

Made for learning DevOps practices by deploying 3-tier application to EKS.
