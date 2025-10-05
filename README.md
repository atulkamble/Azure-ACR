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
