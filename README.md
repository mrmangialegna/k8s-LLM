k8s-LLM

On-premise platform to run LLMs locally on Kubernetes.
Managed with GitOps: ArgoCD watches this repo and applies every change to the cluster.

1) ----How it works

- Ollama runs the models, CPU only. Models are stored on NFS, so they survive pod restarts.
- Open WebUI is the chat interface. Multi-user, public registration disabled.
- ollama-exporter sits between Open WebUI and Ollama. It forwards the requests and exposes metrics to Prometheus: requests per model, response time, model load time, tokens per second. It's the open source project [frcooper/ollama-exporter](https://github.com/frcooper/ollama-exporter), built from source and pushed to the private registry.
- Docker Registry hosts the images built for the cluster. HTTP only, exposed on NodePort `30964`.
- MetalLB + Traefik give the cluster an external IP and route traffic to each service by hostname.

Models: constantly updating


2) ----Access

Add the following lines to `/etc/hosts` on the client machine:
```
192.168.0.201  openwebui.local
192.168.0.201  grafana.local
192.168.0.201  prometheus.local
192.168.0.201  argocd.local
192.168.0.22   registry.local
```

Services URL 
```
Open WebUI http://openwebui.local 
Grafana http://grafana.local 
Prometheus http://prometheus.local 
ArgoCD http://argocd.local 
Docker Registry http://registry.local:30964/v2/ 
```

3) ----Prerequisites + Helm components

Cluster Kubernetes 1.35+, Calico CNI, Helm on the control-plane.

```
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb -n metallb-system --create-namespace

helm repo add traefik https://traefik.github.io/charts
helm install traefik traefik/traefik -n kube-system

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace --server-side

helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace

helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager -n cert-manager --create-namespace \
  --version v1.21.3 --set crds.enabled=true
```

4) ----StorageClass
```
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

5) ----NFS

NFS for the models: server on the Proxmox host, client on every worker.
```
Proxmox host
apt install nfs-kernel-server -y
echo "/usr/share/ollama/.ollama 192.168.0.0/24(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
exportfs -ra

every worker
sudo apt install nfs-common -y
```

6) ----Nodes
```
node that runs Ollama
kubectl label nodes <node-name> role=llm

registry node
kubectl taint nodes registry dedicated=registry:NoSchedule
```
7) ---- Secret creation
````
kubectl create namespace open-webui
kubectl create secret generic open-webui-secret -n open-webui \
  --from-literal=WEBUI_SECRET_KEY="$(openssl rand -hex 32)"
````

8) ----Once ArgoCD is running
```
kubectl apply -f root-app.yaml
```

9) ----ollama-exporter image
```
git clone https://github.com/frcooper/ollama-exporter.git
cd ollama-exporter
docker build -t 192.168.0.22:30964/ollama-exporter:latest .
docker push 192.168.0.22:30964/ollama-exporter:latest
```

The registry is HTTP: add `192.168.0.22:30964` to `insecure-registries` in the Docker daemon of the build machine.