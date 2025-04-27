# Final Project Deployment Guide

## Steps to Deploy

### 1. Launch Networking Infrastructure
- Deploy the **VPC** and **subnets** using the `VPC.yaml` file:
  ```bash
  Launch "VPC.yaml" in AWS CloudFormation.
  ```

### 2. Launch Kubernetes Client Instance
- Deploy the **Kubernetes client EC2 instance** using the `Kubernetes.yaml` file:
  ```bash
  Launch "kubernetes.yaml" in AWS CloudFormation.
  ```

### 3. Connect to the Kubernetes Client
- SSH into the EC2 instance:
  ```bash
  ssh -i <your-key>.pem ubuntu@<your-ec2-public-ip>
  ```

### 4. Create the EKS Cluster
- Inside the terminal of the EC2 instance, run:
  ```bash
  eksctl create cluster \
    --name group20-final-eks-cluster \
    --region us-east-1 \
    --vpc-public-subnets subnet-0b86e2ce2f52cb797,subnet-0c876f78820ff8b4c,subnet-0c7e88b436b1441e6 \
    --nodegroup-name group20-final-workers \
    --node-type t3.large \
    --nodes 2 \
    --nodes-min 2 \
    --nodes-max 4 \
    --managed
  ```
- Wait for the EKS control plane and worker nodes to be created (this can take several minutes).

### 5. Deploy Kubernetes Resources
- After the cluster is ready, inside the EC2 instance terminal, run:
  ```bash
  python3 kube.py
  ```
- This script will deploy all necessary Kubernetes resources.

### 6. Access the Application

#### Step 1: Forward Port Inside EC2 Instance
- Inside the Kubernetes client EC2 instance, run:
  ```bash
  kubectl port-forward -n otel-demo svc/frontend-proxy 8080:8080
  ```

#### Step 2: Tunnel Port from Your Local PC
- From **your PC's terminal**, run this SSH tunnel command:
  ```bash
  ssh -i <your-key>.pem -L 8080:localhost:8080 ubuntu@<your-ec2-public-ip>
  ```

#### Step 3: Open in Browser
- Now, on your PC, open your browser and navigate to:
  ```
  http://localhost:8080
  ```


