# Final Project Deployment Guide

# Phase 1

This phase demonstrates manual deployment of the OpenTelemetry application first using Docker Compose on EC2, then on EKS using Kubernetes manifests and `kubectl`.

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

# Phase 2

This phase simplifies application deployment using Helm charts, including upgrade and rollback features for efficient version management.

## Steps to Deploy

### 1. Launch Networking Infrastructure
- Deploy the **VPC** and **subnets** using the `VPC.yaml` file:
  ```bash
  Launch "VPC.yaml" in AWS CloudFormation.
  ```

### 2. Launch Kubernetes Client Instance
- Deploy the **Kubernetes client EC2 instance** using the `helm.yaml` file:
  ```bash
  Launch "helm.yaml" in AWS CloudFormation.
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

### 5. Deploy the Application Using Helm
- After the cluster is ready, inside the EC2 instance terminal, run:
  ```bash
  helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
  helm repo update
  helm install otel-demo open-telemetry/opentelemetry-demo -n helm-deployment --create-namespace
  ```
- This command will deploy all necessary Kubernetes resources using the Helm chart.

### 6. Upgrade and Rollback using helm (Optional)

#### Step 1: Upgrade the Deployment
- If you'd like to test Helm’s upgrade capability, apply the modified values file:
  ```bash
  helm upgrade otel-demo open-telemetry/opentelemetry-demo \
  -f otel-demo-values.yaml \
  -n helm-deployment
  ```

#### Step 2: Roll Back to Previous Version
- You can roll back to the original deployment:
  ```bash
  helm rollback otel-demo 1 -n helm-deployment
  ```
  
### 7. Access the Application

#### Step 1: Forward Port Inside EC2 Instance
- Inside the Kubernetes client EC2 instance, run:
  ```bash
  kubectl port-forward -n helm-deployment svc/frontend-proxy 8080:8080
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
# Phase 3
- This phase will give you steps to configure prometheus and alertmanager for monitoring your system
- After you have finished the Phase 2. Please follow these steps to launch your alertmanager
#### Step 1: Inside the instance of your EKS cluster run these commands to add the prometheus repository and install the kube state metrics
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install kube-state-metrics prometheus-community/kube-state-metrics \
  --namespace kube-system
```
#### Step 2: Verify the installation
```bash
kubectl get pods -n kube-system | grep kube-state-metrics
```
#### Step 3: Inside the same instance you configure SNS topic and subscribe your email
```bash
aws sns create-topic --name kube-alerts
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:<YOUR_ACCOUNT_ID>:kube-alerts \
  --protocol email \
  --notification-endpoint you@example.com
```
##### Note: Make sure to subscribe your email which you will be receiving
#### Step 4: Create the alertmanager-config.yaml, alertmanager-deployment.yaml, alertmanager-service.yaml file and paste the contents from alertmanager folder. After which deploy them using below commands in same order.
#### Note: Make sure your namespace is same as the one you have used in hel deployment
```bash
kubectl apply -f alertmanager-config.yaml
kubectl apply -f alertmanager-deployment.yaml
kubectl apply -f alertmanager-service.yaml
```
#### Step 5: Grant IAM permission for alertmanager to publish to SNS. Add the below policy to the NodegroupInstance(eg: eksctl-otel-demo-nodegroup-...-NodeInstanceRole-xxx)
```bash
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "SNS:Publish",
      "Resource": "arn:aws:sns:us-east-1:<YOUR_ACCOUNT_ID>:kube-alerts"
    }
  ]
}
```
#### Step 6: Simulate to evaluate your policy. Makesure "EvalDecision": "allowed"
```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:role/<NodeInstanceRole> \
  --action-names sns:Publish \
  --resource-arns arn:aws:sns:us-east-1:<YOUR_ACCOUNT_ID>:kube-alerts
```
#### Step 7: Add alerting rules and data to ConfigMap: Check you pod server for prometheus and alertmanager to add the configmap 
```bash
kubectl get svc -n helm-deployment
```
- You can check your configmap using below.
```bash
kubectl edit cm <your-prometheus-server> -n helm-deployment
```
- Add these contents under the data.prometheus.yaml
```bash
scrape_configs:
  - job_name: 'kube-state-metrics'
    static_configs:
      - targets: ['kube-state-metrics.kube-system.svc.cluster.local:8080']
    relabel_configs:
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
```
- Add this alerting rule for the configmap
```bash
rule_files:
  - /etc/prometheus/alerting_rules.yml

# this creates a new file alerting_rules.yml in the same ConfigMap:

data:
  alerting_rules.yml: |
    groups:
    - name: pod-restart-alerts
      rules:
      - alert: HighPodRestart
        expr: increase(kube_pod_container_status_restarts_total{namespace="otel-demo"}[5m]) > 3
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} has high restarts"
          description: "Restart count over 3 in last 5m: {{ $value }}"
```
- Also check if these content exists. If not add them
```bash
alerting:
    alertmanagers:
    - static_configs:
      - targets:
        - 'alertmanager.helm-deployment.svc.cluster.local:9093'
```
#### Note : There might be errors if your naming is different. So double check the pod server names
#### Step 8: Apply the changes and rooll-out prometheus restart
```bash
kubectl rollout restart deployment prometheus -n helm-deployment
```
#### Step 9: Verify if everything is in order by port forwarding. You should be able to see the alerting rule you have configured
```bash
kubectl port-forward svc/prometheus -n helm-deployment 9090:9090
kubectl port-forward svc/alertmanager -n helm-deployment 9093:9093
```
#### Step 10: Do a crash test to check if alerting rule is able to fire and send alerts to your email.
- Create a file test-crash.yaml and add these contents
```bash
apiVersion: v1
kind: Pod
metadata:
  name: test-crash
  namespace: helm-deployment
spec:
  restartPolicy: Always
  containers:
  - name: crash
    image: busybox
    command: ["sh", "-c", "exit 1"]
```
- Apply the changes
```bash
kubectl apply -f test-crash.yaml
```
- Watch the pod restarts triggered by crash test
```bash
kubectl get pod test-crash -n helm-deployment --watch
```
- Wait for few minutes and in your prometheus and alertmanager UI you should see
- Prometheus → fires the HighPodRestart alert
- Alertmanager → groups and forwards to the sns-alerts receiver
- You will get an email to your inbox







