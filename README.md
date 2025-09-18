# 🧪 Guide d’Installation — Lab DevSecOps avec k3s + Helm

> ✅ **100% Open Source**  
> ✅ **Stack Pro : GitLab CE, Semgrep, Trivy, OWASP ZAP, Cosign, Vault, Falco, PLG (Prometheus, Loki, Grafana)**  
> ✅ **OS : Ubuntu 22.04/24.04 LTS**  
> ✅ **Architecture : k3s (Lightweight Kubernetes) + Helm Charts**  
> ✅ **Idéal pour étudiants SI, projets de fin d’étude, labs professionnels**

---

## 🖥️ Prérequis Matériels & Logiciels

| Élément | Spécification minimale |
|--------|-------------------------|
| **OS** | Ubuntu 22.04 LTS ou 24.04 LTS |
| **CPU** | 4 cœurs (8 recommandés) |
| **RAM** | 16 Go (minimum 12 Go) |
| **Stockage** | 50 Go SSD |
| **Accès root/sudo** | Oui |
| **Connexion Internet** | Oui |

> 💡 Ce lab est conçu pour une **machine physique ou VM dédiée**. WSL2 n’est pas recommandé pour k3s en production-like.

---

## 🚀 Étape 1 — Préparer le Système Ubuntu

```
# Mettre à jour le système
sudo apt update && sudo apt upgrade -y

# Installer les dépendances
sudo apt install -y curl wget git vim gnupg2 software-properties-common apt-transport-https ca-certificates jq

# Désactiver swap (requis pour Kubernetes)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Activer les modules kernel
sudo modprobe overlay
sudo modprobe br_netfilter

# Configurer sysctl
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```
--- 
## 🐄 Étape 2 — Installer k3s (Lightweight Kubernetes)
```
# Installer k3s en mode "single node" avec Traefik désactivé
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --disable metrics-server" sh -

# Vérifier le statut
sudo systemctl status k3s

# Copier le kubeconfig pour l'utilisateur courant
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config

# Vérifier que kubectl fonctionne
kubectl get nodes
```
✅ Vous devriez voir un nœud Ready. 
--- 
## 📦 Étape 3 — Installer Helm
```
# Télécharger et installer Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Vérifier
helm version
🌐 Étape 4 — Ajouter les Repositories Helm

helm repo add gitlab https://charts.gitlab.io/
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add loki https://grafana.github.io/loki/charts
helm repo add falco https://falcosecurity.github.io/charts
helm repo add hashicorp https://helm.releases.hashicorp.com

# Mettre à jour
helm repo update
🐳 Étape 5 — Déployer GitLab CE avec Helm

# Créer un namespace
kubectl create namespace gitlab

# Installer GitLab CE
helm upgrade --install gitlab gitlab/gitlab \
  --namespace gitlab \
  --set global.hosts.domain=localhost \
  --set global.hosts.externalIP=127.0.0.1 \
  --set certmanager.install=false \
  --set global.ingress.configureCertmanager=false \
  --set global.ingress.tls.enabled=false \
  --set global.edition=ce \
  --set prometheus.install=false \
  --set gitlab-runner.install=false \
  --timeout 600s

# Suivre le déploiement
kubectl get pods -n gitlab -w

## 🌐 Accès : 

URL : http://gitlab.localhost
Ajoutez dans /etc/hosts :

echo "127.0.0.1 gitlab.localhost" | sudo tee -a /etc/hosts

🔐 Mot de passe root : 
kubectl get secret gitlab-gitlab-initial-root-password -n gitlab -ojsonpath='{.data.password}' | base64 -d ; echo
```

## 📊 Étape 6 — Déployer la Stack PLG (Prometheus + Loki + Grafana)
```
# Créer un namespace
kubectl create namespace monitoring

# Déployer Prometheus
helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.enabled=false \
  --set prometheus.prometheusSpec.scrapeInterval=15s

# Déployer Loki
helm upgrade --install loki loki/loki \
  --namespace monitoring \
  --set config.schema_config.configs[0].from=2020-10-24 \
  --set config.schema_config.configs[0].schema=v11 \
  --set config.schema_config.configs[0].store=boltdb-shipper \
  --set config.schema_config.configs[0].object_store=filesystem

# Déployer Promtail
helm upgrade --install promtail loki/promtail \
  --namespace monitoring \
  --set "config.clients[0].url=http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push"

# Déployer Grafana
helm upgrade --install grafana grafana/grafana \
  --namespace monitoring \
  --set adminPassword=Admin123! \
  --set service.type=NodePort \
  --set service.nodePort=30001

# Obtenir l'URL d'accès à Grafana
echo "Grafana : http://$(hostname -I | awk '{print $1}'):30001"
echo "User: admin | Pass: Admin123!"
➕ Dans Grafana, ajoutez les sources de données : 

Prometheus : http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090
Loki : http://loki.monitoring.svc.cluster.local:3100
```
--- 
## 🚨 Étape 7 — Déployer Falco (Sécurité Runtime)
```
# Installer le driver kernel
sudo apt install -y linux-headers-$(uname -r)
curl -s https://falco.org/repo/falcosecurity-3672BA8F.asc | sudo apt-key add -
echo "deb https://download.falco.org/packages/deb stable main" | sudo tee -a /etc/apt/sources.list.d/falcosecurity.list
sudo apt update
sudo apt install -y falco

# Redémarrer le service Falco
sudo systemctl restart falco

# (Optionnel) Déployer l'agent Falco dans le cluster
helm upgrade --install falco-agent falco/falco \
  --namespace monitoring \
  --create-namespace \
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=true \
  --set tty=true
🕵️‍♂️ Voir les alertes : sudo journalctl -fu falco 
```
---
## 🔐 Étape 8 — Déployer HashiCorp Vault (Gestion des Secrets)
```
# Créer un namespace
kubectl create namespace vault

# Installer Vault
helm upgrade --install vault hashicorp/vault \
  --namespace vault \
  --set "server.dev.enabled=true" \
  --set "ui.enabled=true" \
  --set "service.type=NodePort" \
  --set "service.nodePort=30002"

# Vérifier
kubectl get pods -n vault -w

# Accès UI : http://<IP>:30002
# Initialiser Vault (dans un autre terminal)
kubectl exec -n vault vault-0 -- vault operator init -key-shares=1 -key-threshold=1

# Débloquer avec la clé fournie
kubectl exec -n vault vault-0 -- vault operator unseal <CLÉ>

# Activer l'auth Kubernetes
kubectl exec -n vault vault-0 -- vault auth enable kubernetes
kubectl exec -n vault vault-0 -- sh -c 'vault write auth/kubernetes/config \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt'
```
---
## 🧪 Étape 9 — Déployer OWASP Juice Shop (App Vulnérable)
```
kubectl create namespace apps

kubectl create deployment juice-shop --image=bkimminich/juice-shop -n apps
kubectl expose deployment juice-shop --port=3000 --type=NodePort -n apps

# Obtenir le port
NODE_PORT=$(kubectl get svc juice-shop -n apps -o jsonpath='{.spec.ports[0].nodePort}')
echo "Juice Shop disponible sur : http://$(hostname -I | awk '{print $1}'):${NODE_PORT}"
```
--- 
## 🧰 Étape 10 — Intégrer les Outils dans GitLab CI (.gitlab-ci.yml)
Créez un projet dans GitLab → ajoutez un fichier .gitlab-ci.yml :
```
stages:
  - test
  - scan
  - sign
  - deploy

variables:
  KUBECONFIG: /tmp/kubeconfig

before_script:
  - apk add --no-cache curl bash git python3 py3-pip docker-cli kubectl

# ============ SAST ============
semgrep:
  stage: scan
  script:
    - curl -L https://semgrep.dev/install.sh | sh
    - ~/.local/bin/semgrep scan --config auto --json -o semgrep-report.json
  artifacts:
    paths: [semgrep-report.json]
    when: always

# ============ SCA & Container Scan ============
trivy:
  stage: scan
  script:
    - curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
    - trivy image --exit-code 1 --severity CRITICAL,HIGH --format json -o trivy-report.json bkimminich/juice-shop:latest
  artifacts:
    paths: [trivy-report.json]
    when: always

# ============ DAST ============
zap:
  stage: test
  script:
    - mkdir -p /zap/wrk
    - docker run --rm -v $(pwd):/zap/wrk owasp/zap2docker-stable zap-baseline.py -t http://juice-shop.apps.svc.cluster.local:3000 -j -r zap-report.json
  artifacts:
    paths: [zap-report.json]
    when: always

# ============ Sign Image with Cosign ============
cosign-sign:
  stage: sign
  script:
    - curl -L https://github.com/sigstore/cosign/releases/download/v2.2.4/cosign-linux-amd64 -o /usr/local/bin/cosign
    - chmod +x /usr/local/bin/cosign
    - cosign generate-key-pair
    - cosign sign --key cosign.key bkimminich/juice-shop:latest
  when: manual

# ============ Déployer sur k3s (si scan OK) ===========
deploy:
  stage: deploy
  script:
    - mkdir -p /root/.kube
    - kubectl config view --raw > $KUBECONFIG
    - kubectl apply -f https://raw.githubusercontent.com/bkimminich/juice-shop/master/kubernetes/juice-shop.yaml -n apps
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```
## 🎨 Étape 11 — Importer le Dashboard Grafana “Security Post-Deploy”
```
Allez sur Grafana → http://<IP>:30001
Login : admin / Admin123!
Configuration → Data Sources → ajoutez :
Prometheus : http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090
Loki : http://loki.monitoring.svc.cluster.local:3100
Create → Import → Collez le JSON du dashboard “Security Post-Deploy”
```

## 🧹 Étape 12 — Nettoyage & Sauvegarde
```
# Supprimer tout le lab
helm uninstall gitlab -n gitlab
helm uninstall prometheus -n monitoring
helm uninstall loki -n monitoring
helm uninstall promtail -n monitoring
helm uninstall grafana -n monitoring
helm uninstall falco-agent -n monitoring
helm uninstall vault -n vault
kubectl delete namespace gitlab monitoring vault apps
```
# Supprimer k3s
/usr/local/bin/k3s-uninstall.sh
