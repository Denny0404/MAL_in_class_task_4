
# 🚀 K8S Infra → SigNoz on TrueNAS (Class Assignment)

> **Goal:** Use the **K8S Infra Helm chart** to send **Kubernetes logs/metrics** to **SigNoz** running on **TrueNAS**, controlled by **Ansible**.  
> You’ll deploy on a vSphere Ubuntu VM, verify with a tiny **rolldice** app, and capture screenshots for grading.

---

## 🧭 Index

- 📚 [Overview](#-overview)  
- 🗂️ [Repo Structure](#️-repo-structure)  
- 🧩 [Prereqs & IPs](#-prereqs--ips)  
- 🛠️ [Step A — Prepare the Repo (Codespaces)](#️-step-a--prepare-the-repo-codespaces)  
- 🚀 [Step B — Deploy k8s-infra via Ansible to your Kubernetes](#-step-b--deploy-k8s-infra-via-ansible-to-your-kubernetes)  
  - ⚙️ [Install/Use MicroK8s (fast path)](#️-installuse-microk8s-fast-path)  
  - 🧽 [Fix apt “cdrom” warning](#-fix-apt-cdrom-warning)  
  - 🧪 [vSphere Proof](#-vsphere-proof)  
- 🎲 [Step C — Add the rolldice app](#-step-c--add-the-rolldice-app)  
- 🖥️ [Step D — SigNoz on TrueNAS: expose UI + OTLP](#️-step-d--signoz-on-truenas-expose-ui--otlp)  
  - 🐳 [Docker/Compose path](#-dockercompose-path)  
  - 📦 [TrueNAS Apps (k3s) path](#-truenas-apps-k3s-path)  
- 🌐 [Open SigNoz UI from your laptop (SSH tunnel)](#-open-signoz-ui-from-your-laptop-ssh-tunnel)  
- 🔌 [Point k8s-infra at the correct upstream collector](#-point-k8s-infra-at-the-correct-upstream-collector)  
- ✅ [What to Submit (Checklist)](#-what-to-submit-checklist)  
- 🧯 [Troubleshooting](#-troubleshooting)  
- 🧹 [Uninstall / Clean-up](#-uninstall--clean-up)  
- 📎 [Appendix: Ansible Playbooks](#-appendix-ansible-playbooks)

---

## 📚 Overview

You’ll:
1. Add **Ansible playbooks** to install/uninstall the **k8s-infra** Helm chart.  
2. Override values to send telemetry to **SigNoz** (on **TrueNAS** at `10.172.27.3`).  
3. Prove it works on **vSphere** (Ubuntu VM).  
4. Deploy a tiny **rolldice** app and see logs/metrics in SigNoz.

> **Grading rubric match:**
> - ✅ Repo with `up.yml`/`down.yml` and values override (3 marks)  
> - ✅ Override upstream OTLP collector (3 marks)  
> - ✅ Test on vSphere (2 marks)  
> - ✅ Add `rolldice` and verify telemetry (2 marks)

---

## 🗂️ Repo Structure

Inside your class repo (example: `MAL_in_class_task_4`):

```
k8s-infra-signoz-ansible/
└─ ansible/
   ├─ inventory.ini
   ├─ up.yml
   ├─ down.yml
   └─ values-k8s-infra.yaml.j2
```

- **`up.yml`**: Adds Helm repo, creates namespace, renders values file, installs/updates chart.  
- **`down.yml`**: Uninstalls the release + namespace clean-up.  
- **`values-k8s-infra.yaml.j2`**: Lets you set `signoz_otlp_endpoint`, `signoz_otlp_insecure`, `cluster_name`, etc.  
- **`inventory.ini`**: Target host(s). For one-box MicroK8s, use `localhost ansible_connection=local`.

---

## 🧩 Prereqs & IPs

- Ubuntu VM on vSphere (Kubernetes control-plane or single-node **MicroK8s**).  
- **TrueNAS IP:** `10.172.27.3` (SigNoz lives here, but ports may not be open by default).  
- **Ubuntu VM IP:** `10.172.27.30`  
- Access from your laptop to `10.172.27.30` (for SSH tunneling).

---

## 🛠️ Step A — Prepare the Repo (Codespaces)

> You already committed the folder, but here’s the minimal reference in case you need it again.

```bash
# In Codespaces terminal, create layout
mkdir -p ~/k8s-infra-signoz-ansible/ansible
cat > ~/k8s-infra-signoz-ansible/ansible/up.yml <<'YAML'
# (Your up.yml content; already present in repo)
YAML

cat > ~/k8s-infra-signoz-ansible/ansible/down.yml <<'YAML'
# (Your down.yml content; shown in Appendix below)
YAML

cat > ~/k8s-infra-signoz-ansible/ansible/values-k8s-infra.yaml.j2 <<'J2'
# (Your values template; shown in Appendix below)
J2

# Move into your class repo and commit
cd /workspaces/MAL_in_class_task_4
mv ~/k8s-infra-signoz-ansible ./k8s-infra-signoz-ansible
rm -rf ./k8s-infra-signoz-ansible/.git
git add k8s-infra-signoz-ansible
git commit -m "add k8s-infra-signoz-ansible (ansible up/down + values override)"
git push
```

---

## 🚀 Step B — Deploy k8s-infra via Ansible to your Kubernetes

### 1) SSH to the Ubuntu VM
From your laptop:
```bash
ssh ubuntu@10.172.27.30
```

### 2) Tools on the VM
```bash
sudo apt-get update -y

# Fix "cdrom" apt warning if it appears
sudo sed -i 's|^deb cdrom:|# deb cdrom:|' /etc/apt/sources.list || true
sudo apt-get update -y

# Base tools
sudo apt-get install -y ansible curl ca-certificates apt-transport-https netcat-openbsd git

# helm
if ! command -v helm >/dev/null; then \
  curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash; \
fi

# kubectl (official repo)
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update -y
sudo apt-get install -y kubectl
```

### ⚙️ Install/Use MicroK8s (fast path)
If your VM is not already a Kubernetes control-plane, use **MicroK8s**:

```bash
sudo snap install microk8s --channel=1.30/stable --classic
sudo usermod -a -G microk8s $USER
sudo chown -R "$USER":"$USER" ~/.kube || true
newgrp microk8s
microk8s enable dns hostpath-storage metrics-server
mkdir -p ~/.kube && microk8s config > ~/.kube/config
kubectl get nodes
```

Ensure it becomes **Ready**. Then make it work for sudo as well (Ansible uses `become`):
```bash
sudo mkdir -p /root/.kube
sudo cp -f ~/.kube/config /root/.kube/config
sudo kubectl get nodes
```

### 🧽 Fix apt “cdrom” warning
If you still see it:
```bash
sudo bash -lc "sed -i '/cdrom:/ s/^/# /' /etc/apt/sources.list /etc/apt/sources.list.d/*.list 2>/dev/null || true"
sudo sed -i '/^deb .*file:\/cdrom/d' /etc/apt/sources.list
sudo apt-get update -y
```

### 3) Pull your repo & set inventory
```bash
cd ~
git clone https://github.com/<YOUR_GH_USERNAME>/MAL_in_class_task_4.git
cd MAL_in_class_task_4/k8s-infra-signoz-ansible

# Run Ansible locally
printf "[k8s_control]\nlocalhost ansible_connection=local\n" | tee ansible/inventory.ini
```

### 4) (Optional) Test connectivity to TrueNAS
```bash
nc -zv 10.172.27.3 4317 || true
nc -zv 10.172.27.3 4318 || true
```

> It’s OK if these fail initially; we’ll open them on TrueNAS later.

### 5) Install/Upgrade k8s-infra (points at TrueNAS)
```bash
ansible-playbook -i ansible/inventory.ini ansible/up.yml \
  -e "signoz_otlp_endpoint=10.172.27.3:4317 signoz_otlp_insecure=true \
      cluster_name=vsphere-k8s deployment_environment=dev \
      namespace=signoz-infra release_name=k8s-infra"
```

### 6) Verify
```bash
kubectl get pods -n signoz-infra
helm list -n signoz-infra
kubectl get deploy,ds -n signoz-infra
kubectl -n signoz-infra logs deploy/k8s-infra-otel-deployment --tail=80 | sed -n '1,80p'
```
If OTLP ports aren’t open yet on TrueNAS, you’ll see **connection refused** to `10.172.27.3:4317` — that **proves your override works**.

---

## 🧪 vSphere Proof

```bash
sudo dmidecode -s system-manufacturer
sudo dmidecode -s system-product-name
systemd-detect-virt

kubectl get nodes -o wide
kubectl top nodes
kubectl top pods -A | head
```

> **Screenshots** of these outputs demonstrate you’re running on vSphere and that cluster metrics are flowing.

---

## 🎲 Step C — Add the rolldice app

```bash
kubectl create ns apps --dry-run=client -o yaml | kubectl apply -f -

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolldice
  namespace: apps
  labels: { app: rolldice }
spec:
  replicas: 1
  selector:
    matchLabels: { app: rolldice }
  template:
    metadata: { labels: { app: rolldice } }
    spec:
      containers:
        - name: rolldice
          image: busybox:1.36
          args: ["/bin/sh","-c","awk 'BEGIN{srand(); while(1){ printf(\"rolldice value=%d\\n\", 1+int(rand()*6)); system(\"sleep 2\")}}'"]
          resources:
            requests: { cpu: '10m', memory: '16Mi' }
            limits:   { cpu: '50m', memory: '64Mi' }
EOF

kubectl -n apps rollout status deploy/rolldice
kubectl -n apps logs deploy/rolldice --tail=20
```

> You should see: `rolldice value=…` lines. k8s-infra collects these and attempts to ship to SigNoz.

---

## 🖥️ Step D — SigNoz on TrueNAS: expose UI + OTLP

You must **open ports on TrueNAS** so the Ubuntu VM can send to SigNoz and you can view the UI.

### 🐳 Docker/Compose path
In the **TrueNAS shell** (or Web UI → System Settings → Shell):

```bash
cd /root/signoz || cd /opt/signoz || pwd
docker compose up -d
docker ps --format 'table {{.Names}}\t{{.Ports}}' | egrep 'collector|front'

# If ports are not mapped, edit docker-compose.yml and add:
# otel-collector:
#   ports: ["4317:4317","4318:4318"]
# frontend:
#   ports: ["8080:3301"]
docker compose up -d
```

- **UI:** `http://10.172.27.3:8080`  
- **OTLP gRPC:** `10.172.27.3:4317`  
- **OTLP HTTP:** `10.172.27.3:4318`

### 📦 TrueNAS Apps (k3s) path
If SigNoz is installed as an **App**:

```bash
# Find the namespace
k3s kubectl get ns | grep -i signo
NS=<your-namespace>

# Expose collector + UI as NodePorts
k3s kubectl -n $NS patch svc otel-collector -p '{
  "spec":{"type":"NodePort",
  "ports":[
    {"name":"otlp-grpc","port":4317,"targetPort":4317,"nodePort":30317},
    {"name":"otlp-http","port":4318,"targetPort":4318,"nodePort":30318}
  ]}}'

k3s kubectl -n $NS patch svc signoz-frontend -p '{
  "spec":{"type":"NodePort",
  "ports":[{"name":"ui","port":3301,"targetPort":3301,"nodePort":30080}]}}'

k3s kubectl -n $NS get svc otel-collector signoz-frontend -o wide
```

- **UI:** `http://10.172.27.3:30080`  
- **OTLP gRPC:** `10.172.27.3:30317`  
- **OTLP HTTP:** `10.172.27.3:30318`

---

## 🌐 Open SigNoz UI from your laptop (SSH tunnel)

> Codespaces cannot reach 10.x; use your **laptop** as the browser.

**Tunnel to TrueNAS via the Ubuntu VM (jump host):**
```bash
# If using Docker/Compose (UI on 8080)
ssh -L 8080:10.172.27.3:8080 ubuntu@10.172.27.30 -N
# If using Apps (NodePort UI 30080)
ssh -L 30080:10.172.27.3:30080 ubuntu@10.172.27.30 -N
```

Open in your laptop browser:  
- `http://localhost:8080` **or** `http://localhost:30080`

In SigNoz: **Logs → Explorer** (search `rolldice`), **Dashboards → Kubernetes**.

---

## 🔌 Point k8s-infra at the correct upstream collector

If using **Docker/Compose** on TrueNAS (host ports `4317/4318`):
```bash
ansible-playbook -i ansible/inventory.ini ansible/up.yml \
  -e "signoz_otlp_endpoint=10.172.27.3:4317 signoz_otlp_insecure=true \
      cluster_name=vsphere-k8s deployment_environment=dev \
      namespace=signoz-infra release_name=k8s-infra"
```

If using **TrueNAS Apps (k3s NodePort `30317`)**:
```bash
ansible-playbook -i ansible/inventory.ini ansible/up.yml \
  -e "signoz_otlp_endpoint=10.172.27.3:30317 signoz_otlp_insecure=true \
      cluster_name=vsphere-k8s deployment_environment=dev \
      namespace=signoz-infra release_name=k8s-infra"
```

---

## ✅ What to Submit (Checklist)

- 📦 **Repo proof**: screenshot showing `up.yml`, `down.yml`, `values-k8s-infra.yaml.j2`  
- 🧭 **Ansible install output**: the `PLAY RECAP` + k8s pods listed  
- 📈 **Pods running**: `kubectl get pods -n signoz-infra` shows both **Running**  
- 🔧 **Override proof**: logs from `k8s-infra-otel-deployment` showing attempts to `10.172.27.3:<port>`  
- 🧪 **vSphere proof**: outputs of `dmidecode`, `systemd-detect-virt`, `kubectl get nodes -o wide`, `kubectl top`  
- 🎲 **rolldice**: `kubectl -n apps logs deploy/rolldice --tail=20`  
- 🖥️ **SigNoz UI**: a screenshot of Logs/Dashboards (when ports are exposed)

---

## 🧯 Troubleshooting

- **Pods stuck in `ContainerCreating`**  
  Check events:  
  ```bash
  kubectl -n signoz-infra describe pod/<pod>
  kubectl -n signoz-infra get events --sort-by=.lastTimestamp | tail -n 50
  ```

- **Exporter shows `connection refused`**  
  TrueNAS OTLP not open yet. Fix ports (see Step D), then re-run `up.yml` with the correct endpoint.

- **Cannot SSH to TrueNAS (`connection refused`)**  
  Use the **TrueNAS Web UI** and its **Shell** instead, or ask your professor to enable SSH.

- **Codespaces cannot see 10.x**  
  Always use **laptop + SSH tunnel** through `10.172.27.30`.

- **Apt `cdrom` warning**  
  See “Fix apt cdrom warning” section above.

---

## 🧹 Uninstall / Clean-up

### Remove k8s-infra
```bash
ansible-playbook -i ansible/inventory.ini ansible/down.yml \
  -e "namespace=signoz-infra release_name=k8s-infra"
```

### Remove rolldice
```bash
kubectl delete deploy rolldice -n apps --ignore-not-found=true
kubectl delete ns apps --ignore-not-found=true
```

---

## 📎 Appendix: Ansible Playbooks

### `ansible/up.yml` (example pattern)
```yaml
- name: Install or upgrade SigNoz k8s-infra
  hosts: k8s_control
  become: yes
  vars:
    release_name: "{{ release_name | default('k8s-infra') }}"
    namespace: "{{ namespace | default('signoz-infra') }}"
  tasks:
    - name: Ensure Helm repo present/updated
      ansible.builtin.shell: |
        helm repo add signoz https://charts.signoz.io || true
        helm repo update
      args: { executable: /bin/bash }

    - name: Create namespace (idempotent)
      ansible.builtin.shell: "kubectl create ns {{ namespace }} --dry-run=client -o yaml | kubectl apply -f -"

    - name: Render override values
      ansible.builtin.template:
        src: values-k8s-infra.yaml.j2
        dest: /tmp/values-k8s-infra.yaml

    - name: Install/Upgrade k8s-infra
      ansible.builtin.shell: |
        helm upgrade --install {{ release_name }} signoz/k8s-infra \
          -n {{ namespace }} -f /tmp/values-k8s-infra.yaml --create-namespace
      args: { executable: /bin/bash }

    - name: Show pods
      ansible.builtin.shell: "kubectl get pods -n {{ namespace }}"
      register: pods_out

    - debug:
        var: pods_out.stdout_lines
```

### `ansible/down.yml`
```yaml
- name: Remove k8s-infra
  hosts: k8s_control
  become: yes
  vars:
    release_name: "{{ release_name | default('k8s-infra') }}"
    namespace: "{{ namespace | default('signoz-infra') }}"
  tasks:
    - name: Uninstall release (ignore if not found)
      ansible.builtin.shell: "helm uninstall {{ release_name }} -n {{ namespace }} || true"
      args: { executable: /bin/bash }

    - name: Delete namespace (ignore if not found)
      ansible.builtin.shell: "kubectl delete ns {{ namespace }} --ignore-not-found=true"
      args: { executable: /bin/bash }
```

### `ansible/values-k8s-infra.yaml.j2` (key parts)
```yaml
clusterName: "{{ cluster_name | default('vsphere-k8s') }}"
environment: "{{ deployment_environment | default('dev') }}"

logsCollection:
  enabled: true

metricsCollection:
  enabled: true

tracesCollection:
  enabled: false  # can be true later if you add tracing

otelCollector:
  exporters:
    otlp:
      endpoint: "{{ signoz_otlp_endpoint | default('10.172.27.3:4317') }}"
      tls:
        insecure: {{ signoz_otlp_insecure | default(true) }}
```

> You can add more processors/receivers as needed; above is the minimal override to point upstream to SigNoz.

---

**🎉 That’s it!** Capture the screenshots listed in the **Checklist**, push your repo updates, and you’re done.
