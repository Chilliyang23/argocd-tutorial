# Modern Argo CD Tutorial: GitOps Made Simple

This tutorial is a **streamlined, reliable, and modernized** guide for learning Argo CD, GitOps, and Kubernetes deployments in real-world, production-like environments.

---

## 🔧 What You'll Build

- A GitOps-driven Kubernetes deployment workflow
- Argo CD installed on a production-grade Kubernetes cluster
- Kubernetes cluster provisioned using **kubeadm** on an EC2 instance

---

## ☁️ Why Use AWS EC2 + Kubeadm

- ✅ Avoids port-forwarding/gRPC issues on local tools
- ✅ Public IP makes Argo CD UI and CLI easily accessible
- ✅ Mirrors production-grade on-prem Kubernetes setups
- ✅ Teaches you the industry-standard cluster bootstrapping method

This guide can also be applied to any VPS, private cloud, or bare-metal server.

---

## 🚀 Step 1: Provision an EC2 Instance

1. Launch an **Ubuntu 22.04 or 24.04 t2.medium** EC2 instance (or larger).
2. Open ports **22, 80, 443, 6443, 8080** in the Security Group.
3. SSH into your EC2 instance:
   ```bash
   ssh -i my-key.pem ubuntu@<your-ec2-public-ip>
   ```

---

## 🛠️ Step 2: Install Kubernetes with `kubeadm`

### Why Use `containerd` Instead of Docker?

Docker used to be the default container runtime for Kubernetes, but has been deprecated in favor of **containerd**, which is now the industry standard.

- `containerd` is lighter, faster, and has fewer moving parts.
- It is the core container runtime behind Docker — Kubernetes now talks directly to it.
- In real-world deployments, Kubernetes interacts with containers via CRI (Container Runtime Interface), and `containerd` is CRI-compatible.
- You’ll mostly use `kubectl` for day-to-day work. Direct `containerd` CLI (`ctr` or `crictl`) is rarely needed except for debugging or advanced node operations.

### Install containerd and Prerequisites:

```bash
sudo apt update && sudo apt install -y \
  apt-transport-https ca-certificates curl gnupg lsb-release

# Install containerd
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd
```

> ✅ After installing, verify containerd is running:

```bash
sudo systemctl status containerd
```

If you later see `InvalidDiskCapacity` or node `NotReady`, restart and clear:

```bash
sudo rm -rf /var/lib/containerd/*
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

> ⚠️ **Important:** Do not restart `containerd` or `kubelet` during or immediately after `kubeadm init`. Let the process finish and ensure the node is registered before restarting anything.

### Enable IP forwarding (required for kubeadm):

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo bash -c 'echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf'
```

> ✅ Kubernetes requires this to route traffic between pods and services.

### Install Kubernetes CLI tools (Ubuntu 22.04+):

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  gpg --dearmor | sudo tee /etc/apt/keyrings/kubernetes-apt-keyring.gpg > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | \
sudo tee /etc/apt/sources.list.d/kubernetes.list > /dev/null

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

> ✅ These steps ensure compatibility with newer Ubuntu versions and use the latest Kubernetes release.

### Why Disable Swap?

Kubernetes requires that **swap be disabled** to ensure consistent and predictable performance.

- The Kubernetes scheduler assumes that available memory is actual RAM, not virtual.
- Swap can cause resource accounting issues, node instability, and unexpected pod eviction.

Disable swap temporarily and permanently:

```bash
sudo swapoff -a                      # Disable swap now
sudo sed -i '/ swap / s/^/#/' /etc/fstab  # Disable swap on boot
```

---

## 🧱 Step 3: Initialize Kubernetes Cluster

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

This sets up the Kubernetes control plane. The `--pod-network-cidr` flag is required for the Calico plugin.

> ✅ You can customize the CIDR range, but `/16` is the standard size (\~65,000 IPs). Avoid `/8` or `/32` as they may conflict with host routing.

> ⚠️ If you encounter node identity errors (e.g., `system:node:<hostname>` cannot access pods), reset cleanly:

```bash
sudo kubeadm reset --force
sudo rm -rf /etc/cni /var/lib/cni /var/lib/kubelet /etc/kubernetes /var/lib/containerd/*
sudo systemctl restart containerd kubelet
```

Then re-run the `kubeadm init` step.

### Configure kubectl for your user:

These steps allow your normal user to run `kubectl` without needing `sudo` or `--kubeconfig`.

- `$HOME` is your user’s home directory (e.g. `/home/ubuntu`).
- `~/.kube/config` is where `kubectl` expects to find credentials.

```bash
mkdir -p $HOME/.kube                     # Create kube config directory
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config   # Copy kubeconfig
sudo chown $(id -u):$(id -g) $HOME/.kube/config            # Give correct ownership
```

### What Is a Network Plugin (Calico)?

Kubernetes doesn’t provide built-in networking. It expects you to install a **CNI (Container Network Interface)** plugin.

**Calico**:

- Enables pod-to-pod communication across nodes
- Implements **Network Policies** (like firewalls)
- Routes internal traffic across your cluster

Without a plugin like Calico, your pods can’t talk to each other, and Kubernetes won’t work correctly.

Install Calico:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

> 🧪 If you get errors like `connect: connection refused` or CRDs failing to apply, wait 10–30 seconds and retry. This happens if the API server is still stabilizing after init.

> 💡 To monitor:

```bash
watch kubectl get pods -n kube-system
```

---

## 🚦 Step 4: Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Expose Argo CD with insecure HTTP mode:

```bash
kubectl patch configmap argocd-cmd-params-cm -n argocd --type merge \
  -p '{"data":{"server.insecure":"true","server.listen":":8080"}}'

kubectl rollout restart deployment argocd-server -n argocd
```

> 🔁 `kubectl rollout restart` forces the Argo CD deployment to restart and reload its config changes.

---

## 🌐 Step 5: Access Argo CD UI

Visit:

```bash
http://<your-ec2-public-ip>:8080
```

Retrieve password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Login via CLI:

```bash
argocd login <your-ec2-public-ip>:8080 \
  --username admin --password <password> --insecure
```

---

## 📦 Step 6: Deploy an Application via GitOps

Sample app (optional):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

Deploy using Argo CD:

```bash
argocd app create nginx-app \
  --repo https://github.com/<your-username>/<your-repo> \
  --path apps \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

argocd app sync nginx-app
```

> 🔓 If your Git repo is public, Argo CD does **not** require authentication. For private repos, you must register credentials using `argocd repo add`.

---

## 🎉 You're Done!

- Access Argo CD UI at: `http://<your-ec2-ip>:8080`
- Login as `admin` with initial password
- Manage apps visually or via CLI

---

## 🧠 Summary

| Feature            | Kind (Dev/Test) | Kubeadm (Production-like) |
| ------------------ | --------------- | ------------------------- |
| Portability        | High            | Medium                    |
| Production realism | ❌               | ✅                         |
| Argo CD networking | Requires tweaks | Public IP, no issues      |
| Cluster management | Simple          | Realistic                 |

---

## ✅ Next Steps

- Set up HTTPS with Ingress and cert-manager
- Use `kubeadm join` to add worker nodes
- Explore High Availability with etcd and load balancer
- Automate Argo CD app creation with GitHub Actions

