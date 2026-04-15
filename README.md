# k8s-LLM

Kubernetes manifests for the 2LMlocal platform — an on-premise AI platform running local language models.

This repo is managed by ArgoCD. Every change pushed here is automatically reconciled with the cluster.

---

## Repository Structure

```
k8s/
├── metalLB/
│   ├── ipaddresspool.yaml     # IP pool 192.168.0.20-40
│   └──  ingress/              # IngressRoute definitions
|       ├── argocd-ingress.yaml    # LB argo.local
│       ├── grafana-ingress.yaml   # LB grafana.local
│       ├── open-webui.yaml        # LB open-webui.local
│       ├── prometheus.yaml        # LB prometheus.local
├── ollama/
│   ├── pv.yaml                # PersistentVolume — NFS mount from Proxmox host
│   ├── pvc.yaml               # PVC for NFS models (40Gi, ReadWriteMany)
│   ├── pvc-data.yaml          # PVC for Ollama local data (15Gi, local-path)
│   ├── deployment.yaml        # Ollama deployment (pinned to worker-02)
│   └── service.yaml           # ClusterIP service on port 11434
└── open-webui/
    ├── pvc.yaml               # PVC for Open WebUI data (5Gi, local-path)
    ├── deployment.yaml        # Open WebUI deployment
    ├── service.yaml           # ClusterIP service on port 8080
    └── ingressroute.yaml      # Traefik IngressRoute → openwebui.local
```

---

## Prerequisites

Before applying these manifests, the following must be in place:

### Cluster
- Kubernetes 1.35.1+
- Calico CNI
- Helm installed on control-plane

### Components installed via Helm
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
# local-path-provisioner (required for PVCs)
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

### DNS on client machines
```
# /etc/hosts (Linux/Mac) or C:\Windows\System32\drivers\etc\hosts (Windows)
<TRAEFIK_IP>  openwebui.local
```

---

## ArgoCD Applications

This repo is structured so each folder maps to an ArgoCD Application:

| Application | Path | Namespace |
|---|---|---|
| k8s-llm-ollama | ollama | ollama |
| k8s-llm-openwebui | open-webui | open-webui |
| metallb | metalLB | metallb |

---

## Components

### MetalLB

Provides external IP addresses to LoadBalancer services (Traefik).

IP pool: `192.168.0.20 — 192.168.0.40`

> Important: use explicit range, not CIDR /24 — the /24 causes ARP conflicts.

### Ollama

LLM backend. Exposes a REST API compatible with OpenAI on port 11434.

- Pinned to `worker-02` via `nodeSelector`
- Models served via NFS from Proxmox host at `192.168.0.44:/usr/share/ollama/.ollama`
- Local data (cache, keys) stored in a separate `local-path` PVC
- CPU-only inference (GPU support planned)

Current models available:
- `mistral:7b-instruct-q4_K_M`
- `qwen2.5:7b-instruct`

### Open WebUI

Multi-user chat interface. Connects to Ollama via internal cluster DNS.

- `OLLAMA_BASE_URL`: `http://ollama.ollama.svc.cluster.local:11434`
- Exposed externally via Traefik IngressRoute at `http://openwebui.local`
- User data and conversations stored in PVC

---

## Access

| Service | URL |
|---|---|
| Open WebUI | http://openwebui.local |

---

## Notes

- Ollama processes one request at a time on CPU. Requests are queued.
- First model load after pod start is slow — the model (~4GB) is read from NFS into RAM.
- `KEEP_ALIVE` is set to 5 minutes — model stays in RAM for 5 minutes after last request.
- worker-02 is intentionally over-provisioned (10 vCPU, 20GB RAM) to handle LLM inference load.