# ☁️ Azure Container Registry (ACR) — Setup & Push Docker Image

## 🧭 Overview

This guide walks you through creating an **Azure Container Registry (ACR)**, logging in from a VM, and pushing a Docker image from your local or Azure VM to ACR.

---

## ⚙️ Prerequisites

* Active **Azure Subscription**
* Access to **Azure Portal** or **Azure CLI**
* A **Docker project** (e.g., [`docker-hello-world`](https://github.com/atulkamble/docker-hello-world))
* Azure VM (Ubuntu recommended)

---

## 🏗️ Step 1 — Create Azure Container Registry (ACR)

1. Go to **Azure Portal** → **All Services** → **Containers** → **Container Registries**
2. Click **Create**
3. Fill in details:

   * **Registry name:** `atulkamble2`
   * **Resource group:** choose or create one
   * **SKU:** `Basic`
   * **Admin user:** **Enable**
4. Click **Review + Create**

---

## 🖥️ Step 2 — Connect to Azure VM

```bash
ssh -i yourkey.pem azureuser@<vm-public-ip>
```

Then install Docker and Git:

```bash
sudo apt update -y
sudo apt install docker.io git -y
sudo systemctl start docker
sudo systemctl enable docker
```

---

## 🧰 Step 3 — Install Azure CLI

Follow Microsoft’s official installation reference:
👉 [Install Azure CLI on Linux (APT)](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-linux?view=azure-cli-latest&pivots=apt)

### Commands:

```bash
sudo apt-get update
sudo apt-get install azure-cli -y
```

Verify installation:

```bash
az version
```

Login to Azure:

```bash
az login
```

---

## 🔑 Step 4 — ACR Login

In Azure Portal → **ACR** → **Access keys**
Enable **Admin user** and note:

* **Username:** atulkamble2
* **Password:** (copy from Access Keys)

Now login to your ACR:

```bash
sudo docker login atulkamble2.azurecr.io
```

Enter your username and password when prompted.

---

## 📦 Step 5 — Clone Project Repository

```bash
git clone https://github.com/atulkamble/docker-hello-world.git
cd docker-hello-world
```

---

## 🧱 Step 6 — Build and Tag Docker Image

Build the image using your ACR path:

```bash
sudo docker build -t atulkamble2.azurecr.io/cloudnautic/hello-world .
```

Verify:

```bash
sudo docker images
```

---

## ☁️ Step 7 — Push Image to ACR

```bash
sudo docker push atulkamble2.azurecr.io/cloudnautic/hello-world
```

Wait until upload completes successfully.

---

## ✅ Step 8 — Verify in Azure Portal

Go to
**Azure Portal → ACR → Repositories → `cloudnautic/hello-world`**

You should see your image listed with its tag.

---

## 🧾 Summary

| Task              | Command / Step                                |
| ----------------- | --------------------------------------------- |
| Create ACR        | Portal → Containers → Container Registry      |
| Install Docker    | `sudo apt install docker.io`                  |
| Install Azure CLI | `sudo apt install azure-cli`                  |
| Login Azure       | `az login`                                    |
| Login ACR         | `docker login atulkamble2.azurecr.io`         |
| Build Image       | `docker build -t <registry>/<repo>/<image> .` |
| Push Image        | `docker push <registry>/<repo>/<image>`       |

---
Awesome—here’s the **AKS deployment extension** you can append to your doc so the flow is **build → push → deploy**. It covers **two secure ways** to let AKS pull from ACR:

* **Recommended:** Attach ACR to AKS (managed identity; no imagePullSecrets needed)
* **Alternative:** Use an **imagePullSecret** (docker-registry secret)

---

# 🚀 Deploy to AKS from ACR

## 📋 Prerequisites

* ACR with your image pushed (e.g., `atulkamble2.azurecr.io/cloudnautic/hello-world:v1`)
* Azure CLI logged in: `az login`
* `kubectl` installed (`az aks install-cli` if needed)

> Tip: Tag explicitly before pushing so you know what you’re deploying:

```bash
cd docker-hello-world
sudo docker build -t atulkamble2.azurecr.io/cloudnautic/hello-world:v1 .
sudo docker push atulkamble2.azurecr.io/cloudnautic/hello-world:v1
```

---

## 1) Create AKS Cluster

```bash
# Vars
RG=rg-aks-cloudnautic
LOC=eastus
AKS=aks-cloudnautic
ACR=atulkamble2    # registry name only (no .azurecr.io)

# Resource Group
az group create -n $RG -l $LOC

# AKS cluster (basic, 1 node; adjust as needed)
az aks create \
  -g $RG -n $AKS \
  --node-count 1 \
  --node-vm-size Standard_B2s \
  --enable-managed-identity

# Kubeconfig
az aks get-credentials -g $RG -n $AKS
kubectl get nodes
```

---

## 2) Allow AKS to pull images from ACR

### Option A (Recommended): Attach ACR to AKS (no secrets)

```bash
az aks update -g $RG -n $AKS --attach-acr $ACR
```

After this, you can reference images like `atulkamble2.azurecr.io/...` directly—**no** `imagePullSecrets` required.

### Option B (Alternative): Use docker-registry secret

Enable ACR Admin on the registry (ACR → **Access keys** → Admin user **On**), then:

```bash
# Create a namespace for isolation (optional)
kubectl create namespace cloudnautic

# Create image pull secret in that namespace
kubectl create secret docker-registry acr-pull-secret \
  --docker-server=atulkamble2.azurecr.io \
  --docker-username=atulkamble2 \
  --docker-password='<ACR_PASSWORD>' \
  --docker-email='devnull@example.com' \
  -n cloudnautic
```

> Use **either** Option A **or** Option B (don’t mix). If you choose Option B, remember to add `imagePullSecrets` in your Deployment spec below.

---

## 3) Kubernetes Manifests

Create a folder (e.g., `k8s/`) and add these files.

### `k8s/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: cloudnautic
  labels:
    app: hello-world
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      # If you used Option B (secret), uncomment imagePullSecrets:
      # imagePullSecrets:
      #   - name: acr-pull-secret
      containers:
        - name: hello-world
          image: atulkamble2.azurecr.io/cloudnautic/hello-world:v1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 20
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
```

### `k8s/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world-svc
  namespace: cloudnautic
  labels:
    app: hello-world
spec:
  type: LoadBalancer
  selector:
    app: hello-world
  ports:
    - name: http
      port: 80
      targetPort: 80
```

> ✅ Note the correct service type spelling: `LoadBalancer` (not “LoabBalancer”).

---

## 4) Apply Manifests

```bash
# Create namespace (if not created via secret step)
kubectl create namespace cloudnautic || true

# Apply
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

# Watch rollout
kubectl rollout status deployment/hello-world -n cloudnautic

# Check resources
kubectl get pods -n cloudnautic -o wide
kubectl get svc  -n cloudnautic
```

When the service shows an **EXTERNAL-IP**, open it in the browser to verify.

---

## 5) Zero-downtime update (rollout a new image)

```bash
# Build & push a new tag
sudo docker build -t atulkamble2.azurecr.io/cloudnautic/hello-world:v2 .
sudo docker push atulkamble2.azurecr.io/cloudnautic/hello-world:v2

# Update the deployment image
kubectl set image deployment/hello-world hello-world=atulkamble2.azurecr.io/cloudnautic/hello-world:v2 -n cloudnautic

# Monitor
kubectl rollout status deployment/hello-world -n cloudnautic
kubectl rollout history deployment/hello-world -n cloudnautic

# Roll back if needed
# kubectl rollout undo deployment/hello-world -n cloudnautic
```

---

## 6) Common Errors & Fixes

* **`ImagePullBackOff` / `ErrImagePull`**

  * If using **Option A**: confirm `az aks update --attach-acr $ACR` ran without errors; re-pull after a minute.
  * If using **Option B**: ensure secret name matches and is in the **same namespace**; verify credentials; re-deploy.
  * Confirm the **image tag** exists in ACR and matches your manifest.
* **`Service type invalid`**

  * Use `LoadBalancer` (case-sensitive).
* **Probes failing**

  * Ensure the container serves on `:80` and `/`. Adjust `path`/`port` accordingly.

Quick checks:

```bash
kubectl describe pod <pod-name> -n cloudnautic
kubectl logs <pod-name> -n cloudnautic
```

---

## 7) (Optional) Ingress + DNS (Production-ish)

If you don’t want a public LoadBalancer per service:

1. Install NGINX Ingress Controller (AKS add-on or Helm).
2. Expose a single public IP.
3. Use an `Ingress` resource to route hostnames/paths to services.
4. Point your DNS `A` record to the ingress public IP.

(Ask me if you want a ready Helm + Ingress YAML for this app.)

---

## 8) Cleanup

```bash
# App resources only
kubectl delete -f k8s/service.yaml
kubectl delete -f k8s/deployment.yaml
kubectl delete namespace cloudnautic

# Whole cluster (irreversible)
az group delete -n $RG --yes --no-wait
```

---

## 📦 Folder Structure (suggested)

```
.
├── ACR-Setup-and-Push.md
├── AKS-Deploy.md
└── k8s
    ├── deployment.yaml
    └── service.yaml
```

---

If you want, I can also add:

* **Helm chart** for this app (values for image/tag/replicas)
* **GitHub Actions** pipeline to build → push to ACR → deploy to AKS automatically (with OpenID Connect, no secrets)

