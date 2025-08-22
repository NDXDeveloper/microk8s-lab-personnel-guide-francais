🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.03 - Configuration des targets et service discovery

## Comprendre les targets dans Prometheus

### Qu'est-ce qu'un target ?

Un **target** (cible) est un endpoint depuis lequel Prometheus va collecter des métriques. Chaque target expose ses métriques sur un endpoint HTTP, généralement `/metrics`.

**Anatomie d'un target :**
```
http://mon-service:8080/metrics
  │         │        │      │
  │         │        │      └── Chemin des métriques
  │         │        └────────── Port
  │         └──────────────────── Nom du service/host
  └────────────────────────────── Protocole
```

### Le cycle de scraping

```
┌─────────────────────────────────────────────────────┐
│                 Cycle de Scraping                   │
│                                                     │
│  1. Prometheus          2. Target répond            │
│     ┌──────┐              ┌──────┐                  │
│     │      │─────GET────▶ │      │                  │
│     │ Prom │              │Target│                  │
│     │      │◀───Metrics───│      │                  │
│     └──────┘              └──────┘                  │
│        │                                            │
│        ▼                                            │
│  3. Stockage TSDB                                   │
│     ┌──────────┐                                    │
│     │ Database │                                    │
│     └──────────┘                                    │
│                                                     │
│  Répétition toutes les X secondes (scrape_interval) │
└─────────────────────────────────────────────────────┘
```

## Types de configuration des targets

### 1. Configuration statique

La méthode la plus simple mais la moins flexible :

```yaml
scrape_configs:
  - job_name: 'mes-services-statiques'
    scrape_interval: 30s
    static_configs:
    - targets:
      - 'service1:8080'
      - 'service2:9090'
      - '192.168.1.10:3000'
      labels:
        environment: 'lab'
        team: 'devops'
```

### 2. Service Discovery dynamique

Prometheus découvre automatiquement les targets selon les changements dans Kubernetes :

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods-auto'
    kubernetes_sd_configs:
    - role: pod  # Découverte basée sur les pods
```

## Service Discovery Kubernetes en détail

### Les différents rôles disponibles

Prometheus peut découvrir différents types d'objets Kubernetes :

| Role | Découvre | Utilisation typique |
|------|----------|-------------------|
| `node` | Nodes du cluster | Métriques système des machines |
| `pod` | Tous les pods | Applications dans les pods |
| `service` | Services K8s | Endpoints de services |
| `endpoints` | Endpoints | Backends des services |
| `ingress` | Ingress controllers | Métriques d'entrée HTTP |

### Configuration complète pour les pods

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
    - role: pod
      namespaces:
        names: ['default', 'production', 'monitoring']  # Limiter aux namespaces

    # Métadonnées disponibles pour le relabeling
    # __meta_kubernetes_pod_name
    # __meta_kubernetes_pod_namespace
    # __meta_kubernetes_pod_label_<labelname>
    # __meta_kubernetes_pod_annotation_<annotationname>
    # __meta_kubernetes_pod_container_name
    # __meta_kubernetes_pod_container_port_name
    # __meta_kubernetes_pod_container_port_number
    # __meta_kubernetes_pod_container_port_protocol
    # __meta_kubernetes_pod_ready
    # __meta_kubernetes_pod_phase
    # __meta_kubernetes_pod_node_name
    # __meta_kubernetes_pod_host_ip
    # __meta_kubernetes_pod_uid

    relabel_configs:
    # Garder seulement les pods avec l'annotation prometheus.io/scrape: "true"
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true

    # Utiliser le port depuis l'annotation prometheus.io/port
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      target_label: __address__
      regex: (.+)
      replacement: ${__meta_kubernetes_pod_ip}:${1}

    # Utiliser le path depuis l'annotation prometheus.io/path
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__
      regex: (.+)

    # Ajouter les labels du pod comme labels Prometheus
    - action: labelmap
      regex: __meta_kubernetes_pod_label_(.+)

    # Ajouter le namespace comme label
    - source_labels: [__meta_kubernetes_pod_namespace]
      action: replace
      target_label: namespace

    # Ajouter le nom du pod comme label
    - source_labels: [__meta_kubernetes_pod_name]
      action: replace
      target_label: pod
```

## Relabeling : La magie de la transformation

### Comprendre le relabeling

Le relabeling permet de modifier les labels avant le scraping (relabel_configs) ou avant le stockage (metric_relabel_configs).

**Actions disponibles :**

| Action | Description | Utilisation |
|--------|-------------|------------|
| `keep` | Garde seulement les targets qui matchent | Filtrage |
| `drop` | Supprime les targets qui matchent | Exclusion |
| `replace` | Remplace la valeur d'un label | Transformation |
| `labelmap` | Copie les labels selon un pattern | Import de labels K8s |
| `labeldrop` | Supprime des labels | Nettoyage |
| `labelkeep` | Garde seulement certains labels | Simplification |

### Exemples pratiques de relabeling

**Exemple 1 : Filtrer par annotation**
```yaml
relabel_configs:
# Ne scraper que les pods avec prometheus.io/scrape: "true"
- source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
  action: keep
  regex: true
```

**Exemple 2 : Construire l'adresse du target**
```yaml
relabel_configs:
# Construire l'adresse depuis l'IP du pod et le port de l'annotation
- source_labels: [__meta_kubernetes_pod_ip, __meta_kubernetes_pod_annotation_prometheus_io_port]
  action: replace
  target_label: __address__
  separator: ':'
  regex: (.+);(.+)
  replacement: ${1}:${2}
```

**Exemple 3 : Exclure des namespaces**
```yaml
relabel_configs:
# Exclure les namespaces système
- source_labels: [__meta_kubernetes_namespace]
  action: drop
  regex: kube-system|kube-public|kube-node-lease
```

**Exemple 4 : Renommer des labels**
```yaml
relabel_configs:
# Renommer le label pod_name en instance
- source_labels: [__meta_kubernetes_pod_name]
  action: replace
  target_label: instance
```

## Configuration pour différents types de targets

### 1. Nodes Kubernetes

```yaml
- job_name: 'kubernetes-nodes'
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

  kubernetes_sd_configs:
  - role: node

  relabel_configs:
  # Ajouter tous les labels du node
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)

  # Définir l'adresse et le port du kubelet
  - source_labels: [__address__]
    action: replace
    target_label: __address__
    regex: ([^:]+)(?::\d+)?
    replacement: $1:10250

  # Définir le metrics path pour le kubelet
  - source_labels: [__meta_kubernetes_node_name]
    action: replace
    target_label: __metrics_path__
    replacement: /metrics
```

### 2. Services Kubernetes

```yaml
- job_name: 'kubernetes-services'
  kubernetes_sd_configs:
  - role: service

  relabel_configs:
  # Ne garder que les services avec l'annotation prometheus.io/scrape
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
    action: keep
    regex: true

  # Utiliser le schéma depuis l'annotation (http/https)
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
    action: replace
    target_label: __scheme__
    regex: (https?)

  # Utiliser le path depuis l'annotation
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
    action: replace
    target_label: __metrics_path__
    regex: (.+)

  # Construire l'adresse avec le nom du service et le port
  - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
    action: replace
    target_label: __address__
    regex: ([^:]+)(?::\d+)?;(\d+)
    replacement: $1:$2

  # Ajouter les labels du service
  - action: labelmap
    regex: __meta_kubernetes_service_label_(.+)

  # Ajouter le namespace
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: kubernetes_namespace

  # Ajouter le nom du service
  - source_labels: [__meta_kubernetes_service_name]
    action: replace
    target_label: kubernetes_name
```

### 3. Ingress Controllers

```yaml
- job_name: 'kubernetes-ingresses'
  kubernetes_sd_configs:
  - role: ingress

  relabel_configs:
  # Garder tous les labels de l'ingress
  - action: labelmap
    regex: __meta_kubernetes_ingress_label_(.+)

  # Ajouter le namespace
  - source_labels: [__meta_kubernetes_namespace]
    target_label: namespace

  # Ajouter le nom de l'ingress
  - source_labels: [__meta_kubernetes_ingress_name]
    target_label: ingress_name

  # Créer un label pour chaque host
  - source_labels: [__meta_kubernetes_ingress_host]
    target_label: host

  # Créer un label pour le path
  - source_labels: [__meta_kubernetes_ingress_path]
    target_label: path
```

## Annotations Kubernetes pour Prometheus

### Annotations standard pour les pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-application
  annotations:
    prometheus.io/scrape: "true"        # Active le scraping
    prometheus.io/port: "8080"          # Port des métriques
    prometheus.io/path: "/metrics"      # Chemin des métriques
    prometheus.io/scheme: "http"        # Protocole (http/https)
    prometheus.io/interval: "30s"       # Intervalle de scraping personnalisé
spec:
  containers:
  - name: app
    image: mon-app:latest
    ports:
    - containerPort: 8080
      name: metrics
```

### Annotations pour les services

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-service
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/custom/metrics"
spec:
  selector:
    app: mon-app
  ports:
  - port: 80
    targetPort: 8080
    name: http
  - port: 9090
    targetPort: 9090
    name: metrics
```

### Exemple complet : Déploiement avec métriques

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: exemple-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: exemple-app
  template:
    metadata:
      labels:
        app: exemple-app
        version: v1
        environment: production
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: app
        image: exemple/app:latest
        ports:
        - containerPort: 3000
          name: http
        - containerPort: 8080
          name: metrics
        env:
        - name: METRICS_PORT
          value: "8080"
---
apiVersion: v1
kind: Service
metadata:
  name: exemple-app
  namespace: production
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
spec:
  selector:
    app: exemple-app
  ports:
  - port: 80
    targetPort: 3000
    name: http
  - port: 8080
    targetPort: 8080
    name: metrics
```

## Configuration avancée du scraping

### Paramètres de scraping personnalisés

```yaml
scrape_configs:
  - job_name: 'slow-targets'
    scrape_interval: 60s      # Au lieu des 15s par défaut
    scrape_timeout: 30s       # Timeout plus long
    metrics_path: /metrics
    scheme: https

    # Configuration TLS
    tls_config:
      insecure_skip_verify: true  # Pour les certificats auto-signés
      # ou
      ca_file: /path/to/ca.crt
      cert_file: /path/to/cert.crt
      key_file: /path/to/key.key

    # Authentification basique
    basic_auth:
      username: prometheus
      password_file: /path/to/password

    # Ou Bearer token
    bearer_token_file: /path/to/token

    # Paramètres HTTP
    params:
      module: [http_2xx]  # Paramètres de requête additionnels

    static_configs:
    - targets: ['secure-service:443']
```

### Limitation du nombre de métriques

```yaml
scrape_configs:
  - job_name: 'limited-metrics'
    kubernetes_sd_configs:
    - role: pod

    # Limiter les métriques après scraping
    metric_relabel_configs:
    # Garder seulement certaines métriques
    - source_labels: [__name__]
      action: keep
      regex: 'http_requests_total|http_duration_seconds|up'

    # Supprimer les métriques avec certains labels
    - source_labels: [environment]
      action: drop
      regex: 'test|dev'

    # Supprimer des labels inutiles
    - regex: 'pod_template_hash|controller_revision_hash'
      action: labeldrop
```

## Monitoring des targets dans Prometheus

### Interface Web des targets

Accéder à l'interface des targets :
```bash
# Port-forward vers Prometheus
kubectl port-forward -n monitoring svc/prometheus 9090:9090

# Ouvrir dans le navigateur
# http://localhost:9090/targets
```

### États possibles des targets

| État | Description | Action |
|------|-------------|--------|
| **UP** (vert) | Target accessible et métriques collectées | Aucune |
| **DOWN** (rouge) | Target inaccessible | Vérifier connectivité |
| **UNKNOWN** (gris) | Pas encore scrapé | Attendre le prochain cycle |

### Debugging des targets DOWN

```bash
# 1. Vérifier que le pod/service existe
kubectl get pods -n <namespace> -l app=<app-name>
kubectl get svc -n <namespace>

# 2. Tester l'accès aux métriques depuis Prometheus
kubectl exec -n monitoring prometheus-xxx -- wget -O- http://service:port/metrics

# 3. Vérifier les annotations
kubectl describe pod <pod-name> -n <namespace> | grep prometheus

# 4. Vérifier les logs de Prometheus
kubectl logs -n monitoring prometheus-xxx | grep "target"

# 5. Utiliser la requête PromQL pour voir les targets
# Dans l'interface Prometheus, exécuter :
# up{job="kubernetes-pods"}
```

## ServiceMonitor et PodMonitor (Prometheus Operator)

### Qu'est-ce qu'un ServiceMonitor ?

Si vous utilisez Prometheus Operator, vous pouvez définir des targets via des CRDs :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: exemple-app-monitor
  namespace: monitoring
spec:
  # Sélecteur de namespace
  namespaceSelector:
    matchNames:
    - production
    - staging

  # Sélecteur de services
  selector:
    matchLabels:
      app: exemple-app

  # Configuration des endpoints
  endpoints:
  - port: metrics        # Nom du port dans le service
    interval: 30s        # Intervalle de scraping
    path: /metrics       # Path des métriques
    scheme: http         # Protocole

    # Relabeling optionnel
    relabelings:
    - sourceLabels: [__meta_kubernetes_service_name]
      targetLabel: service

    # Metric relabeling optionnel
    metricRelabelings:
    - sourceLabels: [__name__]
      regex: 'go_.*'
      action: drop
```

### PodMonitor pour les pods sans service

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: pods-monitor
  namespace: monitoring
spec:
  # Sélecteur de pods
  selector:
    matchLabels:
      monitoring: "true"

  # Sélecteur de namespaces
  namespaceSelector:
    any: true  # Tous les namespaces

  # Configuration des pods
  podMetricsEndpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

## Cas pratiques courants

### Cas 1 : Application multi-ports

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-port-app
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"  # Port principal des métriques
    # Pour plusieurs ports, utiliser ServiceMonitor
spec:
  containers:
  - name: app
    ports:
    - containerPort: 8080
      name: http
    - containerPort: 9090
      name: metrics
    - containerPort: 9091
      name: debug-metrics
```

### Cas 2 : Métriques avec authentification

```yaml
# ConfigMap avec credentials
apiVersion: v1
kind: Secret
metadata:
  name: metrics-auth
  namespace: monitoring
type: Opaque
data:
  username: cHJvbWV0aGV1cw==  # prometheus en base64
  password: c2VjcmV0          # secret en base64

---
# Configuration Prometheus
scrape_configs:
  - job_name: 'secure-app'
    basic_auth:
      username: prometheus
      password_file: /etc/prometheus/secrets/password
    kubernetes_sd_configs:
    - role: pod
      namespaces:
        names: ['secure-namespace']
```

### Cas 3 : Blackbox monitoring (probe externe)

```yaml
- job_name: 'blackbox-http'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
      - https://mon-site.com
      - https://api.mon-service.com/health
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox-exporter:9115
```

## Optimisation du service discovery

### Réduire la charge sur l'API Kubernetes

```yaml
# Limiter les namespaces scannés
kubernetes_sd_configs:
- role: pod
  namespaces:
    names:
    - default
    - production
    # Au lieu de scanner tous les namespaces

# Utiliser des sélecteurs
- role: pod
  selectors:
  - role: pod
    label: "monitoring=enabled"
```

### Cache et performance

```yaml
# Configuration globale pour réduire la charge
global:
  scrape_interval: 30s      # Augmenter l'intervalle si possible
  scrape_timeout: 10s
  evaluation_interval: 30s

  # Limites externes
  external_labels:
    cluster: 'microk8s'
    environment: 'lab'

# Limite par job
scrape_configs:
  - job_name: 'high-frequency'
    scrape_interval: 5s     # Seulement pour les targets critiques

  - job_name: 'low-frequency'
    scrape_interval: 60s    # Pour les targets moins importantes
```

## Validation et tests

### Vérifier la configuration

```bash
# Valider le fichier de configuration
kubectl exec -n monitoring prometheus-xxx -- promtool check config /etc/prometheus/prometheus.yml

# Recharger la configuration sans redémarrer
kubectl exec -n monitoring prometheus-xxx -- kill -HUP 1

# Vérifier les targets actifs
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, instance: .labels.instance, health: .health}'
```

### Requêtes PromQL pour le monitoring des targets

```promql
# Nombre de targets UP par job
count by (job) (up == 1)

# Targets DOWN
up == 0

# Durée du dernier scrape
scrape_duration_seconds

# Nombre de métriques par target
scrape_samples_scraped

# Targets avec scrape lent (>5s)
scrape_duration_seconds > 5

# Taux d'échec de scraping sur 5 minutes
rate(up[5m]) < 1
```

## Bonnes pratiques

### Organisation des jobs

```yaml
scrape_configs:
  # Grouper par type d'application
  - job_name: 'infrastructure'  # Nodes, système
  - job_name: 'kubernetes'      # Composants K8s
  - job_name: 'applications'    # Apps métier
  - job_name: 'databases'       # BDD
  - job_name: 'monitoring'      # Stack monitoring elle-même
```

### Labels cohérents

Toujours inclure ces labels essentiels :
- `environment`: dev, staging, prod, lab
- `namespace`: namespace Kubernetes
- `service`: nom du service
- `version`: version de l'application
- `team`: équipe responsable

### Documentation des targets

```yaml
# Toujours documenter vos configurations
scrape_configs:
  - job_name: 'payment-service'
    # DESCRIPTION: Service de paiement critique
    # OWNER: team-payments
    # SLA: 99.9%
    # ALERT: PagerDuty haute priorité
    scrape_interval: 10s  # Monitoring fréquent car critique
    kubernetes_sd_configs:
    - role: pod
      namespaces:
        names: ['payments']
```

## Checklist de configuration

- [ ] Service discovery configuré pour tous les types de ressources nécessaires
- [ ] Annotations Prometheus ajoutées aux pods/services
- [ ] Relabeling rules testées et fonctionnelles
- [ ] Tous les targets critiques sont UP
- [ ] Labels cohérents appliqués
- [ ] Intervalles de scraping optimisés
- [ ] Authentification configurée si nécessaire
- [ ] Métriques inutiles filtrées
- [ ] Configuration documentée
- [ ] Monitoring des targets eux-mêmes en place

## Préparation pour la suite

Avec vos targets correctement configurés et le service discovery opérationnel, vous êtes prêt à :
- Apprendre PromQL pour interroger vos métriques (section 8.04)
- Configurer des exporters spécialisés (section 8.05)
- Visualiser vos métriques dans Grafana (section 8.06)

La configuration des targets est cruciale : c'est elle qui détermine quelles données vous collectez et comment vous les organisez pour l'analyse future.

⏭️
