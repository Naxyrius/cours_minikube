# 📚 Kubernetes — Support de Formation

> Ce document est le support théorique de l'atelier. Il couvre les concepts essentiels pour comprendre ce que vous allez manipuler dans le lab.

---

## Table des matières

1. [Pourquoi Kubernetes ?](#1-pourquoi-kubernetes)
2. [Architecture d'un cluster](#2-architecture-dun-cluster)
3. [Les objets fondamentaux](#3-les-objets-fondamentaux)
   - [Namespace](#31-namespace)
   - [Pod](#32-pod)
   - [Deployment](#33-deployment)
   - [Service](#34-service)
   - [Ingress](#35-ingress)
   - [ConfigMap & Secret](#36-configmap--secret)
   - [PersistentVolume & PVC](#37-persistentvolume--persistentvolumeclaim)
4. [Le cycle de vie d'une application](#4-le-cycle-de-vie-dune-application)
5. [Traefik dans Kubernetes](#5-traefik-dans-kubernetes)
6. [OpenBao dans Kubernetes](#6-openbao-dans-kubernetes)
7. [Vers le GitOps](#7-vers-le-gitops)
8. [Glossaire](#8-glossaire)

---

## 1. Pourquoi Kubernetes ?

### Le problème qu'il résout

Avant Kubernetes, déployer une application en production impliquait de gérer manuellement :
- Sur quel serveur tourne l'application ?
- Que se passe-t-il si ce serveur tombe ?
- Comment passer de 1 à 10 instances en cas de charge ?
- Comment déployer une nouvelle version sans coupure de service ?

**Docker** a résolu la problématique de packaging (une application = une image). Kubernetes résout la problématique d'**orchestration** : faire tourner ces conteneurs à grande échelle, de façon fiable et automatisée.

### Ce que Kubernetes apporte

| Besoin                        | Solution Kubernetes                        |
|-------------------------------|--------------------------------------------|
| Haute disponibilité           | Redémarrage automatique des pods           |
| Mise à l'échelle              | Horizontal Pod Autoscaler                  |
| Déploiement sans interruption | Rolling updates / Blue-Green               |
| Découverte de services        | DNS interne + Services                     |
| Gestion de la configuration   | ConfigMaps, Secrets                        |
| Isolation                     | Namespaces, NetworkPolicies, RBAC          |

### Kubernetes n'est pas…

- Un outil de CI/CD (il exécute, il ne build pas)
- Un outil de monitoring (Prometheus/Grafana s'ajoutent)
- Magique : il faut comprendre ce qu'on lui demande

---

## 2. Architecture d'un cluster

Un cluster Kubernetes est composé de deux types de nœuds :

```
┌─────────────────────────────────────────────────────────────┐
│                        CLUSTER                              │
│                                                             │
│  ┌──────────────────────────────┐                           │
│  │       CONTROL PLANE          │                           │
│  │  ┌────────────┐              │                           │
│  │  │ API Server │◄─── kubectl  │                           │
│  │  └─────┬──────┘              │                           │
│  │        │                     │                           │
│  │  ┌─────▼──────┐  ┌────────┐  │                           │
│  │  │  Scheduler │  │  etcd  │  │                           │
│  │  └────────────┘  └────────┘  │                           │
│  │  ┌─────────────────────────┐ │                           │
│  │  │   Controller Manager    │ │                           │
│  │  └─────────────────────────┘ │                           │
│  └──────────────────────────────┘                           │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │    NODE 1    │  │    NODE 2    │  │    NODE 3    │       │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │       │
│  │  │ kubelet│  │  │  │ kubelet│  │  │  │ kubelet│  │       │
│  │  ├────────┤  │  │  ├────────┤  │  │  ├────────┤  │       │
│  │  │ k-proxy│  │  │  │ k-proxy│  │  │  │ k-proxy│  │       │
│  │  ├────────┤  │  │  ├────────┤  │  │  ├────────┤  │       │
│  │  │  Pods  │  │  │  │  Pods  │  │  │  │  Pods  │  │       │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### Le Control Plane

| Composant              | Rôle                                                                 |
|------------------------|----------------------------------------------------------------------|
| **API Server**         | Point d'entrée unique. Toutes les commandes passent par lui.         |
| **etcd**               | Base de données clé/valeur distribuée. Stocke l'état du cluster.    |
| **Scheduler**          | Décide sur quel nœud placer un nouveau Pod.                          |
| **Controller Manager** | Boucles de contrôle qui s'assurent que l'état réel = état désiré.   |

### Les Nœuds (Workers)

| Composant          | Rôle                                                              |
|--------------------|-------------------------------------------------------------------|
| **kubelet**        | Agent sur chaque nœud. Exécute les instructions du control plane. |
| **kube-proxy**     | Gère les règles réseau pour exposer les Services.                 |
| **Container Runtime** | Docker, containerd… Fait tourner les conteneurs.               |

> 💡 **Dans Minikube**, le control plane et le worker tournent sur la même machine virtuelle. C'est un cluster à nœud unique, parfait pour le dev et l'apprentissage.

---

## 3. Les objets fondamentaux

Kubernetes fonctionne sur le principe de l'**état désiré** (desired state) : vous déclarez ce que vous voulez (en YAML), Kubernetes fait en sorte que la réalité corresponde.

```
Vous écrivez un YAML  →  kubectl apply  →  API Server  →  Kubernetes réconcilie
```

---

### 3.1 Namespace

Un **Namespace** est un espace de nommage logique qui partitionne les ressources d'un cluster.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: traefik
  labels:
    app: traefik
```

**Utilité :**
- Isoler les environnements (dev / staging / prod) sur un même cluster
- Isoler les applications les unes des autres *(notre approche : 1 NS = 1 app)*
- Appliquer des quotas de ressources par périmètre
- Gérer les droits d'accès (RBAC) par Namespace

> Certaines ressources sont "cluster-scoped" (non namespaced) : Nodes, PersistentVolumes, ClusterRoles…

---

### 3.2 Pod

Le **Pod** est la plus petite unité déployable dans Kubernetes. Un Pod encapsule un ou plusieurs conteneurs qui partagent :
- Le même réseau (même IP)
- Le même stockage (volumes)
- Le même cycle de vie

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
  namespace: demo
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
```

>  En pratique, on ne crée jamais de Pods directement. On utilise des **Deployments** qui gèrent les Pods pour nous.

**Caractéristiques importantes :**
- Un Pod est **éphémère** : s'il meurt, il n'est pas recréé automatiquement (sauf si géré par un Deployment)
- Un Pod a sa propre **adresse IP** dans le cluster (non accessible de l'extérieur directement)
- Les conteneurs d'un même Pod communiquent via `localhost`

---

### 3.3 Deployment

Un **Deployment** décrit comment déployer et maintenir un ensemble de Pods identiques (réplicas).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: demo
spec:
  replicas: 3                      # Je veux 3 Pods
  selector:
    matchLabels:
      app: nginx
  template:                        # Template du Pod à créer
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "500m"
```

**Ce que le Deployment gère pour vous :**
- Maintient le nombre de réplicas souhaité (si un Pod crash → il en recrée un)
- Gère les **rolling updates** : mise à jour progressive sans interruption de service
- Permet le **rollback** vers une version précédente

**Le chemin de création :**
```
Deployment  →  ReplicaSet  →  Pods
```
Le Deployment crée un ReplicaSet, qui lui gère les Pods. Vous n'interagissez généralement qu'avec le Deployment.

---

### 3.4 Service

Un **Service** expose un ensemble de Pods via une adresse stable (IP virtuelle + DNS). Sans Service, les Pods sont inaccessibles (et de toute façon leur IP change à chaque redémarrage).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: demo
spec:
  selector:
    app: nginx              # Cible les Pods avec ce label
  ports:
    - protocol: TCP
      port: 80              # Port du Service
      targetPort: 80        # Port du conteneur
  type: ClusterIP           # Type de Service
```

**Les types de Services :**

| Type           | Accessibilité                          | Usage                              |
|----------------|----------------------------------------|------------------------------------|
| `ClusterIP`    | Uniquement interne au cluster          | Communication entre services       |
| `NodePort`     | Via l'IP du nœud + un port (30000-32767) | Accès externe basique (dev)      |
| `LoadBalancer` | Via un load balancer externe (cloud)   | Prod sur AWS/GCP/Azure             |
| `ExternalName` | Alias DNS vers un service externe      | Intégration services externes      |

>  Dans Minikube, `LoadBalancer` est émulé via `minikube tunnel`.

**Comment le Service trouve les Pods ?**  
Via les **labels**. Le `selector` du Service correspond aux `labels` des Pods. C'est le mécanisme de couplage lâche de Kubernetes.

---

### 3.5 Ingress

Un **Ingress** gère le trafic HTTP/HTTPS entrant dans le cluster et le route vers les bons Services selon des règles (host, path…).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-ingress
  namespace: demo
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
    - host: monapp.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80
```

**Ingress ≠ Ingress Controller**

- L'**Ingress** est la ressource (l'objet YAML que vous créez)
- L'**Ingress Controller** est le composant qui lit ces ressources et fait le routage réel (c'est Traefik dans notre lab)

Sans Ingress Controller, les ressources Ingress sont ignorées.

```
Internet → Ingress Controller (Traefik) → lit les ressources Ingress → route vers Services → Pods
```

---

### 3.6 ConfigMap & Secret

**ConfigMap** : stocker de la configuration non sensible sous forme clé/valeur.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: demo
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  DATABASE_HOST: "postgres-service"
```

**Secret** : stocker des données sensibles (mots de passe, tokens, certificats). Les valeurs sont encodées en base64 (*attention : encodé ≠ chiffré*).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: demo
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=   # base64 de "password123"
  API_TOKEN: bXlzZWNyZXR0b2tlbg==
```

> Les Secrets Kubernetes natifs sont en base64, pas chiffrés. Pour une vraie gestion des secrets, on utilise **OpenBao** (ou Vault), qui sera intégré dans ce lab.

**Injection dans les Pods :**
```yaml
# Via variables d'environnement
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: LOG_LEVEL

# Via volume (fichiers montés)
volumes:
  - name: config-volume
    configMap:
      name: app-config
```

---

### 3.7 PersistentVolume & PersistentVolumeClaim

Par défaut, le stockage d'un Pod est **éphémère** : si le Pod redémarre, les données sont perdues.

- **PersistentVolume (PV)** : ressource de stockage provisionnée dans le cluster (disque, NFS…)
- **PersistentVolumeClaim (PVC)** : demande de stockage faite par un Pod

```yaml
# Le PVC (ce que l'application demande)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: openbao-storage
  namespace: openbao
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```yaml
# Utilisation dans un Deployment
volumes:
  - name: data
    persistentVolumeClaim:
      claimName: openbao-storage
containers:
  - name: openbao
    volumeMounts:
      - mountPath: /vault/data
        name: data
```

---

## 4. Le cycle de vie d'une application

### Du YAML au Pod en production

```
1. Ecriture des manifestes YAML
        ↓
2. kubectl apply -f manifests/
        ↓
3. L'API Server valide et stocke dans etcd
        ↓
4. Le Scheduler choisit un nœud pour les Pods
        ↓
5. kubelet télécharge l'image et démarre le conteneur
        ↓
6. Le Pod est Running
        ↓
7. Le Service route le trafic vers le Pod
        ↓
8. L'Ingress expose le Service à l'extérieur
```

### États d'un Pod

| État          | Signification                                         |
|---------------|-------------------------------------------------------|
| `Pending`     | En attente de scheduling (ressources, image en dl…)   |
| `Running`     | Au moins un conteneur tourne                          |
| `Succeeded`   | Tous les conteneurs ont terminé avec succès (jobs)    |
| `Failed`      | Au moins un conteneur a terminé en erreur             |
| `CrashLoopBackOff` | Le conteneur redémarre en boucle (erreur applicative) |
| `ImagePullBackOff` | Impossible de télécharger l'image                |

### Debugging rapide

```bash
# Voir l'état des Pods
kubectl get pods -n <namespace>

# Détails et événements d'un Pod
kubectl describe pod <nom-du-pod> -n <namespace>

# Logs du conteneur
kubectl logs <nom-du-pod> -n <namespace>

# Shell dans le conteneur
kubectl exec -it <nom-du-pod> -n <namespace> -- /bin/sh
```

---

## 5. Traefik dans Kubernetes

### Qu'est-ce que Traefik ?

Traefik est un **reverse proxy et ingress controller** moderne, conçu nativement pour les environnements cloud. Il se distingue par :
- Sa **découverte automatique** des services (il lit les annotations et les ressources Ingress)
- Son **dashboard** intégré
- Son support natif de **Let's Encrypt** (TLS automatique)
- Ses **middlewares** (authentification, rate limiting, redirections…)

### Architecture dans le lab

```
Requête HTTP
     ↓
[Traefik Pod - namespace: traefik]
     │
     ├── Lit les ressources Ingress du cluster
     ├── Route vers le bon Service
     └── Applique les middlewares configurés
          ↓
     [Service cible]
          ↓
     [Pods de l'application]
```

### Concepts clés Traefik

| Concept        | Description                                              |
|----------------|----------------------------------------------------------|
| **EntryPoint** | Port d'écoute (ex: `web` sur le port 80, `websecure` sur 443) |
| **Router**     | Règle de routage (quel host/path → quel service)        |
| **Service**    | Destination du trafic (différent du Service Kubernetes) |
| **Middleware** | Transformation du trafic (auth, headers, redirects…)    |

### Configuration via IngressRoute (CRD Traefik)

Traefik propose ses propres Custom Resource Definitions (CRDs), plus puissantes que les Ingress standards :

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: openbao-route
  namespace: traefik
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`openbao.local`)
      kind: Rule
      services:
        - name: openbao-service
          namespace: openbao
          port: 8200
```

---

## 6. OpenBao dans Kubernetes

### Qu'est-ce qu'OpenBao ?

**OpenBao** est un fork open-source de HashiCorp Vault (suite au changement de licence de Vault en 2023). C'est un outil de **gestion des secrets** qui permet de :
- Stocker des secrets de façon chiffrée
- Distribuer des secrets aux applications de façon dynamique
- Gérer des certificats PKI
- Contrôler qui a accès à quoi (policies)

### Pourquoi OpenBao plutôt que les Secrets Kubernetes ?

| Aspect               | Secrets Kubernetes       | OpenBao                         |
|----------------------|--------------------------|---------------------------------|
| Chiffrement          | Base64 (pas chiffré)     | Chiffrement AES-256             |
| Audit                | Limité                   | Audit log complet               |
| Rotation automatique | Non                      | Oui (secrets dynamiques)        |
| PKI / Certificats    | Manuel                   | CA intégrée                     |
| Secrets dynamiques   | Non                      | Oui (DB, AWS, etc.)             |

### Concepts OpenBao

| Concept         | Description                                                    |
|-----------------|----------------------------------------------------------------|
| **Seal/Unseal** | OpenBao démarre "scellé". Il faut le "desceller" avec des clés |
| **Token**       | Mécanisme d'authentification principal                         |
| **Path**        | Organisation des secrets (ex: `secret/myapp/db`)              |
| **Policy**      | Définit les accès (qui peut lire/écrire quel path)             |
| **Auth Method** | Méthode d'auth (Kubernetes, AppRole, LDAP…)                    |
| **Secret Engine** | Backend de stockage (KV, PKI, Database…)                    |

### Intégration avec Kubernetes

OpenBao peut s'authentifier avec Kubernetes et injecter des secrets dans les Pods via :
1. **Vault Agent Injector** : sidecar container qui monte les secrets comme fichiers
2. **Vault Secrets Operator** : operator qui synchronise les secrets en ressources Kubernetes
3. **Lecture directe depuis l'app** via l'API OpenBao

---

## 7. Vers le GitOps

### Pourquoi aller au-delà des manifestes YAML ?

Les manifestes YAML à plat ont des limites :
- Duplication entre environnements (dev/staging/prod)
- Pas de templating (répétition de valeurs)
- Déploiements manuels (`kubectl apply` à la main)
- Pas de traçabilité des changements d'état

### Terraform + Kubernetes

**Terraform** avec le provider Kubernetes (ou Helm) permet de :
- Gérer l'infrastructure ET les déploiements dans le même outil
- Utiliser des variables, des modules, des workspaces
- Avoir un state pour suivre ce qui est déployé

```hcl
# Exemple : créer un namespace avec Terraform
resource "kubernetes_namespace" "traefik" {
  metadata {
    name = "traefik"
    labels = {
      app = "traefik"
    }
  }
}
```

### ArgoCD et le GitOps

**ArgoCD** implémente le principe GitOps : **Git est la source de vérité**.

```
Git Repository (manifestes YAML / Helm charts)
        ↓
   ArgoCD surveille le repo
        ↓
   Détecte une différence entre Git et le cluster
        ↓
   Synchronise automatiquement le cluster
```

**Avantages :**
- Tout changement passe par une Pull Request (revue, historique)
- Rollback = revert du commit Git
- Dashboard de visualisation de l'état des déploiements
- Multi-cluster possible

### Évolution du lab

```
Phase 1 (actuelle) : kubectl apply -f manifests/
Phase 2 (à venir)  : terraform apply
Phase 3 (à venir)  : GitOps avec ArgoCD
```

---

## 8. Glossaire

| Terme               | Définition                                                               |
|---------------------|--------------------------------------------------------------------------|
| **API Server**      | Point d'entrée du control plane Kubernetes                               |
| **ArgoCD**          | Outil GitOps pour le déploiement continu sur Kubernetes                  |
| **Cluster**         | Ensemble de machines (nodes) géré par Kubernetes                         |
| **ConfigMap**       | Objet pour stocker de la configuration non sensible                      |
| **Container**       | Instance d'une image Docker                                              |
| **CRD**             | Custom Resource Definition — étend l'API Kubernetes                      |
| **Deployment**      | Contrôleur qui gère des Pods répliqués                                   |
| **etcd**            | Base de données distribuée du control plane                              |
| **HPA**             | Horizontal Pod Autoscaler — mise à l'échelle automatique                 |
| **Ingress**         | Règle de routage HTTP vers les Services                                  |
| **Ingress Controller** | Composant qui implémente les règles Ingress (ex: Traefik)             |
| **kubectl**         | CLI officielle pour interagir avec Kubernetes                            |
| **kubelet**         | Agent Kubernetes sur chaque nœud worker                                  |
| **Label**           | Paire clé/valeur attachée aux objets pour les sélectionner               |
| **Minikube**        | Outil pour faire tourner un cluster Kubernetes local                     |
| **Namespace**       | Partition logique d'un cluster Kubernetes                                |
| **Node**            | Machine (VM ou physique) dans le cluster                                 |
| **OpenBao**         | Gestionnaire de secrets open-source (fork de Vault)                      |
| **Operator**        | Pattern Kubernetes pour automatiser la gestion d'applications complexes  |
| **PersistentVolume**| Stockage persistant dans le cluster                                      |
| **PVC**             | PersistentVolumeClaim — demande de stockage par un Pod                   |
| **Pod**             | Plus petite unité de déploiement Kubernetes (1+ conteneurs)              |
| **RBAC**            | Role-Based Access Control — gestion des droits                           |
| **ReplicaSet**      | Maintient un nombre N de Pods identiques en vie                          |
| **Secret**          | Objet pour stocker des données sensibles (base64)                        |
| **Selector**        | Filtre sur les labels pour cibler des objets                             |
| **Service**         | Exposition stable d'un ensemble de Pods                                  |
| **ServiceAccount**  | Identité pour les Pods (utilisé par RBAC et OpenBao)                     |
| **StatefulSet**     | Comme Deployment, mais pour les apps avec état (BDD…)                    |
| **Terraform**       | Outil d'Infrastructure as Code                                           |
| **Traefik**         | Ingress controller et reverse proxy                                      |

---

*Support de cours lab Kube, mise à jour : 2026*
