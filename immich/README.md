# Immich Kubernetes Deployment Guide

This guide explains how to deploy Immich on a Kubernetes cluster, both for local development and on a remote server with an automated workflow.

---

## **1. Prerequisites**

Ensure you have the following installed:

- **kubectl** (to interact with Kubernetes)
- **Helm** (to manage Helm charts)
- **Minikube** (for local Kubernetes cluster, optional)
- **Ingress Controller** (e.g., Nginx for local testing)
- **A Kubernetes cluster** (for remote deployment)
- **A domain name** (for remote deployment, optional)

---

## **2. Local Deployment**

### **Step 1: Start a Kubernetes Cluster (Minikube)**

```sh
minikube start
```

### **Step 2: Enable Ingress Controller (For Minikube)**

```sh
minikube addons enable ingress
```

### **Step 3: Create Namespace**

```sh
kubectl create namespace immich
```

### **Step 4: Apply Persistent Volume Claim (PVC)**

```sh
kubectl apply -f 01-pvc.yaml
```

### **Step 5: Deploy PostgreSQL with pgvector Support**

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-postgres bitnami/postgresql -f 02-postgres-values.yaml -n immich
```

### **Step 6: Deploy Immich Server**

```sh
helm repo add bjw-s https://bjw-s.github.io/helm-charts/
helm install immich bjw-s/common -f 03-immich-values.yaml -n immich
```

### **Step 7: Configure Ingress for Local Access**

Edit `/etc/hosts` and add:

```
127.0.0.1 immich.local
```

Access Immich via `http://immich.local` in your browser.

---

## **3. Remote Deployment Workflow (CI/CD)**

### **Step 1: Configure Your CI/CD Pipeline**

Create a GitHub Actions workflow `.github/workflows/deploy.yaml`:

```yaml
name: Deploy Immich to Kubernetes

on:
  push:
    paths:
      - 'k8s/**'
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3

      - name: Set up Kubernetes CLI
        uses: azure/setup-kubectl@v3
        with:
          version: 'latest'

      - name: Set up Helm
        uses: azure/setup-helm@v3
        with:
          version: 'latest'

      - name: Authenticate with Cluster
        run: echo "${{ secrets.KUBECONFIG }}" | base64 --decode > $HOME/.kube/config

      - name: Apply PVC
        run: kubectl apply -f k8s/01-pvc.yaml -n immich

      - name: Deploy PostgreSQL
        run: |
          helm repo add bitnami https://charts.bitnami.com/bitnami
          helm upgrade --install my-postgres bitnami/postgresql -f k8s/02-postgres-values.yaml -n immich

      - name: Deploy Immich
        run: |
          helm repo add bjw-s https://bjw-s.github.io/helm-charts/
          helm upgrade --install immich bjw-s/common -f k8s/03-immich-values.yaml -n immich
```

### **Step 2: Configure Secrets for CI/CD**

Go to your GitHub repository → **Settings** → **Secrets and variables** → **Actions**, and add:

- **KUBECONFIG**: Base64-encoded Kubernetes config file (`cat ~/.kube/config | base64 -w 0`)

### **Step 3: Push Changes to GitHub**

Any time you update the `k8s/` configuration files and push to the `main` branch, the GitHub workflow will automatically deploy the latest changes.

---

## **4. Verifying Deployment**

Run the following to check deployment status:

```sh
kubectl get pods -n immich
kubectl get services -n immich
kubectl get ingress -n immich
```

If there are any issues, check logs with:

```sh
kubectl logs -l app=immich -n immich
```

---

## **5. Cleaning Up**

To remove the deployment, run:

```sh
helm uninstall immich -n immich
helm uninstall my-postgres -n immich
kubectl delete -f 01-library-pvc.yaml -n immich
```

For Minikube:

```sh
minikube delete
```

---

## **6. Conclusion**

This guide provides a structured approach to deploying Immich locally and on a remote Kubernetes cluster. The CI/CD pipeline ensures automatic deployment when configuration changes, improving maintainability and reliability.
