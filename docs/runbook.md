# Runbook

The commands I ran, in order, as I ran them. No wrapper scripts. If a step is repeated on several nodes it is
written once and I ran it on each node over SSH.

## Nodes

| Name | Role | Public IP | Size | Image |
|---|---|---|---|---|
| cp-1 | control plane | 65.108.33.68 | CPU.4V.16G (AMD Rome+) | Ubuntu 24.04 Minimal |
| worker-1 | worker | 65.108.32.164 | CPU.4V.16G | Ubuntu 24.04 Minimal |
| worker-2 | worker | 65.108.33.95 | CPU.4V.16G | Ubuntu 24.04 Minimal |

All in FIN-01. Verda only offers public IPs there (private networking is FIN-03 only), so node-to-node traffic
goes over the public interface and the host firewall does the work. Login is `root@<ip>` with the SSH key
registered in the console.

## 1. Node prep (every node)

```sh
swapoff -a; sed -i '/ swap / s/^/#/' /etc/fstab

cat >/etc/modules-load.d/k8s.conf <<EOF2
overlay
br_netfilter
EOF2
modprobe overlay; modprobe br_netfilter

# ip_forward for the CNI. The other three are what kubelet protectKernelDefaults=true insists on;
# the Verda image ships them at 0 and kubelet refuses to start otherwise (see worklog, 1 Nov).
cat >/etc/sysctl.d/k8s.conf <<EOF2
net.ipv4.ip_forward = 1
vm.overcommit_memory = 1
kernel.panic = 10
kernel.panic_on_oops = 1
EOF2
sysctl --system

# sshd: keys only. The image allows password auth by default.
cat >/etc/ssh/sshd_config.d/10-key-only.conf <<EOF2
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password
PubkeyAuthentication yes
EOF2
sshd -t && systemctl reload ssh

apt-get update
apt-get install -y apt-transport-https ca-certificates curl gpg containerd nftables cri-tools
mkdir -p /etc/containerd
containerd config default | sed 's/SystemdCgroup = false/SystemdCgroup = true/' >/etc/containerd/config.toml
systemctl restart containerd && systemctl enable containerd

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' >/etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
systemctl enable kubelet
```

## 2. Control plane (cp-1)

```sh
mkdir -p /etc/kubernetes/audit /etc/kubernetes/enc /var/log/kubernetes
scp infra/kubeadm-config.yaml infra/audit-policy.yaml root@65.108.33.68:/root/
mv /root/audit-policy.yaml /etc/kubernetes/audit/policy.yaml

# secrets-at-rest key is generated on the host and never leaves it. infra/encryption.yaml.example shows the shape.
scp infra/encryption.yaml.example root@65.108.33.68:/root/
KEY=$(head -c 32 /dev/urandom | base64)
sed "s|<base64 32 bytes.*>|$KEY|" /root/encryption.yaml.example > /etc/kubernetes/enc/encryption.yaml
chmod 600 /etc/kubernetes/enc/encryption.yaml

kubeadm init --config /root/kubeadm-config.yaml --upload-certs
mkdir -p ~/.kube && cp /etc/kubernetes/admin.conf ~/.kube/config
```

## 3. Workers

```sh
# on cp-1, print a join command (token lives 24h, not committed)
kubeadm token create --print-join-command
# on each worker
kubeadm join 65.108.33.68:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --node-name worker-N
# kubelet serverTLSBootstrap=true means serving certs need approval once per node
kubectl get csr; kubectl certificate approve <pending csr names>
```

## 4. Cilium (from the laptop, kubeconfig copied from /etc/kubernetes/admin.conf)

```sh
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.20.4 -n kube-system -f bootstrap/cilium/values.yaml
cilium status --wait
```

## 5. Host firewall (every node)

```sh
scp infra/nftables.conf root@<ip>:/etc/nftables.conf
nft -c -f /etc/nftables.conf && nft -f /etc/nftables.conf && systemctl enable nftables
```

## 6. Argo CD (from the laptop)

```sh
# read-only deploy key for the repo; the private half goes into the cluster, nowhere else
ssh-keygen -t ed25519 -N '' -C 'argocd read-only deploy key' -f argocd_deploy_key
gh repo deploy-key add argocd_deploy_key.pub -R ToluGIT/verda-k8s-hardening -t argocd
kubectl create namespace argocd
kubectl -n argocd create secret generic repo-verda-k8s-hardening \
  --from-literal=type=git --from-literal=url=git@github.com:ToluGIT/verda-k8s-hardening.git \
  --from-file=sshPrivateKey=argocd_deploy_key
kubectl -n argocd label secret repo-verda-k8s-hardening argocd.argoproj.io/secret-type=repository
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd --version 10.11.6 -n argocd -f bootstrap/argocd/values.yaml
kubectl apply -f bootstrap/argocd/root-app.yaml      # from here on Argo CD owns everything under platform/
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

## 7. Reviewer access

```sh
# cluster: the reviewer ServiceAccount and ClusterRole are in platform/reviewer/ (synced by Argo CD)
kubectl -n reviewer create token reviewer --duration=720h        # goes into the reviewer kubeconfig, server 127.0.0.1:6443
# grafana: a Viewer user, created through the API with the admin password from the monitoring-grafana secret
# argo cd: the local account `reviewer` (bootstrap/argocd/values.yaml) mapped to role:readonly; set its password:
#   argocd account update-password --account reviewer   (as admin, over a port-forward)
# ssh: the reviewer public key on cp-1 only, restricted to the API server tunnel
echo 'restrict,port-forwarding,permitopen="127.0.0.1:6443",command="/bin/false" ssh-ed25519 AAAA... verda reviewer' >> /root/.ssh/authorized_keys
```

## 8. Teardown

Delete the three instances in the Verda console. Revoke the reviewer token and the deploy key.
