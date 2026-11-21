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

#Add the following lines to `/etc/hosts` on the client machine:
```
192.168.0.201  openwebui.local
192.168.0.201  grafana.local
192.168.0.201  prometheus.local
192.168.0.201  argocd.local
192.168.0.22   registry.local
```

#Services URL 
```
Open WebUI http://openwebui.local 
Grafana http://grafana.local 
Prometheus http://prometheus.local 
ArgoCD http://argocd.local 
Docker Registry http://registry.local:30964/v2/ 
```

3) ----Prerequisites + Helm components

#Cluster Kubernetes 1.35+, Calico CNI, Helm on the control-plane.

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
#Proxmox host
apt install nfs-kernel-server -y
echo "/usr/share/ollama/.ollama 192.168.0.0/24(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
exportfs -ra

#every worker
sudo apt install nfs-common -y
```

6) ----Nodes
```
#node that runs Ollama
kubectl label nodes <node-name> role=llm

#registry node
kubectl taint nodes registry dedicated=registry:NoSchedule
```
7) ---- Secret creation
````
kubectl create namespace open-webui
kubectl create secret generic open-webui-secret -n open-webui \
  --from-literal=WEBUI_SECRET_KEY="$(openssl rand -hex 32)"
````
8) ----CA
````
#!!!YOUR ROOT KEY MUST BE STORED ALWAYS OFFLINE!!!
#CREATE THE FILE WITH:
mkdir -p ~/ca
chmod 700 ~/ca
cd ~/ca
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -aes256 -out root.key

#REVIEW THE FILE HEADER
head -1 ~/ca/root.key

#CREATE THE CA
openssl req -x509 -new -key root.key -sha256 -days 3650 \
  -subj "/CN=yourrootCA" \
  -addext "basicConstraints=critical,CA:TRUE" \
  -addext "keyUsage=critical,keyCertSign,cRLSign" \
  -out root.crt

#CHECK IT AGAIN
openssl x509 -in root.crt -noout -subject -dates -ext basicConstraints,keyUsage

#CREATE THE INTERMIDIATE KEY
cd ~/ca
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out intermediate.key
chmod 600 intermediate.key

#CSR Request
openssl req -new -key intermediate.key \
  -subj "/CN= --A-NAME-YOU-CHOOSE--" \
  -out intermediate.csr

#DEFINE INTERMIDIATE
nano ~/ca/intermediate.ext

# paste this into the file:
basicConstraints = critical, CA:TRUE, pathlen:0
keyUsage = critical, keyCertSign, cRLSign
nameConstraints = critical, permitted;DNS:local
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always

#SIGN WITH THE ROOT
openssl x509 -req -in intermediate.csr \
  -CA root.crt -CAkey root.key -CAcreateserial \
  -days 1825 -sha256 \
  -extfile intermediate.ext \
  -out intermediate.crt

#LAST CHECKS
openssl x509 -in intermediate.crt -noout -subject -issuer -dates -ext basicConstraints,nameConstraints
openssl verify -CAfile root.crt intermediate.crt

#AND LAST FOR K8
kubectl create secret tls intermediate-ca -n cert-manager \
  --cert=intermediate.crt --key=intermediate.key
````
9) ----Once ArgoCD is running
```
kubectl apply -f root-app.yaml
```

10) ----ollama-exporter image
```
git clone https://github.com/frcooper/ollama-exporter.git
cd ollama-exporter
docker build -t 192.168.0.22:30964/ollama-exporter:latest .
docker push 192.168.0.22:30964/ollama-exporter:latest
```

The registry is HTTP: add `192.168.0.22:30964` to `insecure-registries` in the Docker daemon of the build machine.



