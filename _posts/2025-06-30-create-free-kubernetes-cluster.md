---
title: Create a Kubernetes Cluster on windows for FREE
date: 2025-06-30 12:05:00 +0800
tags: [KinD, helm, kubernetes, grafana]
categories: [DevOps, Kubernetes]
image:
  path: /images/post-2/banner-logo.png
  fit: contain
  width: 100%
  height: auto
show_top_image: false
---

If you’re just getting started with Kubernetes and want a simple way to experiment locally, KinD is a great option.  
KinD stands for Kubernetes in Docker, and it lets you run a Kubernetes cluster right on your laptop using Docker containers.

This guide explains how to set up a <span style="color: #BF360C;"><strong>KinD cluster on a Windows laptop</strong></span>, for free.

---
## Why Use KinD

- Runs entirely on your local machine
- Uses Docker to simulate Kubernetes nodes
- Great for learning and testing


## Prerequisites

To set this up, I already had:

1. Download and install Docker Desktop: [docker-desktop](https://www.docker.com/products/docker-desktop)
2. Terminal to access cluster: [Git Bash](https://git-scm.com/downloads)
3. Kubernetes CLI: [kubectl](https://kubernetes.io/docs/tasks/tools/)
4. HELM: [HELM](https://helm.sh/docs/intro/install/)

### Setting Up KinD CLI in Git Bash on Windows

**Step 1:** Download kind binaries `kind-windows-amd64.exe`
 [KinD](https://github.com/kubernetes-sigs/kind/releases)

**Step 2:** Rename the file to `kind.exe`

  ```bash
mv ~/Downloads/kind-windows-amd64.exe ~/Downloads/kind.exe
```

**Step 3:** Create a folder and move `kind.exe` binary to `C:\kind`

```bash
mkdir -p /c/kind
mv ~/Downloads/kind.exe /c/kind/
```

**Step 4:** Add `C:\kind` to the Windows system PATH

- Open <span style="color: #4CAF50;"><strong>Start Menu</strong></span> → search “Environment Variables”
- Click <span style="color: #4CAF50;"><strong>Environment Variables</strong></span>
- Under <span style="color: #4CAF50;"><strong>System Variables</strong></span>, find and select `Path` → click <span style="color: #4CAF50;"><strong>Edit</strong></span>
- Click <span style="color: #4CAF50;"><strong>New</strong></span> and add `C:\kind`
- Click <span style="color: #4CAF50;"><strong>OK</strong></span> to save

**Step 5:** Verify the installation

Close and reopen Git Bash, then run:

```bash
kind version

kind v0.22.0 go1.22.2 windows/amd64
```

This confirms `kind` is installed and ready to use.


---

## Create a Cluster

**Step 1:** Create the Kind Config File `kind-cluster-config.yaml`

<span style="color: #9C27B0;"><code>extraPortMappings</code></span> helps to expose your applications locally. This ensures port 31034 from the Kind container maps to your Windows localhost. You can add any number of ports and create cluster.

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 31034
        hostPort: 31034
        protocol: TCP
  - role: worker
  - role: worker
```

**Step 2:** Create the Cluster. Open your terminal and run command

```bash
kind create cluster --name my-cluster --config kind-cluster-config.yaml
```
**Step 3:** Verify the Cluster Once the cluster is created, verify the nodes with

```bash
[git@vamsi194 ~]# kind create cluster --name my-cluster --config kind-cluster.yaml
Creating cluster "my-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.29.2) 🖼
 ✓ Preparing nodes 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-my-cluster"
You can now use your cluster with:

kubectl cluster-info --context kind-my-cluster
```

**Step 4:** Verify the cluster
```bash
[git@vamsi194 ~]# kubectl get nodes
NAME                       STATUS   ROLES           AGE     VERSION
my-cluster-control-plane   Ready    control-plane   2m38s   v1.29.2
my-cluster-worker          Ready    <none>          2m17s   v1.29.2
my-cluster-worker2         Ready    <none>          2m14s   v1.29.2
```

## Exposing application service locally

I'm installing grafana application via HELM to verify the process

```bash
[git@vamsi194 ~]# helm repo add grafana https://grafana.github.io/helm-charts
"grafana" has been added to your repositories


[git@vamsi194 ~]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "grafana" chart repository
Update Complete. ⎈Happy Helming!⎈
```

Creating Custom Values `grafana-values.yaml` for accessing the service on specific port 31034

```yaml
service:
  type: NodePort
  port: 80
  targetPort: 3000
  nodePort: 31034

adminUser: admin
adminPassword: admin
```

Installing and verifying the service status on the cluster

```bash
[git@vamsi194 ~]# helm install grafana grafana/grafana -f grafana-values.yaml
NAME: grafana
LAST DEPLOYED: Thu Jul  3 20:12:44 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1

[git@vamsi194 ~]# kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
grafana-78b7855d8b-f4ktn   1/1     Running   0          42s

[git@vamsi194 ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
grafana      NodePort    10.96.65.126   <none>        80:31034/TCP   46s
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP        22m
```

Access the service locally through browser..
<div style="display: flex; justify-content: center; margin: 10px 0;">
  <!-- Left Section: Image + Caption -->
  <div style="flex: 1; display: flex; flex-direction: column; align-items: center;">
    <a href="/images/post-2/grafana-post2.png" class="popup img-link shimmer">
      <img src="/images/post-2/grafana-post2.png" alt="GitLab CI/CD flow for Docker releases"
           style="width: 100%; max-width: 600px; border-radius: 8px;" loading="lazy" />
    </a>
    <div style="text-align: center; font-size: 0.9em; color: gray; margin-top: 8px;">
      <em>Figure: Locally accessed grafana application</em>
    </div>
  </div>

  <!-- Right Section: Empty -->
  <div style="flex: 1;"></div>
</div>



