---
author: Abhijeet Yadav
pubDatetime: 2025-10-18T10:00:00+05:45
modDatetime: 2025-10-18T10:00:00+05:45
title: Automating Kubernetes Deployments with Argo CD Image Updater
featured: true
draft: false
tags:
  - Kubernetes
  - Argo CD
  - DevOps
  - GitOps
  - CI/CD
canonicalURL: https://smale.codes/posts/automating-kubernetes-with-argocd-image-updater/
description: A comprehensive step-by-step guide to automating Kubernetes deployments using Argo CD Image Updater. Learn how to configure, secure, and troubleshoot automated image updates in a production-ready GitOps workflow.
---

# Automating Kubernetes Deployments with Argo CD Image Updater

Modern DevOps teams strive for **speed, reliability, and automation** in their deployment pipelines.  
In Kubernetes environments, **Argo CD** is the de-facto GitOps controller — but manually updating image tags in manifests can quickly become a bottleneck.

This post explains how to eliminate that manual step using **Argo CD Image Updater**, ensuring your deployments stay aligned with the latest container images automatically.

---

## Table of Contents

- [The Problem: Manual Image Tagging](#the-problem-manual-image-tagging)
- [Introducing Argo CD Image Updater](#introducing-argo-cd-image-updater)
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Step 1: Install Argo CD](#step-1-install-argo-cd)
- [Step 2: Create a Sample Application](#step-2-create-a-sample-application)
- [Step 3: Register the Application](#step-3-register-the-application)
- [Step 4: Install Image Updater](#step-4-install-image-updater)
- [Step 5: Configure Image Update Strategy](#step-5-configure-image-update-strategy)
- [Step 6: Configure Git Credentials](#step-6-configure-git-credentials)
- [Step 7: Test Auto-Update](#step-7-test-auto-update)
- [Step 8: Enable Pull Request Mode (Optional)](#step-8-enable-pull-request-mode-optional)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [Conclusion](#conclusion)
- [References](#references)

---

## The Problem: Manual Image Tagging

A typical CI/CD pipeline involves:
1. Developer pushes code.
2. CI builds and pushes a new Docker image.
3. CD (or a script) updates the Kubernetes manifests with the new tag.
4. Argo CD syncs the change into the cluster.

While effective, this setup creates a **tight coupling between CI and CD**, leading to errors when:
- Tokens expire or credentials fail.
- Manifest updates lag behind new image pushes.
- Multiple teams touch the same repository.

This creates unnecessary friction and manual intervention.

---

## Introducing Argo CD Image Updater

**Argo CD Image Updater** bridges this gap.  
It continuously monitors image registries, detects new tags, and updates Kubernetes manifests automatically — maintaining a **GitOps-first deployment pipeline**.

### Key Features
- Monitors registries for new tags.
- Updates manifests in Git automatically.
- Supports direct commits or pull requests.
- Works with Docker Hub, ECR, GCR, Harbor, and others.
- Compatible with semantic versioning and digest-based strategies.

---

## Architecture Overview

![Argo CD Image Updater Architecture](../../../src/assets/images/argocd-image-updater.png)

**Workflow Summary:**
1. CI pushes a new image tag.
2. Argo CD Image Updater detects it.
3. It commits the updated manifest to Git.
4. Argo CD syncs and deploys the new version automatically.

Git remains the **single source of truth**.  
Deployments are **traceable** and **reproducible**.

---

## Prerequisites

Before starting, make sure you have:

- A running **Kubernetes cluster** (Minikube, EKS, AKS, or GKE)
- **Argo CD** installed and accessible
- **kubectl** configured with the correct context
- A **GitHub repository** containing your Kubernetes manifests
- Optional: Docker registry credentials for private images

---

## Step 1: Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd   -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Expose the UI (optional for local environments):
```bash
kubectl patch svc argocd-server -n argocd   -p '{"spec": {"type": "NodePort"}}'
```

Retrieve admin password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret   -o jsonpath="{.data.password}" | base64 -d; echo
```

Access the UI via:  
`https://<minikube-ip>:31154`

---

## Step 2: Create a Sample Application

**Deployment.yaml**
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

**Service.yaml**
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


**kustomization.yaml**
```yaml
resources:
  - deployment.yaml
  - service.yaml
  ```


Commit and push to your Git repository.

---

## Step 3: Register the Application

In Argo CD UI:

- **Application Name:** `webapp`
- **Project:** default
- **Repository URL:** your Git repo
- **Path:** `.`
- **Cluster:** your Kubernetes cluster
- **Namespace:** default
- **Sync Policy:** automatic

Click **Create** → Argo CD will deploy it.

---

## Step 4: Install Image Updater

```bash
kubectl apply -n argocd   -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/master/manifests/install.yaml
```

Confirm installation:
```bash
kubectl get pods -n argocd | grep image-updater
```

![kubectl get pods -n argocd grep image-updater](../../../src/assets/images/get-pods-grep-image-updater.png)

---

## Step 5: Configure Image Update Strategy

Edit your Argo CD Application annotations:
```bash
kubectl edit app webapp -n argocd
```

Add these annotations:
```yaml
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: abhizeet/sample-flask-app:v1.x
    argocd-image-updater.argoproj.io/sample-flask-app.update-strategy: semver
    argocd-image-updater.argoproj.io/write-back-method: git:secret:argocd/git-credentials
```

---

## Step 6: Configure Git Credentials

### Option 1: HTTPS (Personal Access Token)
```bash
kubectl create secret generic git-credentials   -n argocd   --from-literal=username=<git-username>   --from-literal=password=<git-token>
```

### Option 2: SSH (Recommended)
```bash
ssh-keygen -t ed25519 -C "argocd@yourcompany" -f /etc/argocd/keys/argocd-key
argocd repo add git@github.com:<username>/<repo>.git   --ssh-private-key-path /etc/argocd/keys/argocd-key
```

---

## Step 7: Test Auto-Update

Push a new Docker image tag:
```bash
docker tag my-web-app:v1.2 my-web-app:v1.3
docker push my-web-app:v1.3
```

Tail logs:
```bash
kubectl logs -n argocd deploy/argocd-image-updater -f
```

### Expected Log Snippet

![Argocd Image updater Log](./../../../src/assets/images/argocd-updater-logs.png)


Refresh your Git repo — the Deployment.yaml or kustomization.yaml should now reference new version.

![Kustomize taml](./../../../src/assets/images/kustomize.png)

---

## Step 8: Enable Pull Request Mode (Optional)

For safer workflows, enable PR mode.

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

## Verification

Check Argo CD UI — you should see the new image tag automatically reflected and synced.

![Verify ArgoCD Deployment](../../../src/assets/images/verify-in-argocd-ui.png)

---

## Troubleshooting

| Issue | Possible Cause | Solution |
|-------|----------------|-----------|
| **Repository not accessible** | Invalid Git credentials | Verify token or SSH key |
| **ImagePullBackOff** | Registry authentication issue | Add imagePullSecret to deployment |
| **Updater not committing** | Insufficient Git write access | Check token permissions (`repo:write`) |
| **Stuck in OutOfSync** | Argo CD sync disabled | Enable auto-sync |

---

## Conclusion

With **Argo CD Image Updater**, your Kubernetes deployments become truly **self-updating and GitOps-compliant**.  
By automating image tag updates and syncing directly through Git, you ensure your environments remain consistent, auditable, and production-ready.

**Automation. Stability. GitOps.** — that’s the Argo way.

---

## References

- [Argo CD Documentation](https://argo-cd.readthedocs.io/)
- [Argo CD Image Updater](https://argocd-image-updater.readthedocs.io/en/stable/)
- [Kustomize Docs](https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/)
- [Tutorial Reference](https://www.youtube.com/watch?v=9zic7kKh_zU&t=1504s)
