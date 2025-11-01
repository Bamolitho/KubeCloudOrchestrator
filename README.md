# KubeCloudOrchestrator
Projet démontrant l’orchestration d’applications multi-conteneurs avec Kubernetes, de la configuration au déploiement automatisé. Il analyse l’architecture, les avantages et les limites de Kubernetes face à Docker et aux machines virtuelles.

[TOC]

# **Qu’est-ce que Kubernetes ?**

Une **plateforme open-source d’orchestration de conteneurs** développée initialement par Google, aujourd’hui maintenue par la **Cloud Native Computing Foundation (CNCF)**.

Il permet de **déployer, gérer, mettre à l’échelle et maintenir** des applications conteneurisées de manière automatique.

### **Définition simple :**

Kubernetes est au conteneur ce qu’un système d’exploitation est à un ordinateur : il gère les ressources, planifie les tâches et garantit la disponibilité.



# ARCHITECTURE DE KUBERNETES [[2]](#ref2)

![Architecture de kubernetes](./Images/architecture.png)

## Composantes principales

- Master Node (API Server, Controller Manager, Scheduler, etc.)
- Worker Nodes (Kubelet, Kube Proxy, Pod, Container Runtime)

## Fonctionnement global et communication interne

### **1. Vue d’ensemble : le rôle de Kubernetes**

Un **cluster Kubernetes** est composé de :

- **1 Master Node** (ou plusieurs pour la haute disponibilité)
- **Plusieurs Worker Nodes**

### **2. Master Node — le cerveau du cluster**

C’est le **centre de contrôle** du cluster. Il gère **où et quand** exécuter les conteneurs, et surveille leur état.

#### **a. API Server**

- C’est **l’interface centrale** entre les utilisateurs et le cluster.
- Tous les composants (kubectl, nodes, etc.) communiquent via cette API REST.
- Chaque commande kubectl (ex. kubectl get pods) passe par elle.

**Rôle :** point d’entrée unique pour gérer le cluster.

#### **b. Scheduler**

- Décide **sur quel nœud** chaque Pod doit être exécuté.
- Il se base sur :
  - les ressources disponibles (CPU, RAM),
  - les contraintes de l’application (affinités, labels, etc.).

**Rôle :** placement intelligent des Pods dans le cluster.

#### **c. Controller Manager**

- Surveille en permanence l’état du cluster.
- Compare **l’état désiré** (défini par les fichiers YAML) avec **l’état actuel**.
- Si un Pod crash, il le redéploie automatiquement.

**Rôle :** maintenir le cluster conforme à la configuration voulue.

#### **d. etcd**

- Base de données clé-valeur distribuée (type NoSQL).
- Contient **toute la configuration du cluster** (état des Pods, des Services, des Secrets, etc.).

**Rôle :** stockage central de l’état global du cluster.

#### **e. Authentication & Authorization**

- Gère les identités et permissions (RBAC, comptes de service…).
- Contrôle **qui peut faire quoi** dans le cluster.

**Rôle :** sécurité et contrôle d’accès.

### **3. Worker Nodes — les muscles du cluster**

Chaque Node (machine) exécute **les conteneurs réels**. Le Master leur dit quoi faire.

#### **a. Kubelet**

- Agent qui communique avec le Master.
- Reçoit les instructions du Scheduler (exécuter un Pod).
- Surveille la santé des conteneurs sur le node.

**Rôle :** faire exécuter les Pods et rapporter leur état.

#### **b. Proxy (kube-proxy)**

- Gère le **trafic réseau** entrant et sortant des Pods.
- Met en place le **load balancing** interne et les **règles d’accès**.

**Rôle :** assurer la communication entre Pods et entre l’extérieur et le cluster.

#### **c. Docker (ou autre runtime comme containerd, CRI-O)**

- Exécute les **conteneurs** eux-mêmes.
- Kubernetes n’exécute pas directement les conteneurs — il délègue à Docker/containerd.

**Rôle :** moteur d’exécution des conteneurs.

#### **d. Pods**

- **Unité de base d’exécution** dans Kubernetes.
- Contient un ou plusieurs conteneurs qui partagent le même espace réseau et stockage.

**Rôle :** encapsule les conteneurs pour leur fournir un environnement cohérent.

### **4. Communication et workflow**

1. L’utilisateur ou DevOps exécute une commande via **kubectl (CLI)**.
2. Cette commande est envoyée au **API Server** du Master Node.
3. Le Master enregistre la configuration dans **etcd**.
4. Le **Scheduler** choisit un Node adapté.
5. Le **Kubelet** du Node reçoit la consigne, et crée le Pod avec Docker.
6. Le **kube-proxy** s’assure que le trafic réseau fonctionne correctement.
7. Le **Controller Manager** vérifie que le Pod tourne bien.

### **5. Internet et Services**

Le schéma montre aussi une connexion avec Internet. Les Pods ne sont **pas directement exposés au monde extérieur**. Kubernetes utilise des **Services** (LoadBalancer, NodePort, Ingress) pour exposer les applications.

### **En résumé :**

| **Élément**            | **Rôle**                                |
| ---------------------- | --------------------------------------- |
| **API Server**         | Point central de communication          |
| **Scheduler**          | Choisit où exécuter les Pods            |
| **Controller Manager** | Surveille et maintient l’état désiré    |
| **etcd**               | Base de données du cluster              |
| **Kubelet**            | Exécute et surveille les Pods           |
| **kube-proxy**         | Gère la communication réseau            |
| **Docker/containerd**  | Exécute les conteneurs                  |
| **Pod**                | Unité de base d’exécution               |
| **kubectl**            | Interface CLI pour interagir avec l’API |

### **Exemple : déploiement d’une app Flask avec** **kubectl apply**

#### **1. Lancer une commande :** 

```bash
kubectl apply -f flask-deployment.yaml
```

#### **2. Communication entre les composants :**

Voici ce qui se passe étape par étape :

1. **kubectl (CLI)** envoie notre requête au **Kubernetes API Server** (dans le **Master Node**).
   → C’est une requête HTTP REST vers https://<master-ip>:6443.

2. **API Server** vérifie son identité via **Authentication & Authorization**

   → Il s’assure qu'on a le droit de créer un *Deployment*.

3. Une fois validée, la ressource (*Deployment*) est enregistrée dans la **Distributed Storage** (souvent *etcd*).
    → etcd conserve l’état désiré du cluster : “je veux 2 pods Flask”.

4. Le **Scheduler** est notifié qu’un nouveau pod doit être créé.
    → Il choisit un **Worker Node** où exécuter ce pod (selon la charge, la mémoire, etc.).

5. Le **Controller Manager** surveille la différence entre :

   - l’état désiré (2 pods)
   - et l’état actuel (0 pod)
      → Il demande au **Kubelet** du nœud choisi de lancer les conteneurs.

6. Sur le **Node** :

   - **Kubelet** reçoit l’instruction du **Master Node**.
   - Il demande à **Docker** (ou un autre moteur de conteneur) de **tirer l’image** (flask-hello:1.0) et de **démarrer le conteneur**.
   - Le **Pod** devient actif et envoie un signal de santé au **Kubelet**.

7. Le **Kube Proxy** sur le même Node s’assure que les autres pods et services peuvent **communiquer** entre eux.
    → Par exemple, un autre pod peut accéder à l'app Flask via le service exposé sur le port 31181.

### **Le flux :**

kubectl → API Server → Authentication → etcd

   ↓

Scheduler → choisit un Node

   ↓

Controller Manager → informe Kubelet

   ↓

Kubelet → demande à Docker de lancer le conteneur

   ↓

Kube Proxy → gère la communication réseau entre les pods



### Modèle de virtualisation et isolation des conteneurs

------



## **Étude de cas : Déploiement d’une application Flask avec Kubernetes** [[1]](#ref1)

### Description de l’application Flask

### Environnement utilisé (Minikube, Docker, kubectl)



### Structure du projet 

Obtenue via la commande suivante

```bash
make tree
```

```basic
.
├── app
│   ├── app.py
│   └── __init__.py
├── conteneurs_système_de_k8s_.md
├── Dockerfile
├── Images
│   └── architecture.png
├── install_kubernetes_env.sh
├── k8s
│   ├── base
│   │   ├── configmap.yaml
│   │   ├── deployment.yaml
│   │   ├── kustomization.yaml
│   │   ├── secret.yaml
│   │   └── service.yaml
│   └── overlays
│       ├── dev
│       │   └── kustomization.yaml
│       └── prod
│           └── kustomization.yaml
├── LICENSE
├── Makefile
├── README.md
├── requirements.txt
└── run_system.sh
```

### Étapes de déploiement automatisé

- Build de l’image
- Création du cluster Minikube
- Déploiement des ressources
- Accès via service NodePort

### Vérification et tests du déploiement

------



# **Analyse critique de Kubernetes**

## **Pourquoi utiliser k8s ?**

Sans Kubernetes, les développeurs déploient et gèrent les conteneurs (Docker, par exemple) manuellement.
Mais dès qu’on parle de **plusieurs serveurs, plusieurs conteneurs**, les problèmes apparaissent :

- Comment redémarrer un conteneur qui crash ?
- Comment distribuer la charge sur plusieurs machines ?
- Comment mettre à jour sans interruption ?
- Comment gérer des centaines de conteneurs ?

Kubernetes **automatise** tout ça.



### **Ses grands avantages :**

1. **Haute disponibilité** – redémarre automatiquement les conteneurs en cas de panne.
2. **Scalabilité** – augmente ou diminue automatiquement le nombre d’instances.
3. **Load balancing** – distribue le trafic entre les pods.
4. **Mises à jour continues** (rolling updates).
5. **Gestion simplifiée du déploiement** sur des environnements hybrides (on-premise ou cloud).

## **k8s apporte des solutions aux problèmes suivants**

| **Problème (avant K8s)**         | **Solution avec Kubernetes**                     |
| -------------------------------- | ------------------------------------------------ |
| Serveurs difficiles à configurer | Définis l'infra en YAML (Infrastructure-as-Code) |
| Déploiements manuels             | Automatisation via kubectl ou CI/CD              |
| Downtime lors des updates        | Rolling updates + rollbacks                      |
| Scaling manuel                   | Horizontal Pod Autoscaler                        |
| Monitoring complexe              | Intégration Prometheus / Grafana                 |
| Load balancing artisanal         | Services intégrés                                |
| Pannes imprévisibles             | Self-healing automatique                         |



### Inconvénients et limites

### Bonnes pratiques d’utilisation

------



# **Comparaison technique**

### Kubernetes vs Docker (Docker Compose)

### Kubernetes vs Machines Virtuelles

### Tableau comparatif : Maintenance, Scalabilité, Interaction événementielle

------



# **Conclusion et perspectives**

### Synthèse des apprentissages

### Améliorations futures (CI/CD, multi-cluster, Cloud public, etc.)

------



# RÉFÉRENCES

[<a id="ref1">1</a>] [https://github.com/Bamolitho/InsideKubernetes](https://github.com/Bamolitho/InsideKubernetes)  

[<a id="ref2">2</a>] [**STRATOSCALE EVERYTHING KUBERNETES: A PRACTICAL GUIDE**](https://iamondemand.com/wp-content/uploads/2019/11/Kubernetes-eBook.pdf)  

**Tech With Nana** : https://youtu.be/-ykwb1d0DXU?si=Mny3zBcnFVE5bXBt