# Quickstart — Minikube + Traefik + OpenBao

> Debian sous WSL2 · Cluster local Minikube · Déploiement via manifestes YAML

---

## Table des matières

1. [Pré-requis WSL2/Debian](#1-pré-requis-wsl2--debian)
2. [Installation de kubectl](#2-installation-de-kubectl)
3. [Installation de Minikube](#3-installation-de-minikube)
4. [Démarrer le cluster](#4-démarrer-le-cluster)
5. [Cheat sheet kubectl](#5-cheat-sheet-kubectl)
6. [Déployer Traefik](#6-déployer-traefik)
7. [Déployer OpenBao](#7-déployer-openbao)
8. [Vérification globale](#8-vérification-globale)
9. [Accès aux interfaces](#9-accès-aux-interfaces)
10. [Troubleshooting WSL2](#10-troubleshooting-wsl2)

---

## 1. Pré-requis WSL2 / Debian

### Vérifier WSL2

Depuis PowerShell (Windows) :
```powershell
wsl --list --verbose
```

Si ce n'est pas le cas :
```powershell
wsl --set-version Debian 2
```

### Allouer assez de RAM à WSL2

Créer/éditer `C:\Users\<VotreUser>\.wslconfig` :
```ini
[wsl2]
memory=8GB
processors=4
swap=2GB
```
Puis redémarrer WSL2 : `wsl --shutdown` depuis PowerShell.

### Mettre à jour Debian et installer les dépendances

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget apt-transport-https ca-certificates \
  gnupg lsb-release conntrack socat
```

### Docker dans WSL2

**Option A — Docker Desktop** (recommandé, intégration WSL2 native) :  
Installer Docker Desktop sur Windows, activer l'intégration WSL2 dans les settings.

**Option B — Docker natif dans Debian** :
```bash
# Ajouter le dépôt Docker
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
  https://download.docker.com/linux/debian $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER
# Se déconnecter/reconnecter ou : newgrp docker
```

---

## 2. Installation de kubectl

```bash
# Télécharger la dernière version stable
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Rendre exécutable et déplacer dans le PATH
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl

# Vérifier
kubectl version --client
```

### Auto-complétion (fortement recommandé)

```bash
# Bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
```

---

## 3. Installation de Minikube

```bash
# Télécharger Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

# Installer
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64

# Vérifier
minikube version
```

---

## 4. Démarrer le cluster

### Premier démarrage

```bash
minikube start \
  --driver=docker \
  --cpus=2 \
  --memory=4096 \
  --disk-size=20g \
  --kubernetes-version=stable
```

>  Le premier démarrage télécharge l'image (~500 Mo).

### Activer les addons utiles

```bash
# Ingress controller (nécessaire si vous n'utilisez pas Traefik en standalone)
minikube addons enable ingress

# Dashboard Kubernetes
minikube addons enable dashboard
minikube addons enable metrics-server

# Voir tous les addons disponibles
minikube addons list
```

### Vérifier que le cluster fonctionne

```bash
# État du cluster
minikube status

# Voir les noeuds
kubectl get nodes

# Voir les pods système
kubectl get pods -n kube-system
```

Résultat attendu :
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   2m    v1.28.x
```

### Commandes de gestion Minikube

```bash
minikube stop          # Arrêter le cluster (préserve les données)
minikube start         # Redémarrer
minikube delete        # Supprimer complètement le cluster
minikube pause         # Mettre en pause (libère CPU)
minikube unpause       # Reprendre

minikube ip            # IP du cluster (utile pour /etc/hosts)
minikube dashboard     # Ouvrir le dashboard Kubernetes
minikube tunnel        # Activer les LoadBalancers (dans un autre terminal)
```

---

## 5. Cheat sheet kubectl

### Navigation et contextes

```bash
kubectl config get-contexts              # Lister les contextes
kubectl config use-context minikube      # Changer de contexte
kubectl config current-context          # Contexte actif
kubectl cluster-info                     # Infos du cluster
```

### Namespaces

```bash
kubectl get namespaces                   # Lister les namespaces
kubectl get ns                           # Raccourci
kubectl create namespace mon-ns          # Créer un namespace
kubectl delete namespace mon-ns          # Supprimer

# Définir un namespace par défaut (évite de taper -n à chaque fois)
kubectl config set-context --current --namespace=traefik
```

### Pods

```bash
kubectl get pods                         # Pods du namespace courant
kubectl get pods -n traefik              # Pods d'un namespace spécifique
kubectl get pods -A                      # Tous les pods de tous les namespaces
kubectl get pods -o wide                 # Avec plus d'infos (IP, node…)
kubectl get pods -w                      # Watch (mise à jour en temps réel)

kubectl describe pod <nom>               # Détails complets + événements
kubectl logs <nom>                       # Logs du conteneur
kubectl logs <nom> -f                    # Logs en streaming (follow)
kubectl logs <nom> --previous            # Logs du conteneur précédent (après crash)

kubectl exec -it <nom> -- /bin/sh        # Shell dans le conteneur
kubectl exec -it <nom> -- /bin/bash      # Si bash disponible
kubectl port-forward pod/<nom> 8080:80   # Forward port local → pod
```

### Deployments

```bash
kubectl get deployments -n <ns>
kubectl get deploy -n <ns>               # Raccourci

kubectl rollout status deployment/<nom>  # Statut du déploiement
kubectl rollout history deployment/<nom> # Historique des versions
kubectl rollout undo deployment/<nom>    # Rollback vers la version précédente

kubectl scale deployment <nom> --replicas=3    # Scaler manuellement
kubectl set image deployment/<nom> <container>=<image>:<tag>  # Mettre à jour l'image
```

### Services & Ingress

```bash
kubectl get services -n <ns>
kubectl get svc -n <ns>                  # Raccourci
kubectl get ingress -n <ns>
kubectl get ing -n <ns>                  # Raccourci

kubectl port-forward svc/<nom> 8080:80   # Forward port local → service
```

### Appliquer des manifestes

```bash
kubectl apply -f fichier.yaml            # Appliquer un fichier
kubectl apply -f dossier/                # Appliquer tous les fichiers d'un dossier
kubectl apply -f https://url/fichier.yaml # Depuis une URL

kubectl delete -f fichier.yaml           # Supprimer les ressources d'un fichier
kubectl diff -f fichier.yaml             # Voir les différences avant apply
```

### Informations et debugging

```bash
kubectl get all -n <ns>                  # Tout voir dans un namespace
kubectl get events -n <ns>               # Événements du namespace
kubectl get events --sort-by=.lastTimestamp   # Triés par date

kubectl top pods -n <ns>                 # Consommation CPU/RAM des pods
kubectl top nodes                        # Consommation des nœuds

kubectl explain deployment               # Documentation d'une ressource
kubectl explain deployment.spec.template # Documentation d'un champ spécifique
```

### ConfigMaps & Secrets

```bash
kubectl get configmaps -n <ns>
kubectl get cm -n <ns>                   # Raccourci
kubectl describe cm <nom> -n <ns>

kubectl get secrets -n <ns>
kubectl describe secret <nom> -n <ns>

# Décoder un secret
kubectl get secret <nom> -n <ns> -o jsonpath='{.data.<clé>}' | base64 --decode
```

### Labels et sélecteurs

```bash
kubectl get pods -l app=traefik          # Filtrer par label
kubectl label pod <nom> env=prod         # Ajouter un label
kubectl annotate pod <nom> description="mon pod"   # Ajouter une annotation
```

### Sortie formatée

```bash
kubectl get pods -o yaml                 # Sortie YAML complète
kubectl get pods -o json                 # Sortie JSON
kubectl get pods -o jsonpath='{.items[*].metadata.name}'  # JSONPath
kubectl get pods -o custom-columns='NAME:.metadata.name,STATUS:.status.phase'
```

---

## 6. Déployer Traefik

Créer la structure de fichiers :
```bash
mkdir -p manifests/traefik
```

### 6.1 Namespace

```bash
cat > manifests/traefik/namespace.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: traefik
  labels:
    app: traefik
EOF
```

### 6.2 RBAC — ServiceAccount & ClusterRole

Traefik a besoin de lire les ressources Kubernetes (Ingress, Services…) pour fonctionner.

```bash
cat > manifests/traefik/rbac.yaml << 'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik
  namespace: traefik
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: traefik
rules:
  - apiGroups: [""]
    resources: ["services", "endpoints", "secrets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["extensions", "networking.k8s.io"]
    resources: ["ingresses", "ingressclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["extensions", "networking.k8s.io"]
    resources: ["ingresses/status"]
    verbs: ["update"]
  - apiGroups: ["traefik.io", "traefik.containo.us"]
    resources: ["*"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: traefik
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik
subjects:
  - kind: ServiceAccount
    name: traefik
    namespace: traefik
EOF
```

### 6.3 Deployment

```bash
cat > manifests/traefik/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: traefik
  namespace: traefik
  labels:
    app: traefik
spec:
  replicas: 1
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik
      containers:
        - name: traefik
          image: traefik:v3.0
          args:
            - --api.insecure=true           # Dashboard sans auth (lab uniquement !)
            - --api.dashboard=true
            - --providers.kubernetesingress
            - --entrypoints.web.address=:80
            - --log.level=INFO
          ports:
            - name: web
              containerPort: 80
            - name: dashboard
              containerPort: 8080
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "300m"
              memory: "128Mi"
          readinessProbe:
            httpGet:
              path: /ping
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /ping
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
EOF
```

### 6.4 Services

```bash
cat > manifests/traefik/service.yaml << 'EOF'
# Service pour le trafic HTTP entrant
apiVersion: v1
kind: Service
metadata:
  name: traefik
  namespace: traefik
  labels:
    app: traefik
spec:
  type: NodePort
  selector:
    app: traefik
  ports:
    - name: web
      port: 80
      targetPort: web
      nodePort: 30080
---
# Service pour le dashboard Traefik
apiVersion: v1
kind: Service
metadata:
  name: traefik-dashboard
  namespace: traefik
  labels:
    app: traefik
spec:
  type: ClusterIP
  selector:
    app: traefik
  ports:
    - name: dashboard
      port: 8080
      targetPort: dashboard
EOF
```

### 6.5 Appliquer les manifestes Traefik

```bash
# Appliquer dans l'ordre
kubectl apply -f manifests/traefik/namespace.yaml
kubectl apply -f manifests/traefik/rbac.yaml
kubectl apply -f manifests/traefik/deployment.yaml
kubectl apply -f manifests/traefik/service.yaml

# Ou tout d'un coup (Kubernetes gère l'ordre pour les dépendances simples)
kubectl apply -f manifests/traefik/

# Vérifier
kubectl get all -n traefik
kubectl rollout status deployment/traefik -n traefik
```

Résultat attendu :
```
deployment.apps/traefik successfully rolled out
```

---

## 7. Déployer OpenBao

```bash
mkdir -p manifests/openbao
```

### 7.1 Namespace

```bash
cat > manifests/openbao/namespace.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: openbao
  labels:
    app: openbao
EOF
```

### 7.2 ServiceAccount & RBAC

```bash
cat > manifests/openbao/rbac.yaml << 'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: openbao
  namespace: openbao
---
# ClusterRoleBinding pour l'auth Kubernetes d'OpenBao
# (permet à OpenBao de vérifier les tokens des ServiceAccounts des pods)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: openbao-auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: openbao
    namespace: openbao
EOF
```

### 7.3 ConfigMap (configuration OpenBao)

```bash
cat > manifests/openbao/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: openbao-config
  namespace: openbao
data:
  config.hcl: |
    ui = true

    listener "tcp" {
      address     = "0.0.0.0:8200"
      tls_disable = 1          # TLS désactivé en lab local
    }

    storage "file" {
      path = "/vault/data"     # Stockage fichier (dev/lab)
    }

    # Pour la production, utiliser Consul ou Raft comme backend de stockage

    api_addr = "http://0.0.0.0:8200"
    cluster_addr = "http://0.0.0.0:8201"
EOF
```

### 7.4 PersistentVolumeClaim

```bash
cat > manifests/openbao/pvc.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: openbao-data
  namespace: openbao
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  # storageClassName: standard  # Décommenter si besoin de spécifier une storageClass
EOF
```

### 7.5 Deployment

```bash
cat > manifests/openbao/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openbao
  namespace: openbao
  labels:
    app: openbao
spec:
  replicas: 1
  selector:
    matchLabels:
      app: openbao
  template:
    metadata:
      labels:
        app: openbao
    spec:
      serviceAccountName: openbao
      containers:
        - name: openbao
          image: quay.io/openbao/openbao:latest
          command: ["bao", "server", "-config=/vault/config/config.hcl"]
          ports:
            - name: http
              containerPort: 8200
            - name: cluster
              containerPort: 8201
          env:
            - name: BAO_ADDR
              value: "http://0.0.0.0:8200"
            - name: BAO_API_ADDR
              value: "http://0.0.0.0:8200"
            - name: SKIP_SETCAP
              value: "true"
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          securityContext:
            capabilities:
              add: ["IPC_LOCK"]    # Empêche le swap des secrets en mémoire
          volumeMounts:
            - name: config
              mountPath: /vault/config
            - name: data
              mountPath: /vault/data
          readinessProbe:
            httpGet:
              path: /v1/sys/health?standbyok=true&uninitcode=204&sealedcode=204
              port: 8200
            initialDelaySeconds: 5
            periodSeconds: 10
      volumes:
        - name: config
          configMap:
            name: openbao-config
        - name: data
          persistentVolumeClaim:
            claimName: openbao-data
EOF
```

### 7.6 Service

```bash
cat > manifests/openbao/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: openbao
  namespace: openbao
  labels:
    app: openbao
spec:
  type: ClusterIP
  selector:
    app: openbao
  ports:
    - name: http
      port: 8200
      targetPort: http
    - name: cluster
      port: 8201
      targetPort: cluster
EOF
```

### 7.7 Ingress (via Traefik)

```bash
cat > manifests/openbao/ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: openbao
  namespace: openbao
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
    - host: openbao.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: openbao
                port:
                  number: 8200
EOF
```

### 7.8 Appliquer les manifestes OpenBao

```bash
kubectl apply -f manifests/openbao/namespace.yaml
kubectl apply -f manifests/openbao/rbac.yaml
kubectl apply -f manifests/openbao/configmap.yaml
kubectl apply -f manifests/openbao/pvc.yaml
kubectl apply -f manifests/openbao/deployment.yaml
kubectl apply -f manifests/openbao/service.yaml
kubectl apply -f manifests/openbao/ingress.yaml

# Vérifier
kubectl get all -n openbao
kubectl rollout status deployment/openbao -n openbao
```

### 7.9 Initialiser OpenBao

OpenBao démarre en mode **sealed** (scellé). Il faut l'initialiser et le desceller.

```bash
# Accéder au pod OpenBao
kubectl exec -it -n openbao deployment/openbao -- /bin/sh

# Dans le pod :
# Initialiser (génère les unseal keys et le root token)
bao operator init

# Résultat (CONSERVER CES CLÉS PRÉCIEUSEMENT) :
# Unseal Key 1: xxxx
# Unseal Key 2: xxxx
# Unseal Key 3: xxxx
# Unseal Key 4: xxxx
# Unseal Key 5: xxxx
# Initial Root Token: hvs.xxxx

# Desceller avec 3 clés sur 5 (threshold par défaut)
bao operator unseal <Unseal Key 1>
bao operator unseal <Unseal Key 2>
bao operator unseal <Unseal Key 3>

# Vérifier le statut
bao status

# Se connecter
export VAULT_TOKEN="<Initial Root Token>"
bao login

# Activer le moteur KV v2
bao secrets enable -path=secret kv-v2

# Créer un premier secret
bao kv put secret/hello foo=bar

# Le lire
bao kv get secret/hello
```

> En production (ni en preprod), ne jamais stocker le root token. Créer des tokens avec des policies limitées.

---

## 8. Vérification globale

```bash
# État général du cluster
kubectl get nodes
kubectl get namespaces

# Tous les pods de toutes les apps
kubectl get pods -n traefik
kubectl get pods -n openbao

# Toutes les ressources
kubectl get all -n traefik
kubectl get all -n openbao

# Vérifier les Ingress
kubectl get ingress -A

# Événements récents
kubectl get events -A --sort-by=.lastTimestamp | tail -20
```

---

## 9. Accès aux interfaces

### Méthode 1 — Port-forward (recommandé en WSL2)

```bash
# Dashboard Traefik
kubectl port-forward -n traefik svc/traefik-dashboard 8080:8080 &
# → http://localhost:8080/dashboard/

# Interface OpenBao
kubectl port-forward -n openbao svc/openbao 8200:8200 &
# → http://localhost:8200/ui/
```

### Méthode 2 — Via /etc/hosts + NodePort

```bash
# Récupérer l'IP de Minikube
MINIKUBE_IP=$(minikube ip)
echo $MINIKUBE_IP

# Ajouter dans /etc/hosts (dans WSL2)
echo "$MINIKUBE_IP traefik.local openbao.local" | sudo tee -a /etc/hosts

# Ou dans C:\Windows\System32\drivers\etc\hosts (Windows, en admin)
# <IP_MINIKUBE> traefik.local openbao.local
```

### Méthode 3 — Dashboard Kubernetes

```bash
minikube dashboard
# Ouvre automatiquement le navigateur avec le dashboard complet
```

---

## 10. Troubleshooting WSL2

### Minikube ne démarre pas

```bash
# Vérifier Docker
docker info
docker ps

# Nettoyer et recommencer
minikube delete
minikube start --driver=docker

# Si erreur de connexion au socket Docker
sudo chmod 666 /var/run/docker.sock
# ou
sudo usermod -aG docker $USER && newgrp docker
```

### Pods en CrashLoopBackOff

```bash
# Voir les logs
kubectl logs <pod> -n <namespace> --previous

# Voir les événements
kubectl describe pod <pod> -n <namespace>

# Vérifier les ressources disponibles
kubectl top nodes
kubectl describe nodes
```

### Impossible d'accéder aux services

```bash
# Vérifier que le service existe et a le bon selector
kubectl get svc -n <namespace>
kubectl describe svc <service> -n <namespace>

# Vérifier que les pods ont les bons labels
kubectl get pods -n <namespace> --show-labels

# Tester la connectivité depuis l'intérieur du cluster
kubectl run debug --image=busybox --rm -it --restart=Never -- wget -qO- http://<service>.<namespace>:8200
```

### OpenBao sealed après redémarrage

OpenBao se rescelle automatiquement à chaque redémarrage. Pour les environnements de développement, on pzut configurer l'**auto-unseal** (via AWS KMS, GCP KMS, etc.) ou simplement gérer  manuellement après chaque démarrage du pod.

```bash
# Vérifier le statut
kubectl exec -n openbao deployment/openbao -- bao status

# Si Sealed: true, resceller avec vos clés
kubectl exec -it -n openbao deployment/openbao -- bao operator unseal <key1>
kubectl exec -it -n openbao deployment/openbao -- bao operator unseal <key2>
kubectl exec -it -n openbao deployment/openbao -- bao operator unseal <key3>
```

### WSL2 : problème de réseau avec Minikube

```bash
# Vérifier le DNS
cat /etc/resolv.conf

# Redémarrer WSL2 depuis PowerShell
wsl --shutdown
# puis relancer votre terminal Debian

# Si minikube ip ne répond pas
minikube stop && minikube start
```

---

## Étapes suivantes

Une fois le lab fonctionnel, la suite logique est :

1. **Explorer** les interfaces : dashboard Traefik, UI OpenBao
2. **Créer une application de test** et la router via Traefik
3. **Intégrer OpenBao** pour injecter des secrets dans vos pods
4. **Passer à Terraform** pour gérer l'infra en code
5. **Mettre en place ArgoCD** pour le GitOps

---

*Quickstart — Atelier Kubernetes Lab | Minikube + Traefik + OpenBao*
