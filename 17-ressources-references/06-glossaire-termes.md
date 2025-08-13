🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.6 Glossaire des termes

## Introduction au glossaire

Ce glossaire rassemble tous les termes techniques que vous rencontrerez dans votre parcours avec MicroK8s et Kubernetes. Les définitions sont volontairement accessibles aux débutants, avec des exemples concrets et des analogies quand c'est pertinent. Les termes sont classés par ordre alphabétique pour faciliter la recherche.

---

## A

### Addon
**Définition** : Extension ou module complémentaire qui ajoute des fonctionnalités à MicroK8s.
**Exemple** : L'addon `dns` active CoreDNS pour la résolution de noms dans le cluster.
**Commande** : `microk8s enable dashboard`
**Analogie** : Comme les extensions d'un navigateur web qui ajoutent des fonctionnalités.

### Admission Controller
**Définition** : Composant qui intercepte les requêtes à l'API Kubernetes avant la persistance des objets, pour valider ou modifier ces requêtes.
**Utilité** : Enforcement de policies, injection de sidecars, validation de sécurité.
**Types** : Validating (valide) et Mutating (modifie).

### Affinity
**Définition** : Règles qui influencent où les pods sont programmés en fonction d'autres pods ou des caractéristiques des nodes.
**Types** :
- **Node Affinity** : Préférence pour certains nodes
- **Pod Affinity** : Rapprocher des pods
- **Pod Anti-Affinity** : Éloigner des pods

### Annotation
**Définition** : Métadonnées non-identifiantes attachées aux objets Kubernetes, utilisées pour stocker des informations arbitraires.
**Format** : Paires clé-valeur dans metadata.annotations
**Différence avec Labels** : Les annotations ne sont pas utilisées pour la sélection.
**Exemple** : `kubernetes.io/ingress.class: "nginx"`

### API Server
**Définition** : Composant central de Kubernetes qui expose l'API REST, traite les requêtes et met à jour l'état dans etcd.
**Rôle** : Point d'entrée unique pour toutes les opérations sur le cluster.
**Port par défaut** : 6443

### Application (App)
**Définition** : Ensemble de conteneurs et ressources Kubernetes travaillant ensemble pour fournir un service.
**Composition type** : Deployment + Service + ConfigMap + Ingress.

## B

### Backend
**Définition** : Service ou ensemble de pods qui traitent les requêtes, généralement derrière un service ou un ingress.
**Contexte** : Souvent opposé au frontend (interface utilisateur).

### Backing Service
**Définition** : Service externe dont dépend une application (base de données, cache, message queue).
**Exemples** : PostgreSQL, Redis, RabbitMQ.

### Blue-Green Deployment
**Définition** : Stratégie de déploiement où deux environnements identiques (blue et green) sont alternés.
**Avantage** : Rollback instantané en cas de problème.
**Processus** : Déployer sur green, tester, basculer le trafic, garder blue en backup.

### Bootstrap
**Définition** : Processus d'initialisation et de configuration initiale d'un cluster ou d'une application.
**Dans MicroK8s** : Installation et configuration des composants de base.

## C

### Canary Deployment
**Définition** : Stratégie de déploiement progressif où une nouvelle version reçoit un petit pourcentage du trafic.
**Nom** : Référence aux canaris dans les mines de charbon (détection précoce de problèmes).
**Progression type** : 5% → 25% → 50% → 100%.

### CertManager
**Définition** : Contrôleur Kubernetes qui automatise la gestion et le renouvellement des certificats TLS.
**Utilité** : Certificats Let's Encrypt automatiques pour HTTPS.
**Addon MicroK8s** : `microk8s enable cert-manager`

### CI/CD
**Définition** : Continuous Integration / Continuous Deployment - Pratiques d'automatisation du développement.
**CI** : Intégration continue du code
**CD** : Déploiement continu vers les environnements
**Outils** : Jenkins, GitLab CI, GitHub Actions, ArgoCD.

### Cluster
**Définition** : Ensemble de machines (nodes) travaillant ensemble pour exécuter des applications conteneurisées.
**Composition minimale** : Au moins un control plane et un worker node.
**Dans MicroK8s** : Peut être single-node pour le développement.

### ClusterIP
**Définition** : Type de service Kubernetes accessible uniquement depuis l'intérieur du cluster.
**IP** : Virtuelle et stable, assignée automatiquement.
**Cas d'usage** : Communication interne entre microservices.

### ConfigMap
**Définition** : Objet Kubernetes pour stocker des données de configuration non-sensibles sous forme de paires clé-valeur.
**Utilisation** : Variables d'environnement, fichiers de configuration, arguments de commande.
**Limite** : 1MB de données maximum.
**Exemple** : Configuration nginx, properties files.

### Container
**Définition** : Unité d'exécution légère et portable contenant une application et ses dépendances.
**Runtime** : Docker, containerd, CRI-O.
**Dans un Pod** : Un pod peut contenir plusieurs containers.

### Container Registry
**Définition** : Service de stockage et distribution d'images de conteneurs.
**Exemples** : Docker Hub, Google Container Registry, Amazon ECR.
**Local dans MicroK8s** : `microk8s enable registry`

### Control Plane
**Définition** : Ensemble des composants qui prennent les décisions globales sur le cluster.
**Composants** : API Server, Scheduler, Controller Manager, etcd.
**Anciennement** : Appelé "Master".

### Controller
**Définition** : Boucle de contrôle qui surveille l'état du cluster et effectue des changements pour atteindre l'état désiré.
**Principe** : Observe → Analyse → Agit.
**Exemples** : ReplicaSet Controller, Deployment Controller, Job Controller.

### CoreDNS
**Définition** : Serveur DNS utilisé par Kubernetes pour la résolution de noms de services.
**Rôle** : Permet d'accéder aux services par leur nom plutôt que par IP.
**Dans MicroK8s** : Activé avec `microk8s enable dns`

### CRD (Custom Resource Definition)
**Définition** : Extension de l'API Kubernetes permettant de créer ses propres types de ressources.
**Utilité** : Étendre Kubernetes avec des objets métier spécifiques.
**Exemple** : Prometheus ServiceMonitor, Cert-Manager Certificate.

### CRI (Container Runtime Interface)
**Définition** : Interface standard entre Kubernetes et les runtimes de conteneurs.
**Implémentations** : containerd, CRI-O, Docker (via dockershim, déprécié).

### CronJob
**Définition** : Ressource Kubernetes qui crée des Jobs selon un planning défini (syntaxe cron).
**Format cron** : `* * * * *` (minute heure jour mois jour-semaine).
**Cas d'usage** : Backups, rapports, nettoyage.

### CSI (Container Storage Interface)
**Définition** : Standard pour exposer des systèmes de stockage aux orchestrateurs de conteneurs.
**Avantage** : Plugins de stockage indépendants de Kubernetes.

## D

### DaemonSet
**Définition** : Contrôleur qui garantit qu'un pod tourne sur tous (ou certains) nodes du cluster.
**Cas d'usage** : Agents de monitoring, collecteurs de logs, network plugins.
**Particularité** : Automatiquement déployé sur les nouveaux nodes.

### Dashboard
**Définition** : Interface web pour visualiser et gérer les ressources Kubernetes.
**Dans MicroK8s** : `microk8s enable dashboard`
**Accès** : Via proxy ou port-forward.

### Deployment
**Définition** : Ressource qui gère le déploiement déclaratif de pods via ReplicaSets.
**Fonctionnalités** : Rolling updates, rollbacks, scaling.
**Usage** : La méthode standard pour déployer des applications stateless.

### DNS
**Définition** : Domain Name System - Service de résolution de noms en adresses IP.
**Dans Kubernetes** : CoreDNS résout les noms de services et pods.
**Format** : `service.namespace.svc.cluster.local`

### Docker
**Définition** : Plateforme de conteneurisation historique, maintenant une option parmi d'autres.
**Dans Kubernetes** : Runtime déprécié, remplacé par containerd.
**Image format** : OCI (Open Container Initiative) standard.

### Downward API
**Définition** : Mécanisme permettant aux pods d'accéder à leurs propres métadonnées.
**Informations disponibles** : Nom du pod, namespace, labels, annotations, ressources.
**Méthodes** : Variables d'environnement ou volumes.

## E

### Endpoint
**Définition** : Adresse IP et port d'un pod faisant partie d'un service.
**Gestion** : Automatique par Kubernetes basée sur les selectors.
**Objet** : `kubectl get endpoints`

### Environment Variable
**Définition** : Variable passée à un conteneur pour configurer son comportement.
**Sources** : Directe, ConfigMap, Secret, Downward API.
**Format** : `env:` dans la spec du container.

### Ephemeral Container
**Définition** : Container temporaire ajouté à un pod existant pour le debugging.
**Utilité** : Dépanner des containers minimalistes sans outils.
**Commande** : `kubectl debug`

### etcd
**Définition** : Base de données clé-valeur distribuée stockant toute la configuration du cluster.
**Criticité** : Perte d'etcd = perte du cluster.
**Backup** : Essentiel pour la disaster recovery.
**Port** : 2379 (client), 2380 (peer).

### Event
**Définition** : Enregistrement d'un événement survenu dans le cluster.
**Durée de vie** : Limité (1 heure par défaut).
**Consultation** : `kubectl get events`
**Types** : Normal, Warning.

### Eviction
**Définition** : Processus de suppression forcée d'un pod d'un node.
**Causes** : Manque de ressources, node drain, pod disruption budget.

## F

### Failover
**Définition** : Basculement automatique vers un système de secours en cas de panne.
**Dans Kubernetes** : Reprogrammation automatique des pods sur des nodes sains.

### Federation
**Définition** : Gestion de plusieurs clusters Kubernetes comme une seule entité.
**Utilité** : Multi-région, haute disponibilité, disaster recovery.

### Finalizer
**Définition** : Clé qui empêche la suppression d'une ressource jusqu'à ce que certaines conditions soient remplies.
**Usage** : Nettoyage de ressources externes avant suppression.

### Flannel
**Définition** : Plugin réseau (CNI) simple pour Kubernetes.
**Alternative dans MicroK8s** : Calico (par défaut).

## G

### GitOps
**Définition** : Pratique où Git est la source de vérité pour l'infrastructure et les déploiements.
**Principe** : Déclarer l'état désiré dans Git, automatiser la convergence.
**Outils** : ArgoCD, Flux, Jenkins X.

### Grafana
**Définition** : Plateforme de visualisation et d'analyse de métriques.
**Dans MicroK8s** : Souvent utilisé avec Prometheus.
**Usage** : Dashboards de monitoring du cluster et des applications.

### gRPC
**Définition** : Framework RPC (Remote Procedure Call) haute performance utilisé dans Kubernetes.
**Usage** : Communication entre composants Kubernetes.

## H

### HA (High Availability)
**Définition** : Haute disponibilité - Configuration éliminant les points de défaillance uniques.
**Dans Kubernetes** : Multiple control plane nodes, etcd clustering.
**Objectif** : 99.99% uptime ou plus.

### Headless Service
**Définition** : Service sans IP de cluster (ClusterIP: None) qui retourne directement les IPs des pods.
**Usage** : StatefulSets, découverte de service directe.

### Health Check
**Définition** : Vérification de l'état de santé d'un container.
**Types** : Liveness probe, Readiness probe, Startup probe.

### Helm
**Définition** : Gestionnaire de paquets pour Kubernetes ("apt/yum pour Kubernetes").
**Concepts** : Charts (packages), Releases (instances), Values (configuration).
**Dans MicroK8s** : `microk8s enable helm3`

### HorizontalPodAutoscaler (HPA)
**Définition** : Ressource qui ajuste automatiquement le nombre de pods selon la charge.
**Métriques** : CPU, mémoire, custom metrics.
**Limites** : Min et max replicas définis.

### HostNetwork
**Définition** : Mode où le pod utilise directement le réseau du node hôte.
**Implication** : Pas d'isolation réseau, ports directement exposés.
**Usage** : Agents système, network plugins.

### HostPath
**Définition** : Type de volume qui monte un fichier ou répertoire du node dans le pod.
**Danger** : Brise la portabilité, risques de sécurité.
**Usage** : Development, DaemonSets système.

## I

### Image
**Définition** : Package read-only contenant le système de fichiers et les métadonnées d'un container.
**Format** : Layers empilés, standard OCI.
**Stockage** : Container registry.
**Exemple** : `nginx:1.21`, `myapp:v2.0.1`

### ImagePullPolicy
**Définition** : Politique déterminant quand Kubernetes doit télécharger une image.
**Options** :
- **Always** : Toujours pull
- **IfNotPresent** : Pull si absent localement
- **Never** : Jamais pull

### Ingress
**Définition** : Ressource qui gère l'accès HTTP/HTTPS externe vers les services du cluster.
**Fonctionnalités** : Routing par nom d'hôte/chemin, SSL/TLS, load balancing.
**Controller** : NGINX, Traefik, HAProxy.
**Dans MicroK8s** : `microk8s enable ingress`

### Init Container
**Définition** : Container qui s'exécute avant les containers principaux d'un pod.
**Usage** : Préparation de l'environnement, attente de dépendances, initialisation.
**Particularité** : Doit se terminer avec succès avant le démarrage des containers principaux.

### IPVS
**Définition** : IP Virtual Server - Mode de proxy kube-proxy plus performant qu'iptables.
**Avantage** : Meilleure performance avec beaucoup de services.

## J

### Job
**Définition** : Ressource qui exécute un ou plusieurs pods jusqu'à completion réussie.
**Usage** : Batch processing, migrations, calculs one-shot.
**Paramètres** : Parallelism, completions, backoffLimit.

### JSON
**Définition** : JavaScript Object Notation - Format de données structurées.
**Dans Kubernetes** : Alternative à YAML pour les manifests (moins courant).

### JSONPath
**Définition** : Langage de requête pour extraire des données JSON.
**Usage** : `kubectl get pods -o jsonpath='{.items[*].metadata.name}'`

## K

### K8s
**Définition** : Abréviation courante de Kubernetes (K + 8 lettres + s).
**Prononciation** : "K-eight-s" ou "Kates".

### Kube-proxy
**Définition** : Agent réseau sur chaque node qui maintient les règles réseau pour les services.
**Modes** : iptables (défaut), IPVS, userspace.
**Rôle** : Load balancing au niveau L4.

### Kubectl
**Définition** : Outil en ligne de commande pour interagir avec les clusters Kubernetes.
**Prononciation** : "Kube-control", "Kube-C-T-L", ou "Kube-cuddle".
**Dans MicroK8s** : `microk8s kubectl` ou alias `kubectl`.

### Kubelet
**Définition** : Agent principal sur chaque node qui gère les pods et containers.
**Responsabilités** : Démarrer/arrêter les containers, health checks, rapporter le statut.
**Communication** : Avec l'API server et le container runtime.

### Kubeconfig
**Définition** : Fichier de configuration contenant les informations de connexion aux clusters.
**Localisation** : `~/.kube/config` par défaut.
**Contenu** : Clusters, users, contexts.

### Kubernetes
**Définition** : Système open-source d'orchestration de containers.
**Origine** : Google, basé sur Borg.
**Signification** : Du grec "Κυβερνήτης" (gouvernail, pilote).
**Projet** : Géré par la CNCF depuis 2015.

### Kustomize
**Définition** : Outil de gestion de configuration Kubernetes natif sans templates.
**Principe** : Patches et overlays sur des bases YAML.
**Intégration** : Native dans kubectl depuis v1.14.

## L

### Label
**Définition** : Paire clé-valeur attachée aux objets pour l'identification et la sélection.
**Usage** : Grouper, filtrer, sélectionner des ressources.
**Format** : `app=nginx`, `environment=production`
**Sélecteur** : `kubectl get pods -l app=nginx`

### Liveness Probe
**Définition** : Vérification périodique pour déterminer si un container doit être redémarré.
**Types** : HTTP GET, TCP socket, Exec command.
**Action en cas d'échec** : Restart du container.

### LoadBalancer
**Définition** : Type de service qui expose l'application via un load balancer externe.
**Cloud** : Provisionne automatiquement (AWS ELB, GCP LB).
**On-premise** : Nécessite MetalLB ou similaire.
**Dans MicroK8s** : `microk8s enable metallb`

### Logging
**Définition** : Collecte et gestion des logs des applications et du système.
**Niveaux** : stdout/stderr des containers.
**Stack** : ELK (Elasticsearch, Logstash, Kibana) ou EFK (avec Fluentd).

## M

### Manifest
**Définition** : Fichier YAML ou JSON décrivant les ressources Kubernetes à créer.
**Structure** : apiVersion, kind, metadata, spec.
**Application** : `kubectl apply -f manifest.yaml`

### Master
**Définition** : Ancien terme pour Control Plane (déprécié).
**Raison du changement** : Terminologie plus inclusive.

### MetalLB
**Définition** : Load balancer pour environnements bare metal.
**Modes** : Layer 2 (ARP/NDP) ou BGP.
**Dans MicroK8s** : `microk8s enable metallb`

### Metrics Server
**Définition** : Composant collectant les métriques de ressources (CPU, RAM) des nodes et pods.
**Usage** : HPA, VPA, `kubectl top`.
**Dans MicroK8s** : Inclus dans certains addons.

### MicroK8s
**Définition** : Distribution légère de Kubernetes par Canonical, optimisée pour le développement et l'edge.
**Particularités** : Installation simple, faible empreinte, addons intégrés.
**Commande** : `microk8s` au lieu de systèmes séparés.

### Monitoring
**Définition** : Surveillance de l'état et des performances du cluster et des applications.
**Stack populaire** : Prometheus + Grafana.
**Métriques** : CPU, mémoire, réseau, disque, latence, erreurs.

### Multi-tenancy
**Définition** : Capacité d'un cluster à servir plusieurs utilisateurs ou équipes isolées.
**Isolation** : Namespaces, RBAC, Network Policies, Resource Quotas.

## N

### Namespace
**Définition** : Mécanisme de séparation logique des ressources dans un cluster.
**Analogie** : Comme des dossiers dans un système de fichiers.
**Défaut** : `default`, `kube-system`, `kube-public`, `kube-node-lease`.
**Usage** : Isolation par environnement, équipe, ou projet.

### Network Policy
**Définition** : Règles contrôlant le trafic réseau entre pods et/ou avec l'extérieur.
**Types** : Ingress (entrant), Egress (sortant).
**Principe** : Deny par défaut une fois activé.
**CNI requis** : Calico, Cilium, Weave.

### Node
**Définition** : Machine (physique ou virtuelle) faisant partie du cluster Kubernetes.
**Types** : Control plane nodes, Worker nodes.
**Composants** : Kubelet, kube-proxy, container runtime.
**Commande** : `kubectl get nodes`

### NodePort
**Définition** : Type de service exposant l'application sur un port statique de chaque node.
**Range** : 30000-32767 par défaut.
**Accès** : `<NodeIP>:<NodePort>`
**Usage** : Testing, environnements sans load balancer.

### Node Selector
**Définition** : Mécanisme simple pour contraindre les pods à des nodes avec certains labels.
**Exemple** : `nodeSelector: disktype: ssd`
**Alternative plus flexible** : Node Affinity.

## O

### Observability
**Définition** : Capacité à comprendre l'état interne d'un système depuis ses outputs.
**Piliers** : Metrics (métriques), Logs (journaux), Traces (traces distribuées).
**Outils** : Prometheus, Grafana, Jaeger, ELK stack.

### Operator
**Définition** : Pattern et méthode pour étendre Kubernetes avec des connaissances opérationnelles spécifiques.
**Principe** : Controller + CRD + logique métier.
**Exemples** : Prometheus Operator, MySQL Operator.
**Framework** : Operator SDK, Kubebuilder.

### OCI (Open Container Initiative)
**Définition** : Standards pour les formats de containers et runtimes.
**Standards** : Image-spec, runtime-spec, distribution-spec.

### Orchestration
**Définition** : Gestion automatisée du cycle de vie des containers.
**Aspects** : Déploiement, scaling, networking, load balancing.
**Solution** : Kubernetes est un orchestrateur.

## P

### PVC (PersistentVolumeClaim)
**Définition** : Demande de stockage par un utilisateur, comme un "bon de commande" pour du stockage.
**Binding** : Automatique avec un PV correspondant.
**Modes d'accès** : ReadWriteOnce, ReadOnlyMany, ReadWriteMany.

### PV (PersistentVolume)
**Définition** : Ressource de stockage dans le cluster, provisionné par un admin ou dynamiquement.
**Types** : NFS, iSCSI, cloud disks, local.
**Cycle de vie** : Available → Bound → Released → Reclaimed/Deleted.

### Pod
**Définition** : Plus petite unité déployable dans Kubernetes, contenant un ou plusieurs containers.
**Analogie** : Comme une "machine logique" pour vos containers.
**Partage** : Network namespace, storage, IPC.
**IP** : Unique par pod, éphémère.

### Pod Disruption Budget (PDB)
**Définition** : Limite le nombre de pods d'une application qui peuvent être indisponibles simultanément.
**Usage** : Protection pendant les maintenances, upgrades.
**Paramètres** : minAvailable ou maxUnavailable.

### Port Forward
**Définition** : Redirection d'un port local vers un pod pour accès direct.
**Commande** : `kubectl port-forward pod/mypod 8080:80`
**Usage** : Debugging, accès temporaire.

### Probe
**Définition** : Mécanisme de vérification de l'état d'un container.
**Types** :
- **Liveness** : Le container est-il vivant?
- **Readiness** : Peut-il recevoir du trafic?
- **Startup** : A-t-il fini de démarrer?

### Prometheus
**Définition** : Système de monitoring et d'alerting open-source, standard de facto pour Kubernetes.
**Caractéristiques** : Time-series database, PromQL, pull-based.
**Dans MicroK8s** : Via addon ou Helm chart.

### Proxy
**Définition** : Intermédiaire qui transmet les requêtes.
**Dans Kubernetes** : kube-proxy (services), kubectl proxy (API access), ingress (HTTP).

## Q

### QoS (Quality of Service)
**Définition** : Classification des pods selon leurs demandes de ressources.
**Classes** :
- **Guaranteed** : Requests = Limits
- **Burstable** : Requests < Limits
- **BestEffort** : Pas de requests/limits

### Quota
**Voir** : Resource Quota

## R

### RBAC (Role-Based Access Control)
**Définition** : Système d'autorisation basé sur les rôles pour contrôler l'accès aux ressources.
**Composants** : Roles/ClusterRoles, RoleBindings/ClusterRoleBindings.
**Principe** : Qui (Subject) peut faire Quoi (Verbs) sur Quoi (Resources).

### Readiness Probe
**Définition** : Vérification pour déterminer si un pod peut recevoir du trafic.
**Action en cas d'échec** : Retrait des endpoints du service (pas de restart).
**Usage** : Warm-up time, dépendances externes.

### Registry
**Définition** : Service de stockage et distribution d'images de containers.
**Types** : Public (Docker Hub), Privé (Harbor), Cloud (ECR, GCR).
**Dans MicroK8s** : `microk8s enable registry` pour un registry local.

### ReplicaSet
**Définition** : Contrôleur garantissant qu'un nombre spécifié de pods identiques sont en cours d'exécution.
**Usage** : Rarement créé directement, géré par Deployment.
**Fonction** : Maintient le nombre de replicas désiré.

### Resource
**Définition** : Objet de l'API Kubernetes (Pod, Service, Deployment, etc.).
**CRUD** : Create, Read, Update, Delete via l'API.
**Types** : Namespaced ou Cluster-scoped.

### Resource Limits
**Définition** : Maximum de CPU/mémoire qu'un container peut utiliser.
**Enforcement** : Container tué (OOMKilled) si dépassement mémoire.
**Format** : `limits: memory: "512Mi", cpu: "500m"`

### Resource Quota
**Définition** : Limites sur les ressources totales consommables dans un namespace.
**Types** : Compute (CPU/RAM), Storage, Objects count.
**Usage** : Multi-tenancy, prévention des abus.

### Resource Requests
**Définition** : Minimum de CPU/mémoire garanti pour un container.
**Usage** : Scheduling decisions, QoS class.
**Format** : `requests: memory: "256Mi", cpu: "250m"`

### Rolling Update
**Définition** : Stratégie de mise à jour progressive des pods sans interruption de service.
**Paramètres** : maxSurge, maxUnavailable.
**Rollback** : `kubectl rollout undo`

### Rollback
**Définition** : Retour à une version précédente d'un deployment.
**Commande** : `kubectl rollout undo deployment/myapp`
**Historique** : `kubectl rollout history`

## S

### Scheduler
**Définition** : Composant qui assigne les pods aux nodes selon les contraintes et ressources.
**Critères** : Resources, affinity, taints/tolerations, node selectors.
**Custom** : Possible d'écrire son propre scheduler.

### Secret
**Définition** : Objet pour stocker des données sensibles (passwords, tokens, clés).
**Encodage** : Base64 (pas du chiffrement!).
**Types** : Opaque, docker-registry, TLS, service-account-token.
**Limite** : 1MB maximum.

### Security Context
**Définition** : Paramètres de sécurité pour un pod ou container.
**Options** : User/Group ID, capabilities, read-only filesystem, SELinux.
**Exemple** : `runAsNonRoot: true`, `runAsUser: 1000`

### Selector
**Définition** : Mécanisme pour identifier un ensemble d'objets via leurs labels.
**Types** : Equality-based, Set-based.
**Usage** : Services, ReplicaSets, Deployments, Network Policies.

### Service
**Définition** : Abstraction définissant un ensemble logique de pods et une politique d'accès.
**Types** : ClusterIP, NodePort, LoadBalancer, ExternalName.
**DNS** : `service-name.namespace.svc.cluster.local`
**Rôle** : Load balancing, service discovery.

### Service Account
**Définition** : Identité pour les processus qui s'exécutent dans un pod.
**Usage** : Authentification auprès de l'API server.
**Token** : Monté automatiquement dans `/var/run/secrets/kubernetes.io/serviceaccount/`
**Par défaut** : Chaque namespace a un SA `default`.

### Service Mesh
**Définition** : Infrastructure dédiée pour gérer la communication service-à-service.
**Fonctionnalités** : Traffic management, security, observability.
**Exemples** : Istio, Linkerd, Consul Connect.
**Pattern** : Sidecar proxy (Envoy).

### Sidecar
**Définition** : Container auxiliaire dans un pod qui étend les fonctionnalités du container principal.
**Exemples** : Proxy (Envoy), log collector, configuration reloader.
**Communication** : Via localhost ou volumes partagés.

### StatefulSet
**Définition** : Contrôleur pour applications stateful nécessitant identité stable et stockage persistant.
**Garanties** : Ordre de déploiement, noms stables, stockage persistant.
**Usage** : Bases de données, systèmes distribués (Kafka, Elasticsearch).
**Différence avec Deployment** : Identité fixe des pods.

### Storage Class
**Définition** : Abstraction définissant les types de stockage disponibles et leurs paramètres.
**Provisioning** : Statique ou dynamique.
**Paramètres** : Type de disque, IOPS, zone, réplication.
**Default** : Une classe peut être marquée par défaut.

## T

### Taint
**Définition** : Marque sur un node qui repousse les pods sauf ceux avec une toleration correspondante.
**Effects** : NoSchedule, PreferNoSchedule, NoExecute.
**Usage** : Nodes dédiés, GPU nodes, maintenance.
**Commande** : `kubectl taint nodes node1 key=value:NoSchedule`

### Toleration
**Définition** : Permet à un pod d'être schedulé sur un node avec un taint correspondant.
**Format** : Défini dans la spec du pod.
**Exemple** : Tolérer les nodes GPU, master nodes.

### Topology Spread Constraints
**Définition** : Règles pour distribuer les pods uniformément à travers les zones de failure.
**Usage** : Haute disponibilité, distribution géographique.
**Paramètres** : maxSkew, topologyKey, whenUnsatisfiable.

## U

### UID
**Définition** : Identifiant unique universel pour chaque objet Kubernetes.
**Format** : UUID (ex: 550e8400-e29b-41d4-a716-446655440000).
**Immutable** : Ne change jamais pendant la vie de l'objet.

### Upgrade
**Définition** : Mise à jour de version de Kubernetes ou d'une application.
**Stratégies** : Rolling update, blue-green, canary.
**MicroK8s** : `snap refresh microk8s --channel=1.29/stable`

### Upstream
**Définition** : Source originale d'un projet ou code.
**Contexte** : Kubernetes upstream = projet Kubernetes officiel.
**Pour MicroK8s** : Suit l'upstream Kubernetes avec des modifications minimales.

## V

### Volume
**Définition** : Répertoire accessible aux containers dans un pod.
**Types** : emptyDir, hostPath, configMap, secret, persistentVolumeClaim.
**Cycle de vie** : Lié au pod (sauf PVC).
**Montage** : Dans `volumeMounts` du container.

### VolumeMount
**Définition** : Point de montage d'un volume dans un container.
**Paramètres** : name, mountPath, readOnly, subPath.
**Exemple** : `mountPath: /etc/config`

### VPA (Vertical Pod Autoscaler)
**Définition** : Ajuste automatiquement les requests/limits de CPU et mémoire des pods.
**Modes** : Off, Initial, Auto.
**Différence avec HPA** : Scale vertical (ressources) vs horizontal (replicas).

## W

### Watch
**Définition** : Mécanisme pour observer les changements sur les ressources en temps réel.
**API** : Watch parameter sur les requêtes GET.
**kubectl** : `kubectl get pods --watch`
**Usage** : Controllers, operators, monitoring.

### Webhook
**Définition** : HTTP callback pour étendre Kubernetes.
**Types** : Admission webhooks (validating/mutating), CRD conversion webhooks.
**Usage** : Validation custom, injection de sidecars, policy enforcement.

### Worker Node
**Définition** : Node qui exécute les charges de travail (pods).
**Composants** : Kubelet, kube-proxy, container runtime.
**Anciennement** : Appelé "Minion" (terme obsolète).

### Workload
**Définition** : Application ou charge de travail s'exécutant sur Kubernetes.
**Types** : Deployment, StatefulSet, DaemonSet, Job, CronJob.
**Gestion** : Via les workload controllers.

## Y

### YAML
**Définition** : YAML Ain't Markup Language - Format de sérialisation de données lisible par l'humain.
**Dans Kubernetes** : Format principal pour les manifests.
**Structure** : Indentation significative (2 espaces standard).
**Alternative** : JSON (moins lisible mais valide).

## Z

### Zone
**Définition** : Domaine de failure dans une région (datacenter, rack, etc.).
**Label** : `topology.kubernetes.io/zone`
**Usage** : Distribution des pods pour haute disponibilité.
**Exemples** : us-east-1a, europe-west1-b.

### Zero-downtime Deployment
**Définition** : Déploiement sans interruption de service.
**Techniques** : Rolling updates, blue-green, canary.
**Requis** : Multiple replicas, readiness probes, PodDisruptionBudget.

### Zombie Process
**Définition** : Processus terminé mais toujours présent dans la table des processus.
**Problème** : Peut s'accumuler dans les containers.
**Solution** : Init system (tini, dumb-init) ou proper signal handling.

---

## Acronymes courants

### API
**Application Programming Interface** : Interface de programmation permettant l'interaction avec Kubernetes.

### CA
**Certificate Authority** : Autorité de certification pour TLS/SSL.

### CDR
**Custom Resource Definition** : Voir CRD.

### CI/CD
**Continuous Integration/Continuous Deployment** : Pratiques DevOps d'automatisation.

### CLI
**Command Line Interface** : Interface en ligne de commande (kubectl).

### CNI
**Container Network Interface** : Standard pour les plugins réseau.

### CNCF
**Cloud Native Computing Foundation** : Organisation qui héberge Kubernetes.

### CPU
**Central Processing Unit** : Ressource de calcul, mesurée en millicores (m).

### CRI
**Container Runtime Interface** : Interface pour les runtimes de containers.

### CRD
**Custom Resource Definition** : Extension de l'API Kubernetes.

### CSI
**Container Storage Interface** : Standard pour les plugins de stockage.

### DNS
**Domain Name System** : Système de résolution de noms.

### EFK
**Elasticsearch, Fluentd, Kibana** : Stack de logging.

### ELK
**Elasticsearch, Logstash, Kibana** : Stack de logging.

### FQDN
**Fully Qualified Domain Name** : Nom de domaine complet.

### GC
**Garbage Collection** : Nettoyage automatique des ressources inutilisées.

### GPU
**Graphics Processing Unit** : Utilisé pour calcul intensif, ML/AI.

### gRPC
**gRPC Remote Procedure Call** : Framework de communication.

### HA
**High Availability** : Haute disponibilité.

### HPA
**Horizontal Pod Autoscaler** : Mise à l'échelle horizontale automatique.

### HTTP/HTTPS
**HyperText Transfer Protocol (Secure)** : Protocole web.

### IOPS
**Input/Output Operations Per Second** : Mesure de performance storage.

### IP
**Internet Protocol** : Protocole d'adressage réseau.

### JSON
**JavaScript Object Notation** : Format de données.

### K8s
**Kubernetes** : Abréviation courante (K + 8 lettres + s).

### LB
**Load Balancer** : Répartiteur de charge.

### MAC
**Mandatory Access Control** : Contrôle d'accès obligatoire (SELinux, AppArmor).

### NAT
**Network Address Translation** : Translation d'adresses réseau.

### NFS
**Network File System** : Système de fichiers réseau.

### OCI
**Open Container Initiative** : Standards pour containers.

### OOM
**Out Of Memory** : Manque de mémoire.

### PDB
**Pod Disruption Budget** : Budget de perturbation des pods.

### PSP
**Pod Security Policy** : Politique de sécurité des pods (déprécié).

### PSS
**Pod Security Standards** : Standards de sécurité des pods (remplace PSP).

### PV
**Persistent Volume** : Volume persistant.

### PVC
**Persistent Volume Claim** : Demande de volume persistant.

### QoS
**Quality of Service** : Qualité de service.

### RAM
**Random Access Memory** : Mémoire vive.

### RBAC
**Role-Based Access Control** : Contrôle d'accès basé sur les rôles.

### REST
**Representational State Transfer** : Style d'architecture API.

### RPC
**Remote Procedure Call** : Appel de procédure à distance.

### SA
**Service Account** : Compte de service.

### SAN
**Storage Area Network** : Réseau de stockage.

### SDK
**Software Development Kit** : Kit de développement.

### SLA
**Service Level Agreement** : Accord de niveau de service.

### SLI
**Service Level Indicator** : Indicateur de niveau de service.

### SLO
**Service Level Objective** : Objectif de niveau de service.

### SPOF
**Single Point Of Failure** : Point de défaillance unique.

### SRE
**Site Reliability Engineering** : Ingénierie de fiabilité.

### SSH
**Secure Shell** : Shell sécurisé.

### SSL/TLS
**Secure Sockets Layer/Transport Layer Security** : Protocoles de sécurité.

### TCP
**Transmission Control Protocol** : Protocole de transmission.

### TTL
**Time To Live** : Durée de vie.

### UDP
**User Datagram Protocol** : Protocole de datagramme.

### UI
**User Interface** : Interface utilisateur.

### UUID
**Universally Unique Identifier** : Identifiant unique universel.

### VM
**Virtual Machine** : Machine virtuelle.

### VPA
**Vertical Pod Autoscaler** : Mise à l'échelle verticale automatique.

### YAML
**YAML Ain't Markup Language** : Format de sérialisation.

---

## Expressions et concepts clés

### "Cattle vs Pets"
**Concept** : Les containers sont du "bétail" (remplaçables) pas des "animaux de compagnie" (uniques).
**Implication** : Design pour la failure, pas d'attachement aux instances.

### "Cloud Native"
**Définition** : Applications conçues pour exploiter les avantages du cloud computing.
**Caractéristiques** : Containerisées, microservices, dynamiques, orchestrées.

### "Day 1 vs Day 2 Operations"
**Day 1** : Installation, configuration initiale.
**Day 2** : Opérations quotidiennes, maintenance, monitoring.

### "Declarative vs Imperative"
**Declarative** : Décrire l'état désiré (YAML manifests).
**Imperative** : Dire quoi faire étape par étape (kubectl commands).
**Kubernetes** : Favorise l'approche déclarative.

### "Desired State"
**Concept** : L'état que vous voulez pour vos ressources.
**Reconciliation** : Kubernetes travaille continuellement pour atteindre cet état.

### "Eventually Consistent"
**Principe** : Le système convergera vers l'état désiré, mais pas instantanément.
**Dans Kubernetes** : Les controllers font converger progressivement.

### "GitOps"
**Définition** : Git comme source unique de vérité pour l'infrastructure.
**Workflow** : Commit → CI/CD → Deploy to Kubernetes.

### "Immutable Infrastructure"
**Principe** : Ne pas modifier les serveurs/containers en production, les remplacer.
**Avantage** : Prévisibilité, reproductibilité.

### "Infrastructure as Code (IaC)"
**Concept** : Gérer l'infrastructure via du code versionné.
**Dans Kubernetes** : Manifests YAML dans Git.

### "Lift and Shift"
**Définition** : Migration d'applications existantes vers Kubernetes sans refactoring.
**Alternative** : Refactoring pour architecture cloud-native.

### "Observability"
**Au-delà du monitoring** : Comprendre le pourquoi, pas juste le quoi.
**Piliers** : Metrics, Logs, Traces.

### "Pet vs Cattle"
**Voir** : "Cattle vs Pets"

### "Reconciliation Loop"
**Mécanisme** : Observe → Compare → Act → Repeat.
**Controllers** : Implémentent ce pattern.

### "Self-healing"
**Capacité** : Kubernetes répare automatiquement les failures.
**Exemples** : Restart de pods crashed, rescheduling sur nodes sains.

### "Shift Left"
**Concept** : Déplacer les tests et la sécurité plus tôt dans le cycle de développement.
**Dans Kubernetes** : Validation des manifests, security scanning en CI.

### "Single Source of Truth"
**Principe** : Une seule source autoritaire pour la configuration.
**GitOps** : Git est cette source pour Kubernetes.

### "Twelve-Factor App"
**Méthodologie** : 12 principes pour applications cloud-native.
**Pertinent pour Kubernetes** : Config via env vars, stateless processes, port binding.

---

## Notes pour les débutants

### Termes souvent confondus

**Pod vs Container** : Un pod contient un ou plusieurs containers.

**Service vs Ingress** : Service pour accès interne, Ingress pour accès externe HTTP/HTTPS.

**Deployment vs StatefulSet** : Deployment pour stateless, StatefulSet pour stateful.

**ConfigMap vs Secret** : ConfigMap pour config non-sensible, Secret pour données sensibles.

**PV vs PVC** : PV est le stockage physique, PVC est la demande de stockage.

**Label vs Annotation** : Labels pour sélection, Annotations pour métadonnées.

**Node vs Pod** : Node est la machine, Pod est l'unité d'exécution.

**Namespace vs Cluster** : Namespace est une division logique dans un cluster.

### Premiers termes à maîtriser

1. **Pod** : Unité de base
2. **Node** : Machine du cluster
3. **Deployment** : Pour déployer des apps
4. **Service** : Pour exposer des apps
5. **Namespace** : Pour organiser
6. **kubectl** : Pour interagir
7. **YAML** : Pour décrire
8. **Container** : Ce qui s'exécute
9. **Image** : Le package de l'app
10. **Label** : Pour identifier

---

Ce glossaire est votre référence rapide. N'hésitez pas à y revenir régulièrement. La maîtrise du vocabulaire est essentielle pour progresser avec Kubernetes et communiquer efficacement avec la communauté.

⏭️
