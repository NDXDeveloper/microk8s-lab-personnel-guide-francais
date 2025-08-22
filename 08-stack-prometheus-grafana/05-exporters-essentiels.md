🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.05 - Exporters essentiels (node, kube-state, cadvisor)

## Introduction aux exporters

### Qu'est-ce qu'un exporter ?

Un **exporter** est un composant qui expose des métriques dans un format que Prometheus peut comprendre. C'est un traducteur qui convertit des métriques d'un système en métriques Prometheus.

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Système    │────▶│   Exporter   │────▶│  Prometheus  │
│   (Linux,    │     │  (Traduit)   │     │   (Scrape)   │
│   K8s, etc)  │     │              │     │              │
└──────────────┘     └──────────────┘     └──────────────┘
                          │
                          ▼
                    HTTP /metrics
                    Format Prometheus
```

### Pourquoi des exporters ?

Tous les systèmes n'exposent pas nativement des métriques au format Prometheus :
- **Systèmes d'exploitation** : Ont leurs propres APIs (proc, sys)
- **Kubernetes** : API complexe avec objets multiples
- **Containers** : Métriques via cgroups
- **Applications tierces** : MySQL, Redis, etc.

Les exporters comblent ce fossé en traduisant ces métriques.

## Vue d'ensemble des trois exporters essentiels

| Exporter | Rôle | Métriques collectées | Déployé comme |
|----------|------|---------------------|---------------|
| **Node Exporter** | Métriques hardware/OS | CPU, RAM, disque, réseau | DaemonSet (1 par node) |
| **kube-state-metrics** | État des objets K8s | Pods, deployments, services | Deployment (1 instance) |
| **cAdvisor** | Métriques containers | CPU/RAM par container | Intégré au kubelet |

### Architecture complète

```
┌────────────────────────────────────────────────────────┐
│                    Kubernetes Node                     │
│                                                        │
│  ┌────────────────────────────────────────────┐        │
│  │            Applications (Pods)             │        │
│  └────────────────────────────────────────────┘        │
│                        │                               │
│                        ▼                               │
│  ┌──────────────────────────────────────────┐          │
│  │         cAdvisor (dans kubelet)          │          │
│  │    Métriques: containers CPU/RAM         │          │
│  └──────────────────────────────────────────┘          │
│                                                        │
│  ┌──────────────────────────────────────────┐          │
│  │           Node Exporter Pod              │          │
│  │    Métriques: système, hardware          │          │
│  └──────────────────────────────────────────┘          │
└────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────┐
│               Cluster Kubernetes (Master)              │
│                                                        │
│  ┌──────────────────────────────────────────┐          │
│  │        kube-state-metrics Pod            │          │
│  │    Métriques: objets Kubernetes          │          │
│  └──────────────────────────────────────────┘          │
└────────────────────────────────────────────────────────┘
                        │
                        ▼
                  ┌──────────┐
                  │Prometheus│
                  └──────────┘
```

## Node Exporter : Les métriques système

### Présentation détaillée

Node Exporter collecte les métriques au niveau du système d'exploitation et du hardware. C'est l'équivalent de commandes comme `top`, `df`, `free`, mais exposé pour Prometheus.

### Installation sur MicroK8s

**Méthode 1 : Via l'addon observability (automatique)**
```bash
# Si vous avez activé l'addon observability
# Node Exporter est déjà installé
kubectl get daemonset -n observability | grep node-exporter
```

**Méthode 2 : Installation manuelle avec DaemonSet**
```yaml
# node-exporter-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9100"
        prometheus.io/path: "/metrics"
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.6.1
        args:
          - --path.procfs=/host/proc
          - --path.sysfs=/host/sys
          - --path.rootfs=/host/root
          - --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/pods/.+)($|/)
          - --collector.filesystem.fs-types-exclude=^(autofs|binfmt_misc|cgroup|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|iso9660|mqueue|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|sysfs|tracefs)$
        ports:
        - containerPort: 9100
          name: metrics
        resources:
          requests:
            cpu: 50m
            memory: 50Mi
          limits:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: root
          mountPath: /host/root
          mountPropagation: HostToContainer
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /
---
apiVersion: v1
kind: Service
metadata:
  name: node-exporter
  namespace: monitoring
  annotations:
    prometheus.io/scrape: "true"
spec:
  selector:
    app: node-exporter
  ports:
  - name: metrics
    port: 9100
    targetPort: 9100
```

```bash
# Appliquer la configuration
kubectl apply -f node-exporter-daemonset.yaml

# Vérifier le déploiement
kubectl get pods -n monitoring -l app=node-exporter
```

### Métriques principales du Node Exporter

#### CPU
```promql
# Utilisation CPU par mode
node_cpu_seconds_total{mode="idle"}     # Temps idle
node_cpu_seconds_total{mode="user"}     # Temps utilisateur
node_cpu_seconds_total{mode="system"}   # Temps système
node_cpu_seconds_total{mode="iowait"}   # Attente I/O

# Nombre de cores CPU
count(count by (cpu) (node_cpu_seconds_total))

# Load average
node_load1   # Sur 1 minute
node_load5   # Sur 5 minutes
node_load15  # Sur 15 minutes
```

#### Mémoire
```promql
# Mémoire totale et disponible
node_memory_MemTotal_bytes
node_memory_MemAvailable_bytes
node_memory_MemFree_bytes

# Buffers et cache
node_memory_Buffers_bytes
node_memory_Cached_bytes

# Swap
node_memory_SwapTotal_bytes
node_memory_SwapFree_bytes
```

#### Disque
```promql
# Espace disque
node_filesystem_size_bytes      # Taille totale
node_filesystem_avail_bytes     # Espace disponible
node_filesystem_files           # Nombre d'inodes total
node_filesystem_files_free      # Inodes libres

# I/O disque
node_disk_read_bytes_total      # Bytes lus
node_disk_written_bytes_total   # Bytes écrits
node_disk_io_time_seconds_total # Temps I/O
```

#### Réseau
```promql
# Trafic réseau
node_network_receive_bytes_total   # Bytes reçus
node_network_transmit_bytes_total  # Bytes transmis
node_network_receive_errs_total    # Erreurs de réception
node_network_transmit_errs_total   # Erreurs de transmission

# État des interfaces
node_network_up                    # Interface up (1) ou down (0)
node_network_info                  # Informations sur l'interface
```

### Configuration avancée du Node Exporter

```yaml
# Arguments supplémentaires pour activer/désactiver des collectors
args:
  # Activer des collectors additionnels
  - --collector.processes       # Informations sur les processus
  - --collector.systemd         # Métriques systemd
  - --collector.tcpstat         # Statistiques TCP

  # Désactiver des collectors non nécessaires
  - --no-collector.arp          # Désactiver ARP
  - --no-collector.ipvs         # Désactiver IPVS

  # Collectors avec configuration
  - --collector.filesystem.mount-points-exclude=^/(dev|proc|sys)($|/)
  - --collector.netclass.ignored-devices=^(veth|docker).*
```

## kube-state-metrics : L'état de Kubernetes

### Présentation détaillée

kube-state-metrics génère des métriques sur l'état des objets Kubernetes en interrogeant l'API Kubernetes. Contrairement à cAdvisor qui mesure l'utilisation des ressources, kube-state-metrics indique l'état désiré vs actuel.

### Installation sur MicroK8s

**Méthode 1 : Installation avec manifests**
```yaml
# kube-state-metrics-deployment.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-state-metrics
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-state-metrics
rules:
- apiGroups: [""]
  resources:
  - configmaps
  - secrets
  - nodes
  - pods
  - services
  - resourcequotas
  - replicationcontrollers
  - limitranges
  - persistentvolumeclaims
  - persistentvolumes
  - namespaces
  - endpoints
  verbs: ["list", "watch"]
- apiGroups: ["apps"]
  resources:
  - deployments
  - daemonsets
  - replicasets
  - statefulsets
  verbs: ["list", "watch"]
- apiGroups: ["batch"]
  resources:
  - cronjobs
  - jobs
  verbs: ["list", "watch"]
- apiGroups: ["autoscaling"]
  resources:
  - horizontalpodautoscalers
  verbs: ["list", "watch"]
- apiGroups: ["networking.k8s.io"]
  resources:
  - ingresses
  verbs: ["list", "watch"]
- apiGroups: ["storage.k8s.io"]
  resources:
  - storageclasses
  verbs: ["list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
  namespace: monitoring
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-state-metrics
  template:
    metadata:
      labels:
        app: kube-state-metrics
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: kube-state-metrics
      containers:
      - name: kube-state-metrics
        image: registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.10.0
        ports:
        - containerPort: 8080
          name: metrics
        - containerPort: 8081
          name: telemetry
        resources:
          requests:
            cpu: 100m
            memory: 150Mi
          limits:
            cpu: 200m
            memory: 250Mi
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 8081
          initialDelaySeconds: 5
          timeoutSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: kube-state-metrics
  namespace: monitoring
  annotations:
    prometheus.io/scrape: "true"
spec:
  selector:
    app: kube-state-metrics
  ports:
  - name: metrics
    port: 8080
    targetPort: 8080
  - name: telemetry
    port: 8081
    targetPort: 8081
```

```bash
# Appliquer la configuration
kubectl apply -f kube-state-metrics-deployment.yaml

# Vérifier le déploiement
kubectl get pods -n monitoring -l app=kube-state-metrics
```

### Métriques principales de kube-state-metrics

#### Pods
```promql
# État des pods
kube_pod_info                          # Informations générales
kube_pod_status_phase                  # Phase (Running, Pending, etc.)
kube_pod_status_ready                  # Pod ready ou non
kube_pod_container_status_restarts_total # Nombre de restarts

# Ressources des pods
kube_pod_container_resource_requests   # Ressources demandées
kube_pod_container_resource_limits     # Limites de ressources

# Conditions des pods
kube_pod_condition                     # Conditions (Ready, Scheduled, etc.)
kube_pod_status_scheduled              # Pod schedulé ou non
```

#### Deployments
```promql
# État des deployments
kube_deployment_status_replicas        # Replicas actuels
kube_deployment_spec_replicas          # Replicas désirés
kube_deployment_status_replicas_available # Replicas disponibles
kube_deployment_status_replicas_unavailable # Replicas indisponibles
kube_deployment_status_replicas_updated # Replicas mis à jour

# Metadata
kube_deployment_metadata_generation    # Génération du deployment
kube_deployment_created                # Timestamp de création
kube_deployment_labels                 # Labels du deployment
```

#### Services et Endpoints
```promql
# Services
kube_service_info                      # Informations du service
kube_service_spec_type                 # Type (ClusterIP, NodePort, etc.)
kube_service_labels                    # Labels du service

# Endpoints
kube_endpoint_address_available        # Adresses disponibles
kube_endpoint_address_not_ready        # Adresses non prêtes
kube_endpoint_info                     # Informations des endpoints
```

#### Nodes
```promql
# État des nodes
kube_node_info                         # Informations du node
kube_node_status_condition            # Conditions (Ready, DiskPressure, etc.)
kube_node_status_allocatable          # Ressources allouables
kube_node_status_capacity             # Capacité totale

# Spécifications
kube_node_spec_unschedulable          # Node unschedulable
kube_node_spec_taint                  # Taints du node
```

#### Namespaces et ResourceQuotas
```promql
# Namespaces
kube_namespace_created                 # Date de création
kube_namespace_status_phase           # Phase du namespace
kube_namespace_labels                  # Labels

# Resource Quotas
kube_resourcequota                    # Quotas définis
kube_resourcequota_created            # Date de création
```

### Requêtes utiles avec kube-state-metrics

```promql
# Pods non Running
kube_pod_status_phase{phase!="Running"} == 1

# Deployments avec replicas manquants
kube_deployment_status_replicas_available <
  kube_deployment_spec_replicas

# Services sans endpoints
kube_service_info unless on(namespace, service)
  kube_endpoint_address_available > 0

# Pods avec beaucoup de restarts
rate(kube_pod_container_status_restarts_total[15m]) > 0

# Nodes non Ready
kube_node_status_condition{condition="Ready", status="true"} == 0
```

## cAdvisor : Les métriques des containers

### Présentation détaillée

cAdvisor (Container Advisor) est intégré directement dans le kubelet et collecte les métriques d'utilisation des ressources des containers. Il utilise les cgroups Linux pour obtenir ces informations.

### Accès aux métriques cAdvisor

cAdvisor est automatiquement disponible via le kubelet :

```yaml
# Configuration Prometheus pour scraper cAdvisor
- job_name: 'kubernetes-cadvisor'
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

  kubernetes_sd_configs:
  - role: node

  relabel_configs:
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
  - target_label: __address__
    replacement: kubernetes.default.svc:443
  - source_labels: [__meta_kubernetes_node_name]
    regex: (.+)
    target_label: __metrics_path__
    replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
```

### Métriques principales de cAdvisor

#### CPU des containers
```promql
# Utilisation CPU cumulée
container_cpu_usage_seconds_total

# Throttling CPU
container_cpu_cfs_throttled_seconds_total
container_cpu_cfs_throttled_periods_total
container_cpu_cfs_periods_total

# Limite et request CPU
container_spec_cpu_quota
container_spec_cpu_period
container_spec_cpu_shares
```

#### Mémoire des containers
```promql
# Utilisation mémoire actuelle
container_memory_usage_bytes
container_memory_working_set_bytes  # Mémoire active

# Cache et RSS
container_memory_cache
container_memory_rss

# Limites mémoire
container_spec_memory_limit_bytes
container_memory_max_usage_bytes

# Failures
container_memory_failures_total
```

#### I/O des containers
```promql
# I/O disque
container_fs_reads_bytes_total
container_fs_writes_bytes_total
container_fs_reads_total
container_fs_writes_total

# Utilisation du filesystem
container_fs_usage_bytes
container_fs_limit_bytes
```

#### Réseau des containers
```promql
# Trafic réseau
container_network_receive_bytes_total
container_network_transmit_bytes_total

# Paquets
container_network_receive_packets_total
container_network_transmit_packets_total

# Erreurs
container_network_receive_errors_total
container_network_transmit_errors_total

# Drops
container_network_receive_packets_dropped_total
container_network_transmit_packets_dropped_total
```

### Requêtes utiles avec cAdvisor

```promql
# Utilisation CPU par container (en cores)
rate(container_cpu_usage_seconds_total[5m])

# Utilisation mémoire par pod
sum by (pod) (container_memory_working_set_bytes)

# Pourcentage d'utilisation mémoire vs limite
container_memory_working_set_bytes /
container_spec_memory_limit_bytes * 100

# Throttling CPU
rate(container_cpu_cfs_throttled_seconds_total[5m])

# I/O disque par pod
sum by (pod) (rate(container_fs_writes_bytes_total[5m]))

# Top 10 pods par utilisation CPU
topk(10,
  sum by (pod) (
    rate(container_cpu_usage_seconds_total[5m])
  )
)
```

## Combinaison des trois exporters

### Vue d'ensemble complète

La combinaison des trois exporters donne une vue à 360° :

```promql
# Vue complète d'un pod
# De kube-state-metrics : État désiré
kube_pod_container_resource_requests{resource="cpu"}
kube_pod_container_resource_limits{resource="cpu"}
kube_pod_status_phase

# De cAdvisor : Utilisation réelle
rate(container_cpu_usage_seconds_total[5m])
container_memory_working_set_bytes

# Du Node Exporter : Contexte du node
node_memory_MemAvailable_bytes
rate(node_cpu_seconds_total{mode="idle"}[5m])
```

### Dashboards types combinant les métriques

#### Dashboard Node Overview
```promql
# CPU Node (Node Exporter)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Mémoire Node (Node Exporter)
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Nombre de pods sur le node (kube-state-metrics)
count by (node) (kube_pod_info)

# Utilisation CPU des pods sur ce node (cAdvisor)
sum by (node) (rate(container_cpu_usage_seconds_total[5m]))
```

#### Dashboard Pod Performance
```promql
# État du pod (kube-state-metrics)
kube_pod_status_phase{pod="my-pod"}

# CPU demandé vs utilisé
kube_pod_container_resource_requests{pod="my-pod", resource="cpu"}
rate(container_cpu_usage_seconds_total{pod="my-pod"}[5m])

# Mémoire demandée vs utilisée
kube_pod_container_resource_requests{pod="my-pod", resource="memory"}
container_memory_working_set_bytes{pod="my-pod"}

# Restarts
kube_pod_container_status_restarts_total{pod="my-pod"}
```

#### Dashboard Cluster Capacity
```promql
# Capacité totale CPU (Node Exporter)
sum(machine_cpu_cores)

# CPU alloué (kube-state-metrics)
sum(kube_pod_container_resource_requests{resource="cpu"})

# CPU utilisé (cAdvisor)
sum(rate(container_cpu_usage_seconds_total[5m]))

# Ratio d'allocation
sum(kube_pod_container_resource_requests{resource="cpu"}) /
sum(kube_node_status_allocatable{resource="cpu"})
```

## Configuration optimale pour un lab

### Ressources recommandées

```yaml
# Pour un lab avec ressources limitées
# node-exporter
resources:
  requests:
    cpu: 10m
    memory: 20Mi
  limits:
    cpu: 50m
    memory: 50Mi

# kube-state-metrics
resources:
  requests:
    cpu: 50m
    memory: 100Mi
  limits:
    cpu: 100m
    memory: 200Mi

# cAdvisor (intégré au kubelet)
# Pas de configuration séparée
```

### Optimisation du scraping

```yaml
scrape_configs:
  # Node Exporter - Scraping moins fréquent
  - job_name: 'node-exporter'
    scrape_interval: 30s  # Au lieu de 15s
    kubernetes_sd_configs:
    - role: pod
      namespaces:
        names: ['monitoring']
    relabel_configs:
    - source_labels: [__meta_kubernetes_pod_label_app]
      action: keep
      regex: node-exporter

  # kube-state-metrics - Normal
  - job_name: 'kube-state-metrics'
    scrape_interval: 30s
    kubernetes_sd_configs:
    - role: service
      namespaces:
        names: ['monitoring']
    relabel_configs:
    - source_labels: [__meta_kubernetes_service_name]
      action: keep
      regex: kube-state-metrics

  # cAdvisor - Plus fréquent pour containers
  - job_name: 'cadvisor'
    scrape_interval: 15s
    scheme: https
    tls_config:
      insecure_skip_verify: true
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    kubernetes_sd_configs:
    - role: node
    relabel_configs:
    - target_label: __metrics_path__
      replacement: /metrics/cadvisor
```

## Troubleshooting

### Problèmes courants et solutions

#### Node Exporter ne démarre pas
```bash
# Vérifier les permissions
kubectl describe pod -n monitoring node-exporter-xxx

# Solution commune : hostNetwork requis
# Ajouter dans la spec du pod :
hostNetwork: true
hostPID: true
hostIPC: true
```

#### kube-state-metrics : Erreurs RBAC
```bash
# Vérifier les permissions
kubectl auth can-i list pods --as=system:serviceaccount:monitoring:kube-state-metrics

# Solution : Vérifier le ClusterRole
kubectl get clusterrole kube-state-metrics -o yaml
```

#### cAdvisor métriques manquantes
```bash
# Vérifier que le kubelet expose les métriques
curl -k https://NODE_IP:10250/metrics/cadvisor

# Si erreur 403 : Problème d'authentification
# Vérifier le token et certificat dans Prometheus config
```

### Commandes de diagnostic

```bash
# Vérifier les endpoints exposés
# Node Exporter
kubectl exec -n monitoring node-exporter-xxx -- curl localhost:9100/metrics | head -20

# kube-state-metrics
kubectl exec -n monitoring kube-state-metrics-xxx -- curl localhost:8080/metrics | grep "kube_deployment"

# cAdvisor (depuis un node)
curl -k -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  https://kubernetes.default.svc/api/v1/nodes/$(hostname)/proxy/metrics/cadvisor | head -20

# Vérifier dans Prometheus
# Requête pour voir si les métriques arrivent
curl -G http://localhost:9090/api/v1/query --data-urlencode 'query=up{job=~"node.*|kube.*|cadvisor"}'
```

## Alertes basées sur les exporters

### Exemples d'alertes critiques

```yaml
# alerts.yaml
groups:
- name: node_alerts
  rules:
  - alert: NodeDown
    expr: up{job="node-exporter"} == 0
    for: 2m
    annotations:
      summary: "Node {{ $labels.instance }} is down"

  - alert: HighMemoryUsage
    expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 0.9
    for: 5m
    annotations:
      summary: "Memory usage is above 90% on {{ $labels.instance }}"

- name: kubernetes_alerts
  rules:
  - alert: PodCrashLooping
    expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
    annotations:
      summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"

  - alert: DeploymentReplicasMismatch
    expr: |
      kube_deployment_spec_replicas !=
      kube_deployment_status_replicas_available
    for: 10m
    annotations:
      summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} has replica mismatch"

- name: container_alerts
  rules:
  - alert: ContainerCPUThrottling
    expr: |
      rate(container_cpu_cfs_throttled_seconds_total[5m]) > 1
    annotations:
      summary: "Container {{ $labels.pod }}/{{ $labels.container }} is being CPU throttled"
```

## Bonnes pratiques

### 1. Organisation des exporters

- **Node Exporter** : Un par node (DaemonSet)
- **kube-state-metrics** : Une instance pour tout le cluster
- **cAdvisor** : Automatique via kubelet

### 2. Labels cohérents

Toujours maintenir des labels cohérents entre exporters :
```promql
# Jointure sur le label 'pod'
kube_pod_container_resource_limits * on(pod)
  group_left() container_memory_usage_bytes
```

### 3. Filtrage des métriques

Pour économiser les ressources :
```yaml
# Node Exporter : Désactiver les collectors inutiles
- --no-collector.arp
- --no-collector.bcache
- --no-collector.bonding

# kube-state-metrics : Limiter les ressources surveillées
- --resources=pods,deployments,services,nodes
```

### 4. Monitoring des exporters eux-mêmes

```promql
# Santé des exporters
up{job=~"node-exporter|kube-state-metrics"}

# Durée de scrape
scrape_duration_seconds{job=~"node-exporter|kube-state-metrics"}

# Nombre de métriques
scrape_samples_scraped{job=~"node-exporter|kube-state-metrics"}
```

## Checklist de validation

- [ ] Node Exporter déployé sur tous les nodes
- [ ] kube-state-metrics opérationnel avec RBAC correct
- [ ] Métriques cAdvisor accessibles via kubelet
- [ ] Prometheus scrape tous les exporters avec succès
- [ ] Métriques de base disponibles (CPU, RAM, pods)
- [ ] Labels cohérents entre les exporters
- [ ] Ressources allouées appropriées pour le lab
- [ ] Intervalles de scraping optimisés
- [ ] Alertes de base configurées
- [ ] Documentation des métriques custom

## Métriques essentielles à surveiller

### Top 10 des métriques critiques

1. **up** - État des targets (tous les exporters)
2. **node_memory_MemAvailable_bytes** - Mémoire disponible des nodes
3. **node_cpu_seconds_total** - Utilisation CPU des nodes
4. **kube_pod_status_phase** - État des pods
5. **kube_pod_container_status_restarts_total** - Restarts des containers
6. **container_memory_working_set_bytes** - Mémoire active des containers
7. **container_cpu_usage_seconds_total** - CPU des containers
8. **node_filesystem_avail_bytes** - Espace disque disponible
9. **kube_deployment_status_replicas_available** - Replicas disponibles
10. **container_cpu_cfs_throttled_seconds_total** - CPU throttling

### Dashboard de santé rapide

```promql
# Query unique pour vue d'ensemble
(
  label_join(
    count(up{job="node-exporter"} == 1),
    "metric", "", "nodes_up"
  )
) or (
  label_join(
    count(kube_pod_status_phase{phase="Running"} == 1),
    "metric", "", "pods_running"
  )
) or (
  label_join(
    avg(1 - rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100,
    "metric", "", "cpu_usage_percent"
  )
) or (
  label_join(
    avg((1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100),
    "metric", "", "memory_usage_percent"
  )
)
```

## Exporters additionnels utiles pour un lab

### Pour aller plus loin

Une fois les trois exporters essentiels maîtrisés, considérez :

1. **Blackbox Exporter** : Pour le monitoring externe (HTTP, DNS, ICMP)
2. **MySQL/PostgreSQL Exporter** : Si vous avez des bases de données
3. **Redis Exporter** : Pour Redis
4. **NGINX Exporter** : Pour NGINX/Ingress Controller
5. **Custom Exporters** : Pour vos applications

### Exemple : Blackbox Exporter pour monitoring HTTP

```yaml
# blackbox-exporter.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: blackbox-config
  namespace: monitoring
data:
  blackbox.yml: |
    modules:
      http_2xx:
        prober: http
        timeout: 5s
        http:
          valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
          valid_status_codes: [200, 201, 202, 204]
      http_post_2xx:
        prober: http
        timeout: 5s
        http:
          method: POST
          valid_status_codes: [200, 201, 202, 204]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blackbox-exporter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blackbox-exporter
  template:
    metadata:
      labels:
        app: blackbox-exporter
    spec:
      containers:
      - name: blackbox-exporter
        image: prom/blackbox-exporter:v0.24.0
        args:
        - --config.file=/config/blackbox.yml
        ports:
        - containerPort: 9115
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        configMap:
          name: blackbox-config
```

Configuration Prometheus pour Blackbox :
```yaml
- job_name: 'blackbox-http'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
      - https://example.com
      - https://api.myapp.local/health
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox-exporter:9115
```

## Architecture de monitoring complète

### Vue finale avec tous les composants

```
┌───────────────────────────────────────────────────────────┐
│                    Cluster MicroK8s                       │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │                     Master/Control Plane            │  │
│  │  ┌────────────────────────────────────────────┐     │  │
│  │  │         kube-state-metrics (Deploy)        │     │  │
│  │  │         Port: 8080, 8081                   │     │  │
│  │  └────────────────────────────────────────────┘     │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │                     Worker Nodes                    │  │
│  │  ┌────────────────────────────────────────────┐     │  │
│  │  │      Node Exporter (DaemonSet)             │     │  │
│  │  │      Port: 9100                            │     │  │
│  │  └────────────────────────────────────────────┘     │  │
│  │  ┌────────────────────────────────────────────┐     │  │
│  │  │      Kubelet + cAdvisor                    │     │  │
│  │  │      Port: 10250/metrics/cadvisor          │     │  │
│  │  └────────────────────────────────────────────┘     │  │
│  │  ┌────────────────────────────────────────────┐     │  │
│  │  │      Application Pods                      │     │  │
│  │  │      Custom metrics: various ports         │     │  │
│  │  └────────────────────────────────────────────┘     │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │                 Monitoring Namespace                │  │
│  │  ┌────────────────────────────────────────────┐     │  │
│  │  │         Prometheus Server                  │     │  │
│  │  │         Scraping all exporters             │     │  │
│  │  └────────────────────────────────────────────┘     │  │
│  │                        │                            │  │
│  │                        ▼                            │  │
│  │  ┌────────────────────────────────────────────┐     │  │
│  │  │         Grafana                            │     │  │
│  │  │         Visualization                      │     │  │
│  │  └────────────────────────────────────────────┘     │  │
│  └─────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
```

## Migration et évolution

### Évolution naturelle des exporters

1. **Phase 1 : Basics** (Début)
   - Node Exporter
   - kube-state-metrics
   - cAdvisor (automatique)

2. **Phase 2 : Application** (Intermédiaire)
   - Métriques custom dans vos apps
   - Blackbox Exporter
   - Exporters pour vos services (DB, cache)

3. **Phase 3 : Advanced** (Avancé)
   - ServiceMonitor/PodMonitor
   - Custom exporters
   - Push Gateway pour jobs batch

### Passage à la production

Pour un environnement de production, considérez :

```yaml
# Configuration production
# Node Exporter
resources:
  requests:
    cpu: 100m
    memory: 100Mi
  limits:
    cpu: 250m
    memory: 200Mi

# kube-state-metrics avec sharding
replicas: 2
env:
- name: KUBE_STATE_METRICS_SHARD
  value: "0"  # ou "1" pour le second replica

# Haute disponibilité
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - kube-state-metrics
      topologyKey: kubernetes.io/hostname
```

## Préparation pour la suite

Avec vos trois exporters essentiels configurés et opérationnels, vous disposez maintenant d'une base solide de métriques couvrant :

- **Infrastructure** : CPU, mémoire, disque, réseau des nodes
- **Kubernetes** : État et configuration de tous les objets
- **Containers** : Utilisation réelle des ressources

Vous êtes maintenant prêt à :
- Installer et configurer Grafana pour visualiser ces métriques (section 8.06)
- Créer des dashboards exploitant ces trois sources (section 8.07-8.09)
- Configurer des alertes basées sur ces métriques (section 8.11)

Ces trois exporters forment le socle de votre observabilité. Maîtrisez-les bien avant d'ajouter d'autres exporters plus spécialisés.

⏭️
