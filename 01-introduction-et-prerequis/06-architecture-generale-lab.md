🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.6 - Architecture générale du lab

## Introduction

Comprendre l'architecture de votre laboratoire MicroK8s est essentiel pour tirer le meilleur parti de votre environnement d'apprentissage. Cette section présente une vue d'ensemble complète de l'architecture, depuis les composants de base jusqu'aux configurations avancées, en expliquant comment chaque élément s'articule pour créer un environnement Kubernetes fonctionnel et évolutif.

## Vue d'ensemble de l'architecture

### Architecture conceptuelle d'un lab MicroK8s

```
┌────────────────────────────────────────────────────────────┐
│                        INTERNET                            │
└────────────────────┬───────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────┐
│                  Routeur / Box Internet                    │
│                    (Port forwarding)                       │
└────────────────────┬───────────────────────────────────────┘
                     │
                     ▼
┌───────────────────────────────────────────────────────────┐
│                    Machine Hôte                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │                  MicroK8s Cluster                   │  │
│  │  ┌────────────────────────────────────────────────┐ │  │
│  │  │            Control Plane                       │ │  │
│  │  │  • API Server  • Scheduler  • Controller       │ │  │
│  │  │  • etcd        • Cloud Controller Manager      │ │  │
│  │  └────────────────────────────────────────────────┘ │  │
│  │                                                     │  │
│  │  ┌────────────────────────────────────────────────┐ │  │
│  │  │            Data Plane (Worker)                 │ │  │
│  │  │  • Kubelet    • Kube-proxy  • Container Runtime│ │  │
│  │  │  • Pods       • Services    • Volumes          │ │  │
│  │  └────────────────────────────────────────────────┘ │  │
│  │                                                     │  │
│  │  ┌────────────────────────────────────────────────┐ │  │
│  │  │            Addons & Extensions                 │ │  │
│  │  │  • Ingress    • DNS         • Dashboard        │ │  │
│  │  │  • Monitoring • Storage     • Registry         │ │  │
│  │  └────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
```

## Composants fondamentaux

### 1. Control Plane (Plan de contrôle)

Le Control Plane est le cerveau de votre cluster Kubernetes. Il prend les décisions globales et répond aux événements du cluster.

**API Server (kube-apiserver) :**
```
Rôle : Point d'entrée unique pour toutes les opérations
├── Reçoit les commandes kubectl
├── Valide et traite les requêtes REST
├── Met à jour l'état dans etcd
└── Notifie les autres composants des changements
```

**Scheduler (kube-scheduler) :**
```
Rôle : Décide où placer les nouveaux pods
├── Évalue les ressources disponibles
├── Applique les contraintes (affinité, taints)
├── Optimise la répartition de charge
└── Assigne les pods aux nodes
```

**Controller Manager :**
```
Rôle : Maintient l'état désiré du cluster
├── Node Controller : Surveille la santé des nodes
├── Replication Controller : Maintient le nombre de replicas
├── Endpoints Controller : Gère les services endpoints
├── Service Account Controller : Crée les comptes par défaut
└── Et plus de 30 autres controllers...
```

**etcd :**
```
Rôle : Base de données distribuée clé-valeur
├── Stocke toute la configuration du cluster
├── État actuel et désiré des ressources
├── Haute disponibilité avec consensus Raft
└── Source de vérité unique
```

### 2. Data Plane (Plan de données)

Le Data Plane exécute réellement vos applications conteneurisées.

**Kubelet :**
```
Rôle : Agent sur chaque node
├── Communique avec l'API Server
├── Démarre et arrête les containers
├── Rapporte l'état des pods
├── Gère les volumes et les secrets
└── Effectue les health checks
```

**Kube-proxy :**
```
Rôle : Gestion du réseau pour les services
├── Maintient les règles réseau (iptables/IPVS)
├── Load balancing entre les pods
├── Permet la communication service-to-service
└── Gère les NodePorts et ClusterIPs
```

**Container Runtime (containerd) :**
```
Rôle : Exécute les containers
├── Pull des images depuis les registries
├── Création et destruction des containers
├── Gestion des namespaces et cgroups
└── Interface avec le kernel Linux
```

## Architecture réseau du lab

### Modèle de réseau Kubernetes

```
┌─────────────────────────────────────────────────────────────┐
│                     Réseau Externe                          │
│                    (Internet/LAN)                           │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│               Node Network (192.168.1.0/24)                 │
│                  IP de votre machine                        │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│            Cluster Network (10.152.183.0/24)                │
│                  Services Kubernetes                        │
│  ┌─────────────────────────────────────────────────────┐    │
│  │   • ClusterIP : Accessible dans le cluster          │    │
│  │   • NodePort : Accessible depuis l'extérieur        │    │
│  │   • LoadBalancer : Via MetalLB pour bare metal      │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Pod Network (10.1.0.0/16)                      │
│                    Pods individuels                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │  Pod A   │  │  Pod B   │  │  Pod C   │  │  Pod D   │     │
│  │10.1.1.2  │  │10.1.1.3  │  │10.1.2.2  │  │10.1.2.3  │     │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### Types de réseaux dans le lab

**1. Réseau Host :**
- IP de votre machine physique/VM
- Accessible depuis votre réseau local
- Utilisé pour l'accès initial à MicroK8s

**2. Réseau Cluster (Services) :**
- IPs virtuelles pour les services
- Load balancing automatique
- Trois types : ClusterIP, NodePort, LoadBalancer

**3. Réseau Pod :**
- Chaque pod a une IP unique
- Communication directe pod-to-pod
- Géré par CNI (Container Network Interface)

### Configuration DNS du lab

```
┌─────────────────────────────────────────────────────────────┐
│                    DNS Externe                              │
│              (Votre domaine : lab.example.com)              │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                  CoreDNS (Interne)                          │
│                    10.152.183.10                            │
│  ┌─────────────────────────────────────────────────────┐    │
│  │   Résolution interne :                              │    │
│  │   • service.namespace.svc.cluster.local             │    │
│  │   • pod-ip.namespace.pod.cluster.local              │    │
│  │   • Forwarding vers DNS externe                     │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

## Architecture de stockage

### Hiérarchie du stockage

```
┌─────────────────────────────────────────────────────────────┐
│                   Stockage Physique                         │
│               Disque de la machine hôte                     │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                  Storage Classes                            │
│  ┌─────────────────────────────────────────────────────┐    │
│  │   • microk8s-hostpath (par défaut)                  │    │
│  │   • nfs-client (si configuré)                       │    │
│  │   • ceph-rbd (pour cluster avancé)                  │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│             Persistent Volumes (PV)                         │
│         Ressources de stockage provisionnées                │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│         Persistent Volume Claims (PVC)                      │
│           Demandes de stockage par les pods                 │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    Pod Volumes                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │ConfigMap │  │ Secret   │  │HostPath  │  │EmptyDir  │     │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### Types de volumes dans le lab

| Type | Usage | Persistance | Cas d'usage lab |
|------|-------|-------------|-----------------|
| **emptyDir** | Temporaire | Pod lifetime | Cache, données temporaires |
| **hostPath** | Chemin local | Permanent | Développement, logs |
| **configMap** | Configuration | Cluster | Fichiers config |
| **secret** | Données sensibles | Cluster | Mots de passe, certificats |
| **persistentVolumeClaim** | Stockage durable | Permanent | Bases de données |

## Architecture des addons

### Écosystème d'addons MicroK8s

```
┌─────────────────────────────────────────────────────────────┐
│                    Addons MicroK8s                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Networking           Observability        Storage          │
│  ┌─────────┐         ┌─────────────┐     ┌──────────┐       │
│  │ DNS     │         │ Dashboard   │     │ Hostpath │       │
│  │ Ingress │         │ Prometheus  │     │ OpenEBS  │       │
│  │ MetalLB │         │ Grafana     │     │ Mayastor │       │
│  │ Istio   │         │ Jaeger      │     └──────────┘       │
│  └─────────┘         │ Fluentd     │                        │
│                      └─────────────┘                        │
│                                                             │
│  Security            Developer Tools       Operations       │
│  ┌─────────┐         ┌─────────────┐     ┌──────────┐       │
│  │ RBAC    │         │ Registry    │     │ Metrics  │       │
│  │ Cert-   │         │ Helm        │     │ GPU      │       │
│  │ Manager │         │ Knative     │     │ HA       │       │
│  └─────────┘         └─────────────┘     └──────────┘       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Addons essentiels pour le lab

**1. CoreDNS (dns) :**
- Résolution DNS interne
- Service discovery
- Configuration automatique

**2. NGINX Ingress Controller :**
- Routage HTTP/HTTPS
- Virtual hosts
- SSL termination

**3. Dashboard :**
- Interface web
- Visualisation des ressources
- Gestion simplifiée

**4. Prometheus + Grafana :**
- Métriques système
- Alerting
- Dashboards visuels

**5. Registry :**
- Registry Docker privé
- Stockage local d'images
- Développement rapide

## Architecture d'exposition des services

### Flux de trafic entrant

```
┌─────────────────────────────────────────────────────────────┐
│     Utilisateur (Navigateur web)                            │
│     https://app.monlab.com                                  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│     DNS Public                                              │
│     app.monlab.com → IP publique                            │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│     Routeur / Box (NAT)                                     │
│     Port 443 → IP locale:443                                │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────┐
│     Ingress Controller (NGINX)                             │
│     Écoute sur ports 80/443                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │   Rules:                                            │   │
│  │   - app.monlab.com → service-app:8080               │   │
│  │   - api.monlab.com → service-api:3000               │   │
│  │   - grafana.monlab.com → service-grafana:3000       │   │
│  └─────────────────────────────────────────────────────┘   │
└──────────────────────┬─────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│     Service Kubernetes                                      │
│     Type: ClusterIP                                         │
│     Selector: app=monapp                                    │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌───────────────────────────────────────────────────────────┐
│     Pods                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│  │  Pod 1   │  │  Pod 2   │  │  Pod 3   │                 │
│  │  app:v1  │  │  app:v1  │  │  app:v1  │                 │
│  └──────────┘  └──────────┘  └──────────┘                 │
└───────────────────────────────────────────────────────────┘
```

## Architecture de sécurité

### Couches de sécurité du lab

```
┌─────────────────────────────────────────────────────────────┐
│                 Sécurité Périmétrique                       │
│  • Firewall         • Fail2ban                              │
│  • Port forwarding  • VPN (optionnel)                       │
└─────────────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                 Sécurité Réseau                             │
│  • Network Policies • Segmentation                          │
│  • TLS/SSL         • Service Mesh (Istio)                   │
└─────────────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                 Sécurité Kubernetes                         │
│  • RBAC            • Pod Security Policies                  │
│  • Service Accounts • Admission Controllers                 │
└─────────────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                 Sécurité Applicative                        │
│  • Secrets         • ConfigMaps                             │
│  • Image scanning  • Runtime protection                     │
└─────────────────────────────────────────────────────────────┘
```

### Composants de sécurité

**RBAC (Role-Based Access Control) :**
```yaml
Principe : Qui peut faire quoi
├── Users/ServiceAccounts : Identités
├── Roles/ClusterRoles : Permissions
├── RoleBindings : Associations
└── Principe du moindre privilège
```

**Secrets Management :**
```yaml
Types de secrets :
├── Docker registry : Authentification images
├── TLS : Certificats SSL
├── Opaque : Données arbitraires
└── Service Account : Tokens d'authentification
```

**Network Policies :**
```yaml
Contrôle du trafic :
├── Ingress : Qui peut accéder au pod
├── Egress : Où le pod peut se connecter
├── Default deny : Blocage par défaut
└── Explicit allow : Autorisation explicite
```

## Architecture de monitoring

### Stack d'observabilité

```
┌───────────────────────────────────────────────────────────┐
│                    Visualization Layer                    │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   │
│  │   Grafana    │   │  Dashboard   │   │   Lens/K9s   │   │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘   │
└─────────┼──────────────────┼──────────────────┼───────────┘
          │                  │                  │
          ▼                  ▼                  ▼
┌───────────────────────────────────────────────────────────┐
│                     Data Storage Layer                    │
│  ┌──────────────┐  ┌──────────────┐    ┌──────────────┐   │
│  │ Prometheus   │  │   InfluxDB   │    │ Elasticsearch│   │
│  │  (Metrics)   │  │   (Metrics)  │    │    (Logs)    │   │
│  └──────┬───────┘  └──────┬───────┘    └──────┬───────┘   │
└─────────┼─────────────────┼───────────────────┼───────────┘
          │                 │                   │
          ▼                 ▼                   ▼
┌───────────────────────────────────────────────────────────┐
│                    Collection Layer                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │Node Exporter │  │Kube-state    │  │   Fluentd    │     │
│  │              │  │  Metrics     │  │              │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  cAdvisor    │  │   Metrics    │  │   Jaeger     │     │
│  │              │  │   Server     │  │   (Traces)   │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└───────────────────────────────────────────────────────────┘
```

### Métriques collectées

**Métriques système :**
- CPU, RAM, Disk, Network
- Temps de réponse
- Taux d'erreur
- Saturation

**Métriques Kubernetes :**
- Pods running/pending/failed
- Deployments status
- Resource utilization
- API server latency

**Métriques application :**
- Business metrics
- Custom metrics
- Performance indicators
- User experience

## Architecture CI/CD

### Pipeline de déploiement

```
┌─────────────────────────────────────────────────────────────┐
│                    Developer Workstation                    │
│                         git push                            │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    Git Repository                           │
│                  (GitHub/GitLab/Gitea)                      │
└──────────────────────┬──────────────────────────────────────┘
                       │ Webhook/Poll
                       ▼
┌────────────────────────────────────────────────────────────┐
│                    CI/CD System                            │
│              (Jenkins/GitLab CI/ArgoCD)                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │   1. Clone repo                                     │   │
│  │   2. Run tests                                      │   │
│  │   3. Build Docker image                             │   │
│  │   4. Push to registry                               │   │
│  │   5. Update Kubernetes manifests                    │   │
│  │   6. Deploy to MicroK8s                             │   │
│  └─────────────────────────────────────────────────────┘   │
└──────────────────────┬─────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────┐
│                    MicroK8s Cluster                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │   Namespaces:                                       │   │
│  │   • development : Derniers commits                  │   │
│  │   • staging : Pre-production                        │   │
│  │   • production : Version stable                     │   │
│  └─────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────┘
```

## Évolution de l'architecture

### Phase 1 : Lab basique (Débutant)

```
┌────────────────────────────────────────────────────────────┐
│               Single Node MicroK8s                         │
│  • Control Plane + Worker sur même machine                 │
│  • Addons : DNS, Dashboard                                 │
│  • 2-3 applications simples                                │
│  • Accès local uniquement                                  │
└────────────────────────────────────────────────────────────┘
```

### Phase 2 : Lab intermédiaire

```
┌─────────────────────────────────────────────────────────────┐
│               Single Node Enhanced                          │
│  • Ingress Controller + Cert-Manager                        │
│  • Domaine externe + HTTPS                                  │
│  • Monitoring (Prometheus/Grafana)                          │
│  • Registry privé                                           │
│  • 5-10 applications                                        │
└─────────────────────────────────────────────────────────────┘
```

### Phase 3 : Lab avancé

```
┌───────────────────────────────────────────────────────────┐
│                Multi-Node Cluster                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   Master    │  │   Worker 1  │  │   Worker 2  │        │
│  │Control Plane│  │  Workloads  │  │  Workloads  │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
│  • High Availability                                      │
│  • Load Balancing                                         │
│  • Distributed Storage                                    │
│  • Service Mesh (Istio)                                   │
│  • GitOps (ArgoCD)                                        │
└───────────────────────────────────────────────────────────┘
```

### Phase 4 : Lab production-like

```
┌────────────────────────────────────────────────────────────┐
│                  Hybrid Architecture                       │
│  ┌──────────────────────────┐  ┌────────────────────────┐  │
│  │     On-Premise Lab       │  │    Cloud Resources     │  │
│  │  • Development/Test      │  │  • Production replicas │  │
│  │  • CI/CD Pipeline        │  │  • Backup/DR           │  │
│  │  • Local monitoring      │  │  • Public services     │  │
│  └────────────┬─────────────┘  └───────────┬────────────┘  │
│               │        Federation           │              │
│               └─────────────┬───────────────┘              │
│                             ▼                              │
│                   Unified Management                       │
└────────────────────────────────────────────────────────────┘
```

## Patterns architecturaux communs

### Pattern 1 : Microservices

```
┌───────────────────────────────────────────────────────────┐
│                    Ingress Controller                     │
└────────┬──────────┬──────────┬──────────┬─────────────────┘
         │          │          │          │
         ▼          ▼          ▼          ▼
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│Frontend  │  │   API    │  │   Auth   │  │  Admin   │
│  Service │  │ Gateway  │  │ Service  │  │  Panel   │
└────┬─────┘  └────┬─────┘  └──────────┘  └──────────┘
     │             │
     │      ┌──────┴──────┬──────────┬──────────┐
     │      ▼             ▼          ▼          ▼
     │ ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
     │ │ Service A│  │ Service B│  │ Service C│  │ Service D│
     │ └──────────┘  └──────────┘  └──────────┘  └──────────┘
     │                    │               │
     └────────┬───────────┴───────────────┘
              ▼
         ┌──────────┐  ┌──────────┐  ┌──────────┐
         │PostgreSQL│  │  Redis   │  │ RabbitMQ │
         └──────────┘  └──────────┘  └──────────┘
```

### Pattern 2 : Multi-environnement

```
┌─────────────────────────────────────────────────────────────┐
│                    MicroK8s Cluster                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Namespace: dev         Namespace: staging    Namespace: prod│
│  ┌──────────────┐      ┌──────────────┐     ┌──────────────┐│
│  │ App v2.1-dev │      │ App v2.0-rc1 │     │ App v1.9     ││
│  │ DB dev data  │      │ DB test data │     │ DB prod copy ││
│  │ Debug enabled│      │ Monitoring on│     │ HA enabled   ││
│  └──────────────┘      └──────────────┘     └──────────────┘│
│                                                             │
│  Resources:            Resources:           Resources:      │
│  • 1 replica           • 2 replicas          • 3 replicas   │
│  • 512MB RAM           • 1GB RAM             • 2GB RAM      │
│  • No limits           • CPU limits          • Strict limits│
└─────────────────────────────────────────────────────────────┘
```

### Pattern 3 : Event-Driven

```
┌──────────────────────────────────────────────────────────┐
│                    Event Sources                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │   API    │  │   Cron   │  │  Webhook │  │   File   │  │
│  │  Calls   │  │   Jobs   │  │  Events  │  │  Upload  │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
└───────┼─────────────┼─────────────┼─────────────┼────────┘
        └─────────────┴─────────────┴─────────────┘
                             │
                             ▼
┌────────────────────────────────────────────────────────────┐
│                    Message Queue                           │
│                 (RabbitMQ/Kafka/NATS)                      │
└──────────┬────────────┬────────────┬───────────────────────┘
           │            │            │
           ▼            ▼            ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Processor 1 │ │  Processor 2 │ │  Processor 3 │
│  (Scaling)   │ │  (Transform) │ │  (Notify)    │
└──────────────┘ └──────────────┘ └──────────────┘
```

## Considérations de performance

### Répartition des ressources

```
┌────────────────────────────────────────────────────────────┐
│            Ressources Totales Machine (100%)               │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌─────────────────────────────────────────────────┐       │
│  │         Système d'exploitation (10-15%)         │       │
│  │  • Kernel, systemd, networking                  │       │
│  └─────────────────────────────────────────────────┘       │
│                                                            │
│  ┌─────────────────────────────────────────────────┐       │
│  │         MicroK8s Core Components (15-20%)       │       │
│  │  • Control plane, etcd, kubelet                 │       │
│  └─────────────────────────────────────────────────┘       │
│                                                            │
│  ┌─────────────────────────────────────────────────┐       │
│  │         System Addons (10-15%)                  │       │
│  │  • DNS, Ingress, Dashboard                      │       │
│  └─────────────────────────────────────────────────┘       │
│                                                            │
│  ┌─────────────────────────────────────────────────┐       │
│  │         Monitoring Stack (15-20%)               │       │
│  │  • Prometheus, Grafana, Exporters               │       │
│  └─────────────────────────────────────────────────┘       │
│                                                            │
│  ┌─────────────────────────────────────────────────┐       │
│  │         Applications (35-45%)                   │       │
│  │  • Vos pods, services, jobs                     │       │
│  └─────────────────────────────────────────────────┘       │
│                                                            │
│  ┌─────────────────────────────────────────────────┐       │
│  │         Buffer/Cache (5-10%)                    │       │
│  │  • Marge de sécurité                            │       │
│  └─────────────────────────────────────────────────┘       │
└────────────────────────────────────────────────────────────┘
```

### Optimisations architecturales

**1. Placement des pods :**
```yaml
Stratégies :
├── Node Affinity : Préférence de nodes
├── Pod Affinity : Colocalisation
├── Pod Anti-Affinity : Séparation
└── Taints/Tolerations : Isolation
```

**2. Gestion des ressources :**
```yaml
Limites et requêtes :
├── Requests : Garantie minimale
├── Limits : Maximum autorisé
├── QoS Classes : Guaranteed/Burstable/BestEffort
└── Resource Quotas : Par namespace
```

**3. Scaling strategies :**
```yaml
Types de scaling :
├── HPA : Horizontal Pod Autoscaler
├── VPA : Vertical Pod Autoscaler
├── Cluster Autoscaler : Ajout de nodes
└── Manual scaling : Contrôle direct
```

## Architecture de backup et restauration

### Stratégie de sauvegarde

```
┌───────────────────────────────────────────────────────────┐
│                    Éléments à sauvegarder                 │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  Configuration                     Data                   │
│  ┌──────────────┐                 ┌──────────────┐        │
│  │ Manifests    │                 │ Volumes      │        │
│  │ YAML files   │                 │ PV data      │        │
│  └──────┬───────┘                 └──────┬───────┘        │
│         │                                 │               │
│  ┌──────────────┐                 ┌──────────────┐        │
│  │ Secrets      │                 │ Databases    │        │
│  │ ConfigMaps   │                 │ Backups      │        │
│  └──────┬───────┘                 └──────┬───────┘        │
│         │                                 │               │
│  ┌──────────────┐                 ┌──────────────┐        │
│  │ Certificates │                 │ Application  │        │
│  │ Keys         │                 │ State        │        │
│  └──────┬───────┘                 └──────┬───────┘        │
└─────────┼─────────────────────────────────┼───────────────┘
          │                                 │
          ▼                                 ▼
┌───────────────────────────────────────────────────────────┐
│                    Backup Destinations                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Local Disk   │  │   NAS/NFS    │  │ Cloud Storage│     │
│  │ /backups/    │  │              │  │   (S3/GCS)   │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└───────────────────────────────────────────────────────────┘
```

### Niveaux de backup

| Niveau | Quoi | Fréquence | Retention | Restauration |
|--------|------|-----------|-----------|--------------|
| **L1 : Config** | Manifests YAML | Temps réel (Git) | Infinie | Minutes |
| **L2 : État** | etcd snapshot | Quotidien | 7 jours | 30 minutes |
| **L3 : Données** | PV backups | Quotidien | 30 jours | 1-2 heures |
| **L4 : Images** | Registry backup | Hebdomadaire | 3 mois | 2-4 heures |
| **L5 : Complet** | Full cluster | Mensuel | 6 mois | 4-8 heures |

## Architecture réseau avancée

### Segmentation réseau

```
┌───────────────────────────────────────────────────────────┐
│                    DMZ Network                            │
│              Services exposés publiquement                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Ingress    │  │ LoadBalancer │  │   WAF        │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└──────────────────────┬────────────────────────────────────┘
                       │
                  [Firewall Rules]
                       │
┌──────────────────────┴────────────────────────────────────┐
│                  Application Network                      │
│                  Services métier                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Frontend   │  │     API      │  │   Workers    │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└──────────────────────┬────────────────────────────────────┘
                       │
               [Network Policies]
                       │
┌──────────────────────┴────────────────────────────────────┐
│                    Data Network                           │
│                  Services de données                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Database   │  │    Cache     │  │   Storage    │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└───────────────────────────────────────────────────────────┘
```

### Service Mesh (Istio)

```
┌───────────────────────────────────────────────────────────┐
│                    Istio Control Plane                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │    Pilot     │  │    Citadel   │  │    Galley    │     │
│  │   (Config)   │  │  (Security)  │  │ (Validation) │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└──────────────────────┬────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────┐
│                    Data Plane (Envoy Proxies)              │
│  ┌─────────────────────────┐  ┌─────────────────────────┐  │
│  │      Service A          │  │      Service B          │  │
│  │  ┌─────────────────┐   │  │  ┌─────────────────┐     │  │
│  │  │   Application   │   │  │  │   Application   │     │  │
│  │  └─────────────────┘   │  │  └─────────────────┘     │  │
│  │  ┌─────────────────┐   │  │  ┌─────────────────┐     │  │
│  │  │  Envoy Sidecar  │◄──┼──┼─►│  Envoy Sidecar  │     │  │
│  │  └─────────────────┘   │  │  └─────────────────┘     │  │
│  └─────────────────────────┘  └─────────────────────────┘  │
│         mTLS, Tracing, Metrics, Rate Limiting              │
└────────────────────────────────────────────────────────────┘
```

## Architecture de développement

### Environnement de développement local

```
┌─────────────────────────────────────────────────────────────┐
│              Developer Workstation                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  IDE/Editor                    Tools                        │
│  ┌──────────────┐             ┌──────────────┐              │
│  │   VS Code    │             │   kubectl    │              │
│  │   + K8s ext  │             │   Helm       │              │
│  └──────┬───────┘             │   Skaffold   │              │
│         │                     │   Telepresence│             │
│         │                     └──────┬───────┘              │
│         │                            │                      │
│         └────────────┬───────────────┘                      │
│                      ▼                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           Local MicroK8s Cluster                    │    │
│  │  • Hot reload avec Skaffold                         │    │
│  │  • Port forwarding pour debug                       │    │
│  │  • Registry local pour images                       │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

## Exemples de configurations types

### Lab minimal (2 CPU, 4GB RAM)

```yaml
Configuration:
  OS: Ubuntu Server 22.04 (sans GUI)
  MicroK8s: Single node
  Addons:
    - dns
    - dashboard
    - storage

Capacité:
  - 3-5 applications légères
  - Pas de monitoring lourd
  - Développement basique

Usage type:
  - Apprentissage Kubernetes
  - Tests simples
  - Petits projets personnels
```

### Lab standard (4 CPU, 8GB RAM)

```yaml
Configuration:
  OS: Ubuntu Server 22.04
  MicroK8s: Single node
  Addons:
    - dns
    - dashboard
    - ingress
    - cert-manager
    - registry
    - prometheus (light)

Capacité:
  - 10-15 applications
  - Monitoring basique
  - CI/CD simple

Usage type:
  - Développement complet
  - Tests d'intégration
  - Projets personnels avancés
  - Préparation certifications
```

### Lab avancé (8+ CPU, 16GB+ RAM)

```yaml
Configuration:
  OS: Ubuntu Server 22.04
  MicroK8s: Multi-node possible
  Addons:
    - Tous les addons essentiels
    - prometheus + grafana
    - istio
    - metallb
    - gpu (si disponible)

Capacité:
  - 20+ applications
  - Stack complète monitoring
  - Multiple environnements
  - Service mesh

Usage type:
  - Simulation production
  - Architecture microservices
  - Tests de charge
  - R&D avancée
```

## Best practices architecturales

### 1. Organisation des namespaces

```yaml
Stratégie recommandée:
├── kube-system : Système (ne pas toucher)
├── default : Éviter d'utiliser
├── monitoring : Stack Prometheus/Grafana
├── ingress-nginx : Ingress controller
├── dev : Environnement développement
├── staging : Environnement pre-prod
├── production : Environnement production
└── tools : CI/CD, registry, etc.
```

### 2. Naming conventions

```yaml
Resources naming:
├── Deployments : app-name-env
├── Services : svc-app-name
├── ConfigMaps : cm-app-name-config
├── Secrets : secret-app-name-type
├── PVC : pvc-app-name-purpose
└── Ingress : ing-app-name
```

### 3. Labeling strategy

```yaml
Labels standards:
├── app: application-name
├── version: v1.0.0
├── component: frontend|backend|database
├── environment: dev|staging|prod
├── managed-by: helm|kubectl|argocd
└── team: team-name
```

### 4. Resource management

```yaml
Bonnes pratiques:
├── Toujours définir requests/limits
├── Utiliser des namespaces quotas
├── Implémenter HPA pour scaling
├── Monitoring des ressources
└── Regular cleanup des ressources inutilisées
```

## Checklist d'architecture

### Avant de commencer

- [ ] Objectifs du lab définis
- [ ] Ressources hardware vérifiées
- [ ] OS choisi et installé
- [ ] Réseau planifié (IP, DNS)
- [ ] Stratégie de backup définie

### Installation de base

- [ ] MicroK8s installé
- [ ] Addons essentiels activés
- [ ] Kubectl configuré
- [ ] Dashboard accessible
- [ ] DNS fonctionnel

### Sécurité

- [ ] RBAC configuré
- [ ] Secrets management en place
- [ ] Network policies définies
- [ ] Firewall configuré
- [ ] Certificats SSL/TLS

### Production readiness

- [ ] Monitoring opérationnel
- [ ] Alerting configuré
- [ ] Backup automatisé
- [ ] Documentation à jour
- [ ] Procédures de recovery testées

## Conclusion

L'architecture de votre lab MicroK8s est un système vivant qui évoluera avec vos besoins et vos compétences. Commencez simple avec une architecture minimale, puis ajoutez progressivement des composants selon vos besoins.

### Points clés à retenir

1. **Modularité** : MicroK8s permet d'activer/désactiver des fonctionnalités à la demande
2. **Évolutivité** : L'architecture peut grandir du single-node au multi-node
3. **Flexibilité** : Adaptable à différents cas d'usage et contraintes
4. **Standardisation** : Suit les patterns Kubernetes standards
5. **Apprentissage** : Chaque composant ajouté est une opportunité d'apprendre

### Prochaines étapes

Avec cette compréhension de l'architecture générale, vous êtes prêt à :
- Installer MicroK8s sur votre système
- Configurer les composants de base
- Déployer vos premières applications
- Explorer les fonctionnalités avancées

L'architecture présentée ici servira de référence tout au long de votre parcours avec MicroK8s, vous permettant de comprendre comment chaque nouvelle fonctionnalité s'intègre dans l'ensemble.

---

*La section suivante vous guidera dans l'installation concrète de MicroK8s sur votre système d'exploitation.*

⏭️
