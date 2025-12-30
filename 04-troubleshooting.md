# Troubleshooting Guide

## 1. `kubectl` Cannot Connect (`localhost:8080`)

### ❌ Symptom

```text
couldn't get current server API group list:
Get "http://localhost:8080/api?timeout=32s":
dial tcp [::1]:8080: connect: connection refused
```

### 🔍 Root Cause

* `kubectl` defaults to `localhost:8080` **when kubeconfig is missing or invalid**
* Common causes:

  * `~/.kube/config` not present
  * Wrong context selected
  * kubeconfig never copied from control plane

### ✅ Workaround / Fix

```bash
mkdir -p ~/.kube
scp pi@<control-plane-ip>:/etc/kubernetes/admin.conf ~/.kube/config
chmod 600 ~/.kube/config
kubectl config get-contexts
kubectl config use-context kubernetes-admin@kubernetes
```

✅ **Key Insight**
`kubectl` is just a **client binary**.
It does **not need to run on the control plane** — it only needs a valid kubeconfig pointing to the API server.

---

## 2. Confusion: Where Is `kubectl` Installed?

### ❓ Question

> “If I use AKS and run kubectl locally — how does that work?”

### ✔️ Explanation

* `kubectl` runs **entirely on your local machine**
* It talks to:

  * AKS API Server
  * OR your Raspberry Pi API Server
* Authentication & routing are fully controlled by **kubeconfig**

```text
kubectl (local) ──▶ API Server (remote)
```

### ✅ Practical Takeaway

You can:

* Run `kubectl` on macOS
* Control a Pi-based cluster
* Control AKS
* Switch clusters instantly using contexts

---

## 3. kube-proxy Errors on Worker Nodes

### ❌ Symptom

```text
memcache.go:265 couldn't get current server API group list
```

### 🔍 Root Cause

* kube-proxy depends on:

  * Valid kubeconfig
  * API server reachability
* Happens when:

  * Node join partially failed
  * Certificates expired
  * kube-proxy daemonset not healthy

### ✅ Workarounds

```bash
kubectl -n kube-system get pods | grep kube-proxy
kubectl -n kube-system logs kube-proxy-<pod-id>
```

If broken:

```bash
kubectl -n kube-system delete pod -l k8s-app=kube-proxy
```

✅ **Key Insight**
kube-proxy is **not optional** — without it:

* Services won’t route
* NodePorts won’t work
* iptables rules won’t get programmed

---

## 4. NodePort Created, but NGINX Page Not Loading

### ❌ Symptom

* Service exists
* NodePort allocated
* Browser shows **connection refused / timeout**

```bash
kubectl get svc
```

```text
NodePort: 30086
```

### 🔍 Root Causes (Multiple)

1. nginx pod not running
2. Pod running but:

   * Wrong selector
   * Wrong container port
3. Firewall blocking NodePort
4. kube-proxy not programming iptables

### ✅ Debug Checklist (Order Matters)

```bash
kubectl get pods -o wide
kubectl describe svc nginx
kubectl get endpoints nginx
```

Check iptables on worker:

```bash
sudo iptables -L -n | grep 30086
```

Test locally on node:

```bash
curl http://localhost:30086
```

### ✅ Common Fix

Expose correctly:

```bash
kubectl expose deployment nginx \
  --type=NodePort \
  --port=80 \
  --target-port=80
```

If this doesn't work, try from a different browser or clear browser cache.( We had it worked from safari)

---

## 5. iptables Rules Exist but Traffic Still Fails

### ❓ Observation

```text
KUBE-EXT-xxxx tcp dpt:30086
```

…but traffic still not flowing.

### 🔍 Root Cause

* iptables rules exist **but backend endpoints missing**
* kube-proxy programmed rules but pods not ready or unreachable

### ✅ Fix

```bash
kubectl get endpoints nginx
```

If empty → pod issue, not service issue.

---

## 6. Docker vs containerd Confusion

### ❓ Question

> “Should I install Docker, containerd, or both?”

### ✔️ Reality in kubeadm world

* Kubernetes **does not talk to Docker directly**
* It talks to **containerd**
* Docker installs containerd internally (legacy convenience)

### ✅ Recommendation for Learning

| Scenario     | Recommendation               |
| ------------ | ---------------------------- |
| Clean lab    | containerd only              |
| Faster setup | Docker (includes containerd) |

If switching:

```bash
sudo kubeadm reset -f
sudo systemctl restart containerd
```

---

## 7. Mixing macOS and Raspberry Pi as Workers

### ⚠️ Hidden Constraint

* macOS **cannot run kubelet natively**
* Docker Desktop provides a **VM-based node**, not a real worker

### ✅ What Actually Works

| Role         | Reality                            |
| ------------ | ---------------------------------- |
| Pi           | True worker / control plane        |
| macOS        | kubectl client                     |
| macOS worker | Only via Docker Desktop Kubernetes |

### ✔️ Best Practice

Use:

* **Pis = cluster**
* **Mac = control + kubectl + docs**

---

## 8. Control Plane vs Data Plane Traffic Confusion

### ❓ Conflicting Statements

* “All traffic goes through API server”
* “Worker nodes handle client traffic”

### ✅ Clarified Truth

| Traffic Type      | Goes Through          |
| ----------------- | --------------------- |
| kubectl / control | API Server            |
| User HTTP traffic | Worker Nodes          |
| Service routing   | kube-proxy + iptables |

✅ API Server is **not in the request path** for your app traffic.

---

## 9. Pod IP vs Node IP Confusion

### ❓ Question

> “Node has one IP — where does pod IP come from?”

### ✔️ Explanation

* Each pod gets:

  * **Virtual IP** from CNI plugin
* Node routes traffic internally

```text
Node IP: 192.168.1.10
Pod IPs:
  - 10.244.0.2
  - 10.244.0.3
```

---

## 10. Cluster Reset Hell (Things Break Randomly)

### ❌ Reality

Early Kubernetes labs **will break**.

### ✅ Safe Reset Sequence

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo iptables -F
sudo systemctl restart containerd
```

Re-init:

```bash
kubeadm init
```

---

## 11. Key Lessons Learned (Hard-Won)

* Kubernetes is **control-plane driven**, not magic
* `kubectl` issues are almost always **kubeconfig problems**
* Service issues ≠ pod issues ≠ node issues
* Debug bottom-up:
  **Pod → Service → Node → Network**
* Raspberry Pi clusters teach **real Kubernetes**, not abstractions

---

