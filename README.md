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
Here’s a clean, well-documented version of your **AKS + ACR Deployment Guide** — formatted like a professional step-by-step README.

---

# ☁️ **AKS + ACR Deployment Guide**

This guide walks through **creating an Azure Kubernetes Service (AKS)** cluster, connecting it to **Azure Container Registry (ACR)**, and deploying your application using Kubernetes manifests.

---

## 🔧 **1. Create Resource Group**

```bash
az group create --name devops --location eastus
```

---

## 🚀 **2. Create AKS Cluster**

You can create a simple cluster:

```bash
az aks create --resource-group devops --name mycluster --generate-ssh-keys
```

Or create a **customized cluster** with monitoring enabled:

```bash
az aks create \
  --resource-group devops \
  --name mycluster \
  --node-count 1 \
  --enable-addons monitoring \
  --generate-ssh-keys
```

---

## ⚙️ **3. Install kubectl**

```bash
sudo apt update -y
sudo apt install snapd -y
sudo snap install kubectl --classic
```

Verify:

```bash
kubectl version --client
```

---

## 🔑 **4. Connect to Azure Subscription**

If you manage multiple subscriptions, set the right one:

```bash
az account set --subscription 50818730-e898-4bc4-bc35-d998af53d719
```

---

## 🔗 **5. Connect to AKS Cluster**

```bash
az aks get-credentials --resource-group devops --name mycluster --overwrite-existing
```

Check connection:

```bash
kubectl get nodes
```

---

## 🧩 **6. Connect AKS to ACR**

This step allows AKS to **pull images** from your private ACR without manual Docker login.

```bash
az aks update -g devops -n mycluster --attach-acr atulkamble2
```

Confirm:

```bash
az aks show -g devops -n mycluster --query "servicePrincipalProfile"
```

---

## 🗂️ **7. Create Namespace**

```bash
kubectl create namespace cloudnautic
```

---

## 📦 **8. Deploy Application from ACR**

Make sure your Kubernetes YAML files reference your ACR image, e.g.:

**`deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-web
  namespace: cloudnautic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-web
  template:
    metadata:
      labels:
        app: hello-web
    spec:
      containers:
      - name: hello-web
        image: atulkamble2.azurecr.io/cloudnautic/hello-world:latest
        ports:
        - containerPort: 80
```

**`service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-web-svc
  namespace: cloudnautic
spec:
  type: LoadBalancer
  selector:
    app: hello-web
  ports:
  - port: 80
    targetPort: 80
```

Apply both:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

---

## ✅ **9. Verify Deployment**

```bash
kubectl get pods -n cloudnautic
kubectl get svc -n cloudnautic
```

Once the external IP appears under the service, open your app in a browser:

```
http://<external-ip>
```

---

## 🧹 **10. Cleanup Resources**

When done, delete all resources to avoid charges:

```bash
az group delete --name devops --yes --no-wait
```

---

Would you like me to extend this guide into a **GitHub-ready README.md** (with emojis, table of contents, and diagram for CI/CD flow)?
It would make a perfect documentation addition to your repo `docker-hello-world/k8s`.
