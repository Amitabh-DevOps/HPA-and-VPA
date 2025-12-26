# HPA & VPA on AWS EC2 (Ubuntu) using kind

This guide assumes:

* EC2 instance: **Ubuntu 22.04**
* Instance type: **t3.medium or higher** (VPA needs some headroom)
* User: `ubuntu`

---

## 🔹 Step 0: Update the System

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 🔹 Step 1: Install Docker (Required for kind)

```bash
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
```

Add user to docker group:

```bash
sudo usermod -aG docker ubuntu
newgrp docker
```

Verify:

```bash
docker version
```

---

## 🔹 Step 2: Install kubectl

```bash
curl -LO https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

Verify:

```bash
kubectl version --client
```

---

## 🔹 Step 3: Install kind

```bash
curl -Lo kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x kind
sudo mv kind /usr/local/bin/
```

Verify:

```bash
kind version
```

---

## 🔹 Step 4: Create kind Cluster (Recommended Config)

Create cluster:

```bash
kind create cluster --name hpa-vpa-demo --config kind-config.yaml
```

Verify:

```bash
kubectl get nodes
```

---

## 🔹 Step 5: Install Metrics Server (Mandatory for HPA)

### Apply metrics-server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Patch for kind + EC2

```bash
kubectl patch deployment metrics-server -n kube-system \
--type='json' \
-p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--kubelet-insecure-tls"
  }
]'
```

Verify:

```bash
kubectl top nodes
kubectl top pods
```

---

# 🟦 PART 1: HPA (Horizontal Pod Autoscaler)

---

## 1️⃣ HPA Deployment (`hpa-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-nginx
  template:
    metadata:
      labels:
        app: hpa-nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
```

```bash
kubectl apply -f hpa-deployment.yaml
```

---

## 2️⃣ Service (`hpa-service.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hpa-nginx
spec:
  selector:
    app: hpa-nginx
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f hpa-service.yaml
```

---

## 3️⃣ HPA Manifest (`hpa.yaml`)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-nginx
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-nginx
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

```bash
kubectl apply -f hpa.yaml
```

---

## 4️⃣ Generate Load (Trigger HPA)

```bash
kubectl run load-generator \
--image=busybox \
--restart=Never \
-- /bin/sh -c "while true; do wget -q -O- http://hpa-nginx; done"
```

Check scaling:

```bash
kubectl get hpa
kubectl get pods -w
```

---

# 🟩 PART 2: VPA (Vertical Pod Autoscaler)

---

## 🔹 Step 6: Install VPA Components

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
```

Verify:

```bash
kubectl get pods -n kube-system | grep vpa
```

---

## 1️⃣ VPA Deployment (`vpa-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vpa-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vpa-nginx
  template:
    metadata:
      labels:
        app: vpa-nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

```bash
kubectl apply -f vpa-deployment.yaml
```

---

## 2️⃣ VPA Manifest (`vpa.yaml`)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-nginx
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vpa-nginx
  updatePolicy:
    updateMode: Auto
```

```bash
kubectl apply -f vpa.yaml
```

---

## 3️⃣ View VPA Recommendations

```bash
kubectl describe vpa vpa-nginx
```

You will see:

* Recommended CPU
* Recommended memory
* Current usage

⚠️ VPA **restarts pods** to apply new resources.
