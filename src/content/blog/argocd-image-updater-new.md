---
author: Abhijeet Yadav
pubDatetime: 2025-05-31T22:44:00+05:45
modDatetime: 2025-10-18T22:44:00+05:45
title: Automating Kubernetes Deployments with Argo CD Image Updater
featured: false
draft: false
tags:
  - kubernetes
  - argocd
  - devops
  - automation
canonicalURL: https://smale.codes/posts/argocd-image-updater/
description: Learn how to automate Kubernetes deployments using **Argo CD Image Updater**. This guide walks you through setup, configuration, and debugging — enabling seamless, GitOps-driven image updates without manual intervention.
---

## 💡 The Problem — Manual Tag Updates

In a traditional CI/CD pipeline:

1. Developers push code to GitHub or GitLab.
2. A **CI pipeline** (Jenkins, GitHub Actions, etc.) builds and tags a Docker image.
3. The new image is pushed to the registry (Docker Hub, ECR, Harbor, etc.).
4. A **CD pipeline** updates the Kubernetes manifests (via shell scripts or manual commits).
5. Argo CD detects the change and syncs your cluster.

### ❌ The Drawback

The **CD pipeline depends on the CI pipeline**.  
If your CI job fails to push manifest updates (due to Git token or permission issues), deployments stall even when new images exist.

This tight coupling is fragile in production environments.

---

## 🚀 The Solution — Argo CD Image Updater

**Argo CD Image Updater** is an official add-on that automates this process by:

- Watching your image registry for new tags.
- Updating Kubernetes manifests automatically.
- Optionally creating Git commits or Pull Requests.
- Triggering Argo CD to sync your cluster.

✅ It **decouples CI and CD** — your cluster updates itself when new images are available.

---

## 🏗️ Architecture Overview

Here’s how the flow looks with Image Updater:

![Argo CD Image Updater Architecture](../../../src/assets/images/argocd-image-updater.png)

**Flow Summary:**

1. Developer pushes code → CI builds and pushes new image.
2. Argo CD Image Updater detects the new tag.
3. It commits updated image tags to the GitOps repo.
4. Argo CD automatically syncs the change into the cluster.

✅ Fully GitOps-driven  
✅ No dependency on CI  
✅ Reproducible, auditable deployments

---

## ⚙️ Prerequisites

Before you begin, ensure you have:

- A **Kubernetes cluster** (Minikube, EKS, GKE, etc.)
- **Argo CD** installed (`argocd` namespace)
- **kubectl** installed and configured
- A **GitHub repository** with your manifests
- Optional: Docker registry credentials (for private images)

---

## 🔧 Step 1: Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd   -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Expose Argo CD externally:
```bash
kubectl patch svc argocd-server -n argocd   -p '{"spec": {"type": "NodePort"}}'
```

Get admin password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret   -o jsonpath="{.data.password}" | base64 -d; echo
```

Access via:
```
https://<minikube-ip>:31154
```

---

## 🧱 Step 2: Create a Sample Application

**Deployment**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web-app
  labels:
    app: my-web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-web-app
  template:
    metadata:
      labels:
        app: my-web-app
    spec:
      containers:
      - name: my-web-app
        image: abhizeet/sample-flask-app:v1.2
        ports:
        - containerPort: 80
```

**Service**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-web-app
spec:
  type: NodePort
  selector:
    app: my-web-app
  ports:
  - port: 80
    targetPort: 80
```

Commit and push these files to your GitHub repository.

---

## 🧩 Step 3: Register the Application in Argo CD

In Argo CD UI → “New Application”

- Name: `webapp`
- Repo URL: your Git repo
- Path: `.`
- Namespace: `default`
- Sync Policy: `Automatic`

Click **Create** and watch your app deploy!

---

## 🧰 Step 4: Install Argo CD Image Updater

Install the official manifest:

```bash
kubectl apply -n argocd   -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/master/manifests/install.yaml
```

Check pod status:
```bash
kubectl get pods -n argocd | grep image-updater
```

---

## ⚙️ Step 5: Configure Image Update Strategy

Edit your Argo CD Application annotations:

```bash
kubectl edit app webapp -n argocd
```

Add:

```yaml
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: abhizeet/sample-flask-app:v1.x
    argocd-image-updater.argoproj.io/sample-flask-app.update-strategy: semver
    argocd-image-updater.argoproj.io/write-back-method: git:secret:argocd/git-credentials
```

---

## 🔐 Step 6: Configure Git Credentials

### Option 1 — HTTPS (PAT)
```bash
kubectl create secret generic git-credentials   -n argocd   --from-literal=username=<git-username>   --from-literal=password=<git-token>
```

Then reference it:
```yaml
argocd-image-updater.argoproj.io/git-credentials=secret:argocd/git-credentials
```

### Option 2 — SSH (Recommended)

```bash
ssh-keygen -t ed25519 -C "argocd@yourcompany" -f /etc/argocd/keys/argocd-key
```

Add the public key as a **Deploy Key** in GitHub.  
Then register the repo in Argo CD:

```bash
argocd repo add git@github.com:<username>/<repo>.git   --ssh-private-key-path /etc/argocd/keys/argocd-key
```

---

## 🧪 Step 7: Test Image Auto-Update

Push a new image tag:
```bash
docker tag my-web-app:v1.2 my-web-app:v1.3
docker push my-web-app:v1.3
```

Tail logs:
```bash
kubectl logs -n argocd deploy/argocd-image-updater -f
```

### Example Log Output

```text
time="2025-10-18T09:40:03Z" level=info msg="Successfully updated image 'abhizeet/sample-flask-app:2' to 'abhizeet/sample-flask-app:3'"
time="2025-10-18T09:44:07Z" level=info msg="Successfully committed manifest update to Git"
time="2025-10-18T09:46:08Z" level=info msg="Found new tag v1.3 for image abhizeet/sample-flask-app"
```

---

## 🧩 Step 8: (Optional) Enable Pull Request Mode

Instead of committing directly, you can have Image Updater create PRs.

Edit configmap:
```bash
kubectl edit configmap argocd-image-updater-config -n argocd
```

Add:
```yaml
git.write-back-method: "pull-request"
git.user: "argocd-bot"
git.email: "argocd-bot@yourcompany.com"
```

---

## ✅ Verification

Once synced, check the Argo CD UI — the new image tag should appear automatically.

![Verify deployment in ArgoCD UI](../../../src/assets/images/verify-in-argocd-ui.png)

---

## ⚡ Benefits

| Feature | Benefit |
|----------|----------|
| **Automatic tag detection** | Removes need for manual manifest edits |
| **CI/CD decoupled** | Deployments run even if CI fails |
| **GitOps-compliant** | Every change is auditable |
| **Multi-registry support** | Docker Hub, ECR, GCR, Harbor |
| **PR mode support** | Safe updates for production |

---

## 🧰 Troubleshooting

- **Repo not accessible** → Check Git credentials or SSH key.  
- **ImagePullBackOff** → Verify registry secret in the app namespace.  
- **Updater not committing** → Ensure write permissions or PAT scopes include `repo`.

---

## 🏁 Conclusion

With **Argo CD Image Updater**, you get a clean, automated, and production-safe deployment flow:

1. CI builds and pushes images.  
2. Image Updater detects new tags.  
3. Git gets updated automatically.  
4. Argo CD syncs the cluster.

Goodbye manual tag edits — hello fully automated GitOps 🚀

---

### 📚 References

- [Argo CD Official Docs](https://argo-cd.readthedocs.io/)
- [Argo CD Image Updater Docs](https://argocd-image-updater.readthedocs.io/en/stable/)
- [Kustomize Documentation](https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/)
