# k8s-LLM

Kubernetes manifests for the 2LMlocal platform — an on-premise AI platform running local language models.

This repo is managed by ArgoCD. Every change pushed here is automatically reconciled with the cluster.

---

## Repository Structure

```
k8s/
├── metalLB/
│   ├── ipaddresspool.yaml          # IP pool 192.168.0.20-40
│   ├── l2advertisement.yaml        # L2 advertisement config
│   ├── argocd-ingress.yaml         # IngressRoute → argocd.local
│   ├── grafana-ingress.yaml        # IngressRoute → grafana.local
│   ├── open-webui-ingress.yaml     # IngressRoute → openwebui.local
│   ├── prometheus-ingress.yaml     # IngressRoute → prometheus.local
│   └── registry-ingress.yaml      # IngressRoute → registry.local
├── ollama/
│   ├── pv.yaml                     # PersistentVolume — NFS from Proxmox host
│   ├── pvc.yaml                    # PVC NFS models (40Gi, ReadWriteMany)
│   ├── pvc-data.yaml               # PVC local data (15Gi, local-path)
│   ├── deployment.yaml             # Ollama deployment (pinned to llm-worker)
│   └── service.yaml                # ClusterIP on port 11434
├── ollama-exporter/
│   ├── deployment.yaml             # Ollama exporter proxy + metrics
│   ├── service.yaml                # ClusterIP on port 8000
│   └── servicemonitor.yaml         # Prometheus ServiceMonitor
├── open-webui/
│   ├── pvc.yaml                    # PVC (15Gi, local-path)
│   ├── deployment.yaml             # Open WebUI → ollama-exporter:8000
│   └── service.yaml                # ClusterIP on port 8080
└── registry/
    ├── pvc.yaml                    # PVC (20Gi, local-path)
    ├── deployment.yaml             # Docker Registry (pinned to registry node)
    └── service.yaml                # NodePort 30964
```

---

## Prerequisites

Before applying these manifests the following must be in place:

### Cluster
- Kubernetes 1.35.1+
- Calico CNI
- Helm installed on control-plane

### Helm components

```bash
# MetalLB
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb -n metallb --create-namespace

# Traefik
helm repo add traefik https://traefik.github.io/charts
helm install traefik traefik/traefik -n kube-system

# Prometheus + Grafana
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace --server-side

# ArgoCD
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace
```

### StorageClass

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

### NFS on Proxmox host

```bash
apt install nfs-kernel-server -y
echo "/usr/share/ollama/.ollama 192.168.0.0/24(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
exportfs -ra
```

### NFS client on all worker nodes

```bash
sudo apt install nfs-common -y
```

### Node taints

```bash
# registry node
kubectl taint nodes registry dedicated=registry:NoSchedule
```

### Docker Registry — build and push images

```bash
# on lab2 (Proxmox host)
git clone https://github.com/frcooper/ollama-exporter.git
cd ollama-exporter
docker build -t 192.168.0.22:30964/ollama-exporter:latest .
docker push 192.168.0.22:30964/ollama-exporter:latest
```

### DNS on client machines

```
# /etc/hosts
192.168.0.201  openwebui.local
192.168.0.201  grafana.local
192.168.0.201  prometheus.local
192.168.0.201  argocd.local
192.168.0.22   registry.local
```

---

## ArgoCD Applications

| Application | Path | Namespace |
|---|---|---|
| metallb | metalLB | metallb |
| k8s-llm-ollama | ollama | ollama |
| k8s-llm-openwebui | open-webui | open-webui |
| ollama-exporter | ollama-exporter | ollama |
| registry | registry | registry |

---

## Components

### MetalLB

Provides external IP addresses to LoadBalancer services (Traefik).

- IP pool: `192.168.0.20 — 192.168.0.40`
- All IngressRoutes are managed in this folder

> Important: use explicit range, not CIDR /24 — causes ARP conflicts.

### Ollama

LLM backend. Exposes OpenAI-compatible REST API on port 11434.

- Pinned to `llm-worker` via `nodeSelector`
- Models served via NFS from `192.168.0.44:/usr/share/ollama/.ollama`
- Local data stored in a separate `local-path` PVC
- CPU-only inference (GPU support planned)

Models available:
- `mistral:7b-instruct-q4_K_M`
- `qwen2.5:7b-instruct`

### Ollama Exporter

Transparent proxy between Open WebUI and Ollama. Collects metrics and exposes them to Prometheus.

```
Open WebUI → ollama-exporter:8000 → Ollama:11434
                    ↓
             /metrics → Prometheus
```

Metrics collected:
- `ollama_requests_total` — total requests per model
- `ollama_response_seconds` — response time
- `ollama_load_duration_seconds` — model load time
- `ollama_tokens_processed_total` — prompt tokens
- `ollama_tokens_generated_total` — generated tokens
- `ollama_tokens_per_second` — generation speed

### Open WebUI

Multi-user chat interface. 

- Connects to Ollama via ollama-exporter: `http://ollama-exporter.ollama.svc.cluster.local:8000`
- Exposed at `http://openwebui.local`
- User data and conversations stored in PVC
- Admin panel: disable public registration, set default role to `pending`

### Docker Registry

Private image registry for cluster-built images.

- Pinned to `registry` node via `nodeSelector`
- Taint: `dedicated=registry:NoSchedule`
- Exposed via NodePort `30964`
- Insecure (HTTP) — configure Docker daemon on build machines accordingly

---

## Access

| Service | URL |
|---|---|
| Open WebUI | http://openwebui.local |
| Grafana | http://grafana.local |
| Prometheus | http://prometheus.local |
| ArgoCD | http://argocd.local |
| Docker Registry | http://registry.local:30964/v2/ |

---

## Notes

- Ollama processes one request at a time on CPU. Requests are queued.
- First model load after pod restart is slow — ~4GB model read from NFS into RAM.
- `KEEP_ALIVE` default is 5 minutes — model stays in RAM after last request.
- `llm-worker` is intentionally over-provisioned (10 vCPU, 20GB RAM).
- `registry` node is minimal (1 vCPU, 2GB RAM) — Docker Registry is lightweight.
- Prometheus CRDs are too large for standard `kubectl apply` — always use `--server-side`.