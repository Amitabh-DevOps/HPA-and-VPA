# Kubernetes HPA and VPA Implementation Guide

This repository provides a comprehensive guide for implementing Horizontal Pod Autoscaling (HPA) and Vertical Pod Autoscaling (VPA) using a Kind cluster. The instructions are optimized for an AWS EC2 instance running Ubuntu 22.04.

## Prerequisites

*   Environment: AWS EC2 Instance (Ubuntu 22.04)
*   Recommended Instance Type: t3.medium or higher (required for VPA component overhead)
*   User: ubuntu

---

## Step 0: System Preparation

Begin by updating the system packages.

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Step 1: Install Docker

Docker is required as the runtime for Kind clusters.

```bash
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
```

Add the current user to the Docker group to run commands without sudo:

```bash
sudo usermod -aG docker ubuntu
newgrp docker
```

Verify the installation:

```bash
docker version
```

---

## Step 2: Install kubectl

Download and install the Kubernetes command-line tool.

```bash
curl -LO https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

Verify the installation:

```bash
kubectl version --client
```

---

## Step 3: Install Kind

Kind allows you to run local Kubernetes clusters using Docker container nodes.

```bash
curl -Lo kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x kind
sudo mv kind /usr/local/bin/
```

Verify the installation:

```bash
kind version
```

---

## Step 4: Cluster Creation

Create a Kind cluster using the provided configuration file.

```bash
kind create cluster --name hpa-vpa-demo --config kind-config.yaml
```

Verify the cluster status:

```bash
kubectl get nodes
```

---

## Step 5: Install Metrics Server

The Metrics Server is mandatory for HPA to function as it collects resource usage data.

### Deploy the Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Configure the Metrics Server for Kind

Edit the deployment to allow insecure TLS (required for Kind's self-signed certificates):

```bash
kubectl -n kube-system edit deployment metrics-server
```

Add the following arguments under `spec.containers.args`:

*   `--kubelet-insecure-tls`
*   `--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname`

Restart the deployment to apply changes:

```bash
kubectl -n kube-system rollout restart deployment metrics-server
```

Verify that the Metrics Server is running and reporting data:

```bash
kubectl get pods -n kube-system
kubectl top nodes
```

---

## Part 1: Horizontal Pod Autoscaler (HPA)

HPA scales the number of pod replicas based on observed CPU or memory utilization.

### 1. Deploy the Application and Service

Apply the deployment and service manifests:

```bash
kubectl apply -f hpa/hpa-deployment.yaml
kubectl apply -f hpa/hpa-service.yaml
```

### 2. Configure HPA

Apply the HPA manifest which targets a 20% CPU utilization:

```bash
kubectl apply -f hpa/hpa.yaml
```

### 3. Generate Load and Observe Scaling

Start a load generator to increase CPU usage:

```bash
kubectl run load-generator --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://hpa-nginx; done"
```

Monitor the HPA status and pod expansion:

```bash
kubectl get hpa
kubectl get pods -w
```

### 4. Cleanup

Remove HPA-related resources:

```bash
kubectl delete -f hpa/
kubectl delete pod load-generator
```

---

## Part 2: Vertical Pod Autoscaler (VPA)

VPA automatically adjusts the CPU and memory reservations for your pods to match actual usage.

### 1. Install VPA Components

VPA requires additional components to be installed from the official autoscaler repository:

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
cd ../../
```

Verify the VPA components in the `kube-system` namespace:

```bash
kubectl get pods -n kube-system | grep vpa
```

### 2. Deploy the Target Application

Apply the deployment manifest designed for VPA testing:

```bash
kubectl apply -f vpa/vpa-deployment.yaml
```

### 3. Configure VPA

Apply the VPA manifest:

```bash
kubectl apply -f vpa/vpa.yaml
```

### 4. Observe Recommendations

Check the current status and recommendations provided by VPA:

```bash
kubectl get vpa
kubectl describe vpa vpa-nginx
```

### 5. Generate Load and Observe Resource Updates

Under the `Recreate` update policy, VPA will restart pods to apply new resource limits when significant changes are needed.

Generate load:

```bash
kubectl run vpa-load \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://vpa-nginx; done"
```

Watch the pods to see them being terminated and recreated with updated resources:

```bash
kubectl get pods -w
```

### 6. Cleanup

Remove VPA-related resources:

```bash
kubectl delete -f vpa/
kubectl delete pod vpa-load
```
