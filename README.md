#  Kubernetes Lab — Introduction & Guide du projet

> Atelier pratique d'introduction à Kubernetes avec un lab local sous **Minikube** (Debian / WSL2).  
> Stack initiale : **Traefik** (ingress controller) + **OpenBao** (gestion des secrets).  
> Évolution prévue vers **Terraform** + **ArgoCD** pour une gestion GitOps.

---

## Objectifs

- Comprendre les concepts fondamentaux de Kubernetes
- Monter un cluster local fonctionnel avec Minikube sur Debian/WSL2
- Déployer des applications réelles (Traefik, OpenBao) en manifestes YAML
- Poser les bases d'une infrastructure reproductible et versionnée

Dans un premier temps nous utiliserons des manifestes afin d'en comprendre les rouages

---

## Structure du projet

```
k8s-lab/
│
├── README.md               ← Vous êtes ici
├── cours.md                ← Support de formation (concepts Kubernetes)
├── quickstart.md           ← Installation Minikube + cheat sheet + déploiements
│
├── manifests/
│   ├── traefik/            ← Manifestes Kubernetes pour Traefik
│   │   ├── namespace.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ingress.yaml
│   │
│   └── openbao/            ← Manifestes Kubernetes pour OpenBao
│       ├── namespace.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       └── config.yaml
│
└── terraform/              ← (À venir) Infrastructure as Code
    └── ...
```

---

## Philosophie du lab

### Un Namespace = Une Application

Dans ce lab, chaque application est isolée dans son propre **Namespace** Kubernetes.  
Cela reflète une bonne pratique réelle : isolation logique, gestion des droits et des ressources par périmètre applicatif.

| Namespace       | Application        | Rôle                               |
|-----------------|--------------------|------------------------------------|
| `traefik`       | Traefik            | Ingress controller / reverse proxy |
| `openbao`       | OpenBao            | Gestionnaire de secrets            |
| *(à venir)*     |  apps              | tbd                                |

---

## Stack technique

| Composant      | Rôle                                      | Lien                                              |
|----------------|-------------------------------------------|---------------------------------------------------|
| **Minikube**   | Cluster Kubernetes local                  | https://minikube.sigs.k8s.io                      |
| **kubectl**    | CLI officielle Kubernetes                 | https://kubernetes.io/docs/reference/kubectl/     |
| **Traefik**    | Ingress controller & reverse proxy        | https://traefik.io                                |
| **OpenBao**    | Fork open-source de HashiCorp Vault       | https://openbao.org                               |
| **Terraform**  | *(à venir)* Infrastructure as Code        | https://www.terraform.io                          |
| **ArgoCD**     | *(à venir)* GitOps / déploiement continu  | https://argo-cd.readthedocs.io                    |

---

## Par où commencer ?

1. **Lire le support de cours** → [`COURS.md`](./COURS.md)  
   Concepts clés : Pods, Deployments, Services, Ingress, Namespaces, ConfigMaps, Secrets…

2. **Suivre le Quickstart** → [`QUICKSTART.md`](./QUICKSTART.md)  
   Installation de Minikube, cheat sheet `kubectl`, déploiement de Traefik et OpenBao.

3. **Explorer les manifestes** → dossier `manifests/`  
   Lire, modifier, appliquer les fichiers YAML pour comprendre ce qui tourne dans le cluster.

---

##  Roadmap

- [x] Lab local Minikube (Debian/WSL2)
- [x] Déploiement Traefik via manifestes YAML
- [x] Déploiement OpenBao via manifestes YAML
- [ ] Gestion de l'infrastructure avec **Terraform** (provider Kubernetes + Helm)
- [ ] GitOps avec **ArgoCD** — synchronisation depuis un dépôt Git
- [ ] Ajout d'applications exemple (app web, base de données…)
- [ ] RBAC, NetworkPolicies, Resource Quotas

---

## Public visé

Ce lab est avant tout destiné à mes amis, et à toutes personnes désireuses d'avoir un premier contact avec Kube. L'utilisation de minikube permet une approche "homelab" afin de se familiariser avec le vocabulaire, les commandes, et le fonctionnement de kubernetes.
Cela étant dit le lab est conçu pour des personnes ayant des bases en ligne de commande Linux et une première expérience avec Docker. Aucune connaissance préalable de Kubernetes n'est requise.

---

## Prérequis système

- Windows 10/11 avec **WSL2** activé
- Distribution **Debian** installée sous WSL2
- Au moins **4 Go de RAM** allouée à WSL2 (8 Go recommandés)
- **Docker Desktop** ou `docker` natif dans WSL2

> Pour augmenter la RAM allouée à WSL2, éditer `C:\Users\<user>\.wslconfig` :
> ```ini
> [wsl2]
> memory=8GB
> processors=4
> ```

---

*Bonne lecture !*
