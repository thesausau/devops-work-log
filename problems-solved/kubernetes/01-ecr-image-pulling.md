# ECR Image Pulling on Self-Managed Kubernetes

## The Problem

Pods fail to pull images from Amazon ECR with:

```
Failed to pull image "<account_id>.dkr.ecr.us-east-1.amazonaws.com/app:latest":
authorization failed: no basic auth credentials
```

This happens even when the EC2 instance has an IAM role with
`AmazonEC2ContainerRegistryReadOnly` attached.

---

## Why It Happens

ECR uses temporary tokens that expire every 12 hours. On EKS, token fetching
is handled natively. On self-managed Kubernetes, you have to set it up.

The root cause is that **kubelet** (not the container) handles image pulls.
kubelet runs as a systemd service outside the container world and does not
automatically inherit the EC2 instance IAM role.

```
Pod (container)   → inherits EC2 IAM role automatically ✅
kubelet (systemd) → does NOT inherit EC2 IAM role       ❌
```

Additionally, Kubernetes 1.27+ removed built-in ECR credential fetching
(KEP-2133) to be cloud-agnostic. A credential provider plugin is now required.

---

## What Does NOT Work

### amazon-ecr-credential-helper

```yaml
# WRONG - this is a Docker daemon tool, not a containerd tool
- name: Install amazon-ecr-credential-helper
  apt:
    name: amazon-ecr-credential-helper

- name: Configure containerd
  blockinfile:
    path: /etc/containerd/config.toml
    block: |
      [plugins."io.containerd.grpc.v1.cri".registry.configs."<account_id>...".cred_helper]
        helper = "ecr-login"
```

Fails because `amazon-ecr-credential-helper` is for Docker daemon.
The `cred_helper` syntax is Docker syntax — invalid in containerd.

### CronJob Token Refresh

Works but is the wrong approach:
- Stores tokens in etcd (security concern)
- Silent failure — pods fail hours after CronJob fails
- Requires CronJob + RBAC + ServiceAccount overhead because CronJob uses kubectl and kubctl needs to talk to Kubernetes API hence, the need for RBAC, ServiceAccount
- Needs secret seeded in every namespace
---

## The Correct Solution: ecr-credential-provider

**Official repo:** `github.com/kubernetes/cloud-provider-aws`

kubelet calls this binary on-demand when it needs to pull an ECR image.
The binary uses the EC2 instance IAM role to fetch a token, returns it to
kubelet, and exits. Token is cached for 12 hours and auto-renewed. kubelet never talks to Kubernetes API.

```
kubelet needs ECR image
        │
        ▼
Token cached and valid?
   YES → use it
   NO  → execute ecr-credential-provider binary
              │
              ▼
         calls 169.254.169.254 (EC2 metadata)
         gets IAM credentials
              │
              ▼
         calls ECR GetAuthorizationToken
         gets fresh token (12h)
              │
              ▼
         returns token to kubelet
         (binary exits — not a daemon)
              │
              ▼
         kubelet caches token 12h ✅
```

### Identity

The binary uses the **EC2 Instance IAM Role** as its identity — fetched
directly from the metadata service at `169.254.169.254`. It has no
Kubernetes identity and never talks to the K8s API. This is why no
ServiceAccount or RBAC is needed (unlike the CronJob approach).

---

## Setup

Run against **all nodes** (master + workers). See:
[examples/setup-ecr-credential-provider.yml](examples/setup-ecr-credential-provider.yml)

```bash
ansible-playbook -i inventory kubernetes/examples/setup-ecr-credential-provider.yml
```

### Version Matching

The binary version must match your Kubernetes version:

| Kubernetes | cloud-provider-aws |
|------------|-------------------|
| 1.29.x | v1.29.10 |
| 1.30.x | v1.30.10 |
| 1.31.x | v1.31.9 |
| 1.32.x | v1.32.6 |
| 1.33.x | v1.33.2 |

### Key Gotchas

**1. No pre-built binary** — must build from source.

**2. Ubuntu apt Go is too old:**
```bash
go version  # 1.18 — too old
# cloud-provider-aws requires Go 1.24+
# Download from go.dev/dl/ directly
```

**3. Use full Go binary path in Ansible:**
```yaml
# Wrong — uses old system Go from PATH:
shell: go build ./cmd/ecr-credential-provider/

# Correct — explicit path:
shell: /usr/local/go/bin/go build ./cmd/ecr-credential-provider/
```

**4. Correct kubelet override file:**
```bash
# The kubelet service is at: found using systemctl cat kubelet
# /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
# But that file's comment says to use:
/etc/default/kubelet   ← set KUBELET_EXTRA_ARGS here
```

---

## Result

After setup, pod manifests need nothing special:

```yaml
spec:
  containers:
    - name: app
      image: <account_id>.dkr.ecr.us-east-1.amazonaws.com/app:latest
      # No imagePullSecrets needed ✅
```

Works on self-managed K8s today. Same manifests work on EKS later
(EKS handles ECR natively using K8 POD AGENT, plugin not needed).

---

## Verify

```bash
# kubelet is using the credential provider
ps aux | grep kubelet | grep credential-provider

# Test image pull
kubectl run test-ecr \
  --image=<account_id>.dkr.ecr.us-east-1.amazonaws.com/app:latest \
  -n your-namespace
kubectl describe pod test-ecr -n your-namespace
# Look for: "Successfully pulled image"
kubectl delete pod test-ecr -n your-namespace
```
## Examples
- [`examples/setup-ecr-credential-provider.yml`](examples/setup-ecr-credential-provider.yml) — Ansible playbook to install and configure the credential provider on all nodes
