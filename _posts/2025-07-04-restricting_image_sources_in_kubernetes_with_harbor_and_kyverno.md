---
title: Restricting Image Sources in Kubernetes with Harbor and Kyverno
date: 2025-07-12 12:00:00 -0600
categories: [Kubernetes, Security, CNCF]
tags: [harbor, kyverno, container-security, private-registry]
image: /images/post-3/banner-post-3.png
show_top_image: false
---

When deploying workloads to Kubernetes, it’s critical to control which container images are allowed into the environment. Pulling images from public or unverified registries introduces security risks such as unscanned vulnerabilities or untrusted third-party code.

We walk through how to **restrict image pulls to an internal private registry (Harbor)** using **Kyverno**, a Kubernetes-native policy engine.

<div style="width: 100%; display: flex; justify-content: center; margin: 20px 0;">
  <div style="max-width: 1000px; width: 100%;">
    <a href="/images/post-3/post-3.png" class="popup img-link shimmer">
      <img src="/images/post-3/post-3.png" 
           alt="Harbor project demo with nginx image"
           style="width: 100%; height: auto; display: block; border-radius: 8px;" 
           loading="lazy" />
    </a>
    <div style="text-align: center; font-size: 1em; color: gray; margin-top: 8px;">
      <em>Figure: Enforcing image pull policy using Harbor and Kyverno in Kubernetes</em>
    </div>
  </div>
</div>

---

### Prerequisites

To set this up, I already had:

1. Running Kubernetes cluster : [KinD](https://www.docker.com/products/docker-desktop)
2. Kubernetes CLI: [kubectl](https://kubernetes.io/docs/tasks/tools/)
3. HELM: [HELM](https://helm.sh/docs/intro/install/) 

---

### KinD Cluster Configuration

KinD config to expose NodePort 31034 on the control-plane node for external access from AWS EC2.

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
```yaml
[root@ip-172-31-2-14 ~]# kind create cluster --name my-cluster --config config.yaml
Creating cluster "my-cluster" ...
```

---

### Installing Harbor

Installing Harbor to set up a private container registry for securely storing and managing Docker images.

Providing custom values to install harbor to customize according to the domain.

```yaml
[root@ip-172-31-2-14 ~]# helm install harbor harbor/harbor -n default \
  --set expose.type=nodePort \
  --set expose.tls.enabled=false \
  --set expose.nodePort.name=harbor \
  --set expose.nodePort.ports.http.nodePort=31034 \
  --set harborAdminPassword=Harbor12345 \
  --set externalURL=http://ec2-3-148-167-213.us-east-2.compute.amazonaws.com:31034
NAME: harbor
LAST DEPLOYED: Sun Jul  6 07:38:45 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Please wait for several minutes for Harbor deployment to complete.
Then you should be able to visit the Harbor portal at http://ec2-3-148-167-213.us-east-2.compute.amazonaws.com:31034
For more details, please visit https://github.com/goharbor/harbor
```

Able to access the console & login

<div style="display: flex; gap: 20px; justify-content: center; flex-wrap: wrap; margin-bottom: 2em;">
  <div style="flex: 1; display: flex; flex-direction: column; align-items: center;">
    <a href="/images/post-3/login.png" class="popup img-link shimmer">
      <img src="/images/post-3/login.png" alt="GitLab Pipeline"
           style="width: 100%; max-width: 400px; border-radius: 8px;" loading="lazy" />
    </a>
    <p style="margin-top: 0.5em; font-size: 14px;">Harbor Login Page</p>
  </div>

  <div style="flex: 1; display: flex; flex-direction: column; align-items: center;">
    <a href="/images/post-3/console.png" class="popup img-link shimmer">
      <img src="/images/post-3/console.png" alt="DockerHub Tag"
           style="width: 100%; max-width: 400px; border-radius: 8px;" loading="lazy" />
    </a>
    <p style="margin-top: 0.5em; font-size: 14px;">Harbor Console</p>
  </div>
</div>

---

### Installing Kyverno

We are using Kyverno to make sure all images come only from our internal Harbor registry.

```bash
[root@ip-172-31-2-14 ~]# helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
"kyverno" has been added to your repositories
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ingress-nginx" chart repository
...Successfully got an update from the "harbor" chart repository
...Successfully got an update from the "kyverno" chart repository
Update Complete. ⎈Happy Helming!⎈
[root@ip-172-31-2-14 ~]# helm install kyverno kyverno/kyverno -n kyverno --create-namespace
NAME: kyverno
LAST DEPLOYED: Sun Jul  6 07:50:03 2025
NAMESPACE: kyverno
STATUS: deployed
REVISION: 1
NOTES:
Chart version: 3.4.4
Kyverno version: v1.14.4

Thank you for installing kyverno! Your release is named kyverno.
```
---

### Implement Image Restriction with Kyverno

This Kyverno policy `restrict-image-policy.yaml` ensures that all containers and initContainers in pods pull images only from our internal Harbor registry, blocking any from external sources.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registry
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: allow-only-internal-registry
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "Only images from ec2-3-148-167-213.us-east-2.compute.amazonaws.com are allowed."
        pattern:
          spec:
            containers:
              - image: "ec2-3-148-167-213.us-east-2.compute.amazonaws.com/*"
            initContainers:
              - image: "ec2-3-148-167-213.us-east-2.compute.amazonaws.com/*"
```

---
### Push Sample Image to Internal Harbor Registry

To validate now Kubernetes can only pull images from our internal Harbor registry, we’ll build a simple Docker image, tag it properly, and push it to our project **demo** on Harbor registry.

```bash
[root@ip-172-31-2-14 ~]# docker login ec2-3-148-167-213.us-east-2.compute.amazonaws.com:31034
Username: admin
Password:
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
[root@ip-172-31-2-14 ~]# docker pull nginx:latest
latest: Pulling from library/nginx
3da95a905ed5: Pull complete
6c8e51cf0087: Pull complete
9bbbd7ee45b7: Pull complete
48670a58a68f: Pull complete
ce7132063a56: Pull complete
23e05839d684: Pull complete
ee95256df030: Pull complete
Digest: sha256:93230cd54060f497430c7a120e2347894846a81b6a5dd2110f7362c5423b4abc
Status: Downloaded newer image for nginx:latest
docker.io/library/nginx:latest
[root@ip-172-31-2-14 ~]# docker tag nginx:latest ec2-3-148-167-213.us-east-2.compute.amazonaws.com:31034/demo/nginx:latest
[root@ip-172-31-2-14 ~]# docker push ec2-3-148-167-213.us-east-2.compute.amazonaws.com:31034/demo/nginx:latest
The push refers to repository [ec2-3-148-167-213.us-east-2.compute.amazonaws.com:31034/demo/nginx]
07eaefc6ebf2: Pushed
de2ef8ceb76a: Pushed
e6c40b7bdc83: Pushed
f941308035cf: Pushed
81a9d30670ec: Pushed
1bf33238ab09: Pushed
1bb35e8b4de1: Pushed
latest: digest: sha256:ccde53834eab53e85b35526a647cdb714ea4521b1ddf5a07b5c8787298d13087 size: 1778
```
We could see the nginx image is tagged and pushed to our registry.

<div style="width: 100%; display: flex; justify-content: center; margin: 20px 0;">
  <div style="max-width: 1000px; width: 100%;">
    <a href="/images/post-3/image_registery.png" class="popup img-link shimmer">
      <img src="/images/post-3/image_registery.png" 
           alt="Harbor project demo with nginx image"
           style="width: 100%; height: auto; display: block; border-radius: 8px;" 
           loading="lazy" />
    </a>
    <div style="text-align: center; font-size: 1em; color: gray; margin-top: 8px;">
      <em>Figure: Project demo with latest nginx image</em>
    </div>
  </div>
</div>
---
### Proving Image Pull Works Only from Internal Harbor Registry

To confirm that Kubernetes is restricted to pulling container images only from our internal Harbor registry, we’ll attempt to run a pod using the NGINX image that we previously pushed to Harbor.


We tested our image pull policy by running a pod using the NGINX image hosted on our internal Harbor registry
```bash
[root@ip-172-31-2-14 ~]# kubectl run nginx-harbor \
  --image=ec2-3-148-167-213.us-east-2.compute.amazonaws.com:31034/demo/nginx:latest
pod/nginx-harbor created
```

To verify the enforcement, we attempted to run a pod using the default NGINX image from the public Docker registry
```bash
[root@ip-172-31-2-14 ~]# k run nginx-normal --image=nginx:latest
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request:

resource Pod/default/nginx-normal was blocked due to the following policies

restrict-image-registry:
  allow-only-internal-registry: 'validation error: Only images from ec2-3-148-167-213.us-east-2.compute.amazonaws.com
    are allowed. rule allow-only-internal-registry failed at path /spec/containers/0/image/'
```

This pod failed to launch. Kyverno blocked the request and returned a validation error:

<p style="color: darkred;"><strong>Only images from ec2-3-148-167-213.us-east-2.compute.amazonaws.com:31034 are allowed.</strong></p>



 Using Kyverno and Harbor, we successfully enforced image pull restrictions to allow only trusted images from our internal registry, helping prevent random or unauthorized installations in the cluster.