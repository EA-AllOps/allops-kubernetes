# allops-kubernetes

### Setup self-managed kubernetes cluster:
- Create EC2 Ubuntu 24.04, one control-plane EC2 instance, two worker EC2 instances, kubeadm, containerd, and Calico.
Before running commands, create three EC2 instances in the same VPC/subnet:
- k8s-cp-1: t3.large or larger
- k8s-worker-1, k8s-worker-2: t3.medium or larger
- Ubuntu 24.04 LTS
- A shared security group:
  - Allow all inbound traffic from that same security group.
  - Allow SSH (TCP 22) only from your public IP.
  - Do not expose Kubernetes API port 6443 to the whole internet.
- Instances need outbound internet access, through public IPs or a NAT gateway.
Run this setup on all three instances:

```sh
cat <<'EOF' | sudo tee /usr/local/sbin/install-k8s-node.sh >/dev/null
#!/usr/bin/env bash
set -euo pipefail

swapoff -a
sed -ri '/\sswap\s/s/^/#/' /etc/fstab

cat <<'MODULES' | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
MODULES

modprobe overlay
modprobe br_netfilter

cat <<'SYSCTL' | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
SYSCTL

sysctl --system

apt-get update
apt-get install -y ca-certificates curl gpg containerd

mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl enable --now containerd
systemctl restart containerd

mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.37/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

cat <<'REPO' | tee /etc/apt/sources.list.d/kubernetes.list
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.37/deb/ /
REPO

apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
systemctl enable --now kubelet
EOF

sudo chmod +x /usr/local/sbin/install-k8s-node.sh
sudo /usr/local/sbin/install-k8s-node.sh
```

On the control-plane instance, find its private EC2 IP:
```
hostname -I
```
Then initialize Kubernetes, replacing the IP:
```
CONTROL_PLANE_IP="<ip>"

sudo kubeadm init \
  --apiserver-advertise-address="$CONTROL_PLANE_IP" \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket=unix:///run/containerd/containerd.sock
```
Configure kubectl on that control-plane instance:

```
mkdir -p "$HOME/.kube"
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
```
Install Calico networking:
```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/calico.yaml
kubectl get pods -n kube-system -w
```
When calico-node is running, get a fresh worker join command:
```
kubeadm token create --print-join-command
```
Run the resulting command on each worker, for example:
```
sudo kubeadm join <IP>:6443 \
  --token YOUR_TOKEN \
  --discovery-token-ca-cert-hash sha256:YOUR_HASH \
  --cri-socket=unix:///run/containerd/containerd.sock
```
Verify from the control plane:
```
kubectl get nodes -o wide
kubectl get pods -A
```





