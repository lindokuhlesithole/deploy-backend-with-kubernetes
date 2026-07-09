<h1 align="center">Deploy Backend with Kubernetes on Amazon EKS</h1>

<p align="center">
  <img src="https://img.shields.io/badge/AWS-EKS-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white" alt="AWS EKS" />
  <img src="https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white" alt="Kubernetes" />
  <img src="https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white" alt="Docker" />
  <img src="https://img.shields.io/badge/Amazon%20ECR-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white" alt="Amazon ECR" />
  <img src="https://img.shields.io/badge/Python-Flask-000000?style=for-the-badge&logo=flask&logoColor=white" alt="Flask" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/AWS%20Solutions%20Architect-Professional-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white" alt="AWS SA Pro" />
  <img src="https://img.shields.io/badge/AWS%20Security-Specialty-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=FF9900" alt="AWS Security" />
</p>

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [What I Built (Step by Step)](#what-i-built-step-by-step)
  - [Step 1: Provision the Management Machine](#step-1-provision-the-management-machine)
  - [Step 2: Build the Container Image](#step-2-build-the-container-image)
  - [Step 3: Push to Amazon ECR](#step-3-push-to-amazon-ecr)
  - [Step 4: Create the EKS Cluster](#step-4-create-the-eks-cluster)
  - [Step 5: Write Kubernetes Manifests](#step-5-write-kubernetes-manifests)
  - [Step 6: Deploy and Verify](#step-6-deploy-and-verify)
- [Kubernetes Manifests](#kubernetes-manifests)
- [Key Concepts Demonstrated](#key-concepts-demonstrated)
- [What I Learned](#what-i-learned)
- [Author](#author)

---

## Overview

I deployed a production-ready Flask backend on Amazon EKS using Docker, Kubernetes, and container orchestration. This project covers the full deployment lifecycle from source code to running containers managed by Kubernetes.

**What this project demonstrates:**
- Containerization of a Python Flask application with Docker
- Publishing container images to Amazon ECR with versioned tags
- Provisioning an Amazon EKS cluster using eksctl and CloudFormation
- Writing declarative Kubernetes manifests (Deployment and Service)
- Deploying and managing containerized applications with kubectl
- Exposing applications via NodePort Service for stable network access

---

## Architecture

```
GitHub (Flask Source Code)
    |
    | git clone
    v
EC2 Management Instance
    |
    | docker build
    v
Docker Image (flask-backend:latest)
    |
    | docker push
    v
Amazon ECR (Container Registry)
    |
    | eksctl create cluster
    v
Amazon EKS Cluster
    |
    |-- Control Plane (Managed by AWS)
    |
    |-- Managed Node Group (EC2 Worker Nodes)
    |       |
    |       | kubectl apply
    |       v
    |   3x Pod (Flask App, Port 8080)
    |       |
    |       | Service
    |       v
    |   NodePort:30080
    |
    v
User Access via NodeIP:30080
```

**Architecture Components:**

| Component | Purpose |
|-----------|---------|
| **EC2 Management Instance** | Control machine for building images, running eksctl, and kubectl |
| **Docker** | Packages the Flask app into a portable, reproducible container image |
| **Amazon ECR** | Private container registry for storing and versioning images |
| **Amazon EKS** | Managed Kubernetes service for orchestrating container deployment |
| **eksctl** | CLI tool that simplifies EKS cluster creation via CloudFormation |
| **kubectl** | Kubernetes CLI for managing cluster resources |
| **NodePort Service** | Exposes the application on a static port across all cluster nodes |

---

## What I Built (Step by Step)

### Step 1: Provision the Management Machine

I used an EC2 instance (Amazon Linux 2) as my management machine. This is where all the tooling lives: eksctl for cluster management, kubectl for Kubernetes API communication, AWS CLI for authentication, Docker for building images, and Git for pulling source code.

```bash
# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Install kubectl
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.28.3/bin/linux/amd64/kubectl
chmod +x kubectl && sudo mv kubectl /usr/local/bin

# Configure AWS credentials
aws configure

# Clone the backend code
git clone https://github.com/nextwork-flask-backend.git
cd nextwork-flask-backend
```

### Step 2: Build the Container Image

Kubernetes doesn't run raw source code -- it orchestrates containers spawned from images. I built a Docker image that includes the Python interpreter, all dependencies from `requirements.txt`, and the runtime configuration. This creates a portable blueprint that behaves identically across any environment.

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8080
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "4", "app:app"]
```

```bash
docker build -t flask-backend:latest .
```

### Step 3: Push to Amazon ECR

The EKS cluster needs access to the container image. I pushed it to Amazon ECR -- a private container registry integrated with AWS IAM. Worker nodes authenticate via IAM roles, so there's no manual credential management.

```bash
aws ecr create-repository --repository-name flask-backend --region us-east-1
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <ACCOUNT>.dkr.ecr.us-east-1.amazonaws.com
docker tag flask-backend:latest <ACCOUNT>.dkr.ecr.us-east-1.amazonaws.com/flask-backend:latest
docker push <ACCOUNT>.dkr.ecr.us-east-1.amazonaws.com/flask-backend:latest
```

### Step 4: Create the EKS Cluster

I used eksctl to provision the entire EKS cluster. Behind the scenes, eksctl uses CloudFormation to deploy the control plane, managed node groups, VPC, subnets, security groups, and IAM roles. This step takes 15-20 minutes.

```bash
eksctl create cluster \
    --name flask-backend-cluster \
    --region us-east-1 \
    --node-type t3.medium \
    --nodes 3 \
    --nodes-min 2 \
    --nodes-max 5 \
    --managed \
    --full-ecr-access
```

The `--full-ecr-access` flag is key -- it automatically configures the worker node IAM role with permissions to pull images from ECR. No manual IAM policy attachment needed.

### Step 5: Write Kubernetes Manifests

I wrote two YAML manifests that declaratively define the desired state of my application.

**Deployment Manifest** -- tells Kubernetes what to run:
- Uses the ECR image
- Runs 3 replicas for high availability
- Exposes port 8080
- Includes health checks (liveness and readiness probes)
- Sets resource requests and limits

**Service Manifest** -- tells Kubernetes how to expose it:
- Type: NodePort
- Routes traffic to pods matching the `app: flask-backend` label
- Exposes the service on port 30080 on every node

### Step 6: Deploy and Verify

```bash
# Fix kubectl cluster access (common issue)
aws eks update-kubeconfig --region us-east-1 --name flask-backend-cluster

# Deploy
kubectl apply -f flask-deployment.yaml
kubectl apply -f flask-service.yaml

# Verify
kubectl get deployment flask-backend    # 3/3 replicas ready
kubectl get pods -o wide                # All Running
kubectl get service flask-service       # NodePort 30080

# Test
curl http://<NODE_IP>:30080/
# {"message":"Flask backend is running!","status":"healthy"}
```

---

## Kubernetes Manifests

### flask-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-backend
  labels:
    app: flask-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: flask-backend
  template:
    metadata:
      labels:
        app: flask-backend
    spec:
      containers:
        - name: flask-backend
          image: <AWS_ACCOUNT>.dkr.ecr.us-east-1.amazonaws.com/flask-backend:latest
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
```

### flask-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-service
  labels:
    app: flask-backend
spec:
  type: NodePort
  selector:
    app: flask-backend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30080
```

---

## Key Concepts Demonstrated

### Containerization
Packaging an application with all its dependencies into a single, portable container image. The container is the machine -- it eliminates "works on my machine" problems because the environment is identical everywhere.

### Container Orchestration
Kubernetes automates deployment, scaling, and management of containers. It schedules pods across nodes, restarts failed containers, scales replicas based on demand, and handles service discovery -- all without manual intervention.

### Declarative Infrastructure
Instead of manually provisioning resources, I define the desired state in YAML manifests and let Kubernetes figure out how to achieve it. A pod crashes? Kubernetes creates a replacement automatically. Want 3 replicas instead of 2? Change one number and apply.

### NodePort Service
Pods are ephemeral -- their IP addresses change constantly. A NodePort Service provides a stable network endpoint by exposing the service on a static port (30080) on every worker node, load-balancing traffic across all matching pods.

### IAM Integration for ECR Access
EKS worker nodes pull container images from ECR using IAM roles. No credentials to manage, no secrets to rotate. The `--full-ecr-access` flag on eksctl handles the IAM configuration automatically.

---

## What I Learned

**Containerization as a Foundation:** Building a Docker image and seeing it run identically across different environments demonstrated why containers are the standard for modern application deployment.

**The Power of Declarative Infrastructure:** Writing YAML manifests and running `kubectl apply` changed how I think about infrastructure. I declare what I want, and Kubernetes makes it happen. A pod crashes? It gets replaced. Need more replicas? Change one number.

**Kubernetes Networking:** Understanding why Services are necessary was a key insight. Pods come and go -- their IPs are temporary. A NodePort Service provides a stable endpoint that load-balances across all pods.

**IAM and Security Integration:** Configuring ECR access through IAM roles instead of embedding credentials reinforced why AWS security design is elegant. No credentials to manage -- just roles doing their job.

**Debugging:** I won't lie -- I got stuck. kubectl connection issues, pending pods, image pull errors. Every error taught me something, and by the end I could diagnose issues systematically. That's what production engineering looks like.

---

## Author

**Lindokuhle Sithole** - *Cloud Engineer | Cloud DevOps Engineer | Cloud Security Specialist*

Based in Bremen, Germany. BSc Mathematical Science from the University of the Witwatersrand. 5x AWS Certified (Solutions Architect Professional, Security Specialty, CloudOps Engineer Associate, Solutions Architect Associate, Cloud Practitioner) plus CompTIA Security+.

- **LinkedIn:** [linkedin.com/in/lindokuhle-sithole-bb701b19a](https://www.linkedin.com/in/lindokuhle-sithole-bb701b19a)
- **GitHub:** [github.com/lindokuhlesithole](https://github.com/lindokuhlesithole)
- **Email:** sitholelindokuhle371@gmail.com

---

<p align="center">
  <b>Built by <a href="https://www.linkedin.com/in/lindokuhle-sithole-bb701b19a">Lindokuhle Sithole</a> - Cloud Engineer | Cloud DevOps Engineer | Cloud Security Specialist</b>
</p>
