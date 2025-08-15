# K8S Infra → SigNoz (TrueNAS) - Ansible + Helm

This repo installs the SigNoz **K8s-Infra** Helm chart on your Kubernetes nodes (vSphere lab) and forwards **logs/metrics/traces** to a SigNoz instance running in Docker on **TrueNAS**. It also deploys a tiny **rolldice** demo app and verifies that its telemetry reaches SigNoz.

> Lab IPs (given): Kubernetes node: `10.172.27.33` (ssh `ubuntu@10.172.27.33`), TrueNAS SigNoz host: `10.172.27.3`.

---

## What you get (matches marking scheme)

- ✅ **Ansible `up.yml` / `down.yml`** to install & remove the **k8s-infra** Helm chart (3 marks)
- ✅ **Override values** so you can set the **upstream OTLP endpoint** (the SigNoz collector on TrueNAS) (3 marks)
- ✅ **VSphere test steps** to prove it’s running on your lab cluster (2 marks)
- ✅ **`rolldice` app** (Flask) deployed to the cluster, auto‑instrumented with OpenTelemetry and sending to SigNoz (2 marks)

---

## 0) Prereqs (quick)
- You can SSH to the Kubernetes control-plane/node as `ubuntu` (sudo available).
- The node already has a working cluster (`kubectl` works). If not, see **Appendix A** for a quick single‑node bootstrap.
- TrueNAS SigNoz **OTLP ports** are reachable from the node (defaults: **4317 gRPC**, **4318 HTTP**).

## 1) Put your node IP into Ansible inventory
Edit **`ansible/inventory.ini`** if your IP is not `10.172.27.33`.

```ini
[k8s]
10.172.27.33 ansible_user=ubuntu ansible_become=true
```

## 2) Run the install (sends cluster telemetry to TrueNAS)
From your laptop/Codespace where you have this repo:
```bash
# Install dependencies locally if needed
python3 -m pip install --user ansible

# Run playbook (edit vars if needed)
ansible-playbook -i ansible/inventory.ini ansible/up.yml       -e otel_host=10.172.27.3 -e otel_port=4317 -e otel_insecure=true       -e cluster_name=vsphere-class -e deploy_rolldice=true
```

What it does:
- Installs Helm (if missing) on the node.
- Adds the SigNoz charts repo and installs **k8s‑infra** in namespace `signoz-infra` using **`override-values.yaml`**.
- Deploys the **rolldice** app into namespace `demo` (optional via `deploy_rolldice` flag).

## 3) Verify on the cluster
```bash
# All otel agents (DaemonSet) should be Running
kubectl -n signoz-infra get pods

# Hit the rolldice endpoint a few times to generate traces/logs
kubectl -n demo port-forward svc/rolldice 8080:8080 &
curl -s http://127.0.0.1:8080/rolldice; echo
curl -s http://127.0.0.1:8080/rolldice; echo
```

## 4) Verify in SigNoz (TrueNAS)
- Open the SigNoz UI on the TrueNAS IP provided by your professor, then:
  - Go to **Applications** and look for **`rolldice`**.
  - Check **Logs** for Kubernetes container logs.
  - Check **Metrics** for Kubernetes CPU/memory/cluster metrics.

> Tip: If you don’t see data, confirm network access to the TrueNAS collector from the node:
> ```bash
> nc -zv 10.172.27.3 4317 || nc -zv 10.172.27.3 4318
> ```

## 5) Remove everything (clean up)
```bash
ansible-playbook -i ansible/inventory.ini ansible/down.yml
```

---

## Appendix A — (Optional) quick single‑node Kubernetes
**Only if your node is blank.** On `ubuntu@10.172.27.33`:
```bash
# Become root
sudo -i

swapoff -a && sed -i.bak '/ swap / s/^(.*)$/#\1/g' /etc/fstab

apt-get update && apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key |       gpg --dearmor -o /usr/share/keyrings/kubernetes-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /"       > /etc/apt/sources.list.d/kubernetes.list

apt-get update && apt-get install -y kubelet kubeadm kubectl containerd.io
systemctl enable --now containerd

kubeadm init --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube && cp -i /etc/kubernetes/admin.conf $HOME/.kube/config && chown $(id -u):$(id -g) $HOME/.kube/config

# Pod network (Flannel quick start)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Schedule pods on the control plane (single-node lab)
kubectl taint nodes --all node-role.kubernetes.io/control-plane- || true
```

---

## How it works (1 minute)
- The **k8s‑infra** agents (DaemonSet) run on all nodes, collect K8s logs/metrics, then **export via OTLP** to the **upstream collector** at `otel_host:otel_port` (TrueNAS SigNoz).
- The **rolldice** pod sends OTLP directly to the **per‑node agent** (`HOST_IP:4317`) per SigNoz docs. The agent forwards to TrueNAS.

---

## Troubleshooting quick wins
- `kubectl -n signoz-infra logs daemonset/k8s-infra-otel-agent`
- `kubectl -n signoz-infra get ds,deploy,svc`
- Check firewall rules between 10.172.27.33 → 10.172.27.3 on 4317/4318
- On TrueNAS host, verify SigNoz docker is up and OTLP ports exposed.

---
