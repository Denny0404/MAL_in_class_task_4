# k8s-infra-to-signoz (TrueNAS Docker SigNoz)

Use the SigNoz **k8s-infra** Helm chart to send Kubernetes logs/metrics/traces to a SigNoz instance
running in **Docker on TrueNAS**. Includes Ansible playbooks to install/remove the chart and a sample
**rolldice** app to verify traces.

## Repo layout
```text
ansible/
  up.yaml                 # installs/updates the k8s-infra Helm chart (and deploys rolldice)
  down.yaml               # uninstalls chart and removes rolldice
  inventory.ini           # localhost inventory (uses your current kube-context)
  group_vars/all.yml      # central variables (update TrueNAS IP etc. here)
  ansible.cfg             # local ansible configuration
helm-values/
  k8s-infra.values.yaml.j2  # Jinja template for Helm overrides (sets the upstream OTLP endpoint)
k8s/rolldice/
  deployment.yaml
  service.yaml
```

---

## Prereqs
- Kubernetes cluster (your vSphere lab is fine) and current kube-context points to it:
  ```bash
  kubectl cluster-info
  ```
- Tools installed: `kubectl`, `helm` (v3+), `ansible` (core 2.16+).
- Your **TrueNAS** host is running **SigNoz in Docker** and exposes:
  - UI on `http://<truenas-ip>:8080`
  - OTLP gRPC `4317` and OTLP HTTP `4318` (no TLS)
- Network: Kubernetes nodes/Pods can reach `<truenas-ip>` on ports `4317/4318`.

## Quick start
1) Clone this repo and update the variables in `ansible/group_vars/all.yml` (set `truenas_ip`, `cluster_name`, etc.).  
2) Install or update the chart and deploy the **rolldice** app:
   ```bash
   cd ansible
   ansible-playbook up.yaml
   ```
3) Generate some traffic to the app (inside the cluster):
   ```bash
   kubectl run -it curl --image=curlimages/curl --rm --          sh -c 'for i in $(seq 1 20); do curl -sS http://rolldice.demo.svc.cluster.local:8080/rolldice?rolls=5; sleep 0.5; done'
   ```
4) Open SigNoz UI → **Services** tab, look for `rolldice` and the Kubernetes dashboards.

## Remove
```bash
cd ansible
ansible-playbook down.yaml
```

## Notes
- The k8s-infra chart is configured to export to your TrueNAS **OTLP** endpoint via `otelCollectorEndpoint` and `otelInsecure: true`.
- The sample **rolldice** app sends traces to the **node-local Otel Agent** (hostPort 4317) deployed by k8s-infra, which forwards to SigNoz.
- If your environment blocks hostPorts, you can make the app send **directly** to the TrueNAS collector by setting env vars in `deployment.yaml`.
