# Kubernetes & ArgoCD Setup on AWS EC2 (Ubuntu)

Reference video: https://www.youtube.com/watch?v=F7WMRXLUQRM&list=PLy7NrYWoggjzSIlwxeBbcgfAdYoxCIrM2&index=9

This guide covers the installation and configuration of:

* Docker
* Minikube
* kubectl
* ArgoCD

on an Ubuntu EC2 instance.

---

# Prerequisites

## Recommended EC2 Instance

For Minikube and ArgoCD, use:

| Instance Type  | vCPU | RAM   | Recommendation    |
| -------------- | ---- | ----- | ----------------- |
| t3.micro       | 2    | 1 GiB | ❌ Not Recommended |
| t3.small       | 2    | 2 GiB | ⚠️ Minimum        |
| t3.medium      | 2    | 4 GiB | ✅ Recommended     |
| m7i-flex.large | 2    | 8 GiB | ✅ Best            |

Minikube requires at least:

* 2 CPUs
* 2 GB RAM
* 20 GB Storage

---

# Step 1: Connect to EC2

```bash
ssh -i <key.pem> ubuntu@<EC2-PUBLIC-IP>
```

---

# Step 2: Update Packages

```bash
sudo apt update
sudo apt upgrade -y
```

Install required utilities:

```bash
sudo apt install -y curl wget apt-transport-https ca-certificates
```

---

# Step 3: Install Docker

```bash
sudo apt install docker.io -y
```

Start Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Verify:

```bash
docker --version
```

Expected output:

```text
Docker version xx.xx.xx
```

Add current user to Docker group:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Verify:

```bash
docker run hello-world
```

---

# Step 4: Install Minikube

Download latest Minikube:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
```

Install:

```bash
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

Verify:

```bash
minikube version
```

---

# Step 5: Install kubectl

Download kubectl:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

Install:

```bash
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

Verify:

```bash
kubectl version --client
```

---

# Step 6: Start Minikube

Start cluster:

```bash
minikube start --driver=docker
```

If memory errors occur:

```bash
minikube start --driver=docker --memory=4096 --cpus=2
```

Verify:

```bash
kubectl get nodes
```

Expected:

```text
NAME       STATUS   ROLES           AGE
minikube   Ready    control-plane
```

---

# Step 7: Install ArgoCD

Create namespace:

```bash
kubectl create namespace argocd
```

Install ArgoCD:

```bash
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Verify:

```bash
kubectl get pods -n argocd
```

Wait until all pods are:

```text
Running
```

---

# Step 8: Access ArgoCD from EC2

## Method 1 (Recommended)

Port Forward:

```bash
kubectl port-forward svc/argocd-server \
-n argocd 8080:443 \
--address 0.0.0.0
```

Add Security Group Rule:

| Type       | Port |
| ---------- | ---- |
| Custom TCP | 8080 |

Access:

```text
https://<EC2-PUBLIC-IP>:8080
```

---

## Method 2 (NodePort)

Convert service:

```bash
kubectl patch svc argocd-server \
-n argocd \
-p '{"spec":{"type":"NodePort"}}'
```

Check NodePort:

```bash
kubectl get svc -n argocd
```

Example:

```text
argocd-server NodePort 443:32080/TCP
```

Open port 32080 in Security Group.

Access:

```text
https://<EC2-PUBLIC-IP>:32080
```

---

# Step 9: Get ArgoCD Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d
```

Username:

```text
admin
```

Password:

```text
<command output>
```

---

# Step 10: Docker Hub Image Setup

Pull Nana's sample application:

```bash
docker pull nanajanashia/argocd-app:1.2
```

Verify:

```bash
docker images
```

Tag image:

```bash
docker tag nanajanashia/argocd-app:1.2 \
krishna1369/argocd-app:1.2
```

Login:

```bash
docker login
```

Push:

```bash
docker push krishna1369/argocd-app:1.2
```

---

# Step 11: Update Deployment YAML

Use your Docker Hub image:

```yaml
image: krishna1369/argocd-app:1.2
```

Commit and push:

```bash
git add .
git commit -m "Updated image"
git push origin main
```

---

# Step 12: Create ArgoCD Application

application.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd-app
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/krishna1369/argocd-app-config.git
    targetRevision: main
    path: .

  destination:
    server: https://kubernetes.default.svc
    namespace: default

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply:

```bash
kubectl apply -f application.yaml
```

---

# Troubleshooting

## application.yaml not found

```bash
find ~ -name application.yaml
```

## Docker not found

```bash
sudo apt install docker.io -y
```

## Minikube memory error

Upgrade EC2 instance:

```text
t3.medium
or
m7i-flex.large
```

## Check Cluster Status

```bash
minikube status
kubectl get nodes
kubectl get pods -A
```

---

# Useful Commands

Stop Cluster:

```bash
minikube stop
```

Delete Cluster:

```bash
minikube delete
```

Dashboard:

```bash
minikube dashboard
```

Check Pods:

```bash
kubectl get pods -A
```

Check Services:

```bash
kubectl get svc -A
```
