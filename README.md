---

````markdown
# 🚀 CI/CD Demo: Jenkins → Docker Hub → Kubernetes

This project shows how to **automate application delivery** using:

- Jenkins for CI/CD pipelines
- Docker Hub as container registry
- Kubernetes cluster (1 master, 2 workers on AWS EC2)

---

## 🏗️ Cluster Setup (Pre-requisite)

We already have a Kubernetes cluster:
- 1 **Control Plane (master)** node  
- 2 **Worker** nodes  
- All nodes are Ubuntu 24.04 on AWS EC2  
- Container runtime: containerd  
- Cluster bootstrapped with `kubeadm`

👉 You should already be able to run on the **control-plane**:
```bash
kubectl get nodes -o wide
````

and see all 3 nodes (`Ready` status).

---

## 📌 What Happens in This Demo

1. Developer pushes code → GitHub repo (this project).
2. Jenkins picks it up (Pipeline job).
3. Jenkins builds a Docker image → pushes to Docker Hub.
4. Jenkins applies Kubernetes manifests → updates Deployment.
5. Application is live on Kubernetes cluster.

---

## 🔑 One-Time Setup (Manual)

### 1. On **Docker Hub**

* Create a repo named `k8s-cicd-demo` under your account.
* Example image name:

  ```
  <dockerhub-username>/k8s-cicd-demo:latest
  ```

### 2. On **Control Plane (master)**

Apply Kubernetes manifests the first time:

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

Verify:

```bash
kubectl get pods -o wide
kubectl get svc k8s-cicd-demo-svc
```

This will create the Deployment and Service. Jenkins will update them later.

### 3. On **Jenkins Server**

(Jenkins can run on a separate EC2 or on one of your worker nodes)

* Ensure Jenkins has:

  * **Docker** installed and running
  * **kubectl** installed
  * Access to Kubernetes cluster (`~/.kube/config` copied from master’s `/etc/kubernetes/admin.conf`)
* Add credentials in Jenkins:

  * Go to **Manage Jenkins → Credentials → System → Global credentials**
  * Add Docker Hub username/password (ID = `dockerhub-creds`)

---

## 🚀 Running the Pipeline

### Step 1: Create Jenkins Pipeline Job

* In Jenkins UI → New Item → Pipeline
* Enter name: `k8s-cicd-demo`
* Choose **Pipeline from SCM**
* Point to this GitHub repository
* Jenkinsfile path = `Jenkinsfile`

### Step 2: Run the Pipeline

* Click **Build Now**
* Jenkins will execute stages:

  1. Checkout code
  2. Build Docker image
  3. Push to Docker Hub
  4. Deploy/update in Kubernetes
  5. Verify rollout

### Step 3: Confirm Deployment

On the **control-plane node**, run:

```bash
kubectl get all -o wide
```

You should see:

* `Deployment` named `k8s-cicd-demo`
* `ReplicaSet` created by Deployment
* 2 running `Pods`
* `Service` named `k8s-cicd-demo-svc`

---

## 🌐 Access the Application

The Service is of type **NodePort** (port `30080`).

👉 From your browser:

```
http://<any-worker-node-public-ip>:30080
```

You should see **“Hello from Kubernetes!”**



## 🔄 Everyday Usage

* Any time you push changes (like editing `index.html`), Jenkins will automatically:

  * Build a new Docker image
  * Push it to Docker Hub
  * Update Kubernetes Deployment
  * Rollout the new version

No manual `kubectl apply` needed after first setup 🎉

---

## 📝 Summary

* **Control Plane** → runs `kubectl`, hosts cluster config
* **Worker Nodes** → run app Pods
* **Jenkins** → automates build & deploy
* **Docker Hub** → stores images

You now have a **full CI/CD pipeline** on a self-hosted Kubernetes cluster.

Perfect 👍 — adding a **Troubleshooting** section is a good idea because in real QA they’ll often test if you understand what to do when things break.
Here’s a section you can **append to your README.md**:

---

````markdown
## 🛠️ Troubleshooting Guide

Even with automation, things can go wrong. Here are common issues and fixes:

---

### 1. ❌ Pods stuck in `ImagePullBackOff` or `ErrImagePull`
**Cause:** Kubernetes can’t pull image from Docker Hub.  
**Fix:**
- Check the image name in `k8s/deployment.yaml` matches your Docker Hub repo.
- Ensure the Jenkins pipeline **successfully pushed** the image (check Docker Hub web UI).
- Pull image manually from worker node:
  ```bash
  docker pull <dockerhub-username>/k8s-cicd-demo:latest
````

* If repo is private, create a Kubernetes secret:

  ```bash
  kubectl create secret docker-registry dockerhub-secret \
    --docker-username=<username> \
    --docker-password=<password> \
    --docker-email=<email>
  ```

  and patch the Deployment to use it.

---

### 2. ❌ Jenkins can’t run `kubectl`

**Cause:** Jenkins doesn’t have cluster config.
**Fix:**

* On **control-plane node**:

  ```bash
  sudo cat /etc/kubernetes/admin.conf
  ```
* Copy this file to Jenkins server at:

  ```
  /var/lib/jenkins/.kube/config
  ```
* Ensure Jenkins user owns it:

  ```bash
  sudo chown jenkins:jenkins /var/lib/jenkins/.kube/config
  ```

---

### 3. ❌ Service created but app not accessible

**Cause:** Service type is `NodePort` and firewall/Security Group is blocking.
**Fix:**

* Check service:

  ```bash
  kubectl get svc k8s-cicd-demo-svc
  ```

  Example:

  ```
  NodePort: 30080/TCP
  ```
* Open port `30080` in your AWS Security Group for all worker nodes.
* Access app via:

  ```
  http://<worker-node-public-ip>:30080
  ```

---

### 4. ❌ Jenkins build fails at Docker push

**Cause:** Wrong or missing Docker Hub credentials in Jenkins.
**Fix:**

* In Jenkins UI → **Manage Jenkins → Credentials**
* Check credentials ID matches `dockerhub-creds` in `Jenkinsfile`.
* Re-enter Docker Hub username/password if needed.

---

### 5. ❌ Old Pods still running after deploy

**Cause:** Deployment not updating.
**Fix:**

* Force rollout restart:

  ```bash
  kubectl rollout restart deployment k8s-cicd-demo
  ```
* Watch status:

  ```bash
  kubectl rollout status deployment k8s-cicd-demo
  ```

---

### 6. ❌ Nodes show `NotReady`

**Cause:** Cluster was restarted (AWS EC2 free tier often stopped/started).
**Fix:**

* Restart kubelet and containerd on each node:

  ```bash
  sudo systemctl restart kubelet
  sudo systemctl restart containerd
  ```
* On control-plane, check:

  ```bash
  kubectl get nodes
  ```

---



